# Optuna 연동과 MLflow 로깅

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
