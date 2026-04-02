# 컴퓨트 최적화

## 2. 컴퓨트 최적화

### 2.1 Serverless vs Classic 선택 기준

| 판단 기준 | Serverless 추천 | Classic 추천 |
|----------|----------------|-------------|
| **워크로드 패턴** | 간헐적, 불규칙 실행 | 24시간 상시 실행 |
| **시작 시간 민감도** | 즉시 시작 필요 (< 10초) | 수 분 대기 가능 |
| **비용 예측 필요** | 사용량 기반 과금 OK | 고정 예산 필수 |
| **특수 라이브러리** | 표준 라이브러리만 사용 | 커스텀 Docker, GPU, R |
| **데이터 규모** | 소~중규모 (TB 미만) | 대규모 (수십 TB 이상) |
| **관리 여력** | 인프라 팀 없음 | 전담 플랫폼 팀 있음 |

> 💡 **의사결정 공식**: 일일 평균 가동률이 **40% 미만** 이면 Serverless가 경제적입니다. 40% 이상 상시 가동한다면 Classic + Reserved Capacity가 유리합니다.

### 2.2 Auto Stop 설정 전략

Auto Stop은 유휴 클러스터를 자동으로 종료하여 불필요한 비용을 방지합니다.

| 환경 | 권장 Auto Stop | 이유 |
|------|--------------|------|
| **개발 (Interactive)** | 10분 | 개발자가 잊고 퇴근해도 자동 종료 |
| **Ad-hoc 분석** | 15분 | 쿼리 간 대기 시간 고려 |
| **공유 클러스터** | 30분 | 여러 사용자의 간헐적 사용 |
| **프로덕션 (Jobs)** | 설정 불필요 | Jobs는 작업 완료 시 자동 종료 |
| **SQL Warehouse (개발)** | 10분 | 빠른 종료로 유휴 비용 제거 |
| **SQL Warehouse (프로덕션)** | 5~10분 | 자동 재시작 기능이 있으므로 공격적 설정 가능 |

```python
# Compute Policy로 Auto Stop 강제 적용 (관리자용)
{
  "autotermination_minutes": {
    "type": "range",
    "maxValue": 30,
    "defaultValue": 10
  },
  "custom_tags.CostCenter": {
    "type": "fixed",
    "value": "engineering"
  }
}
```

### 2.3 Spot 인스턴스 활용

Spot 인스턴스(AWS) / Spot VM(Azure)을 활용하면 **60~90% 비용 절감** 이 가능합니다.

| 구성 요소 | Spot 사용 가능 | 권장 비율 | 주의사항 |
|----------|--------------|----------|---------|
| **Driver** | ❌ On-Demand 필수 | 0% Spot | 드라이버 중단 시 전체 작업 실패 |
| **Worker (배치)** | ✅ 적극 활용 | 80-100% Spot | 재시도 로직 필수 |
| **Worker (스트리밍)** | ⚠️ 주의 필요 | 50% 이하 Spot | 인터럽트 시 재처리 지연 |
| **Worker (ML 학습)** | ⚠️ 체크포인트 필수 | 50-80% Spot | 체크포인트 주기 설정 |

```python
# Spot 인스턴스 + Fallback 설정 예시
{
  "aws_attributes": {
    "first_on_demand": 1,          # 드라이버는 On-Demand
    "availability": "SPOT_WITH_FALLBACK",  # Spot 우선, 불가 시 On-Demand
    "spot_bid_price_percent": 100   # On-Demand 가격까지 입찰
  }
}
```

### 2.4 클러스터 사이징 가이드

워크로드 유형에 따른 인스턴스 선택 기준입니다.

| 워크로드 유형 | 병목 지점 | 권장 인스턴스 | 시작 구성 |
|-------------|----------|-------------|----------|
| **기본 ETL** | I/O | Storage Optimized (i3, i4i) | 4 workers, Medium |
| **복잡한 ETL (대량 조인)** | Memory + Shuffle | Memory Optimized (r5, r6i) | 8 workers, Large |
| **ML 학습** | CPU/GPU | Compute Optimized (c5) 또는 GPU (p3, g5) | 1 node (단일), Large |
| **Ad-hoc 분석** | 다양 | General Purpose (m5, m6i) | 2-4 workers, Medium |
| **스트리밍** | CPU + I/O | Compute Optimized (c5, c6i) | 4 workers, Medium |

> 💡 **사이징 공식**: `필요 코어 수 = (데이터 크기 GB / 목표 처리 시간 분) × 2`. 예를 들어 100GB 데이터를 10분 안에 처리하려면 약 20코어가 필요합니다. 이를 기준으로 워커 수를 결정하세요.

### 2.5 Instance Pool 활용

Instance Pool은 유휴 인스턴스를 미리 확보하여 클러스터 시작 시간을 단축합니다.

```python
# Instance Pool 설정 예시
{
  "instance_pool_name": "data-engineering-pool",
  "node_type_id": "i3.xlarge",
  "min_idle_instances": 2,        # 항상 2대 대기 (비용 발생)
  "max_capacity": 20,              # 최대 20대까지 확장
  "idle_instance_autotermination_minutes": 15,  # 유휴 인스턴스 15분 후 반환
  "preloaded_spark_versions": ["15.4.x-scala2.12"]  # 런타임 사전 로드
}
```

