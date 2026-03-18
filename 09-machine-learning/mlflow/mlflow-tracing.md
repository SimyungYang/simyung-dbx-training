# MLflow Tracing

## GenAI 앱의 관찰 가능성

> 💡 **MLflow Tracing**은 GenAI 애플리케이션(LLM 호출, RAG 파이프라인, AI 에이전트)의 **실행 흐름을 추적**하는 기능입니다. 각 단계별 입력/출력, 지연 시간, 토큰 사용량을 기록합니다.

---

## 자동 트레이싱

```python
# LangChain 자동 트레이싱
mlflow.langchain.autolog()

# OpenAI 자동 트레이싱
mlflow.openai.autolog()
```

## 수동 트레이싱

```python
import mlflow

@mlflow.trace
def rag_pipeline(question: str):
    # 1. 문서 검색
    docs = retrieve_documents(question)
    # 2. LLM 호출
    answer = call_llm(question, docs)
    return answer

@mlflow.trace(span_type="RETRIEVER")
def retrieve_documents(query):
    return vector_search.similarity_search(query, k=3)

@mlflow.trace(span_type="LLM")
def call_llm(question, context):
    return openai.chat(messages=[...])
```

---

## 트레이스 확인

MLflow UI에서 각 트레이스의 호출 트리, 입출력, 지연 시간을 시각적으로 확인할 수 있습니다.

---

## 참고 링크

- [Databricks: MLflow Tracing](https://docs.databricks.com/aws/en/mlflow/mlflow-tracing.html)
