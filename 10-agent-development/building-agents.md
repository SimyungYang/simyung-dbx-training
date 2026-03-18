# 에이전트 구축

## ChatAgent 인터페이스

Databricks에서 에이전트를 구축하는 표준 인터페이스는 **ChatAgent**입니다.

```python
from mlflow.pyfunc import ChatAgent
from mlflow.types.agent import ChatAgentMessage, ChatAgentResponse

class MyAgent(ChatAgent):
    def predict(self, messages: list[ChatAgentMessage], context=None) -> ChatAgentResponse:
        # 1. 사용자 메시지 추출
        user_message = messages[-1].content

        # 2. 관련 문서 검색 (RAG)
        docs = self.vector_search(user_message)

        # 3. LLM 호출
        response = self.call_llm(user_message, docs)

        return ChatAgentResponse(messages=[
            ChatAgentMessage(role="assistant", content=response)
        ])
```

---

## UC Functions를 Tool로 활용

Unity Catalog의 SQL/Python 함수를 에이전트의 **Tool**로 등록할 수 있습니다.

```sql
-- UC Function 생성
CREATE FUNCTION catalog.schema.get_order_status(order_id BIGINT)
RETURNS STRING
RETURN (SELECT status FROM orders WHERE order_id = get_order_status.order_id);
```

```python
# 에이전트에서 Tool로 사용
from databricks.sdk import WorkspaceClient
tools = [{"function": "catalog.schema.get_order_status"}]
```

> 🆕 **Managed MCP Servers**: AI 에이전트가 Databricks 리소스와 외부 API에 안전하게 연결할 수 있는 관리형 MCP(Model Context Protocol) 서버가 추가되었습니다.

---

## 참고 링크

- [Databricks: Build agents](https://docs.databricks.com/aws/en/generative-ai/agent-framework/)
