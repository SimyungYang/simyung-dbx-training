# 커스텀 모델 배포 (Custom Model Deployment)

## 왜 커스텀 모델 배포가 필요한가?

Databricks Model Serving은 등록된 MLflow 모델을 REST API 엔드포인트로 배포하는 관리형 서비스입니다. **Foundation Model API (FMAPI)** 가 제공하는 사전 학습 모델(llama, mixtral 등)로 충분하지 않은 경우, 커스텀 모델 배포가 필요합니다.

### Foundation Model API vs. 커스텀 모델 배포

| 구분 | Foundation Model API | 커스텀 모델 배포 |
|------|---------------------|-----------------|
| **모델 제어** | Databricks가 제공하는 모델만 사용 | 직접 학습/파인튜닝한 모델 사용 가능 |
| **전처리/후처리** | 불가 (입출력 형식 고정) | 완전한 자유도 |
| **비용** | 토큰당 종량제 (Pay-per-token) | 인스턴스 시간당 과금 |
| **운영 복잡도** | 낮음 (인프라 자동 관리) | 높음 (의존성, 스케일링 직접 설정) |
| **레이턴시** | 공유 인프라 (변동 가능) | 전용 인스턴스 (예측 가능) |
| **규정 준수** | 데이터가 외부 모델 통과 | 워크스페이스 내부에서 처리 |

### 커스텀 배포가 필요한 경우

| 사용 사례 | 설명 |
|-----------|------|
| **전후처리 파이프라인** | 입력 데이터 정규화, 피처 엔지니어링, 출력 후처리가 필요한 경우 |
| **파인튜닝 모델** | 도메인 특화 데이터로 Fine-tuning한 모델을 배포해야 하는 경우 |
| **자체 학습 모델** | 사내 데이터로 처음부터 학습한 ML/DL 모델 |
| **앙상블 모델** | 여러 모델의 예측을 결합해야 하는 경우 |
| **외부 라이브러리 의존** | 표준 MLflow flavor에 없는 프레임워크를 사용하는 경우 |
| **비즈니스 로직 포함** | 예측값에 규칙 기반 로직을 적용해야 하는 경우 |
| **GPU 추론** | 대규모 딥러닝 모델을 GPU로 서빙해야 하는 경우 |
| **데이터 거버넌스** | 입력 데이터가 외부 서비스로 나가면 안 되는 경우 |

> **모델 서빙 엔드포인트 (Model Serving Endpoint)** 란 학습된 ML 모델을 REST API로 노출하여, 애플리케이션에서 실시간으로 예측을 요청할 수 있게 해주는 서비스입니다. Databricks에서는 인프라 관리, 오토스케일링, 모니터링을 자동으로 처리합니다.

---

## 배포 가능한 모델 유형

| 배포 유형 | 설명 | 포함 모델 |
|-----------|------|----------|
| **표준 Flavor** | 기본 제공 MLflow Flavor를 사용합니다 | sklearn, xgboost, lightgbm, spark |
| **커스텀 PyFunc** | `mlflow.pyfunc.PythonModel`을 상속하여 자유로운 추론 로직을 구현합니다 | 전처리/후처리, 멀티모델 앙상블 등 |
| **외부 모델** | 외부 LLM 제공사를 프록시합니다 | OpenAI, Anthropic, Cohere |

| 모델 유형 | 특징 | 적합한 경우 |
|-----------|------|------------|
| **표준 MLflow Flavor** | `mlflow.sklearn`, `mlflow.xgboost` 등으로 저장한 모델 | 단순 predict 호출로 충분한 경우 |
| **커스텀 PyFunc** | `mlflow.pyfunc.PythonModel`을 상속하여 자유로운 추론 로직 구현 | 전후처리, 앙상블, 비즈니스 로직이 필요한 경우 |
| **외부 모델 (External Models)** | OpenAI, Anthropic 등 외부 API를 Databricks 엔드포인트로 프록시 | 외부 LLM API에 거버넌스/레이트 리밋을 적용할 때 |

---

## 모델 시그니처 정의 (Model Signature)

