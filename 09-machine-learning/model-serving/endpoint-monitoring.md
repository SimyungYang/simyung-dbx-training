# 엔드포인트 모니터링 (Endpoint Monitoring)

## 왜 모니터링이 중요한가요?

모델을 학습하고 배포하는 것은 ML 라이프사이클의 **시작** 에 불과합니다. 프로덕션 환경에서 모델은 다양한 이유로 성능이 저하될 수 있습니다.

- **데이터 드리프트(Data Drift)**: 실제 입력 데이터의 분포가 학습 데이터와 달라집니다
- **모델 성능 저하**: 시간이 지나면서 예측 정확도가 떨어집니다
- **인프라 장애**: 지연 시간 증가, 에러율 상승, 리소스 부족 등이 발생합니다
- **비용 폭증**: 예상치 못한 트래픽 증가로 비용이 급격히 올라갑니다

> 💡 **MLOps의 핵심 원칙**: "모니터링 없는 모델 배포는, 계기판 없이 비행기를 조종하는 것과 같습니다." 모니터링은 모델의 건강 상태를 지속적으로 확인하고, 문제를 조기에 발견하여 대응할 수 있게 해줍니다.

### 프로덕션 ML 장애 패턴과 SLA 관리

실제 프로덕션 환경에서 발생하는 대표적인 장애 패턴과 SLA(Service Level Agreement) 관리 방법을 이해하는 것이 중요합니다.

**대표적인 프로덕션 장애 패턴:**

| 장애 유형 | 발생 시기 | 탐지 난이도 | 영향 범위 |
|-----------|----------|------------|----------|
| **콜드 스타트 지연** | Scale-to-Zero 후 첫 요청 | 쉬움 (즉시 감지) | 개별 요청 |
| **데이터 스키마 변경** | 업스트림 파이프라인 배포 후 | 중간 (에러율 급증) | 전체 서비스 |
| **개념 드리프트(Concept Drift)** | 수주~수개월 경과 후 | 어려움 (점진적 성능 저하) | 비즈니스 전체 |
| **메모리 누수** | 오랜 운영 후 | 중간 (지연 시간 증가) | 서버 전체 |
| **토큰 폭발(Token Explosion)** | 사용자 프롬프트 변화 | 쉬움 (비용 급증) | 비용 및 지연 시간 |
| **모델 버전 불일치** | A/B 테스트 중 배포 오류 | 어려움 (부분적 오류) | 일부 사용자 |

**SLA 기준값 (일반 권장사항):**

| 메트릭 | 경고 임계값 | 위험 임계값 | SLA 예시 |
|--------|------------|------------|---------|
| **P99 지연시간** | > 500ms | > 1,000ms | 99%의 요청을 500ms 이내 처리 |
| **에러율(Error Rate)** | > 1% | > 5% | 99.9% 가용성 보장 |
| **처리량(Throughput)** | < 계획 대비 80% | < 계획 대비 50% | 최대 1,000 RPS 처리 |
| **드리프트 PSI** | > 0.1 | > 0.2 | 월간 재학습 트리거 |

> ⚠️ **중요**: SLA 임계값은 비즈니스 요구사항에 따라 조정해야 합니다. 추천 시스템과 실시간 사기 탐지는 요구되는 지연시간 SLA가 크게 다릅니다.

---

## 모니터링 계층 구조

효과적인 모니터링은 세 가지 계층으로 나누어 접근합니다. 인프라/서빙 → 모델 성능 → 비즈니스 메트릭 순으로 계층적으로 모니터링합니다.

| 계층 | 주요 메트릭 | 도구 |
|------|-----------|------|
| **1. 인프라/서빙** | QPS, 지연시간(P50/P95/P99), 에러율, GPU/CPU 사용률, 비용(DBU/토큰) | Serving UI, 시스템 테이블 |
| **2. 모델 성능** | 예측 정확도, 데이터 드리프트, 피처 분포 변화 | Inference Table + Lakehouse Monitoring |
| **3. 비즈니스** | 전환율, 매출 영향, 사용자 만족도 | 커스텀 대시보드, AI/BI |

