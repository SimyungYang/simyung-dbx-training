# 실시간 피처 서빙 (Online Feature Serving)

## 왜 온라인 서빙이 필요한가요?

ML 모델이 실시간으로 예측을 수행하려면, 입력 피처도 **실시간으로 조회**할 수 있어야 합니다. 예를 들어, 고객이 결제 버튼을 누른 순간 사기 탐지 모델이 동작하려면, 해당 고객의 최근 거래 패턴, 평균 결제 금액 등의 피처를 **수 밀리초 안에** 가져와야 합니다.

일반 Delta 테이블은 대량 데이터 스캔과 집계에 최적화되어 있어, 단건 키 기반 조회(Point Lookup)에는 적합하지 않습니다. 이 문제를 해결하기 위해 **Online Table**이 필요합니다.

| 환경 | 구성 요소 | 역할 | 연결 |
|------|-----------|------|------|
| **오프라인 (학습/배치)** | Feature Table (Delta Lake) | 대량 데이터 스캔 | 모델 학습에 데이터 제공 |
|  | 모델 학습 | Feature Table에서 대량 데이터를 읽어 학습 | - |
| **온라인 (실시간 서빙)** | Online Table (밀리초 조회) | Feature Table에서 자동 동기화 | Model Serving에 단건 조회 제공 |
|  | Model Serving Endpoint | 실시간 추론 수행 | Online Table에서 피처를 조회하여 예측 |

> 💡 **Feature Store 패턴**: 오프라인 스토어(배치 학습용)와 온라인 스토어(실시간 서빙용)를 분리하되, 동일한 피처 정의를 공유하는 것이 현대 ML 아키텍처의 핵심 패턴입니다. 이를 통해 **학습-서빙 편향(Training-Serving Skew)** 을 방지할 수 있습니다.

---

## 온라인 스토어 vs 오프라인 스토어

| 비교 항목 | 오프라인 스토어 (Delta Table) | 온라인 스토어 (Online Table) |
|----------|---------------------------|---------------------------|
| **최적화 대상** | 대량 스캔, 집계, 조인 | 단건 키 기반 조회 (Point Lookup) |
| **응답 시간** | 수 초 ~ 수 분 | **수 밀리초 (< 10ms)** |
| **적합한 용도** | 배치 학습, 탐색적 분석, 피처 백필 | 실시간 추론, 온라인 서빙 |
| **데이터 형식** | Parquet (Delta Lake) | 키-값 저장소 (내부 최적화) |
| **동기화** | 원본 (Source of Truth) | Delta 테이블과 자동 동기화 |
| **스케일** | 페타바이트 규모 | 밀리초 응답을 위한 크기 제한 있음 |
| **비용 모델** | 스토리지 + 컴퓨트 | 프로비저닝된 용량 단위(CU) 과금 |

> 💡 **Training-Serving Skew(학습-서빙 편향)**: 학습 시 사용한 피처와 서빙 시 사용하는 피처가 달라서 모델 성능이 저하되는 현상입니다. Online Table은 오프라인 Feature Table과 자동 동기화되므로 이 문제를 근본적으로 해결합니다.

---

## Databricks Online Tables 상세

### 아키텍처

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | Feature Table (Delta Lake) | CDF 스트리밍으로 동기화 파이프라인에 변경 사항을 전달합니다 |
| 2 | 동기화 파이프라인 | 증분 업데이트로 Online Table에 반영합니다 |
| 3 | Online Table (키-값 스토어) | 밀리초 단위 단건 조회를 지원합니다 |
| 4 | 애플리케이션 | `customer_id=12345`로 Model Serving Endpoint에 요청합니다 |
| 5 | Model Serving Endpoint | Online Table에서 자동으로 피처를 조회하여 예측 결과를 반환합니다 |

1. Feature Table(Delta Lake)에 피처 데이터가 저장됩니다
2. Online Table이 **Change Data Feed(CDF)** 를 통해 변경 사항을 **자동으로 동기화**합니다
3. Model Serving 엔드포인트가 예측 요청을 받으면, Online Table에서 **피처를 자동 조회**합니다
4. 클라이언트는 **Primary Key만 전송**하면 됩니다 (피처를 직접 전달할 필요 없음)

### 동기화 메커니즘

Online Table은 소스 Delta 테이블의 변경 사항을 세 가지 모드로 동기화합니다.

