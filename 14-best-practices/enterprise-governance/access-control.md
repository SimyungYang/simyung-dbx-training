# 접근 제어 설계

## 2. 접근 제어 설계

### 2.1 그룹 기반 권한 관리

**핵심 원칙**: 개별 사용자에게 직접 권한을 부여하지 않습니다. 반드시 **그룹** 을 통해 관리합니다.

| 역할 그룹 | 카탈로그 권한 | 스키마 권한 | 테이블 권한 | 비고 |
|----------|-------------|-----------|-----------|------|
| `data-engineers` | USE CATALOG | USE SCHEMA, CREATE TABLE | SELECT, MODIFY | Bronze/Silver 쓰기 |
| `data-analysts` | USE CATALOG | USE SCHEMA | SELECT | Gold 읽기 전용 |
| `data-scientists` | USE CATALOG | USE SCHEMA, CREATE TABLE | SELECT, MODIFY | Silver 읽기, Sandbox 쓰기 |
| `platform-admins` | ALL PRIVILEGES | ALL PRIVILEGES | ALL PRIVILEGES | 최소 인원으로 제한 |
| `svc-prod-pipeline` | USE CATALOG | USE SCHEMA, CREATE TABLE | SELECT, MODIFY | Service Principal |

| 단계 | 원칙 | 설명 |
|------|------|------|
| 기본 | 모든 권한 거부 (Deny by Default) | 시작점 |
| 1단계 | 역할 기반 접근 제어 (RBAC) | 그룹별 필요 권한만 부여 |
| 2단계 | 데이터 수준 접근 제어 | 행/열 수준 필터링 |
| 3단계 | 정기 감사 | 분기별 권한 리뷰, 미사용 권한 회수 |

> 💡 **중요**: `USE CATALOG`과 `USE SCHEMA`는 해당 객체를 **탐색** 할 수 있는 권한일 뿐, 데이터를 읽을 수 있는 `SELECT` 권한을 포함하지 않습니다. 반드시 별도로 부여해야 합니다.

### 2.3 Row Filter / Column Mask 적용 전략

민감한 데이터를 행 수준, 열 수준에서 제어합니다.

```sql
-- Row Filter: 사용자 그룹에 따라 볼 수 있는 행 제한
CREATE FUNCTION prod_sales.gold.region_filter(region STRING)
RETURN
  CASE
    WHEN IS_MEMBER('global-analysts') THEN TRUE
    WHEN IS_MEMBER('korea-team') AND region = 'KR' THEN TRUE
    WHEN IS_MEMBER('japan-team') AND region = 'JP' THEN TRUE
    ELSE FALSE
  END;

ALTER TABLE prod_sales.gold.sales_data
SET ROW FILTER prod_sales.gold.region_filter ON (region);

-- Column Mask: 민감한 컬럼 값을 마스킹
CREATE FUNCTION prod_sales.gold.mask_email(email STRING)
RETURN
  CASE
    WHEN IS_MEMBER('pii-authorized') THEN email
    ELSE CONCAT(LEFT(email, 2), '****@', SPLIT(email, '@')[1])
  END;

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN email SET MASK prod_sales.gold.mask_email;

-- Column Mask: 주민등록번호 마스킹
CREATE FUNCTION prod_sales.gold.mask_ssn(ssn STRING)
RETURN
  CASE
    WHEN IS_MEMBER('pii-authorized') THEN ssn
    ELSE CONCAT('***-**-', RIGHT(ssn, 4))
  END;

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN ssn SET MASK prod_sales.gold.mask_ssn;
```

| PII/PHI 유형 | 마스킹 전략 | 예시 |
|-------------|-----------|------|
| 이메일 | 앞 2글자만 표시 | `si****@company.com` |
| 전화번호 | 뒷 4자리만 표시 | `***-****-5678` |
| 주민등록번호/SSN | 뒷 4자리만 표시 | `***-**-1234` |
| 신용카드 | 뒷 4자리만 표시 | `****-****-****-5678` |
| 주소 | 시/도만 표시 | `서울특별시 ***` |
| 진료기록(PHI) | 완전 숨김 | `[REDACTED]` |

### 2.4 Service Principal 관리 체계

| 용도 | Service Principal 명 | 권한 수준 | 비고 |
|------|---------------------|----------|------|
| 프로덕션 ETL | `svc-prod-etl` | Silver/Gold MODIFY | Jobs에서만 사용 |
| 프로덕션 ML | `svc-prod-ml` | Model Registry MANAGE | Serving 배포용 |
| CI/CD 파이프라인 | `svc-cicd` | Workspace 관리 | DAB 배포용 |
| 외부 연동 | `svc-external-api` | 특정 테이블 SELECT | API 접근용 |
| 모니터링 | `svc-monitoring` | 시스템 테이블 SELECT | 비용/감사 모니터링 |