---

## 핵심 메트릭 심화

### 지연 시간 Percentile (Latency Percentiles)

단순 평균(Average) 지연 시간은 이상치(Outlier)에 민감하여 실제 사용자 경험을 왜곡할 수 있습니다. 반드시 **Percentile 기반 메트릭** 을 사용해야 합니다.

| 메트릭 | 의미 | 활용 |
|--------|------|------|
| **P50 (중앙값, Median)** | 요청의 50%가 이 시간 이내에 처리됨 | "보통 사용자"의 경험 |
| **P95** | 요청의 95%가 이 시간 이내에 처리됨 | 대부분 사용자의 경험 |
| **P99** | 요청의 99%가 이 시간 이내에 처리됨 | 느린 사용자의 경험, SLA 기준으로 주로 사용 |
| **P99.9** | 요청의 99.9%가 이 시간 이내에 처리됨 | 매우 엄격한 SLA (금융, 의료 등) |

> 💡 **P99가 중요한 이유**: P50이 100ms이더라도 P99가 5,000ms이면, 100명 중 1명은 5초를 기다립니다. 높은 트래픽에서는 이 1%가 수백 명에 달할 수 있습니다.

```sql
-- system 테이블에서 Percentile 메트릭 계산
SELECT
    endpoint_name,
    DATE(request_time) AS day,
    COUNT(*) AS total_requests,
    ROUND(PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p50_ms,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p95_ms,
    ROUND(PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p99_ms,
    ROUND(PERCENTILE_CONT(0.999) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p999_ms,
    ROUND(MAX(execution_time_ms), 1) AS max_ms
FROM system.serving.served_model_requests
WHERE DATE(request_time) >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY endpoint_name, DATE(request_time)
ORDER BY day DESC, endpoint_name;
```

### 처리량 (Throughput)

**처리량(Throughput)** 은 단위 시간당 처리된 요청 수를 의미합니다. 엔드포인트가 부하를 감당할 수 있는지 판단하는 핵심 지표입니다.

| 메트릭 | 단위 | 설명 |
|--------|------|------|
| **RPS (Requests Per Second)** | 요청/초 | 초당 처리 요청 수, 일반 모델에 사용 |
| **TPS (Tokens Per Second)** | 토큰/초 | LLM 생성 속도, 스트리밍 경험에 직접 영향 |
| **QPS (Queries Per Second)** | 쿼리/초 | RPS와 동일 개념으로 혼용 |

```sql
-- 시간대별 평균 RPS(Requests Per Second) 계산
SELECT
    endpoint_name,
    DATE_TRUNC('hour', request_time) AS hour_bucket,
    COUNT(*) AS total_requests,
    -- 1시간(3600초) 기준 평균 RPS
    ROUND(COUNT(*) / 3600.0, 2) AS avg_rps
FROM system.serving.served_model_requests
WHERE DATE(request_time) >= CURRENT_DATE() - INTERVAL 1 DAYS
GROUP BY endpoint_name, DATE_TRUNC('hour', request_time)
ORDER BY hour_bucket DESC;
```

### 에러율 분류

에러는 HTTP 상태 코드로 분류하며, 원인에 따라 대응 방법이 다릅니다.

| 상태 코드 | 분류 | 일반적인 원인 | 대응 방법 |
|----------|------|-------------|----------|
| **400** | 클라이언트 오류 | 잘못된 요청 형식, 스키마 불일치 | 입력 유효성 검증 강화 |
| **429** | Rate Limiting | 클라이언트 요청 폭주 | Rate Limit 설정, 클라이언트 재시도 로직 |
| **500** | 서버 오류 | 모델 로딩 실패, 메모리 부족 | 리소스 확대, 모델 디버깅 |
| **503** | 서비스 불가 | 콜드 스타트, 과부하 | Provisioned Concurrency 설정 |
| **504** | 타임아웃 | 추론 시간 초과 | 타임아웃 임계값 조정, 모델 최적화 |

---

## Inference Tables (추론 테이블)

