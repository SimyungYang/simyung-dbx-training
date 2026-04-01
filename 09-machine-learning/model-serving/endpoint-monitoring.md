# 엔드포인트 모니터링 (Endpoint Monitoring)

## 왜 모니터링이 중요한가요?

모델을 학습하고 배포하는 것은 ML 라이프사이클의 **시작** 에 불과합니다. 프로덕션 환경에서 모델은 다양한 이유로 성능이 저하될 수 있습니다.

- **데이터 드리프트(Data Drift)**: 실제 입력 데이터의 분포가 학습 데이터와 달라집니다
- **모델 성능 저하**: 시간이 지나면서 예측 정확도가 떨어집니다
- **인프라 장애**: 지연 시간 증가, 에러율 상승, 리소스 부족 등이 발생합니다
- **비용 폭증**: 예상치 못한 트래픽 증가로 비용이 급격히 올라갑니다

> 💡 **MLOps의 핵심 원칙**: "모니터링 없는 모델 배포는, 계기판 없이 비행기를 조종하는 것과 같습니다." 모니터링은 모델의 건강 상태를 지속적으로 확인하고, 문제를 조기에 발견하여 대응할 수 있게 해줍니다.

---

## 모니터링 계층 구조

효과적인 모니터링은 세 가지 계층으로 나누어 접근합니다.

| 계층 | 메트릭 | 설명 |
|------|--------|------|
| **1. 인프라/서빙**| QPS / 지연시간 | 요청 처리량과 응답 속도입니다 |
|  | 에러율 | 실패한 요청의 비율입니다 |
|  | GPU/CPU 사용률 | 컴퓨팅 리소스 사용 현황입니다 |
|  | 비용 (DBU/토큰) | 서빙에 소요되는 비용입니다 |
| **2. 모델 성능**| 예측 정확도 | 모델의 예측 품질입니다 |
|  | 데이터 드리프트 | 입력 데이터 분포의 변화입니다 |
|  | 피처 분포 변화 | 개별 피처의 분포 변화입니다 |
| **3. 비즈니스 메트릭**| 전환율 변화 | 비즈니스 전환율에 미치는 영향입니다 |
|  | 매출 영향 | 매출에 미치는 영향입니다 |
|  | 사용자 만족도 | 사용자 만족도 변화입니다 |

> 인프라/서빙 → 모델 성능 → 비즈니스 메트릭 순으로 계층적으로 모니터링합니다.

| 계층 | 모니터링 대상 | 도구 |
|------|-------------|------|
| **1. 인프라/서빙**| QPS, 지연시간, 에러율, 리소스 사용률 | Serving UI, 시스템 테이블 |
| **2. 모델 성능**| 데이터 드리프트, 예측 분포 변화 | Inference Table + Lakehouse Monitoring |
| **3. 비즈니스**| 전환율, 매출, 사용자 행동 변화 | 커스텀 대시보드, AI/BI |

---

## Inference Tables (추론 테이블)

> 💡 **Inference Table** 은 Model Serving 엔드포인트의 모든 요청과 응답을 **자동으로 Delta 테이블에 기록** 하는 기능입니다. 모델 성능 모니터링, 디버깅, 규정 준수 감사에 활용됩니다.

### 기록되는 정보

| 항목 | 설명 |
|------|------|
| **요청 입력**| 모델에 전달된 입력 데이터 (피처, 프롬프트 등) |
| **응답 출력**| 모델의 예측 결과 (점수, 생성 텍스트 등) |
| **타임스탬프**| 요청/응답 시간 |
| **지연 시간**| 추론에 걸린 시간 (밀리초) |
| **상태 코드**| HTTP 상태 (200 성공, 4xx/5xx 오류) |
| **모델 버전**| 어떤 모델 버전이 응답했는지 (A/B 테스트 시 유용) |
| **토큰 사용량**| LLM 엔드포인트의 입력/출력 토큰 수 |
| **요청 메타데이터**| 클라이언트 ID, 요청 ID 등 추적 정보 |

### 자동 캡처 설정