| 모드 | 설명 | 지연시간 | 적합한 경우 |
|------|------|---------|-----------|
| **TRIGGERED** | 수동/스케줄로 증분 업데이트 | 분 ~ 시간 | 피처가 주기적으로 갱신되는 경우 |
| **CONTINUOUS** | 스트리밍 파이프라인으로 즉시 반영 | **초 단위** | 실시간 피처 갱신이 필요한 경우 |
| **SNAPSHOT** | 전체 데이터를 한 번에 동기화 | 분 ~ 시간 | 초기 적재, CDF 미지원 소스 |

> 💡 **TRIGGERED/CONTINUOUS 모드**를 사용하려면 소스 테이블에 **Change Data Feed(CDF)** 가 활성화되어 있어야 합니다. CDF는 Delta 테이블의 행 수준 변경 이력을 추적하는 기능입니다.

```sql
-- CDF 활성화 (TRIGGERED/CONTINUOUS 모드에 필수)
ALTER TABLE ml_prod.features.customer_features
SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');
```

### 지연시간 특성

| 단계 | 소요 시간 |
|------|----------|
| 피처 조회 (Online Table → Endpoint) | **1~5ms** |
| 동기화 지연 (CONTINUOUS 모드) | **수 초 ~ 1분** |
| 동기화 지연 (TRIGGERED 모드) | **설정된 스케줄에 따름** |
| 모델 추론 포함 전체 응답 | **10~100ms** (모델 복잡도에 따라 다름) |

---

## Online Table 생성 방법

### 방법 1: SQL로 생성

```sql
-- 가장 간단한 방법: SQL DDL
CREATE ONLINE TABLE ml_prod.features.customer_features_online
FROM ml_prod.features.customer_features;

-- 동기화 모드를 지정하는 경우
CREATE ONLINE TABLE ml_prod.features.customer_features_online
FROM ml_prod.features.customer_features
WITH (
    SYNC_MODE = 'TRIGGERED'  -- TRIGGERED | CONTINUOUS | SNAPSHOT
);
```

### 방법 2: UI로 생성

1. **Catalog Explorer**에서 소스 Feature Table로 이동합니다
2. 우측 상단 **"Create Online Table"** 버튼을 클릭합니다
3. Online Table 이름, 동기화 모드를 설정합니다
4. **"Create"**를 클릭하면 동기화가 시작됩니다

### 방법 3: Python SDK로 생성

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Online Table 생성
online_table = w.online_tables.create(
    name="ml_prod.features.customer_features_online",
    spec={
        "source_table_full_name": "ml_prod.features.customer_features",
        "primary_key_columns": ["customer_id"],
        "run_triggered": {}  # TRIGGERED 모드
        # "run_continuously": {}  # CONTINUOUS 모드를 원하는 경우
    }
)

print(f"Online Table 생성 완료: {online_table.name}")
print(f"상태: {online_table.status}")
```

### 동기화 상태 확인

```sql
-- Online Table 동기화 상태 확인
DESCRIBE ONLINE TABLE ml_prod.features.customer_features_online;
```

```python
# Python으로 상태 확인
status = w.online_tables.get("ml_prod.features.customer_features_online")
print(f"상태: {status.status.detailed_state}")
print(f"마지막 동기화: {status.status.message}")
```

---

## 모델 서빙에서 Online Table 활용

### FeatureFunction으로 자동 피처 조회

Online Table의 가장 큰 장점은 Model Serving 엔드포인트에서 **피처를 자동으로 조회**하는 것입니다. 클라이언트는 Primary Key만 전송하면 됩니다.

```python
from databricks.feature_engineering import FeatureEngineeringClient, FeatureLookup

fe = FeatureEngineeringClient()

# 피처 조회 정의
feature_lookups = [
    FeatureLookup(
        table_name="ml_prod.features.customer_features",
        lookup_key=["customer_id"],
        feature_names=["avg_transaction_amount", "transaction_count_30d", "account_age_days"]
    ),
    FeatureLookup(
        table_name="ml_prod.features.merchant_features",
        lookup_key=["merchant_id"],
        feature_names=["avg_fraud_rate", "category"]
    ),
]

# 피처 조회가 포함된 학습 데이터셋 생성
training_set = fe.create_training_set(
    df=raw_transactions_df,
    feature_lookups=feature_lookups,
    label="is_fraud",
)

