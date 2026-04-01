# 실험 추적 (Experiment Tracking)

## 왜 실험 추적이 필요한가?

머신러닝 모델을 개발할 때는 수십~수백 번의 실험을 반복합니다. 하이퍼파라미터를 조금씩 바꿔가며 학습하고, 다양한 피처 조합을 시도하며, 여러 알고리즘을 비교합니다. 이 과정에서 **체계적인 기록 없이 진행하면** 다음과 같은 문제가 발생합니다:

| 문제 | 설명 |
|------|------|
| ** 재현 불가** | "지난주에 정확도 95% 나왔던 설정이 뭐였지?" — 코드, 데이터, 파라미터를 모두 기억할 수 없습니다 |
| ** 비교 어려움** | 노트북에 결과를 수동으로 적어두면, 수십 개 실험을 체계적으로 비교하기 어렵습니다 |
| ** 협업 장벽** | 팀원이 내 실험을 재현하거나 이어서 작업하기 어렵습니다 |
| ** 감사 추적 불가** | 프로덕션 모델이 어떤 실험에서 나왔는지 추적할 수 없습니다 |

> 💡 ** 실험 추적(Experiment Tracking)** 이란 머신러닝 실험의 모든 요소(코드, 데이터, 파라미터, 메트릭, 모델 아티팩트)를 자동으로 기록하고 버전 관리하는 체계를 말합니다. MLflow Tracking은 이를 위한 업계 표준 오픈소스 도구입니다.

---

## 핵심 개념: Experiment와 Run

MLflow의 실험 추적은 **Experiment(실험)** 와 **Run(실행)** 이라는 두 가지 핵심 단위로 구성됩니다.

| 계층 | 구성 요소 | 예시 |
|------|-----------|------|
| **MLflow Tracking Server** | 최상위 | 모든 실험을 관리합니다 |
| **Experiment** | 프로젝트 단위 | "사기 탐지 모델", "추천 시스템" |
| **Run** | 개별 실험 실행 | Run 1: RF baseline (accuracy=0.92), Run 2: XGBoost v1 (accuracy=0.95), Run 3: XGBoost v2 (accuracy=0.96) |
| **Run 기록 항목** | 각 Run에 포함 | Parameters, Metrics, Artifacts, Tags |

| 구성 요소 | 설명 | 예시 |
|-----------|------|------|
| **Experiment** | 관련 실험을 묶는 최상위 컨테이너 | `/Workspace/experiments/fraud-detection` |
| **Run** | 한 번의 모델 학습 실행 단위 | `rf-baseline`, `xgboost-tuned-v2` |
| **Parameters** | 학습에 사용한 하이퍼파라미터 | `n_estimators=100`, `learning_rate=0.01` |
| **Metrics** | 모델 성능 평가 지표 | `accuracy=0.95`, `f1_score=0.93` |
| **Artifacts** | 모델 파일, 시각화, 데이터 샘플 등 | `model.pkl`, `confusion_matrix.png` |
| **Tags** | 실험 분류 및 메타데이터 | `team=fraud`, `stage=development` |

---

## 수동 로깅 상세

### 기본 로깅 API

MLflow는 실험의 각 요소를 기록하기 위한 명확한 API를 제공합니다.

