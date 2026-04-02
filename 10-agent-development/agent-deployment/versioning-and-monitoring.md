# 버전 관리와 모니터링 (Versioning & Monitoring)

---

## 1. 왜 버전 관리와 모니터링이 중요한가

### AI 에이전트의 비결정성 (Non-determinism)

전통적인 소프트웨어와 달리 AI 에이전트는 동일한 입력에도 다른 출력을 생성할 수 있습니다. 이 특성은 다음과 같은 운영 리스크를 만들어냅니다.

- **LLM 드리프트 (LLM Drift):** 파운데이션 모델 제공자(예: OpenAI, Anthropic)가 내부적으로 모델을 업데이트하면 동일한 API 호출도 다른 결과를 낳습니다.
- **데이터 드리프트 (Data Drift):** RAG 파이프라인의 지식 베이스(Knowledge Base)가 변경되면 검색 결과가 달라지고, 에이전트의 응답 품질이 변합니다.
- **프롬프트 민감성 (Prompt Sensitivity):** 프롬프트 템플릿의 사소한 변경이 응답 스타일이나 정확도에 큰 영향을 줄 수 있습니다.

### 프로덕션 장애의 대표 패턴

| 장애 유형 | 원인 | 영향 |
|-----------|------|------|
| **할루시네이션 급증** | 프롬프트 변경 또는 모델 업그레이드 | 잘못된 정보 제공, 신뢰도 손상 |
| **응답 지연 급증** | 도구 호출 루프, 컨텍스트 길이 초과 | 타임아웃, 사용자 이탈 |
| **오류율 증가** | 외부 API 장애, 스키마 변경 | 서비스 중단 |
| **비용 폭증** | 토큰 소비 이상, 무한 루프 | 예산 초과 |

이러한 리스크를 통제하려면 모든 변경을 버전으로 추적하고, 프로덕션 동작을 지속적으로 관찰해야 합니다.

---

## 2. 모델 버전 관리 — Unity Catalog Model Registry

### Unity Catalog에서의 버전 등록

Databricks Unity Catalog (UC)는 모델을 3단계 네임스페이스 (`catalog.schema.model_name`)로 관리합니다. 모델을 등록하면 버전 번호가 자동으로 부여됩니다.

```python
import mlflow

mlflow.set_registry_uri("databricks-uc")

with mlflow.start_run(run_name="customer-support-agent-v4"):
    model_info = mlflow.langchain.log_model(
        lc_model=agent_chain,
        artifact_path="agent",
        registered_model_name="catalog.schema.customer_support_agent",
        input_example={"messages": [{"role": "user", "content": "환불 방법을 알려주세요."}]},
    )
    print(f"등록된 버전: {model_info.registered_model_version}")
```

### Champion/Challenger 패턴

**앨리어스 (Alias)** 를 활용하면 버전 번호 대신 역할(Role) 기반으로 모델을 참조할 수 있습니다.

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# champion: 현재 프로덕션 버전
# challenger: 검증 중인 신규 버전
w.model_registry.set_registered_model_alias(
    full_name="catalog.schema.customer_support_agent",
    alias="champion",
    version_num=3
)
w.model_registry.set_registered_model_alias(
    full_name="catalog.schema.customer_support_agent",
    alias="challenger",
    version_num=4
)
```

### 자동 승격 기준 (Promotion Criteria)

Challenger 버전이 아래 조건을 모두 만족하면 Champion으로 자동 승격합니다.

| 메트릭 | 최소 기준 | 측정 기간 |
|--------|----------|----------|
| 정확도 (Accuracy) | Champion 대비 ≥ 95% | 24시간 |
| p95 Latency | ≤ 3,000ms | 24시간 |
| Error Rate | ≤ 1% | 24시간 |
| 사용자 긍정 피드백 | ≥ 70% | 24시간 |

```python
def auto_promote_if_ready(challenger_metrics: dict, champion_metrics: dict) -> bool:
    """Challenger가 Champion 기준을 초과하면 자동 승격"""
    conditions = [
        challenger_metrics["accuracy"] >= champion_metrics["accuracy"] * 0.95,
        challenger_metrics["p95_latency_ms"] <= 3000,
        challenger_metrics["error_rate"] <= 0.01,
        challenger_metrics["positive_feedback_rate"] >= 0.70,
    ]
    return all(conditions)
