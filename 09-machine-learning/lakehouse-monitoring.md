# Lakehouse Monitoring — 데이터 및 모델 품질 모니터링

## Lakehouse Monitoring이란?

> 💡 **Lakehouse Monitoring** 은 Databricks의 **데이터 품질 및 모델 성능 모니터링 서비스** 입니다. Delta 테이블에 모니터를 연결하면, 데이터의 통계 프로필, 분포 변화(드리프트), 이상치 등을 자동으로 추적하고 대시보드로 시각화합니다.

### 왜 모니터링이 필요한가요?

데이터 파이프라인과 ML 모델은 배포 후에도 **지속적으로 품질이 변할 수 있습니다**. 다음과 같은 문제를 사전에 감지하지 못하면 비즈니스에 큰 영향을 미칩니다.

| 문제 유형 | 예시 | 영향 |
|-----------|------|------|
| **데이터 드리프트**| 고객 연령 분포가 갑자기 변함 | 모델 예측 정확도 하락 |
| **데이터 품질 저하**| null 비율 급증, 이상 값 유입 | 잘못된 비즈니스 의사결정 |
| **컨셉 드리프트**| 계절 변화로 구매 패턴 변경 | 추천 모델 성능 저하 |
| **스키마 변경**| 업스트림 테이블 컬럼 변경 | 파이프라인 실패 |

> 💡 **비유**: Lakehouse Monitoring은 데이터 파이프라인의 "건강 검진"입니다. 정기적으로 데이터의 건강 상태를 검사하여, 문제가 심각해지기 전에 미리 발견합니다.

---

## 모니터 유형

Lakehouse Monitoring은 세 가지 유형의 모니터를 지원합니다.

| 모니터 유형 | 설명 | 사용 시나리오 |
|------------|------|-------------|
| **Snapshot**| 테이블의 현재 상태를 주기적으로 프로파일링합니다 | 정적 테이블, 차원 테이블 |
| **TimeSeries**| 시간 컬럼 기준으로 데이터 변화를 추적합니다 | 이벤트 로그, 거래 데이터 |
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

모니터가 실행되면 **프로필 메트릭 테이블** 과 **드리프트 메트릭 테이블** 두 개가 자동 생성됩니다.

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

현재 윈도우와 **기준선(Baseline)** 또는 **이전 윈도우** 를 비교하여 변화를 수치화합니다.

| 드리프트 메트릭 | 설명 | 적용 대상 |
|---------------|------|----------|
| **KS 검정 (Kolmogorov-Smirnov)**| 두 분포의 최대 차이를 측정합니다 | 수치형 |
| **카이제곱 검정 (Chi-Square)**| 범주형 분포의 변화를 측정합니다 | 범주형 |
| **Jensen-Shannon Divergence**| 두 확률 분포의 차이를 측정합니다 | 수치형/범주형 |
| **PSI (Population Stability Index)**| 모집단 안정성을 측정합니다 | 수치형/범주형 |

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

기본 메트릭 외에 **비즈니스 로직에 맞는 커스텀 메트릭** 을 정의할 수 있습니다.

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

드리프트를 감지하려면 **"정상 상태"의 기준** 이 필요합니다. 별도의 기준선 테이블을 지정할 수 있습니다.

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
| **기준선 테이블 지정**| 과거 안정 기간의 데이터를 별도 테이블로 제공합니다 |
| **이전 윈도우 비교**| 기준선을 지정하지 않으면 이전 시간 윈도우와 비교합니다 |

---

## 대시보드 자동 생성

모니터를 생성하면 `assets_dir`에 **Lakeview 대시보드가 자동으로 생성** 됩니다. 이 대시보드에는 다음 정보가 포함됩니다.

| 대시보드 섹션 | 내용 |
|-------------|------|
| **데이터 개요**| 행 수 추이, 테이블 크기 변화 |
| **컬럼별 프로필**| 각 컬럼의 분포, 통계 변화 |
| **드리프트 지표**| KS/카이제곱 검정 결과, 드리프트 점수 |
| **null 비율 추이**| 시간에 따른 null 비율 변화 |
| **모델 성능**(InferenceLog) | 정확도, 정밀도, 재현율 추이 |

---

## 알림 설정

