# Trace 조회와 프로덕션 모니터링

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

> 🆕 **MLflow Traces in Unity Catalog (Preview)**: 트레이스를 Unity Catalog에 저장하여 **SQL로 직접 조회** 할 수 있습니다. 무제한 용량으로 보존하고, 대시보드에서 모니터링할 수 있습니다.

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

Model Serving 엔드포인트에 배포된 에이전트는 **자동으로 모든 요청이 트레이싱** 됩니다. Inference Table과 결합하여 프로덕션 품질을 모니터링할 수 있습니다.

| 구성 요소 | 역할 | 연결 |
|-----------|------|------|
| **사용자 요청** | 요청을 전송합니다 | Model Serving Endpoint로 전달 |
| **Model Serving Endpoint** | 요청을 수신합니다 | AI 에이전트에 전달 |
| **AI 에이전트** | 요청을 처리합니다 | MLflow Trace + Inference Table에 기록 |
| **MLflow Trace** | 실행 흐름을 자동 기록합니다 | 모니터링 대시보드에 데이터 제공 |
| **Inference Table** | 입출력을 기록합니다 | 모니터링 대시보드에 데이터 제공 |
| ** 모니터링 대시보드** | 종합 모니터링을 제공합니다 | Trace + Inference Table 데이터를 시각화 |

---

## Trace 데이터 구조 심층 분석 — Span 트리

### Trace의 구조

하나의 Trace는 ** 트리(Tree) 구조의 Span 계층**으로 구성됩니다. 루트 Span은 전체 요청을 나타내고, 하위 Span은 각 처리 단계를 나타냅니다.

| Span | Type | Duration |
|------|------|----------|
| **Root: rag_agent** | AGENT | 1,350ms |
| - query_rewrite | CHAIN | 15ms |
|   - llm_rewrite | LLM | 12ms |
| - document_retrieval | RETRIEVER | 85ms |
|   - embedding | EMBEDDING | 25ms |
|   - vector_search | RETRIEVER | 55ms |
| - reranking | RERANKER | 30ms |
| - answer_generation | LLM | 1,200ms |
|   - response_parse | PARSER | 5ms |

> Trace ID: "abc-123" 

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
