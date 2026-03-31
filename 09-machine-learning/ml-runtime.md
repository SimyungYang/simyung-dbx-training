# ML Runtime

## ML Runtime이란?

머신러닝 프로젝트를 시작할 때 가장 번거로운 작업 중 하나가 **환경 설정**입니다. PyTorch, TensorFlow, scikit-learn, XGBoost, MLflow 등 수십 개의 라이브러리를 설치하고, 서로의 버전 호환성을 맞추는 데만 상당한 시간이 소요됩니다.

> 💡 **ML Runtime**은 이 문제를 해결합니다. 머신러닝에 필요한 모든 라이브러리가 **사전 설치되고 호환성이 검증된** Databricks Runtime입니다. 클러스터를 시작하면 바로 모델 학습을 시작할 수 있습니다.

---

## Runtime 유형별 비교

Databricks는 용도에 따라 3가지 Runtime을 제공합니다.

| Runtime | 포함 내용 | 적합한 워크로드 | 예시 |
|---------|----------|-------------|------|
| **Standard Runtime** | Spark + Delta Lake + 기본 Python | 데이터 엔지니어링, SQL 분석 | ETL 파이프라인, 배치 처리 |
| **ML Runtime** | Standard + **ML/DL 라이브러리** | ML 모델 학습, Feature Engineering | scikit-learn, XGBoost 학습 |
| **ML Runtime (GPU)** | ML Runtime + **GPU 드라이버** (CUDA, cuDNN) | 딥러닝, LLM 파인튜닝 | PyTorch, Transformer 학습 |

| 런타임 | 포함 구성 | 추가 요소 |
|--------|-----------|-----------|
| **Standard Runtime** | Spark + Delta Lake | 기본 런타임 |
| **ML Runtime** | Standard + ML 라이브러리 | scikit-learn, XGBoost, PyTorch 등 사전 설치 |
| **ML Runtime (GPU)** | ML Runtime + CUDA/cuDNN | GPU 드라이버가 추가됩니다 |

---

## ML Runtime에 포함된 주요 라이브러리

### 전통 ML (Traditional Machine Learning)

| 라이브러리 | 용도 | 특징 |
|-----------|------|------|
| **scikit-learn** | 분류, 회귀, 클러스터링, 전처리 | 가장 범용적인 ML 라이브러리 |
| **XGBoost** | 그래디언트 부스팅 | 정형 데이터에서 최고 성능, Kaggle 우승 단골 |
| **LightGBM** | 경량 그래디언트 부스팅 | 대용량 데이터에서 빠른 학습 속도 |
| **CatBoost** | 카테고리 피처 부스팅 | 범주형 변수를 자동 처리 |
| **Spark MLlib** | 분산 ML | Spark 위에서 대용량 데이터로 학습 |

### 딥러닝 (Deep Learning)

| 라이브러리 | 용도 | 특징 |
|-----------|------|------|
| **PyTorch** | 딥러닝 프레임워크 | 연구/프로덕션 모두 사용, 가장 인기 |
| **TensorFlow / Keras** | 딥러닝 프레임워크 | 프로덕션 배포에 강점 |
| **Hugging Face Transformers** | 사전학습 모델 활용 | BERT, GPT, LLaMA 등 LLM 사용 |
| **tokenizers** | 텍스트 토크나이저 | Transformers와 함께 사용 |

### 데이터 처리 / 시각화

| 라이브러리 | 용도 |
|-----------|------|
| **pandas** | 데이터 조작 (DataFrame) |
| **NumPy** | 수치 계산 |
| **SciPy** | 과학/통계 계산 |
| **matplotlib / seaborn** | 정적 데이터 시각화 |
| **plotly** | 대화형 시각화 |

### ML Ops / 실험 관리

| 라이브러리 | 용도 |
|-----------|------|
| **MLflow** | 실험 추적, 모델 관리, 트레이싱 |
| **Hyperopt** | 하이퍼파라미터 자동 튜닝 (Bayesian Optimization) |
| **Optuna** | 하이퍼파라미터 튜닝 (최신) |
| **SHAP** | 모델 해석 (피처 중요도 시각화) |