```python
# Service Principal으로 프로덕션 Job 실행 (Terraform 예시)
resource "databricks_job" "daily_etl" {
  name = "daily-etl-pipeline"

  # Service Principal으로 실행 (개인 계정 아님)
  run_as {
    service_principal_name = "svc-prod-etl"
  }

  task {
    task_key = "etl_bronze_to_silver"
    pipeline_task {
      pipeline_id = databricks_pipeline.bronze_to_silver.id
    }
  }
}
```

> ⚠️ **핵심 원칙**: 프로덕션 워크로드는 반드시 **Service Principal** 으로 실행합니다. 개인 계정으로 실행하면 퇴사/이동 시 파이프라인이 중단될 수 있습니다.

---

## 3. Row Filter / Column Mask 실전 패턴

### 3.1 다중 조건 Row Filter

실무에서는 단순 그룹 멤버십 외에 복잡한 조건이 필요한 경우가 많습니다.

```sql
-- 패턴 1: 부서별 + 직급별 접근 제어
CREATE FUNCTION prod_hr.gold.salary_filter(
  department STRING,
  salary DECIMAL(18,2)
)
RETURN
  CASE
    -- HR 관리자는 전체 접근
    WHEN IS_MEMBER('hr-admins') THEN TRUE
    -- 팀장은 자기 부서만 접근
    WHEN IS_MEMBER('team-leads') AND department = current_user_attribute('department') THEN TRUE
    -- 일반 직원은 급여 정보가 포함된 행 접근 불가
    WHEN salary IS NOT NULL AND NOT IS_MEMBER('salary-authorized') THEN FALSE
    ELSE TRUE
  END;

ALTER TABLE prod_hr.gold.employee_data
SET ROW FILTER prod_hr.gold.salary_filter ON (department, salary);

-- 패턴 2: 시간 기반 접근 제어 (데이터 엠바고)
CREATE FUNCTION prod_finance.gold.embargo_filter(report_date DATE)
RETURN
  CASE
    -- IR팀은 항상 접근 가능
    WHEN IS_MEMBER('ir-team') THEN TRUE
    -- 일반 사용자는 발표 후에만 접근 가능 (7일 엠바고)
    WHEN report_date <= current_date() - INTERVAL 7 DAYS THEN TRUE
    ELSE FALSE
  END;

ALTER TABLE prod_finance.gold.quarterly_results
SET ROW FILTER prod_finance.gold.embargo_filter ON (report_date);
```

### 3.2 동적 Column Mask

```sql
-- 패턴 3: 집계 수준에 따른 마스킹
CREATE FUNCTION prod_sales.gold.mask_revenue(
  revenue DECIMAL(18,2),
  region STRING
)
RETURN
  CASE
    -- 재무팀은 정확한 수치 조회
    WHEN IS_MEMBER('finance-team') THEN revenue
    -- 영업팀은 자기 지역만 정확한 수치
    WHEN IS_MEMBER('sales-team') AND region = current_user_attribute('region') THEN revenue
    -- 기타는 범위로 표시
    WHEN revenue < 1000000 THEN 999999.99  -- "100만 미만"
    WHEN revenue < 10000000 THEN 9999999.99  -- "1000만 미만"
    ELSE 99999999.99  -- "1억 미만"
  END;

-- 패턴 4: 해시 기반 비식별화 (분석 가능한 마스킹)
CREATE FUNCTION prod_sales.gold.hash_customer_id(customer_id STRING)
RETURN
  CASE
    WHEN IS_MEMBER('pii-authorized') THEN customer_id
    ELSE SHA2(CONCAT(customer_id, 'salt_key_2026'), 256)
    -- 해시값은 일관되므로 JOIN/GROUP BY 가능 (원본 복원 불가)
  END;

ALTER TABLE prod_sales.gold.transactions
ALTER COLUMN customer_id SET MASK prod_sales.gold.hash_customer_id;
```

{% hint style="info" %}
**해시 기반 마스킹의 장점**: SHA2 해시를 사용하면 원본 데이터를 볼 수 없으면서도, 동일한 고객의 거래를 GROUP BY나 JOIN으로 분석할 수 있습니다. 분석 가능성과 프라이버시를 동시에 확보하는 패턴입니다.
{% endhint %}

---

## 4. ABAC 태그 기반 정책

### 4.1 태그 체계 설계

Unity Catalog의 태그(Tags)를 활용하여 속성 기반 접근 제어(ABAC)를 구현합니다.

