# Instance Pools — 클러스터 시작 시간 단축

## Instance Pool이란?

> 💡 **Instance Pool(인스턴스 풀)** 은 **유휴 클라우드 인스턴스를 미리 확보해 두는 대기실** 입니다. 클러스터가 시작될 때 새 인스턴스를 클라우드에 요청하는 대신, 풀에 미리 준비된 인스턴스를 즉시 할당받아 **클러스터 시작 시간을 크게 단축** 합니다.

### 왜 Instance Pool이 필요한가요?

Databricks 클러스터를 시작하면 다음 과정이 순차적으로 진행됩니다.

| 단계 | Pool 없을 때 | Pool 있을 때 |
|------|-------------|-------------|
| 1. 클라우드에 인스턴스 요청 | 1~5분 소요 | **건너뜀**(이미 확보됨) |
| 2. OS 부팅 및 초기화 | 1~2분 소요 | **건너뜀**(이미 부팅됨) |
| 3. Databricks Runtime 설치 | 1~3분 소요 | **건너뜀**(이미 설치됨) |
| 4. Spark 클러스터 구성 | 30초~1분 | 30초~1분 |
| **총 시작 시간** | **3~10분** | **30초~2분** |

> 💡 **비유**: Instance Pool은 "렌터카 대기장"과 같습니다. 매번 공장에서 차를 생산(인스턴스 생성)하는 대신, 대기장에 미리 준비된 차(유휴 인스턴스)를 바로 가져가는 것입니다.

---

## Instance Pool의 동작 원리

**Instance Pool 개념**:

| 항목 | 값 |
|------|-----|
| 사용 중 인스턴스 | 3개 |
| 유휴 대기 인스턴스 | 2개 |
| 최소 유휴 | 2개 |
| 최대 용량 | 10개 |

| 요청 | 결과 |
|------|------|
| 클러스터 A: 3개 요청 | 풀에서 즉시 할당 |
| 클러스터 B: 2개 요청 | 풀에서 즉시 할당 |

| 상황 | Pool 동작 |
|------|----------|
| 클러스터가 인스턴스 요청 | 풀에서 유휴 인스턴스를 즉시 할당합니다 |
| 풀의 유휴 인스턴스 부족 | 클라우드에서 새 인스턴스를 생성합니다 (시간 소요) |
| 클러스터가 인스턴스 반환 | 풀로 돌아가 유휴 상태로 대기합니다 |
| 유휴 인스턴스가 최소 수 초과 | 유휴 타임아웃 후 클라우드에 반환합니다 |

---

## Pool 설정

### UI에서 생성

Workspace → **Compute**→ **Pools** 탭 → **Create Pool** 버튼을 클릭합니다.

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

유휴 상태인 인스턴스가 **지정 시간 동안 사용되지 않으면 자동으로 종료** 되어 비용을 절감합니다. `Min Idle Instances`로 설정한 수만큼은 유지됩니다.

---

## Pool을 클러스터에 연결

### UI에서 연결

1. **Compute**→ 클러스터 생성/편집
2. **Advanced Options**→ **Instances** 탭
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

> 💡 위 예시는 `m5.xlarge` (AWS, On-Demand $0.192/hr) 기준이며, 팀 규모와 클러스터 시작 빈도에 따라 달라집니다. 일반적으로 **하루에 3회 이상 클러스터를 시작하는 팀** 이라면 Pool 사용이 경제적입니다.

### Spot 인스턴스와 조합

Instance Pool에서도 **Spot 인스턴스를 사용** 하여 비용을 추가로 절감할 수 있습니다.

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

> 💡 **Serverless가 지원되는 워크로드** 라면 Serverless를 우선 사용하고, GPU나 특수 인스턴스가 필요한 경우에만 Instance Pool을 사용하는 것이 관리 부담을 줄이는 좋은 전략입니다.

---

## Pool 사이징 공식 (유휴 인스턴스 수 최적화)

Pool의 `min_idle_instances`를 너무 높게 설정하면 유휴 비용이 낭비되고, 너무 낮으면 시작 시간 이점이 없어집니다. 최적 값을 계산하는 공식을 알아보겠습니다.

### 최적 Min Idle 계산

```
# 기본 공식
최적 Min Idle = 동시 클러스터 시작 빈도 x 평균 클러스터 Worker 수

# 시간대별 가중치 적용 공식 (더 정밀)
업무 시간 Min Idle = Peak 시간대 동시 클러스터 수 x 평균 Worker 수 x 0.7
비업무 시간 Min Idle = 0 또는 1
```

### 실무 사이징 예시