```python
# 엔드포인트 생성 시 Inference Table 활성화
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    EndpointCoreConfigInput,
    ServedEntityInput,
    AutoCaptureConfigInput,
)

w = WorkspaceClient()

auto_capture = AutoCaptureConfigInput(
    enabled=True,
    catalog_name="ml_prod",
    schema_name="monitoring",
    table_name_prefix="fraud_model"
)

w.serving_endpoints.create(
    name="fraud-detector",
    config=EndpointCoreConfigInput(
        served_entities=[
            ServedEntityInput(
                entity_name="ml_prod.models.fraud_detector",
                entity_version="3",
                workload_size="Small",
                scale_to_zero_enabled=True,
            )
        ],
        auto_capture_config=auto_capture,
    ),
)
# → ml_prod.monitoring.fraud_model_payload 테이블이 자동 생성됩니다
```

> 💡 **Inference Table 이름 규칙**: `{catalog}.{schema}.{prefix}_payload` 형태로 자동 생성됩니다. 기존 엔드포인트에도 `PUT /api/2.0/serving-endpoints/{name}/config`로 활성화할 수 있습니다.

### 페이로드 구조

Inference Table에 기록되는 데이터의 스키마는 다음과 같습니다.

```sql
-- Inference Table 스키마 확인
DESCRIBE TABLE ml_prod.monitoring.fraud_model_payload;

-- 주요 컬럼:
-- date              : DATE        (파티션 키)
-- timestamp_ms      : BIGINT      (요청 시각, 밀리초)
-- status_code       : INT         (HTTP 상태 코드)
-- execution_time_ms : DOUBLE      (추론 소요 시간)
-- request           : STRING      (요청 JSON)
-- response          : STRING      (응답 JSON)
-- request_metadata  : MAP<STRING,STRING> (추적 메타데이터)
-- sampling_fraction : DOUBLE      (샘플링 비율)
```

> ⚠️ **주의**: Inference Table은 파티션 기반으로 데이터를 저장하므로, 쿼리 시 반드시 `date` 컬럼으로 필터링하여 성능을 확보하시기 바랍니다.

---

## Lakehouse Monitoring 연동

Inference Table의 데이터를 **Lakehouse Monitoring** 과 연동하면, 데이터 드리프트와 모델 품질을 체계적으로 모니터링할 수 있습니다.

### 데이터 드리프트 감지

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.catalog import MonitorInferenceLog, MonitorInferenceLogProblemType

w = WorkspaceClient()

# Inference Table에 Lakehouse Monitor 설정
w.quality_monitors.create(
    table_name="ml_prod.monitoring.fraud_model_payload",
    inference_log=MonitorInferenceLog(
        problem_type=MonitorInferenceLogProblemType.PROBLEM_TYPE_CLASSIFICATION,
        prediction_col="prediction",
        label_col="actual_label",       # Ground Truth (사후 레이블링 필요)
        timestamp_col="timestamp_ms",
        model_id_col="model_version",
        granularities=["1 day", "1 hour"],
    ),
    output_schema_name="ml_prod.monitoring",
)
```

모니터가 설정되면 자동으로 두 가지 분석 테이블이 생성됩니다.

| 생성 테이블 | 내용 |
|------------|------|
| `*_profile_metrics` | 각 컬럼의 통계 요약 (평균, 분산, 분포 등) |
| `*_drift_metrics` | 기준 기간 대비 드리프트 지표 (PSI, KL Divergence 등) |

> 💡 **PSI(Population Stability Index)**: 두 분포의 차이를 측정하는 지표입니다. PSI가 0.2를 초과하면 유의미한 드리프트로 판단합니다.

### 모델 품질 모니터링

Ground Truth 레이블이 확보되면 모델의 실제 성능을 추적할 수 있습니다.

```sql
-- Ground Truth 레이블을 Inference Table에 조인
-- (실제 결과가 나중에 확인되는 경우)
CREATE OR REPLACE TABLE ml_prod.monitoring.fraud_model_labeled AS
SELECT
    p.*,
    g.actual_label
FROM ml_prod.monitoring.fraud_model_payload p
LEFT JOIN ml_prod.gold.fraud_labels g
    ON p.request:customer_id = g.customer_id
    AND p.date = g.event_date;

-- 일별 모델 정확도 추적
SELECT
    date,
    COUNT(*) AS total_predictions,
    SUM(CASE WHEN prediction = actual_label THEN 1 ELSE 0 END) AS correct,
    ROUND(AVG(CASE WHEN prediction = actual_label THEN 1.0 ELSE 0.0 END) * 100, 2) AS accuracy_pct