모니터링 메트릭을 기반으로 **Databricks SQL Alert** 를 설정하여 이상 징후를 자동으로 알릴 수 있습니다.

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

1. 위 쿼리를 **SQL Editor** 에서 저장합니다
2. **Alerts** 메뉴에서 해당 쿼리를 기반으로 알림을 생성합니다
3. 결과가 비어 있지 않으면 **이메일 또는 Slack으로 알림** 을 발송합니다

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
| **적절한 윈도우 크기**| 데이터 빈도에 맞는 granularity를 선택합니다 (실시간 → 1 hour, 일간 → 1 day) |
| **슬라이싱 활용**| 지역, 카테고리 등으로 슬라이싱하면 세부 드리프트를 발견할 수 있습니다 |
| **기준선 설정**| 안정된 과거 데이터를 기준선으로 지정하여 드리프트 판단 정확도를 높입니다 |
| **커스텀 메트릭**| 비즈니스에 중요한 지표는 커스텀 메트릭으로 추가합니다 |
| **알림 연동**| 드리프트 감지 시 자동 알림을 설정하여 빠르게 대응합니다 |
| **비용 관리**| 불필요하게 높은 빈도의 스케줄은 비용을 증가시킵니다 |

---

## 현업 사례: 모델 성능이 서서히 저하되는데 아무도 모르고 3개월 지난 사례

모니터링이 왜 필수인지를 가장 잘 보여주는 현업 사례입니다. "배포하면 끝"이라는 생각이 얼마나 위험한지 공유합니다.

### 사례: 이커머스 추천 모델의 조용한 죽음

```
[상황]
- 상품 추천 모델: LightGBM, 클릭률(CTR) 예측
- 배포 시 CTR 예측 정확도: AUC 0.82
- 주 1회 A/B 테스트 리포트를 수동으로 확인

[타임라인]
1월: 배포. AUC 0.82. 추천 클릭률 8.5%
2월: AUC 0.79. 추천 클릭률 7.2%. 하지만 아무도 모니터링하지 않음
3월: AUC 0.71. 추천 클릭률 5.1%. 마케팅팀에서 "추천이 이상하다" 컴플레인
4월: 원인 분석 시작. 알고 보니...

**원인**:
- 2월부터 모바일 앱 개편으로 사용자 행동 패턴이 완전히 변함
- 기존: 홈 → 카테고리 → 상품 → 구매
- 변경: 홈 → 검색 → 상품 → 구매 (카테고리 사용 급감)
- 모델의 핵심 피처인 "카테고리 탐색 횟수"가 거의 0이 됨
- 결과: 피처 분포의 극적 변화 → 예측 정확도 하락

[손실]
- 3개월간 추천 품질 저하로 매출 약 2억 원 감소 (추정)
- 긴급 모델 재학습에 2주 소요
- 이후 Lakehouse Monitoring 도입
```

### 모니터링이 있었다면

| 시점 | 이벤트 |
|------|--------|
| **1월 말**| InferenceLog 모니터가 "카테고리 탐색 횟수" 피처의 KS 검정 p-value < 0.01 감지 |
| | 자동 Slack 알림: "피처 드리프트 감지: category_browse_count 분포 변화" |
| | 담당자 확인 → 모바일 앱 개편과 연관성 파악 |
| | 2주 내 모델 재학습 → 손실 최소화 |

### 드리프트 감지의 현실적 한계

Lakehouse Monitoring의 드리프트 감지는 강력하지만, 현업에서 알아야 할 한계가 있습니다.

| 한계 | 설명 | 대응 방법 |
|------|------|----------|
| **거짓 양성(False Positive)**| 정상적인 계절 변화도 드리프트로 감지 | 기준선을 같은 계절 데이터로 설정 |
| **거짓 음성(False Negative)**| 분포는 비슷한데 관계가 변한 경우 (컨셉 드리프트) | Ground Truth 수집 + 성능 메트릭 모니터링 |
| **알림 피로(Alert Fatigue)**| 너무 많은 알림이 발생하면 무시하게 됨 | 임계값을 엄격하게 설정 + 중요 피처만 모니터 |
| **Ground Truth 지연**| 실제 결과(레이블)는 수일~수주 후에 확인 가능 | 프록시 메트릭(클릭률, 전환율) 활용 |
| **인과 관계 파악 불가**| "분포가 변했다"는 알지만 "왜 변했는지"는 모름 | 슬라이싱 활용 + 도메인 전문가 분석 |

