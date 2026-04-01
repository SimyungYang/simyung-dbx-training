# Feature Engineering 개요

## 피처란?

> 💡 **피처(Feature)** 는 ML 모델의 입력으로 사용되는 **개별 데이터 속성** 입니다. 예를 들어, 사기 감지 모델에서 "거래 금액", "거래 시간", "최근 7일 거래 횟수", "해외 거래 여부" 등이 피처입니다.

모델의 성능은 알고리즘보다 **피처의 품질** 에 더 크게 좌우되는 경우가 많습니다. 좋은 피처를 설계하고 관리하는 것이 Feature Engineering의 핵심입니다.

---

## 왜 Feature Engineering이 중요한가요?

### 피처 관리의 문제점

| 문제 | 설명 |
|------|------|
| **중복 작업**| 여러 팀이 같은 피처를 각자 만들어 사용합니다 |
| **학습-서빙 불일치**| 학습 시 사용한 피처와 서빙 시 사용한 피처가 다릅니다 (Training-Serving Skew) |
| **데이터 유출**| 미래 데이터가 학습에 포함되어 현실과 다른 성능을 보입니다 (Data Leakage) |
| **버전 관리 부재**| 어떤 피처로 학습한 모델인지 추적이 어렵습니다 |

### Databricks Feature Engineering의 해결

| 해결 | 설명 |
|------|------|
| **중앙 피처 저장소**| Unity Catalog의 Feature Table에 피처를 중앙 관리합니다 |
| **자동 피처 조회**| `FeatureLookup`으로 학습과 서빙에서 동일한 피처를 사용합니다 |
| **Point-in-Time Lookup**| 시간 기반으로 피처를 조회하여 데이터 유출을 방지합니다 |
| **Online Tables**| 실시간 서빙에 최적화된 피처 저장소를 제공합니다 |

---

## Feature Engineering 워크플로우

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | 원본 데이터 (Delta 테이블) | 소스 데이터입니다 |
| 2 | 피처 계산 (Spark/SQL) | 원본 데이터에서 피처를 계산합니다 |
| 3 | Feature Table (Unity Catalog) | 계산된 피처를 저장하고 관리합니다 |
| 4a | 모델 학습 (FeatureLookup) | Feature Table에서 피처를 조회하여 학습합니다 |
| 4b | Online Table (실시간 서빙) | Feature Table을 실시간 조회 가능한 형태로 동기화합니다 |
| 5 | 모델 등록 | 피처 의존성을 기록하며 모델을 등록합니다 |
| 6 | Model Serving | 자동으로 Online Table에서 피처를 조회하여 추론합니다 |

---

## Feature Table

> 💡 **Feature Table** 은 Unity Catalog의 Delta 테이블을 **피처 저장소** 로 활용하는 것입니다. Primary Key를 기준으로 피처를 관리하며, 학습과 서빙 시 동일한 피처를 일관되게 사용할 수 있습니다.

### 피처 테이블 생성

```python
from databricks.feature_engineering import FeatureEngineeringClient

fe = FeatureEngineeringClient()

# 피처 계산
customer_features_df = spark.sql("""
    SELECT
        customer_id,
        COUNT(*) AS total_orders,
        SUM(amount) AS lifetime_revenue,
        AVG(amount) AS avg_order_value,
        MAX(order_date) AS last_order_date,
        DATEDIFF(CURRENT_DATE(), MAX(order_date)) AS days_since_last_order,
        COUNT(DISTINCT product_category) AS unique_categories
    FROM silver_orders
    GROUP BY customer_id
""")

# Feature Table로 등록
fe.create_table(
    name="catalog.schema.customer_features",
    primary_keys=["customer_id"],
    df=customer_features_df,
    description="고객 행동 기반 피처"
)
```

### 피처 업데이트

```python
# 새로운 피처 데이터로 업데이트
fe.write_table(
    name="catalog.schema.customer_features",
    df=updated_features_df,
    mode="merge"  # 기존 데이터와 병합 (upsert)
)
```

