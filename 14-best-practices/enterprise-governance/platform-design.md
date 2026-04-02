# 플랫폼 설계

## 이 문서에서 다루는 내용

대규모 조직에서 Databricks 플랫폼을 안전하고 체계적으로 운영하기 위한 거버넌스 전략을 다룹니다. 플랫폼 아키텍처 설계부터 접근 제어, 데이터 분류, 감사/컴플라이언스, 데이터 품질 관리, CI/CD 자동화까지 엔터프라이즈 수준의 모범 사례를 제공합니다.

---

## 1. 플랫폼 설계

### 1.1 Workspace 분리 전략

Workspace를 어떻게 분리하느냐에 따라 보안, 비용 관리, 운영 효율성이 결정됩니다.

| 전략 | 구조 | 장점 | 단점 | 적합한 조직 |
|------|------|------|------|-----------|
| **환경별 분리** | Dev / Staging / Prod | 환경 간 완전 격리, 규제 충족 용이 | Workspace 수 증가 | 규제 산업 (금융, 의료) |
| **팀별 분리** | DE / DS / Analytics | 팀별 독립 운영, 비용 추적 용이 | 크로스팀 협업 어려움 | 팀 간 독립성 높은 조직 |
| **하이브리드** | 환경 × 팀 매트릭스 | 최대 유연성 | 관리 복잡도 높음 | 대기업 (1000+ 사용자) |
| **단일 Workspace** | 하나의 Workspace | 관리 단순, Unity Catalog로 격리 | 규모 확장 시 한계 | 스타트업, 소규모 팀 |

> 💡 **권장 패턴**: 대부분의 엔터프라이즈는 **환경별 분리 (Dev/Staging/Prod)** 를 기본으로 하되, Unity Catalog의 카탈로그 수준에서 팀/프로젝트를 분리하는 방식이 효과적입니다.

| 수준 | 구성 |
|------|------|
| **Unity Catalog Metastore** | Account 수준 - 모든 Workspace에서 공유 |
| Dev Workspace | 개발팀 자유 실험, 느슨한 정책 |
| Staging Workspace | 통합 테스트, 프로덕션과 유사한 정책 |
| Prod Workspace | 엄격한 접근 제어, 감사 로그 필수 |

### 1.2 Unity Catalog 네임스페이스 설계

**3단계 네임스페이스**: `catalog.schema.table`

| 수준 | 명명 규칙 | 예시 | 용도 |
|------|----------|------|------|
| **Catalog** | `{환경}_{도메인}` 또는 `{환경}` | `prod_sales`, `dev_hr` | 환경 + 비즈니스 도메인 격리 |
| **Schema** | `{데이터 계층}` 또는 `{팀}` | `bronze`, `silver`, `gold` | Medallion 계층 또는 팀 분리 |
| **Table** | `{엔티티}_{설명}` | `orders_daily`, `customers_dim` | 비즈니스 엔티티 |

**명명 규칙 표준안**

```sql
-- 카탈로그 명명 패턴
-- 패턴 1: 환경 중심 (소규모 조직)
-- prod / dev / staging

-- 패턴 2: 환경 + 도메인 (중대형 조직, 권장)
-- prod_sales / prod_marketing / dev_sales / dev_marketing

-- 패턴 3: 도메인 중심 (데이터 메시 구조)
-- sales / marketing / finance (환경은 Workspace로 분리)

-- 스키마 명명: Medallion 계층
CREATE SCHEMA prod_sales.bronze;    -- 원본 데이터
CREATE SCHEMA prod_sales.silver;    -- 정제된 데이터
CREATE SCHEMA prod_sales.gold;      -- 비즈니스 집계
CREATE SCHEMA prod_sales.sandbox;   -- 개발/실험용

-- 테이블 명명 규칙
-- {엔티티}_{유형}: orders_fact, customers_dim, daily_revenue_agg
-- 접미사: _fact (팩트), _dim (디멘션), _agg (집계), _raw (원본)
```

### 1.3 멀티 클라우드 / 멀티 리전 거버넌스

| 고려사항 | 전략 | 구현 방법 |
|---------|------|----------|
| **데이터 주권** | 리전별 Metastore 분리 | 리전별 Unity Catalog Metastore 생성 |
| **크로스 리전 공유** | Delta Sharing | 리전 간 데이터를 복제 없이 공유 |
| **통합 거버넌스** | Account 수준 관리 | Account Console에서 전체 Workspace 관리 |
| **재해 복구** | 멀티 리전 복제 | 핵심 데이터를 다른 리전에 비동기 복제 |

