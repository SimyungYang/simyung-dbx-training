# Early Stopping과 대규모 탐색 전략

## Early Stopping 전략

하이퍼파라미터 튜닝 비용을 줄이는 가장 효과적인 방법은 **성능이 낮은 시행을 조기에 중단** 하는 것입니다.

### Hyperopt에서의 Early Stopping

Hyperopt 자체에는 내장 Early Stopping이 없지만, 목적 함수 내에서 구현할 수 있습니다.

```python
from hyperopt import fmin, tpe, hp, STATUS_OK, STATUS_FAIL
from hyperopt import SparkTrials
from xgboost import XGBClassifier
from sklearn.model_selection import cross_val_score
import mlflow
import numpy as np

# 글로벌 최적값 추적
best_score = float('-inf')
no_improve_count = 0
MAX_NO_IMPROVE = 20  # 20회 연속 개선 없으면 중단

def objective(params):
    global best_score, no_improve_count

    with mlflow.start_run(nested=True):
        mlflow.log_params(params)

        model = XGBClassifier(
            **params,
            use_label_encoder=False,
            eval_metric="logloss",
            early_stopping_rounds=10,  # XGBoost 내부 Early Stopping
            n_estimators=500  # 최대 500 라운드, 실제로는 조기 종료
        )

        # XGBoost의 early_stopping_rounds 활용
        model.fit(
            X_train, y_train,
            eval_set=[(X_val, y_val)],
            verbose=False
        )

        score = model.best_score
        mlflow.log_metric("best_score", score)
        mlflow.log_metric("best_iteration", model.best_iteration)

        if score > best_score:
            best_score = score
            no_improve_count = 0
        else:
            no_improve_count += 1

        return {"loss": -score, "status": STATUS_OK}
```

### Optuna의 Pruner (내장 Early Stopping)

Optuna는 강력한 가지치기(Pruning) 기능을 내장하고 있어, 유망하지 않은 시행을 학습 도중에 중단할 수 있습니다.

```python
import optuna

def objective(trial):
    params = {
        "max_depth": trial.suggest_int("max_depth", 3, 12),
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.3, log=True),
        "n_estimators": trial.suggest_int("n_estimators", 100, 1000, step=50),
    }

    model = XGBClassifier(**params, use_label_encoder=False, eval_metric="logloss")

    # K-Fold 교차 검증의 각 Fold에서 중간 결과 보고
    scores = []
    for fold_idx, (train_idx, val_idx) in enumerate(kfold.split(X, y)):
        model.fit(X[train_idx], y[train_idx])
        score = model.score(X[val_idx], y[val_idx])
        scores.append(score)

        # 중간 결과 보고 → Pruner가 판단
        trial.report(np.mean(scores), fold_idx)

        # Pruner가 "유망하지 않음"으로 판단하면 조기 중단
        if trial.should_prune():
            raise optuna.TrialPruned()

    return np.mean(scores)

study = optuna.create_study(
    direction="maximize",
    pruner=optuna.pruners.MedianPruner(
        n_startup_trials=10,    # 최소 10회는 완주
        n_warmup_steps=2,       # 최소 2 Fold는 실행
        interval_steps=1        # 매 Fold마다 가지치기 판단
    )
)
study.optimize(objective, n_trials=200)

# 가지치기로 실제 완주한 시행 수 확인
complete_trials = [t for t in study.trials if t.state == optuna.trial.TrialState.COMPLETE]
print(f"200회 중 {len(complete_trials)}회만 완주 → {200 - len(complete_trials)}회 조기 중단으로 비용 절감")
```

---

## Optuna vs Hyperopt 성능 비교 심층

### 벤치마크 결과 (일반적 관찰)

| 비교 항목 | Hyperopt (TPE) | Optuna (TPE) |
|-----------|---------------|-------------|
| **수렴 속도**| 빠름 | 빠름 (유사) |
| **최종 성능**| 유사 | 유사 (가지치기로 더 많은 시행 가능) |
| **100회 시행 실행 시간**| 기준 | 가지치기 적용 시 30~50% 빠름 |
| **메모리 사용**| 낮음 | 약간 높음 (Study 객체) |
| **대시보드**| MLflow UI | Optuna Dashboard + MLflow |
| **분산 지원**| SparkTrials (네이티브) | Joblib + Spark (간접) |

### 선택 가이드

```
Hyperopt를 선택하세요:
  ✅ Databricks ML Runtime에서 바로 사용하고 싶을 때
  ✅ SparkTrials로 쉬운 분산 실행이 필요할 때
  ✅ 추가 라이브러리 설치를 피하고 싶을 때

Optuna를 선택하세요:
  ✅ 가지치기(Pruning)로 비용을 절감하고 싶을 때
  ✅ 다목적 최적화가 필요할 때 (정확도 + 추론 시간 동시 최적화)
  ✅ 더 직관적인 API와 시각화를 원할 때
  ✅ 조건부 탐색 공간이 복잡할 때

둘 다 사용:
  ✅ Hyperopt로 빠른 초기 탐색 → Optuna로 세밀한 최적화
```

---

## 대규모 탐색 공간에서의 효율적 전략