> 💡 ML Runtime에 포함된 라이브러리 버전은 Runtime 버전마다 다릅니다. 정확한 버전 목록은 [릴리즈 노트](https://docs.databricks.com/aws/en/release-notes/runtime/ml.html)를 확인하세요.

---

## GPU Runtime 구성

### GPU 인스턴스 유형

| 인스턴스 (AWS) | GPU | GPU 메모리 | vCPU | RAM | 적합한 워크로드 |
|---------------|-----|----------|------|-----|-------------|
| **g4dn.xlarge** | NVIDIA T4 × 1 | 16 GB | 4 | 16 GB | 추론, 소규모 학습 |
| **g5.xlarge** | NVIDIA A10G × 1 | 24 GB | 4 | 16 GB | 중규모 학습, 파인튜닝 |
| **g5.12xlarge** | NVIDIA A10G × 4 | 96 GB | 48 | 192 GB | 대규모 학습 |
| **p3.2xlarge** | NVIDIA V100 × 1 | 16 GB | 8 | 61 GB | 고성능 학습 |
| **p4d.24xlarge** | NVIDIA A100 × 8 | 320 GB | 96 | 1,152 GB | LLM 파인튜닝, 대규모 분산 학습 |

### 선택 가이드

| 워크로드 | 권장 인스턴스 | 이유 |
|---------|------------|------|
| 추론 (Inference) | g4dn.xlarge (T4) | 비용 대비 추론 성능이 우수 |
| 소규모 모델 학습 | g5.xlarge (A10G) | 24GB GPU 메모리로 대부분의 모델 처리 |
| 이미지/비전 모델 학습 | g5.12xlarge (A10G × 4) | 멀티 GPU로 빠른 학습 |
| LLM 파인튜닝 (7B~13B) | p4d.24xlarge (A100 × 8) | 대용량 GPU 메모리 필수 |

---

## AutoML

Databricks의 **AutoML**은 데이터를 주면 **자동으로 최적의 모델을 학습**합니다. ML 전문 지식이 없어도 빠르게 좋은 성능의 모델을 얻을 수 있습니다.

### AutoML의 동작 과정

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 데이터셋 입력 | Delta 테이블에서 데이터를 읽습니다 |
| 2 | 자동 전처리 | 결측값 처리, 인코딩 등을 자동 수행합니다 |
| 3 | 다중 모델 학습 | XGBoost, LightGBM, RF, LR 등 여러 모델을 학습합니다 |
| 4 | 자동 평가 | 교차 검증으로 성능을 평가합니다 |
| 5 | 최적 모델 선택 | 최적 모델을 선택하고 MLflow에 로깅합니다 |

### AutoML 사용 방법

**UI에서 실행:**
1. **Experiments** → **Create AutoML Experiment**
2. 데이터셋(Delta 테이블) 선택
3. 타겟 컬럼(예측할 컬럼) 지정
4. 문제 유형 선택 (분류 / 회귀 / 예측)
5. **Start** 클릭

**API로 실행:**

```python
from databricks import automl

# 분류 문제
result = automl.classify(
    dataset="catalog.schema.customer_churn",
    target_col="churned",
    timeout_minutes=30,
    max_trials=50
)

# 최적 모델 확인
print(f"Best model: {result.best_trial.model_description}")
print(f"Metric (F1): {result.best_trial.metrics['val_f1_score']:.4f}")

# 모델 URI (배포에 사용)
print(f"Model URI: {result.best_trial.model_uri}")
```

### AutoML이 지원하는 문제 유형

| 문제 유형 | 함수 | 평가 지표 |
|----------|------|----------|
| **분류 (Classification)** | `automl.classify()` | F1, Accuracy, ROC-AUC |
| **회귀 (Regression)** | `automl.regress()` | RMSE, MAE, R² |
| **시계열 예측 (Forecasting)** | `automl.forecast()` | SMAPE, RMSE |

### AutoML의 핵심 장점

| 장점 | 설명 |
|------|------|
| **자동 전처리** | 결측값 처리, 범주형 인코딩, 피처 스케일링을 자동으로 수행합니다 |
| **다양한 알고리즘** | XGBoost, LightGBM, Random Forest, Logistic Regression 등을 모두 시도합니다 |
| **노트북 생성** | 각 시행(trial)의 전체 코드를 노트북으로 자동 생성합니다. 결과를 검토하고 커스터마이즈할 수 있습니다 |
| **MLflow 자동 연동** | 모든 실험 결과가 MLflow에 자동 로깅됩니다 |
| **피처 중요도** | SHAP 기반 피처 중요도를 자동으로 계산합니다 |

> 💡 **AutoML이 생성하는 노트북을 활용하세요.** AutoML은 블랙박스가 아닙니다. 최적 모델의 학습 코드가 담긴 노트북이 자동 생성되므로, 이를 기반으로 추가 커스터마이징이 가능합니다.

---

## 분산 학습 (Distributed Training)

### TorchDistributor

Databricks ML Runtime은 **TorchDistributor**를 통해 PyTorch 모델의 멀티 GPU / 멀티 노드 분산 학습을 지원합니다.

```python
from pyspark.ml.torch.distributor import TorchDistributor

def train_function():
    import torch
    import torch.distributed as dist

    # 분산 환경 자동 초기화
    dist.init_process_group("nccl")
    rank = dist.get_rank()
    device = torch.device(f"cuda:{rank}")

    # 모델 정의 및 DDP 래핑
    model = MyModel().to(device)
    model = torch.nn.parallel.DistributedDataParallel(model, device_ids=[rank])

    # 학습 루프
    for epoch in range(10):
        train_one_epoch(model, device)

# 4개 GPU에서 분산 학습 실행
distributor = TorchDistributor(
    num_processes=4,       # 총 GPU 수
    local_mode=False,      # False = 여러 노드에 분산
    use_gpu=True
)
distributor.run(train_function)
```

### DeepSpeed 연동

대규모 LLM 학습에는 DeepSpeed를 활용할 수 있습니다.

```python
# DeepSpeed ZeRO-3로 LLM 파인튜닝
from pyspark.ml.torch.distributor import TorchDistributor

def train_with_deepspeed():
    import deepspeed
    model_engine, optimizer, _, _ = deepspeed.initialize(
        model=model,
        config="ds_config.json"  # ZeRO-3 설정
    )
    # 학습 루프...

distributor = TorchDistributor(num_processes=8, use_gpu=True, local_mode=False)
distributor.run(train_with_deepspeed)
```

---

## 라이브러리 관리

ML Runtime에 포함되지 않은 라이브러리가 필요한 경우 추가 설치할 수 있습니다.

### 설치 방법

| 방법 | 범위 | 설명 |
|------|------|------|
| **%pip install** | 노트북/클러스터 세션 | 현재 노트북에서만 유효 |
| **클러스터 라이브러리** | 클러스터 전체 | 클러스터 설정에서 라이브러리 추가 |
| **Init Script** | 클러스터 시작 시 | 클러스터 시작할 때마다 자동 실행 |
| **requirements.txt** | Job/Pipeline | Job 설정에서 의존성 파일 지정 |

```python
# 노트북에서 추가 라이브러리 설치
%pip install sentence-transformers==2.2.2 faiss-cpu==1.7.4

# 설치 후 반드시 restart
dbutils.library.restartPython()
```

> ⚠️ **`%pip install`은 기존 라이브러리의 버전을 변경할 수 있습니다.** 이로 인해 MLflow나 다른 사전 설치 라이브러리와 충돌이 발생할 수 있으므로, 버전 충돌 여부를 주의하세요.

---

## Runtime 버전 선택 가이드

| 상황 | 권장 Runtime |
|------|------------|
| 최신 라이브러리 필요 | 최신 ML Runtime (예: 16.x ML) |
| 프로덕션 안정성 우선 | LTS(Long Term Support) ML Runtime |
| GPU 학습 필요 | ML Runtime GPU |
| 기존 코드 호환성 유지 | 현재 사용 중인 Runtime 유지 |

> 💡 **LTS (Long Term Support)**: 장기 지원 버전으로, 보안 패치를 2년간 받을 수 있습니다. 프로덕션 환경에서는 LTS를 권장합니다.

| 질문 | 조건 | 권장 런타임 |
|------|------|------------|
| 워크로드 유형은? | 데이터 엔지니어링/SQL | Standard Runtime |
| 워크로드 유형은? | ML 모델 학습 → GPU 불필요 (sklearn, XGBoost) | ML Runtime (CPU) |
| 워크로드 유형은? | ML 모델 학습 → GPU 필요 → 소~중형 모델 | ML Runtime GPU (g5.xlarge) |
| 워크로드 유형은? | ML 모델 학습 → GPU 필요 → 대형 LLM 파인튜닝 | ML Runtime GPU (p4d.24xlarge) |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **ML Runtime** | ML 라이브러리가 사전 설치되고 호환성이 검증된 Databricks Runtime입니다 |
| **GPU Runtime** | CUDA/cuDNN GPU 드라이버가 포함되어 딥러닝에 사용합니다 |
| **AutoML** | 데이터를 주면 자동으로 최적의 모델을 학습하고 노트북을 생성합니다 |
| **TorchDistributor** | 멀티 GPU / 멀티 노드 분산 학습을 간편하게 실행합니다 |
| **LTS Runtime** | 프로덕션 환경에 권장되는 장기 지원 버전입니다 |

---

## 참고 링크

- [Databricks: ML Runtime release notes](https://docs.databricks.com/aws/en/release-notes/runtime/ml.html)
- [Databricks: GPU clusters](https://docs.databricks.com/aws/en/compute/gpu.html)
- [Databricks: AutoML](https://docs.databricks.com/aws/en/machine-learning/automl/)
- [Databricks: Distributed training](https://docs.databricks.com/aws/en/machine-learning/train-model/distributed-training/)
- [Azure Databricks: ML Runtime](https://learn.microsoft.com/en-us/azure/databricks/release-notes/runtime/ml)
