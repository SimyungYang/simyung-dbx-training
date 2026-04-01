# 에이전트 구축

## Databricks에서 에이전트를 구축하는 방법

Databricks는 AI 에이전트를 구축하기 위한 **Mosaic AI Agent Framework** 를 제공합니다. 에이전트의 표준 인터페이스인 **ChatAgent** 를 구현하면, MLflow로 추적하고, Model Serving으로 배포하고, Agent Evaluation으로 평가할 수 있습니다.

---

## ChatAgent 인터페이스

> 💡 **ChatAgent** 는 Databricks에서 AI 에이전트를 구축하기 위한 **표준 인터페이스** 입니다. 이 인터페이스를 구현하면 MLflow 로깅, Model Serving 배포, Review App, Agent Evaluation 등 Databricks의 모든 에이전트 도구와 자동으로 연동됩니다.

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

에이전트가 사용할 수 있는 **도구(Tool)** 를 Unity Catalog의 SQL/Python 함수로 정의할 수 있습니다. 이렇게 하면 도구에 대한 ** 거버넌스(권한 관리, 감사)** 가 자동으로 적용됩니다.

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

> 🆕 **MCP(Model Context Protocol) Servers**: AI 에이전트가 Databricks 리소스(테이블, SQL Warehouse, 볼륨 등)와 외부 API에 ** 안전하게 연결**할 수 있는 관리형 서버입니다.

MCP를 사용하면 에이전트가 접근할 수 있는 리소스를 ** 중앙에서 관리**하고, 각 리소스에 대한 접근 권한을 Unity Catalog와 연동하여 제어할 수 있습니다.

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

## 첫 에이전트를 만들 때 가장 많이 하는 실수 5가지

> 🔥 ** 현업에서 수십 건의 에이전트 프로젝트를 진행하며 발견한 패턴입니다.**

### 실수 1: 프롬프트 없이 LLM에 직접 질문 전달

```python
# ❌ 시스템 프롬프트 없이 사용자 질문만 전달
response = self.llm_client.predict(
    endpoint="databricks-meta-llama-3-3-70b-instruct",
    inputs={"messages": [{"role": "user", "content": user_message}]}
)
# 결과: 에이전트가 범위 밖의 질문에도 아무 답변을 생성하고,
# 환각(hallucination)이 빈번하게 발생합니다

# ✅ 명확한 시스템 프롬프트로 역할과 규칙 정의
system_prompt = """당신은 XX 회사의 고객 지원 전문가입니다.
규칙:
1. 제공된 문서에 있는 정보만 사용하여 답변합니다.
2. 문서에 없는 내용은 "확인 후 답변드리겠습니다"라고 안내합니다.
3. 항상 한국어로 답변합니다.
4. 개인정보(전화번호, 주소 등)는 절대 제공하지 않습니다."""
```

### 실수 2: 에러 처리 없이 Tool 호출

```python
# ❌ Tool 호출 실패 시 에이전트가 통째로 에러
result = self.vector_index.similarity_search(query_text=query, num_results=5)

# ✅ Graceful degradation 패턴
def _safe_search(self, query: str) -> list:
    try:
        results = self.vector_index.similarity_search(
            query_text=query, num_results=5
        )
        return results["result"]["data_array"]
    except Exception as e:
        # 검색 실패 시 빈 결과 반환, LLM이 "문서를 찾지 못했습니다"로 응답
        import logging
        logging.warning(f"Vector search failed: {e}")
        return []
```

### 실수 3: 대화 이력을 무한정 전달

```python
# ❌ 모든 대화 이력을 LLM에 전달 → 토큰 한도 초과, 비용 폭증
messages_for_llm = [{"role": m.role, "content": m.content} for m in messages]

# ✅ 최근 N턴만 유지 + 시스템 프롬프트 토큰 예산 관리
MAX_HISTORY = 10  # 최근 10개 메시지만 유지
recent_messages = messages[-MAX_HISTORY:]
messages_for_llm = [
    {"role": "system", "content": system_prompt},
    *[{"role": m.role, "content": m.content} for m in recent_messages]
]
```

> 💡 **현업 팁**: 대화가 20턴을 넘어가면 토큰 비용이 급격히 증가합니다. 대부분의 고객 지원 시나리오에서는 최근 5~10턴이면 충분합니다.

### 실수 4: predict_stream()을 구현하지 않음

```python
# ❌ predict()만 구현 → 긴 답변에서 사용자가 빈 화면을 오래 봐야 함
# predict_stream()이 없으면 Review App, Playground에서 스트리밍 미지원

# ✅ 반드시 predict_stream()도 구현하세요
def predict_stream(self, messages, context=None):
    for chunk in self._stream_llm_response(messages):
        yield ChatAgentChunk(
            delta=ChatAgentMessage(role="assistant", content=chunk)
        )
```

### 실수 5: MLflow에 로깅하지 않고 바로 배포

```python
# ❌ 로깅 없이 배포 → 어떤 버전이 배포되었는지 추적 불가
# agents.deploy()는 MLflow에 등록된 모델만 배포 가능

# ✅ 항상 MLflow에 로깅한 후 배포
with mlflow.start_run():
    mlflow.log_param("llm_model", "llama-3.3-70b")
    mlflow.log_param("temperature", 0.1)
    mlflow.pyfunc.log_model(
        artifact_path="agent",
        python_model=agent,
        registered_model_name="catalog.schema.my_agent"
    )
# 이후 배포
agents.deploy(model_name="catalog.schema.my_agent", model_version=1)
```

