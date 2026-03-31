# Spot 인스턴스 활용 가이드

## Spot 인스턴스란?

**Spot 인스턴스**는 클라우드 제공업체(AWS, Azure, GCP)가 **유휴 컴퓨팅 자원을 할인된 가격으로 제공**하는 인스턴스입니다. On-Demand 대비 **60~90% 저렴**하지만, 클라우드 제공업체가 자원이 필요하면 **사전 경고 후 회수(종료)**할 수 있다는 특징이 있습니다.

> 💡 **비유**: Spot 인스턴스는 "항공사 땡처리 좌석"과 같습니다. 빈 좌석이 있으면 크게 할인하여 판매하지만, 만석이 되면 해당 좌석을 돌려받을 수 있습니다.

---

## On-Demand vs Spot 비교

| 비교 항목 | On-Demand | Spot |
|-----------|-----------|------|
| **가격** | 정가 (100%) | 할인 (10~40% 수준) |
| **가용성** | 항상 보장 | 수요에 따라 변동 |
| **종료 가능성** | 사용자가 종료할 때만 | 클라우드가 회수할 수 있음 |
| **사전 경고** | 해당 없음 | AWS: 2분 전 경고, Azure: 30초 |
| **적합한 용도** | 미션 크리티컬 워크로드 | 내결함성 있는 배치 처리 |

### 클라우드별 명칭

| 클라우드 | 명칭 | 할인율 |
|---------|------|--------|
| **AWS** | Spot Instances | 60~90% |
| **Azure** | Spot VMs (Azure Spot) | 60~80% |
| **GCP** | Preemptible VMs / Spot VMs | 60~91% |

---

## Driver는 항상 On-Demand

Databricks 클러스터에서 **Driver 노드**는 반드시 On-Demand로 실행해야 합니다. Driver가 종료되면 전체 Spark 애플리케이션이 실패하기 때문입니다.

| 노드 역할 | 권장 인스턴스 | 이유 |
|----------|-------------|------|
| **Driver** | On-Demand (필수) | Spark 드라이버가 종료되면 전체 작업이 실패합니다 |
| **Worker** | Spot (권장) | Worker가 일부 종료되어도 Spark가 자동으로 태스크를 재배치합니다 |

```yaml
# Driver는 On-Demand, Worker는 Spot
new_cluster:
  driver_node_type_id: "m5.xlarge"  # Driver
  node_type_id: "r5.xlarge"         # Worker
  num_workers: 8
  aws_attributes:
    availability: "SPOT_WITH_FALLBACK"
    first_on_demand: 1  # 첫 1개 노드(Driver)는 On-Demand
```

---

## Worker 혼합 전략

### first_on_demand 설정

`first_on_demand`는 On-Demand로 시작할 노드 수를 지정합니다. 나머지는 Spot으로 시작합니다.

```yaml
aws_attributes:
  first_on_demand: 1  # Driver만 On-Demand, 나머지 Worker는 모두 Spot
```

| `first_on_demand` | 동작 |
|-------------------|------|
| `0` | 모든 노드(Driver 포함)가 Spot입니다 (비권장) |
| `1` | Driver만 On-Demand, Worker는 모두 Spot입니다 (권장) |
| `3` | Driver + Worker 2개가 On-Demand, 나머지 Worker는 Spot입니다 |

### SPOT_WITH_FALLBACK (권장)

Spot 인스턴스를 먼저 요청하고, Spot이 부족하면 **자동으로 On-Demand로 전환**합니다.

```yaml
aws_attributes:
  availability: "SPOT_WITH_FALLBACK"
  first_on_demand: 1
  spot_bid_max_price: -1  # 시장 가격 사용 (현재 Spot 가격)
  zone_id: "auto"          # 가용 영역 자동 선택
```

| availability 옵션 | 동작 |
|-------------------|------|
| `SPOT` | Spot만 사용합니다. Spot이 없으면 클러스터가 시작되지 않습니다 |
| `SPOT_WITH_FALLBACK` | Spot 우선, 부족 시 On-Demand로 대체합니다 (권장) |
| `ON_DEMAND` | On-Demand만 사용합니다 (가장 비쌈) |

### Azure에서의 설정

```yaml
azure_attributes:
  availability: "SPOT_WITH_FALLBACK_AZURE"
  first_on_demand: 1
  spot_bid_max_price: -1  # Azure Spot 시장 가격
```

---

## Spot 인스턴스 종료 처리

### Spark의 자동 복구

Worker Spot 인스턴스가 종료되면 Spark는 다음과 같이 자동으로 대응합니다.

1. **종료된 Worker의 태스크를 다른 Worker에 재배치**합니다
2. 셔플 데이터가 손실되었으면 **해당 스테이지를 재실행**합니다
3. Auto-scaling이 활성화되어 있으면 **대체 노드를 자동 요청**합니다

### Spot 종료 시 영향