> ⚠️ **주의**: Unity Catalog의 리니지(Lineage)와 접근 제어는 **동일 Metastore 내에서만** 작동합니다. 크로스 Metastore 데이터 공유 시에는 Delta Sharing을 사용하며, 리니지 추적이 단절될 수 있습니다.

---

## 2. Workspace 분리 전략 상세

### 2.1 환경별 분리 (Dev / Staging / Prod)

가장 일반적인 패턴으로, 소프트웨어 개발 생명주기(SDLC)에 맞춰 Workspace를 분리합니다.

| Workspace | 용도 | 접근 정책 | 클러스터 정책 | 비용 관리 |
|-----------|------|----------|-------------|----------|
| **Dev** | 개발, 실험, PoC | 개발자 전원 접근 허용 | Single Node 권장, GPU 제한 | 예산 상한 설정 (태그 기반) |
| **Staging** | 통합 테스트, UAT | CI/CD Service Principal + QA 팀 | Prod와 동일한 정책 (축소 스케일) | Dev 대비 30~50% 예산 |
| **Prod** | 프로덕션 워크로드 | Service Principal 중심, 개인 접근 최소화 | 엄격한 Compute Policy | SLA 기반 예산 |

```text
[환경별 분리 아키텍처]

Account Console (단일 관리 포인트)
  ├── Unity Catalog Metastore (공유)
  │     ├── dev 카탈로그     ← Dev Workspace에서만 접근
  │     ├── staging 카탈로그  ← Staging Workspace에서만 접근
  │     └── prod 카탈로그    ← Prod Workspace에서만 접근
  │
  ├── Dev Workspace
  │     ├── 개발자 그룹 (자유 실험)
  │     └── 클러스터: Single Node, Auto-stop 10분
  │
  ├── Staging Workspace
  │     ├── CI/CD Service Principal
  │     └── 클러스터: Prod 축소 스케일
  │
  └── Prod Workspace
        ├── Service Principal (워크로드 실행)
        ├── 관리자 그룹 (최소 인원)
        └── 클러스터: 엄격한 Compute Policy
```

### 2.2 팀별 분리

팀 간 독립성이 높고, 비용 추적이 중요한 조직에 적합합니다.

| Workspace | 팀 | 카탈로그 | 공유 데이터 접근 |
|-----------|-----|---------|---------------|
| **DE Workspace** | 데이터 엔지니어링 | `de_dev`, `de_prod` | `shared` 카탈로그 읽기 |
| **DS Workspace** | 데이터 사이언스 | `ds_dev`, `ds_prod` | `shared` 카탈로그 읽기 |
| **Analytics Workspace** | 분석가 | `analytics` | `shared` + `de_prod` 읽기 |

{% hint style="warning" %}
**팀별 분리의 함정**: 팀 간 데이터 공유가 빈번하면 Workspace 분리가 오히려 병목이 됩니다. 이 경우 단일 Workspace + Unity Catalog 카탈로그 수준 분리가 더 효과적입니다.
{% endhint %}

### 2.3 프로젝트별 분리

규제 산업(금융, 의료)에서 프로젝트별 데이터 격리가 필수인 경우 사용합니다.

| 프로젝트 | Workspace | 격리 수준 | 근거 |
|---------|-----------|----------|------|
| **고객 신용 평가** | `credit-scoring-prod` | 네트워크 격리 + CMK | 금융 규제 (여신전문금융업법) |
| **환자 데이터 분석** | `patient-analytics-prod` | VPC 격리 + IP 제한 | HIPAA / 개인정보보호법 |
| **일반 분석** | `analytics-prod` | 표준 보안 | 규제 대상 아님 |

---

## 3. Account Console 설정 가이드

### 3.1 핵심 설정 체크리스트

| 설정 영역 | 항목 | 권장 설정 | 위치 |
|----------|------|----------|------|
| **계정 보안** | 2FA/MFA 강제 | 전체 사용자 필수 | Settings → Authentication |
| **SSO** | SAML 2.0 / OIDC | 사내 IdP 연동 (Okta, Azure AD) | Settings → Single sign-on |
| **사용자 프로비저닝** | SCIM | IdP에서 자동 동기화 | Settings → User provisioning |
| **감사 로그** | 활성화 | 전체 Workspace 대상 | Settings → Audit logs |
| **IP 접근 제한** | IP Access List | 사무실 + VPN IP만 허용 | Settings → IP access lists |

