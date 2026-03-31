# 하이퍼파라미터 튜닝 상세

## 하이퍼파라미터 튜닝이란?

**하이퍼파라미터(Hyperparameter)** 는 모델 학습 **전에 사람이 직접 설정**하는 값입니다(예: 학습률, 트리 깊이, 은닉층 크기). 이 값에 따라 모델 성능이 크게 달라지므로, **최적의 조합을 찾는 과정**을 하이퍼파라미터 튜닝이라고 합니다.

> 💡 **비유**: 요리에서 "불의 세기(학습률)", "양념의 양(정규화 강도)", "조리 시간(에포크 수)"을 조절하여 최고의 맛(모델 성능)을 찾는 것과 같습니다.

---

## 탐색 전략 비교

| 전략 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **Grid Search** | 모든 가능한 조합을 시도합니다 | 놓치는 조합 없음 | 차원이 많으면 비용 폭발 |
| **Random Search** | 무작위로 조합을 샘플링합니다 | 효율적, 차원에 강건 | 최적값을 놓칠 수 있음 |
| **Bayesian Optimization** | 이전 결과를 바탕으로 다음 시도를 선택합니다 | 가장 효율적 | 구현이 복잡 |

---

## Hyperopt + SparkTrials

**Hyperopt**는 Databricks에 기본 내장된 Bayesian Optimization 기반 하이퍼파라미터 튜닝 라이브러리입니다. **SparkTrials**를 사용하면 Spark 클러스터에서 **병렬로 분산 튜닝**을 실행할 수 있습니다.

### 기본 사용법

```python
from hyperopt import fmin, tpe, hp, Trials, STATUS_OK
from hyperopt import SparkTrials
from sklearn.model_selection import cross_val_score
from xgboost import XGBClassifier
import mlflow
import numpy as np

# 1. 탐색 공간 정의
search_space = {
    "max_depth": hp.choice("max_depth", [3, 5, 7, 9, 11]),
    "n_estimators": hp.choice("n_estimators", [100, 200, 300, 500]),
    "learning_rate": hp.loguniform("learning_rate", np.log(0.01), np.log(0.3)),
    "min_child_weight": hp.uniform("min_child_weight", 1, 10),
    "subsample": hp.uniform("subsample", 0.5, 1.0),
    "colsample_bytree": hp.uniform("colsample_bytree", 0.5, 1.0),
    "gamma": hp.uniform("gamma", 0, 5),
    "reg_alpha": hp.loguniform("reg_alpha", np.log(0.001), np.log(10)),
    "reg_lambda": hp.loguniform("reg_lambda", np.log(0.001), np.log(10))
}

# 2. 목적 함수 정의
def objective(params):
    with mlflow.start_run(nested=True):
        # 파라미터 로깅
        mlflow.log_params(params)

        # 모델 학습 및 교차 검증
        model = XGBClassifier(**params, use_label_encoder=False, eval_metric="logloss")
        scores = cross_val_score(model, X_train, y_train, cv=5, scoring="f1_weighted")

        avg_score = np.mean(scores)
        mlflow.log_metric("cv_f1_score", avg_score)

        # Hyperopt는 최소화하므로 음수로 반환
        return {"loss": -avg_score, "status": STATUS_OK}

# 3. 분산 튜닝 실행
with mlflow.start_run(run_name="xgboost-tuning"):
    spark_trials = SparkTrials(parallelism=8)  # 8개 동시 시행

    best_params = fmin(
        fn=objective,
        space=search_space,
        algo=tpe.suggest,         # Tree-structured Parzen Estimator
        max_evals=100,            # 최대 100번 시행
        trials=spark_trials
    )

    print(f"Best parameters: {best_params}")
```

### SparkTrials vs Trials

| 클래스 | 실행 방식 | 적합한 상황 |
|--------|---------|-----------|
| `Trials` | 단일 노드에서 순차 실행 | 가벼운 모델, 빠른 학습 |
| `SparkTrials` | Spark Worker에서 병렬 실행 | 무거운 모델, 많은 시행 수 |

### SparkTrials parallelism 설정

```python
# Worker 노드 수에 맞춰 병렬도 설정
spark_trials = SparkTrials(parallelism=8)  # Worker 8개에서 동시 실행
```

> ⚠️ **parallelism이 너무 높으면** Bayesian Optimization의 효과가 줄어듭니다. 이전 결과를 충분히 반영하기 전에 다음 시행이 시작되기 때문입니다. `parallelism`은 `max_evals`의 1/3~1/2 정도를 권장합니다.

---

## 탐색 공간 정의 (hp 함수)

| 함수 | 설명 | 예시 |
|------|------|------|
| `hp.choice(name, options)` | 목록에서 하나를 선택합니다 | `hp.choice("depth", [3, 5, 7, 9])` |
| `hp.uniform(name, low, high)` | 균일 분포에서 실수를 샘플링합니다 | `hp.uniform("dropout", 0.1, 0.5)` |
| `hp.loguniform(name, low, high)` | 로그 균일 분포에서 샘플링합니다 | `hp.loguniform("lr", log(0.001), log(0.1))` |
| `hp.quniform(name, low, high, q)` | q 간격의 균일 분포입니다 | `hp.quniform("layers", 1, 5, 1)` |
| `hp.normal(name, mu, sigma)` | 정규 분포에서 샘플링합니다 | `hp.normal("weight", 0, 1)` |
| `hp.randint(name, upper)` | 정수 균일 분포입니다 | `hp.randint("seed", 100)` |

### 어떤 함수를 사용해야 하는가?

