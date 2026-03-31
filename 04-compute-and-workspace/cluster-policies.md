# 클러스터 정책(Cluster Policies) 상세

## 클러스터 정책이란?

**클러스터 정책(Cluster Policy)** 은 관리자가 클러스터 생성 시 **설정 가능한 옵션을 제한하거나 기본값을 지정**하는 JSON 기반 규칙입니다. 사용자가 과도한 리소스를 사용하거나, 비효율적인 설정으로 클러스터를 만드는 것을 방지합니다.

> 💡 **비유**: 클러스터 정책은 "회사 차량 이용 규정"과 같습니다. 직원이 차를 이용할 때 차종, 이용 시간, 주행 거리에 제한을 두어 비용을 통제하는 것처럼, 클러스터 정책은 인스턴스 타입, 노드 수, 자동 종료 시간 등을 제한합니다.

---

## 왜 클러스터 정책이 필요한가?

| 문제 | 정책으로 해결 |
|------|-------------|
| 사용자가 불필요하게 큰 클러스터를 생성합니다 | 최대 노드 수를 제한합니다 |
| 클러스터가 사용 종료 후에도 계속 실행됩니다 | 자동 종료 시간을 강제합니다 |
| 비용이 높은 On-Demand만 사용합니다 | Spot 인스턴스를 강제합니다 |
| 비용 추적이 어렵습니다 | 커스텀 태그를 강제합니다 |
| 특정 Runtime만 사용해야 합니다 | Runtime 버전을 제한합니다 |

> 🔥 **현업 사례: 정책 없이 운영하면 벌어지는 일**
>
> 한 고객사에서 클러스터 정책을 설정하지 않고 Databricks를 운영했습니다. 어느 날 한 데이터 사이언티스트가 딥러닝 실험을 위해 `p4d.24xlarge`(NVIDIA A100 GPU 8장 장착, 시간당 약 $32.77 On-Demand) 인스턴스로 워커 노드 4대 클러스터를 생성했습니다. 금요일 오후에 생성한 클러스터를 종료하지 않고 퇴근했고, 자동 종료 설정도 되어 있지 않았습니다. **주말 2일 동안 방치된 클러스터의 비용만 약 $6,300(약 850만원)** 에 달했습니다. 월요일 아침에 비용 알림을 확인하고서야 클러스터를 종료했습니다.
>
> 이 사고 이후 해당 고객사는 즉시 클러스터 정책을 도입했고, GPU 인스턴스는 관리자 승인 후에만 사용할 수 있도록 변경했습니다. **정책 도입 후 월간 컴퓨트 비용이 40% 이상 절감**되었습니다.

---

## JSON 정책 정의

### 기본 구조

클러스터 정책은 JSON 형식으로 정의되며, 각 설정 항목에 대해 **제한 유형**을 지정합니다.

```json
{
    "설정_경로": {
        "type": "제한_유형",
        "value": "값",
        "hidden": true/false
    }
}
```

### 제한 유형

| 제한 유형 | 설명 | 예시 |
|----------|------|------|
| `fixed` | 고정 값으로 변경 불가합니다 | `{"type": "fixed", "value": 4}` |
| `range` | 최솟값~최댓값 범위를 지정합니다 | `{"type": "range", "minValue": 1, "maxValue": 10}` |
| `allowlist` | 허용된 값 목록을 지정합니다 | `{"type": "allowlist", "values": ["a", "b"]}` |
| `blocklist` | 금지된 값 목록을 지정합니다 | `{"type": "blocklist", "values": ["x"]}` |
| `unlimited` | 제한 없음 (기본값 설정용) | `{"type": "unlimited", "defaultValue": 2}` |
| `regex` | 정규식 패턴을 지정합니다 | `{"type": "regex", "pattern": "15\\..+"}` |

---

## 주요 정책 패턴

### 1. 인스턴스 타입 및 노드 수 제한