**모델 시그니처 (Model Signature)** 는 모델의 입력/출력 스키마를 명시적으로 정의합니다. 시그니처가 있으면 서빙 시 입력 데이터 유효성 검사가 자동으로 수행되어 오류를 조기에 감지할 수 있습니다.

```python
import mlflow
from mlflow.models import ModelSignature, infer_signature
from mlflow.types.schema import Schema, ColSpec, ParamSchema, ParamSpec

# 방법 1: 학습 데이터에서 자동 추론 (권장)
import pandas as pd
import numpy as np

X_sample = pd.DataFrame({
    "amount": [50000.0, 25.0],
    "merchant_category": ["online_retail", "grocery"],
    "hour": [3, 12],
    "is_international": [1, 0]
})
y_sample = pd.DataFrame({
    "fraud_probability": [0.85, 0.02],
    "is_fraud": [1, 0]
})

# 입력/출력 데이터로 시그니처 자동 추론
signature = infer_signature(X_sample, y_sample)
print(signature)
# inputs: [amount: double, merchant_category: string, hour: long, is_international: long]
# outputs: [fraud_probability: double, is_fraud: long]
```

```python
# 방법 2: 수동으로 시그니처 정의 (정밀한 제어가 필요할 때)
signature = ModelSignature(
    inputs=Schema([
        ColSpec("double", "amount"),
        ColSpec("string", "merchant_category"),
        ColSpec("long", "hour"),
        ColSpec("integer", "is_international")
    ]),
    outputs=Schema([
        ColSpec("double", "fraud_probability"),
        ColSpec("integer", "is_fraud"),
        ColSpec("string", "risk_level")
    ]),
    # params: 추론 시 동적으로 변경할 수 있는 파라미터 (선택)
    params=ParamSchema([
        ParamSpec("threshold", "double", 0.7)
    ])
)
```

```python
# 방법 3: params를 활용한 동적 추론 파라미터
class FraudModelWithParams(mlflow.pyfunc.PythonModel):
    def load_context(self, context):
        import joblib
        self.model = joblib.load(context.artifacts["model_path"])
        self.scaler = joblib.load(context.artifacts["scaler_path"])

    def predict(self, context, model_input, params=None):
        # params에서 임계값을 런타임에 받음 (기본값 0.7)
        threshold = (params or {}).get("threshold", 0.7)

        scaled_input = self.scaler.transform(model_input)
        probabilities = self.model.predict_proba(scaled_input)[:, 1]

        return pd.DataFrame({
            "fraud_probability": probabilities,
            "is_fraud": (probabilities >= threshold).astype(int)
        })

# params가 포함된 시그니처 생성
X_sample = X_test[:5]
predictions = FraudModelWithParams().predict(None, X_sample)
signature = infer_signature(
    X_sample,
    predictions,
    params={"threshold": 0.7}   # 기본 파라미터 값
)
```

> **시그니처 없이 배포하면** 입력 형식 오류가 서빙 환경에서 런타임에 발생합니다. `input_example`과 함께 `signature`를 항상 명시하는 것을 권장합니다.

---

## PyFunc 커스텀 모델 작성법

`mlflow.pyfunc.PythonModel`을 상속하면 `load_context()`와 `predict()` 메서드를 직접 구현하여 자유로운 추론 로직을 만들 수 있습니다.

### 기본 구조

```python
import mlflow.pyfunc
import pandas as pd
import numpy as np

class FraudDetectionModel(mlflow.pyfunc.PythonModel):
    """전처리 + 모델 추론 + 후처리를 하나로 묶는 커스텀 모델"""

    def load_context(self, context):
        """모델 로드 시 한 번 실행 (무거운 초기화 작업)"""
        import joblib

        # artifacts에서 모델과 전처리기 로드
        self.model = joblib.load(context.artifacts["model_path"])
        self.scaler = joblib.load(context.artifacts["scaler_path"])
        self.threshold = 0.7  # 커스텀 임계값

    def predict(self, context, model_input, params=None):
        """추론 요청마다 실행되는 메서드"""
        # 1. 전처리
        scaled_input = self.scaler.transform(model_input)

        # 2. 예측 확률 계산
        probabilities = self.model.predict_proba(scaled_input)[:, 1]

        # 3. 후처리 (커스텀 임계값 + 리스크 등급)
        results = pd.DataFrame({
            "fraud_probability": probabilities,
            "is_fraud": (probabilities >= self.threshold).astype(int),
            "risk_level": pd.cut(
                probabilities,
                bins=[0, 0.3, 0.7, 1.0],
                labels=["LOW", "MEDIUM", "HIGH"]
            )
        })

        return results
```

