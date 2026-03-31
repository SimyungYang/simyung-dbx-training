# MLflow Tracing — GenAI 앱의 관찰 가능성

## 왜 Tracing이 필요한가요?

GenAI 애플리케이션(RAG 챗봇, AI 에이전트 등)은 전통적인 소프트웨어와 달리, LLM 호출, 문서 검색, Tool 실행 등 **여러 단계가 체인으로 연결**되어 동작합니다. 사용자에게 잘못된 답변이 반환되었을 때, "어느 단계에서 문제가 생겼는지" 파악하기 어렵습니다.

> 💡 **MLflow Tracing**은 GenAI 앱의 **실행 흐름을 단계별로 추적**하는 기능입니다. 각 단계(Span)의 입력, 출력, 지연 시간, 토큰 사용량을 기록하여, 디버깅과 성능 최적화를 가능하게 합니다.

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
| **LLM** | LLM API 호출 | OpenAI, Claude, Llama 호출 |
| **RETRIEVER** | 문서/데이터 검색 | Vector Search, SQL 쿼리 |
| **TOOL** | 도구 실행 | UC Function, API 호출 |
| **CHAIN** | 여러 단계의 연쇄 | LangChain Chain, 파이프라인 |
| **AGENT** | 에이전트의 추론 루프 | Tool 선택 → 실행 → 결과 분석 루프 |
| **EMBEDDING** | 임베딩 변환 | 텍스트 → 벡터 변환 |
| **PARSER** | 응답 파싱 | JSON 파싱, 구조화 |
| **RERANKER** | 검색 결과 재순위화 | Reranker 모델 호출 |

---

## 자동 트레이싱 (Autolog)

MLflow는 주요 GenAI 프레임워크에 대해 **코드 한 줄로 자동 트레이싱**을 활성화할 수 있습니다.

### 지원 프레임워크

| 프레임워크 | 활성화 코드 | 설명 |
|-----------|-----------|------|
| **OpenAI** | `mlflow.openai.autolog()` | ChatCompletion, Embedding 호출 추적 |
| **LangChain** | `mlflow.langchain.autolog()` | Chain, Agent, Tool 호출 추적 |
| **LangGraph** | `mlflow.langchain.autolog()` | 그래프 기반 에이전트 추적 |
| **LlamaIndex** | `mlflow.llama_index.autolog()` | 인덱스 조회, LLM 호출 추적 |
| **DSPy** | `mlflow.dspy.autolog()` | 프로그래밍 방식 LLM 활용 추적 |
| **Anthropic** | `mlflow.anthropic.autolog()` | Claude API 호출 추적 |
| **CrewAI** | `mlflow.crewai.autolog()` | 멀티 에이전트 추적 |
| **AutoGen** | `mlflow.autogen.autolog()` | 멀티 에이전트 대화 추적 |
| **Databricks SDK** | 자동 | Foundation Model API 호출 자동 추적 |

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

자동 트레이싱이 지원되지 않는 프레임워크나 커스텀 로직에는 **수동으로 Span을 추가**할 수 있습니다.

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

## Trace 조회 및 분석

### MLflow UI에서 확인

Experiment 페이지에서 **Traces** 탭을 클릭하면, 모든 트레이스를 시간순으로 확인할 수 있습니다. 각 트레이스를 클릭하면 Span 트리, 입출력, 메트릭을 시각적으로 볼 수 있습니다.

### Python API로 조회

```python
import mlflow

# 특정 실험의 최근 트레이스 조회
traces = mlflow.search_traces(
    experiment_ids=["12345"],
    max_results=10,
    order_by=["timestamp DESC"]
)

# 특정 트레이스 상세 조회
trace = mlflow.get_trace(trace_id="abc-123-def")
for span in trace.data.spans:
    print(f"  {span.name}: {span.status} ({span.end_time - span.start_time}ms)")
```

### SQL로 조회 (Unity Catalog 통합)

> 🆕 **MLflow Traces in Unity Catalog (Preview)**: 트레이스를 Unity Catalog에 저장하여 **SQL로 직접 조회**할 수 있습니다. 무제한 용량으로 보존하고, 대시보드에서 모니터링할 수 있습니다.

```sql
-- 최근 실패한 트레이스 조회
SELECT
    trace_id,
    timestamp,
    status,
    total_tokens,
    latency_ms
FROM system.mlflow.traces
WHERE status = 'ERROR'
    AND timestamp >= CURRENT_DATE() - INTERVAL 1 DAY
ORDER BY timestamp DESC;

-- 토큰 사용량 추이
SELECT
    DATE(timestamp) AS day,
    COUNT(*) AS trace_count,
    SUM(total_tokens) AS total_tokens,
    AVG(latency_ms) AS avg_latency
FROM system.mlflow.traces
GROUP BY DATE(timestamp)
ORDER BY day DESC;
```

---

## 프로덕션 모니터링

Model Serving 엔드포인트에 배포된 에이전트는 **자동으로 모든 요청이 트레이싱**됩니다. Inference Table과 결합하여 프로덕션 품질을 모니터링할 수 있습니다.

| 구성 요소 | 역할 | 연결 |
|-----------|------|------|
| **사용자 요청** | 요청을 전송합니다 | Model Serving Endpoint로 전달 |
| **Model Serving Endpoint** | 요청을 수신합니다 | AI 에이전트에 전달 |
| **AI 에이전트** | 요청을 처리합니다 | MLflow Trace + Inference Table에 기록 |
| **MLflow Trace** | 실행 흐름을 자동 기록합니다 | 모니터링 대시보드에 데이터 제공 |
| **Inference Table** | 입출력을 기록합니다 | 모니터링 대시보드에 데이터 제공 |
| **모니터링 대시보드** | 종합 모니터링을 제공합니다 | Trace + Inference Table 데이터를 시각화 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Trace** | GenAI 앱의 한 번의 실행 전체를 기록한 것입니다 |
| **Span** | Trace 내의 개별 단계(LLM 호출, 검색, Tool 실행 등)입니다 |
| **Autolog** | 한 줄로 자동 트레이싱을 활성화합니다. 8+ 프레임워크를 지원합니다 |
| **@mlflow.trace** | 데코레이터로 커스텀 함수에 수동 트레이싱을 추가합니다 |
| **Unity Catalog 통합** | 트레이스를 SQL로 조회하고 대시보드에서 모니터링할 수 있습니다 |

---

## 참고 링크

- [Databricks: MLflow Tracing](https://docs.databricks.com/aws/en/mlflow/mlflow-tracing.html)
- [MLflow: Tracing](https://mlflow.org/docs/latest/llms/tracing/index.html)
- [Databricks: Instrument agents](https://docs.databricks.com/aws/en/generative-ai/agent-framework/trace-agents.html)