---

## FeatureLookup — 학습 데이터 생성

```python
from databricks.feature_engineering import FeatureLookup

# 학습 데이터에 피처를 자동으로 결합
training_set = fe.create_training_set(
    df=labels_df,  # 라벨만 있는 데이터 (customer_id, is_churned)
    feature_lookups=[
        FeatureLookup(
            table_name="catalog.schema.customer_features",
            lookup_key="customer_id"
        ),
        FeatureLookup(
            table_name="catalog.schema.product_features",
            lookup_key="product_id",
            feature_names=["avg_rating", "return_rate"]  # 특정 피처만 선택
        )
    ],
    label="is_churned"
)

training_df = training_set.load_df()
# → customer_id, is_churned + total_orders, lifetime_revenue, avg_order_value, ... + avg_rating, return_rate
```

---

## Point-in-Time Lookup

> 💡 **Point-in-Time Lookup** 은 시간 기반 피처를 결합할 때, 해당 시점 기준으로만 과거 데이터를 사용하여 **미래 데이터 유출(Data Leakage)** 을 방지하는 기능입니다.

```python
training_set = fe.create_training_set(
    df=labels_df,
    feature_lookups=[
        FeatureLookup(
            table_name="catalog.schema.customer_features",
            lookup_key="customer_id",
            timestamp_lookup_key="event_timestamp"  # 이 시점 기준으로 조회
        )
    ],
    label="is_churned"
)
```

---

## Feature Store vs Feature Table — 개념의 변화

Databricks의 피처 관리 체계는 시간에 따라 진화해 왔습니다. 이 변화를 이해하면 기존 문서/블로그와의 용어 혼동을 방지할 수 있습니다.

| 시기 | 이름 | 저장 위치 | 특징 |
|------|------|----------|------|
| **초기 (~2022)**| Feature Store (Workspace) | Workspace 내 자체 저장소 | Workspace 레벨 관리, 별도 UI |
| **현재 (2023~)**| Feature Engineering (UC 통합) | Unity Catalog Delta 테이블 | UC 3-Level Namespace, 통합 거버넌스 |

> 💡 **핵심 변화**: 과거에는 "Feature Store"라는 별도의 저장소가 있었습니다. 현재는 **Unity Catalog의 Delta 테이블 자체가 Feature Table** 로 사용됩니다. 별도의 저장소가 아니라, 기존 테이블에 Primary Key를 지정하면 Feature Table로 활용할 수 있습니다.

```python
# 기존 방식 (Workspace Feature Store) — deprecated
from databricks.feature_store import FeatureStoreClient
fs = FeatureStoreClient()  # ❌ 더 이상 권장하지 않음

# 현재 방식 (Unity Catalog Feature Engineering)
from databricks.feature_engineering import FeatureEngineeringClient
fe = FeatureEngineeringClient()  # ✅ 권장
```

### 마이그레이션 영향

| 항목 | Workspace Feature Store | UC Feature Engineering |
|------|------------------------|----------------------|
| **네임스페이스**| `feature_store.table_name` | `catalog.schema.table_name` |
| **권한**| Workspace ACL | Unity Catalog GRANT/REVOKE |
| **리니지**| 제한적 | UC 리니지 자동 추적 |
| **크로스 워크스페이스**| 불가 | 가능 (UC 공유) |
| **온라인 테이블**| 별도 구성 | 네이티브 통합 |

---

## Training-Serving Skew 방지 메커니즘

> 💡 **Training-Serving Skew(학습-서빙 불일치)** 란 모델 학습 시 사용한 피처와 프로덕션 서빙 시 사용하는 피처가 다르게 계산되는 문제입니다. 이 불일치는 모델 성능을 심각하게 저하시킬 수 있습니다.

### Skew가 발생하는 일반적인 원인