> 💡 **Inference Table** 은 Model Serving 엔드포인트의 모든 요청과 응답을 **자동으로 Delta 테이블에 기록** 하는 기능입니다. 모델 성능 모니터링, 디버깅, 규정 준수 감사에 활용됩니다.

### 기록되는 정보

| 항목 | 설명 |
|------|------|
| **요청 입력** | 모델에 전달된 입력 데이터 (피처, 프롬프트 등) |
| **응답 출력** | 모델의 예측 결과 (점수, 생성 텍스트 등) |
| **타임스탬프** | 요청/응답 시간 |
| **지연 시간** | 추론에 걸린 시간 (밀리초) |
| **상태 코드** | HTTP 상태 (200 성공, 4xx/5xx 오류) |
| **모델 버전** | 어떤 모델 버전이 응답했는지 (A/B 테스트 시 유용) |
| **토큰 사용량** | LLM 엔드포인트의 입력/출력 토큰 수 |
| **요청 메타데이터** | 클라이언트 ID, 요청 ID 등 추적 정보 |

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

### 샘플링 전략과 데이터 보존 정책

트래픽이 많은 엔드포인트에서 **모든 요청을 기록하면 스토리지 비용이 빠르게 증가** 합니다. 샘플링 전략과 보존 정책을 적절히 설정해야 합니다.

**샘플링 전략 비교:**

| 전략 | 방법 | 장점 | 단점 |
|------|------|------|------|
| **전수 기록 (100%)** | `sampling_fraction=1.0` | 완전한 감사 추적 | 스토리지 비용 높음 |
| **랜덤 샘플링** | `sampling_fraction=0.1` | 비용 절감, 통계적 대표성 | 희귀 이벤트 누락 가능 |
| **계층적 샘플링** | 에러는 100%, 정상은 10% | 오류 완전 캡처 + 비용 절감 | 구현 복잡성 |
| **시간 기반 샘플링** | 피크 시간대만 100% | 중요 시간대 완전 캡처 | 비피크 시간대 누락 |

```python
# 샘플링 비율 설정 예시 (엔드포인트 업데이트 시)
auto_capture = AutoCaptureConfigInput(
    enabled=True,
    catalog_name="ml_prod",
    schema_name="monitoring",
    table_name_prefix="fraud_model",
    # 10% 샘플링: 고트래픽 환경에서 비용 절감
    # 기본값은 100% (sampling_fraction 미설정 시)
)
```

**데이터 보존 정책 (Delta Table TTL):**

```sql
-- Inference Table에 데이터 보존 정책 설정 (90일)
ALTER TABLE ml_prod.monitoring.fraud_model_payload
SET TBLPROPERTIES (
  'delta.deletedFileRetentionDuration' = 'interval 7 days',
  'delta.logRetentionDuration' = 'interval 90 days'
);

-- 90일 이전 데이터 정기 삭제 (Lakeflow Jobs으로 예약)
DELETE FROM ml_prod.monitoring.fraud_model_payload
WHERE date < CURRENT_DATE() - INTERVAL 90 DAYS;

-- VACUUM으로 물리적 파일 정리
VACUUM ml_prod.monitoring.fraud_model_payload RETAIN 168 HOURS;
```

> 💡 **권장 보존 기간**: 규정 준수 요구사항이 없는 경우 30~90일을 권장합니다. GDPR/개인정보보호법 적용 환경에서는 법적 의무 보존 기간을 반드시 확인하십시오.

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

**드리프트 지표 해석 기준:**

| 지표 | 범위 | 해석 |
|------|------|------|
| **PSI** | < 0.1 | 안정 (No Change) |
| **PSI** | 0.1 ~ 0.2 | 소폭 변화 (Minor Shift), 모니터링 강화 |
| **PSI** | > 0.2 | 유의미한 드리프트 (Major Shift), 재학습 권장 |
| **KS 통계량** | < 0.05 | 두 분포 동일 (p-value 기준) |
| **Jensen-Shannon Divergence** | > 0.1 | 분포 차이 감지 |

### Lakehouse Monitoring 알림 설정