### 커스텀 모델 저장 및 등록

```python
import joblib
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.preprocessing import StandardScaler

# 모델과 스케일러 학습
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
model = GradientBoostingClassifier(n_estimators=200)
model.fit(X_train_scaled, y_train)

# 아티팩트 저장
joblib.dump(model, "/tmp/model.joblib")
joblib.dump(scaler, "/tmp/scaler.joblib")

# 시그니처 자동 추론
sample_input = pd.DataFrame(X_test[:3], columns=feature_cols)
sample_output = FraudDetectionModel().predict(None, sample_input)
signature = infer_signature(sample_input, sample_output)

# 커스텀 모델을 MLflow에 로깅
with mlflow.start_run(run_name="fraud-custom-pyfunc"):
    model_info = mlflow.pyfunc.log_model(
        artifact_path="fraud_model",
        python_model=FraudDetectionModel(),
        artifacts={
            "model_path": "/tmp/model.joblib",
            "scaler_path": "/tmp/scaler.joblib"
        },
        conda_env={
            "dependencies": [
                "python=3.10",
                {"pip": ["scikit-learn==1.4.0", "pandas>=2.0", "numpy>=1.24"]}
            ]
        },
        signature=signature,
        input_example=sample_input,
        registered_model_name="catalog.schema.fraud_detection_custom"
    )
```

> **의존성 관리 주의**: `conda_env` 또는 `pip_requirements`를 명시하지 않으면, 로컬 환경의 패키지 버전이 서빙 환경과 달라 오류가 발생할 수 있습니다. 반드시 학습에 사용한 주요 라이브러리 버전을 명시하십시오.

---

## 실전 패턴

### 패턴 1: 앙상블 모델 (Ensemble)

여러 모델의 예측을 결합하여 최종 예측을 만드는 패턴입니다. 각 모델을 개별 artifact로 저장하고 `load_context()`에서 한꺼번에 로드합니다.

```python
class EnsembleModel(mlflow.pyfunc.PythonModel):
    """XGBoost + LightGBM + Logistic Regression 앙상블"""

    def load_context(self, context):
        import joblib
        self.models = {
            "xgb": joblib.load(context.artifacts["xgb_model"]),
            "lgbm": joblib.load(context.artifacts["lgbm_model"]),
            "lr": joblib.load(context.artifacts["lr_model"]),
        }
        # 각 모델의 가중치 (검증 세트 성능 기반)
        self.weights = {"xgb": 0.5, "lgbm": 0.35, "lr": 0.15}

    def predict(self, context, model_input, params=None):
        # 각 모델로 예측 후 가중 평균
        weighted_pred = sum(
            self.weights[name] * model.predict_proba(model_input)[:, 1]
            for name, model in self.models.items()
        )

        return pd.DataFrame({
            "ensemble_probability": weighted_pred,
            "prediction": (weighted_pred >= 0.5).astype(int)
        })
```

### 패턴 2: 파인튜닝 임베딩 모델 (Fine-tuned Embedding)

Hugging Face 모델을 파인튜닝한 후 커스텀 전처리와 함께 배포하는 패턴입니다.

```python
class FineTunedEmbeddingModel(mlflow.pyfunc.PythonModel):
    """파인튜닝된 sentence-transformers 모델"""

    def load_context(self, context):
        from sentence_transformers import SentenceTransformer

        # 파인튜닝된 모델 경로에서 로드
        model_dir = context.artifacts["finetuned_model_dir"]
        self.model = SentenceTransformer(model_dir)
        self.model.eval()

    def predict(self, context, model_input, params=None):
        # 입력이 DataFrame인 경우 텍스트 컬럼 추출
        if isinstance(model_input, pd.DataFrame):
            texts = model_input["text"].tolist()
        else:
            texts = list(model_input)

        # 임베딩 생성 (배치 처리)
        batch_size = (params or {}).get("batch_size", 32)
        embeddings = self.model.encode(
            texts,
            batch_size=batch_size,
            normalize_embeddings=True
        )

        return pd.DataFrame(
            embeddings,
            columns=[f"dim_{i}" for i in range(embeddings.shape[1])]
        )
```