# 모델 학습 후 로깅 (피처 조회 메타데이터가 함께 저장됩니다)
fe.log_model(
    model=trained_model,
    artifact_path="fraud_model",
    flavor=mlflow.sklearn,
    training_set=training_set,
    registered_model_name="ml_prod.models.fraud_detector",
)
```

> 💡 **핵심 포인트**: `fe.log_model()`로 로깅된 모델은 어떤 Feature Table에서 어떤 피처를 조회해야 하는지 **메타데이터에 자동 기록**됩니다. 이 모델을 서빙 엔드포인트에 배포하면, Online Table에서 피처를 자동으로 조회합니다.

### 엔드포인트에서의 실시간 조회

```python
# 클라이언트 코드: Primary Key만 전송하면 됩니다
import requests

response = requests.post(
    f"https://{workspace_url}/serving-endpoints/fraud-detector/invocations",
    json={
        "dataframe_records": [
            {
                "customer_id": 12345,
                "merchant_id": "M-9876",
                "transaction_amount": 450.00,
                # ↑ 이 값들만 전송하면 됩니다
                # avg_transaction_amount, transaction_count_30d 등은
                # Online Table에서 자동으로 조회됩니다
            }
        ]
    },
    headers={"Authorization": f"Bearer {token}"}
)

print(response.json())
# {"predictions": [{"is_fraud": 0.87}]}
```

| 단계 | 발신 | 수신 | 내용 |
|------|------|------|------|
| 1 | 클라이언트 | 서빙 엔드포인트 | customer_id, merchant_id, amount 전송 |
| 2 | 서빙 엔드포인트 | Online Table (고객 피처) | customer_id로 조회 |
| 3 | Online Table (고객 피처) | 서빙 엔드포인트 | avg_amount, count_30d, age 반환 |
| 4 | 서빙 엔드포인트 | Online Table (가맹점 피처) | merchant_id로 조회 |
| 5 | Online Table (가맹점 피처) | 서빙 엔드포인트 | fraud_rate, category 반환 |
| 6 | 서빙 엔드포인트 | ML 모델 | 전체 피처 조합 전달 |
| 7 | ML 모델 | 서빙 엔드포인트 | prediction: 0.87 |
| 8 | 서빙 엔드포인트 | 클라이언트 | `{"is_fraud": 0.87}` |

---

## 실습: Online Table 생성부터 모델 서빙까지

### Step 1: Feature Table 준비

```sql
-- Feature Table 생성
CREATE TABLE IF NOT EXISTS ml_prod.features.customer_features (
    customer_id BIGINT,
    avg_transaction_amount DOUBLE,
    transaction_count_30d INT,
    account_age_days INT,
    preferred_category STRING,
    update_timestamp TIMESTAMP
)
TBLPROPERTIES (
    'delta.enableChangeDataFeed' = 'true'  -- CDF 활성화
);

-- 피처 데이터 적재
INSERT INTO ml_prod.features.customer_features
SELECT
    customer_id,
    AVG(amount) AS avg_transaction_amount,
    COUNT(*) AS transaction_count_30d,
    DATEDIFF(CURRENT_DATE(), MIN(signup_date)) AS account_age_days,
    MODE(category) AS preferred_category,
    CURRENT_TIMESTAMP() AS update_timestamp
FROM ml_prod.gold.transactions
WHERE transaction_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY customer_id;
```

### Step 2: Online Table 생성

```sql
CREATE ONLINE TABLE ml_prod.features.customer_features_online
FROM ml_prod.features.customer_features;
```

### Step 3: 모델 학습 및 로깅 (피처 조회 포함)

```python
from databricks.feature_engineering import FeatureEngineeringClient, FeatureLookup
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier

fe = FeatureEngineeringClient()

feature_lookups = [
    FeatureLookup(
        table_name="ml_prod.features.customer_features",
        lookup_key=["customer_id"],
    ),
]

training_set = fe.create_training_set(
    df=labeled_transactions_df,   # customer_id + is_fraud 레이블
    feature_lookups=feature_lookups,
    label="is_fraud",
)

training_df = training_set.load_df().toPandas()
X = training_df.drop(columns=["is_fraud", "customer_id"])
y = training_df["is_fraud"]

