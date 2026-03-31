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

## Trace 데이터 구조 심층 분석 — Span 트리

### Trace의 구조

하나의 Trace는 **트리(Tree) 구조의 Span 계층**으로 구성됩니다. 루트 Span은 전체 요청을 나타내고, 하위 Span은 각 처리 단계를 나타냅니다.

```
Trace (trace_id: "abc-123")
├── Root Span: "rag_agent" (type: AGENT, 총 1,350ms)
│   ├── Span: "query_rewrite" (type: CHAIN, 15ms)
│   │   └── Span: "llm_rewrite" (type: LLM, 12ms)
│   ├── Span: "document_retrieval" (type: RETRIEVER, 85ms)
│   │   ├── Span: "embedding" (type: EMBEDDING, 25ms)
│   │   └── Span: "vector_search" (type: RETRIEVER, 55ms)
│   ├── Span: "reranking" (type: RERANKER, 30ms)
│   └── Span: "answer_generation" (type: LLM, 1,200ms)
│       └── Span: "response_parse" (type: PARSER, 5ms)
```

### Span의 핵심 속성

| 속성 | 타입 | 설명 |
|------|------|------|
| `span_id` | String | Span의 고유 식별자 |
| `parent_id` | String | 부모 Span ID (루트 Span은 null) |
| `name` | String | Span 이름 (함수명 또는 커스텀 이름) |
| `span_type` | Enum | LLM, RETRIEVER, TOOL, CHAIN, AGENT, EMBEDDING, PARSER, RERANKER |
| `start_time` | Timestamp | 시작 시간 (나노초 정밀도) |
| `end_time` | Timestamp | 종료 시간 |
| `status` | Enum | OK, ERROR |
| `inputs` | JSON | 입력 데이터 (프롬프트, 쿼리 등) |
| `outputs` | JSON | 출력 데이터 (응답, 검색 결과 등) |
| `attributes` | Dict | 커스텀 메타데이터 (토큰 수, 모델명, 점수 등) |
| `events` | List | 예외, 경고 등의 이벤트 |

### Trace 레벨 메타데이터

```python
# Trace에 태그 추가 (필터링/검색에 활용)
trace = mlflow.get_last_active_trace()
mlflow.set_trace_tag("user_id", "user-12345")
mlflow.set_trace_tag("session_id", "session-abc")
mlflow.set_trace_tag("environment", "production")
mlflow.set_trace_tag("model_version", "v2.1")
```

---

## 자동 계측 vs 수동 계측 — 상세 비교

### 자동 계측 (Autolog)의 동작 원리

Autolog는 대상 프레임워크의 핵심 메서드를 **몽키패치(Monkey Patch)**하여 호출 전후에 Span 생성/종료 코드를 주입합니다.

```python
# 내부 동작 의사 코드 (실제 구현은 더 복잡)
original_create = openai.ChatCompletion.create

def patched_create(*args, **kwargs):
    with mlflow.start_span(name="ChatCompletion", span_type="LLM") as span:
        span.set_inputs({"messages": kwargs.get("messages")})
        result = original_create(*args, **kwargs)
        span.set_outputs({"response": result.choices[0].message.content})
        span.set_attributes({
            "token_usage.input": result.usage.prompt_tokens,
            "token_usage.output": result.usage.completion_tokens,
            "model": kwargs.get("model")
        })
        return result

openai.ChatCompletion.create = patched_create
```

### 자동 vs 수동 계측 선택 가이드

| 기준 | 자동 계측 | 수동 계측 |
|------|----------|----------|
| **설정 난이도** | 한 줄 (`mlflow.xxx.autolog()`) | 각 함수에 데코레이터/컨텍스트 추가 |
| **커버리지** | 프레임워크 내부 호출만 | 비즈니스 로직 포함 전체 |
| **커스텀 속성** | 제한적 (프레임워크가 제공하는 정보만) | 자유롭게 추가 가능 |
| **오버헤드** | 매우 낮음 | 매우 낮음 |
| **권장 사용** | 빠른 프로토타이핑, 표준 파이프라인 | 프로덕션, 세밀한 제어 필요 시 |

> 💡 **실무 패턴**: 자동 계측과 수동 계측을 **함께 사용**하는 것이 가장 효과적입니다. 프레임워크 호출은 Autolog로 자동 추적하고, 비즈니스 로직(전처리, 후처리, 캐시 확인 등)은 수동으로 Span을 추가합니다.