```json
{
    "node_type_id": {
        "type": "allowlist",
        "values": ["m5.xlarge", "m5.2xlarge", "r5.xlarge", "r5.2xlarge"],
        "defaultValue": "m5.xlarge"
    },
    "driver_node_type_id": {
        "type": "fixed",
        "value": "m5.xlarge"
    },
    "num_workers": {
        "type": "range",
        "minValue": 1,
        "maxValue": 10,
        "defaultValue": 2
    },
    "autoscale.min_workers": {
        "type": "range",
        "minValue": 1,
        "maxValue": 4,
        "defaultValue": 1
    },
    "autoscale.max_workers": {
        "type": "range",
        "minValue": 2,
        "maxValue": 10,
        "defaultValue": 4
    }
}
```

### 2. 자동 종료 강제

```json
{
    "autotermination_minutes": {
        "type": "range",
        "minValue": 10,
        "maxValue": 120,
        "defaultValue": 30
    }
}
```

### 3. Spot 인스턴스 강제

```json
{
    "aws_attributes.availability": {
        "type": "fixed",
        "value": "SPOT_WITH_FALLBACK",
        "hidden": true
    },
    "aws_attributes.first_on_demand": {
        "type": "fixed",
        "value": 1,
        "hidden": true
    }
}
```

### 4. 커스텀 태그 강제

```json
{
    "custom_tags.team": {
        "type": "fixed",
        "value": "data-engineering"
    },
    "custom_tags.cost_center": {
        "type": "regex",
        "pattern": "CC-[0-9]{4}",
        "hidden": false
    },
    "custom_tags.environment": {
        "type": "allowlist",
        "values": ["dev", "staging", "production"],
        "defaultValue": "dev"
    }
}
```

### 5. Runtime 버전 제한

```json
{
    "spark_version": {
        "type": "regex",
        "pattern": "15\\.[0-9]+\\.x-scala2\\.12",
        "defaultValue": "15.4.x-scala2.12"
    }
}
```

### 6. 종합 정책 예제 (비용 최적화 정책)

```json
{
    "spark_version": {
        "type": "regex",
        "pattern": "15\\.[0-9]+\\.x-(scala2\\.12|photon-scala2\\.12)",
        "defaultValue": "15.4.x-photon-scala2.12"
    },
    "node_type_id": {
        "type": "allowlist",
        "values": ["m5.xlarge", "m5.2xlarge", "r5.xlarge"],
        "defaultValue": "m5.xlarge"
    },
    "num_workers": {
        "type": "range",
        "minValue": 1,
        "maxValue": 8,
        "defaultValue": 2
    },
    "autotermination_minutes": {
        "type": "range",
        "minValue": 15,
        "maxValue": 60,
        "defaultValue": 30
    },
    "aws_attributes.availability": {
        "type": "fixed",
        "value": "SPOT_WITH_FALLBACK",
        "hidden": true
    },
    "aws_attributes.first_on_demand": {
        "type": "fixed",
        "value": 1,
        "hidden": true
    },
    "custom_tags.team": {
        "type": "fixed",
        "value": "data-engineering"
    },
    "runtime_engine": {
        "type": "fixed",
        "value": "PHOTON",
        "hidden": true
    }
}
```

---

## 정책 적용 및 권한

### 정책 생성 (관리자)

1. **Workspace 설정** → **Compute** → **Policies** → **Create Policy**
2. 정책 이름과 JSON 정의를 입력합니다
3. 정책을 적용할 그룹/사용자에 권한을 부여합니다

### 정책 권한 수준

| 권한 | 설명 |
|------|------|
| **CAN_USE** | 해당 정책을 사용하여 클러스터를 생성할 수 있습니다 |
| **CAN_MANAGE** | 정책을 수정하고 삭제할 수 있습니다 |

### 정책 적용 시 사용자 경험