model = GradientBoostingClassifier(n_estimators=100)
model.fit(X, y)

fe.log_model(
    model=model,
    artifact_path="fraud_model",
    flavor=mlflow.sklearn,
    training_set=training_set,
    registered_model_name="ml_prod.models.fraud_detector",
)
```

### Step 4: 서빙 엔드포인트 배포

엔드포인트를 생성하면, 모델 메타데이터에 기록된 Feature Table에 대응하는 Online Table에서 피처를 자동 조회합니다.

---

## 성능 최적화

### Primary Key 설계

| 원칙 | 설명 |
|------|------|
| **복합 키 최소화** | PK 컬럼 수가 적을수록 조회 성능이 좋습니다 |
| **정수형 키 선호** | 문자열보다 정수형 키가 조회가 빠릅니다 |
| **핫 키 분산** | 특정 키에 트래픽이 집중되지 않도록 설계합니다 |

### 테이블 크기 관리

```sql
-- 필요한 피처만 포함하여 Online Table 크기를 최소화합니다
CREATE ONLINE TABLE ml_prod.features.customer_features_slim_online
FROM (
    SELECT customer_id, avg_transaction_amount, transaction_count_30d
    FROM ml_prod.features.customer_features
);
```

### 캐싱 전략

- Model Serving 엔드포인트는 자주 조회되는 키에 대해 **자동 캐싱**을 수행합니다
- CONTINUOUS 모드에서는 캐시가 변경 사항에 따라 자동 무효화됩니다
- 캐시 적중률이 높을수록 P50 지연시간이 개선됩니다

---

## 모범 사례 및 제한사항

### 모범 사례

| 항목 | 권장 사항 |
|------|---------|
| **동기화 모드 선택** | 피처 갱신 빈도에 맞는 모드를 선택합니다. 대부분 TRIGGERED로 충분합니다 |
| **피처 수 최적화** | 서빙에 필요한 피처만 Online Table에 포함합니다 |
| **모니터링** | 동기화 상태와 조회 지연시간을 정기적으로 확인합니다 |
| **CDF 활성화** | TRIGGERED/CONTINUOUS 모드를 위해 반드시 CDF를 활성화합니다 |
| **테스트** | 배포 전 Online Table의 데이터 정합성을 검증합니다 |

### 제한사항

| 제한 사항 | 내용 |
|----------|------|
| **테이블 크기** | 최대 수백 GB (정확한 한도는 CU 크기에 따라 다름) |
| **컬럼 수** | 최대 200개 컬럼 |
| **Primary Key** | 복합 키 포함 최대 10개 컬럼 |
| **지원 데이터 타입** | 기본 타입 지원, 복잡한 중첩 구조는 제한적 |
| **Unity Catalog 필수** | Online Table은 Unity Catalog에 등록된 테이블에서만 생성 가능합니다 |
| **리전 제한** | 일부 클라우드 리전에서는 사용 불가능할 수 있습니다 |

> ⚠️ **비용 고려**: Online Table은 프로비저닝된 용량(CU) 단위로 과금됩니다. 테이블 크기와 조회 트래픽에 맞는 적절한 CU를 선택하시기 바랍니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Online Table** | 실시간 단건 조회에 최적화된 피처 저장소입니다 |
| **자동 동기화** | Delta Feature Table의 변경이 CDF를 통해 자동 반영됩니다 |
| **밀리초 응답** | 1~5ms 수준의 초저지연 피처 조회를 지원합니다 |
| **자동 피처 조회** | 클라이언트는 PK만 전송하면, 피처는 엔드포인트가 자동으로 조회합니다 |
| **학습-서빙 일관성** | 오프라인/온라인 동일 피처를 사용하여 Skew를 방지합니다 |

---

## 참고 링크

- [Databricks: Online tables](https://docs.databricks.com/aws/en/machine-learning/feature-store/online-tables.html)
- [Databricks: Feature Serving](https://docs.databricks.com/aws/en/machine-learning/feature-store/feature-function-serving.html)
- [Databricks: Feature Engineering client](https://docs.databricks.com/aws/en/machine-learning/feature-store/feature-engineering-client.html)
- [Databricks: Online tables pricing](https://www.databricks.com/product/pricing)
- [Databricks Blog: Real-time ML with Feature Store](https://www.databricks.com/blog/feature-engineering-unity-catalog)