> ⚠️ **현업에서는 이렇게 합니다**: 드리프트 감지 알림이 발생했다고 즉시 모델을 재학습하지 않습니다. "** 알림 → 영향도 분석 → 비즈니스 메트릭 확인 → 재학습 결정**" 프로세스를 따릅니다. 드리프트가 감지되었는데 비즈니스 메트릭(매출, 전환율)에 영향이 없으면, 모니터링만 강화하고 재학습은 보류합니다.

---

## 모니터링 대시보드 실전 구성 패턴

자동 생성 대시보드만으로는 부족한 경우가 많습니다. 현업에서 구성하는 효과적인 모니터링 대시보드 패턴을 공유합니다.

### 3-Layer 모니터링 구조

| Layer | 대상 | 메트릭 |
|-------|------|--------|
| **Layer 1: 비즈니스 메트릭**| 경영진/PM | 추천 클릭률(CTR) 추이, 전환율 추이, 예측 정확도(정밀도/재현율) 추이, 모델 대비 베이스라인 성능 비교 |
| **Layer 2: 데이터 품질**| 데이터 엔지니어 | 테이블별 row count 추이(급변 감지), 핵심 컬럼 null 비율 추이, 피처 분포 변화(KS 검정 p-value), 파이프라인 지연 시간 |
| **Layer 3: 모델 성능 상세** | ML 엔지니어 | 피처별 드리프트 점수, 예측 분포 변화(평균/분산), 모델 버전별 성능 비교, 슬라이싱별 성능(지역, 고객 세그먼트 등) |

### 알림 설정 실전 패턴

```sql
-- 알림 1: 핵심 피처 드리프트 감지
-- KS 검정 p-value가 0.01 미만이면 긴급 알림
SELECT
    window.start AS check_time,
    column_name,
    ks_test.statistic AS drift_score,
    ks_test.pvalue AS p_value
FROM catalog.schema.model_inference_drift_metrics
WHERE column_name IN ('age', 'income', 'purchase_frequency')  -- 핵심 피처만
  AND ks_test.pvalue < 0.01
  AND window.start >= current_date() - INTERVAL 1 DAY;

-- 알림 2: 예측 분포 이상
-- 모델 예측값의 평균이 기준선 대비 20% 이상 벗어나면 알림
SELECT
    window.start,
    mean AS current_mean,
    baseline_mean,
    ABS(mean - baseline_mean) / baseline_mean * 100 AS deviation_pct
FROM catalog.schema.model_inference_profile_metrics
WHERE column_name = 'prediction'
  AND ABS(mean - baseline_mean) / baseline_mean > 0.20
  AND window.start >= current_date() - INTERVAL 1 DAY;
```

> 💡 **현업 팁**: 알림의 핵심은 "**Action을 취할 수 있는 알림**" 만 보내는 것입니다. "피처 X의 분포가 바뀌었습니다"만으로는 부족합니다. "피처 X의 분포가 바뀌었고, 이로 인해 예측 정확도가 Y% 하락했습니다. 재학습을 검토하세요."까지 포함해야 담당자가 행동할 수 있습니다.

### 모니터링 비용 관리

| 모니터링 대상 | 권장 빈도 | 비용 영향 |
|-------------|----------|----------|
| **핵심 프로덕션 모델**| 매시간 | 높음 — 하지만 비즈니스 영향이 크므로 필수 |
| **배치 모델 (일 1회 실행)**| 매일 | 중간 |
| **데이터 품질 (Bronze/Silver)**| 매일 | 중간 |
| **개발/실험 모델**| 주 1회 또는 수동 | 낮음 |
| **아카이브 테이블**| 모니터 불필요 | 없음 |

> ⚠️ **이것을 안 하면**: 모든 테이블에 매시간 모니터를 걸면, 메트릭 테이블이 급격히 커지고 비용이 예상 외로 증가합니다. 프로덕션에 직접 영향을 주는 테이블만 고빈도로 모니터링하고, 나머지는 일간 또는 주간으로 설정하세요.

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