```python
import mlflow
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score, precision_score, recall_score
from sklearn.model_selection import train_test_split
import matplotlib.pyplot as plt
import json

# 1. 실험 설정 (없으면 자동 생성)
mlflow.set_experiment("/Workspace/experiments/fraud-detection")

# 2. Run 시작
with mlflow.start_run(run_name="rf-baseline-v1") as run:
    # ── 파라미터 로깅 ──
    params = {
        "n_estimators": 200,
        "max_depth": 15,
        "min_samples_split": 5,
        "random_state": 42,
        "class_weight": "balanced"
    }
    mlflow.log_params(params)  # 딕셔너리로 한 번에 기록

    # ── 모델 학습 ──
    model = RandomForestClassifier(**params)
    model.fit(X_train, y_train)
    predictions = model.predict(X_test)

    # ── 메트릭 로깅 ──
    metrics = {
        "accuracy": accuracy_score(y_test, predictions),
        "f1_score": f1_score(y_test, predictions),
        "precision": precision_score(y_test, predictions),
        "recall": recall_score(y_test, predictions)
    }
    mlflow.log_metrics(metrics)  # 딕셔너리로 한 번에 기록

    # ── 에포크별 메트릭 기록 (step 활용) ──
    for epoch in range(10):
        mlflow.log_metric("training_loss", 1.0 / (epoch + 1), step=epoch)

    # ── 아티팩트 로깅 ──
    # Confusion Matrix 저장
    from sklearn.metrics import ConfusionMatrixDisplay
    fig, ax = plt.subplots()
    ConfusionMatrixDisplay.from_predictions(y_test, predictions, ax=ax)
    fig.savefig("/tmp/confusion_matrix.png")
    mlflow.log_artifact("/tmp/confusion_matrix.png", artifact_path="plots")

    # JSON 설정 파일 저장
    with open("/tmp/config.json", "w") as f:
        json.dump({"feature_columns": ["amount", "hour", "merchant"]}, f)
    mlflow.log_artifact("/tmp/config.json", artifact_path="config")

    # ── 모델 로깅 ──
    mlflow.sklearn.log_model(
        model,
        "model",
        input_example=X_test[:5],       # 입력 예시 자동 저장
        registered_model_name="fraud_detection_rf"  # UC에 모델 등록
    )

    # ── 태그 설정 ──
    mlflow.set_tags({
        "team": "fraud-detection",
        "data_version": "2025-03",
        "framework": "sklearn"
    })

    print(f"Run ID: {run.info.run_id}")
```

### 주요 로깅 함수 요약

| 함수 | 용도 | 단건/복수 |
|------|------|-----------|
| `log_param(key, value)` | 하이퍼파라미터 1개 기록 | 단건 |
| `log_params(dict)` | 하이퍼파라미터 여러 개 기록 | 복수 |
| `log_metric(key, value, step)` | 메트릭 1개 기록 (step으로 시계열 추적 가능) | 단건 |
| `log_metrics(dict)` | 메트릭 여러 개 기록 | 복수 |
| `log_artifact(local_path, artifact_path)` | 파일을 아티팩트로 저장 | 단건 |
| `log_artifacts(local_dir)` | 디렉터리 전체를 아티팩트로 저장 | 복수 |
| `sklearn.log_model(model, path)` | 모델을 MLflow 형식으로 저장 | 단건 |
| `set_tag(key, value)` | 태그 1개 설정 | 단건 |
| `set_tags(dict)` | 태그 여러 개 설정 | 복수 |

---

## 자동 로깅 (Autolog) 심화

`mlflow.autolog()`를 활성화하면 ** 별도의 로깅 코드 없이** 파라미터, 메트릭, 모델이 자동으로 기록됩니다. Databricks 노트북에서는 기본적으로 autolog가 활성화되어 있습니다.

```python
import mlflow
mlflow.autolog()

# 이것만으로 파라미터, 메트릭, 모델이 모두 자동 기록됩니다
from sklearn.ensemble import GradientBoostingClassifier
model = GradientBoostingClassifier(n_estimators=200, learning_rate=0.05)
model.fit(X_train, y_train)
```

### 프레임워크별 자동 로깅 동작