### 패턴 3: 비즈니스 룰 결합 모델

ML 예측에 규칙 기반 로직을 결합하는 패턴입니다.

```python
class CreditScoringModel(mlflow.pyfunc.PythonModel):
    """ML 예측 + 하드 룰(Hard Rule)을 결합한 신용 평가 모델"""

    def load_context(self, context):
        import joblib
        self.ml_model = joblib.load(context.artifacts["ml_model"])

    def _apply_hard_rules(self, df, ml_score):
        """비즈니스 룰로 ML 점수를 오버라이드"""
        final_score = ml_score.copy()

        # 하드 룰: 연체 이력이 있으면 무조건 거절
        final_score[df["delinquency_count"] > 0] = 0.0

        # 하드 룰: 신용 기간 1년 미만은 최대 0.4로 제한
        new_customers = df["credit_age_months"] < 12
        final_score[new_customers] = final_score[new_customers].clip(upper=0.4)

        return final_score

    def predict(self, context, model_input, params=None):
        ml_score = self.ml_model.predict_proba(model_input)[:, 1]
        final_score = self._apply_hard_rules(model_input, ml_score)

        return pd.DataFrame({
            "ml_score": ml_score,
            "final_score": final_score,
            "approved": (final_score >= 0.5).astype(int)
        })
```

---

## 엔드포인트 생성

### 방법 1: SDK (MLflow Deployments)

```python
import mlflow.deployments

client = mlflow.deployments.get_deploy_client("databricks")

# 엔드포인트 생성
endpoint = client.create_endpoint(
    name="fraud-detection-v2",
    config={
        "served_entities": [{
            "entity_name": "catalog.schema.fraud_detection_custom",
            "entity_version": "3",
            "workload_size": "Small",           # Small / Medium / Large
            "scale_to_zero_enabled": True,      # 트래픽 없을 때 0으로 축소
            "workload_type": "CPU"              # CPU 또는 GPU_SMALL / GPU_MEDIUM / GPU_LARGE
        }],
        "auto_capture_config": {
            "catalog_name": "catalog",
            "schema_name": "schema",
            "table_name_prefix": "fraud_endpoint"
        }
    }
)
```

### 방법 2: Databricks SDK

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    EndpointCoreConfigInput,
    ServedEntityInput,
    AutoCaptureConfigInput
)

w = WorkspaceClient()

# 엔드포인트 생성
endpoint = w.serving_endpoints.create(
    name="fraud-detection-v2",
    config=EndpointCoreConfigInput(
        served_entities=[
            ServedEntityInput(
                entity_name="catalog.schema.fraud_detection_custom",
                entity_version="3",
                workload_size="Small",
                scale_to_zero_enabled=True,
                workload_type="CPU"
            )
        ],
        auto_capture_config=AutoCaptureConfigInput(
            catalog_name="catalog",
            schema_name="schema",
            table_name_prefix="fraud_endpoint",
            enabled=True
        )
    )
)

# 엔드포인트 준비 대기
endpoint = w.serving_endpoints.wait_get_serving_endpoint_not_updating(
    name="fraud-detection-v2"
)
print(f"Endpoint state: {endpoint.state.ready}")
```

### 방법 3: REST API

```bash
curl -X POST "https://<workspace-url>/api/2.0/serving-endpoints" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "fraud-detection-v2",
    "config": {
      "served_entities": [{
        "entity_name": "catalog.schema.fraud_detection_custom",
        "entity_version": "3",
        "workload_size": "Small",
        "scale_to_zero_enabled": true
      }]
    }
  }'
