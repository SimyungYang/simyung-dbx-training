# Databricks AutoML 상세

## AutoML이란?

**AutoML(Automated Machine Learning)** 은 데이터를 입력하면 **자동으로 전처리, 모델 선택, 하이퍼파라미터 튜닝, 평가**까지 수행하여 최적의 모델을 찾아주는 기능입니다. ML 전문 지식이 없어도 빠르게 좋은 성능의 모델을 얻을 수 있습니다.

> 💡 **AutoML은 블랙박스가 아닙니다.** Databricks AutoML의 핵심 차별점은 각 시행(trial)의 **전체 코드가 담긴 노트북을 자동 생성**한다는 것입니다. 결과를 검토하고, 필요한 부분을 수정하여 직접 커스터마이즈할 수 있습니다.

---

## 지원하는 문제 유형

| 문제 유형 | API 함수 | 평가 지표 | 설명 |
|----------|---------|----------|------|
| **분류 (Classification)** | `automl.classify()` | F1, Accuracy, ROC-AUC, Log Loss | 이진/다중 클래스 분류 |
| **회귀 (Regression)** | `automl.regress()` | RMSE, MAE, R², MSE | 연속 값 예측 |
| **시계열 예측 (Forecasting)** | `automl.forecast()` | SMAPE, RMSE, MSE, MAE | 시계열 데이터 예측 |

---

## UI에서 AutoML 사용하기

### 실행 순서

1. 좌측 메뉴에서 **Experiments** 클릭
2. **Create AutoML Experiment** 선택
3. 설정 입력:
   - **Dataset**: Delta 테이블 또는 DataFrame 선택
   - **Prediction target**: 예측할 컬럼 선택
   - **Problem type**: 분류 / 회귀 / 시계열 예측 선택
4. **고급 설정** (선택):
   - Timeout (최대 실행 시간)
   - Evaluation metric (평가 지표)
   - Training frameworks (사용할 알고리즘)
5. **Start** 클릭

### UI에서 결과 확인

실행이 완료되면 다음을 확인할 수 있습니다:

| 확인 항목 | 설명 |
|----------|------|
| **리더보드** | 모든 시행의 성능 지표를 비교합니다 |
| **최적 모델** | 가장 좋은 성능의 모델이 하이라이트됩니다 |
| **생성된 노트북** | 각 시행의 전체 코드를 담은 노트북 링크입니다 |
| **피처 중요도** | SHAP 기반 피처 중요도 그래프입니다 |

---

## API에서 AutoML 사용하기

### 분류 (Classification)

```python
from databricks import automl

# 분류 모델 학습
result = automl.classify(
    dataset="catalog.schema.customer_churn",  # Delta 테이블 경로
    target_col="churned",                      # 예측 대상 컬럼
    primary_metric="f1",                       # 최적화할 지표
    timeout_minutes=30,                        # 최대 실행 시간
    max_trials=50,                             # 최대 시행 수
    exclude_cols=["customer_id"],              # 제외할 컬럼
    experiment_dir="/Workspace/Users/user@company.com/automl"  # 실험 저장 경로
)

# 결과 확인
print(f"Best model: {result.best_trial.model_description}")
print(f"F1 Score: {result.best_trial.metrics['val_f1_score']:.4f}")
print(f"Accuracy: {result.best_trial.metrics['val_accuracy_score']:.4f}")
print(f"Model URI: {result.best_trial.model_uri}")

# 최적 모델의 노트북 확인
print(f"Best trial notebook: {result.best_trial.notebook_url}")
```

### 회귀 (Regression)

```python
result = automl.regress(
    dataset="catalog.schema.house_prices",
    target_col="price",
    primary_metric="rmse",
    timeout_minutes=60,
    max_trials=100,
    exclude_cols=["id", "address"]
)

print(f"Best model: {result.best_trial.model_description}")
print(f"RMSE: {result.best_trial.metrics['val_rmse']:.2f}")
print(f"R²: {result.best_trial.metrics['val_r2_score']:.4f}")
```

### 시계열 예측 (Forecasting)

```python
result = automl.forecast(
    dataset="catalog.schema.daily_sales",
    target_col="revenue",
    time_col="date",                    # 시간 컬럼
    identity_col=["store_id"],          # 시계열 식별 컬럼 (멀티 시계열)
    horizon=30,                         # 예측 기간 (30일)
    frequency="D",                      # 데이터 빈도 (일간)
    primary_metric="smape",
    timeout_minutes=60
)

print(f"Best model: {result.best_trial.model_description}")
print(f"SMAPE: {result.best_trial.metrics['val_smape']:.4f}")
```

---

## AutoML 주요 파라미터

| 파라미터 | 설명 | 기본값 |
|---------|------|--------|
| `dataset` | 학습 데이터 (테이블 경로 또는 DataFrame)입니다 | (필수) |
| `target_col` | 예측 대상 컬럼입니다 | (필수) |
| `primary_metric` | 최적화할 평가 지표입니다 | 문제 유형에 따라 자동 |
| `timeout_minutes` | 최대 실행 시간(분)입니다 | 120 |
| `max_trials` | 최대 시행(모델 학습) 수입니다 | None (시간 제한 내 최대) |
| `exclude_cols` | 학습에서 제외할 컬럼 목록입니다 | `[]` |
| `exclude_frameworks` | 제외할 알고리즘 프레임워크입니다 | `[]` |
| `experiment_dir` | MLflow 실험 저장 경로입니다 | 기본 디렉토리 |