```sql
-- 테이블/컬럼에 태그 부여
ALTER TABLE prod_sales.gold.customers
SET TAGS ('data_classification' = 'confidential', 'pii_level' = 'high');

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN email SET TAGS ('pii_type' = 'email');

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN phone SET TAGS ('pii_type' = 'phone');

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN ssn SET TAGS ('pii_type' = 'ssn', 'sensitivity' = 'critical');
```

| 태그 키 | 값 예시 | 용도 |
|--------|--------|------|
| `data_classification` | `public`, `internal`, `confidential`, `restricted` | 데이터 등급 분류 |
| `pii_level` | `none`, `low`, `medium`, `high` | 개인정보 민감도 |
| `pii_type` | `email`, `phone`, `ssn`, `credit_card` | PII 유형 |
| `retention_days` | `30`, `90`, `365`, `permanent` | 보존 기간 |
| `owner_team` | `sales`, `hr`, `finance` | 소유 팀 |

### 4.2 태그 기반 자동 마스킹 정책

```sql
-- Security Policy로 태그 기반 자동 마스킹 (Unity Catalog 기능)
-- pii_type 태그가 있는 모든 컬럼에 자동으로 마스킹 적용
CREATE SECURITY POLICY pii_auto_mask
ON COLUMN
WHERE TAG('pii_type') IS NOT NULL
USING (
  CASE
    WHEN IS_MEMBER('pii-authorized') THEN ORIGINAL_VALUE
    WHEN TAG('pii_type') = 'email' THEN CONCAT(LEFT(ORIGINAL_VALUE, 2), '****')
    WHEN TAG('pii_type') = 'phone' THEN CONCAT('***-****-', RIGHT(ORIGINAL_VALUE, 4))
    WHEN TAG('pii_type') = 'ssn' THEN '***-**-****'
    ELSE '[REDACTED]'
  END
);
```

{% hint style="warning" %}
**Security Policy는 Preview 기능입니다**: 태그 기반 자동 마스킹 Security Policy는 현재 Public Preview입니다. 프로덕션에 적용하기 전에 테스트 환경에서 충분히 검증하세요. 현재는 개별 테이블에 Row Filter/Column Mask를 수동 설정하는 것이 안정적입니다.
{% endhint %}

---

## 5. 크로스 Workspace 권한 동기화

### 5.1 Account 수준 그룹 관리

```text
[권한 동기화 아키텍처]

Account Console
  ├── Account 그룹 (IdP에서 SCIM 동기화)
  │     ├── data-engineers (Account 수준)
  │     ├── data-analysts (Account 수준)
  │     └── platform-admins (Account 수준)
  │
  ├── Dev Workspace
  │     └── Account 그룹 자동 상속
  │
  ├── Staging Workspace
  │     └── Account 그룹 자동 상속
  │
  └── Prod Workspace
        └── Account 그룹 자동 상속 + Workspace 수준 추가 제한
```

| 그룹 유형 | 생성 위치 | 동기화 | 권장 |
|----------|---------|-------|------|
| **Account 그룹** | Account Console | 모든 Workspace에 자동 | ✅ 권장 |
| **Workspace 로컬 그룹** | 개별 Workspace | 해당 Workspace만 | ⚠️ 특수 경우만 |

```sql
-- Account 그룹에 Unity Catalog 권한 부여
-- 이 권한은 Metastore를 공유하는 모든 Workspace에서 유효

-- Dev 카탈로그: 엔지니어에게 전체 접근
GRANT ALL PRIVILEGES ON CATALOG dev TO `data-engineers`;

-- Prod 카탈로그: 엔지니어에게 읽기만
GRANT USE CATALOG ON CATALOG prod TO `data-engineers`;
GRANT USE SCHEMA ON SCHEMA prod.gold TO `data-engineers`;
GRANT SELECT ON SCHEMA prod.gold TO `data-engineers`;

-- Prod 카탈로그: Service Principal에게 쓰기
GRANT USE CATALOG ON CATALOG prod TO `svc-prod-etl`;
GRANT ALL PRIVILEGES ON SCHEMA prod.silver TO `svc-prod-etl`;
GRANT ALL PRIVILEGES ON SCHEMA prod.gold TO `svc-prod-etl`;
```

---

## 6. 외부 사용자 접근 제어 (Delta Sharing)

### 6.1 Delta Sharing 권한 모델