| 팀 규모 | 동시 클러스터 | 평균 Worker | 최적 Min Idle | 월 유휴 비용 (m5.xlarge) |
|---------|------------|-----------|-------------|----------------------|
| **소규모 (5명)** | 2~3개 | 3개 | 3~5개 | $100~170 |
| **중규모 (20명)** | 5~8개 | 4개 | 8~12개 | $270~410 |
| **대규모 (50명+)** | 10~20개 | 5개 | 15~25개 | $510~850 |

### 시간대별 동적 사이징

업무 시간과 비업무 시간의 Min Idle를 다르게 설정하면 비용을 크게 절감할 수 있습니다. Databricks API를 활용한 자동화가 가능합니다.

```python
# 시간대별 Pool Min Idle 자동 조정 스크립트
import requests
from datetime import datetime

POOL_ID = "0123-456789-pool01"
WORKSPACE_URL = "https://<workspace>.cloud.databricks.com"
TOKEN = "<pat-token>"

def adjust_pool_size():
    hour = datetime.now().hour

    if 8 <= hour < 19:  # 업무 시간 (08:00~19:00)
        min_idle = 10
    elif 7 <= hour < 8 or 19 <= hour < 20:  # 전환 시간
        min_idle = 3
    else:  # 야간/주말
        min_idle = 0

    response = requests.patch(
        f"{WORKSPACE_URL}/api/2.0/instance-pools/edit",
        headers={"Authorization": f"Bearer {TOKEN}"},
        json={
            "instance_pool_id": POOL_ID,
            "min_idle_instances": min_idle
        }
    )
    print(f"Pool Min Idle 설정: {min_idle} (현재 시각: {hour}시)")

adjust_pool_size()
```

---

## Pool vs Serverless 비용 비교 시나리오

### 상세 비용 비교

| 시나리오 | Pool (m5.xlarge, Min Idle 5) | Serverless Compute |
|---------|---------------------------|-------------------|
| **월 고정 비용 (유휴)** | $0.192/hr x 5 x 24hr x 30일 = **$691** | **$0**(사용한 만큼만) |
| **시작 시간** | 30초~2분 | 수초~1분 |
| **DBU 단가** | Standard DBU 단가 | Serverless DBU 단가 (약 2~2.5x) |
| **인스턴스 선택** | 원하는 유형 자유 선택 | 자동 선택 (제한적) |
| **GPU 지원** | 지원 | 제한적 |

### 손익 분기점 분석

```
# Pool이 유리한 조건
Pool 유휴 비용 < (Serverless DBU 프리미엄 x 실제 사용 DBU)

# 예시: 월 2,000 DBU 사용 기준
- Pool 유휴 비용: $691/월
- Serverless DBU 프리미엄: $0.15/DBU x 2,000 = $300/월
→ Pool 유휴 비용($691) > Serverless 프리미엄($300) → Serverless가 유리

# 예시: 월 10,000 DBU 사용 기준
- Pool 유휴 비용: $691/월
- Serverless DBU 프리미엄: $0.15/DBU x 10,000 = $1,500/월
→ Pool 유휴 비용($691) < Serverless 프리미엄($1,500) → Pool이 유리
```

> 💡 **판단 기준**: 월 DBU 사용량이 **5,000 DBU 이상** 이고 **GPU/특수 인스턴스가 필요** 하면 Pool이 유리합니다. 그 이하거나 관리 부담을 줄이고 싶다면 Serverless를 권장합니다.

---

## 멀티 Pool 전략 (Dev/Staging/Prod)

엔터프라이즈 환경에서는 용도별로 Pool을 분리하여 비용 추적과 리소스 격리를 확보합니다.

### 환경별 Pool 설계

| Pool | 인스턴스 유형 | Min Idle | Max Capacity | Spot | 용도 |
|------|------------|---------|-------------|------|------|
| **dev-general-pool** | m5.xlarge | 2 | 20 | Spot | 개발/탐색 클러스터 |
| **dev-gpu-pool** | g4dn.xlarge | 0 | 5 | Spot | ML 실험 |
| **staging-pool** | r5.2xlarge | 1 | 15 | Spot w/ Fallback | 스테이징 ETL Job |
| **prod-driver-pool** | m5.xlarge | 3 | 10 | On-Demand | 프로덕션 Driver 전용 |
| **prod-worker-pool** | r5.2xlarge | 5 | 50 | Spot w/ Fallback | 프로덕션 Worker 전용 |
| **prod-ml-pool** | g5.2xlarge | 0 | 10 | On-Demand | ML 서빙/추론 |

### Pool과 Cluster Policy 연동

Cluster Policy로 특정 Pool 사용을 강제하면 비용 거버넌스가 강화됩니다.