| 상황 | Pool 사용 권장 | 이유 |
|------|--------------|------|
| 하루 10회 이상 클러스터 시작 | ✅ | 시작 시간 5분 → 30초 |
| 하루 1-2회 실행 | ❌ | 유휴 비용 > 절감 효과 |
| 다수 팀 공유 환경 | ✅ | 시작 시간 SLA 보장 |

---

## 3. Spot 인스턴스 심화 전략

### 3.1 Fault Tolerance 설계

Spot 인스턴스는 클라우드 제공자가 언제든 회수할 수 있으므로, 워크로드의 내결함성을 설계해야 합니다.

| 워크로드 유형 | Fault Tolerance 전략 | 구현 방법 |
|-------------|---------------------|----------|
| **배치 ETL** | Task 재시도 | Job 설정에서 `max_retries: 3` |
| **스트리밍** | 체크포인트 기반 복구 | `checkpointLocation` 설정 필수 |
| **ML 학습** | 체크포인트 + Fallback | epoch 단위 체크포인트 저장 |
| **SQL 쿼리** | 자동 재실행 | SQL Warehouse가 자동 처리 |

```python
# 배치 Job의 Spot 인스턴스 Fault Tolerance 설정
# databricks.yml 또는 Job 설정
{
  "job_clusters": [{
    "job_cluster_key": "etl_cluster",
    "new_cluster": {
      "num_workers": 8,
      "node_type_id": "i3.2xlarge",
      "aws_attributes": {
        "first_on_demand": 1,
        "availability": "SPOT_WITH_FALLBACK",
        "spot_bid_price_percent": 100,
        "zone_id": "auto"  # AZ 자동 선택으로 가용성 극대화
      },
      "autoscale": {
        "min_workers": 4,
        "max_workers": 12
      }
    }
  }],
  "tasks": [{
    "task_key": "etl_task",
    "max_retries": 3,           # Spot 회수 시 자동 재시도
    "min_retry_interval_millis": 60000,  # 1분 후 재시도
    "retry_on_timeout": true
  }]
}
```

### 3.2 Spot Interruption 핸들링

```python
# ML 학습 시 체크포인트 기반 Spot 내결함성
import mlflow
import torch

def train_with_checkpoint(model, train_loader, epochs, checkpoint_dir):
    start_epoch = 0
    
    # 기존 체크포인트가 있으면 복원
    checkpoint_path = f"{checkpoint_dir}/latest_checkpoint.pt"
    if os.path.exists(checkpoint_path):
        checkpoint = torch.load(checkpoint_path)
        model.load_state_dict(checkpoint['model_state'])
        start_epoch = checkpoint['epoch'] + 1
        print(f"체크포인트에서 복원: epoch {start_epoch}부터 재개")
    
    for epoch in range(start_epoch, epochs):
        train_one_epoch(model, train_loader)
        
        # 매 epoch마다 체크포인트 저장 (Spot 회수 대비)
        torch.save({
            'epoch': epoch,
            'model_state': model.state_dict(),
        }, checkpoint_path)
        
        mlflow.log_metric("epoch", epoch)
```

### 3.3 Spot 인스턴스 비용 절감 효과 분석

| 인스턴스 유형 | On-Demand ($/hr) | Spot 평균 ($/hr) | 절감율 | 회수 빈도 |
|-------------|-----------------|-----------------|-------|----------|
| **i3.xlarge** | $0.312 | $0.094 | 70% | 낮음 |
| **i3.2xlarge** | $0.624 | $0.187 | 70% | 낮음 |
| **r5.2xlarge** | $0.504 | $0.151 | 70% | 중간 |
| **c5.4xlarge** | $0.680 | $0.204 | 70% | 중간 |
| **g5.xlarge** (GPU) | $1.006 | $0.302 | 70% | 높음 |
| **p3.2xlarge** (GPU) | $3.060 | $0.918 | 70% | 높음 |

{% hint style="warning" %}
**GPU Spot 주의**: GPU 인스턴스는 Spot 가용성이 낮고 회수 빈도가 높습니다. ML 학습에 GPU Spot을 사용할 때는 반드시 체크포인트를 짧은 주기(5~10분)로 저장하고, `SPOT_WITH_FALLBACK` 을 설정하세요.
{% endhint %}

---

## 4. Photon 비용 효과 분석

### 4.1 Photon 비용 구조

Photon은 DBU 단가가 약 2배이지만, 실행 시간이 크게 단축되어 총 비용이 절감됩니다.

| 시나리오 | Classic (Non-Photon) | Photon | 비용 변화 |
|---------|---------------------|--------|----------|
| **ETL (100GB 조인 + 집계)** | 10분 × 1x DBU = 10 DBU | 3분 × 2x DBU = 6 DBU | **40% 절감** |
| **대시보드 쿼리 (동시 20명)** | 5초 × 1x = 5 DBU | 1.5초 × 2x = 3 DBU | **40% 절감** |
| **Python UDF 중심** | 10분 × 1x = 10 DBU | 10분 × 2x = 20 DBU | **100% 증가** (효과 없음) |
| **단순 SELECT** | 1초 × 1x = 1 DBU | 1초 × 2x = 2 DBU | **100% 증가** (불필요) |