드리프트 감지 시 자동으로 알림을 발송하도록 설정할 수 있습니다.

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.catalog import (
    MonitorInferenceLog,
    MonitorInferenceLogProblemType,
    MonitorNotificationsConfig,
    MonitorNotificationsSpec,
)

w = WorkspaceClient()

# 알림이 포함된 모니터 생성
w.quality_monitors.create(
    table_name="ml_prod.monitoring.fraud_model_payload",
    inference_log=MonitorInferenceLog(
        problem_type=MonitorInferenceLogProblemType.PROBLEM_TYPE_CLASSIFICATION,
        prediction_col="prediction",
        label_col="actual_label",
        timestamp_col="timestamp_ms",
        model_id_col="model_version",
        granularities=["1 day"],
    ),
    output_schema_name="ml_prod.monitoring",
    # 드리프트 감지 시 이메일 알림
    notifications=MonitorNotificationsConfig(
        on_new_classification_tag_detected=MonitorNotificationsSpec(
            email_addresses=["ml-team@company.com"]
        )
    ),
    # 모니터 실행 실패 시 알림
    # on_failure=MonitorNotificationsSpec(email_addresses=["ml-team@company.com"])
)
```

```sql
-- 생성된 드리프트 메트릭 조회 (자동 생성 테이블)
SELECT
    window,
    column_name,
    drift_type,
    drift_value,
    threshold_value,
    -- 0.2 초과 시 알림 트리거
    CASE WHEN drift_value > 0.2 THEN 'ALERT' ELSE 'OK' END AS status
FROM ml_prod.monitoring.fraud_model_payload_drift_metrics
WHERE window_start >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY window_start DESC, drift_value DESC;
```

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
| **사용자 요청** | 입력 | 에이전트에 요청을 전송합니다 |
| **에이전트** | 오케스트레이션 | 요청을 분석하고 필요한 작업을 수행합니다 |
| **문서 검색** | Vector Search | 관련 문서를 검색합니다 |
| **LLM 호출** | Foundation Model | 답변을 생성합니다 |
| **도구 실행** | SQL, API 등 | 외부 도구를 호출합니다 |
| **최종 응답** | 출력 | 사용자에게 결과를 반환합니다 |

Tracing이 캡처하는 정보는 다음과 같습니다.

| 트레이스 항목 | 설명 |
|-------------|------|
| **Span 계층구조** | 에이전트의 각 단계(검색, LLM 호출, 도구 실행)를 트리 형태로 표시합니다 |
| **입출력** | 각 단계의 입력과 출력을 기록합니다 |
| **토큰 사용량** | LLM 호출별 입력/출력 토큰 수를 추적합니다 |
| **지연 시간** | 각 단계별 소요 시간을 밀리초 단위로 측정합니다 |
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
| **QPS (Queries Per Second)** | 초당 처리 요청 수 |
| **Latency (P50, P99)** | 응답 시간 분포 |
| **Error Rate** | 실패한 요청 비율 |
| **GPU Utilization** | GPU 엔드포인트의 리소스 활용도 |
| **Provisioned Concurrency** | 동시 처리 가능한 요청 수 |
| **Scale-to-Zero Events** | 0으로 축소/복원된 횟수 |
| **Token Throughput** | LLM 엔드포인트의 초당 토큰 처리량 |

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
| **Email** | SQL Alert에서 직접 지정 | 일일 리포트, 비긴급 알림 |
| **Slack** | Webhook URL 연동 | 팀 채널 실시간 알림 |
| **PagerDuty** | Webhook URL 연동 | 긴급 장애 대응 (On-call 연동) |
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

## 대시보드 구축 (System 테이블 기반)

### 종합 모니터링 대시보드 쿼리

아래 쿼리들을 Databricks SQL 대시보드에 위젯으로 추가하면, 종합 모니터링 대시보드를 구축할 수 있습니다.

```sql
-- 1. 일별 요청량 및 Percentile 지연 시간 (시계열 차트)
SELECT
    DATE(timestamp_ms / 1000) AS day,
    COUNT(*) AS total_requests,
    ROUND(PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p50_ms,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p95_ms,
    ROUND(PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY execution_time_ms), 1) AS p99_ms,
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

