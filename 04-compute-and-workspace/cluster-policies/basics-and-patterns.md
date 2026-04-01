# 기본 개념과 주요 패턴

## 클러스터 정책이란?

**클러스터 정책(Cluster Policy)**은 관리자가 클러스터 생성 시 ** 설정 가능한 옵션을 제한하거나 기본값을 지정**하는 JSON 기반 규칙입니다. 사용자가 과도한 리소스를 사용하거나, 비효율적인 설정으로 클러스터를 만드는 것을 방지합니다.

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
