# Databricks AutoML 상세

## AutoML이란?

**AutoML(Automated Machine Learning)**은 데이터를 입력하면 **자동으로 전처리, 모델 선택, 하이퍼파라미터 튜닝, 평가** 까지 수행하여 최적의 모델을 찾아주는 기능입니다. ML 전문 지식이 없어도 빠르게 좋은 성능의 모델을 얻을 수 있습니다.

> 💡 **AutoML은 블랙박스가 아닙니다.**Databricks AutoML의 핵심 차별점은 각 시행(trial)의 **전체 코드가 담긴 노트북을 자동 생성** 한다는 것입니다. 결과를 검토하고, 필요한 부분을 수정하여 직접 커스터마이즈할 수 있습니다.

---

## 지원하는 문제 유형

| 문제 유형 | API 함수 | 평가 지표 | 설명 |
|----------|---------|----------|------|
| **분류 (Classification)**| `automl.classify()` | F1, Accuracy, ROC-AUC, Log Loss | 이진/다중 클래스 분류 |
| **회귀 (Regression)**| `automl.regress()` | RMSE, MAE, R², MSE | 연속 값 예측 |
| **시계열 예측 (Forecasting)**| `automl.forecast()` | SMAPE, RMSE, MSE, MAE | 시계열 데이터 예측 |

---

## UI에서 AutoML 사용하기

### 실행 순서

1. 좌측 메뉴에서 **Experiments** 클릭
2. **Create AutoML Experiment** 선택
3. 설정 입력:
   - **Dataset**: Delta 테이블 또는 DataFrame 선택
   - **Prediction target**: 예측할 컬럼 선택
   - **Problem type**: 분류 / 회귀 / 시계열 예측 선택
4. **고급 설정**(선택):
   - Timeout (최대 실행 시간)
   - Evaluation metric (평가 지표)
   - Training frameworks (사용할 알고리즘)
5. **Start** 클릭

### UI에서 결과 확인

실행이 완료되면 다음을 확인할 수 있습니다:

| 확인 항목 | 설명 |
|----------|------|
| **리더보드**| 모든 시행의 성능 지표를 비교합니다 |
| **최적 모델**| 가장 좋은 성능의 모델이 하이라이트됩니다 |
| **생성된 노트북**| 각 시행의 전체 코드를 담은 노트북 링크입니다 |
| **피처 중요도**| SHAP 기반 피처 중요도 그래프입니다 |

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
| **데이터 전처리**| 결측값 처리, 인코딩, 스케일링 코드 |
| **피처 엔지니어링**| 날짜 분해, 상호작용 피처 등 |
| **모델 학습**| 알고리즘 설정, 하이퍼파라미터 |
| **평가**| 교차 검증, 지표 계산, 혼동 행렬 |
| **SHAP 분석**| 피처 중요도 시각화 |

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

AutoML의 모든 시행(trial)은 **MLflow에 자동으로 로깅** 됩니다.

| 자동 기록 항목 | 설명 |
|-------------|------|
| **파라미터**| 알고리즘, 하이퍼파라미터 |
| **지표**| 학습/검증 성능 지표 |
| **모델 아티팩트**| 학습된 모델 파일 |
| **피처 중요도**| SHAP 기반 중요도 그래프 |
| **노트북**| 해당 시행의 전체 코드 노트북 |

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
| **XGBoost**| 분류, 회귀 | 정형 데이터에서 최고 성능, Gradient Boosting |
| **LightGBM**| 분류, 회귀 | 대용량 데이터에서 빠른 학습 |
| **sklearn (RF, LR)**| 분류, 회귀 | Random Forest, Logistic/Linear Regression |
| **Prophet**| 시계열 예측 | Meta의 시계열 예측 라이브러리 |
| **ARIMA**| 시계열 예측 | 전통적 시계열 분석 |

---

## 현업 사례: AutoML 결과를 그대로 프로덕션에 올리면 안 되는 이유

> 🔥 **AutoML의 가장 큰 오해: "자동이니까 바로 배포해도 되겠지"**

AutoML은 놀라울 정도로 좋은 성능을 보여주지만, 그 결과를 **검증 없이 프로덕션에 배포하는 것은 위험** 합니다. 현업에서 자주 보는 실수 패턴을 살펴보겠습니다.

### AutoML 결과를 바로 배포했을 때 벌어지는 일

```
1주차: AutoML 실행, F1 = 0.95 달성! 🎉
  - "와, 이 정도면 바로 프로덕션에 올리자"
  - 모델 서빙 엔드포인트 생성, API 연동

2주차: 비즈니스팀에서 이상한 보고
  - "예측 결과가 너무 편향돼요. VIP 고객을 전부 이탈 예측으로 분류해요"
  - 원인: AutoML이 customer_tier 컬럼을 피처로 사용했는데,
    학습 데이터에서 VIP 이탈 비율이 높았음 (표본 편향)

3주차: 더 심각한 문제 발견
  - "주말 데이터가 들어오면 예측이 완전히 틀려요"
  - 원인: 학습 데이터가 평일 데이터 위주였음
  - AutoML은 데이터의 편향을 알려주지 않음

4주차: 성능 급락
  - F1이 0.95 → 0.68로 하락
  - 원인: Data Drift (실제 서비스 데이터의 분포가 학습 데이터와 달라짐)
  - AutoML은 학습 데이터 내에서만 최적화했을 뿐,
    프로덕션 환경의 데이터 변화는 고려하지 않음
```

