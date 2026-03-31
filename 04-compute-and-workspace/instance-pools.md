# Instance Pools — 클러스터 시작 시간 단축

## Instance Pool이란?

> 💡 **Instance Pool(인스턴스 풀)**은 **유휴 클라우드 인스턴스를 미리 확보해 두는 대기실**입니다. 클러스터가 시작될 때 새 인스턴스를 클라우드에 요청하는 대신, 풀에 미리 준비된 인스턴스를 즉시 할당받아 **클러스터 시작 시간을 크게 단축**합니다.

### 왜 Instance Pool이 필요한가요?

Databricks 클러스터를 시작하면 다음 과정이 순차적으로 진행됩니다.

| 단계 | Pool 없을 때 | Pool 있을 때 |
|------|-------------|-------------|
| 1. 클라우드에 인스턴스 요청 | 1~5분 소요 | **건너뜀** (이미 확보됨) |
| 2. OS 부팅 및 초기화 | 1~2분 소요 | **건너뜀** (이미 부팅됨) |
| 3. Databricks Runtime 설치 | 1~3분 소요 | **건너뜀** (이미 설치됨) |
| 4. Spark 클러스터 구성 | 30초~1분 | 30초~1분 |
| **총 시작 시간** | **3~10분** | **30초~2분** |

> 💡 **비유**: Instance Pool은 "렌터카 대기장"과 같습니다. 매번 공장에서 차를 생산(인스턴스 생성)하는 대신, 대기장에 미리 준비된 차(유휴 인스턴스)를 바로 가져가는 것입니다.

---

## Instance Pool의 동작 원리

```
[Instance Pool]
┌─────────────────────────────────────┐
│  유휴 인스턴스:  ■ ■ ■ □ □          │  ■ = 사용 중, □ = 유휴 대기
│  최소 유휴: 2개                      │
│  최대 용량: 10개                     │
└─────────────────────────────────────┘
        │                    │
   클러스터 A 요청 3개    클러스터 B 요청 2개
   → 풀에서 즉시 할당     → 풀에서 즉시 할당
```

| 상황 | Pool 동작 |
|------|----------|
| 클러스터가 인스턴스 요청 | 풀에서 유휴 인스턴스를 즉시 할당합니다 |
| 풀의 유휴 인스턴스 부족 | 클라우드에서 새 인스턴스를 생성합니다 (시간 소요) |
| 클러스터가 인스턴스 반환 | 풀로 돌아가 유휴 상태로 대기합니다 |
| 유휴 인스턴스가 최소 수 초과 | 유휴 타임아웃 후 클라우드에 반환합니다 |

---

## Pool 설정

### UI에서 생성

Workspace → **Compute** → **Pools** 탭 → **Create Pool** 버튼을 클릭합니다.

### 주요 설정 항목

| 설정 | 설명 | 권장값 |
|------|------|--------|
| **Pool Name** | 풀 이름 | 용도를 명확히 (예: `data-eng-pool`) |
| **Min Idle Instances** | 항상 유지할 최소 유휴 인스턴스 수 | 팀 크기에 따라 2~5개 |
| **Max Capacity** | 풀이 보유할 수 있는 최대 인스턴스 수 | 동시 사용 피크에 맞춰 설정 |
| **Node Type** | 인스턴스 유형 (CPU/메모리 사양) | 워크로드에 맞는 타입 선택 |
| **Idle Instance Auto Termination** | 유휴 인스턴스의 자동 종료 시간 | 업무 시간 패턴에 따라 10~60분 |
| **Preloaded Spark Version** | 미리 설치할 Databricks Runtime 버전 | 팀에서 사용하는 주요 버전 |

### API / JSON 설정 예시