---

## 생성된 노트북 활용

AutoML이 생성하는 노트북에는 다음 내용이 포함되어 있습니다:

| 섹션 | 내용 |
|------|------|
| **데이터 전처리** | 결측값 처리, 인코딩, 스케일링 코드 |
| **피처 엔지니어링** | 날짜 분해, 상호작용 피처 등 |
| **모델 학습** | 알고리즘 설정, 하이퍼파라미터 |
| **평가** | 교차 검증, 지표 계산, 혼동 행렬 |
| **SHAP 분석** | 피처 중요도 시각화 |

### 노트북 커스터마이징 워크플로

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | AutoML 실행 | 자동으로 여러 모델을 학습합니다 |
| 2 | 노트북 다운로드 | AutoML이 생성한 노트북을 다운로드합니다 |
| 3 | 코드 검토 및 수정 | 생성된 코드를 검토하고 필요에 따라 수정합니다 |
| 4 | 추가 피처 엔지니어링 | 도메인 지식을 반영하여 피처를 추가합니다 |
| 5 | 하이퍼파라미터 세밀 조정 | 모델 성능을 최적화합니다 |
| 6 | 최종 모델 MLflow 등록 | 최종 모델을 MLflow에 등록합니다 |

```python
# AutoML 생성 노트북에서 가져온 코드를 기반으로 커스터마이징
# (실제 AutoML이 생성하는 코드의 구조 예시)

import mlflow
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.compose import ColumnTransformer
from xgboost import XGBClassifier

# AutoML이 생성한 전처리 파이프라인 (수정 가능)
preprocessor = ColumnTransformer(
    transformers=[
        ("num", StandardScaler(), numerical_cols),
        ("cat", OneHotEncoder(handle_unknown="ignore"), categorical_cols),
    ]
)

# AutoML이 찾은 최적 하이퍼파라미터 (세밀 조정 가능)
model = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", XGBClassifier(
        n_estimators=200,       # AutoML 결과 기반
        max_depth=6,            # 직접 조정 가능
        learning_rate=0.1,
        min_child_weight=3,
        subsample=0.8
    ))
])

# MLflow 자동 로깅
with mlflow.start_run(run_name="customized-automl-model"):
    model.fit(X_train, y_train)
    mlflow.sklearn.log_model(model, "model",
        registered_model_name="catalog.schema.churn_model")
```

---

## MLflow 자동 연동

AutoML의 모든 시행(trial)은 **MLflow에 자동으로 로깅**됩니다.

| 자동 기록 항목 | 설명 |
|-------------|------|
| **파라미터** | 알고리즘, 하이퍼파라미터 |
| **지표** | 학습/검증 성능 지표 |
| **모델 아티팩트** | 학습된 모델 파일 |
| **피처 중요도** | SHAP 기반 중요도 그래프 |
| **노트북** | 해당 시행의 전체 코드 노트북 |

```python
# MLflow에서 AutoML 실험 결과 조회
import mlflow

experiment = mlflow.get_experiment_by_name("/Workspace/Users/user@company.com/automl")
runs = mlflow.search_runs(
    experiment_ids=[experiment.experiment_id],
    order_by=["metrics.val_f1_score DESC"],
    max_results=10
)

# 상위 10개 모델 비교
print(runs[["run_id", "params.model_type", "metrics.val_f1_score", "metrics.val_accuracy_score"]])
```

---

## AutoML 시행에서 사용되는 알고리즘

| 알고리즘 | 문제 유형 | 특징 |
|---------|----------|------|
| **XGBoost** | 분류, 회귀 | 정형 데이터에서 최고 성능, Gradient Boosting |
| **LightGBM** | 분류, 회귀 | 대용량 데이터에서 빠른 학습 |
| **sklearn (RF, LR)** | 분류, 회귀 | Random Forest, Logistic/Linear Regression |
| **Prophet** | 시계열 예측 | Meta의 시계열 예측 라이브러리 |
| **ARIMA** | 시계열 예측 | 전통적 시계열 분석 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **AutoML** | 데이터를 주면 자동으로 전처리, 모델 선택, 튜닝을 수행합니다 |
| **3가지 문제 유형** | 분류(classify), 회귀(regress), 시계열 예측(forecast)을 지원합니다 |
| **노트북 생성** | 각 시행의 전체 코드가 담긴 노트북이 자동 생성되어 커스터마이즈할 수 있습니다 |
| **MLflow 연동** | 모든 시행이 MLflow에 자동 로깅되어 비교, 추적, 배포가 가능합니다 |
| **SHAP 분석** | 피처 중요도를 자동으로 계산하여 모델 해석성을 제공합니다 |

---

## 참고 링크

- [Databricks: AutoML](https://docs.databricks.com/aws/en/machine-learning/automl/)
- [Databricks: AutoML API reference](https://docs.databricks.com/aws/en/machine-learning/automl/automl-api-reference.html)
- [Databricks: AutoML classification](https://docs.databricks.com/aws/en/machine-learning/automl/train-ml-model-automl-api.html)
- [Databricks: AutoML forecasting](https://docs.databricks.com/aws/en/machine-learning/automl/train-forecast-automl.html)
- [Azure Databricks: AutoML](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/)