```json
{
  "instance_pool_id": {
    "type": "fixed",
    "value": "0123-456789-prod-worker-pool"
  },
  "driver_instance_pool_id": {
    "type": "fixed",
    "value": "0123-456789-prod-driver-pool"
  },
  "custom_tags.Environment": {
    "type": "fixed",
    "value": "production"
  },
  "custom_tags.CostCenter": {
    "type": "fixed",
    "value": "data-engineering"
  }
}
```

---

## Pool 모니터링 (시스템 테이블)

### 감사 로그로 Pool 사용 현황 분석

```sql
-- Pool 사용률 분석: 얼마나 자주 Pool에서 인스턴스를 할당받는지
SELECT
  date_trunc('hour', event_time) AS hour,
  request_params.instance_pool_id,
  COUNT(*) AS pool_requests,
  COUNT(CASE WHEN response.status_code = 200 THEN 1 END) AS successful,
  COUNT(CASE WHEN response.status_code != 200 THEN 1 END) AS failed
FROM system.access.audit
WHERE action_name LIKE '%instancePool%'
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 7 DAYS
GROUP BY 1, 2
ORDER BY 1 DESC;
```

```sql
-- 클러스터별 Pool 사용 현황
SELECT
  request_params.cluster_name,
  request_params.instance_pool_id,
  COUNT(*) AS start_count,
  AVG(TIMESTAMPDIFF(SECOND, event_time,
    LEAD(event_time) OVER (PARTITION BY request_params.cluster_id ORDER BY event_time)
  )) AS avg_runtime_seconds
FROM system.access.audit
WHERE action_name = 'create'
  AND service_name = 'clusters'
  AND request_params.instance_pool_id IS NOT NULL
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY start_count DESC;
```

### Pool 효율성 핵심 지표

| 지표 | 계산 | 건강 기준 |
|------|------|----------|
| **Pool Hit Rate** | Pool에서 할당 / 전체 요청 | > 80% (Pool 용량 충분) |
| **유휴 비용 비율** | 유휴 비용 / 전체 Pool 비용 | < 30% (너무 높으면 Min Idle 줄이기) |
| **평균 시작 시간** | 클러스터 시작 요청 ~ 사용 가능 | < 2분 (Pool 사용 시) |
| **Max Capacity 도달 빈도** | 피크 시 Pool 포화 횟수 | 주 1회 미만 (자주 달하면 Max 증가) |

---

## Spot Pool 구성 상세

### Spot Pool 설계 시 고려사항

| 항목 | 권장 | 이유 |
|------|------|------|
| **Driver Pool** | On-Demand 전용 Pool | Driver 안정성 확보 |
| **Worker Pool** | Spot with Fallback | 비용 절감 + 안정성 |
| **Min Idle + Spot** | Min Idle 인스턴스도 Spot으로 | 유휴 비용 추가 절감 |
| **Zone** | `auto` | Spot 가용성이 높은 AZ 자동 선택 |

### Spot Pool의 함정: 유휴 인스턴스 Eviction

> ⚠️ **주의**: Pool의 유휴 Spot 인스턴스도 클라우드에 의해 회수될 수 있습니다. 이 경우 Pool의 유휴 인스턴스가 0이 되어 다음 클러스터 시작 시 Pool의 이점이 사라집니다.

| 대응 전략 | 설명 |
|-----------|------|
| **Min Idle 여유분** | 예상보다 1~2개 더 높게 설정하여 Spot 회수에 대비 |
| **Spot + On-Demand 혼합** | Driver Pool은 On-Demand, Worker Pool만 Spot 사용 |
| **Fallback 설정** | `SPOT_WITH_FALLBACK`으로 Spot 부족 시 자동 On-Demand 전환 |
| **모니터링** | Pool의 실제 유휴 인스턴스 수를 모니터링하고 알림 설정 |

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
| 사이징 공식 | 동시 클러스터 수 x 평균 Worker 수 x 0.7로 Min Idle를 계산합니다 |
| 멀티 Pool | Dev/Staging/Prod 환경별로 Pool을 분리하여 거버넌스를 확보합니다 |
| Pool 모니터링 | Hit Rate, 유휴 비용 비율, 시작 시간을 핵심 지표로 추적합니다 |
| Spot Pool 주의 | 유휴 Spot 인스턴스도 회수될 수 있으므로 여유분을 설정합니다 |

---

## 참고 링크

- [Instance Pool 공식 문서](https://docs.databricks.com/aws/en/compute/pool-index.html)
- [Pool 생성 및 관리](https://docs.databricks.com/aws/en/compute/pool-create.html)
- [클러스터에 Pool 연결](https://docs.databricks.com/aws/en/compute/pool-cluster.html)
- [Spot 인스턴스 가이드](https://docs.databricks.com/aws/en/compute/spot-instances.html)
- [Serverless Compute 공식 문서](https://docs.databricks.com/aws/en/compute/serverless.html)
