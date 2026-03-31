# Lakehouse Monitoring — 데이터 및 모델 품질 모니터링

## Lakehouse Monitoring이란?

> 💡 **Lakehouse Monitoring**은 Databricks의 **데이터 품질 및 모델 성능 모니터링 서비스**입니다. Delta 테이블에 모니터를 연결하면, 데이터의 통계 프로필, 분포 변화(드리프트), 이상치 등을 자동으로 추적하고 대시보드로 시각화합니다.

### 왜 모니터링이 필요한가요?

데이터 파이프라인과 ML 모델은 배포 후에도 **지속적으로 품질이 변할 수 있습니다**. 다음과 같은 문제를 사전에 감지하지 못하면 비즈니스에 큰 영향을 미칩니다.

| 문제 유형 | 예시 | 영향 |
|-----------|------|------|
| **데이터 드리프트** | 고객 연령 분포가 갑자기 변함 | 모델 예측 정확도 하락 |
| **데이터 품질 저하** | null 비율 급증, 이상 값 유입 | 잘못된 비즈니스 의사결정 |
| **컨셉 드리프트** | 계절 변화로 구매 패턴 변경 | 추천 모델 성능 저하 |
| **스키마 변경** | 업스트림 테이블 컬럼 변경 | 파이프라인 실패 |

> 💡 **비유**: Lakehouse Monitoring은 데이터 파이프라인의 "건강 검진"입니다. 정기적으로 데이터의 건강 상태를 검사하여, 문제가 심각해지기 전에 미리 발견합니다.

---

## 모니터 유형

Lakehouse Monitoring은 세 가지 유형의 모니터를 지원합니다.

| 모니터 유형 | 설명 | 사용 시나리오 |
|------------|------|-------------|
| **Snapshot** | 테이블의 현재 상태를 주기적으로 프로파일링합니다 | 정적 테이블, 차원 테이블 |
| **TimeSeries** | 시간 컬럼 기준으로 데이터 변화를 추적합니다 | 이벤트 로그, 거래 데이터 |
| **InferenceLog** | ML 모델의 입력/출력/레이블을 모니터링합니다 | 모델 서빙 추론 테이블 |

---

## 모니터 생성

### Python API로 모니터 생성

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.catalog import MonitorTimeSeries

w = WorkspaceClient()

# TimeSeries 모니터 생성
monitor = w.quality_monitors.create(
    table_name="catalog.schema.sales_data",
    assets_dir="/Shared/monitoring/sales_data",
    output_schema_name="catalog.schema",
    time_series=MonitorTimeSeries(
        timestamp_col="order_date",
        granularities=["1 day", "1 week"]
    ),
    schedule={"quartz_cron_expression": "0 0 8 * * ?",  # 매일 오전 8시
              "timezone_id": "Asia/Seoul"},
    slicing_exprs=["region", "product_category"]  # 슬라이싱 기준
)
```

### SQL로 모니터 생성

```sql
-- TimeSeries 모니터 생성
CREATE OR REFRESH MONITOR catalog.schema.sales_data
TIMETABLE CRON '0 0 8 * * ?'
TIMEZONE 'Asia/Seoul'
USING TIMESERIES (
    TIMESTAMP_COL order_date,
    GRANULARITIES ('1 day', '1 week')
)
ASSETS DIR '/Shared/monitoring/sales_data'
OUTPUT SCHEMA catalog.schema
SLICING_EXPRS (region, product_category);
```

### InferenceLog 모니터 (ML 모델용)

```python
from databricks.sdk.service.catalog import MonitorInferenceLog