```sql
-- 1. Share 생성 (공유할 데이터 정의)
CREATE SHARE partner_analytics_share;

-- 2. Share에 테이블 추가 (필요한 테이블만 선택적으로)
ALTER SHARE partner_analytics_share
ADD TABLE prod_sales.gold.monthly_summary
  COMMENT '파트너에게 월별 매출 요약만 공유';

-- Row Filter가 적용된 테이블도 공유 가능
-- 파트너는 필터링된 결과만 볼 수 있음
ALTER SHARE partner_analytics_share
ADD TABLE prod_sales.gold.regional_data
  PARTITION (region = 'KR');  -- 한국 데이터만 공유

-- 3. Recipient 생성 (외부 조직)
CREATE RECIPIENT partner_company
  USING ID 'partner-org-id'
  COMMENT '파트너사 분석팀';

-- 4. Share를 Recipient에 부여
GRANT SELECT ON SHARE partner_analytics_share TO RECIPIENT partner_company;
```

| Delta Sharing 유형 | 수신자 플랫폼 | 인증 방식 | 비고 |
|------------------|-----------|----------|------|
| **Databricks-to-Databricks** | Databricks | Unity Catalog 통합 | 가장 간편 |
| **Open Sharing** | Spark, Pandas, 기타 | Bearer Token | 다양한 플랫폼 지원 |

{% hint style="info" %}
**Delta Sharing 거버넌스**: Delta Sharing으로 공유된 데이터는 수신자가 복사할 수 있습니다. 민감한 데이터는 Column Mask가 적용된 View를 공유하거나, 집계된 Gold 테이블만 공유하세요.
{% endhint %}

---

## 7. Audit Log 활용한 권한 감사

### 7.1 시스템 테이블을 활용한 감사

```sql
-- 최근 30일간 권한 변경 이력 조회
SELECT
  event_time,
  user_identity.email AS changed_by,
  action_name,
  request_params.securable_type,
  request_params.securable_full_name,
  request_params.principal,
  request_params.changes
FROM system.access.audit
WHERE action_name IN (
  'grantPermission',
  'revokePermission',
  'updatePermissions'
)
AND event_time >= current_date() - INTERVAL 30 DAYS
ORDER BY event_time DESC;

-- 미사용 권한 탐지: 90일간 접근하지 않은 테이블의 SELECT 권한
SELECT
  g.principal,
  g.securable_full_name,
  g.privilege,
  MAX(a.event_time) AS last_access
FROM system.information_schema.table_privileges g
LEFT JOIN system.access.audit a
  ON a.request_params.securable_full_name = g.securable_full_name
  AND a.user_identity.email = g.principal
  AND a.action_name = 'commandSubmit'
GROUP BY g.principal, g.securable_full_name, g.privilege
HAVING MAX(a.event_time) IS NULL
   OR MAX(a.event_time) < current_date() - INTERVAL 90 DAYS;
```

### 7.2 정기 감사 체크리스트

| 감사 항목 | 주기 | SQL/방법 | 조치 |
|----------|------|---------|------|
| **과도한 권한 보유자** | 월 1회 | ALL PRIVILEGES 보유 사용자 조회 | 최소 권한으로 축소 |
| **미사용 권한** | 분기 1회 | 90일간 미접근 테이블 권한 | 권한 회수 |
| **Service Principal 키 순환** | 분기 1회 | 키 생성일 90일 초과 확인 | 새 키 발급, 이전 키 비활성화 |
| **외부 공유 현황** | 월 1회 | Delta Sharing Recipient 목록 | 불필요한 공유 해제 |
| **관리자 그룹 멤버** | 월 1회 | platform-admins 그룹 멤버 확인 | 불필요한 멤버 제거 |

```sql
-- ALL PRIVILEGES 보유자 감사 (최소화해야 함)
SELECT
  principal,
  securable_type,
  securable_full_name,
  privilege
FROM system.information_schema.table_privileges
WHERE privilege = 'ALL PRIVILEGES'
ORDER BY principal;

-- Workspace 관리자 목록 확인
SELECT
  user_name,
  groups
FROM system.information_schema.catalog_privileges
WHERE privilege = 'ALL PRIVILEGES'
AND securable_type = 'CATALOG';
```

{% hint style="warning" %}
**감사 로그 보존**: 시스템 테이블의 감사 로그는 기본 365일 보존됩니다. 규제 요건에 따라 장기 보존이 필요하면, 감사 로그를 별도 스토리지(S3/ADLS)로 Export하는 파이프라인을 구축하세요.
{% endhint %}

---

## 참고 링크

- [Databricks: Unity Catalog Privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/)
- [Databricks: Row Filters and Column Masks](https://docs.databricks.com/aws/en/tables/row-and-column-filters)
- [Databricks: Delta Sharing](https://docs.databricks.com/aws/en/delta-sharing/)
- [Databricks: Audit Logs](https://docs.databricks.com/aws/en/admin/account-settings/audit-logs)
- [Databricks: Security Policies](https://docs.databricks.com/aws/en/data-governance/unity-catalog/security-policies)
- [Databricks: Tags](https://docs.databricks.com/aws/en/data-governance/unity-catalog/tags)

---