```

---

## 3. 배포 전략 (Deployment Strategies)

### 3-1. Canary 배포

신규 버전에 소량의 트래픽(예: 5–10%)만 먼저 흘려 리스크를 제한합니다.

```python
from databricks.sdk.service.serving import (
    ServedEntityInput, TrafficConfig, Route, EndpointCoreConfigInput
)

# Canary: v4에 10% 트래픽 할당
w.serving_endpoints.update_config(
    name="customer-support-agent",
    served_entities=[
        ServedEntityInput(
            entity_name="catalog.schema.customer_support_agent",
            entity_version="3",
            name="v3-stable",
            workload_size="Small",
            scale_to_zero_enabled=False
        ),
        ServedEntityInput(
            entity_name="catalog.schema.customer_support_agent",
            entity_version="4",
            name="v4-canary",
            workload_size="Small",
            scale_to_zero_enabled=False
        ),
    ],
    traffic_config=TrafficConfig(
        routes=[
            Route(served_model_name="v3-stable", traffic_percentage=90),
            Route(served_model_name="v4-canary",  traffic_percentage=10),
        ]
    )
)
```

### 3-2. Blue-Green 배포

두 개의 동일한 환경(Blue = 현재, Green = 신규)을 운영하고, 검증 후 즉시 전환합니다. 트래픽 전환이 순간적(0→100%)이므로 롤백도 빠릅니다.

```python
# Green 환경 검증 완료 후 100% 전환
w.serving_endpoints.update_config(
    name="customer-support-agent",
    served_entities=[
        ServedEntityInput(
            entity_name="catalog.schema.customer_support_agent",
            entity_version="4",  # Green → 전체 트래픽
            workload_size="Small",
            scale_to_zero_enabled=True
        )
    ]
)
```

### 3-3. A/B 테스트 (Traffic Splitting)

두 버전을 동시에 운영하며 비즈니스 메트릭(전환율, 해결률 등)을 비교합니다.

```python
# A/B 테스트: 50/50 분할
traffic_config = TrafficConfig(
    routes=[
        Route(served_model_name="variant-a", traffic_percentage=50),
        Route(served_model_name="variant-b", traffic_percentage=50),
    ]
)
```

> **참고:** A/B 테스트는 통계적 유의성(Statistical Significance)이 확보될 때까지 유지해야 합니다. 최소 수백 건 이상의 샘플이 필요합니다.

---

## 4. 모니터링 핵심 메트릭

### 인프라 메트릭 (Infrastructure Metrics)

| 메트릭 | 설명 | 권장 임계값 |
|--------|------|------------|
| **Latency (p50/p95/p99)** | 응답 시간 분포 | p95 > 5s → 알림 |
| **Throughput (RPS)** | 초당 요청 수 | 비정상 급증/급감 감지 |
| **Error Rate** | 4xx/5xx 비율 | > 5% → 알림, > 10% → 롤백 |
| **Concurrency** | 동시 처리 요청 수 | 스케일 아웃 기준 |

### LLM 특화 메트릭 (LLM-specific Metrics)

| 메트릭 | 설명 | 활용 |
|--------|------|------|
| **Token 사용량** | 입력/출력 토큰 수 | 비용 예측 및 이상 탐지 |
| **Context Length** | 컨텍스트 윈도우 활용률 | 최대 한계 근접 시 최적화 |
| **Tool Call Count** | 에이전트 루프 당 도구 호출 횟수 | 무한 루프 탐지 |
| **Retrieval Relevance** | RAG 검색 관련성 점수 | 지식 베이스 품질 측정 |

### 품질 메트릭 (Quality Metrics)

```sql
-- LLM Judge를 활용한 응답 품질 점수 집계
SELECT
    DATE(timestamp)                                             AS date,
    AVG(quality_score)                                         AS avg_quality_score,
    AVG(relevance_score)                                       AS avg_relevance_score,
    AVG(groundedness_score)                                    AS avg_groundedness_score,
    COUNT(CASE WHEN quality_score < 3 THEN 1 END)              AS low_quality_count