| 프레임워크 | 자동 기록 항목 | 비고 |
|-----------|---------------|------|
| **scikit-learn** | 모든 하이퍼파라미터, 학습/평가 메트릭, 모델, 피처 중요도 | `post_training_metrics=True` 옵션 |
| **XGBoost** | 파라미터, 학습 메트릭(각 boosting round), 모델, 피처 중요도 | 학습 곡선 자동 추적 |
| **LightGBM** | 파라미터, 메트릭, 모델, 피처 중요도 | XGBoost와 유사 |
| **PyTorch** | 에포크별 loss, 모델 가중치 | `log_every_n_epoch` 설정 가능 |
| **TensorFlow/Keras** | 에포크별 메트릭, 콜백 메트릭, 모델 | `log_models=True` 기본값 |
| **Spark ML** | Pipeline 파라미터, 평가 메트릭, 모델 | 분산 학습 자동 추적 |
| **Hugging Face** | 학습 args, 에포크별 메트릭, 모델 체크포인트 | Transformers Trainer 연동 |

> **autolog 세부 제어 옵션:**
> ```python
> mlflow.autolog(
>     log_input_examples=True,    # 입력 데이터 샘플 저장
>     log_model_signatures=True,  # 모델 입출력 스키마 저장
>     log_models=True,            # 모델 아티팩트 저장
>     silent=True                 # 로깅 메시지 숨김
> )
> ```

---

## 런(Run) 비교 및 시각화

MLflow UI에서는 여러 Run을 선택하여 ** 파라미터와 메트릭을 한눈에 비교**할 수 있습니다.

### UI에서의 비교 방법

1. ** 실험 페이지 접속**: 왼쪽 사이드바에서 **Experiments** 클릭
2. **Run 선택**: 비교할 Run들의 체크박스를 선택
3. **Compare 클릭**: 선택한 Run들의 파라미터/메트릭을 테이블, 차트로 비교
4. ** 차트 유형**: Parallel Coordinates Plot, Scatter Plot, Bar Chart 등 제공

### 프로그래밍 방식으로 비교

```python
import mlflow

# 실험의 모든 Run 검색
experiment = mlflow.get_experiment_by_name("/Workspace/experiments/fraud-detection")
runs = mlflow.search_runs(
    experiment_ids=[experiment.experiment_id],
    filter_string="metrics.f1_score > 0.9",
    order_by=["metrics.f1_score DESC"],
    max_results=10
)

# 결과를 Pandas DataFrame으로 확인
print(runs[["run_id", "params.n_estimators", "params.max_depth",
            "metrics.accuracy", "metrics.f1_score"]])

# 최고 성능 Run의 모델 로드
best_run_id = runs.iloc[0]["run_id"]
best_model = mlflow.sklearn.load_model(f"runs:/{best_run_id}/model")
```

---

## 하이퍼파라미터 튜닝 연동

실험 추적의 진정한 가치는 ** 하이퍼파라미터 튜닝과 결합**할 때 발휘됩니다. 각 탐색 조합이 자동으로 기록되므로, 최적 파라미터를 체계적으로 찾을 수 있습니다.

### Optuna + MLflow

```python
import optuna
import mlflow
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import f1_score

mlflow.set_experiment("/Workspace/experiments/fraud-detection-tuning")

def objective(trial):
    with mlflow.start_run(nested=True):  # 부모 Run 하위에 중첩 Run 생성
        params = {
            "n_estimators": trial.suggest_int("n_estimators", 50, 500),
            "max_depth": trial.suggest_int("max_depth", 3, 20),
            "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.3, log=True),
            "subsample": trial.suggest_float("subsample", 0.5, 1.0)
        }
        mlflow.log_params(params)

        model = GradientBoostingClassifier(**params, random_state=42)
        model.fit(X_train, y_train)

        score = f1_score(y_test, model.predict(X_test))
        mlflow.log_metric("f1_score", score)
        mlflow.sklearn.log_model(model, "model")

        return score

# 부모 Run 안에서 튜닝 실행
with mlflow.start_run(run_name="optuna-tuning-50trials"):
    study = optuna.create_study(direction="maximize")
    study.optimize(objective, n_trials=50)

    # 최적 파라미터 기록
    mlflow.log_params({f"best_{k}": v for k, v in study.best_params.items()})
    mlflow.log_metric("best_f1_score", study.best_value)
```