```

### 워크로드 크기 옵션

| 워크로드 크기 | 동시 요청 수 (대략) | 적합한 사용 사례 |
|--------------|---------------------|----------------|
| **Small** | 최대 4 동시 요청 | 개발/테스트, 저트래픽 |
| **Medium** | 최대 16 동시 요청 | 중간 트래픽 프로덕션 |
| **Large** | 최대 64 동시 요청 | 고트래픽 프로덕션 |

---

## GPU 서빙 설정

딥러닝 모델이나 대규모 모델은 GPU 서빙이 필요합니다.

| GPU 워크로드 타입 | GPU 종류 | 적합한 모델 |
|------------------|---------|------------|
| `GPU_SMALL` | T4 (16GB) | 소규모 딥러닝 모델, 임베딩 모델 |
| `GPU_MEDIUM` | A10G (24GB) | 중규모 트랜스포머, 이미지 모델 |
| `GPU_LARGE` | A100 (80GB) | 대규모 LLM, 파인튜닝된 Foundation Model |

```python
# GPU 엔드포인트 생성 예시
endpoint = client.create_endpoint(
    name="embedding-model-gpu",
    config={
        "served_entities": [{
            "entity_name": "catalog.schema.sentence_transformer",
            "entity_version": "1",
            "workload_size": "Small",
            "workload_type": "GPU_SMALL",       # GPU 타입 지정
            "scale_to_zero_enabled": True
        }]
    }
)
```

---

## 트래픽 분할 및 A/B 테스트

하나의 엔드포인트에 여러 모델 버전을 동시에 배포하고, 트래픽 비율을 조절하여 **A/B 테스트** 를 수행할 수 있습니다.

| 구성 요소 | 트래픽 비율 | 설명 |
|-----------|-----------|------|
| 클라이언트 요청 | 100% | 엔드포인트(fraud-detection)로 전송됩니다 |
| 모델 v3 (현재 프로덕션) | 80% | 현재 안정된 프로덕션 모델입니다 |
| 모델 v4 (챌린저) | 20% | 성능을 검증 중인 새 버전 모델입니다 |
| 응답 | - | 두 모델의 응답이 클라이언트에 반환됩니다 |

```python
# 트래픽 분할 설정
client = mlflow.deployments.get_deploy_client("databricks")

client.update_endpoint(
    endpoint="fraud-detection-v2",
    config={
        "served_entities": [
            {
                "entity_name": "catalog.schema.fraud_detection_custom",
                "entity_version": "3",
                "workload_size": "Small",
                "scale_to_zero_enabled": False
            },
            {
                "entity_name": "catalog.schema.fraud_detection_custom",
                "entity_version": "4",
                "workload_size": "Small",
                "scale_to_zero_enabled": False
            }
        ],
        "traffic_config": {
            "routes": [
                {"served_model_name": "fraud_detection_custom-3", "traffic_percentage": 80},
                {"served_model_name": "fraud_detection_custom-4", "traffic_percentage": 20}
            ]
        }
    }
)
```

> **블루-그린 배포**: 새 모델 버전을 100% 트래픽으로 전환하면서도 이전 버전을 유지하는 방식입니다. 문제 발생 시 즉시 이전 버전으로 롤백할 수 있습니다.

---

## 모니터링 (Monitoring)

### Inference Table (추론 로그 테이블)

**Inference Table** 은 엔드포인트의 모든 요청/응답을 Unity Catalog 테이블에 자동으로 저장합니다. 모델 드리프트 감지, 디버깅, 감사 로그 목적으로 활용합니다.

```python
# Inference Table 활성화 (엔드포인트 생성 또는 업데이트 시)
config = {
    "served_entities": [...],
    "auto_capture_config": {
        "catalog_name": "main",
        "schema_name": "serving_logs",
        "table_name_prefix": "fraud_endpoint",   # 테이블명: fraud_endpoint_payload
        "enabled": True
    }
}
```

```sql
-- Inference Table 쿼리 예시
-- 테이블명: {catalog}.{schema}.{prefix}_payload
SELECT
    timestamp,
    request_id,
    databricks_request_id,
    -- 요청 데이터 파싱
    request:dataframe_records[0].amount::double AS amount,
    request:dataframe_records[0].merchant_category::string AS merchant_category,
    -- 응답 데이터 파싱
    response:predictions[0].fraud_probability::double AS fraud_probability,
    response:predictions[0].is_fraud::int AS is_fraud,
    -- 메타데이터
    status_code,
    execution_duration_ms