### AutoML이 해주지 않는 것들

| 단계 | AutoML이 해주는 것 | 사람이 해야 하는 것 |
|------|-------------------|-------------------|
| **데이터 이해**| 기본 통계 | 비즈니스 맥락 이해, 편향 확인 |
| **피처 선택**| 자동 선택 | 도메인 지식 기반 피처 검증 (data leakage 확인) |
| **모델 학습**| 최적 알고리즘/하이퍼파라미터 탐색 | 결과 해석, 비즈니스 로직 확인 |
| **평가**| 수치 메트릭 (F1, RMSE) | 슬라이스별 성능 확인 (성별, 연령대별 편향) |
| **배포**| MLflow 등록 | A/B 테스트, 모니터링, 롤백 계획 |
| **운영**| - | Data Drift 감지, 주기적 재학습 |

---

## AutoML이 정말 유용한 경우: 베이스라인(Baseline) 설정

> 💡 **AutoML의 진짜 가치는 "프로덕션 모델"이 아니라 "베이스라인"입니다.**

현업에서 AutoML은 다음 상황에서 가장 빛납니다.

### 유용한 시나리오 1: "이 데이터로 예측이 가능한가?" 타당성 검증

```python
# 프로젝트 시작 1일차에 실행
# "고객 이탈을 예측할 수 있을까?"에 대한 빠른 답변

result = automl.classify(
    dataset="catalog.schema.customer_data",
    target_col="churned",
    timeout_minutes=30,  # 30분이면 충분
    max_trials=20
)

# 결과 해석:
# F1 > 0.7 → "예측 가능성 있음, 프로젝트 진행하자"
# F1 < 0.5 → "현재 피처로는 어려움, 데이터를 더 모으거나 피처를 추가해야"
print(f"Best F1: {result.best_trial.metrics['val_f1_score']:.4f}")

# SHAP 분석으로 어떤 피처가 중요한지 즉시 파악
# → 도메인 전문가와 "이 피처가 말이 되나요?" 논의의 시작점
```

### 유용한 시나리오 2: 수동 모델 개선의 기준선

```
AutoML 결과: F1 = 0.85 (30분 소요)

이제 데이터 사이언티스트가 수동으로 개선:
- 피처 엔지니어링 추가 → F1 = 0.88 (2일 소요)
- 앙상블 기법 적용 → F1 = 0.91 (3일 소요)
- 도메인 지식 기반 피처 → F1 = 0.93 (1주 소요)

AutoML이 없었다면:
- "기준이 뭔지 몰라서 개선이 된 건지 확인 어려움"
- "경영진에게 '3%p 개선했습니다'라고 수치로 보고하기 어려움"
```

### 유용한 시나리오 3: 비전문가가 빠르게 결과를 내야 할 때

```python
# 데이터 분석가가 "다음 주까지 매출 예측 해주세요"라는 요청을 받았을 때
# ML 전문가가 아니어도 시작할 수 있음

result = automl.forecast(
    dataset="catalog.schema.daily_sales",
    target_col="revenue",
    time_col="date",
    identity_col=["store_id"],
    horizon=30,
    timeout_minutes=60
)

# AutoML이 생성한 노트북을 ML 전문가에게 전달
# "여기서부터 개선해주세요"
print(f"Best trial notebook: {result.best_trial.notebook_url}")
```

---

## 생성된 노트북을 커스터마이징하는 실전 패턴

AutoML이 생성하는 노트북은 **수정 가능한 출발점** 입니다. 현업에서 가장 많이 커스터마이징하는 부분을 소개합니다.

### 커스터마이징 포인트 1: 피처 엔지니어링 추가

```python
# AutoML 노트북에서 가져온 코드 + 도메인 지식 피처 추가

# AutoML이 생성한 기본 전처리 (그대로 유지)
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OneHotEncoder

# === 여기서부터 커스터마이징 ===
# 도메인 지식 기반 피처 추가
def add_domain_features(df):
    # 고객 활동 관련 파생 피처
    df['days_since_last_order'] = (
        pd.Timestamp.now() - df['last_order_date']).dt.days
    df['avg_order_value'] = df['total_spent'] / df['order_count'].clip(lower=1)
    df['order_frequency'] = df['order_count'] / df['tenure_months'].clip(lower=1)

    # 이상치 제거 (AutoML은 이것을 안 해줌)
    df = df[df['avg_order_value'] < df['avg_order_value'].quantile(0.99)]

    return df

# AutoML 결과에서 데이터 로드 부분을 수정
train_df = add_domain_features(train_df)
```

### 커스터마이징 포인트 2: 데이터 분할 전략 변경

