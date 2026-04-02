# 정책 적용과 설계 가이드

## 왜 클러스터 정책(Cluster Policies)이 필요한가

클러스터 정책 없이 운영하면 세 가지 핵심 문제가 발생합니다.

**비용 폭주**— 사용자가 제한 없이 대형 인스턴스를 선택하거나, 자동 종료(Auto Termination)를 비활성화한 채 방치하면 월말 클라우드 청구서가 예상을 크게 초과합니다. Databricks 고객 사례에서 정책 미적용 워크스페이스는 동일 팀 대비 평균 2~3배 높은 컴퓨팅 비용이 발생하는 경우가 보고됩니다.

**보안 위험**— 루트 볼륨 암호화 미설정, 공개 IP 허용, 네트워크 설정 임의 변경 등이 사용자 실수로 발생할 수 있습니다. 클러스터 정책은 이러한 설정을 강제(fixed)하거나 허용 범위를 제한하여 컴플라이언스 위반을 사전에 차단합니다.

**일관성 부재**— 팀원마다 다른 런타임 버전, 다른 인스턴스 타입, 다른 태그 체계를 사용하면 재현성(Reproducibility)이 떨어지고 비용 추적(Cost Attribution)이 어려워집니다.

---

## 정책 적용 및 권한

### 정책 생성 절차 (관리자)

1. **Workspace 설정**→ **Compute**→ **Policies**→ **Create Policy**
2. 정책 이름과 JSON 정의를 입력합니다
3. 정책을 적용할 그룹/사용자에 권한을 부여합니다

### 정책 권한 수준

| 권한 | 설명 |
|------|------|
| **CAN_USE** | 해당 정책을 사용하여 클러스터를 생성할 수 있습니다 |
| **CAN_MANAGE** | 정책을 수정하고 삭제할 수 있습니다 (주로 관리자에게 부여) |

### 정책 상속과 다중 정책 적용 시 우선순위

사용자가 여러 그룹에 속하고 각 그룹이 서로 다른 정책을 갖는 경우, 정책은 **OR 방식** 으로 작동합니다. 즉, 사용자는 자신이 접근 가능한 정책 목록 중 하나를 선택하여 클러스터를 생성합니다. 단일 클러스터에 두 개의 정책이 동시에 적용되지는 않습니다.

Workspace 수준에서 `Default Policy`를 별도로 설정하면, **CAN_USE** 가 부여되지 않은 일반 사용자에게도 기본 정책이 자동으로 적용됩니다. 이 Default Policy는 전체 사용자 대상 최소 안전망 역할을 합니다.

### 정책 적용 시 사용자 경험

정책이 적용된 사용자는 클러스터 생성 UI에서:

- `fixed` 설정은 변경할 수 없습니다 (회색으로 표시)
- `hidden: true` 설정은 UI에 아예 표시되지 않습니다
- `range` 또는 `allowlist` 설정은 허용된 범위 내에서만 선택 가능합니다
- `defaultValue`가 지정된 항목은 기본값이 자동으로 채워집니다

---

## 실전 정책 설계 패턴

역할별 권장 정책과 JSON 예시를 제시합니다. 각 예시는 실제 운영 환경에 바로 적용하거나 출발점(Baseline)으로 활용할 수 있습니다.

### 데이터 엔지니어(Data Engineer) 정책

ETL/ELT 파이프라인 중심으로 중간 규모 클러스터를 허용하되, Spot 인스턴스를 강제하여 비용을 절감합니다.

```json
{
  "spark_version": {
    "type": "unlimited",
    "defaultValue": "15.4.x-scala2.12"
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["m5d.xlarge", "m5d.2xlarge", "m5d.4xlarge"]
  },
  "num_workers": {
    "type": "range",
    "minValue": 1,
    "maxValue": 10
  },
  "aws_attributes.availability": {
    "type": "fixed",
    "value": "SPOT_WITH_FALLBACK"
  },
  "autotermination_minutes": {
    "type": "range",
    "minValue": 30,
    "maxValue": 120,
    "defaultValue": 60
  },
  "custom_tags.team": {
    "type": "fixed",
    "value": "data-engineering"
  },
  "custom_tags.cost_center": {
    "type": "regex",
    "pattern": "CC-[0-9]{4}"
  }
}
```

**핵심 의도:**`SPOT_WITH_FALLBACK` 으로 비용을 최대 70% 절감하면서, `cost_center` 태그를 정규식(Regex)으로 강제하여 비용 추적을 보장합니다.