정책이 적용된 사용자는 클러스터 생성 시:
- `fixed` 설정은 변경할 수 없습니다 (회색으로 표시)
- `hidden: true` 설정은 UI에 표시되지 않습니다
- `range` 또는 `allowlist` 설정은 허용된 범위 내에서만 선택할 수 있습니다
- 기본값이 자동으로 채워집니다

---

## 정책 설계 가이드

| 팀/역할 | 권장 정책 | 이유 |
|---------|----------|------|
| **데이터 엔지니어** | 중간 규모 허용 + Spot 강제 | ETL 워크로드에 적합한 비용/성능 균형 |
| **데이터 사이언티스트** | GPU 허용 + ML Runtime 강제 | ML 학습에 필요한 리소스 허용 |
| **데이터 분석가** | 소규모만 허용 + SQL Warehouse 권장 | 분석 워크로드에는 작은 클러스터로 충분 |
| **프로덕션 Job** | 특정 구성 고정 + 태그 필수 | 안정성과 비용 추적 보장 |

---

## 실전 정책 템플릿

현업에서는 보통 **개발/프로덕션/ML** 세 가지 카테고리로 정책을 나누어 운영합니다. 아래는 수십 곳의 고객사에서 검증된 실전 템플릿입니다.

### 개발용 정책 (Development)

개발자가 자유롭게 실험하되, 비용 폭주를 방지하는 것이 핵심입니다.

```json
{
    "spark_version": {
        "type": "regex",
        "pattern": "(15|16)\\.[0-9]+\\.x.*",
        "defaultValue": "15.4.x-scala2.12"
    },
    "node_type_id": {
        "type": "allowlist",
        "values": ["m5.xlarge", "m5.2xlarge", "r5.xlarge"],
        "defaultValue": "m5.xlarge"
    },
    "driver_node_type_id": {
        "type": "fixed",
        "value": "m5.xlarge",
        "hidden": true
    },
    "num_workers": {
        "type": "range",
        "minValue": 0,
        "maxValue": 4,
        "defaultValue": 1
    },
    "autoscale.max_workers": {
        "type": "range",
        "minValue": 1,
        "maxValue": 4,
        "defaultValue": 2
    },
    "autotermination_minutes": {
        "type": "range",
        "minValue": 10,
        "maxValue": 60,
        "defaultValue": 30
    },
    "aws_attributes.availability": {
        "type": "fixed",
        "value": "SPOT_WITH_FALLBACK",
        "hidden": true
    },
    "aws_attributes.first_on_demand": {
        "type": "fixed",
        "value": 1,
        "hidden": true
    },
    "custom_tags.environment": {
        "type": "fixed",
        "value": "development",
        "hidden": true
    },
    "custom_tags.cost_center": {
        "type": "regex",
        "pattern": "CC-[0-9]{4}"
    }
}
```

> 💡 **현업 팁**: `num_workers`의 `minValue`를 0으로 설정하면 **Single Node 클러스터**를 허용합니다. 노트북 개발이나 소규모 데이터 탐색에는 Single Node로도 충분하며, 비용을 크게 절약할 수 있습니다. 실제로 개발 업무의 70% 이상은 Single Node로 처리 가능합니다.

### 프로덕션 Job 정책 (Production)

안정성과 비용 추적이 최우선입니다. 설정을 최대한 고정하여 예측 가능한 동작을 보장합니다.

