# 에이전트 구축

## Databricks에서 에이전트를 구축하는 방법

Databricks는 AI 에이전트를 구축하기 위한 **Mosaic AI Agent Framework**를 제공합니다. 에이전트의 표준 인터페이스인 **ChatAgent**를 구현하면, MLflow로 추적하고, Model Serving으로 배포하고, Agent Evaluation으로 평가할 수 있습니다.

---

## ChatAgent 인터페이스

> 💡 **ChatAgent**는 Databricks에서 AI 에이전트를 구축하기 위한 **표준 인터페이스**입니다. 이 인터페이스를 구현하면 MLflow 로깅, Model Serving 배포, Review App, Agent Evaluation 등 Databricks의 모든 에이전트 도구와 자동으로 연동됩니다.

```python
from mlflow.pyfunc import ChatAgent
from mlflow.types.agent import (
    ChatAgentMessage,
    ChatAgentResponse,
    ChatAgentChunk,
    ChatContext
)

class CustomerSupportAgent(ChatAgent):

    def __init__(self):
        """에이전트 초기화: 모델, 검색 인덱스, 도구 설정"""
        import mlflow.deployments
        self.llm_client = mlflow.deployments.get_deploy_client("databricks")
        self.vector_index = self._init_vector_search()
        self.tools = self._init_tools()

    def predict(
        self,
        messages: list[ChatAgentMessage],
        context: ChatContext = None
    ) -> ChatAgentResponse:
        """동기 응답: 전체 답변을 한 번에 반환합니다"""

        # 1. 사용자의 최신 메시지 추출
        user_message = messages[-1].content

        # 2. 관련 문서 검색 (RAG)
        docs = self._search_documents(user_message)

        # 3. 프롬프트 구성
        system_prompt = self._build_system_prompt(docs)

        # 4. LLM 호출
        response = self.llm_client.predict(
            endpoint="databricks-meta-llama-3-3-70b-instruct",
            inputs={
                "messages": [
                    {"role": "system", "content": system_prompt},
                    *[{"role": m.role, "content": m.content} for m in messages]
                ],
                "temperature": 0.1,
                "max_tokens": 1000
            }
        )

        answer = response["choices"][0]["message"]["content"]

        # 5. 응답 반환
        return ChatAgentResponse(
            messages=[ChatAgentMessage(role="assistant", content=answer)]
        )

    def predict_stream(
        self,
        messages: list[ChatAgentMessage],
        context: ChatContext = None
    ):
        """스트리밍 응답: 토큰 단위로 점진적으로 반환합니다"""
        # SSE(Server-Sent Events) 방식으로 토큰을 하나씩 전달
        for chunk in self._stream_llm_response(messages):
            yield ChatAgentChunk(
                delta=ChatAgentMessage(role="assistant", content=chunk)
            )

    def _search_documents(self, query: str) -> list:
        """Vector Search에서 관련 문서 검색"""
        results = self.vector_index.similarity_search(
            query_text=query,
            columns=["content", "title", "source"],
            num_results=5
        )
        return results["result"]["data_array"]

    def _build_system_prompt(self, docs: list) -> str:
        """검색된 문서를 기반으로 시스템 프롬프트 구성"""
        context = "\n\n".join([f"[{d[1]}]\n{d[0]}" for d in docs])
        return f"""당신은 고객 지원 전문가입니다. 다음 문서를 참고하여 정확하게 답변해 주세요.
문서에 없는 내용은 "확인 후 답변드리겠습니다"라고 안내해 주세요.

참고 문서:
{context}"""
```

---

## Unity Catalog Functions를 Tool로 활용

에이전트가 사용할 수 있는 **도구(Tool)** 를 Unity Catalog의 SQL/Python 함수로 정의할 수 있습니다. 이렇게 하면 도구에 대한 **거버넌스(권한 관리, 감사)** 가 자동으로 적용됩니다.

### SQL Function Tool 생성

```sql
-- 주문 상태 조회 Tool
CREATE OR REPLACE FUNCTION catalog.schema.get_order_status(
    order_id BIGINT COMMENT '조회할 주문번호'
)
RETURNS TABLE(order_id BIGINT, status STRING, updated_at TIMESTAMP)
COMMENT '주문번호로 현재 주문 상태를 조회합니다'
RETURN
    SELECT order_id, status, updated_at
    FROM catalog.schema.orders
    WHERE order_id = get_order_status.order_id;

-- 고객 정보 조회 Tool
CREATE OR REPLACE FUNCTION catalog.schema.get_customer_info(
    customer_email STRING COMMENT '고객 이메일 주소'
)
RETURNS TABLE(name STRING, tier STRING, total_orders INT)
COMMENT '이메일로 고객 정보를 조회합니다'
RETURN
    SELECT name, tier, lifetime_orders AS total_orders
    FROM catalog.schema.gold_customer_360
    WHERE email = get_customer_info.customer_email;

-- Python Function Tool (외부 API 호출 등)
CREATE OR REPLACE FUNCTION catalog.schema.send_notification(
    customer_email STRING COMMENT '알림을 받을 고객 이메일',
    message STRING COMMENT '알림 메시지 내용'
)
RETURNS STRING
LANGUAGE PYTHON
COMMENT '고객에게 이메일 알림을 발송합니다'
AS $$
    import requests
    response = requests.post("https://api.example.com/notify",
        json={"email": customer_email, "message": message})
    return f"알림 발송 {'성공' if response.ok else '실패'}"
$$;
```