---

### 데이터 사이언티스트(Data Scientist) 정책

ML 학습을 위한 GPU 인스턴스 허용 및 Databricks ML Runtime 강제 적용입니다.

```json
{
  "spark_version": {
    "type": "regex",
    "pattern": ".*-ml-.*"
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["m5d.2xlarge", "g4dn.xlarge", "g4dn.2xlarge", "g4dn.4xlarge"]
  },
  "num_workers": {
    "type": "range",
    "minValue": 0,
    "maxValue": 4
  },
  "aws_attributes.availability": {
    "type": "allowlist",
    "values": ["ON_DEMAND", "SPOT_WITH_FALLBACK"]
  },
  "autotermination_minutes": {
    "type": "range",
    "minValue": 30,
    "maxValue": 240,
    "defaultValue": 120
  },
  "custom_tags.project": {
    "type": "unlimited",
    "isOptional": false
  }
}
```

**핵심 의도:**`spark_version` 에 `-ml-` 패턴을 강제하여 ML Runtime 외의 런타임 선택을 차단합니다. GPU 학습은 인터럽트 시 재현이 어려우므로 `ON_DEMAND` 옵션도 허용합니다.

---

### 데이터 분석가(Data Analyst) 정책

Ad-hoc 분석 용도의 소형 클러스터만 허용하고, SQL Warehouse 사용을 유도합니다.

```json
{
  "spark_version": {
    "type": "unlimited",
    "defaultValue": "15.4.x-scala2.12"
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["m5d.large", "m5d.xlarge"]
  },
  "num_workers": {
    "type": "range",
    "minValue": 1,
    "maxValue": 3
  },
  "aws_attributes.availability": {
    "type": "fixed",
    "value": "SPOT_WITH_FALLBACK"
  },
  "autotermination_minutes": {
    "type": "fixed",
    "value": 30
  },
  "custom_tags.team": {
    "type": "fixed",
    "value": "analytics"
  }
}
```

**핵심 의도:** 자동 종료를 30분으로 고정하여 방치 클러스터를 원천 차단합니다. 대규모 분석은 SQL Warehouse로 유도하여 All-Purpose 클러스터 남용을 방지합니다.

---

### 프로덕션 Job 정책

Job Cluster는 구성 일관성과 비용 추적이 최우선입니다.

```json
{
  "spark_version": {
    "type": "allowlist",
    "values": ["15.4.x-scala2.12", "14.3.x-scala2.12"]
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["m5d.xlarge", "m5d.2xlarge", "m5d.4xlarge", "r5d.2xlarge"]
  },
  "aws_attributes.availability": {
    "type": "fixed",
    "value": "SPOT_WITH_FALLBACK"
  },
  "aws_attributes.spot_bid_price_percent": {
    "type": "range",
    "minValue": 50,
    "maxValue": 100,
    "defaultValue": 100
  },
  "custom_tags.environment": {
    "type": "fixed",
    "value": "production"
  },
  "custom_tags.job_name": {
    "type": "unlimited",
    "isOptional": false
  },
  "custom_tags.owner": {
    "type": "unlimited",
    "isOptional": false
  }
}
```

**핵심 의도:**`job_name` 과 `owner` 태그를 필수화하여 장애 발생 시 책임 소재를 명확히 합니다. 런타임 버전을 LTS 두 개로 제한하여 프로덕션 안정성을 확보합니다.

---

## 장단점과 트레이드오프

### 엄격한 정책(Restrictive Policy)

| 항목 | 내용 |
|------|------|
| **장점** | 비용 예측 가능성 향상, 보안 컴플라이언스 보장, 일관된 환경 |
| **단점** | 사용자 민원 증가, 특수 워크로드 대응 지연, 정책 예외 처리 부담 |
| **적합한 상황** | 규제 산업, 비용 한도가 명확한 팀, 신규 사용자 온보딩 초기 |

### 느슨한 정책(Permissive Policy)

| 항목 | 내용 |
|------|------|
| **장점** | 사용자 자율성 보장, 빠른 실험(Experimentation) 가능 |
| **단점** | 비용 폭주 위험, 보안 설정 누락 가능성, 비용 추적 어려움 |
| **적합한 상황** | 소수의 신뢰할 수 있는 고급 사용자, 격리된 샌드박스 환경 |

### 정책 관리 오버헤드