```python
# 자동 + 수동 혼합 패턴
mlflow.langchain.autolog()  # LangChain 호출은 자동 추적

@mlflow.trace(name="rag_with_cache", span_type="CHAIN")
def rag_pipeline(question: str) -> str:
    # 수동: 캐시 확인 (자동 추적 대상 아님)
    with mlflow.start_span(name="cache_check") as span:
        cached = check_cache(question)
        span.set_attributes({"cache_hit": cached is not None})
        if cached:
            return cached

    # 자동: LangChain 호출은 autolog가 추적
    result = chain.invoke({"question": question})

    # 수동: 응답 품질 검증 (자동 추적 대상 아님)
    with mlflow.start_span(name="quality_check") as span:
        quality = validate_response(result)
        span.set_attributes({"quality_score": quality})

    return result
```

---

## LangChain/LlamaIndex 통합 상세

### LangChain 통합

LangChain의 모든 주요 컴포넌트가 자동으로 추적됩니다.

| 컴포넌트 | Span Type | 추적 내용 |
|----------|-----------|----------|
| `ChatModel` (ChatOpenAI 등) | LLM | 프롬프트, 응답, 토큰 수, 모델명, 온도 |
| `Retriever` (VectorStoreRetriever) | RETRIEVER | 쿼리, 검색 결과, 문서 수, 점수 |
| `Tool` | TOOL | 도구명, 입력, 출력, 실행 시간 |
| `Chain` (RunnableSequence) | CHAIN | 체인 구성, 각 단계 입출력 |
| `Agent` (AgentExecutor) | AGENT | 추론 과정, 도구 선택 이유, 반복 횟수 |

```python
# LangChain LCEL 파이프라인의 자동 추적
mlflow.langchain.autolog()

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (
    ChatPromptTemplate.from_template("Translate to Korean: {text}")
    | ChatOpenAI(model="gpt-4o")
    | StrOutputParser()
)

# 호출 시 자동으로 3개 Span 생성:
# 1. ChatPromptTemplate (CHAIN)
# 2. ChatOpenAI (LLM) - 토큰, 지연시간 포함
# 3. StrOutputParser (PARSER)
result = chain.invoke({"text": "Hello, world!"})
```

### LlamaIndex 통합

```python
mlflow.llama_index.autolog()

from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 인덱스 생성 시 임베딩 호출이 자동 추적됨
documents = SimpleDirectoryReader("data").load_data()
index = VectorStoreIndex.from_documents(documents)

# 쿼리 시 전체 파이프라인 자동 추적
# Retrieval → Reranking → LLM Generation
query_engine = index.as_query_engine()
response = query_engine.query("데이터 레이크하우스란 무엇인가요?")
```

---

## 프로덕션 트레이스 분석 패턴

프로덕션 환경에서 수집된 Trace 데이터를 활용하여 성능 병목, 비용 최적화, 품질 문제를 분석하는 패턴입니다.

### 1. 지연시간 병목 분석

```sql
-- 가장 느린 Span 유형 식별
SELECT
    span_type,
    AVG(latency_ms) AS avg_latency,
    P50(latency_ms) AS p50_latency,
    P95(latency_ms) AS p95_latency,
    P99(latency_ms) AS p99_latency,
    COUNT(*) AS span_count
FROM system.mlflow.trace_spans
WHERE timestamp >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY span_type
ORDER BY p95_latency DESC;

-- 결과 예시:
-- LLM        avg: 1,200ms  p50: 1,100ms  p95: 2,500ms  p99: 5,000ms
-- RETRIEVER  avg: 80ms     p50: 65ms     p95: 200ms    p99: 500ms
-- EMBEDDING  avg: 25ms     p50: 20ms     p95: 50ms     p99: 100ms
-- → LLM이 전체 지연의 90%를 차지. 모델 축소 또는 캐싱 검토 필요
```

### 2. 토큰 사용량 및 비용 분석

```sql
-- 일별 토큰 사용량 추이
SELECT
    DATE(timestamp) AS day,
    SUM(attributes['token_usage.input']) AS total_input_tokens,
    SUM(attributes['token_usage.output']) AS total_output_tokens,
    (SUM(attributes['token_usage.input']) * 0.00001 +
     SUM(attributes['token_usage.output']) * 0.00003) AS estimated_cost_usd
FROM system.mlflow.trace_spans
WHERE span_type = 'LLM'
    AND timestamp >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY DATE(timestamp)
ORDER BY day;
```

### 3. 검색 품질 분석