| 원인 | 설명 | 예시 |
|------|------|------|
| **계산 로직 불일치**| 학습 시 Python, 서빙 시 Java로 같은 로직을 다르게 구현 | 평균 계산에서 NULL 처리 방식 차이 |
| **데이터 소스 차이**| 학습은 배치 테이블, 서빙은 실시간 스트림 | 집계 기간 차이로 값이 다름 |
| **피처 버전 불일치**| 학습 시 v1 피처, 서빙 시 v2 피처 | 새 피처 추가 후 모델 재학습 안 함 |
| **타이밍 차이**| 학습은 일 단위 배치, 서빙은 실시간 | 당일 데이터 포함 여부 차이 |

### Databricks의 Skew 방지 아키텍처

| 단계 | 학습 시 | 서빙 시 |
|------|--------|--------|
| 입력 | labels_df (customer_id, is_churned) | 추론 요청 (customer_id) |
| 피처 조회 | FeatureLookup → Feature Table (Delta) | 동일한 피처 정의 자동 적용 |
| 결과 | training_df → 모델 학습 → 모델 등록 (피처 의존성 자동 기록) | Online Table에서 피처 조회 → 모델 추론 → 예측 결과 반환 |

> **핵심**: 학습과 서빙에서 **동일한 Feature Table 정의** 를 사용하므로 Training-Serving Skew가 방지됩니다.


| 방지 메커니즘 | 설명 |
|-------------|------|
| **단일 피처 소스**| 학습과 서빙 모두 같은 Feature Table에서 피처를 조회합니다 |
| **자동 피처 기록**| 모델 등록 시 어떤 Feature Table의 어떤 컬럼을 사용했는지 자동 기록됩니다 |
| **Online Table 동기화**| Feature Table(오프라인) → Online Table(온라인)이 자동 동기화됩니다 |
| **서빙 시 자동 조회**| Model Serving이 모델에 기록된 피처 의존성을 보고, Online Table에서 자동으로 피처를 조회합니다 |

---

## Point-in-Time 정확성 보장 원리

Point-in-Time Lookup의 내부 동작을 더 상세히 이해하면, 시계열 피처 설계에서 실수를 방지할 수 있습니다.

### 동작 원리

```text
labels_df:
  customer_id | event_timestamp | is_churned
  C001        | 2025-03-15      | True
  C001        | 2025-01-10      | False

Feature Table (customer_features):
  customer_id | feature_timestamp | total_orders | lifetime_revenue
  C001        | 2025-01-01       | 10           | 500,000
  C001        | 2025-02-01       | 15           | 750,000
  C001        | 2025-03-01       | 18           | 900,000

Point-in-Time Lookup 결과:
  C001 | 2025-03-15 → total_orders=18, lifetime_revenue=900,000  (3월 1일 기준)
  C001 | 2025-01-10 → total_orders=10, lifetime_revenue=500,000  (1월 1일 기준)
  ← 각 이벤트 시점 기준으로 "과거의 피처 값"만 사용
```

> ⚠️ **AsOf Join**: Point-in-Time Lookup은 내부적으로 **AsOf Join**(가장 가까운 과거 시점 매칭)을 수행합니다. `event_timestamp`보다 같거나 이전인 `feature_timestamp` 중 가장 최신 값을 선택합니다.

### timestamp_lookup_key 설계 주의사항

| 패턴 | 권장 여부 | 설명 |
|------|----------|------|
| **일별 스냅샷**| ✅ 권장 | 매일 피처를 계산하여 타임스탬프와 함께 저장 |
| **이벤트 기반 업데이트**| ✅ 좋음 | 주문, 결제 등 이벤트 발생 시마다 피처 업데이트 |
| **최종 값만 저장**| ❌ 비권장 | 과거 시점 조회가 불가능하여 Point-in-Time 의미 없음 |
| **불규칙 업데이트** | ⚠️ 주의 | 업데이트 간격이 불규칙하면 오래된 피처가 사용될 수 있음 |

---

## Feature Governance (UC 통합)