### 멀티 엔드포인트 비교 대시보드

여러 엔드포인트를 동시에 비교하는 운영 대시보드용 쿼리입니다.

```sql
-- 5. 엔드포인트별 SLA 준수율 비교 (가로 막대 차트)
-- SLA 기준: P99 < 1000ms, 에러율 < 1%
SELECT
    endpoint_name,
    DATE(request_time) AS day,
    COUNT(*) AS total_requests,
    ROUND(PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY execution_time_ms), 0) AS p99_ms,
    ROUND(SUM(CASE WHEN status_code >= 400 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS error_rate_pct,
    -- SLA 준수 여부
    CASE
        WHEN PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY execution_time_ms) <= 1000
             AND SUM(CASE WHEN status_code >= 400 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) <= 1.0
        THEN 'SLA 준수'
        ELSE 'SLA 위반'
    END AS sla_status
FROM system.serving.served_model_requests
WHERE DATE(request_time) >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY endpoint_name, DATE(request_time)
ORDER BY day DESC, endpoint_name;

-- 6. 비용 추이 (일별, 엔드포인트별)
SELECT
    usage_date,
    usage_metadata.endpoint_name,
    SUM(usage_quantity) AS total_dbus,
    ROUND(SUM(usage_quantity * list_price), 2) AS daily_cost_usd
FROM system.billing.usage
WHERE sku_name LIKE '%SERVING%'
    AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY usage_date, usage_metadata.endpoint_name
ORDER BY usage_date DESC;
```

---

## 모니터링 장단점과 트레이드오프

모니터링 체계를 설계할 때 비용과 가치 사이의 트레이드오프를 이해해야 합니다.

### 모니터링 수준별 비교

| 수준 | 구성 | 월 추가 비용 (예시) | 탐지 가능 문제 |
|------|------|------------------:|--------------|
| **기본** | Serving UI만 활용 | $0 | 인프라 장애, 지연시간 급증 |
| **중간** | + Inference Table (100%) | $50~200 | 스키마 오류, 입력 이상 |
| **고급** | + Lakehouse Monitoring | $200~500 | 데이터 드리프트, 모델 성능 저하 |
| **완전** | + Ground Truth 연동 | $500~2,000 | 실제 예측 정확도, 비즈니스 영향 |

> 💡 비용은 트래픽 규모, 데이터 크기, 클러스터 구성에 따라 크게 달라집니다. 실제 비용은 Databricks Cost Management에서 확인하십시오.

### Inference Table: 전수 기록 vs 샘플링

| 항목 | 전수 기록 (100%) | 샘플링 (10%) |
|------|-----------------|-------------|
| **비용** | 높음 | 낮음 (약 1/10) |
| **드리프트 감지 정확도** | 높음 | 중간 (통계적으로 충분) |
| **희귀 이벤트 캡처** | 모두 캡처 | 누락 가능 |
| **규정 준수 감사** | 완전한 감사 추적 | 불완전 |
| **적합한 환경** | 금융, 의료, 법규 준수 | 추천 시스템, 일반 ML |

### Lakehouse Monitoring 설정 트레이드오프

| 항목 | 빠른 갱신 주기 (1시간) | 느린 갱신 주기 (1일) |
|------|----------------------|---------------------|
| **드리프트 탐지 속도** | 빠름 (1시간 이내) | 느림 (최대 24시간) |
| **컴퓨팅 비용** | 높음 | 낮음 |
| **통계적 신뢰도** | 낮음 (샘플 수 부족 시) | 높음 |
| **적합한 환경** | 실시간 의사결정 시스템 | 배치 예측 시스템 |

---

## 베스트 프랙티스와 흔한 실수

### 베스트 프랙티스 (Best Practices)

**1. 모니터링을 배포 전에 설계하세요**

모델 배포 후 모니터링을 추가하면 초기 데이터가 유실됩니다. Inference Table은 엔드포인트 생성 시 함께 활성화하십시오.