### Hyperopt + SparkTrials (분산 튜닝)

```python
from hyperopt import fmin, tpe, hp, SparkTrials, STATUS_OK
import mlflow

space = {
    "n_estimators": hp.choice("n_estimators", [100, 200, 300, 500]),
    "max_depth": hp.quniform("max_depth", 3, 20, 1),
    "learning_rate": hp.loguniform("learning_rate", -4, -1)
}

def train_model(params):
    with mlflow.start_run(nested=True):
        params["max_depth"] = int(params["max_depth"])
        mlflow.log_params(params)

        model = GradientBoostingClassifier(**params, random_state=42)
        model.fit(X_train, y_train)
        score = f1_score(y_test, model.predict(X_test))
        mlflow.log_metric("f1_score", score)

        return {"loss": -score, "status": STATUS_OK}

# Spark 클러스터에서 병렬 튜닝
spark_trials = SparkTrials(parallelism=4)
with mlflow.start_run(run_name="hyperopt-distributed"):
    best = fmin(fn=train_model, space=space, algo=tpe.suggest,
                max_evals=50, trials=spark_trials)
```

---

## 태그 전략 및 실험 조직화

대규모 ML 팀에서는 ** 체계적인 태그 전략**이 실험 관리의 핵심입니다.

### 추천 태그 체계

| 태그 키 | 용도 | 예시 값 |
|---------|------|---------|
| `team` | 담당 팀 | `fraud`, `recommendation` |
| `project` | 프로젝트 이름 | `credit-scoring-v2` |
| `data_version` | 학습 데이터 버전 | `2025-03-01` |
| `feature_set` | 사용한 피처 그룹 | `basic`, `advanced`, `all` |
| `experiment_type` | 실험 유형 | `baseline`, `tuning`, `ablation` |
| `stage` | 개발 단계 | `exploration`, `validation`, `production` |

### 실험 네이밍 컨벤션

```
/Workspace/experiments/{team}/{project}/{experiment-type}

예시:
/Workspace/experiments/fraud/credit-scoring/baseline
/Workspace/experiments/fraud/credit-scoring/feature-ablation
/Workspace/experiments/fraud/credit-scoring/hyperopt-tuning
```

---

## Databricks 추가 기능

Databricks에서 MLflow를 사용하면 추가적인 편의 기능을 활용할 수 있습니다.

| 구성 요소 | 연결 대상 | 설명 |
|-----------|-----------|------|
| **Databricks Notebook** | MLflow Experiment | 자동으로 연결됩니다 |
| **MLflow Experiment** | Run 기록 | 실험 실행 기록을 관리합니다 |
| **Run 기록** | Unity Catalog 모델 레지스트리 | 모델을 등록합니다 |
| **Run 기록** | 아티팩트 스토리지 | 클라우드 스토리지에 아티팩트를 저장합니다 |
| **MLflow UI** | MLflow Experiment | UI에서 실험을 조회합니다 |
| **Experiment API** | MLflow Experiment | API로 실험을 프로그래밍 방식으로 접근합니다 |

| 기능 | 설명 |
|------|------|
| ** 노트북 자동 연동** | 노트북에서 학습하면 해당 노트북의 실험에 자동으로 Run이 연결됩니다 |
| ** 노트북 스냅샷** | 각 Run에 실행 시점의 노트북 코드가 자동 저장됩니다 |
| **Git 연동** | Git 커밋 해시가 Run에 자동 기록되어 코드 버전 추적이 가능합니다 |
| **Autolog 기본 활성화** | Databricks Runtime ML에서는 autolog가 기본 활성화됩니다 |
| **Unity Catalog 통합** | `registered_model_name`에 3-level namespace(`catalog.schema.model`)를 사용하여 UC에 직접 모델을 등록합니다 |
| ** 클러스터 정보 기록** | 학습에 사용된 클러스터 사양(노드 수, 인스턴스 타입 등)이 자동 기록됩니다 |