Feature Table은 Unity Catalog의 Delta 테이블이므로, 모든 UC 거버넌스 기능이 적용됩니다.

### 피처 수준 거버넌스

```sql
-- Feature Table 접근 권한 관리
GRANT SELECT ON TABLE catalog.ml.customer_features TO `ml_engineers`;
GRANT SELECT, MODIFY ON TABLE catalog.ml.customer_features TO `feature_team`;

-- 피처 태그 지정
ALTER TABLE catalog.ml.customer_features
SET TAGS ('domain' = 'customer', 'pii' = 'false', 'freshness_sla' = 'daily');

-- 컬럼별 태그 (민감 정보 표시)
ALTER TABLE catalog.ml.customer_features
ALTER COLUMN email SET TAGS ('pii' = 'true');

-- 피처 리니지 확인
SELECT * FROM system.lineage.table_lineage
WHERE target_table_full_name = 'catalog.ml.customer_features';
```

### 피처 검색 (Discovery)

```python
# Unity Catalog에서 피처 테이블 검색
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()

# 태그 기반 검색: 'customer' 도메인의 피처 테이블 찾기
tables = w.tables.list(catalog_name="catalog", schema_name="ml")
for t in tables:
    if t.table_type == "MANAGED" and "feature" in t.name:
        print(f"{t.full_name}: {t.comment}")
```

---

## Feature Freshness 모니터링

피처가 오래되면(stale), 모델 성능이 저하됩니다. 피처 신선도(Freshness)를 모니터링하는 것이 중요합니다.

### Freshness 모니터링 패턴

```python
# 피처 테이블의 마지막 업데이트 시간 확인
last_update = spark.sql("""
    DESCRIBE HISTORY catalog.ml.customer_features
    LIMIT 1
""").select("timestamp").collect()[0][0]

from datetime import datetime, timedelta
staleness = datetime.now() - last_update

if staleness > timedelta(hours=24):
    # Alert 발생: 피처가 24시간 이상 업데이트되지 않음
    print(f"⚠️ Feature staleness alert: {staleness}")
```

### Databricks Lakehouse Monitoring 연동

```sql
-- Feature Table에 모니터 설정
CREATE MONITOR catalog.ml.customer_features
WITH (
  SCHEDULE = CRON '0 0 * * *',  -- 매일 체크
  BASELINE_TABLE = 'catalog.ml.customer_features_baseline'
);

-- 모니터링 결과 확인: 데이터 드리프트 감지
SELECT * FROM catalog.ml.customer_features_profile_metrics
WHERE log_type = 'DATA_DRIFT'
  AND drift_detected = true;
```

---

## 대규모 Feature Table 설계 패턴

### Wide vs Narrow 테이블 전략

| 전략 | 구조 | 적합한 경우 |
|------|------|-----------|
| **Wide Table**| 하나의 테이블에 모든 피처 (수백 컬럼) | 단일 엔터티(고객, 상품)의 피처가 함께 사용되는 경우 |
| **Narrow Tables**| 도메인별로 분리된 여러 테이블 | 팀별로 독립적으로 피처를 관리하는 경우 |
| **Hybrid**| 핵심 피처는 Wide, 전문 피처는 Narrow | 대규모 조직에서 권장 |

### Wide Table 예시

```python
# 고객의 모든 피처를 하나의 테이블에
fe.create_table(
    name="catalog.ml.customer_features_wide",
    primary_keys=["customer_id"],
    timestamp_keys=["feature_date"],  # Point-in-Time용
    df=wide_features_df,  # 200+ 컬럼
    description="고객 통합 피처 테이블 (행동, 인구통계, 거래, 상호작용 피처 포함)"
)
```

### Narrow Tables + 다중 FeatureLookup