| 하이퍼파라미터 | 권장 함수 | 이유 |
|-------------|---------|------|
| 학습률 (learning_rate) | `hp.loguniform` | 작은 값(0.001)과 큰 값(0.1) 사이에서 탐색 |
| 트리 깊이 (max_depth) | `hp.choice` | 이산적 정수값 |
| 드롭아웃 비율 | `hp.uniform` | 0~1 사이 연속값 |
| 정규화 강도 | `hp.loguniform` | 로그 스케일에서 균일하게 탐색 |
| 은닉층 수 | `hp.quniform` | 정수값 (1, 2, 3, ...) |

---

## Optuna 연동

**Optuna**는 최신 하이퍼파라미터 최적화 프레임워크로, Hyperopt보다 더 직관적인 API와 고급 기능(가지치기, 다목적 최적화)을 제공합니다.

```python
import optuna
import mlflow
from xgboost import XGBClassifier
from sklearn.model_selection import cross_val_score

def objective(trial):
    params = {
        "max_depth": trial.suggest_int("max_depth", 3, 12),
        "n_estimators": trial.suggest_int("n_estimators", 100, 500, step=50),
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.3, log=True),
        "min_child_weight": trial.suggest_float("min_child_weight", 1, 10),
        "subsample": trial.suggest_float("subsample", 0.5, 1.0),
        "colsample_bytree": trial.suggest_float("colsample_bytree", 0.5, 1.0),
        "gamma": trial.suggest_float("gamma", 0, 5),
    }

    with mlflow.start_run(nested=True):
        mlflow.log_params(params)

        model = XGBClassifier(**params, use_label_encoder=False, eval_metric="logloss")
        scores = cross_val_score(model, X_train, y_train, cv=5, scoring="f1_weighted")
        avg_score = scores.mean()

        mlflow.log_metric("cv_f1_score", avg_score)
        return avg_score

# Optuna 스터디 생성 및 최적화
with mlflow.start_run(run_name="optuna-tuning"):
    study = optuna.create_study(
        direction="maximize",
        sampler=optuna.samplers.TPESampler(seed=42),
        pruner=optuna.pruners.MedianPruner()  # 성능이 낮은 시행 조기 중단
    )
    study.optimize(objective, n_trials=100)

    print(f"Best params: {study.best_params}")
    print(f"Best F1: {study.best_value:.4f}")
    mlflow.log_params(study.best_params)
    mlflow.log_metric("best_f1_score", study.best_value)
```

### Hyperopt vs Optuna 비교

| 비교 항목 | Hyperopt | Optuna |
|-----------|---------|--------|
| **Databricks 내장** | ML Runtime에 포함 | `%pip install optuna` 필요 |
| **Spark 분산** | SparkTrials 지원 | 별도 분산 구현 필요 |
| **가지치기(Pruning)** | 미지원 | 내장 지원 |
| **API 직관성** | 함수형 | 객체지향형 (더 직관적) |
| **시각화** | 제한적 | `optuna.visualization` 내장 |

---

## MLflow 자동 로깅

Hyperopt/Optuna 모두 MLflow와 연동하여 모든 시행을 자동으로 추적할 수 있습니다.

```python
# MLflow 자동 로깅 활성화
mlflow.autolog()

# 또는 수동으로 중요 지표 기록
with mlflow.start_run(run_name="final-best-model"):
    # 최적 파라미터로 최종 모델 학습
    best_model = XGBClassifier(**best_params)
    best_model.fit(X_train, y_train)

    # 평가
    test_f1 = f1_score(y_test, best_model.predict(X_test), average="weighted")
    mlflow.log_metric("test_f1_score", test_f1)

    # 모델 저장 및 등록
    mlflow.sklearn.log_model(
        best_model,
        "model",
        registered_model_name="catalog.schema.churn_classifier"
    )
```

---

## 분산 튜닝 패턴

### Hyperopt + SparkTrials (Spark 네이티브 분산)

```python
# 가장 간편한 분산 튜닝
spark_trials = SparkTrials(parallelism=4)
best = fmin(fn=objective, space=space, algo=tpe.suggest,
            max_evals=50, trials=spark_trials)
```

### Optuna + Joblib (Spark 병렬)

```python
from joblibspark import register_spark

register_spark()  # Joblib의 Spark 백엔드 등록

# Optuna의 n_jobs를 활용한 병렬 실행
study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=100, n_jobs=-1)  # Spark Worker에서 병렬 실행
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **하이퍼파라미터 튜닝** | 모델 학습 전 설정하는 값의 최적 조합을 찾는 과정입니다 |
| **Hyperopt + SparkTrials** | Databricks 내장, Spark 분산 병렬 튜닝을 지원합니다 |
| **Optuna** | 직관적 API, 가지치기, 다목적 최적화를 지원하는 최신 프레임워크입니다 |
| **탐색 공간** | hp.choice, hp.uniform, hp.loguniform 등으로 파라미터 범위를 정의합니다 |
| **MLflow 연동** | 모든 시행이 MLflow에 자동 기록되어 비교 및 추적이 가능합니다 |

---

## 참고 링크

- [Databricks: Hyperopt](https://docs.databricks.com/aws/en/machine-learning/automl/hyperopt/)
- [Databricks: SparkTrials](https://docs.databricks.com/aws/en/machine-learning/automl/hyperopt/hyperopt-spark-mlflow-integration.html)
- [Hyperopt documentation](http://hyperopt.github.io/hyperopt/)
- [Optuna documentation](https://optuna.readthedocs.io/)
- [Azure Databricks: Hyperparameter tuning](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/hyperopt/)