정책 수가 늘어날수록 관리 부담이 증가합니다. 팀이 10개를 초과하면 정책도 비례해서 늘어나는 경향이 있습니다. 이를 방지하려면 **역할 기반 정책(Role-Based Policy)** 을 기준으로 설계하고, 팀별 커스터마이징은 태그(Custom Tags)로 처리하는 방식을 권장합니다.

---

## 베스트 프랙티스(Best Practices)

### 정책 명명 규칙(Naming Convention)

일관된 명명 규칙을 통해 정책 목록이 늘어나도 파악이 쉬워집니다.

```
{역할}-{용도}-{환경}
예시:
  de-etl-prod          # 데이터 엔지니어, ETL, 프로덕션
  ds-ml-training       # 데이터 사이언티스트, ML 학습
  da-adhoc-dev         # 데이터 분석가, Ad-hoc, 개발
  job-batch-prod       # 배치 Job, 프로덕션
```

### 점진적 제한(Progressive Restriction)

정책을 처음부터 매우 엄격하게 설정하면 사용자 불만과 예외 요청이 폭주합니다. 다음 순서로 단계적으로 제한을 강화합니다.

1. **1단계**— 자동 종료(Auto Termination)만 강제 (빠른 비용 절감 효과)
2. **2단계**— 태그 필수화 (비용 추적 기반 마련)
3. **3단계**— 인스턴스 타입 제한 (과도한 리소스 선택 차단)
4. **4단계**— Spot 비율 및 런타임 버전 제한 (최적화 완성)

### 태그 필수화(Mandatory Tagging)

`custom_tags` 의 `isOptional: false` 설정으로 팀/프로젝트/비용 센터 태그를 필수화합니다. 이를 통해 클라우드 비용 청구서를 팀별, 프로젝트별로 분류할 수 있습니다.

### 감사(Audit) 방법

- **시스템 테이블(System Tables)** 의 `system.compute.clusters` 뷰를 쿼리하여 정책 미적용 클러스터, 태그 누락 클러스터를 주기적으로 감사합니다.
- Databricks Audit Logs를 SIEM 또는 S3에 수집하여 `clusters/create` 이벤트를 모니터링합니다.
- 정책 변경(`policies/create`, `policies/edit`) 이벤트를 알림(Alert)으로 설정하여 무단 변경을 탐지합니다.

---

## 흔한 실수(Common Mistakes)

**정책 없이 운영**— "나중에 정책을 적용하면 된다"는 접근은 위험합니다. 기존 클러스터는 정책 적용 후에도 소급 적용되지 않으므로, 운영 초기부터 최소한의 Default Policy라도 설정해야 합니다.

**All-Purpose 클러스터에 무제한 허용**— All-Purpose 클러스터는 대화형(Interactive) 작업용이므로, Job 워크로드와 분리된 정책을 별도로 운영해야 합니다. All-Purpose 에서 대규모 배치 Job을 실행하는 것은 비용 낭비입니다.

**Spot 비율 무시**— `SPOT_WITH_FALLBACK` 만 설정하고 `spot_bid_price_percent` 를 관리하지 않으면, On-Demand 인스턴스로 항상 폴백(Fallback)되어 Spot 절감 효과가 없는 경우가 생깁니다.

**정책 JSON 검증 생략**— 정책을 저장할 때 Databricks가 기본 문법 오류는 잡아주지만, 논리 오류(예: `minValue > maxValue`)는 잡지 못합니다. 정책 적용 전 반드시 테스트 사용자로 클러스터 생성을 검증해야 합니다.

**CAN_MANAGE 과도 부여**— 정책 수정 권한은 관리자 그룹에만 부여하고, 일반 사용자에게는 `CAN_USE` 만 부여합니다. `CAN_MANAGE` 를 팀 리드에게 위임하면 정책 일관성이 깨질 수 있습니다.

---

## 참고 링크

- [Databricks Docs — Cluster Policies](https://docs.databricks.com/en/administration-guide/clusters/policies.html)
- [Databricks Docs — Policy Definition Reference](https://docs.databricks.com/en/administration-guide/clusters/policy-definition.html)
- [Databricks Docs — Cluster Access Control](https://docs.databricks.com/en/security/auth/access-control/cluster-acl.html)
- [Databricks Docs — System Tables: Compute](https://docs.databricks.com/en/admin/system-tables/compute.html)
- [Terraform Registry — databricks_cluster_policy](https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/cluster_policy)