```python
# 여러 도메인별 Feature Table에서 조합
training_set = fe.create_training_set(
    df=labels_df,
    feature_lookups=[
        FeatureLookup(table_name="catalog.ml.customer_demographics", lookup_key="customer_id"),
        FeatureLookup(table_name="catalog.ml.customer_transactions", lookup_key="customer_id",
                      timestamp_lookup_key="event_date"),
        FeatureLookup(table_name="catalog.ml.customer_interactions", lookup_key="customer_id",
                      timestamp_lookup_key="event_date"),
        FeatureLookup(table_name="catalog.ml.product_features", lookup_key="product_id"),
    ],
    label="is_churned"
)
```

### 대용량 Feature Table 최적화

| 최적화 | 설명 | 효과 |
|--------|------|------|
| **Liquid Clustering**| `CLUSTER BY (customer_id)`로 Primary Key 기준 클러스터링 | Lookup 성능 5~10배 향상 |
| **Z-ORDER**| `OPTIMIZE ... ZORDER BY (customer_id)` | 범위 스캔 최적화 |
| **파티션 전략**| 일별/월별 feature_date 파티션 | Point-in-Time 쿼리 시 불필요한 스캔 방지 |
| **Auto Compaction**| Delta Lake 자동 파일 병합 | 소규모 파일 문제 방지 |
| **Column Pruning**| FeatureLookup에서 `feature_names` 명시 | 필요한 컬럼만 읽어서 I/O 절감 |

```python
# Liquid Clustering 적용
spark.sql("""
    ALTER TABLE catalog.ml.customer_features
    CLUSTER BY (customer_id)
""")

# 특정 피처만 선택 (Column Pruning)
FeatureLookup(
    table_name="catalog.ml.customer_features",
    lookup_key="customer_id",
    feature_names=["total_orders", "lifetime_revenue", "avg_order_value"]  # 필요한 것만
)
```

---

## Online Table과 실시간 서빙 심화

### Online Table 동기화 모드

| 모드 | 지연 시간 | 비용 | 적합한 경우 |
|------|----------|------|-----------|
| **Snapshot**| 분~시간 (스케줄) | 낮음 | 피처가 자주 변하지 않는 경우 (인구통계 등) |
| **Triggered**| 분 단위 | 중간 | 피처 업데이트 직후 동기화 필요 시 |
| **Continuous (CDC)**| 초 단위 | 높음 | 실시간 피처가 필요한 경우 (실시간 거래 등) |

```python
# Online Table 생성 (Continuous 동기화)
w.online_tables.create(
    name="catalog.ml.customer_features_online",
    spec={
        "source_table_full_name": "catalog.ml.customer_features",
        "primary_key_columns": ["customer_id"],
        "run_continuously": {
            "triggered": False  # True면 Triggered 모드
        }
    }
)
```

> ⚠️ **비용 주의**: Continuous 모드의 Online Table은 항상 실행 상태이므로 비용이 지속적으로 발생합니다. 실시간성이 정말 필요한 피처에만 적용하고, 나머지는 Snapshot이나 Triggered 모드를 사용하세요.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Feature**| ML 모델의 입력 데이터 속성입니다 |
| **Feature Table**| Unity Catalog에서 피처를 중앙 관리하는 테이블입니다 |
| **FeatureLookup**| 학습 데이터에 피처를 자동으로 결합합니다 |
| **Point-in-Time**| 시간 기준으로 피처를 조회하여 데이터 유출을 방지합니다 |
| **Training-Serving Skew**| 학습과 서빙의 피처 불일치 문제이며, Feature Table로 방지합니다 |
| **Feature Governance**| UC 권한, 태그, 리니지로 피처를 거버넌스합니다 |
| **Feature Freshness**| 피처의 신선도를 모니터링하여 모델 성능 저하를 방지합니다 |
| **Online Table** | 실시간 서빙에 최적화된 피처 저장소입니다 |

---

## 참고 링크

- [Databricks: Feature Engineering](https://docs.databricks.com/aws/en/machine-learning/feature-store/)
- [Databricks: Feature tables](https://docs.databricks.com/aws/en/machine-learning/feature-store/feature-tables.html)
