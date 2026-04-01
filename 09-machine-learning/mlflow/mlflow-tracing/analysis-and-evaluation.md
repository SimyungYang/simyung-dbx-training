# 프로덕션 분석과 평가 데이터 수집

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

프로덕션 트레이스를 활용하여 **모델 평가 데이터셋을 자동으로 구축** 하는 패턴입니다. 이는 GenAI 앱의 지속적 개선에 핵심적인 역할을 합니다.

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
| **Trace**| GenAI 앱의 한 번의 실행 전체를 기록한 것입니다 |
| **Span**| Trace 내의 개별 단계(LLM 호출, 검색, Tool 실행 등)입니다 |
| **Span 트리**| Span은 부모-자식 관계로 트리 구조를 형성합니다 |
| **Autolog**| 한 줄로 자동 트레이싱을 활성화합니다. 8+ 프레임워크를 지원합니다 |
| **@mlflow.trace**| 데코레이터로 커스텀 함수에 수동 트레이싱을 추가합니다 |
| ** 자동+수동 혼합**| 프레임워크 호출은 자동, 비즈니스 로직은 수동으로 조합합니다 |
| **Unity Catalog 통합**| 트레이스를 SQL로 조회하고 대시보드에서 모니터링할 수 있습니다 |
| ** 평가 데이터 수집** | 프로덕션 트레이스에서 평가 데이터셋을 자동 구축할 수 있습니다 |

---

## 참고 링크

- [Databricks: MLflow Tracing](https://docs.databricks.com/aws/en/mlflow/mlflow-tracing.html)
- [MLflow: Tracing](https://mlflow.org/docs/latest/llms/tracing/index.html)
- [Databricks: Instrument agents](https://docs.databricks.com/aws/en/generative-ai/agent-framework/trace-agents.html)