FROM ml_prod.monitoring.fraud_model_labeled
WHERE actual_label IS NOT NULL
GROUP BY date
ORDER BY date DESC;
```

---

## MLflow Tracing (에이전트/LLM 서빙)

LLM 기반 에이전트를 서빙하는 경우, **MLflow Tracing** 을 통해 각 요청의 내부 실행 경로를 추적할 수 있습니다.

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **사용자 요청**| 입력 | 에이전트에 요청을 전송합니다 |
| **에이전트**| 오케스트레이션 | 요청을 분석하고 필요한 작업을 수행합니다 |
| **문서 검색**| Vector Search | 관련 문서를 검색합니다 |
| **LLM 호출**| Foundation Model | 답변을 생성합니다 |
| **도구 실행**| SQL, API 등 | 외부 도구를 호출합니다 |
| **최종 응답**| 출력 | 사용자에게 결과를 반환합니다 |

Tracing이 캡처하는 정보는 다음과 같습니다.

| 트레이스 항목 | 설명 |
|-------------|------|
| **Span 계층구조**| 에이전트의 각 단계(검색, LLM 호출, 도구 실행)를 트리 형태로 표시합니다 |
| **입출력**| 각 단계의 입력과 출력을 기록합니다 |
| **토큰 사용량**| LLM 호출별 입력/출력 토큰 수를 추적합니다 |
| **지연 시간**| 각 단계별 소요 시간을 밀리초 단위로 측정합니다 |
| **에러 정보** | 실패 시 에러 메시지와 스택 트레이스를 기록합니다 |

```python
# MLflow Tracing은 에이전트 서빙 시 자동 활성화됩니다
# Inference Table에서 트레이스 데이터를 조회할 수 있습니다

import json

