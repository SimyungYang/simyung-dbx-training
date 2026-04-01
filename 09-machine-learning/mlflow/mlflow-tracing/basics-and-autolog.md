# 기본 개념과 자동 트레이싱

## 왜 Tracing이 필요한가요?

GenAI 애플리케이션(RAG 챗봇, AI 에이전트 등)은 전통적인 소프트웨어와 달리, LLM 호출, 문서 검색, Tool 실행 등 **여러 단계가 체인으로 연결** 되어 동작합니다. 사용자에게 잘못된 답변이 반환되었을 때, "어느 단계에서 문제가 생겼는지" 파악하기 어렵습니다.

> 💡 **MLflow Tracing** 은 GenAI 앱의 **실행 흐름을 단계별로 추적** 하는 기능입니다. 각 단계(Span)의 입력, 출력, 지연 시간, 토큰 사용량을 기록하여, 디버깅과 성능 최적화를 가능하게 합니다.

**Trace 예시: 사용자 질문 → 답변**

| Span | 작업 | 입력 | 출력 | 소요 시간 |
|------|------|------|------|-----------|
| Span 1 | 질문 전처리 | "환불 규정이 뭔가요?" | 정규화된 쿼리 | 5ms |
| Span 2 | Vector Search | 정규화된 쿼리 | 관련 문서 3건 | 45ms |
| Span 3 | LLM 호출 | 프롬프트 + 컨텍스트 | 답변 텍스트 (토큰: 입력 850, 출력 120) | 1,200ms |

---

## Span 유형

각 Trace는 여러 개의 **Span(구간)** 으로 구성됩니다. Span은 중첩(Nested)될 수 있어, 호출 트리를 형성합니다.

| Span Type | 설명 | 예시 |
|-----------|------|------|
| **LLM**| LLM API 호출 | OpenAI, Claude, Llama 호출 |
| **RETRIEVER**| 문서/데이터 검색 | Vector Search, SQL 쿼리 |
| **TOOL**| 도구 실행 | UC Function, API 호출 |
| **CHAIN**| 여러 단계의 연쇄 | LangChain Chain, 파이프라인 |
| **AGENT**| 에이전트의 추론 루프 | Tool 선택 → 실행 → 결과 분석 루프 |
| **EMBEDDING**| 임베딩 변환 | 텍스트 → 벡터 변환 |
| **PARSER**| 응답 파싱 | JSON 파싱, 구조화 |
| **RERANKER**| 검색 결과 재순위화 | Reranker 모델 호출 |

---

## 자동 트레이싱 (Autolog)

MLflow는 주요 GenAI 프레임워크에 대해 **코드 한 줄로 자동 트레이싱** 을 활성화할 수 있습니다.

### 지원 프레임워크

| 프레임워크 | 활성화 코드 | 설명 |
|-----------|-----------|------|
| **OpenAI**| `mlflow.openai.autolog()` | ChatCompletion, Embedding 호출 추적 |
| **LangChain**| `mlflow.langchain.autolog()` | Chain, Agent, Tool 호출 추적 |
| **LangGraph**| `mlflow.langchain.autolog()` | 그래프 기반 에이전트 추적 |
| **LlamaIndex**| `mlflow.llama_index.autolog()` | 인덱스 조회, LLM 호출 추적 |
| **DSPy**| `mlflow.dspy.autolog()` | 프로그래밍 방식 LLM 활용 추적 |
| **Anthropic**| `mlflow.anthropic.autolog()` | Claude API 호출 추적 |
| **CrewAI**| `mlflow.crewai.autolog()` | 멀티 에이전트 추적 |
| **AutoGen**| `mlflow.autogen.autolog()` | 멀티 에이전트 대화 추적 |
| **Databricks SDK**| 자동 | Foundation Model API 호출 자동 추적 |

### 사용 예시

```python
import mlflow
import openai

# 한 줄로 자동 트레이싱 활성화
mlflow.openai.autolog()

# 이후 모든 OpenAI 호출이 자동으로 추적됩니다
client = openai.OpenAI()
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "데이터 레이크하우스란?"}]
)
# → MLflow UI에서 입력, 출력, 토큰 사용량, 지연시간을 확인할 수 있습니다
```

```python
# LangChain 자동 트레이싱
mlflow.langchain.autolog()

from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

chain = ChatPromptTemplate.from_template("Explain {topic} in Korean") | ChatOpenAI()
result = chain.invoke({"topic": "data lakehouse"})
# → Chain의 각 단계(프롬프트 → LLM 호출)가 자동으로 추적됩니다
```

---

## 수동 트레이싱

자동 트레이싱이 지원되지 않는 프레임워크나 커스텀 로직에는 **수동으로 Span을 추가** 할 수 있습니다.

### 데코레이터 방식 (권장)

```python
import mlflow

@mlflow.trace
def rag_pipeline(question: str) -> str:
    """전체 RAG 파이프라인"""
    docs = retrieve_documents(question)
    answer = generate_answer(question, docs)
    return answer

@mlflow.trace(span_type="RETRIEVER", name="document_search")
def retrieve_documents(query: str) -> list:
    """Vector Search에서 관련 문서 검색"""
    results = vector_index.similarity_search(
        query_text=query,
        num_results=5
    )
    return results

@mlflow.trace(span_type="LLM", name="llm_call")
def generate_answer(question: str, context: list) -> str:
    """LLM에 질문과 컨텍스트를 전달하여 답변 생성"""
    prompt = f"다음 문서를 참고하여 질문에 답해 주세요.\n\n문서:\n{context}\n\n질문: {question}"
    response = client.chat.completions.create(
        model="databricks-meta-llama-3-3-70b-instruct",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

# 호출하면 전체 Trace가 자동 생성됩니다
answer = rag_pipeline("반품 정책이 어떻게 되나요?")
```

### Context Manager 방식

더 세밀한 제어가 필요한 경우 `with` 문을 사용합니다.

```python
import mlflow

with mlflow.start_span(name="custom_pipeline") as span:
    span.set_inputs({"question": question})

    with mlflow.start_span(name="preprocessing", span_type="CHAIN") as prep_span:
        prep_span.set_inputs({"raw_question": question})
        cleaned = preprocess(question)
        prep_span.set_outputs({"cleaned_question": cleaned})

    with mlflow.start_span(name="search", span_type="RETRIEVER") as search_span:
        search_span.set_inputs({"query": cleaned})
        docs = search(cleaned)
        search_span.set_outputs({"num_results": len(docs)})
        search_span.set_attributes({"search_score": docs[0].score})

    with mlflow.start_span(name="generation", span_type="LLM") as llm_span:
        llm_span.set_inputs({"prompt": prompt, "context_length": len(docs)})
        answer = generate(prompt, docs)
        llm_span.set_outputs({"answer": answer})
        llm_span.set_attributes({
            "token_usage.input": 850,
            "token_usage.output": 120,
            "model": "llama-3.3-70b"
        })

    span.set_outputs({"answer": answer})
```

---