FROM main.serving_logs.fraud_endpoint_payload
WHERE timestamp >= current_date() - interval 7 days
ORDER BY timestamp DESC;
```

```python
# Python에서 Inference Table 조회
from databricks.connect import DatabricksSession

spark = DatabricksSession.builder.getOrCreate()

# 최근 1시간 요청 분석
df = spark.sql("""
    SELECT
        date_trunc('minute', timestamp) AS minute,
        count(*) AS request_count,
        avg(execution_duration_ms) AS avg_latency_ms,
        percentile_approx(execution_duration_ms, 0.95) AS p95_latency_ms,
        sum(CASE WHEN status_code != 200 THEN 1 ELSE 0 END) AS error_count
    FROM main.serving_logs.fraud_endpoint_payload
    WHERE timestamp >= now() - interval 1 hour
    GROUP BY 1
    ORDER BY 1
""")
df.display()
```

### 엔드포인트 메트릭 확인

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 엔드포인트 상태 조회
endpoint = w.serving_endpoints.get(name="fraud-detection-v2")
print(f"State: {endpoint.state.ready}")
print(f"Config update: {endpoint.state.config_update}")

# 이벤트 로그 조회 (디버깅용)
for event in w.serving_endpoints.list_serving_endpoint_events(name="fraud-detection-v2"):
    print(f"[{event.timestamp}] {event.type}: {event.message}")
```

---

## 콜드 스타트 최적화

**콜드 스타트 (Cold Start)** 는 `scale_to_zero_enabled=True` 설정 시 트래픽이 없다가 첫 요청이 들어올 때 컨테이너를 다시 시작하는 지연 시간입니다. 일반적으로 30초~5분이 소요됩니다.

| 전략 | 방법 | 트레이드오프 |
|------|------|-------------|
| **Scale-to-zero 비활성화** | `scale_to_zero_enabled=False` | 비용 증가, 콜드 스타트 없음 |
| **Provisioned Concurrency** | 최소 인스턴스 수 설정 | 비용 증가, 항상 warm 상태 |
| **모델 경량화** | ONNX 변환, 양자화 (Quantization) | 로드 시간 단축 |
| **의존성 최소화** | 불필요한 패키지 제거 | 컨테이너 빌드/로드 속도 향상 |

```python
# Provisioned Concurrency 설정 (최소 인스턴스 보장)
endpoint = client.create_endpoint(
    name="fraud-detection-prod",
    config={
        "served_entities": [{
            "entity_name": "catalog.schema.fraud_detection_custom",
            "entity_version": "3",
            "workload_size": "Small",
            "scale_to_zero_enabled": False,     # 항상 최소 1개 인스턴스 유지
            "workload_type": "CPU"
        }]
    }
)
```

> **load_context() 최적화**: 무거운 초기화(모델 로드, 토크나이저 초기화 등)는 반드시 `load_context()`에서 수행하고 `predict()`에서는 수행하지 마십시오. `load_context()`는 인스턴스 시작 시 한 번만 실행되지만, `predict()`는 매 요청마다 실행됩니다.

---

## 비용 관리

커스텀 모델 서빙은 **인스턴스 가동 시간** 기준으로 과금됩니다. Foundation Model API의 토큰 기반 과금과 다른 비용 구조를 갖습니다.

| 비용 요소 | 설명 | 절감 방법 |
|-----------|------|---------|
| **인스턴스 크기** | Small < Medium < Large 순으로 과금 | 실제 트래픽에 맞는 크기 선택 |
| **Scale-to-zero** | 트래픽 없을 때 인스턴스 종료 | 개발/비업무 시간에 활성화 |
| **GPU 비용** | CPU 대비 5~10배 비용 | 실제 GPU 필요 여부 사전 검증 |
| **Inference Table 저장** | Delta Lake 스토리지 비용 | 보존 기간 설정 (`TBLPROPERTIES`) |

```sql
-- Inference Table 데이터 보존 기간 설정 (90일 후 자동 삭제)
ALTER TABLE main.serving_logs.fraud_endpoint_payload
SET TBLPROPERTIES (
    'delta.deletedFileRetentionDuration' = 'interval 90 days'
);
```