FROM catalog.schema.agent_quality_assessments
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

---

## 5. MLflow Tracing — 추론 과정 추적

### Tracing이 필요한 이유

프로덕션에서 에이전트가 잘못된 답변을 생성했을 때, 어느 단계에서 문제가 발생했는지 파악하려면 **분산 추적 (Distributed Tracing)** 이 필수입니다. MLflow Tracing은 LLM 호출, RAG 검색, 도구 실행 각 단계를 Span 단위로 기록합니다.

### 자동 추적 활성화

```python
import mlflow

# LangChain 자동 계측
mlflow.langchain.autolog()

# 또는 LlamaIndex 자동 계측
mlflow.llama_index.autolog()
```

### 커스텀 Span 추가

```python
import mlflow

@mlflow.trace(name="customer-support-agent", span_type="CHAIN")
def run_agent(user_query: str) -> str:

    # 검색 단계 추적
    with mlflow.start_span(name="retrieve_context", span_type="RETRIEVER") as span:
        context_docs = vector_store.similarity_search(user_query, k=5)
        span.set_attribute("num_retrieved_docs", len(context_docs))
        span.set_attribute("query", user_query)

    # LLM 호출 단계 추적
    with mlflow.start_span(name="generate_response", span_type="LLM") as span:
        response = llm.invoke(
            messages=[{"role": "user", "content": user_query}],
            context=context_docs
        )
        span.set_attribute("input_tokens",  response.usage.prompt_tokens)
        span.set_attribute("output_tokens", response.usage.completion_tokens)
        span.set_attribute("total_cost_usd", calculate_cost(response.usage))

    return response.content
```

### 프로덕션 Trace 분석

```python
import mlflow

# 최근 실패한 Trace 조회
failed_traces = mlflow.search_traces(
    filter_string="status = 'ERROR'",
    max_results=50,
    order_by=["timestamp_ms DESC"]
)

# 특정 Request ID로 Trace 조회 (Inference Table과 연계)
trace = mlflow.get_trace(request_id="tr-abc123")
print(trace.to_json(pretty=True))
```

---

## 6. Inference Table — 자동 로깅과 활용

### 활성화 방법

Model Serving 엔드포인트 생성 시 `auto_capture_config`를 설정하면 모든 요청/응답이 Delta Table에 자동 저장됩니다.

```python
from databricks.sdk.service.serving import AutoCaptureConfigInput

w.serving_endpoints.create(
    name="customer-support-agent",
    config=EndpointCoreConfigInput(
        served_entities=[...],
        auto_capture_config=AutoCaptureConfigInput(
            catalog_name="catalog",
            schema_name="schema",
            table_name_prefix="agent_logs",
            enabled=True
        )
    )
)
```

### Inference Table 주요 스키마

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `request_id` | STRING | 고유 요청 ID (Trace와 연계 키) |
| `timestamp` | TIMESTAMP | 요청 수신 시각 |
| `status_code` | INT | HTTP 상태 코드 |
| `execution_time_ms` | BIGINT | 총 처리 시간 |
| `request` | VARIANT | 원본 요청 JSON |
| `response` | VARIANT | 원본 응답 JSON |
| `sampling_fraction` | DOUBLE | 샘플링 비율 |
| `client_request_id` | STRING | 클라이언트 제공 요청 ID |

### 실전 활용 쿼리