| 상황 | 영향 | 대응 |
|------|------|------|
| Worker 1개 종료 | 해당 노드의 태스크만 재실행 | 다른 노드에서 자동 재실행 |
| 다수 Worker 동시 종료 | 셔플 데이터 유실 가능 | 스테이지 재계산, 실행 시간 증가 |
| Driver 종료 | **전체 작업 실패** | Driver는 반드시 On-Demand 사용 |

### 셔플 데이터 보호

대규모 셔플이 있는 워크로드에서는 Spot 종료로 인한 셔플 재계산 비용이 클 수 있습니다.

```yaml
# 셔플 데이터가 많은 워크로드에서 권장 설정
spark_conf:
  "spark.databricks.delta.optimizeWrite.enabled": "true"  # 셔플 최소화
  "spark.sql.shuffle.partitions": "200"
```

---

## 비용 절감 효과

### 비용 계산 예시

가정: `r5.xlarge` 인스턴스, 8 Worker, 일 8시간 실행

| 구성 | 시간당 비용 (예시) | 일 비용 | 월 비용 (22일) |
|------|-----------------|--------|-------------|
| **전체 On-Demand** | $2.40 | $19.20 | $422 |
| **Driver On-Demand + Worker Spot** | $0.90 | $7.20 | $158 |
| **절감율** | — | — | **약 63% 절감** |

> 💡 위 금액은 예시이며, 실제 비용은 리전, 인스턴스 타입, Spot 시장 가격에 따라 달라집니다.

### 비용 최적화 체크리스트

| 전략 | 설정 | 예상 효과 |
|------|------|----------|
| **Spot 기본 사용** | `SPOT_WITH_FALLBACK` | 60~90% 인스턴스 비용 절감 |
| **Driver만 On-Demand** | `first_on_demand: 1` | 안정성 유지하면서 비용 절감 |
| **Auto-scaling** | `autoscale.min/max_workers` | 유휴 리소스 비용 제거 |
| **자동 종료** | `autotermination_minutes: 30` | 미사용 클러스터 비용 방지 |
| **가용 영역 자동 선택** | `zone_id: "auto"` | Spot 가용성이 높은 AZ 자동 선택 |

---

## 워크로드별 Spot 전략

| 워크로드 | Spot 권장 비율 | 이유 |
|---------|-------------|------|
| **배치 ETL Job** | Worker 100% Spot | 재시도 가능, 시간에 유연함 |
| **스트리밍** | Worker 50% Spot | 연속 실행이므로 안정성이 중요함 |
| **인터랙티브 분석** | Worker 100% Spot | 쿼리 재실행이 쉬움 |
| **ML 학습** | Worker 70% Spot | 체크포인트로 진행 상황 보존 |
| **미션 크리티컬 SLA** | Worker On-Demand | SLA 위반 방지 |

---

## 실습: Spot 최적화 클러스터 설정

```yaml
# 비용 최적화 배치 Job 클러스터
resources:
  jobs:
    daily_etl:
      name: "daily-etl-job"
      job_clusters:
        - job_cluster_key: "spot_optimized"
          new_cluster:
            spark_version: "15.4.x-photon-scala2.12"
            driver_node_type_id: "m5.xlarge"
            node_type_id: "r5.xlarge"
            autoscale:
              min_workers: 2
              max_workers: 10
            aws_attributes:
              availability: "SPOT_WITH_FALLBACK"
              first_on_demand: 1
              spot_bid_max_price: -1
              zone_id: "auto"
            runtime_engine: "PHOTON"
            spark_conf:
              "spark.databricks.delta.optimizeWrite.enabled": "true"
              "spark.databricks.delta.autoCompact.enabled": "true"

      tasks:
        - task_key: "etl_transform"
          job_cluster_key: "spot_optimized"
          notebook_task:
            notebook_path: "/Workspace/etl/transform"
          max_retries: 2  # Spot 종료 시 재시도
          min_retry_interval_millis: 60000
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Spot 인스턴스** | 유휴 자원을 할인된 가격으로 제공하지만, 회수될 수 있는 인스턴스입니다 |
| **Driver는 On-Demand** | Driver 종료 시 전체 작업이 실패하므로, 반드시 On-Demand를 사용합니다 |
| **SPOT_WITH_FALLBACK** | Spot 우선, 부족 시 On-Demand로 자동 전환하는 권장 설정입니다 |
| **first_on_demand** | On-Demand로 시작할 노드 수를 지정합니다 (1 권장) |
| **자동 복구** | Worker Spot이 종료되면 Spark가 자동으로 태스크를 재배치합니다 |
| **비용 절감** | 적절한 Spot 활용으로 인스턴스 비용을 60~90% 절감할 수 있습니다 |

---

## 참고 링크

- [Databricks: Configure cluster compute](https://docs.databricks.com/aws/en/compute/configure.html)
- [Databricks: Spot instances](https://docs.databricks.com/aws/en/compute/configure.html#spot-instances)
- [AWS: Spot Instances](https://aws.amazon.com/ec2/spot/)
- [Azure Databricks: Spot instances](https://learn.microsoft.com/en-us/azure/databricks/compute/configure#--use-spot-instances)
