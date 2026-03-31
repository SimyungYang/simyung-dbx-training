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

## Spot 인터럽트 확률과 인스턴스 유형별 특성

모든 인스턴스 유형의 Spot 인터럽트 확률이 같지는 않습니다. 인스턴스 유형, 리전, 시간대에 따라 크게 달라집니다.

### AWS Spot 인터럽트 확률 패턴

| 인스턴스 카테고리 | 대표 유형 | 인터럽트 확률 | 이유 |
|-----------------|---------|-------------|------|
| **범용 (General)** | m5.xlarge, m6i.xlarge | 중간 (5~15%) | 가장 수요가 많아 경쟁이 치열합니다 |
| **메모리 최적화** | r5.xlarge, r6i.xlarge | 낮음~중간 (3~10%) | 수요 대비 공급이 비교적 여유롭습니다 |
| **컴퓨트 최적화** | c5.xlarge, c6i.xlarge | 중간 (5~15%) | 배치 처리 수요가 높은 시간대에 경쟁 |
| **스토리지 최적화** | i3.xlarge, i4i.xlarge | 낮음 (2~8%) | 전문 용도라 경쟁이 적습니다 |
| **이전 세대** | m4.xlarge, r4.xlarge | 매우 낮음 (1~5%) | 신규 채택이 줄어 여유 용량이 많습니다 |
| **대형 인스턴스** | m5.8xlarge+ | 높음 (10~25%) | 대형 인스턴스일수록 회수 우선순위가 높습니다 |