```sql
-- 최근 1시간 주요 지표 요약
SELECT
    COUNT(*)                                                            AS total_requests,
    ROUND(AVG(execution_time_ms), 0)                                   AS avg_latency_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY execution_time_ms)    AS p95_latency_ms,
    ROUND(SUM(CASE WHEN status_code != 200 THEN 1 ELSE 0 END)
          * 100.0 / COUNT(*), 2)                                       AS error_rate_pct
FROM catalog.schema.agent_logs_payload
WHERE timestamp >= CURRENT_TIMESTAMP() - INTERVAL 1 HOUR;

-- 토큰 사용량 및 비용 추정 (일별)
SELECT
    DATE(timestamp)                                                     AS date,
    SUM(response:usage:prompt_tokens)                                  AS total_input_tokens,
    SUM(response:usage:completion_tokens)                              AS total_output_tokens,
    -- Claude 3.5 Sonnet 기준 예시 단가 적용
    ROUND(SUM(response:usage:prompt_tokens)    / 1e6 * 3.0 +
          SUM(response:usage:completion_tokens) / 1e6 * 15.0, 2)      AS estimated_cost_usd
FROM catalog.schema.agent_logs_payload
WHERE status_code = 200
GROUP BY DATE(timestamp)
ORDER BY date DESC;

-- 드리프트 감지: 응답 길이 분포 변화
SELECT
    DATE_TRUNC('hour', timestamp)                                       AS hour,
    AVG(LENGTH(response:choices[0]:message:content::STRING))           AS avg_response_length,
    STDDEV(LENGTH(response:choices[0]:message:content::STRING))        AS stddev_response_length
FROM catalog.schema.agent_logs_payload
GROUP BY DATE_TRUNC('hour', timestamp)
ORDER BY hour DESC;
```

### 피드백 루프 (Feedback Loop) 구성

Inference Table 데이터를 활용해 사용자 피드백을 수집하고 재훈련 데이터셋을 구축합니다.

```python
# 부정 피드백이 달린 요청을 재훈련 후보 데이터셋으로 큐레이션
negative_samples = spark.sql("""
    SELECT
        logs.request_id,
        logs.request:messages                        AS messages,
        logs.response:choices[0]:message:content     AS response,
        feedback.rating,
        feedback.comment
    FROM catalog.schema.agent_logs_payload logs
    JOIN catalog.schema.user_feedback feedback
      ON logs.request_id = feedback.request_id
    WHERE feedback.rating <= 2
      AND logs.timestamp >= CURRENT_TIMESTAMP() - INTERVAL 7 DAY
""")
negative_samples.write.mode("append").saveAsTable("catalog.schema.retraining_candidates")
```

---

## 7. 알림과 자동 대응 (Alerting & Auto-response)

### Lakehouse Monitoring 알림 설정

Unity Catalog의 **Lakehouse Monitoring** 기능으로 메트릭이 임계값을 초과하면 자동 알림을 발송합니다.

```python
from databricks.sdk.service.catalog import MonitorInferenceLog, MonitorCronSchedule

w.quality_monitors.create(
    table_name="catalog.schema.agent_logs_payload",
    inference_log=MonitorInferenceLog(
        granularities=["1 hour", "1 day"],
        timestamp_col="timestamp",
        model_id_col="served_entity_name",
        prediction_col="response",
        problem_type="question_answering"
    ),
    schedule=MonitorCronSchedule(
        quartz_cron_expression="0 0 * * * ?",  # 매시간
        timezone_id="Asia/Seoul"
    ),
    notifications={
        "on_failure": {"email_addresses": ["ml-ops@company.com"]},
        "on_new_classification_tag_detected": {"email_addresses": ["ml-ops@company.com"]}
    },
    output_schema_name="catalog.schema"
)
```

### 자동 롤백 파이프라인

Databricks Jobs를 활용해 메트릭 이상 감지 시 자동 롤백을 구현합니다.

```python
def check_and_rollback(endpoint_name: str, stable_version: str, threshold: dict):
    """메트릭 임계값 초과 시 자동 롤백"""

    # 최근 15분 메트릭 조회
    metrics = spark.sql("""
        SELECT
            COUNT(*) AS total,
            SUM(CASE WHEN status_code != 200 THEN 1 ELSE 0 END)
                * 100.0 / COUNT(*) AS error_rate,
            PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY execution_time_ms)
                AS p95_latency
        FROM catalog.schema.agent_logs_payload
        WHERE timestamp >= CURRENT_TIMESTAMP() - INTERVAL 15 MINUTE
    """).first()

    needs_rollback = (
        metrics["error_rate"] > threshold["max_error_rate"] or
        metrics["p95_latency"] > threshold["max_p95_latency_ms"]
    )

    if needs_rollback:
        print(f"임계값 초과 — 롤백 실행: 현재 버전 → v{stable_version}")
        w.serving_endpoints.update_config(
            name=endpoint_name,
            served_entities=[
                ServedEntityInput(
                    entity_name="catalog.schema.customer_support_agent",
                    entity_version=stable_version,
                    workload_size="Small",
                    scale_to_zero_enabled=True
                )
            ]
        )
        notify_team(
            f"[자동 롤백] {endpoint_name} → v{stable_version} "
            f"(error_rate={metrics['error_rate']:.1f}%)"
        )

# 임계값 설정 및 실행
THRESHOLDS = {"max_error_rate": 10.0, "max_p95_latency_ms": 8000}
check_and_rollback("customer-support-agent", stable_version="3", threshold=THRESHOLDS)
```

