# 워크스페이스 관리

## 워크스페이스 관리가 왜 중요한가요?

Databricks를 조직에 도입하면 **누가 무엇을 할 수 있는지, 비용은 어떻게 통제할 것인지, 환경은 어떻게 분리할 것인지** 를 체계적으로 관리해야 합니다. 이러한 관리 없이 플랫폼을 운영하면 비용 폭증, 보안 사고, 환경 간 간섭 등의 문제가 발생할 수 있습니다.

> 💡 **워크스페이스(Workspace)** 는 Databricks에서 사용자가 작업하는 독립적인 환경입니다. 노트북, 클러스터, Job, 데이터 등이 워크스페이스 단위로 격리됩니다.

---

## Account vs Workspace 관리 구조

Databricks는 **2계층 관리 구조** 를 사용합니다. 이 구조를 이해하는 것이 관리의 출발점입니다.

| 계층 | 역할 | 관리 대상 | 접근 방법 |
|------|------|-----------|-----------|
| **Account** | 전사 수준 관리 | 사용자, 그룹, 워크스페이스 목록, 빌링, Unity Catalog 메타스토어 | Account Console (accounts.cloud.databricks.com) |
| **Workspace** | 프로젝트 수준 관리 | 클러스터, 노트북, Job, 권한, 시크릿 | Workspace UI (xxx.cloud.databricks.com) |

### 관리자 역할

| 역할 | 범위 | 주요 권한 |
|------|------|-----------|
| **Account Admin** | 전체 Account | 워크스페이스 생성/삭제, 사용자 관리, 빌링, Unity Catalog 메타스토어 할당 |
| **Workspace Admin** | 개별 Workspace | 클러스터 정책, 기능 활성화, 워크스페이스 내 권한 관리 |
| **Metastore Admin** | Unity Catalog 메타스토어 | 카탈로그 생성, 데이터 거버넌스, Delta Sharing |

| 수준 | 역할/구성 |
|------|---------|
| **Databricks Account** | Account Admin (전사 관리) |
| **Workspace A** (개발) | Workspace Admin, 개발팀 사용자들, 클러스터/노트북/Job |
| **Workspace B** (스테이징) | Workspace Admin, QA팀 사용자들 |
| **Workspace C** (프로덕션) | Workspace Admin, 운영팀 + 서비스 프린시펄 |
| **Unity Catalog Metastore** (공유) | Metastore Admin |

---

## Workspace 생성 및 설정

### Workspace 생성 (Account Console)

Account Admin이 Account Console에서 워크스페이스를 생성합니다.

| 항목 | 예시 |
|------|------|
| Workspace 이름 | my-company-dev |
| 클라우드 리전 | ap-northeast-2 (서울) |
| 네트워크 설정 | VPC ID, Subnet ID (선택) |
| Unity Catalog | Metastore 연결 (필수 권장) |

### 권장 워크스페이스 분리 전략

| 전략 | 설명 | 적합한 경우 |
|------|------|------------|
| ** 환경별 분리** | Dev / Staging / Prod 별도 Workspace | 가장 일반적, 보안과 안정성 확보 |
| ** 팀별 분리** | 데이터 엔지니어링 / 데이터 사이언스 / 분석 팀 | 팀 간 간섭 방지, 비용 추적 용이 |
| ** 프로젝트별 분리** | 프로젝트 A / 프로젝트 B | 프로젝트 완료 시 정리 용이 |
| ** 규제별 분리** | PII 처리용 / 일반 분석용 | 규제 요건이 엄격한 산업 |

> 💡 Unity Catalog를 사용하면 여러 Workspace에서 ** 동일한 데이터 카탈로그**를 공유할 수 있으므로, Workspace를 분리해도 데이터 접근성은 유지됩니다.

---

## 사용자 및 그룹 관리

### Account 수준 사용자 관리

사용자와 그룹은 **Account 수준** 에서 관리하는 것이 권장됩니다. Account에서 생성된 사용자를 각 Workspace에 할당하는 방식입니다.

