# 분산 학습 상세

## 왜 분산 학습이 필요한가?

딥러닝 모델의 크기와 학습 데이터의 양이 증가하면서, 단일 GPU로는 학습 시간이 수일에서 수주까지 걸릴 수 있습니다. **분산 학습(Distributed Training)** 은 여러 GPU 또는 여러 노드에 학습을 분산하여 **학습 시간을 크게 단축**합니다.

> 💡 **비유**: 1,000페이지 책을 번역할 때, 한 명이 하면 100일 걸리지만, 10명이 나누어 하면 10일이면 됩니다. 분산 학습도 마찬가지로, 데이터와 연산을 여러 GPU에 나누어 병렬로 처리합니다.

---

## 분산 학습 전략

| 전략 | 설명 | 적합한 상황 |
|------|------|-----------|
| **Data Parallelism** | 같은 모델을 여러 GPU에 복제하고, 데이터를 분할하여 학습합니다 | 대부분의 경우 (모델이 단일 GPU에 들어갈 때) |
| **Model Parallelism** | 모델 자체를 여러 GPU에 나누어 저장합니다 | 모델이 단일 GPU 메모리보다 클 때 |
| **Pipeline Parallelism** | 모델 레이어를 파이프라인으로 분할합니다 | 매우 큰 모델 (수십 B 파라미터 이상) |

---

## TorchDistributor 사용법

**TorchDistributor**는 Databricks ML Runtime에서 PyTorch 분산 학습을 쉽게 실행할 수 있게 해주는 도구입니다. Spark 클러스터의 GPU를 자동으로 감지하고, 분산 학습 환경을 설정합니다.

### 기본 사용법

```python
from pyspark.ml.torch.distributor import TorchDistributor

def train_function():
    """각 GPU에서 실행되는 학습 함수"""
    import torch
    import torch.distributed as dist
    from torch.nn.parallel import DistributedDataParallel as DDP

    # 1. 분산 환경 초기화 (TorchDistributor가 자동 설정)
    dist.init_process_group("nccl")
    rank = dist.get_rank()
    local_rank = int(os.environ.get("LOCAL_RANK", 0))
    device = torch.device(f"cuda:{local_rank}")

    # 2. 모델 정의 및 DDP 래핑
    model = MyModel().to(device)
    model = DDP(model, device_ids=[local_rank])

    # 3. 분산 데이터 로더 설정
    sampler = torch.utils.data.distributed.DistributedSampler(dataset)
    dataloader = torch.utils.data.DataLoader(
        dataset, batch_size=32, sampler=sampler
    )

    # 4. 학습 루프
    optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
    for epoch in range(10):
        sampler.set_epoch(epoch)  # 에포크마다 셔플링
        for batch in dataloader:
            inputs, labels = batch[0].to(device), batch[1].to(device)
            optimizer.zero_grad()
            outputs = model(inputs)
            loss = torch.nn.functional.cross_entropy(outputs, labels)
            loss.backward()
            optimizer.step()

        if rank == 0:  # 메인 프로세스에서만 로깅
            print(f"Epoch {epoch}, Loss: {loss.item():.4f}")

    # 5. 모델 저장 (메인 프로세스만)
    if rank == 0:
        torch.save(model.module.state_dict(), "/tmp/model.pt")

    dist.destroy_process_group()

# 4개 GPU에서 분산 학습 실행
distributor = TorchDistributor(
    num_processes=4,       # 총 프로세스(GPU) 수
    local_mode=False,      # False: 여러 노드에 분산 / True: 단일 노드
    use_gpu=True           # GPU 사용
)
result = distributor.run(train_function)
```

### TorchDistributor 주요 파라미터

| 파라미터 | 설명 | 기본값 |
|---------|------|--------|
| `num_processes` | 분산 학습에 사용할 총 프로세스(GPU) 수입니다 | (필수) |
| `local_mode` | `True`: 단일 노드 내 멀티 GPU, `False`: 멀티 노드 | `True` |
| `use_gpu` | GPU 사용 여부입니다 | `True` |

### local_mode 설정 가이드

| 설정 | 동작 | 적합한 상황 |
|------|------|-----------|
| `local_mode=True` | Driver 노드의 GPU만 사용합니다 | 단일 노드에 여러 GPU가 있을 때 |
| `local_mode=False` | Worker 노드의 GPU를 모두 사용합니다 | 여러 노드에 걸쳐 분산할 때 |

---

## DeepSpeed 연동

**DeepSpeed**는 Microsoft가 개발한 분산 학습 최적화 라이브러리로, 대규모 모델 학습 시 메모리 효율을 크게 개선합니다.

### DeepSpeed ZeRO 단계

| 단계 | 분산 대상 | 메모리 절감 | 설명 |
|------|---------|-----------|------|
| **ZeRO-1** | Optimizer States | ~4x | 옵티마이저 상태만 분산합니다 |
| **ZeRO-2** | + Gradients | ~8x | 그래디언트도 분산합니다 |
| **ZeRO-3** | + Parameters | ~Nx | 모델 파라미터까지 분산합니다 |

### DeepSpeed 설정 파일 (ds_config.json)

```json
{
    "train_batch_size": 64,
    "gradient_accumulation_steps": 4,
    "fp16": {
        "enabled": true,
        "loss_scale": 0,
        "loss_scale_window": 1000
    },
    "zero_optimization": {
        "stage": 3,
        "offload_optimizer": {
            "device": "cpu",
            "pin_memory": true
        },
        "offload_param": {
            "device": "cpu",
            "pin_memory": true
        },
        "overlap_comm": true,
        "contiguous_gradients": true
    }
}
```