```json
{
    "spark_version": {
        "type": "fixed",
        "value": "15.4.x-photon-scala2.12",
        "hidden": true
    },
    "node_type_id": {
        "type": "allowlist",
        "values": ["m5.2xlarge", "r5.2xlarge", "i3.2xlarge"],
        "defaultValue": "m5.2xlarge"
    },
    "autoscale.min_workers": {
        "type": "range",
        "minValue": 2,
        "maxValue": 4,
        "defaultValue": 2
    },
    "autoscale.max_workers": {
        "type": "range",
        "minValue": 4,
        "maxValue": 20,
        "defaultValue": 8
    },
    "autotermination_minutes": {
        "type": "fixed",
        "value": 15,
        "hidden": true
    },
    "aws_attributes.availability": {
        "type": "fixed",
        "value": "SPOT_WITH_FALLBACK",
        "hidden": true
    },
    "aws_attributes.first_on_demand": {
        "type": "fixed",
        "value": 1,
        "hidden": true
    },
    "runtime_engine": {
        "type": "fixed",
        "value": "PHOTON",
        "hidden": true
    },
    "custom_tags.environment": {
        "type": "fixed",
        "value": "production",
        "hidden": true
    },
    "custom_tags.team": {
        "type": "regex",
        "pattern": "[a-z]+-[a-z]+"
    },
    "custom_tags.cost_center": {
        "type": "regex",
        "pattern": "CC-[0-9]{4}"
    },
    "custom_tags.job_name": {
        "type": "unlimited"
    }
}
```

> 🔥 **현업에서는**: 프로덕션 정책에서 `spark_version`을 `fixed`로 고정하는 것이 매우 중요합니다. 한 고객사에서 프로덕션 Job이 "최신 LTS"를 자동 선택하도록 설정했다가, Databricks Runtime 버전 업데이트 후 Spark 동작 변경으로 인해 수십 개의 ETL Job이 동시에 실패한 적이 있습니다. **프로덕션에서는 반드시 버전을 명시적으로 고정하고, 별도 테스트 후 업그레이드해야 합니다.**

### ML 학습용 정책

GPU 인스턴스를 허용하되, 비용 상한을 엄격하게 제어합니다.

```json
{
    "spark_version": {
        "type": "regex",
        "pattern": "15\\.[0-9]+\\.x-gpu-ml-scala2\\.12",
        "defaultValue": "15.4.x-gpu-ml-scala2.12"
    },
    "node_type_id": {
        "type": "allowlist",
        "values": ["g5.xlarge", "g5.2xlarge", "g5.4xlarge"],
        "defaultValue": "g5.xlarge"
    },
    "driver_node_type_id": {
        "type": "fixed",
        "value": "m5.xlarge",
        "hidden": true
    },
    "num_workers": {
        "type": "range",
        "minValue": 0,
        "maxValue": 4,
        "defaultValue": 0
    },
    "autotermination_minutes": {
        "type": "range",
        "minValue": 10,
        "maxValue": 120,
        "defaultValue": 60
    },
    "aws_attributes.availability": {
        "type": "fixed",
        "value": "SPOT_WITH_FALLBACK",
        "hidden": true
    },
    "custom_tags.environment": {
        "type": "fixed",
        "value": "ml-training",
        "hidden": true
    },
    "custom_tags.experiment_name": {
        "type": "unlimited"
    }
}
```

> 💡 **현업 팁**: GPU 정책에서 `p4d`, `p5` 같은 대형 GPU 인스턴스를 **allowlist에서 제외**하는 것이 핵심입니다. 대부분의 ML 실험은 `g5.xlarge`(A10G 1장)로 충분하며, 대규모 학습이 필요한 경우만 관리자가 별도 정책을 부여합니다. 이것을 안 하면 앞서 언급한 p4d.24xlarge 사고가 재현됩니다.

---

## 정책 위반 모니터링

정책을 만들었다면, 실제로 잘 지켜지고 있는지 모니터링해야 합니다. 현업에서는 시스템 테이블을 활용하여 클러스터 생성 패턴을 추적합니다.

### 비정상 클러스터 탐지 쿼리

```sql
-- 정책 없이 생성된 클러스터 탐지
SELECT
    cluster_id,
    cluster_name,
    creator_user_name,
    cluster_source,
    node_type_id,
    num_workers,
    autotermination_minutes,
    start_time
FROM system.compute.clusters
WHERE policy_id IS NULL
    AND cluster_source != 'JOB'
    AND start_time > CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY start_time DESC;
```