> ⚠️ ** 주의사항**: Databricks에서 `mlflow.set_tracking_uri()`를 별도로 호출할 필요가 없습니다. Databricks 환경에서는 자동으로 워크스페이스의 MLflow Tracking Server에 연결됩니다.

---

## 실습: 전체 실험 추적 워크플로

다음은 데이터 준비부터 최적 모델 선택까지의 전체 워크플로입니다.

```python
# ── 전체 실험 추적 워크플로 ──
import mlflow
import mlflow.sklearn
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score
import pandas as pd

# 1. 데이터 준비
X, y = make_classification(n_samples=10000, n_features=20,
                           n_informative=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# 2. 실험 설정
mlflow.set_experiment("/Workspace/experiments/classification-benchmark")

# 3. 여러 알고리즘 비교
models = {
    "logistic_regression": LogisticRegression(max_iter=1000, random_state=42),
    "random_forest": RandomForestClassifier(n_estimators=200, max_depth=10, random_state=42),
    "gradient_boosting": GradientBoostingClassifier(n_estimators=200, learning_rate=0.1, random_state=42)
}

for name, model in models.items():
    with mlflow.start_run(run_name=name):
        # 학습
        model.fit(X_train, y_train)
        preds = model.predict(X_test)
        probs = model.predict_proba(X_test)[:, 1]

        # 메트릭 기록
        mlflow.log_metrics({
            "accuracy": accuracy_score(y_test, preds),
            "f1_score": f1_score(y_test, preds),
            "roc_auc": roc_auc_score(y_test, probs)
        })

        # 모델 저장
        mlflow.sklearn.log_model(model, "model", input_example=X_test[:3])

        # 태그 설정
        mlflow.set_tags({
            "algorithm": name,
            "experiment_type": "baseline_comparison"
        })

# 4. 최고 성능 모델 자동 선택
experiment = mlflow.get_experiment_by_name("/Workspace/experiments/classification-benchmark")
best_run = mlflow.search_runs(
    experiment_ids=[experiment.experiment_id],
    order_by=["metrics.roc_auc DESC"],
    max_results=1
).iloc[0]

print(f"최고 성능 모델: {best_run['tags.algorithm']}")
print(f"ROC-AUC: {best_run['metrics.roc_auc']:.4f}")
print(f"Run ID: {best_run['run_id']}")
```

---

## 정리

| 항목 | 핵심 포인트 |
|------|------------|
| **Experiment** | 관련 실험을 묶는 컨테이너. 프로젝트/팀별로 구분합니다 |
| **Run** | 한 번의 학습 실행. 파라미터, 메트릭, 아티팩트, 태그를 기록합니다 |
| ** 수동 로깅** | `log_param`, `log_metric`, `log_artifact`, `log_model`로 세밀하게 제어합니다 |
| ** 자동 로깅** | `mlflow.autolog()`로 코드 변경 없이 자동 추적합니다 |
| ** 튜닝 연동** | Optuna, Hyperopt와 중첩 Run으로 결합하여 체계적으로 최적화합니다 |
| **Databricks 통합** | 노트북 스냅샷, Git 연동, Unity Catalog 모델 등록이 자동으로 지원됩니다 |

---

## 참고 링크

- [Databricks: MLflow Tracking](https://docs.databricks.com/aws/en/mlflow/tracking.html)
- [Databricks: MLflow Experiment 관리](https://docs.databricks.com/aws/en/mlflow/experiments.html)
- [Databricks: Autologging](https://docs.databricks.com/aws/en/mlflow/databricks-autologging.html)
- [MLflow 공식 문서: Tracking](https://mlflow.org/docs/latest/tracking.html)
- [Databricks Blog: Best Practices for MLflow Experiment Tracking](https://www.databricks.com/blog)