```python
# 올바른 순서: 엔드포인트 생성과 Inference Table 활성화를 동시에
w.serving_endpoints.create(
    name="my-model",
    config=EndpointCoreConfigInput(
        served_entities=[...],
        auto_capture_config=AutoCaptureConfigInput(enabled=True, ...)  # 처음부터 활성화
    )
)
```

**2. 기준 분포(Baseline)를 반드시 저장하세요**

드리프트 감지는 기준 분포와의 비교로 이루어집니다. 모델 배포 시점의 입력 데이터 분포를 별도 테이블에 저장해두십시오.

```sql
-- 배포 시점 기준 분포 저장
CREATE TABLE ml_prod.monitoring.fraud_model_baseline AS
SELECT * FROM ml_prod.monitoring.fraud_model_payload
WHERE date BETWEEN '2024-01-01' AND '2024-01-31';  -- 첫 달 데이터
```

**3. 모델 버전을 요청 메타데이터에 포함하세요**

A/B 테스트나 Canary 배포 시, 모델 버전별 성능을 분리하여 분석할 수 있습니다.

```python
# 클라이언트에서 버전 정보를 메타데이터로 전달
import requests

response = requests.post(
    endpoint_url,
    headers={
        "Authorization": f"Bearer {token}",
        "X-Model-Version": "v3",        # 커스텀 메타데이터
        "X-Client-Id": "recommendation-service"
    },
    json={"inputs": features}
)
```

**4. P99 기준으로 SLA를 설정하세요**

평균 지연시간이 아닌 P99를 SLA 기준으로 사용해야 실제 사용자 경험을 보호할 수 있습니다.

**5. 비용 알림을 일별 예산과 연동하세요**

예상치 못한 트래픽 급증이나 프롬프트 길이 증가로 비용이 급등할 수 있습니다.

```sql
-- 일별 비용이 임계값 초과 시 알림 (Databricks SQL Alert로 등록)
SELECT
    usage_date,
    SUM(usage_quantity * list_price) AS daily_cost_usd
FROM system.billing.usage
WHERE sku_name LIKE '%SERVING%'
    AND usage_date = CURRENT_DATE() - INTERVAL 1 DAYS
GROUP BY usage_date
HAVING daily_cost_usd > 100.0;  -- $100 초과 시 알림
```

### 흔한 실수 (Common Mistakes)

**실수 1: Inference Table을 `date` 파티션 없이 쿼리**

```sql
-- 잘못된 예시: 전체 테이블 스캔으로 매우 느림
SELECT * FROM ml_prod.monitoring.fraud_model_payload
WHERE timestamp_ms > UNIX_TIMESTAMP(CURRENT_TIMESTAMP() - INTERVAL 1 DAYS) * 1000;

-- 올바른 예시: date 파티션 필터 추가
SELECT * FROM ml_prod.monitoring.fraud_model_payload
WHERE date >= CURRENT_DATE() - INTERVAL 1 DAYS  -- 파티션 프루닝
    AND timestamp_ms > UNIX_TIMESTAMP(CURRENT_TIMESTAMP() - INTERVAL 1 DAYS) * 1000;
```

**실수 2: Ground Truth 없이 드리프트만 모니터링**

데이터 드리프트(입력 분포 변화)만으로는 모델 성능 저하를 확인할 수 없습니다. 가능하면 Ground Truth 레이블을 수집하여 실제 정확도를 추적하십시오.

**실수 3: Scale-to-Zero 엔드포인트에 너무 짧은 타임아웃 설정**

Scale-to-Zero 엔드포인트는 콜드 스타트 시 10~30초가 소요될 수 있습니다. 클라이언트 타임아웃을 콜드 스타트 시간을 고려하여 설정하십시오.

```python
# 올바른 예시: 콜드 스타트를 고려한 충분한 타임아웃
response = requests.post(
    endpoint_url,
    json=payload,
    timeout=60  # 60초 타임아웃 (콜드 스타트 포함)
)
```

**실수 4: 모니터링 대시보드만 만들고 알림 미설정**