monitor = w.quality_monitors.create(
    table_name="catalog.schema.model_inference_table",
    assets_dir="/Shared/monitoring/model_v1",
    output_schema_name="catalog.schema",
    inference_log=MonitorInferenceLog(
        problem_type="PROBLEM_TYPE_CLASSIFICATION",
        prediction_col="prediction",
        label_col="actual_label",       # Ground Truth (있는 경우)
        timestamp_col="inference_time",
        model_id_col="model_version",
        granularities=["1 hour", "1 day"]
    )
)
```

---

## 프로필 메트릭

모니터가 실행되면 **프로필 메트릭 테이블**과 **드리프트 메트릭 테이블** 두 개가 자동 생성됩니다.

### 프로필 메트릭 테이블

각 윈도우(시간 단위)별로 다음 통계를 자동으로 계산합니다.

| 메트릭 | 설명 | 수치형 | 범주형 |
|--------|------|--------|--------|
| `count` | 행 수 | O | O |
| `null_count` / `null_percentage` | null 값 개수 및 비율 | O | O |
| `distinct_count` | 고유 값 수 | O | O |
| `mean` | 평균 | O | - |
| `stddev` | 표준편차 | O | - |
| `min` / `max` | 최솟값 / 최댓값 | O | - |
| `quantiles` | 사분위수 (25%, 50%, 75%) | O | - |
| `frequent_items` | 빈출 값 목록 | - | O |
| `percent_distinct` | 고유 값 비율 | O | O |

```sql
-- 프로필 메트릭 조회
SELECT window, column_name, count, null_percentage, mean, stddev
FROM catalog.schema.sales_data_profile_metrics
WHERE granularity = '1 day'
ORDER BY window DESC;
```

### 드리프트 메트릭 테이블

현재 윈도우와 **기준선(Baseline)** 또는 **이전 윈도우**를 비교하여 변화를 수치화합니다.

| 드리프트 메트릭 | 설명 | 적용 대상 |
|---------------|------|----------|
| **KS 검정 (Kolmogorov-Smirnov)** | 두 분포의 최대 차이를 측정합니다 | 수치형 |
| **카이제곱 검정 (Chi-Square)** | 범주형 분포의 변화를 측정합니다 | 범주형 |
| **Jensen-Shannon Divergence** | 두 확률 분포의 차이를 측정합니다 | 수치형/범주형 |
| **PSI (Population Stability Index)** | 모집단 안정성을 측정합니다 | 수치형/범주형 |

```sql
-- 드리프트 메트릭 조회
SELECT window, column_name, drift_type,
       ks_test.statistic AS ks_statistic,
       ks_test.pvalue AS ks_pvalue
FROM catalog.schema.sales_data_drift_metrics
WHERE ks_test.pvalue < 0.05  -- 통계적으로 유의미한 드리프트
ORDER BY window DESC;
```

---

## 커스텀 메트릭 정의

기본 메트릭 외에 **비즈니스 로직에 맞는 커스텀 메트릭**을 정의할 수 있습니다.

```python
from databricks.sdk.service.catalog import MonitorMetric, MonitorMetricType

# 커스텀 메트릭 정의
custom_metrics = [
    MonitorMetric(
        type=MonitorMetricType.CUSTOM_METRIC_TYPE_AGGREGATE,
        name="high_value_ratio",
        input_columns=["order_amount"],
        definition="avg(CASE WHEN order_amount > 100000 THEN 1 ELSE 0 END)",
        output_data_type="DOUBLE"
    ),
    MonitorMetric(
        type=MonitorMetricType.CUSTOM_METRIC_TYPE_AGGREGATE,
        name="weekend_order_ratio",
        input_columns=["order_date"],
        definition="avg(CASE WHEN dayofweek(order_date) IN (1, 7) THEN 1 ELSE 0 END)",
        output_data_type="DOUBLE"
    )
]

# 모니터에 커스텀 메트릭 추가
monitor = w.quality_monitors.create(
    table_name="catalog.schema.sales_data",
    assets_dir="/Shared/monitoring/sales_custom",
    output_schema_name="catalog.schema",
    time_series=MonitorTimeSeries(
        timestamp_col="order_date",
        granularities=["1 day"]
    ),
    custom_metrics=custom_metrics
)
```

---

## 기준선 (Baseline) 설정

드리프트를 감지하려면 **"정상 상태"의 기준**이 필요합니다. 별도의 기준선 테이블을 지정할 수 있습니다.

```python
monitor = w.quality_monitors.create(
    table_name="catalog.schema.sales_data",
    assets_dir="/Shared/monitoring/sales_baseline",
    output_schema_name="catalog.schema",
    baseline_table_name="catalog.schema.sales_data_2024",  # 기준선 테이블
    time_series=MonitorTimeSeries(
        timestamp_col="order_date",
        granularities=["1 day"]
    )
)
```

| 기준선 유형 | 설명 |
|------------|------|
| **기준선 테이블 지정** | 과거 안정 기간의 데이터를 별도 테이블로 제공합니다 |
| **이전 윈도우 비교** | 기준선을 지정하지 않으면 이전 시간 윈도우와 비교합니다 |

---

## 대시보드 자동 생성

모니터를 생성하면 `assets_dir`에 **Lakeview 대시보드가 자동으로 생성**됩니다. 이 대시보드에는 다음 정보가 포함됩니다.

| 대시보드 섹션 | 내용 |
|-------------|------|
| **데이터 개요** | 행 수 추이, 테이블 크기 변화 |
| **컬럼별 프로필** | 각 컬럼의 분포, 통계 변화 |
| **드리프트 지표** | KS/카이제곱 검정 결과, 드리프트 점수 |
| **null 비율 추이** | 시간에 따른 null 비율 변화 |
| **모델 성능** (InferenceLog) | 정확도, 정밀도, 재현율 추이 |

---

## 알림 설정

모니터링 메트릭을 기반으로 **Databricks SQL Alert**를 설정하여 이상 징후를 자동으로 알릴 수 있습니다.

```sql
-- 알림 조건 쿼리 예시: null 비율이 5%를 초과하면 알림
SELECT
    window.start AS check_time,
    column_name,
    null_percentage