---

## ChatAgent vs LangChain 실전 비교

| 비교 항목 | ChatAgent (직접 구현) | LangChain/LangGraph |
|-----------|---------------------|---------------------|
| ** 코드 복잡도** | 낮음 (Python 클래스 하나) | 중간~높음 (Chain, Graph 개념 학습 필요) |
| ** 디버깅** | 쉬움 (표준 Python 디버깅) | 어려움 (프레임워크 내부 동작 파악 필요) |
| ** 유연성** | 매우 높음 (완전한 제어) | 높음 (프레임워크 확장 가능) |
| **Databricks 통합** | 네이티브 (MLflow, Review App 자동 연동) | 지원 (mlflow.langchain 모듈) |
| ** 멀티 스텝 로직** | 직접 구현해야 합니다 | LangGraph로 상태 머신 구현 가능 |
| ** 커뮤니티/생태계** | Databricks 문서 중심 | 대규모 오픈소스 커뮤니티 |
| ** 프로덕션 안정성** | 높음 (의존성 최소화) | 중간 (프레임워크 버전 업데이트 영향) |
| ** 학습 곡선** | 낮음 (Python만 알면 됨) | 높음 (Chain, Agent, Tool, Memory 개념) |

### 실전 선택 기준

** 질문: "어떤 방식으로 에이전트를 구축할 것인가?"**

| 상황 | 권장 방식 | 이유 |
|------|---------|------|
| 단순 RAG + Tool 호출 (1~3개) | **ChatAgent 직접 구현** | 가장 간단하고 디버깅이 쉬움 |
| 복잡한 멀티 스텝 로직 (조건 분기, 반복, 상태 관리) | **LangGraph + ChatAgent** | 그래프 기반으로 복잡한 흐름 관리 가능 |
| 빠른 프로토타이핑, 비개발자도 참여 | **Agent Builder (No-Code)** | UI에서 클릭으로 에이전트 구성 |
| 기존 LangChain/CrewAI 코드가 있음 | ** 기존 프레임워크 + MLflow 래핑** | 기존 투자 활용, MLflow로 통합 관리 |

> 🔥 ** 현업에서는**: 에이전트의 80%는 "문서 검색 + Tool 1~2개 호출"로 충분합니다. 이런 경우 **LangChain을 도입하면 오히려 불필요한 복잡성만 추가** 됩니다. LangGraph가 정말 필요한 경우는 "에이전트가 스스로 판단하여 여러 단계를 반복적으로 수행해야 하는 경우"뿐입니다. **단순한 것부터 시작하고, 필요할 때만 복잡성을 추가하세요.**

---

## 프로덕션 에이전트의 에러 처리 패턴

프로덕션 에이전트는 다양한 장애 상황에 대비해야 합니다. 아래는 현업에서 검증된 에러 처리 패턴입니다.

### 패턴 1: Circuit Breaker (회로 차단기)

LLM이나 Vector Search가 지속적으로 실패할 때, 무한 재시도 대신 빠르게 실패를 반환합니다.

```python
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=3, recovery_timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = 0
        self.state = "CLOSED"  # CLOSED(정상), OPEN(차단), HALF_OPEN(시도)

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                return "죄송합니다. 시스템이 일시적으로 불안정합니다. 잠시 후 다시 시도해 주세요."

        try:
            result = func(*args, **kwargs)
            self.failure_count = 0
            self.state = "CLOSED"
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise e
```

### 패턴 2: Fallback 응답

```python
def predict(self, messages, context=None):
    try:
        # 정상 흐름: RAG + LLM
        docs = self._search_documents(messages[-1].content)
        answer = self._generate_response(messages, docs)
    except Exception as e:
        # Fallback: 사전 정의된 안내 메시지
        answer = (
            "죄송합니다. 현재 시스템에 일시적인 문제가 있어 답변을 드리기 어렵습니다. "
            "다음 방법으로 도움을 받으실 수 있습니다:\n"
            "1. 고객센터: 1588-XXXX\n"
            "2. 이메일: support@company.com\n"
            "잠시 후 다시 시도해 주시면 감사하겠습니다."
        )
        import logging
        logging.error(f"Agent prediction failed: {e}")

    return ChatAgentResponse(
        messages=[ChatAgentMessage(role="assistant", content=answer)]
    )
```

### 패턴 3: 입력 검증 (Safety Guard)

```python
def _validate_input(self, user_message: str) -> tuple[bool, str]:
    """사용자 입력을 검증하고 문제가 있으면 거부합니다."""
    # 너무 긴 입력 거부
    if len(user_message) > 2000:
        return False, "질문이 너무 깁니다. 2,000자 이내로 줄여주세요."

    # 빈 입력 거부
    if not user_message.strip():
        return False, "질문을 입력해 주세요."

    # Prompt Injection 기본 방어
    injection_patterns = ["ignore previous instructions", "system prompt", "당신은 이제부터"]
    if any(pattern in user_message.lower() for pattern in injection_patterns):
        return False, "해당 요청은 처리할 수 없습니다."

    return True, ""
```

> 💡 ** 현업 팁**: 프로덕션 에이전트에서 가장 중요한 것은 "** 절대 에러 페이지를 보여주지 않는 것**" 입니다. LLM이 실패하든, Vector Search가 실패하든, 사용자에게는 항상 자연어로 된 안내 메시지를 반환해야 합니다. 기술적 에러 메시지(stack trace)가 사용자에게 노출되면 신뢰를 잃습니다.

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