```python
# AutoML 기본: 랜덤 분할 (train/validation/test)
# 현업에서는 시간 기반 분할이 더 현실적

# ❌ AutoML 기본 (랜덤 분할)
from sklearn.model_selection import train_test_split
X_train, X_test = train_test_split(X, test_size=0.2, random_state=42)

# ✅ 시간 기반 분할로 변경 (더 현실적인 평가)
train_mask = df['order_date'] < '2025-01-01'
test_mask = df['order_date'] >= '2025-01-01'
X_train, y_train = df[train_mask][features], df[train_mask][target]
X_test, y_test = df[test_mask][features], df[test_mask][target]

# 이렇게 하면 "과거 데이터로 학습 → 미래 데이터 예측" 시뮬레이션이 됨
# 랜덤 분할보다 성능이 낮게 나오지만, 더 현실적인 기대치를 줌
```

### 커스터마이징 포인트 3: 슬라이스별 성능 평가 추가

```python
# AutoML은 전체 데이터에 대한 메트릭만 보여줌
# 현업에서는 세그먼트별 성능이 더 중요함

from sklearn.metrics import f1_score, classification_report

# 전체 성능 (AutoML이 이미 보여줌)
print(f"Overall F1: {f1_score(y_test, y_pred):.4f}")

# === 세그먼트별 성능 (직접 추가해야 함) ===
segments = {
    'VIP': test_df['tier'] == 'VIP',
    'Regular': test_df['tier'] == 'Regular',
    'New': test_df['tenure_months'] < 3,
    'High-value': test_df['total_spent'] > 1000000,
}

for name, mask in segments.items():
    if mask.sum() > 0:
        segment_f1 = f1_score(y_test[mask], y_pred[mask])
        print(f"  {name} F1: {segment_f1:.4f} (n={mask.sum()})")

# 결과 예시:
# Overall F1: 0.85
#   VIP F1: 0.62 ← 문제 발견! VIP 세그먼트에서 성능이 낮음
#   Regular F1: 0.89
#   New F1: 0.71 ← 신규 고객도 성능이 낮음
#   High-value F1: 0.58 ← 고가치 고객에서 최악
```

### 커스터마이징 포인트 4: MLflow에 최종 모델 등록

```python
# AutoML 노트북을 커스터마이징한 후, 최종 모델을 MLflow에 등록

import mlflow

with mlflow.start_run(run_name="customized-churn-model-v1") as run:
    # 커스텀 태그 추가
    mlflow.set_tag("base", "automl")
    mlflow.set_tag("customizations", "domain_features, time_split, segment_eval")
    mlflow.set_tag("data_version", "2025-03-31")

    # 모델 학습 + 로깅
    model.fit(X_train, y_train)
    mlflow.sklearn.log_model(
        model, "model",
        registered_model_name="catalog.ml.churn_predictor"
    )

    # 세그먼트별 메트릭 로깅
    for name, mask in segments.items():
        segment_f1 = f1_score(y_test[mask], y_pred[mask])
        mlflow.log_metric(f"f1_{name.lower()}", segment_f1)

    # 전체 메트릭
    mlflow.log_metric("f1_overall", f1_score(y_test, y_pred))
```

> 💡 **현업 팁**: AutoML → 커스터마이징 → MLflow 등록 → Model Serving 배포의 전체 사이클을 처음에 1~2일 만에 완주하는 것을 목표로 하세요. 완벽한 모델을 만드는 것보다 **빠르게 전체 파이프라인을 구축하고, 이후에 모델을 개선하는** 접근이 현업에서 훨씬 효과적입니다. 많은 팀이 "모델 성능을 0.01 올리는 데 2주"를 쓰면서 배포 파이프라인은 구축하지 않습니다. 그러면 아무리 좋은 모델도 비즈니스 가치를 만들 수 없습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **AutoML**| 데이터를 주면 자동으로 전처리, 모델 선택, 튜닝을 수행합니다 |
| **3가지 문제 유형**| 분류(classify), 회귀(regress), 시계열 예측(forecast)을 지원합니다 |
| **노트북 생성**| 각 시행의 전체 코드가 담긴 노트북이 자동 생성되어 커스터마이즈할 수 있습니다 |
| **MLflow 연동**| 모든 시행이 MLflow에 자동 로깅되어 비교, 추적, 배포가 가능합니다 |
| **SHAP 분석** | 피처 중요도를 자동으로 계산하여 모델 해석성을 제공합니다 |

---

## 참고 링크

- [Databricks: AutoML](https://docs.databricks.com/aws/en/machine-learning/automl/)
- [Databricks: AutoML API reference](https://docs.databricks.com/aws/en/machine-learning/automl/automl-api-reference.html)
- [Databricks: AutoML classification](https://docs.databricks.com/aws/en/machine-learning/automl/train-ml-model-automl-api.html)
- [Databricks: AutoML forecasting](https://docs.databricks.com/aws/en/machine-learning/automl/train-forecast-automl.html)
- [Azure Databricks: AutoML](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/)