```json
{
  "instance_pool_name": "data-engineering-pool",
  "node_type_id": "m5.xlarge",
  "min_idle_instances": 3,
  "max_capacity": 20,
  "idle_instance_autotermination_minutes": 30,
  "preloaded_spark_versions": [
    "15.4.x-scala2.12"
  ],
  "aws_attributes": {
    "availability": "SPOT_WITH_FALLBACK",
    "zone_id": "auto",
    "spot_bid_max_price": -1
  }
}
```

### 설정 항목 상세

#### Min Idle Instances (최소 유휴 인스턴스)

| 값 | 동작 | 비용 영향 |
|----|------|----------|
| **0** | 풀에 유휴 인스턴스를 유지하지 않습니다. 첫 요청 시 클라우드에서 생성합니다 | 유휴 비용 없음, 시작 시간 절감 없음 |
| **2~5** | 항상 2~5개의 인스턴스가 대기합니다 | 유휴 비용 발생, 빠른 시작 |
| **팀원 수** | 모든 팀원이 동시에 클러스터를 시작해도 즉시 할당 | 유휴 비용 높음, 최대 속도 |

#### Max Capacity (최대 용량)

풀이 동시에 보유할 수 있는 인스턴스의 상한입니다. 이 값을 초과하면 클러스터는 풀을 우회하여 직접 클라우드에서 인스턴스를 요청합니다.

#### Idle Instance Auto Termination (유휴 자동 종료)

유휴 상태인 인스턴스가 **지정 시간 동안 사용되지 않으면 자동으로 종료**되어 비용을 절감합니다. `Min Idle Instances`로 설정한 수만큼은 유지됩니다.

---

## Pool을 클러스터에 연결

### UI에서 연결

1. **Compute** → 클러스터 생성/편집
2. **Advanced Options** → **Instances** 탭
3. **Driver Pool** 및 **Worker Pool** 드롭다운에서 풀을 선택합니다

### 클러스터 JSON 설정

```json
{
  "cluster_name": "etl-cluster",
  "spark_version": "15.4.x-scala2.12",
  "instance_pool_id": "0123-456789-pool01",
  "driver_instance_pool_id": "0123-456789-pool01",
  "num_workers": 4,
  "autoscale": {
    "min_workers": 2,
    "max_workers": 8
  }
}
```

### Databricks Asset Bundle 설정

```yaml
# databricks.yml
resources:
  clusters:
    etl_cluster:
      cluster_name: "etl-cluster"
      instance_pool_id: ${var.pool_id}
      driver_instance_pool_id: ${var.pool_id}
      num_workers: 4
```

> ⚠️ **주의**: Pool을 사용하는 클러스터에서는 `node_type_id`를 **별도로 지정할 수 없습니다**. 인스턴스 유형은 Pool의 설정을 따릅니다.

---

## 비용 분석

### 유휴 인스턴스 비용 vs 시작 시간 절감

| 시나리오 | Pool 없음 | Pool 있음 (Min Idle: 3) |
|---------|----------|------------------------|
| 인스턴스 비용 (유휴) | $0 | $0.19/hr x 3 = **$0.57/hr** |
| 클러스터 시작 대기 | 5분 x 하루 10회 = **50분/일** | 1분 x 하루 10회 = **10분/일** |
| 일일 생산성 절감 | 50분 x $50/hr = **$41.67** | 10분 x $50/hr = **$8.33** |
| **일일 순이익** | - | **$41.67 - $8.33 - $13.68 = $19.66** |

> 💡 위 예시는 `m5.xlarge` (AWS, On-Demand $0.192/hr) 기준이며, 팀 규모와 클러스터 시작 빈도에 따라 달라집니다. 일반적으로 **하루에 3회 이상 클러스터를 시작하는 팀**이라면 Pool 사용이 경제적입니다.

### Spot 인스턴스와 조합

Instance Pool에서도 **Spot 인스턴스를 사용**하여 비용을 추가로 절감할 수 있습니다.

```json
{
  "aws_attributes": {
    "availability": "SPOT_WITH_FALLBACK",
    "spot_bid_max_price": -1
  }
}
```