> 💡 **실무 팁**: AWS Spot Instance Advisor(https://aws.amazon.com/ec2/spot/instance-advisor/)에서 인스턴스 유형별 인터럽트 빈도를 확인할 수 있습니다. 인터럽트율이 낮은 인스턴스를 선택하면 안정성이 높아집니다.

### 인스턴스 Fleet 전략 (다중 유형 지정)

단일 인스턴스 유형에 의존하면 Spot 풀 고갈 위험이 높아집니다. **여러 유형을 후보로 지정**하면 가용성이 크게 향상됩니다.

```yaml
# Databricks는 node_type_id에 단일 유형만 지정하지만,
# Spot 가용성을 높이려면 동등 사양의 유형을 고려합니다
# 예: m5.xlarge 대신 m5a.xlarge, m5d.xlarge, m6i.xlarge도 검토

# Spot 가용성이 높은 인스턴스 유형 조합 예시
# 1순위: r5.xlarge (메모리 최적화, Spot 안정적)
# 2순위: r5a.xlarge (AMD 프로세서, 가격 더 저렴)
# 3순위: r5d.xlarge (로컬 NVMe 포함, 셔플 성능 우수)
```

---

## Zone-aware 배치 전략

### 가용 영역(AZ)별 Spot 가격 차이

같은 리전 내에서도 가용 영역에 따라 Spot 가격과 가용성이 크게 다릅니다.

| 설정 | 동작 | 장점 | 단점 |
|------|------|------|------|
| `zone_id: "auto"` | Databricks가 최적 AZ를 자동 선택 | 가용성 최대화 | AZ가 바뀌면 EBS 볼륨 재생성 필요 |
| `zone_id: "us-east-1a"` | 고정 AZ 사용 | 예측 가능한 네트워크 지연 | 해당 AZ에 Spot이 없으면 실패 |
| `zone_id: "auto"` + `ebs_volume_type: "GENERAL_PURPOSE_SSD"` | 자동 AZ + SSD | 셔플 성능 유지 | 비용 약간 증가 |

> 💡 **권장**: 배치 Job에는 `zone_id: "auto"`를 사용하고, 스트리밍이나 S3 접근 패턴이 AZ에 민감한 워크로드에서는 고정 AZ를 사용하세요. S3 버킷과 같은 AZ에 클러스터를 배치하면 데이터 전송 비용과 지연을 줄일 수 있습니다.

---

## Spot 실패 시 Job 복구 메커니즘

### Spark 레벨 복구 (자동)

Spot 인스턴스가 회수되면 Spark는 내부적으로 다음 단계를 거칩니다.

```
1. Spot 종료 알림 수신 (AWS: 2분 전, Azure: 30초 전)
2. 종료 예정 Worker의 실행 중 Task를 "FAILED" 처리
3. Blacklist에 해당 노드 추가 (재배치 방지)
4. 다른 살아있는 Worker에 Task 재스케줄링
5. 셔플 데이터가 손실된 경우:
   - 해당 셔플을 생성한 이전 Stage를 재실행
   - 재실행된 Stage의 결과로 후속 Stage 진행
6. Auto-scaling이 활성화되어 있으면 대체 노드 요청
```

### Databricks Job 레벨 복구

Spark 내부 복구가 실패하는 경우(예: 전체 Worker 동시 회수, Driver 연결 끊김 등)에 대비하여 **Job 재시도**를 설정합니다.

```yaml
tasks:
  - task_key: "etl_transform"
    max_retries: 3              # 최대 3번 재시도
    min_retry_interval_millis: 30000  # 재시도 간 30초 대기
    retry_on_timeout: true      # 타임아웃 시에도 재시도
```

### 셔플 데이터 보호 심화

대규모 조인이나 집계에서는 셔플 데이터가 수 GB~수 TB에 달할 수 있습니다. Spot 종료로 셔플 데이터가 유실되면 **전체 Stage를 재계산**해야 하므로 비용과 시간이 급증합니다.

| 보호 전략 | 설명 | 적용 방법 |
|-----------|------|----------|
| **Optimize Write** | 셔플 파티션을 최적화하여 셔플 데이터량을 줄입니다 | `spark.databricks.delta.optimizeWrite.enabled = true` |
| **Adaptive Query Execution** | 런타임에 파티션 크기를 동적 조정합니다 | 기본 활성화 (DBR 14.3+) |
| **체크포인트** | 중간 결과를 Delta 테이블에 저장합니다 | 긴 파이프라인을 중간 테이블로 분할 |
| **브로드캐스트 조인** | 작은 테이블을 브로드캐스트하여 셔플을 제거합니다 | `spark.sql.autoBroadcastJoinThreshold = 100m` |

```python
# 긴 파이프라인을 체크포인트로 분할하여 Spot 내결함성 강화
# 단계 1: 필터링 및 변환 → 중간 테이블 저장
df_filtered = spark.table("bronze.events").filter("event_date >= '2025-01-01'")
df_filtered.write.mode("overwrite").saveAsTable("staging.events_filtered")

# 단계 2: 집계 → 최종 테이블 저장 (Spot 실패 시 단계 2만 재실행)
df_aggregated = spark.table("staging.events_filtered") \
    .groupBy("region", "product") \
    .agg(sum("revenue").alias("total_revenue"))
df_aggregated.write.mode("overwrite").saveAsTable("gold.revenue_summary")
```

---

## 비용 절감 ROI 계산 상세

### 상세 비용 비교 시나리오

**시나리오**: 일일 ETL Job, 8 Workers (r5.2xlarge), 실행 시간 4시간

| 비용 항목 | 전체 On-Demand | Driver OD + Worker Spot | 절감 |
|-----------|-------------|------------------------|------|
| **인스턴스 비용** | | | |
| - Driver (r5.2xlarge OD) | $0.504/hr x 4hr = $2.02 | $0.504/hr x 4hr = $2.02 | $0 |
| - Worker (r5.2xlarge) | OD: $0.504/hr x 8 x 4hr = $16.13 | Spot: $0.15/hr x 8 x 4hr = $4.80 | **$11.33** |
| **DBU 비용** | (동일) | (동일) | $0 |
| **일일 합계** | $18.15 + DBU | $6.82 + DBU | **$11.33/일** |
| **월간 합계 (22일)** | $399 + DBU | $150 + DBU | **$249/월** |
| **연간 합계** | $4,789 + DBU | $1,800 + DBU | **$2,989/년** |

### Spot 인터럽트에 따른 추가 비용 고려

Spot 인스턴스가 중간에 회수되면 재실행 비용이 발생합니다.

```
# 실질적 비용 절감 = 이론적 절감 - 재실행 비용
# 재실행 비용 = 재실행 확률 x 재실행 시 추가 비용

# 예시: 인터럽트율 5%, 인터럽트 시 25% 추가 실행시간
재실행 비용 = 5% x 25% x 기본 비용 = 1.25% 추가
실질 절감율 = 63% - 1.25% ≈ 61.75%
```

---

## 프로덕션 Spot 사용 판단 기준

### Spot 적합성 의사결정 플로우

| 질문 | Yes → | No → |
|------|-------|------|
| SLA가 있는 워크로드인가? | 아래 확인 | Spot 사용 |
| SLA 내에 재시도 시간이 포함되는가? | Spot + 재시도 | On-Demand |
| 데이터 유실 시 재계산 비용이 큰가? | On-Demand 또는 체크포인트+Spot | Spot 사용 |
| 스트리밍 워크로드인가? | 혼합 (50% OD + 50% Spot) | Spot 사용 |

### 프로덕션 안정성 등급별 Spot 전략

| 등급 | SLA | Spot 비율 | 추가 설정 |
|------|-----|----------|----------|
| **Tier 1 (미션 크리티컬)** | 99.9% 가용성 | Worker 0% Spot | 전체 On-Demand, Auto-scaling |
| **Tier 2 (비즈니스 중요)** | 99% 가용성, 30분 지연 허용 | Worker 50% Spot | `first_on_demand: 5` (Worker 4개 OD), 재시도 2회 |
| **Tier 3 (일반 배치)** | 95% 가용성, 2시간 지연 허용 | Worker 100% Spot | `SPOT_WITH_FALLBACK`, 재시도 3회 |
| **Tier 4 (탐색/개발)** | SLA 없음 | Worker 100% Spot | `SPOT`, 재시도 없음 |

### 클라우드별 Spot 사용 시 주의사항

| 주의사항 | AWS | Azure |
|---------|-----|-------|
| **종료 경고 시간** | 2분 | 30초 |
| **가격 변동** | 수요/공급에 따라 실시간 변동 | 정가의 최대 -100% 할인 설정 |
| **가용 영역 전략** | `zone_id: "auto"` 강력 권장 | `availability: "SPOT_WITH_FALLBACK_AZURE"` |
| **Eviction 정책** | Capacity 기반 회수 | 가격 또는 Capacity 기반 회수 |
| **Spot 블록** | 지원 (1~6시간 예약 가능, 비용 상승) | 미지원 |

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
| **인스턴스 유형별 인터럽트** | 이전 세대, 스토리지 최적화 유형이 인터럽트 확률이 낮습니다 |
| **Zone-aware 배치** | `zone_id: "auto"`로 Spot 가용성이 높은 AZ를 자동 선택합니다 |
| **체크포인트 전략** | 긴 파이프라인을 중간 테이블로 분할하면 Spot 내결함성이 높아집니다 |
| **SLA 기반 판단** | 워크로드의 SLA 등급에 따라 Spot 비율과 재시도 전략을 결정합니다 |

---

## 참고 링크

- [Databricks: Configure cluster compute](https://docs.databricks.com/aws/en/compute/configure.html)
- [Databricks: Spot instances](https://docs.databricks.com/aws/en/compute/configure.html#spot-instances)
- [AWS: Spot Instances](https://aws.amazon.com/ec2/spot/)
- [Azure Databricks: Spot instances](https://learn.microsoft.com/en-us/azure/databricks/compute/configure#--use-spot-instances)