### 3.2 IP Access List 설정

```json
{
  "label": "corporate-office-and-vpn",
  "list_type": "ALLOW",
  "ip_addresses": [
    "203.0.113.0/24",
    "198.51.100.0/24",
    "10.0.0.0/8"
  ]
}
```

| IP Access List 유형 | 동작 | 용도 |
|-------------------|------|------|
| **ALLOW** | 목록의 IP만 접근 허용 | 사무실/VPN IP 화이트리스트 |
| **BLOCK** | 목록의 IP 접근 차단 | 특정 IP 블랙리스트 |

{% hint style="info" %}
**Account 수준 vs Workspace 수준**: Account Console에서 설정한 IP Access List는 모든 Workspace에 적용됩니다. Workspace 수준에서 추가 제한을 설정할 수 있지만, Account 수준 제한을 완화할 수는 없습니다.
{% endhint %}

---

## 4. CMK (Customer Managed Key) 설정

### 4.1 CMK 적용 대상

| 암호화 대상 | 설명 | CMK 적용 방법 |
|-----------|------|-------------|
| **관리형 서비스 (Managed Services)** | 노트북 소스, 시크릿, 쿼리 결과 | Account Console → Encryption keys |
| **Workspace 스토리지** | DBFS Root, 시스템 데이터 | Workspace 생성 시 CMK 지정 |
| **Unity Catalog 데이터** | 관리형 테이블, 볼륨 | External Location에 CMK가 적용된 S3/ADLS 사용 |

### 4.2 AWS KMS 설정 예시

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDatabricksUsage",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::414351767826:root"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowDatabricksGrant",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::414351767826:root"
      },
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    }
  ]
}
```

{% hint style="warning" %}
**CMK 삭제 주의**: CMK를 삭제하면 해당 키로 암호화된 모든 데이터에 접근할 수 없습니다. KMS 키 삭제 대기 기간(7~30일)을 반드시 설정하고, 키 순환(rotation) 정책을 사용하세요.
{% endhint %}

---

## 5. 네트워크 보안

### 5.1 VPC Peering vs PrivateLink

| 항목 | VPC Peering | PrivateLink |
|------|-----------|------------|
| **트래픽 경로** | VPC 간 직접 통신 | AWS 백본 네트워크 |
| **보안 수준** | 중 (VPC 간 라우팅) | 높음 (엔드포인트 기반) |
| **IP 충돌 가능성** | CIDR 겹침 시 불가 | 없음 |
| **양방향 통신** | 양방향 | 단방향 (소비자 → 제공자) |
| **적합한 시나리오** | 온프레미스 DB 접근 | Databricks Control Plane 접근 |

### 5.2 PrivateLink 구성 (AWS)

| PrivateLink 유형 | 용도 | VPC Endpoint Service |
|----------------|------|---------------------|
| **Front-end (Workspace)** | 사용자 → Workspace UI/API | Workspace 접근 |
| **Back-end (Data Plane)** | Data Plane → Control Plane | 클러스터 ↔ 관리 서비스 통신 |
| **SCC Relay** | Secure Cluster Connectivity | NAT 없이 아웃바운드 통신 |

```text
[PrivateLink 아키텍처]