| 조합 | 설명 |
|------|------|
| **Pool + On-Demand** | 안정적이지만 비용이 높습니다 |
| **Pool + Spot** | 비용 절감 + 빠른 시작. Worker에 권장합니다 |
| **Pool + Spot with Fallback** | Spot 부족 시 On-Demand로 자동 전환합니다 (가장 권장) |

---

## 모범 사례

| 항목 | 권장 사항 |
|------|----------|
| **용도별 Pool 분리** | 데이터 엔지니어링용, ML 학습용, 대화형 분석용으로 Pool을 분리합니다 |
| **Runtime 사전 로드** | `preloaded_spark_versions`에 팀에서 사용하는 Runtime을 지정합니다 |
| **적절한 Min Idle** | 업무 시간 패턴에 맞춰 설정합니다. 야간에는 0이 되도록 할 수 있습니다 |
| **Max Capacity 설정** | 비용 폭발 방지를 위해 반드시 상한을 설정합니다 |
| **Spot 활용** | Worker Pool은 Spot with Fallback을 사용하여 비용을 절감합니다 |
| **Driver Pool 분리** | Driver는 안정성이 중요하므로 On-Demand Pool을 별도로 구성합니다 |
| **태그 활용** | Pool에 팀/프로젝트 태그를 붙여 비용을 추적합니다 |
| **Cluster Policy 연동** | 클러스터 정책에서 특정 Pool 사용을 강제하여 거버넌스를 적용합니다 |

---

## Pool vs Serverless 비교

Databricks Serverless Compute도 빠른 시작 시간을 제공합니다. 두 옵션을 비교해 보겠습니다.

| 비교 항목 | Instance Pool | Serverless Compute |
|-----------|-------------|-------------------|
| **시작 시간** | 30초~2분 | 수초~1분 |
| **인프라 관리** | Pool 설정 필요 (인스턴스 유형, 크기 등) | 관리 불필요 (완전 자동) |
| **비용 모델** | 유휴 인스턴스 비용 + 사용 비용 | 사용한 만큼만 과금 |
| **커스터마이징** | 인스턴스 유형, GPU, 스토리지 등 세밀한 설정 가능 | 제한적 |
| **GPU 지원** | 지원 | 제한적 |
| **적합한 시나리오** | GPU 워크로드, 특수 인스턴스 필요 시 | 일반 SQL/Python 워크로드 |

> 💡 **Serverless가 지원되는 워크로드**라면 Serverless를 우선 사용하고, GPU나 특수 인스턴스가 필요한 경우에만 Instance Pool을 사용하는 것이 관리 부담을 줄이는 좋은 전략입니다.

---

## 정리

| 개념 | 핵심 내용 |
|------|----------|
| Instance Pool | 유휴 인스턴스를 미리 확보하여 클러스터 시작 시간을 단축합니다 |
| Min Idle | 항상 유지할 최소 유휴 인스턴스 수를 설정합니다 |
| Max Capacity | 풀의 최대 인스턴스 수를 제한하여 비용을 관리합니다 |
| 유휴 타임아웃 | 사용되지 않는 인스턴스를 자동 종료하여 비용을 절감합니다 |
| Runtime 사전 로드 | Spark Runtime을 미리 설치하여 추가 시간을 절약합니다 |
| Spot 조합 | Pool에 Spot 인스턴스를 사용하면 비용을 더 절감합니다 |

---

## 참고 링크

- [Instance Pool 공식 문서](https://docs.databricks.com/aws/en/compute/pool-index.html)
- [Pool 생성 및 관리](https://docs.databricks.com/aws/en/compute/pool-create.html)
- [클러스터에 Pool 연결](https://docs.databricks.com/aws/en/compute/pool-cluster.html)
- [Spot 인스턴스 가이드](https://docs.databricks.com/aws/en/compute/spot-instances.html)
- [Serverless Compute 공식 문서](https://docs.databricks.com/aws/en/compute/serverless.html)