### TorchDistributor + DeepSpeed

```python
from pyspark.ml.torch.distributor import TorchDistributor

def train_with_deepspeed():
    import deepspeed
    import torch

    model = MyLargeModel()

    model_engine, optimizer, _, _ = deepspeed.initialize(
        model=model,
        config="ds_config.json"
    )

    for epoch in range(10):
        for batch in dataloader:
            inputs = batch["input_ids"].to(model_engine.device)
            labels = batch["labels"].to(model_engine.device)

            outputs = model_engine(inputs, labels=labels)
            loss = outputs.loss
            model_engine.backward(loss)
            model_engine.step()

# 8개 GPU에서 DeepSpeed 분산 학습
distributor = TorchDistributor(
    num_processes=8,
    local_mode=False,
    use_gpu=True
)
distributor.run(train_with_deepspeed)
```

---

## Horovod

**Horovod**는 Uber가 개발한 분산 학습 프레임워크로, MPI 기반의 올-리듀스(All-Reduce) 알고리즘을 사용합니다.

```python
import horovod.torch as hvd
import torch

# Horovod 초기화
hvd.init()
torch.cuda.set_device(hvd.local_rank())

# 모델 및 옵티마이저
model = MyModel().cuda()
optimizer = torch.optim.Adam(model.parameters(), lr=0.001 * hvd.size())

# Horovod 분산 옵티마이저로 래핑
optimizer = hvd.DistributedOptimizer(
    optimizer,
    named_parameters=model.named_parameters()
)

# 모델 파라미터 동기화 (초기화)
hvd.broadcast_parameters(model.state_dict(), root_rank=0)
hvd.broadcast_optimizer_state(optimizer, root_rank=0)

# 학습 루프
for epoch in range(10):
    for batch in dataloader:
        inputs, labels = batch[0].cuda(), batch[1].cuda()
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = torch.nn.functional.cross_entropy(outputs, labels)
        loss.backward()
        optimizer.step()
```

> 💡 **TorchDistributor vs Horovod**: 새로운 프로젝트에서는 **TorchDistributor**(PyTorch DDP 기반)를 권장합니다. Horovod는 기존 Horovod 코드를 Databricks로 마이그레이션할 때 유용합니다.

---

## 멀티 GPU / 멀티 노드 설정

### 클러스터 설정 예시

```yaml
# 4노드 x 4GPU = 16 GPU 클러스터
new_cluster:
  spark_version: "15.4.x-gpu-ml-scala2.12"
  driver_node_type_id: "g5.12xlarge"   # A10G x 4
  node_type_id: "g5.12xlarge"          # A10G x 4
  num_workers: 3                        # Worker 3 + Driver 1 = 4노드
```

### 분산 학습 규모별 가이드

| 모델 크기 | 권장 구성 | 분산 전략 |
|----------|---------|----------|
| 소형 (~1B params) | 단일 노드, 1~4 GPU | Data Parallelism (DDP) |
| 중형 (1B~7B) | 1~2 노드, 4~8 GPU | DDP + Mixed Precision |
| 대형 (7B~70B) | 4~8 노드, 16~64 GPU | DeepSpeed ZeRO-3 |
| 초대형 (70B+) | 8+ 노드, 64+ GPU | DeepSpeed ZeRO-3 + Offloading |

---

## MLflow 연동

분산 학습에서도 MLflow를 활용하여 실험을 추적할 수 있습니다.

```python
def train_function():
    import mlflow
    import torch.distributed as dist

    dist.init_process_group("nccl")
    rank = dist.get_rank()

    # 메인 프로세스에서만 MLflow 로깅
    if rank == 0:
        mlflow.start_run()
        mlflow.log_param("num_gpus", dist.get_world_size())
        mlflow.log_param("batch_size", 32)

    # ... 학습 루프 ...

    if rank == 0:
        mlflow.log_metric("final_loss", loss.item())
        mlflow.pytorch.log_model(model.module, "model")
        mlflow.end_run()
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **TorchDistributor** | Databricks에서 PyTorch 분산 학습을 간편하게 실행하는 도구입니다 |
| **Data Parallelism** | 모델을 복제하고 데이터를 분할하여 병렬 학습하는 가장 일반적인 전략입니다 |
| **DeepSpeed** | ZeRO 최적화로 대규모 모델의 메모리 효율을 크게 개선합니다 |
| **Horovod** | MPI 기반 All-Reduce 분산 학습 프레임워크입니다 |
| **local_mode** | 단일 노드(True) vs 멀티 노드(False) 분산을 제어합니다 |

---

## 참고 링크

- [Databricks: Distributed training](https://docs.databricks.com/aws/en/machine-learning/train-model/distributed-training/)
- [Databricks: TorchDistributor](https://docs.databricks.com/aws/en/machine-learning/train-model/distributed-training/spark-pytorch-distributor.html)
- [Databricks: DeepSpeed](https://docs.databricks.com/aws/en/machine-learning/train-model/distributed-training/deepspeed.html)
- [Databricks: Horovod](https://docs.databricks.com/aws/en/machine-learning/train-model/distributed-training/horovod-runner.html)
- [Azure Databricks: Distributed training](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/train-model/distributed-training/)