# 트레이스 데이터 분석 쿼리 (SQL)
# SELECT
#     request_id,
#     trace:spans[0].name AS root_span,
#     trace:spans[0].attributes.total_tokens AS total_tokens,
#     execution_time_ms
# FROM ml_prod.monitoring.agent_payload
# WHERE date = CURRENT_DATE()
# ORDER BY execution_time_ms DESC
# LIMIT 10;
```

---

## 시스템 테이블 활용

Databricks 시스템 테이블(`system.serving`)에는 모든 서빙 엔드포인트의 운영 메트릭이 자동으로 기록됩니다.

```sql
-- 엔드포인트별 일일 요청량 및 에러율
SELECT
    endpoint_name,
    DATE(request_time) AS day,
    COUNT(*) AS total_requests,
    SUM(CASE WHEN status_code >= 400 THEN 1 ELSE 0 END) AS errors,
    ROUND(
        SUM(CASE WHEN status_code >= 400 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2
    ) AS error_rate_pct,
    ROUND(AVG(execution_time_ms), 1) AS avg_latency_ms,
    ROUND(PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p99_latency_ms
FROM system.serving.served_model_requests
WHERE DATE(request_time) >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY endpoint_name, DATE(request_time)
ORDER BY day DESC, endpoint_name;

-- 엔드포인트별 프로비저닝된 동시성 변화 추적
SELECT
    endpoint_name,
    change_time,
    scaled_entity_name,
    previous_scale,
    new_scale
FROM system.serving.endpoint_scaling_events
WHERE DATE(change_time) >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY change_time DESC;
```

> 💡 **시스템 테이블** 은 Unity Catalog의 `system` 카탈로그에 위치하며, 워크스페이스 관리자가 활성화해야 합니다. 추가 비용 없이 90일간의 데이터를 보존합니다.

---

## 비용 모니터링

### 토큰 사용량 추적 (LLM 엔드포인트)

```sql
-- LLM 엔드포인트 일별 토큰 사용량 및 추정 비용
SELECT
    DATE(timestamp_ms / 1000) AS day,
    COUNT(*) AS total_requests,
    SUM(request:usage.prompt_tokens) AS total_input_tokens,
    SUM(request:usage.completion_tokens) AS total_output_tokens,
    SUM(request:usage.total_tokens) AS total_tokens,
    -- 대략적인 비용 추정 (모델별로 단가가 다릅니다)
    ROUND(SUM(request:usage.total_tokens) / 1000000.0 * 2.0, 2) AS estimated_cost_usd
FROM ml_prod.monitoring.agent_payload
WHERE date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY DATE(timestamp_ms / 1000)
ORDER BY day DESC;
```

### DBU 기반 비용 추적

```sql
-- 서빙 엔드포인트의 DBU 사용량 (system.billing 테이블)
SELECT
    usage_date,
    workspace_id,
    sku_name,
    usage_metadata.endpoint_name,
    SUM(usage_quantity) AS total_dbus,
    SUM(usage_quantity * list_price) AS estimated_cost
FROM system.billing.usage
WHERE sku_name LIKE '%SERVING%'
    AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY usage_date, workspace_id, sku_name, usage_metadata.endpoint_name
ORDER BY usage_date DESC;
```

---

## Serving UI 메트릭

Model Serving 엔드포인트 페이지에서 다음 메트릭을 실시간으로 확인할 수 있습니다.

| 메트릭 | 설명 |
|--------|------|
| **QPS (Queries Per Second)**| 초당 처리 요청 수 |
| **Latency (P50, P99)**| 응답 시간 분포 |
| **Error Rate**| 실패한 요청 비율 |
| **GPU Utilization**| GPU 엔드포인트의 리소스 활용도 |
| **Provisioned Concurrency**| 동시 처리 가능한 요청 수 |
| **Scale-to-Zero Events**| 0으로 축소/복원된 횟수 |
| **Token Throughput**| LLM 엔드포인트의 초당 토큰 처리량 |

> 🆕 **Endpoint Telemetry (Beta)**: OpenTelemetry 기반의 관찰 가능성 기능이 추가되었습니다. OTLP(OpenTelemetry Protocol)로 외부 모니터링 시스템(Datadog, Grafana 등)과 연동할 수 있습니다.

---

## 알림 설정 (Alerting)

모니터링 메트릭에 기반한 알림을 설정하여, 문제가 발생하면 즉시 대응할 수 있습니다.

### Databricks SQL Alert 활용

```sql
-- 에러율이 5%를 초과하면 알림 트리거
-- (이 쿼리를 Databricks SQL Alert로 등록합니다)
SELECT
    endpoint_name,
    COUNT(*) AS total_requests,
    SUM(CASE WHEN status_code >= 400 THEN 1 ELSE 0 END) AS errors,
    ROUND(
        SUM(CASE WHEN status_code >= 400 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2
    ) AS error_rate_pct
FROM system.serving.served_model_requests
WHERE request_time >= CURRENT_TIMESTAMP() - INTERVAL 1 HOUR
GROUP BY endpoint_name
HAVING error_rate_pct > 5.0;
```

### 알림 채널 설정

| 채널 | 설정 방법 | 적합한 용도 |
|------|----------|-----------|
| **Email**| SQL Alert에서 직접 지정 | 일일 리포트, 비긴급 알림 |
| **Slack**| Webhook URL 연동 | 팀 채널 실시간 알림 |
| **PagerDuty**| Webhook URL 연동 | 긴급 장애 대응 (On-call 연동) |
| **Microsoft Teams** | Webhook URL 연동 | 팀 채널 실시간 알림 |

```python
# Slack Webhook을 통한 알림 예시 (Lakeflow Jobs의 Task로 실행)
import requests
import json

slack_webhook_url = "https://hooks.slack.com/services/T.../B.../..."

def send_alert(endpoint_name, error_rate, p99_latency):
    payload = {
        "text": f":rotating_light: *서빙 엔드포인트 알림*\n"
                f"- 엔드포인트: `{endpoint_name}`\n"
                f"- 에러율: {error_rate}%\n"
                f"- P99 지연시간: {p99_latency}ms"
    }
    requests.post(slack_webhook_url, json=payload)
```

---

## 실습: 종합 모니터링 대시보드 쿼리

아래 쿼리들을 Databricks SQL 대시보드에 위젯으로 추가하면, 종합 모니터링 대시보드를 구축할 수 있습니다.

```sql
-- 1. 일별 요청량 및 평균 지연 시간 (시계열 차트)
SELECT
    DATE(timestamp_ms / 1000) AS day,
    COUNT(*) AS total_requests,
    ROUND(AVG(execution_time_ms), 1) AS avg_latency_ms,
    ROUND(PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p99_latency_ms,
    SUM(CASE WHEN status_code != 200 THEN 1 ELSE 0 END) AS error_count,
    ROUND(SUM(CASE WHEN status_code != 200 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS error_rate_pct
FROM ml_prod.monitoring.fraud_model_payload
WHERE date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY DATE(timestamp_ms / 1000)
ORDER BY day DESC;

-- 2. LLM 엔드포인트: 토큰 사용량 추적 (시계열 차트)
SELECT
    DATE(timestamp_ms / 1000) AS day,
    SUM(request:usage.prompt_tokens) AS total_input_tokens,
    SUM(request:usage.completion_tokens) AS total_output_tokens,
    SUM(request:usage.total_tokens) AS total_tokens,
    ROUND(AVG(execution_time_ms), 0) AS avg_latency
FROM ml_prod.monitoring.agent_payload
WHERE date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY DATE(timestamp_ms / 1000)
ORDER BY day DESC;

-- 3. 에러 분석 (테이블 위젯)
SELECT
    status_code,
    COUNT(*) AS count,
    FIRST(response) AS sample_error
FROM ml_prod.monitoring.fraud_model_payload
WHERE status_code != 200
    AND date = CURRENT_DATE()
GROUP BY status_code;

-- 4. 시간대별 트래픽 패턴 (히트맵)
SELECT
    DAYOFWEEK(timestamp_ms / 1000) AS day_of_week,
    HOUR(timestamp_ms / 1000) AS hour_of_day,
    COUNT(*) AS request_count
FROM ml_prod.monitoring.fraud_model_payload
WHERE date >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY day_of_week, hour_of_day;
```

---

## 트러블슈팅 가이드

| 증상 | 가능한 원인 | 확인 방법 | 해결 방안 |
|------|-----------|----------|----------|
| **지연 시간 급증**| 모델 크기 증가, 입력 길이 증가 | P99 지연 시간 트렌드 확인 | 모델 최적화, 인스턴스 크기 증가 |
| **에러율 상승**| 입력 스키마 불일치, 메모리 부족 | 에러 로그 샘플 확인 | 입력 유효성 검증 추가, 리소스 확대 |
| **Scale-to-Zero 후 콜드 스타트**| 트래픽 불규칙 | 스케일링 이벤트 로그 확인 | 최소 프로비저닝 동시성 설정 |
| **토큰 비용 급증**| 프롬프트 길이 증가, 트래픽 급증 | 일별 토큰 사용량 확인 | 프롬프트 최적화, Rate Limiting |
| **드리프트 알림**| 입력 데이터 분포 변화 | Lakehouse Monitor 대시보드 확인 | 모델 재학습, 피처 파이프라인 점검 |
| **Inference Table 미기록**| Auto Capture 미활성화 | 엔드포인트 설정 확인 | `auto_capture_config` 활성화 |

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **Inference Table**| 모든 요청/응답을 Delta 테이블에 자동 기록합니다 |
| **Lakehouse Monitoring**| 데이터 드리프트와 모델 품질을 자동으로 감지합니다 |
| **MLflow Tracing**| LLM/에이전트의 실행 경로를 단계별로 추적합니다 |
| **시스템 테이블**| 모든 엔드포인트의 운영 메트릭을 중앙에서 조회합니다 |
| **비용 모니터링**| 토큰 사용량과 DBU 기반 비용을 추적합니다 |
| **알림** | SQL Alert + Webhook으로 실시간 알림을 받을 수 있습니다 |

---

## 참고 링크

- [Databricks: Inference tables](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
- [Databricks: Monitor serving endpoints](https://docs.databricks.com/aws/en/machine-learning/model-serving/monitor-diagnose-endpoints.html)
- [Databricks: Lakehouse Monitoring](https://docs.databricks.com/aws/en/lakehouse-monitoring/index.html)
- [Databricks: MLflow Tracing](https://docs.databricks.com/aws/en/mlflow/mlflow-tracing.html)
- [Databricks: System tables - Serving](https://docs.databricks.com/aws/en/admin/system-tables/serving.html)
- [Databricks: Billing usage system table](https://docs.databricks.com/aws/en/admin/system-tables/billing.html)
- [Databricks Blog: Monitoring ML models in production](https://www.databricks.com/blog/lakehouse-monitoring)