```text
Account 수준 관리 흐름:
1. Account Console에서 사용자/그룹 생성
2. 각 Workspace에 사용자/그룹 할당
3. Workspace 내에서 세부 권한 설정
```

### ID 공급자(IdP) 연동 — SCIM 프로비저닝

대규모 조직에서는 **SCIM**(System for Cross-domain Identity Management)을 통해 IdP와 자동 동기화합니다.

| IdP | 지원 방식 | 설명 |
|-----|-----------|------|
| **Microsoft Entra ID (Azure AD)** | SCIM 2.0 | 가장 많이 사용되는 연동 방식 |
| **Okta** | SCIM 2.0 | SSO + 자동 프로비저닝 |
| **OneLogin** | SCIM 2.0 | SSO + 자동 프로비저닝 |
| **Google Workspace** | SCIM 2.0 | Google 계정 기반 |

```bash
# SCIM 프로비저닝 설정 시 필요한 정보
# Account Console > Settings > Identity Providers

SCIM Endpoint: https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2
SCIM Token: <Account Admin이 생성한 PAT>
```

### 그룹 기반 권한 관리

개별 사용자 대신 ** 그룹**에 권한을 부여하는 것이 모범 사례입니다.

```sql
-- Unity Catalog에서 그룹 기반 권한 부여
GRANT USE CATALOG ON CATALOG analytics TO `data_analysts`;
GRANT USE SCHEMA ON SCHEMA analytics.gold TO `data_analysts`;
GRANT SELECT ON SCHEMA analytics.gold TO `data_analysts`;

-- 데이터 엔지니어 그룹에는 쓰기 권한까지 부여
GRANT ALL PRIVILEGES ON SCHEMA analytics.silver TO `data_engineers`;
```

---

## 클러스터 정책으로 비용 통제

** 클러스터 정책(Cluster Policy)**은 사용자가 생성할 수 있는 클러스터의 사양을 제한하여 비용을 통제하는 핵심 도구입니다.

### 클러스터 정책 예시

```json
{
  "spark_version": {
    "type": "regex",
    "pattern": "16\\.[0-9]+\\.x-scala.*",
    "defaultValue": "16.2.x-scala2.12",
    "hidden": false
  },
  "node_type_id": {
    "type": "allowlist",
    "values": [
      "i3.xlarge",
      "i3.2xlarge"
    ],
    "defaultValue": "i3.xlarge"
  },
  "autoscale.min_workers": {
    "type": "fixed",
    "value": 1,
    "hidden": true
  },
  "autoscale.max_workers": {
    "type": "range",
    "minValue": 1,
    "maxValue": 10,
    "defaultValue": 4
  },
  "autotermination_minutes": {
    "type": "range",
    "minValue": 10,
    "maxValue": 60,
    "defaultValue": 30
  },
  "custom_tags.team": {
    "type": "fixed",
    "value": "data-engineering"
  }
}
```

### 정책 적용 전략

| 대상 그룹 | 정책 이름 | 주요 제한 |
|-----------|-----------|-----------|
| **데이터 분석가** | `analyst-small` | 최대 4 워커, i3.xlarge, 30분 자동종료 |
| ** 데이터 엔지니어** | `engineer-medium` | 최대 10 워커, i3.2xlarge, 60분 자동종료 |
| ** 데이터 사이언스** | `ml-gpu` | 최대 4 GPU 워커, p3.2xlarge |
| ** 프로덕션 Job** | `production-job` | 고정 사양, 태그 필수 |