```sql
-- Retriever의 검색 결과 수와 응답 품질 상관관계
SELECT
    retriever_span.attributes['num_results'] AS num_docs,
    AVG(CASE WHEN feedback.score >= 4 THEN 1.0 ELSE 0.0 END) AS satisfaction_rate,
    COUNT(*) AS request_count
FROM system.mlflow.trace_spans retriever_span
JOIN user_feedback feedback
    ON retriever_span.trace_id = feedback.trace_id
WHERE retriever_span.span_type = 'RETRIEVER'
GROUP BY num_docs
ORDER BY num_docs;
-- → 검색 문서 수가 3~5개일 때 만족도가 가장 높고,
--   10개 이상이면 오히려 저하 (노이즈 증가)
```

### 4. 에러 패턴 분석

```sql
-- 가장 빈번한 에러 유형
SELECT
    span_name,
    span_type,
    events[0].attributes['exception.type'] AS error_type,
    COUNT(*) AS error_count,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS error_pct
FROM system.mlflow.trace_spans
WHERE status = 'ERROR'
    AND timestamp >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY span_name, span_type, error_type
ORDER BY error_count DESC
LIMIT 10;
```

---

## 트레이스 기반 평가 데이터 수집

프로덕션 트레이스를 활용하여 **모델 평가 데이터셋을 자동으로 구축**하는 패턴입니다. 이는 GenAI 앱의 지속적 개선에 핵심적인 역할을 합니다.

### 패턴 1: 사용자 피드백 연계

```python
import mlflow

# 프로덕션에서 트레이스 수집
@mlflow.trace
def rag_pipeline(question: str) -> dict:
    answer = generate_answer(question)
    trace_id = mlflow.get_current_active_span().request_id
    return {"answer": answer, "trace_id": trace_id}

# 사용자 피드백을 trace_id와 연결
def record_feedback(trace_id: str, score: int, comment: str):
    mlflow.set_trace_tag(trace_id, "user_score", str(score))
    mlflow.set_trace_tag(trace_id, "user_comment", comment)
```

### 패턴 2: 고품질 응답을 평가 데이터로 수집

```sql
-- 사용자 만족도 높은 트레이스를 평가 데이터셋으로 추출
CREATE TABLE eval_dataset AS
SELECT
    t.inputs['question'] AS question,
    t.outputs['answer'] AS expected_answer,
    r.outputs AS retrieved_context,
    t.tags['user_score'] AS user_score
FROM system.mlflow.traces t
JOIN system.mlflow.trace_spans r
    ON t.trace_id = r.trace_id AND r.span_type = 'RETRIEVER'
WHERE t.tags['user_score'] >= '4'
    AND t.timestamp >= CURRENT_DATE() - INTERVAL 30 DAYS;
```

### 패턴 3: 오프라인 평가 자동화

```python
import mlflow

# 수집된 평가 데이터로 새 모델 버전 평가
eval_data = spark.table("eval_dataset").toPandas()

results = mlflow.evaluate(
    model="models:/my_rag_agent/staging",
    data=eval_data,
    targets="expected_answer",
    model_type="question-answering",
    evaluators="default",
    extra_metrics=[
        mlflow.metrics.genai.answer_relevance(),
        mlflow.metrics.genai.faithfulness(),
    ]
)
```

> 💡 **Flywheel 효과**: 프로덕션 트레이스 → 평가 데이터 수집 → 모델 평가 → 개선 → 재배포 → 더 나은 트레이스의 선순환을 구축하면, 시간이 지날수록 GenAI 앱의 품질이 자동으로 향상됩니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Trace** | GenAI 앱의 한 번의 실행 전체를 기록한 것입니다 |
| **Span** | Trace 내의 개별 단계(LLM 호출, 검색, Tool 실행 등)입니다 |
| **Span 트리** | Span은 부모-자식 관계로 트리 구조를 형성합니다 |
| **Autolog** | 한 줄로 자동 트레이싱을 활성화합니다. 8+ 프레임워크를 지원합니다 |
| **@mlflow.trace** | 데코레이터로 커스텀 함수에 수동 트레이싱을 추가합니다 |
| **자동+수동 혼합** | 프레임워크 호출은 자동, 비즈니스 로직은 수동으로 조합합니다 |
| **Unity Catalog 통합** | 트레이스를 SQL로 조회하고 대시보드에서 모니터링할 수 있습니다 |
| **평가 데이터 수집** | 프로덕션 트레이스에서 평가 데이터셋을 자동 구축할 수 있습니다 |

---

## 참고 링크

- [Databricks: MLflow Tracing](https://docs.databricks.com/aws/en/mlflow/mlflow-tracing.html)
- [MLflow: Tracing](https://mlflow.org/docs/latest/llms/tracing/index.html)
- [Databricks: Instrument agents](https://docs.databricks.com/aws/en/generative-ai/agent-framework/trace-agents.html)
