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

## TPE(Tree-structured Parzen Estimator) 알고리즘 원리

Hyperopt의 기본 알고리즘인 **TPE**는 Bayesian Optimization의 일종으로, 일반적인 Gaussian Process 기반 방법보다 고차원 공간에서 더 효율적입니다.

### TPE의 핵심 아이디어

일반적인 Bayesian Optimization은 `P(y|x)` (파라미터 x가 주어졌을 때 성능 y의 확률)를 모델링하지만, TPE는 **역방향**으로 `P(x|y)` (성능 y가 주어졌을 때 파라미터 x의 확률)를 모델링합니다.

```
TPE 동작 과정:

1. 초기 시행 (랜덤): 10~20회 랜덤 샘플링으로 기본 데이터 수집

2. 결과를 두 그룹으로 분리:
   - l(x): "좋은" 결과를 낸 파라미터 분포 (상위 γ%, 기본 γ=25%)
   - g(x): "나쁜" 결과를 낸 파라미터 분포 (나머지 75%)

3. EI(Expected Improvement) 최대화:
   - 다음 시도할 파라미터 = l(x) / g(x) 비율이 높은 지점
   - 즉, "좋은 결과에서는 자주 나타나고, 나쁜 결과에서는 드문" 파라미터를 선택

4. 시행 후 결과를 두 그룹에 반영하고 2~3 반복
```

### 왜 TPE가 효과적인가

| 특성 | Gaussian Process (GP) | TPE |
|------|----------------------|-----|
| **고차원 확장성** | 차원이 높으면 성능 저하 | 차원에 강건 |
| **조건부 파라미터** | 처리 어려움 | 자연스럽게 지원 |
| **범주형 파라미터** | 인코딩 필요 | 직접 지원 (`hp.choice`) |
| **계산 비용** | O(n^3) — 시행 수 증가 시 느림 | O(n log n) — 빠름 |
| **병렬화** | 어려움 | 비교적 용이 |

### TPE의 한계

| 한계 | 설명 |
|------|------|
| **탐색-활용 균형** | γ 값이 너무 작으면 과도한 탐색(exploration), 너무 크면 조기 수렴 |
| **상관관계 무시** | 파라미터 간 상호작용을 직접 모델링하지 않음 |
| **초기 시행 의존** | 초기 랜덤 시행이 편향되면 이후 탐색도 편향될 수 있음 |

---

## SparkTrials 분산 동작 구조

SparkTrials는 Spark 클러스터의 Worker 노드를 활용하여 여러 하이퍼파라미터 조합을 동시에 평가합니다.

### 내부 아키텍처

```
Driver Node (Hyperopt 컨트롤러)
├── TPE 알고리즘 실행 (다음 시행할 파라미터 선택)
├── Trial 큐 관리 (대기 → 실행 → 완료)
└── MLflow 결과 로깅

Worker Node 1: Trial #1 (max_depth=5, lr=0.01) → F1=0.85
Worker Node 2: Trial #2 (max_depth=7, lr=0.03) → F1=0.87
Worker Node 3: Trial #3 (max_depth=3, lr=0.1)  → F1=0.82
Worker Node 4: Trial #4 (max_depth=9, lr=0.05) → F1=0.86
    ...
Worker Node N: Trial #N 동시 실행
```

### parallelism과 Bayesian Optimization의 트레이드오프

```
parallelism = 1:  (순차 실행)
  Trial 1 완료 → TPE가 결과 반영 → Trial 2 선택 → 완료 → ...
  장점: 이전 결과를 100% 반영하여 가장 효율적인 탐색
  단점: 매우 느림

parallelism = max_evals/2:  (높은 병렬)
  Trial 1~50 동시 시작 (TPE가 반영할 이전 결과가 거의 없음)
  장점: 빠른 완료
  단점: Random Search에 가까워짐

parallelism = sqrt(max_evals):  (권장 균형점)
  예: max_evals=100 → parallelism=10
  적절한 이전 결과 반영 + 합리적 실행 시간
```

> 💡 **실무 권장**: `parallelism`은 **Worker 노드 수**와 **max_evals의 제곱근** 중 작은 값을 선택합니다. 예를 들어 Worker 8대, max_evals=100이면 `parallelism=8`이 적절합니다.

### SparkTrials 주의사항

| 주의사항 | 설명 |
|----------|------|
| **메모리** | 각 Worker에서 모델을 학습하므로, Worker 메모리가 충분해야 합니다 |
| **데이터 복제** | 학습 데이터가 각 Worker로 복제됩니다. 대규모 데이터는 브로드캐스트 변수 활용 |
| **GPU** | SparkTrials는 GPU를 Worker당 1개씩 할당합니다. GPU 모델 학습 시 유용 |
| **에러 처리** | 개별 Trial 실패 시 해당 Trial만 건너뛰고 나머지 계속 실행 |
| **체크포인트** | `SparkTrials`는 체크포인트를 지원하지 않으므로, 실행 중 클러스터가 중단되면 처음부터 재실행 |

---

## Early Stopping 전략

하이퍼파라미터 튜닝 비용을 줄이는 가장 효과적인 방법은 **성능이 낮은 시행을 조기에 중단**하는 것입니다.

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
| **수렴 속도** | 빠름 | 빠름 (유사) |
| **최종 성능** | 유사 | 유사 (가지치기로 더 많은 시행 가능) |
| **100회 시행 실행 시간** | 기준 | 가지치기 적용 시 30~50% 빠름 |
| **메모리 사용** | 낮음 | 약간 높음 (Study 객체) |
| **대시보드** | MLflow UI | Optuna Dashboard + MLflow |
| **분산 지원** | SparkTrials (네이티브) | Joblib + Spark (간접) |

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

하이퍼파라미터가 10개 이상이고 각각의 범위가 넓은 "대규모 탐색 공간"에서는 단일 전략보다 **단계적 접근**이 효과적입니다.

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
| **하이퍼파라미터 튜닝** | 모델 학습 전 설정하는 값의 최적 조합을 찾는 과정입니다 |
| **TPE 알고리즘** | 좋은/나쁜 결과의 파라미터 분포를 분리 모델링하여 효율적으로 탐색합니다 |
| **Hyperopt + SparkTrials** | Databricks 내장, Spark 분산 병렬 튜닝을 지원합니다 |
| **parallelism** | sqrt(max_evals) 또는 Worker 수 중 작은 값이 권장됩니다 |
| **Optuna** | 직관적 API, 가지치기, 다목적 최적화를 지원하는 최신 프레임워크입니다 |
| **Early Stopping** | 유망하지 않은 시행을 조기 중단하여 비용을 크게 절감합니다 |
| **탐색 공간** | hp.choice, hp.uniform, hp.loguniform 등으로 파라미터 범위를 정의합니다 |
| **3단계 전략** | Random(거친) → Bayesian(세밀) → Grid(검증) 순서가 대규모 탐색에 효과적입니다 |
| **MLflow 연동** | 모든 시행이 MLflow에 자동 기록되어 비교 및 추적이 가능합니다 |

---

## 참고 링크

- [Databricks: Hyperopt](https://docs.databricks.com/aws/en/machine-learning/automl/hyperopt/)
- [Databricks: SparkTrials](https://docs.databricks.com/aws/en/machine-learning/automl/hyperopt/hyperopt-spark-mlflow-integration.html)
- [Hyperopt documentation](http://hyperopt.github.io/hyperopt/)
- [Optuna documentation](https://optuna.readthedocs.io/)
- [Azure Databricks: Hyperparameter tuning](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/hyperopt/)
