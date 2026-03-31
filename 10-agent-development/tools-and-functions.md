# 에이전트 도구(Tools)와 UC 함수

## 에이전트 도구란?

AI 에이전트가 외부 세계와 상호작용하려면 **도구(Tool)** 가 필요합니다. 도구는 에이전트가 호출할 수 있는 함수로, 데이터 조회, API 호출, 계산 수행 등의 작업을 수행합니다. Databricks에서는 **Unity Catalog 함수**를 에이전트 도구로 활용하여, 기존 거버넌스 체계(권한, 감사, 리니지)를 그대로 적용할 수 있습니다.

> 💡 **비유**: 에이전트가 "두뇌(LLM)"라면, 도구는 "손(Tool)"입니다. 두뇌가 아무리 똑똑해도, 손이 없으면 문서를 검색하거나 주문을 확인할 수 없습니다.

---

## 도구 유형

Databricks Agent Framework에서 사용할 수 있는 도구 유형은 다음과 같습니다.

| 도구 유형 | 설명 | 적합한 사용 사례 |
|----------|------|--------------|
| **SQL 함수** | SQL로 정의된 UC 함수입니다 | 데이터 조회, 집계, 검색 |
| **Python 함수** | Python으로 정의된 UC 함수입니다 | 외부 API 호출, 복잡한 로직 |
| **Retriever 도구** | Vector Search 기반 문서 검색입니다 | RAG 패턴, 문서 Q&A |
| **기존 엔드포인트** | Model Serving 엔드포인트를 도구로 사용합니다 | 임베딩 생성, 분류 등 |

---

## SQL 도구 (SQL Function Tool)

### 데이터 조회 도구

```sql
-- 주문 상태 조회 도구
CREATE OR REPLACE FUNCTION catalog.schema.get_order_status(
    order_id BIGINT COMMENT '조회할 주문번호'
)
RETURNS TABLE(order_id BIGINT, status STRING, shipping_date DATE, estimated_arrival DATE)
COMMENT '주문번호로 현재 주문 상태와 배송 정보를 조회합니다. 주문 상태가 궁금하거나 배송 정보를 확인할 때 사용하세요.'
RETURN
    SELECT order_id, status, shipping_date, estimated_arrival
    FROM catalog.schema.orders
    WHERE order_id = get_order_status.order_id;
```

### 집계 도구

```sql
-- 고객별 최근 주문 요약 도구
CREATE OR REPLACE FUNCTION catalog.schema.get_customer_summary(
    customer_email STRING COMMENT '고객 이메일 주소'
)
RETURNS TABLE(name STRING, tier STRING, total_orders INT, lifetime_value DECIMAL(12,2), last_order_date DATE)
COMMENT '이메일로 고객 정보와 주문 요약을 조회합니다. 고객 문의 시 기본 정보를 확인하는 데 사용하세요.'
RETURN
    SELECT
        c.name,
        c.tier,
        COUNT(o.order_id) AS total_orders,
        SUM(o.amount) AS lifetime_value,
        MAX(o.order_date) AS last_order_date
    FROM catalog.schema.customers c
    LEFT JOIN catalog.schema.orders o ON c.customer_id = o.customer_id
    WHERE c.email = get_customer_summary.customer_email
    GROUP BY c.name, c.tier;
```

> 💡 **COMMENT가 매우 중요합니다.** LLM은 함수의 COMMENT를 읽고 "이 도구를 언제 사용해야 하는지" 판단합니다. COMMENT에 도구의 목적, 사용 시나리오, 입력 형식을 상세히 기술하세요.

---

## Python 도구 (Python Function Tool)

외부 API 호출, 이메일 발송, 복잡한 계산 등 SQL로 구현하기 어려운 로직에 사용합니다.

```sql
-- 외부 날씨 API 호출 도구
CREATE OR REPLACE FUNCTION catalog.schema.get_weather(
    city STRING COMMENT '날씨를 조회할 도시 이름 (예: Seoul, Busan)'
)
RETURNS STRING
LANGUAGE PYTHON
COMMENT '도시의 현재 날씨 정보를 조회합니다. 배송 예상이나 야외 이벤트 관련 질문에 사용하세요.'
AS $$
    import requests

    response = requests.get(
        f"https://api.weatherapi.com/v1/current.json",
        params={"key": "API_KEY", "q": city}
    )
    if response.ok:
        data = response.json()
        current = data["current"]
        return f"{city}: {current['temp_c']}°C, {current['condition']['text']}"
    else:
        return f"날씨 정보를 가져올 수 없습니다: {response.status_code}"
$$;
```

```sql
-- 이메일 발송 도구
CREATE OR REPLACE FUNCTION catalog.schema.send_notification(
    recipient_email STRING COMMENT '수신자 이메일 주소',
    subject STRING COMMENT '이메일 제목',
    body STRING COMMENT '이메일 본문'
)
RETURNS STRING
LANGUAGE PYTHON
COMMENT '고객에게 이메일 알림을 발송합니다. 주문 확인, 배송 알림 등에 사용하세요.'
AS $$
    import requests

    response = requests.post(
        "https://api.company.com/notify",
        json={"to": recipient_email, "subject": subject, "body": body},
        headers={"Authorization": "Bearer TOKEN"}
    )
    return f"발송 {'성공' if response.ok else '실패'}: {recipient_email}"
$$;
```

