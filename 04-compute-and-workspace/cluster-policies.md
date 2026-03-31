# 클러스터 정책(Cluster Policies) 상세

## 클러스터 정책이란?

**클러스터 정책(Cluster Policy)**은 관리자가 클러스터 생성 시 **설정 가능한 옵션을 제한하거나 기본값을 지정**하는 JSON 기반 규칙입니다. 사용자가 과도한 리소스를 사용하거나, 비효율적인 설정으로 클러스터를 만드는 것을 방지합니다.

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