사용자 VPC                           Databricks Control Plane
┌──────────────────┐                ┌─────────────────────┐
│                  │                │                     │
│  사용자 브라우저 ──┼── Front-end ──→│  Workspace UI/API   │
│                  │  PrivateLink   │                     │
│                  │                │                     │
│  Data Plane VPC ─┼── Back-end ───→│  Control Plane      │
│  (클러스터)       │  PrivateLink   │  Services           │
│                  │                │                     │
│  Data Plane VPC ─┼── SCC Relay ──→│  SCC Relay          │
│                  │  PrivateLink   │  Endpoint           │
└──────────────────┘                └─────────────────────┘
```

{% hint style="info" %}
**PrivateLink 비용**: VPC Endpoint당 월 $7.30 + 데이터 처리 비용($0.01/GB)이 발생합니다. 보안 요구사항이 높은 Prod Workspace에만 적용하고, Dev에는 공용 네트워크를 사용하는 하이브리드 전략도 고려하세요.
{% endhint %}

### 5.3 Azure VNet Injection + Private Endpoint

| 구성 요소 | 설명 | 서브넷 요구사항 |
|----------|------|-------------|
| **Container 서브넷** | 클러스터 노드 배치 | /26 이상 (최소 64 IP) |
| **Host 서브넷** | 관리 인터페이스 | /26 이상 |
| **Private Endpoint** | Workspace 접근 | 별도 서브넷 권장 |

---

## 6. IaC (Infrastructure as Code) 패턴

### 6.1 Terraform 기반 플랫폼 관리

```hcl
# Workspace 생성 (AWS)
resource "databricks_mws_workspaces" "prod" {
  account_id     = var.databricks_account_id
  workspace_name = "prod-analytics"
  aws_region     = "ap-northeast-2"

  credentials_id           = databricks_mws_credentials.prod.credentials_id
  storage_configuration_id = databricks_mws_storage_configurations.prod.storage_configuration_id
  network_id               = databricks_mws_networks.prod.network_id

  # CMK 설정
  managed_services_customer_managed_key_id = databricks_mws_customer_managed_keys.managed.customer_managed_key_id
  storage_customer_managed_key_id          = databricks_mws_customer_managed_keys.storage.customer_managed_key_id

  custom_tags = {
    Environment = "production"
    CostCenter  = "platform-team"
    ManagedBy   = "terraform"
  }
}

# Unity Catalog 카탈로그 생성
resource "databricks_catalog" "prod_sales" {
  provider = databricks.workspace
  name     = "prod_sales"
  comment  = "프로덕션 영업 데이터 카탈로그"

  properties = {
    "owner_team" = "data-engineering"
    "environment" = "production"
  }
}

# Compute Policy 설정
resource "databricks_cluster_policy" "dev_policy" {
  provider = databricks.workspace
  name     = "dev-standard-policy"

  definition = jsonencode({
    "autotermination_minutes" : {
      "type" : "range",
      "maxValue" : 30,
      "defaultValue" : 10
    },
    "num_workers" : {
      "type" : "range",
      "maxValue" : 8,
      "defaultValue" : 2
    },
    "node_type_id" : {
      "type" : "allowlist",
      "values" : ["i3.xlarge", "i3.2xlarge", "m5.xlarge"]
    },
    "spark_version" : {
      "type" : "regex",
      "pattern" : "15\\.[0-9]+\\.x-scala.*"
    }
  })
}
```

### 6.2 DAB (Databricks Asset Bundles) 패턴

```yaml
# databricks.yml — 멀티 환경 번들 설정
bundle:
  name: sales-pipeline

workspace:
  host: https://prod.cloud.databricks.com

targets:
  dev:
    workspace:
      host: https://dev.cloud.databricks.com
    default: true
    resources:
      jobs:
        daily_etl:
          job_clusters:
            - job_cluster_key: etl_cluster
              new_cluster:
                num_workers: 2
                node_type_id: i3.xlarge
                spark_version: 15.4.x-scala2.12

  staging:
    workspace:
      host: https://staging.cloud.databricks.com
    resources:
      jobs:
        daily_etl:
          job_clusters:
            - job_cluster_key: etl_cluster
              new_cluster:
                num_workers: 4
                node_type_id: i3.2xlarge

  prod:
    workspace:
      host: https://prod.cloud.databricks.com
    run_as:
      service_principal_name: svc-prod-etl
    resources:
      jobs:
        daily_etl:
          job_clusters:
            - job_cluster_key: etl_cluster
              new_cluster:
                num_workers: 8
                node_type_id: i3.4xlarge
                autoscale:
                  min_workers: 4
                  max_workers: 16
```

{% hint style="info" %}
**IaC 권장 조합**: Workspace 인프라(VPC, IAM, Workspace 생성)는 **Terraform** 으로, 데이터 파이프라인과 Job 정의는 **DAB** 로 관리하는 이중 구조가 가장 효과적입니다. Terraform은 플랫폼 팀이, DAB는 각 데이터 팀이 관리합니다.
{% endhint %}

---

## 참고 링크

- [Databricks: Workspace Administration](https://docs.databricks.com/aws/en/admin/)
- [Databricks: PrivateLink](https://docs.databricks.com/aws/en/security/network/classic/privatelink)
- [Databricks: Customer Managed Keys](https://docs.databricks.com/aws/en/security/keys/customer-managed-keys)
- [Databricks: Terraform Provider](https://registry.terraform.io/providers/databricks/databricks/latest/docs)
- [Databricks: Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)

---