FROM catalog.schema.sales_data_profile_metrics
WHERE granularity = '1 day'
  AND null_percentage > 0.05
  AND window.start >= current_date() - INTERVAL 1 DAY;
```

알림 설정 방법:

1. 위 쿼리를 **SQL Editor**에서 저장합니다
2. **Alerts** 메뉴에서 해당 쿼리를 기반으로 알림을 생성합니다
3. 결과가 비어 있지 않으면 **이메일 또는 Slack으로 알림**을 발송합니다

---

## ML 모델 모니터링 워크플로

ML 모델 배포 후 성능 변화를 추적하는 전체 워크플로는 다음과 같습니다.

| 단계 | 작업 | 도구 |
|------|------|------|
| 1 | 모델 서빙 엔드포인트에서 Inference Table 활성화 | Model Serving |
| 2 | Inference Table에 InferenceLog 모니터 생성 | Lakehouse Monitoring |
| 3 | Ground Truth 레이블을 Inference Table에 조인 | Lakeflow Jobs |
| 4 | 자동 생성된 대시보드에서 성능 추적 | Lakeview Dashboard |
| 5 | 드리프트/성능 저하 시 알림 발송 | SQL Alerts |
| 6 | 재학습 파이프라인 트리거 | Lakeflow Jobs |

---

## 모범 사례

| 항목 | 권장 사항 |
|------|----------|
| **적절한 윈도우 크기** | 데이터 빈도에 맞는 granularity를 선택합니다 (실시간 → 1 hour, 일간 → 1 day) |
| **슬라이싱 활용** | 지역, 카테고리 등으로 슬라이싱하면 세부 드리프트를 발견할 수 있습니다 |
| **기준선 설정** | 안정된 과거 데이터를 기준선으로 지정하여 드리프트 판단 정확도를 높입니다 |
| **커스텀 메트릭** | 비즈니스에 중요한 지표는 커스텀 메트릭으로 추가합니다 |
| **알림 연동** | 드리프트 감지 시 자동 알림을 설정하여 빠르게 대응합니다 |
| **비용 관리** | 불필요하게 높은 빈도의 스케줄은 비용을 증가시킵니다 |

---

## 정리

| 개념 | 핵심 내용 |
|------|----------|
| Lakehouse Monitoring | Delta 테이블의 데이터 품질과 모델 성능을 자동으로 추적합니다 |
| 모니터 유형 | Snapshot(현재 상태), TimeSeries(시간 추이), InferenceLog(모델 추론) |
| 프로필 메트릭 | 통계, 분포, null 비율 등을 윈도우별로 자동 계산합니다 |
| 드리프트 감지 | KS 검정, 카이제곱 검정 등으로 데이터 분포 변화를 수치화합니다 |
| 커스텀 메트릭 | SQL 표현식으로 비즈니스 맞춤 메트릭을 정의합니다 |
| 자동 대시보드 | 모니터 생성 시 Lakeview 대시보드가 자동으로 만들어집니다 |

---

## 참고 링크

- [Lakehouse Monitoring 공식 문서](https://docs.databricks.com/aws/en/lakehouse-monitoring/)
- [모니터 생성 가이드](https://docs.databricks.com/aws/en/lakehouse-monitoring/create-monitor-api.html)
- [InferenceLog 모니터링](https://docs.databricks.com/aws/en/lakehouse-monitoring/monitor-model-quality.html)
- [커스텀 메트릭 정의](https://docs.databricks.com/aws/en/lakehouse-monitoring/custom-metrics.html)
- [Databricks 블로그: Lakehouse Monitoring 소개](https://www.databricks.com/blog/lakehouse-monitoring)