---

## 8. 전체 버전 업데이트 워크플로

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 개발 (v4) | 에이전트 신규 버전 개발 |
| 2 | MLflow 로깅 + UC 등록 | `challenger` 앨리어스 부여 |
| 3 | Review App 테스트 | 내부 테스터 검증 (오프라인 평가) |
| 4 | 기준 미충족 | 1단계로 돌아가 재개발 |
| 5 | Canary 배포 (10%) | 소량 트래픽으로 실서비스 검증 |
| 6 | 모니터링 (24h) | Inference Table + MLflow Tracing 분석 |
| 7a | 이상 감지 | 자동 롤백 (v4 → v3) + 알림 발송 |
| 7b | 정상 확인 | 트래픽 100% 전환, `champion` 앨리어스 이동 |

---

## 9. 베스트 프랙티스와 흔한 실수

### 베스트 프랙티스

- **모든 실험을 MLflow Run으로 추적합니다.** 프롬프트, 모델 버전, 하이퍼파라미터를 빠짐없이 기록하면 나중에 어떤 조합이 왜 좋았는지 재현할 수 있습니다.
- **앨리어스(Alias)를 사용해 버전 번호 의존을 제거합니다.** 코드에 버전 번호를 하드코딩하면 배포 시마다 코드를 변경해야 합니다.
- **Canary → Full 롤아웃 단계를 반드시 거칩니다.** 아무리 오프라인 평가가 좋아도 실사용자 트래픽에서 예상치 못한 문제가 발생할 수 있습니다.
- **Inference Table 보존 기간을 정책으로 명시합니다.** 기본적으로 무제한 누적되므로 TTL(Time-to-Live) 정책을 설정해 스토리지 비용을 관리합니다.
- **품질 메트릭과 인프라 메트릭을 분리해 모니터링합니다.** Error Rate가 낮아도 응답 품질이 저하될 수 있습니다.

### 흔한 실수

| 실수 | 결과 | 해결책 |
|------|------|--------|
| 버전 번호를 코드에 하드코딩 | 배포마다 코드 변경 필요 | 앨리어스(champion/challenger) 사용 |
| Canary 없이 즉시 100% 전환 | 전체 서비스 장애 위험 | 항상 점진적 롤아웃 적용 |
| 오프라인 평가만으로 배포 결정 | 실사용 품질 저하 발생 | A/B 테스트 + 온라인 평가 병행 |
| 롤백 절차 미수립 | 장애 시 복구 시간 증가 | 자동 롤백 파이프라인 사전 구축 |
| 토큰 비용 모니터링 누락 | 월말 예산 초과 | 일별 비용 알림 설정 |
| Tracing 비활성화 상태로 운영 | 장애 원인 파악 불가 | `autolog()`를 기본값으로 설정 |

---

## 참고 링크 (References)

- [MLflow Model Registry — Unity Catalog](https://docs.databricks.com/en/mlflow/model-registry-uc.html)
- [MLflow Tracing 개요](https://mlflow.org/docs/latest/llms/tracing/index.html)
- [Databricks Model Serving — Traffic Splitting](https://docs.databricks.com/en/machine-learning/model-serving/traffic-splitting.html)
- [Inference Tables 개요](https://docs.databricks.com/en/machine-learning/model-serving/inference-tables.html)
- [Lakehouse Monitoring — LLM 모니터링](https://docs.databricks.com/en/lakehouse-monitoring/monitor-llm-endpoints.html)
- [Databricks SDK — serving_endpoints](https://databricks-sdk-py.readthedocs.io/en/latest/workspace/serving/serving_endpoints.html)