```python
# 비용 절감: 개발 환경은 scale_to_zero 활성화, 프로덕션은 비활성화
def create_endpoint_by_env(env: str, model_name: str, version: str):
    is_prod = env == "production"
    return client.create_endpoint(
        name=f"fraud-detection-{env}",
        config={
            "served_entities": [{
                "entity_name": model_name,
                "entity_version": version,
                "workload_size": "Medium" if is_prod else "Small",
                "scale_to_zero_enabled": not is_prod,
                "workload_type": "CPU"
            }]
        }
    )
```

---

## 실습: 모델 서빙 엔드포인트 생성 및 호출

### 엔드포인트 호출 방법

```python
import requests
import json

# 방법 1: requests 라이브러리
workspace_url = "https://<workspace-url>"
token = dbutils.notebook.entry_point.getDbutils().notebook().getContext().apiToken().get()

url = f"{workspace_url}/serving-endpoints/fraud-detection-v2/invocations"
headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json"
}

# DataFrame 레코드 형식
payload = {
    "dataframe_records": [
        {"amount": 50000, "merchant_category": "online_retail", "hour": 3, "is_international": 1},
        {"amount": 25, "merchant_category": "grocery", "hour": 12, "is_international": 0}
    ]
}

response = requests.post(url, json=payload, headers=headers)
predictions = response.json()
print(json.dumps(predictions, indent=2))
```

```python
# 방법 2: Databricks SDK
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

response = w.serving_endpoints.query(
    name="fraud-detection-v2",
    dataframe_records=[
        {"amount": 50000, "merchant_category": "online_retail", "hour": 3, "is_international": 1}
    ]
)
print(response.predictions)
```

```python
# 방법 3: MLflow Deployments Client
client = mlflow.deployments.get_deploy_client("databricks")

response = client.predict(
    endpoint="fraud-detection-v2",
    inputs={
        "dataframe_records": [
            {"amount": 50000, "merchant_category": "online_retail", "hour": 3}
        ]
    }
)
print(response)
```

---

## 베스트 프랙티스와 흔한 실수

### 베스트 프랙티스

| 항목 | 권장 사항 |
|------|---------|
| **시그니처 필수 정의** | `infer_signature()`로 항상 입출력 스키마를 명시합니다 |
| **의존성 버전 고정** | `scikit-learn==1.4.0`처럼 정확한 버전을 `pip_requirements`에 명시합니다 |
| **load_context 최적화** | 모델 로드 등 무거운 작업은 `load_context()`에서 처리합니다 |
| **input_example 제공** | `log_model()` 시 실제 샘플 데이터를 `input_example`로 제공합니다 |
| **Inference Table 활성화** | 프로덕션에서는 항상 `auto_capture_config`를 활성화합니다 |
| **환경별 scale_to_zero** | 개발은 `True`, 프로덕션은 `False`로 설정합니다 |

### 흔한 실수

| 실수 | 증상 | 해결 방법 |
|------|------|---------|
| **의존성 미명시** | `ModuleNotFoundError` 발생 | `pip_requirements`에 모든 패키지 버전 명시 |
| **모델 크기 초과** | 엔드포인트 `PENDING` 상태에서 멈춤 | 커스텀 모델 크기는 일반적으로 수 GB 이하 권장 |
| **predict()에서 무거운 로드** | 요청마다 수 초 지연 | 초기화 로직을 `load_context()`로 이동 |
| **시그니처 불일치** | `400 Bad Request` 오류 | 요청 형식을 `input_example`과 동일하게 맞춤 |
| **타임아웃 미고려** | 기본 타임아웃(60초) 초과 오류 | 모델 최적화 또는 비동기 처리 패턴 도입 |
| **GPU 없이 GPU 모델 실행** | CUDA 오류 발생 | `workload_type="GPU_SMALL"` 이상으로 설정 |
| **scale_to_zero + 저레이턴시** | 첫 요청 30초~수 분 지연 | 프로덕션에서는 `scale_to_zero_enabled=False` |

---

## 트러블슈팅