하이퍼파라미터가 10개 이상이고 각각의 범위가 넓은 "대규모 탐색 공간"에서는 단일 전략보다 **단계적 접근** 이 효과적입니다.

### 3단계 튜닝 전략

```
Stage 1: Coarse Search (거친 탐색) — Random Search, 50~100회
  목표: 유망한 영역 식별
  방법: 넓은 범위에서 Random Search
  결과: 상위 10% 결과의 파라미터 범위 추출

Stage 2: Fine Search (세밀한 탐색) — Bayesian (TPE), 100~200회
  목표: 유망 영역 내에서 최적값 탐색
  방법: Stage 1에서 좁혀진 범위로 TPE 실행
  결과: 최적 파라미터 후보 5~10개

Stage 3: Validation (검증) — Grid Search, 10~50회
  목표: 최종 후보들의 안정성 검증
  방법: 후보 파라미터의 미세 변동에 대한 성능 안정성 확인
  결과: 가장 안정적인 최종 파라미터 선택
```

### 구현 예시

```python
import numpy as np
from hyperopt import fmin, tpe, hp, rand, SparkTrials

# Stage 1: Random Search (넓은 범위)
coarse_space = {
    "max_depth": hp.choice("max_depth", range(2, 20)),
    "learning_rate": hp.loguniform("lr", np.log(0.001), np.log(1.0)),
    "n_estimators": hp.choice("n_est", range(50, 1000, 50)),
    "min_child_weight": hp.uniform("mcw", 0.1, 20),
    "subsample": hp.uniform("ss", 0.3, 1.0),
}

coarse_trials = SparkTrials(parallelism=8)
coarse_best = fmin(fn=objective, space=coarse_space,
                   algo=rand.suggest,  # Random Search
                   max_evals=80, trials=coarse_trials)

# Stage 1 결과 분석: 상위 10%의 파라미터 범위 추출
top_results = sorted(coarse_trials.results, key=lambda x: x['loss'])[:8]
# → max_depth: 5~9, lr: 0.01~0.1, n_estimators: 200~500 ...

# Stage 2: Bayesian Search (좁은 범위)
fine_space = {
    "max_depth": hp.choice("max_depth", [5, 6, 7, 8, 9]),
    "learning_rate": hp.loguniform("lr", np.log(0.01), np.log(0.1)),
    "n_estimators": hp.choice("n_est", [200, 300, 400, 500]),
    "min_child_weight": hp.uniform("mcw", 1, 8),
    "subsample": hp.uniform("ss", 0.6, 0.95),
}

fine_trials = SparkTrials(parallelism=4)  # 병렬도 줄여서 TPE 효과 극대화
fine_best = fmin(fn=objective, space=fine_space,
                 algo=tpe.suggest,  # Bayesian (TPE)
                 max_evals=150, trials=fine_trials)
```

### 탐색 전략별 사용 시점 정리

| 상황 | 권장 전략 | 이유 |
|------|----------|------|
| 파라미터 2~3개, 각 5개 이하 값 | Grid Search | 전수 조사 가능 (< 125회) |
| 파라미터 5~8개, 적당한 범위 | Bayesian (TPE) | 효율적 탐색 |
| 파라미터 10개+, 넓은 범위 | Random → Bayesian 2단계 | 단계적 범위 축소 |
| 학습 시간이 매우 긴 모델 | Optuna + Pruning | 조기 중단으로 시간 절약 |
| 정확도 + 추론 시간 동시 최적화 | Optuna Multi-Objective | 파레토 최적 탐색 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **하이퍼파라미터 튜닝**| 모델 학습 전 설정하는 값의 최적 조합을 찾는 과정입니다 |
| **TPE 알고리즘**| 좋은/나쁜 결과의 파라미터 분포를 분리 모델링하여 효율적으로 탐색합니다 |
| **Hyperopt + SparkTrials**| Databricks 내장, Spark 분산 병렬 튜닝을 지원합니다 |
| **parallelism**| sqrt(max_evals) 또는 Worker 수 중 작은 값이 권장됩니다 |
| **Optuna**| 직관적 API, 가지치기, 다목적 최적화를 지원하는 최신 프레임워크입니다 |
| **Early Stopping**| 유망하지 않은 시행을 조기 중단하여 비용을 크게 절감합니다 |
| **탐색 공간**| hp.choice, hp.uniform, hp.loguniform 등으로 파라미터 범위를 정의합니다 |
| **3단계 전략**| Random(거친) → Bayesian(세밀) → Grid(검증) 순서가 대규모 탐색에 효과적입니다 |
| **MLflow 연동** | 모든 시행이 MLflow에 자동 기록되어 비교 및 추적이 가능합니다 |

---

## 참고 링크

- [Databricks: Hyperopt](https://docs.databricks.com/aws/en/machine-learning/automl/hyperopt/)
- [Databricks: SparkTrials](https://docs.databricks.com/aws/en/machine-learning/automl/hyperopt/hyperopt-spark-mlflow-integration.html)
- [Hyperopt documentation](http://hyperopt.github.io/hyperopt/)
- [Optuna documentation](https://optuna.readthedocs.io/)
- [Azure Databricks: Hyperparameter tuning](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/hyperopt/)