---

## Retriever 도구 (Vector Search 기반)

RAG 패턴에서 사용되는 문서 검색 도구입니다. Vector Search Index를 활용하여 의미적으로 관련된 문서를 검색합니다.

```python
from databricks.agents.tools import VectorSearchRetrieverTool

# Vector Search 기반 Retriever 도구
retriever_tool = VectorSearchRetrieverTool(
    index_name="catalog.schema.docs_index",
    num_results=5,
    columns=["content", "title", "source_url"],
    query_type="ANN",  # Approximate Nearest Neighbor
    tool_description="회사 내부 문서, FAQ, 제품 매뉴얼에서 관련 정보를 검색합니다. 고객 질문에 답변하기 위한 정보가 필요할 때 사용하세요."
)
```

---

## UCFunctionToolkit

**UCFunctionToolkit**은 Unity Catalog 함수들을 에이전트 도구로 일괄 등록하는 유틸리티입니다.

```python
from databricks.agents.tools import UCFunctionToolkit

# UC 함수들을 도구 목록으로 변환
toolkit = UCFunctionToolkit(
    function_names=[
        "catalog.schema.get_order_status",
        "catalog.schema.get_customer_summary",
        "catalog.schema.send_notification",
    ]
)

# 도구 목록 확인
for tool in toolkit.tools:
    print(f"Tool: {tool.name}")
    print(f"  Description: {tool.description}")
    print(f"  Parameters: {tool.parameters}")
    print()
```

### LangChain과 연동

```python
from databricks.agents.tools import UCFunctionToolkit
from langchain_community.chat_models import ChatDatabricks
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate

# 1. UC 함수를 LangChain 도구로 변환
toolkit = UCFunctionToolkit(
    function_names=[
        "catalog.schema.get_order_status",
        "catalog.schema.get_customer_summary",
    ]
)
tools = toolkit.tools

# 2. LLM 설정
llm = ChatDatabricks(
    endpoint="databricks-meta-llama-3-3-70b-instruct",
    temperature=0.1
)

# 3. 프롬프트 설정
prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 고객 지원 에이전트입니다. 제공된 도구를 사용하여 정확하게 답변하세요."),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")
])

# 4. 에이전트 생성 및 실행
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 5. 실행
response = executor.invoke({"input": "주문 12345의 상태를 알려주세요", "chat_history": []})
print(response["output"])
```

---

## 도구 호출 흐름

| 단계 | 발신 | 수신 | 내용 |
|------|------|------|------|
| 1 | 사용자 | 에이전트 (LLM) | "주문 12345 상태를 알려주세요" |
| 2 | 에이전트 | (내부 처리) | 의도 분석: 주문 상태 조회 필요 |
| 3 | 에이전트 | 도구 (UC 함수) | `get_order_status(order_id=12345)` 호출 |
| 4 | 도구 | 데이터 (Delta Table) | `SELECT * FROM orders WHERE order_id=12345` 실행 |
| 5 | 데이터 | 도구 | `{status: "배송중", estimated_arrival: "2025-04-02"}` 반환 |
| 6 | 도구 | 에이전트 | 조회 결과 반환 |
| 7 | 에이전트 | (내부 처리) | 결과를 자연어로 변환 |
| 8 | 에이전트 | 사용자 | "주문 12345는 현재 배송 중이며, 4월 2일 도착 예정입니다." |

---

## 도구 설계 Best Practices

| 원칙 | 설명 |
|------|------|
| **명확한 COMMENT** | 함수의 목적, 사용 시기, 입력 형식을 상세히 기술하세요 |
| **파라미터 COMMENT** | 각 파라미터에도 설명을 추가하여 LLM이 올바른 값을 전달하도록 하세요 |
| **단일 책임** | 하나의 도구는 하나의 작업만 수행하세요 (조회와 수정을 분리) |
| **에러 처리** | 잘못된 입력에 대해 명확한 에러 메시지를 반환하세요 |
| **최소 권한** | 도구가 접근하는 데이터의 범위를 최소화하세요 |
| **성능** | 도구의 응답 시간이 길면 에이전트 전체 응답이 느려집니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **UC 함수를 도구로** | Unity Catalog의 SQL/Python 함수를 에이전트 도구로 활용합니다 |
| **SQL 도구** | 데이터 조회, 집계에 적합합니다. RETURNS TABLE로 테이블 반환이 가능합니다 |
| **Python 도구** | 외부 API 호출, 복잡한 로직에 적합합니다 |
| **Retriever 도구** | Vector Search 기반 RAG 문서 검색을 수행합니다 |
| **UCFunctionToolkit** | UC 함수들을 에이전트 도구로 일괄 변환하는 유틸리티입니다 |
| **COMMENT의 중요성** | LLM이 도구 선택과 파라미터 결정에 COMMENT를 참고합니다 |

---

## 참고 링크

- [Databricks: Create agent tools](https://docs.databricks.com/aws/en/generative-ai/agent-framework/create-tools.html)
- [Databricks: UC functions as tools](https://docs.databricks.com/aws/en/generative-ai/agent-framework/author-tools.html)
- [Databricks: UCFunctionToolkit](https://docs.databricks.com/aws/en/generative-ai/agent-framework/uc-function-tools.html)
- [Azure Databricks: Agent tools](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-framework/create-tools)
