# 실전 템플릿과 모니터링

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

> 💡 **현업 팁**: `num_workers`의 `minValue`를 0으로 설정하면 **Single Node 클러스터** 를 허용합니다. 노트북 개발이나 소규모 데이터 탐색에는 Single Node로도 충분하며, 비용을 크게 절약할 수 있습니다. 실제로 개발 업무의 70% 이상은 Single Node로 처리 가능합니다.

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

> 💡 **현업 팁**: GPU 정책에서 `p4d`, `p5` 같은 대형 GPU 인스턴스를 **allowlist에서 제외** 하는 것이 핵심입니다. 대부분의 ML 실험은 `g5.xlarge`(A10G 1장)로 충분하며, 대규모 학습이 필요한 경우만 관리자가 별도 정책을 부여합니다. 이것을 안 하면 앞서 언급한 p4d.24xlarge 사고가 재현됩니다.

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
