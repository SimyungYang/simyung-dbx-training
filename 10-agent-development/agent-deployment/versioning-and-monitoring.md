# 버전 관리와 모니터링

## 에이전트 버전 관리 및 업데이트

### 버전 업데이트 전략

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 새 버전 개발 (v4) | 에이전트의 새 버전을 개발합니다 |
| 2 | MLflow 로깅 + UC 등록 | 모델을 기록하고 등록합니다 |
| 3 | Review App 테스트 | Review App에서 테스터가 검증합니다 |
| 4 | 테스트 통과 여부 판단 | 통과하지 못하면 1단계로 돌아갑니다 |
| 5 | 엔드포인트 업데이트 | v3에서 v4로 업데이트합니다 |
| 6 | 모니터링 | Inference Tables로 성능을 모니터링합니다 |
| 7a | 이상 징후 발견 시 | 롤백 (v4 → v3)을 수행합니다 |
| 7b | 정상 운영 시 | champion 앨리어스를 v4로 이동합니다 |

### 엔드포인트 업데이트

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import ServedEntityInput, EndpointCoreConfigInput

w = WorkspaceClient()

# 새 버전으로 엔드포인트 업데이트 (무중단)
w.serving_endpoints.update_config(
    name="customer-support-agent",
    served_entities=[
        ServedEntityInput(
            entity_name="catalog.schema.customer_support_agent",
            entity_version="4",  # 새 버전
            workload_size="Small",
            scale_to_zero_enabled=True
        )
    ]
)
```

### 롤백

문제가 발생하면 이전 버전으로 빠르게 롤백할 수 있습니다.

```python
# 이전 버전으로 즉시 롤백
w.serving_endpoints.update_config(
    name="customer-support-agent",
    served_entities=[
        ServedEntityInput(
            entity_name="catalog.schema.customer_support_agent",
            entity_version="3",  # 이전 안정 버전으로 복귀
            workload_size="Small",
            scale_to_zero_enabled=True
        )
    ]
)
```

---

## 모니터링

### Inference Tables (추론 테이블)

배포된 에이전트의 모든 요청과 응답이 자동으로 Delta Table에 기록됩니다.

```sql
-- 최근 요청/응답 조회
SELECT
    request_id,
    timestamp,
    request:messages AS user_messages,
    response:choices[0]:message:content AS agent_response,
    status_code,
    execution_time_ms
FROM catalog.schema.agent_logs_payload
ORDER BY timestamp DESC
LIMIT 100;

-- 일별 사용 통계
SELECT
    DATE(timestamp) AS date,
    COUNT(*) AS total_requests,
    AVG(execution_time_ms) AS avg_latency_ms,
    COUNT(CASE WHEN status_code != 200 THEN 1 END) AS error_count,
    ROUND(COUNT(CASE WHEN status_code != 200 THEN 1 END) * 100.0 / COUNT(*), 2)
        AS error_rate_pct
FROM catalog.schema.agent_logs_payload
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

### MLflow Tracing

MLflow Tracing을 활성화하면 에이전트 내부의 각 단계(LLM 호출, 도구 실행, 검색 등)를 상세히 추적할 수 있습니다.

```python
import mlflow

# Tracing 자동 활성화
mlflow.langchain.autolog()

# 또는 수동으로 span 추가
@mlflow.trace
def process_query(query: str):
    with mlflow.start_span(name="retrieve_context") as span:
        context = vector_search(query)
        span.set_attribute("num_results", len(context))

    with mlflow.start_span(name="generate_response") as span:
        response = llm.invoke(query, context=context)
        span.set_attribute("token_count", response.usage.total_tokens)

    return response
```

### 모니터링 대시보드 구축

| 모니터링 지표 | 설명 | 임계값 예시 |
|-------------|------|-----------|
| **요청량** | 시간당/일당 요청 수 | 비정상적 급증 감지 |
| **응답 지연** | p50, p95, p99 응답 시간 | p95 > 5초 시 알림 |
| **오류율** | 4xx/5xx 응답 비율 | > 5% 시 알림 |
| **토큰 사용량** | 요청/응답당 평균 토큰 수 | 비용 관리 |
| **사용자 피드백** | 긍정/부정 피드백 비율 | 부정 > 30% 시 알림 |

---