```bash
# Databricks CLI로 클러스터 정책 생성
databricks cluster-policies create --json '{
  "name": "analyst-small",
  "definition": "{\"node_type_id\":{\"type\":\"allowlist\",\"values\":[\"i3.xlarge\"]},\"autoscale.max_workers\":{\"type\":\"range\",\"minValue\":1,\"maxValue\":4}}"
}'
| 예산 범위 | 금액 |
|----------|------|
| 전사 월 예산 | $50,000 |
| dev-workspace | $10,000/월 |
| staging-workspace | $5,000/월 |
| prod-workspace | $35,000/월 |
| 알림 임계값 | 50%, 80%, 100% |

### 비용 절감 모범 사례

| 방법 | 절감 효과 | 설명 |
|------|-----------|------|
| ** 자동 종료** | ★★★ | 유휴 클러스터를 자동으로 종료합니다 |
| **Jobs Compute 사용** | ★★★ | All-Purpose 대신 Jobs Compute로 배치 작업 실행 |
| **Serverless 활용** | ★★☆ | 간헐적 워크로드에 적합, 유휴 비용 없음 |
| **Spot 인스턴스** | ★★☆ | 워커 노드에 Spot 인스턴스 사용 (최대 90% 절감) |
| ** 클러스터 정책** | ★★☆ | 과도한 클러스터 생성 방지 |
| ** 예약 인스턴스** | ★★☆ | 장기 사용 시 Reserved/Savings Plans 적용 |

---

## 태그 기반 비용 할당

** 태그(Tags)**를 활용하면 비용을 팀, 프로젝트, 환경별로 정확하게 추적할 수 있습니다.

### 태그 적용 대상

| 대상 | 태그 설정 방법 | 예시 |
|------|---------------|------|
| ** 클러스터** | 클러스터 설정 > Custom Tags | `team:analytics`, `project:recommendation` |
| **Job** | Job 설정 > Tags | `env:prod`, `owner:data-eng` |
| **SQL Warehouse** | Warehouse 설정 > Tags | `department:finance` |
| ** 클러스터 정책** | 정책에서 태그 강제 지정 | `cost_center:CC-1234` |

### 태그 전략 예시

```json
{
  "custom_tags.team": {
    "type": "fixed",
    "value": "data-engineering"
  },
  "custom_tags.env": {
    "type": "fixed",
    "value": "production"
  },
  "custom_tags.cost_center": {
    "type": "regex",
    "pattern": "CC-[0-9]{4}",
    "hidden": false
  }
}
```

### 시스템 테이블로 비용 분석

Unity Catalog의 ** 시스템 테이블**을 통해 DBU 사용량을 상세히 분석할 수 있습니다.

```sql
-- 팀별 월간 DBU 사용량 조회
SELECT
    usage_metadata.cluster_tags['team'] AS team,
    billing_origin_product AS product,
    DATE_TRUNC('month', usage_date) AS month,
    SUM(usage_quantity) AS total_dbus,
    SUM(usage_quantity * list_price) AS estimated_cost
FROM system.billing.usage u
LEFT JOIN system.billing.list_prices p
    ON u.sku_name = p.sku_name
WHERE usage_date >= '2025-01-01'
GROUP BY 1, 2, 3
ORDER BY estimated_cost DESC;
```

```sql
-- 비용 상위 10개 클러스터 조회
SELECT
    usage_metadata.cluster_id,
    ANY_VALUE(usage_metadata.cluster_tags['team']) AS team,
    SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE - INTERVAL 30 DAYS
GROUP BY usage_metadata.cluster_id
ORDER BY total_dbus DESC
LIMIT 10;
```

---

## 정리

| 관리 영역 | 핵심 포인트 |
|-----------|-------------|
| **Account vs Workspace** | Account은 전사 관리, Workspace는 프로젝트 관리 |
| ** 사용자 관리** | Account 수준에서 관리, SCIM으로 IdP 연동 |
| ** 클러스터 정책** | 사양 제한으로 비용 통제, 태그 강제로 추적 |
| ** 기능 제어** | 프로덕션 환경에서는 보안상 불필요 기능 비활성화 |
| ** 비용 관리** | 예산 설정 + 태그 + 시스템 테이블로 체계적 추적 |

---

## 참고 링크

- [Databricks Admin 공식 문서](https://docs.databricks.com/aws/en/admin/)
- [클러스터 정책](https://docs.databricks.com/aws/en/admin/clusters/policy-definition.html)
- [빌링 및 사용량](https://docs.databricks.com/aws/en/admin/account-settings/usage.html)
- [시스템 테이블 — 빌링](https://docs.databricks.com/aws/en/admin/system-tables/billing.html)
- [SCIM 프로비저닝](https://docs.databricks.com/aws/en/admin/users-groups/scim/)