### 자주 발생하는 문제와 해결 방법

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| **엔드포인트가 `PENDING` 상태에서 멈춤** | 모델 로드 실패 또는 의존성 문제 | 엔드포인트 이벤트 로그 확인. `conda_env`에 정확한 패키지 버전 명시 |
| **`ModuleNotFoundError`** | 서빙 환경에 필요한 패키지 미설치 | `pip_requirements` 또는 `conda_env`에 모든 의존성 추가 |
| **입력 스키마 불일치** | 요청 데이터의 컬럼명/타입이 모델 시그니처와 다름 | `input_example`과 동일한 형식으로 요청. 모델 시그니처 확인 |
| **Scale-to-zero 후 첫 요청 느림** | Cold Start (컨테이너 부팅 시간) | 프로덕션에서는 `scale_to_zero_enabled=False` 사용 |
| **메모리 부족 (OOM)** | 모델 크기 대비 워크로드 크기가 작음 | `workload_size`를 Medium/Large로 변경 |
| **GPU 할당 실패** | GPU 리소스 부족 | 다른 GPU 타입 시도 또는 워크스페이스 관리자에 문의 |

### 엔드포인트 상태 확인

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 엔드포인트 상태 조회
endpoint = w.serving_endpoints.get(name="fraud-detection-v2")
print(f"State: {endpoint.state.ready}")
print(f"Config update: {endpoint.state.config_update}")

# 이벤트 로그 조회 (디버깅용)
for event in w.serving_endpoints.list_serving_endpoint_events(name="fraud-detection-v2"):
    print(f"[{event.timestamp}] {event.type}: {event.message}")
```

> **프로덕션 배포 체크리스트**:
> - 모델 시그니처(input/output schema)가 올바른지 확인
> - `pip_requirements`에 모든 의존성과 정확한 버전을 명시
> - Scale-to-zero 비활성화 (지연 시간 민감한 경우)
> - Inference Table을 활성화하여 요청/응답 로깅
> - 엔드포인트 모니터링 알림 설정

---

## 정리

| 항목 | 핵심 포인트 |
|------|------------|
| **커스텀 vs FMAPI** | 파인튜닝 모델, 전후처리, 데이터 거버넌스가 필요하면 커스텀 배포를 선택합니다 |
| **모델 시그니처** | `infer_signature()`로 입출력 스키마를 명시하여 런타임 오류를 사전 방지합니다 |
| **커스텀 PyFunc** | `PythonModel`을 상속하여 전처리/후처리/앙상블 등 자유로운 추론 로직을 구현합니다 |
| **엔드포인트 생성** | SDK, REST API, UI 세 가지 방법으로 생성할 수 있습니다 |
| **트래픽 분할** | A/B 테스트와 블루-그린 배포로 안전하게 모델을 교체합니다 |
| **GPU 서빙** | `workload_type`으로 T4/A10G/A100 등 GPU를 선택합니다 |
| **콜드 스타트** | 프로덕션에서는 `scale_to_zero_enabled=False`로 지연 시간을 예측 가능하게 유지합니다 |
| **모니터링** | Inference Table에서 요청/응답을 SQL로 분석하고, 레이턴시와 에러율을 추적합니다 |
| **비용 관리** | 환경별 workload_size와 scale_to_zero 설정으로 불필요한 비용을 절감합니다 |
| **의존성 관리** | `conda_env` 또는 `pip_requirements`에 정확한 버전을 명시하는 것이 핵심입니다 |

---

## 참고 링크

- [Databricks: Deploy custom models](https://docs.databricks.com/aws/en/machine-learning/model-serving/create-manage-serving-endpoints.html)
- [Databricks: Custom Python models (PyFunc)](https://docs.databricks.com/aws/en/machine-learning/model-serving/custom-models.html)
- [Databricks: Model Serving GPU workloads](https://docs.databricks.com/aws/en/machine-learning/model-serving/gpu.html)
- [Databricks: A/B testing with serving endpoints](https://docs.databricks.com/aws/en/machine-learning/model-serving/traffic-config.html)
- [Databricks: Inference Tables](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
- [Databricks: Model Serving scale-to-zero](https://docs.databricks.com/aws/en/machine-learning/model-serving/scale-to-zero.html)
- [MLflow: Custom PyFunc Models](https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html)
- [MLflow: Model Signatures](https://mlflow.org/docs/latest/models.html#model-signature-and-input-example)