{% hint style="info" %}
**Photon 적용 판단 기준**: SQL 중심 워크로드(ETL, 대시보드, 집계)에는 Photon을 적극 사용하세요. Python UDF나 ML 학습 워크로드에는 Photon이 효과가 없으므로 비활성화하는 것이 비용 효율적입니다.
{% endhint %}

### 4.2 Photon vs Non-Photon 전환 가이드

```python
# Photon 활성화/비활성화를 워크로드별로 분리하는 Compute Policy
# ETL 전용 (Photon ON)
{
  "runtime_engine": {
    "type": "fixed",
    "value": "PHOTON"
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["i3.xlarge", "i3.2xlarge"]
  }
}

# ML 전용 (Photon OFF)
{
  "runtime_engine": {
    "type": "fixed",
    "value": "STANDARD"
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["g5.xlarge", "g5.2xlarge", "p3.2xlarge"]
  }
}
```

---

## 5. Serverless vs Classic 비용 상세 비교

### 5.1 시나리오별 비용 시뮬레이션

```text
[시나리오 1: 일일 배치 ETL (하루 2시간 실행)]

Classic (i3.2xlarge × 4 workers):
  EC2: $0.624 × 4 × 2hr = $4.99/일
  DBU: 4 DBU × 4 × 2hr = 32 DBU/일
  연간: $4.99 × 365 + 32 × 365 × $0.22 = $4,389

Serverless:
  DBU: 32 DBU × 2hr = 64 Serverless DBU/일
  연간: 64 × 365 × $0.70 = $16,352
  
→ Classic이 73% 저렴 (예측 가능한 배치는 Classic 유리)

[시나리오 2: Ad-hoc 분석 (하루 평균 30분 실제 사용)]

Classic (m5.xlarge × 2, Auto-stop 10분):
  실제 가동: ~1시간/일 (유휴 대기 포함)
  EC2: $0.192 × 2 × 1hr = $0.38/일
  DBU: 2 × 2 × 1hr = 4 DBU/일
  연간: ($0.38 + 4 × $0.22) × 250 = $315

Serverless:
  실제 사용: 30분/일
  DBU: 4 × 0.5hr = 2 Serverless DBU/일
  연간: 2 × 250 × $0.70 = $350

→ 비슷하지만, Serverless는 즉시 시작 + 관리 부담 없음
```

### 5.2 의사결정 플로우차트

```text
Q1: 일일 가동률이 40% 이상인가?
  → Yes: Classic + Reserved Capacity 검토
  → No: Q2로

Q2: GPU나 커스텀 Docker가 필요한가?
  → Yes: Classic 필수
  → No: Q3로

Q3: 인프라 관리 팀이 있는가?
  → Yes: Classic이 비용 효율적
  → No: Serverless (관리 비용 절감)
```

---

## 6. 클러스터 풀 (Instance Pool) 고급 활용

### 6.1 멀티 풀 전략

조직 규모가 크면 용도별로 Instance Pool을 분리하여 관리합니다.

| Pool 이름 | 인스턴스 유형 | Min Idle | Max Capacity | 용도 |
|----------|------------|----------|-------------|------|
| **de-etl-pool** | i3.2xlarge | 2 | 20 | 데이터 엔지니어링 ETL |
| **ds-experiment-pool** | m5.xlarge | 1 | 10 | 데이터 사이언스 실험 |
| **analytics-pool** | m5.large | 2 | 8 | 분석가 인터랙티브 |
| **gpu-ml-pool** | g5.xlarge | 0 | 4 | ML 학습 (필요 시만) |

```python
# Compute Policy에서 특정 Pool 강제 지정
{
  "instance_pool_id": {
    "type": "fixed",
    "value": "pool-id-12345"
  },
  "driver_instance_pool_id": {
    "type": "fixed",
    "value": "pool-id-12345"
  }
}
```

{% hint style="info" %}
**Pool 비용 팁**: GPU Pool은 `min_idle_instances: 0` 으로 설정하세요. GPU 인스턴스는 유휴 비용이 매우 높으므로, 필요할 때만 확보하는 전략이 적합합니다. 시작 시간이 3~5분 걸리지만, GPU 유휴 비용을 절감하는 효과가 더 큽니다.
{% endhint %}

---

## 참고 링크

- [Databricks: Cluster Configuration Best Practices](https://docs.databricks.com/aws/en/compute/cluster-config-best-practices)
- [Databricks: Serverless Compute](https://docs.databricks.com/aws/en/compute/serverless/)
- [Databricks: Instance Pools](https://docs.databricks.com/aws/en/compute/pool-index)
- [Databricks: Photon](https://docs.databricks.com/aws/en/compute/photon)
- [Databricks: Compute Policies](https://docs.databricks.com/aws/en/admin/clusters/policy-definition)

---