### 에이전트에 Tool 연결

```python
# UC Functions를 에이전트의 Tool로 등록
tools = [
    {
        "type": "uc_function",
        "function": {
            "name": "catalog.schema.get_order_status"
        }
    },
    {
        "type": "uc_function",
        "function": {
            "name": "catalog.schema.get_customer_info"
        }
    },
    {
        "type": "uc_function",
        "function": {
            "name": "catalog.schema.send_notification"
        }
    }
]
```

### Tool 호출 흐름

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | 사용자 입력 | "주문 12345 상태를 알려주세요" |
| 2 | LLM 판단 | "get_order_status를 호출해야겠다" |
| 3 | Tool 호출 | `get_order_status(12345)` 실행 |
| 4 | Tool 결과 | 결과: 배송중 |
| 5 | LLM 응답 생성 | 결과를 자연어로 변환 |
| 6 | 최종 응답 | "주문 12345는 현재 배송 중입니다." |

---

## Managed MCP Servers

> 🆕 **MCP(Model Context Protocol) Servers**: AI 에이전트가 Databricks 리소스(테이블, SQL Warehouse, 볼륨 등)와 외부 API에 **안전하게 연결**할 수 있는 관리형 서버입니다.

MCP를 사용하면 에이전트가 접근할 수 있는 리소스를 **중앙에서 관리**하고, 각 리소스에 대한 접근 권한을 Unity Catalog와 연동하여 제어할 수 있습니다.

---

## 에이전트 로깅 (MLflow)

구축한 에이전트를 MLflow에 로깅하면 버전 관리, 평가, 배포가 가능해집니다.

```python
import mlflow

# 에이전트를 MLflow 모델로 로깅
mlflow.set_experiment("/Workspace/agents/customer-support")

with mlflow.start_run(run_name="v1-rag-agent"):
    # 에이전트 메타데이터 기록
    mlflow.log_param("llm_model", "llama-3.3-70b")
    mlflow.log_param("vector_index", "catalog.schema.docs_index")
    mlflow.log_param("num_retrieval_docs", 5)
    mlflow.log_param("temperature", 0.1)

    # 에이전트를 MLflow 모델로 저장
    model_info = mlflow.pyfunc.log_model(
        artifact_path="agent",
        python_model=CustomerSupportAgent(),
        pip_requirements=[
            "mlflow>=2.12",
            "databricks-vectorsearch",
            "databricks-sdk"
        ],
        registered_model_name="catalog.schema.customer_support_agent"
    )

    print(f"Model URI: {model_info.model_uri}")
```

---

## 로컬 테스트

배포 전에 노트북에서 에이전트를 테스트할 수 있습니다.

```python
# 에이전트 인스턴스 생성
agent = CustomerSupportAgent()

# 테스트 대화
response = agent.predict(
    messages=[
        ChatAgentMessage(role="user", content="주문 12345의 배송 상태가 궁금합니다")
    ]
)
print(response.messages[0].content)

# 멀티턴 대화 테스트
response2 = agent.predict(
    messages=[
        ChatAgentMessage(role="user", content="주문 12345의 배송 상태가 궁금합니다"),
        ChatAgentMessage(role="assistant", content=response.messages[0].content),
        ChatAgentMessage(role="user", content="그러면 예상 도착일은 언제인가요?")
    ]
)
print(response2.messages[0].content)
```

---

## 에이전트 개발 패턴 비교

| 패턴 | 설명 | 적합한 경우 |
|------|------|-----------|
| **ChatAgent 직접 구현** | Python으로 전체 로직을 직접 작성합니다 | 완전한 제어가 필요할 때 |
| **LangChain/LangGraph** | 프레임워크를 사용하여 Chain/Agent를 구성합니다 | 복잡한 멀티스텝 로직, 기존 LangChain 경험이 있을 때 |
| **Agent Bricks** | Databricks가 제공하는 사전 구축 에이전트를 사용합니다 | 빠른 프로토타이핑, 표준 RAG/SQL 에이전트 |
| **Foundation Model API + AI Functions** | SQL만으로 간단한 AI 로직을 구현합니다 | 코드 없이 SQL로 처리 가능한 경우 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **ChatAgent** | Databricks 에이전트의 표준 인터페이스입니다. predict()와 predict_stream()을 구현합니다 |
| **UC Functions as Tools** | Unity Catalog 함수를 에이전트의 도구로 사용합니다. 거버넌스가 자동 적용됩니다 |
| **MCP Servers** | 에이전트가 Databricks 리소스에 안전하게 접근하는 관리형 서버입니다 |
| **MLflow 로깅** | 에이전트를 MLflow 모델로 저장하여 버전 관리, 평가, 배포합니다 |

---

## 참고 링크

- [Databricks: Build agents](https://docs.databricks.com/aws/en/generative-ai/agent-framework/)
- [Databricks: Author tools](https://docs.databricks.com/aws/en/generative-ai/agent-framework/author-tools.html)
- [Databricks: ChatAgent](https://docs.databricks.com/aws/en/generative-ai/agent-framework/chat-agent.html)
- [Databricks: MCP Servers](https://docs.databricks.com/aws/en/generative-ai/agent-framework/mcp-servers.html)