대시보드는 사람이 직접 확인해야 합니다. 반드시 임계값 기반 자동 알림을 함께 설정하십시오.

**실수 5: 너무 많은 알림으로 알림 피로(Alert Fatigue) 유발**

알림이 너무 자주 발생하면 담당자가 알림을 무시하게 됩니다. 알림 임계값을 비즈니스 영향 기준으로 설정하고, 긴급/비긴급 알림을 분리하십시오.

| 알림 유형 | 채널 | 임계값 기준 |
|----------|------|-----------|
| **P1 긴급** | PagerDuty (On-call) | 에러율 > 10%, 서비스 중단 |
| **P2 경고** | Slack #ml-alerts | 에러율 > 5%, P99 > 2초 |
| **P3 정보** | Email 일일 리포트 | 드리프트 감지, 비용 임계값 |

---

## 트러블슈팅 가이드

| 증상 | 가능한 원인 | 확인 방법 | 해결 방안 |
|------|-----------|----------|----------|
| **지연 시간 급증** | 모델 크기 증가, 입력 길이 증가 | P99 지연 시간 트렌드 확인 | 모델 최적화, 인스턴스 크기 증가 |
| **에러율 상승** | 입력 스키마 불일치, 메모리 부족 | 에러 로그 샘플 확인 | 입력 유효성 검증 추가, 리소스 확대 |
| **Scale-to-Zero 후 콜드 스타트** | 트래픽 불규칙 | 스케일링 이벤트 로그 확인 | 최소 프로비저닝 동시성 설정 |
| **토큰 비용 급증** | 프롬프트 길이 증가, 트래픽 급증 | 일별 토큰 사용량 확인 | 프롬프트 최적화, Rate Limiting |
| **드리프트 알림** | 입력 데이터 분포 변화 | Lakehouse Monitor 대시보드 확인 | 모델 재학습, 피처 파이프라인 점검 |
| **Inference Table 미기록** | Auto Capture 미활성화 | 엔드포인트 설정 확인 | `auto_capture_config` 활성화 |

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **Inference Table** | 모든 요청/응답을 Delta 테이블에 자동 기록합니다 |
| **Lakehouse Monitoring** | 데이터 드리프트와 모델 품질을 자동으로 감지합니다 |
| **MLflow Tracing** | LLM/에이전트의 실행 경로를 단계별로 추적합니다 |
| **시스템 테이블** | 모든 엔드포인트의 운영 메트릭을 중앙에서 조회합니다 |
| **비용 모니터링** | 토큰 사용량과 DBU 기반 비용을 추적합니다 |
| **알림** | SQL Alert + Webhook으로 실시간 알림을 받을 수 있습니다 |

**모니터링 설계 원칙 요약:**

1. **배포 전 설계** — Inference Table은 엔드포인트 생성 시 활성화합니다
2. **Percentile 기반 SLA** — P99를 기준으로 서비스 수준을 정의합니다
3. **계층적 알림** — 긴급/경고/정보로 구분하여 알림 피로를 방지합니다
4. **비용과 가치 균형** — 트래픽 규모에 맞는 샘플링 전략을 선택합니다
5. **Ground Truth 연동** — 가능하면 실제 레이블을 수집하여 성능을 검증합니다

---

## 참고 링크

- [Databricks: Inference tables](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
- [Databricks: Monitor serving endpoints](https://docs.databricks.com/aws/en/machine-learning/model-serving/monitor-diagnose-endpoints.html)
- [Databricks: Lakehouse Monitoring](https://docs.databricks.com/aws/en/lakehouse-monitoring/index.html)
- [Databricks: MLflow Tracing](https://docs.databricks.com/aws/en/mlflow/mlflow-tracing.html)
- [Databricks: System tables - Serving](https://docs.databricks.com/aws/en/admin/system-tables/serving.html)
- [Databricks: Billing usage system table](https://docs.databricks.com/aws/en/admin/system-tables/billing.html)
- [Databricks Blog: Monitoring ML models in production](https://www.databricks.com/blog/lakehouse-monitoring)