### 고비용 클러스터 알림 쿼리

```sql
-- 시간당 비용이 높은 클러스터 Top 10
SELECT
    cluster_name,
    creator_user_name,
    node_type_id,
    num_workers,
    ROUND(
        TIMESTAMPDIFF(HOUR, start_time, COALESCE(end_time, CURRENT_TIMESTAMP()))
    ) AS running_hours,
    custom_tags:cost_center AS cost_center
FROM system.compute.clusters
WHERE start_time > CURRENT_DATE() - INTERVAL 30 DAYS
    AND node_type_id LIKE '%xlarge%'
ORDER BY running_hours DESC
LIMIT 10;
```

### 정책 미적용 사용자 추적

```sql
-- 정책 없이 클러스터를 자주 만드는 사용자
SELECT
    creator_user_name,
    COUNT(*) AS clusters_without_policy
FROM system.compute.clusters
WHERE policy_id IS NULL
    AND cluster_source = 'UI'
    AND start_time > CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY creator_user_name
HAVING COUNT(*) >= 3
ORDER BY clusters_without_policy DESC;
```

> 💡 **현업 팁**: 위 쿼리를 **Databricks SQL 알림(Alert)** 으로 설정하면, 정책 없는 클러스터가 생성될 때마다 Slack이나 이메일로 알림을 받을 수 있습니다. 대부분의 성숙한 조직에서는 "정책 없는 클러스터 생성 = 0건"을 KPI로 관리합니다.

### 정책 도입의 단계별 접근

현업에서 클러스터 정책을 도입할 때는 한 번에 모든 것을 제한하지 않습니다. 단계적으로 접근해야 사용자의 반발을 최소화할 수 있습니다.

| 단계 | 기간 | 조치 | 목표 |
|------|------|------|------|
| **1단계: 가시화** | 2주 | 태그 강제 정책만 도입, 모니터링 시작 | 누가 얼마나 쓰는지 파악 |
| **2단계: 기본 제한** | 2주 | 자동 종료 강제, 최대 노드 수 제한 | 방치 클러스터 제거 |
| **3단계: 역할별 정책** | 2주 | 개발/프로덕션/ML 정책 분리 적용 | 용도에 맞는 리소스 할당 |
| **4단계: 강화** | 지속 | Spot 강제, GPU 제한, 정기 감사 | 비용 최적화 극대화 |

> 🔥 **이것을 안 하면**: 첫날부터 모든 것을 제한하면 사용자들이 "Databricks가 불편하다"고 느끼게 되고, 결국 정책을 우회하는 방법을 찾거나 다른 도구로 이탈합니다. **1단계에서 현황을 데이터로 보여주고**, 비용이 얼마나 낭비되고 있는지 사용자 스스로 인지하게 하는 것이 가장 효과적입니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **클러스터 정책** | JSON으로 정의된 클러스터 생성 규칙으로, 비용과 거버넌스를 제어합니다 |
| **제한 유형** | fixed, range, allowlist, blocklist, regex, unlimited 6가지 유형이 있습니다 |
| **hidden 속성** | `true`로 설정하면 사용자에게 해당 옵션이 보이지 않습니다 |
| **권한 관리** | CAN_USE 권한이 있는 사용자만 해당 정책으로 클러스터를 생성할 수 있습니다 |

---

## 참고 링크

- [Databricks: Cluster policies](https://docs.databricks.com/aws/en/admin/clusters/policies.html)
- [Databricks: Cluster policy definitions](https://docs.databricks.com/aws/en/admin/clusters/policy-definition.html)
- [Databricks: Cluster policy permissions](https://docs.databricks.com/aws/en/admin/clusters/policies.html#manage-cluster-policy-permissions)
- [Azure Databricks: Cluster policies](https://learn.microsoft.com/en-us/azure/databricks/admin/clusters/policies)
