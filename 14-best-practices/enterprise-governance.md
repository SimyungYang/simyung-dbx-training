# 엔터프라이즈 거버넌스 모범 사례

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

## 2. 접근 제어 설계

### 2.1 그룹 기반 권한 관리

**핵심 원칙**: 개별 사용자에게 직접 권한을 부여하지 않습니다. 반드시 **그룹**을 통해 관리합니다.

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

> 💡 **중요**: `USE CATALOG`과 `USE SCHEMA`는 해당 객체를 **탐색**할 수 있는 권한일 뿐, 데이터를 읽을 수 있는 `SELECT` 권한을 포함하지 않습니다. 반드시 별도로 부여해야 합니다.

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

> ⚠️ **핵심 원칙**: 프로덕션 워크로드는 반드시 **Service Principal**으로 실행합니다. 개인 계정으로 실행하면 퇴사/이동 시 파이프라인이 중단될 수 있습니다.

---

## 3. 데이터 분류 및 태깅

### 3.1 데이터 분류 체계

| 분류 등급 | 설명 | 예시 데이터 | 접근 제어 |
|----------|------|-----------|----------|
| **Public** | 공개 가능 정보 | 제품 카탈로그, 공시 데이터 | 전체 열람 가능 |
| **Internal** | 사내 한정 정보 | 매출 집계, 내부 리포트 | 임직원 전체 |
| **Confidential** | 기밀 정보 | 고객 개인정보, 계약 조건 | 관련 팀만 |
| **Restricted** | 최고 기밀 | 주민등록번호, 의료기록, 금융정보 | 승인된 개인만 + 감사 |

### 3.2 Unity Catalog 태그를 활용한 분류

```sql
-- 테이블 수준 태그
ALTER TABLE prod_sales.gold.customers
SET TAGS ('data_classification' = 'confidential',
          'contains_pii' = 'true',
          'data_owner' = 'customer-data-team',
          'retention_days' = '730');

-- 컬럼 수준 태그
ALTER TABLE prod_sales.gold.customers
ALTER COLUMN email SET TAGS ('pii_type' = 'email', 'mask_required' = 'true');

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN phone SET TAGS ('pii_type' = 'phone', 'mask_required' = 'true');

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN ssn SET TAGS ('pii_type' = 'ssn', 'data_classification' = 'restricted');
```

### 3.3 자동 태깅 파이프라인

```python
# PII 자동 감지 및 태깅 파이프라인 예시
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 패턴 기반 PII 감지 규칙
pii_patterns = {
    "email": r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$",
    "phone_kr": r"^01[016789]-?\d{3,4}-?\d{4}$",
    "ssn_kr": r"^\d{6}-?[1-4]\d{6}$",
    "credit_card": r"^\d{4}-?\d{4}-?\d{4}-?\d{4}$"
}

def auto_tag_pii_columns(catalog, schema, table):
    """컬럼 데이터 패턴을 분석하여 PII 태그 자동 부여"""
    columns = spark.sql(f"DESCRIBE {catalog}.{schema}.{table}").collect()

    for col in columns:
        if col.data_type == "string":
            # 샘플 데이터로 패턴 매칭
            sample = spark.sql(
                f"SELECT `{col.col_name}` FROM {catalog}.{schema}.{table} "
                f"WHERE `{col.col_name}` IS NOT NULL LIMIT 100"
            ).collect()

            for pii_type, pattern in pii_patterns.items():
                match_count = sum(1 for row in sample
                                 if re.match(pattern, str(row[0])))
                if match_count > len(sample) * 0.8:  # 80% 이상 매칭
                    spark.sql(f"""
                        ALTER TABLE {catalog}.{schema}.{table}
                        ALTER COLUMN `{col.col_name}`
                        SET TAGS ('pii_type' = '{pii_type}',
                                  'mask_required' = 'true')
                    """)
```

---

## 4. 감사 및 컴플라이언스

### 4.1 감사 로그 모니터링

`system.access.audit` 시스템 테이블을 활용하여 모든 데이터 접근과 관리 작업을 추적합니다.

```sql
-- 최근 7일 데이터 접근 감사 (누가 어떤 테이블을 읽었는지)
SELECT
  event_date,
  user_identity.email AS user_email,
  action_name,
  request_params.full_name_arg AS table_name,
  source_ip_address,
  response.status_code
FROM system.access.audit
WHERE
  event_date >= current_date() - INTERVAL 7 DAYS
  AND action_name IN ('getTable', 'commandSubmit')
  AND service_name = 'unityCatalog'
ORDER BY event_date DESC;

-- 민감 테이블 접근 모니터링
SELECT
  event_date,
  user_identity.email AS user_email,
  action_name,
  request_params.full_name_arg AS table_name
FROM system.access.audit
WHERE
  event_date >= current_date() - INTERVAL 30 DAYS
  AND request_params.full_name_arg LIKE '%customers%'
  AND action_name IN ('getTable', 'commandSubmit')
ORDER BY event_date DESC;

-- 비정상 접근 패턴 감지 (야간/주말 접근)
SELECT
  user_identity.email,
  COUNT(*) AS access_count,
  MIN(event_time) AS first_access,
  MAX(event_time) AS last_access
FROM system.access.audit
WHERE
  event_date >= current_date() - INTERVAL 7 DAYS
  AND (HOUR(event_time) NOT BETWEEN 8 AND 20
       OR DAYOFWEEK(event_date) IN (1, 7))
  AND service_name = 'unityCatalog'
GROUP BY user_identity.email
HAVING COUNT(*) > 10
ORDER BY access_count DESC;

-- 권한 변경 이력 추적
SELECT
  event_date,
  user_identity.email AS changed_by,
  action_name,
  request_params
FROM system.access.audit
WHERE
  event_date >= current_date() - INTERVAL 30 DAYS
  AND action_name IN ('updatePermissions', 'grantPermission',
                       'revokePermission', 'updateSecurable')
ORDER BY event_date DESC;
```

### 4.2 규제별 체크리스트

| 규제 | 핵심 요구사항 | Databricks 구현 방법 |
|------|-------------|---------------------|
| **GDPR** | 개인정보 접근 권한, 삭제 권한, 이동 권한 | Row/Column Filter, VACUUM, Delta Sharing |
| **HIPAA** | PHI 암호화, 접근 감사, 최소 권한 | Column Mask, 감사 로그, 그룹 기반 접근 |
| **SOC 2** | 접근 제어, 변경 관리, 모니터링 | Unity Catalog, 감사 로그, Compute Policy |
| **PCI-DSS** | 카드 데이터 암호화, 접근 제한, 로깅 | Column Mask, Row Filter, 감사 로그 |
| **PIPA (한국 개인정보보호법)** | 개인정보 수집/이용 동의, 파기 | 태깅, VACUUM, 접근 감사 |

### 4.3 데이터 보존 정책

```sql
-- 보존 기간이 만료된 데이터 삭제 파이프라인
-- 1. 테이블 태그에서 보존 기간 확인
SELECT
  table_catalog, table_schema, table_name,
  tag_value AS retention_days
FROM system.information_schema.table_tags
WHERE tag_name = 'retention_days';

-- 2. 보존 기간 초과 데이터 삭제
DELETE FROM prod_sales.silver.user_events
WHERE event_date < current_date() - INTERVAL 365 DAYS;

-- 3. VACUUM으로 물리적 파일 삭제
VACUUM prod_sales.silver.user_events RETAIN 168 HOURS;
```

### 4.4 데이터 삭제 요청 처리 (Right to Erasure)

```sql
-- GDPR Article 17: 잊힐 권리 (Right to Erasure) 구현

-- 1단계: 삭제 대상 고객의 모든 데이터 위치 파악 (리니지 활용)
-- Unity Catalog 리니지를 통해 고객 데이터가 존재하는 모든 테이블 확인

-- 2단계: 각 테이블에서 해당 고객 데이터 삭제
DELETE FROM prod_sales.silver.orders
WHERE customer_id = 'CUST-12345';

DELETE FROM prod_sales.silver.interactions
WHERE customer_id = 'CUST-12345';

DELETE FROM prod_sales.gold.customer_360
WHERE customer_id = 'CUST-12345';

-- 3단계: VACUUM으로 물리적 파일에서 완전 삭제
VACUUM prod_sales.silver.orders RETAIN 0 HOURS;
-- 주의: RETAIN 0은 시간 여행 불가 → 프로덕션에서는 주의해서 사용

-- 4단계: 삭제 완료 감사 로그 기록
INSERT INTO prod_sales.audit.erasure_log
VALUES ('CUST-12345', current_timestamp(), current_user(),
        'GDPR erasure request completed', 3);  -- 3개 테이블에서 삭제
```

---

## 5. 데이터 품질 관리

### 5.1 SDP Expectations로 파이프라인 품질 보장

```python
import dlt

@dlt.table(comment="품질 검증이 적용된 실버 주문 테이블")
@dlt.expect("valid_order_id", "order_id IS NOT NULL")
@dlt.expect("valid_amount", "amount > 0 AND amount < 1000000")
@dlt.expect_or_drop("valid_customer", "customer_id IS NOT NULL")
@dlt.expect_or_fail("valid_date", "order_date <= current_date()")
def silver_orders():
    return dlt.read_stream("bronze_orders")
```

| Expectation 유형 | 위반 시 동작 | 사용 시나리오 |
|-----------------|-----------|-------------|
| `@dlt.expect` | 경고 기록, 데이터 유지 | 데이터 품질 모니터링 (소프트 체크) |
| `@dlt.expect_or_drop` | 위반 행 제거 | 잘못된 데이터 필터링 (프로덕션 권장) |
| `@dlt.expect_or_fail` | 파이프라인 중단 | 치명적 오류 (날짜 미래값 등) |

### 5.2 Lakehouse Monitoring으로 드리프트 감지

```sql
-- Lakehouse Monitor 생성 (시계열 프로파일)
CREATE OR REPLACE MONITOR prod_sales.gold.daily_revenue
USING TIME SERIES
  TIMESTAMP_COL event_date
  GRANULARITIES ("1 day", "1 week")
WITH (
  LINKED_ENTITIES = ("prod_sales.gold.daily_revenue_monitor")
);
```

| 모니터링 유형 | 감지 대상 | 적용 테이블 |
|-------------|----------|-----------|
| **Snapshot** | 시점 기준 통계 분포 | 디멘션 테이블, 룩업 테이블 |
| **Time Series** | 시간에 따른 분포 변화 | 팩트 테이블, 이벤트 테이블 |
| **Inference** | ML 모델 입력/출력 드리프트 | ML 추론 결과 테이블 |

모니터링이 생성하는 핵심 지표:

| 지표 | 설명 | 활용 |
|------|------|------|
| **Null 비율** | 컬럼별 NULL 값 비율 | 급증 시 데이터 소스 문제 의심 |
| **고유값 수** | 컬럼별 Distinct 값 수 | 급변 시 데이터 스키마 변경 의심 |
| **분포 변화** | KL Divergence, JS Distance | 데이터 드리프트 감지 |
| **이상치** | Z-score 기반 탐지 | 비정상 데이터 유입 감지 |

### 5.3 데이터 품질 SLA 정의

| SLA 항목 | 골드 등급 | 실버 등급 | 브론즈 등급 |
|---------|---------|---------|----------|
| **완전성 (Completeness)** | NULL < 0.1% | NULL < 1% | NULL < 5% |
| **정확성 (Accuracy)** | 오류율 < 0.01% | 오류율 < 0.1% | 오류율 < 1% |
| **적시성 (Timeliness)** | 15분 이내 갱신 | 1시간 이내 | 24시간 이내 |
| **일관성 (Consistency)** | 참조 무결성 100% | 참조 무결성 99.9% | 참조 무결성 99% |
| **고유성 (Uniqueness)** | 중복 0% | 중복 < 0.01% | 중복 < 0.1% |

---

## 6. CI/CD 및 자동화

### 6.1 Databricks Asset Bundles + GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy Databricks Assets

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main

      - name: Validate bundle
        run: databricks bundle validate
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_SP_TOKEN }}

  deploy-staging:
    needs: validate
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main

      - name: Deploy to staging
        run: databricks bundle deploy --target staging
        env:
          DATABRICKS_HOST: ${{ secrets.STAGING_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.STAGING_SP_TOKEN }}

  deploy-production:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main

      - name: Deploy to production
        run: databricks bundle deploy --target production
        env:
          DATABRICKS_HOST: ${{ secrets.PROD_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.PROD_SP_TOKEN }}
```

### 6.2 환경별 프로모션 (Dev → Staging → Prod)

```yaml
# databricks.yml (Databricks Asset Bundle 설정)
bundle:
  name: sales-data-pipeline

targets:
  dev:
    workspace:
      host: https://dev-workspace.cloud.databricks.com
    default: true
    variables:
      catalog: dev_sales
      warehouse_size: Small

  staging:
    workspace:
      host: https://staging-workspace.cloud.databricks.com
    variables:
      catalog: stg_sales
      warehouse_size: Medium

  production:
    workspace:
      host: https://prod-workspace.cloud.databricks.com
    run_as:
      service_principal_name: svc-prod-etl
    variables:
      catalog: prod_sales
      warehouse_size: Large
    permissions:
      - level: CAN_MANAGE
        group_name: platform-admins
      - level: CAN_VIEW
        group_name: data-engineers
```

| 환경 | 배포 트리거 | 승인 필요 | 실행 계정 |
|------|-----------|----------|----------|
| **Dev** | 로컬 `databricks bundle deploy` | 없음 | 개발자 본인 |
| **Staging** | PR 생성 시 자동 | PR 리뷰 승인 | CI/CD Service Principal |
| **Production** | main 브랜치 merge 시 | 2명 이상 승인 | 프로덕션 Service Principal |

### 6.3 코드 리뷰 프로세스

| 리뷰 항목 | 체크 포인트 | 자동화 가능 여부 |
|----------|-----------|---------------|
| **SQL 문법** | 구문 오류, 성능 안티패턴 | ✅ sqlfluff, sqlglot |
| **데이터 품질** | Expectation 정의 여부 | ✅ DLT Expectation 검사 |
| **보안** | 하드코딩된 비밀번호, PII 노출 | ✅ Secret 스캐너 |
| **비용 영향** | 대용량 스캔, 비효율 조인 | ⚠️ 부분 자동화 |
| **문서화** | 테이블/컬럼 코멘트 | ✅ 코멘트 존재 여부 검사 |
| **테스트** | 단위 테스트 커버리지 | ✅ pytest 연동 |

---

## 7. 거버넌스 성숙도 모델

조직의 데이터 거버넌스 수준을 5단계로 평가하여 개선 로드맵을 수립합니다.

### 5단계 성숙도 평가 프레임워크

| 단계 | 수준 | 접근 제어 | 데이터 품질 | 모니터링 | 자동화 |
|------|------|----------|-----------|---------|-------|
| **1. Initial** | 임의적 | 개인별 직접 권한 | 수동 검증 | 없음 | 없음 |
| **2. Managed** | 기본 | 그룹 기반 권한 | 기본 Expectation | 비용 추적 | 수동 배포 |
| **3. Defined** | 표준화 | Row/Column 필터 | SLA 정의 + 모니터링 | 감사 로그 분석 | CI/CD 구축 |
| **4. Measured** | 정량적 | ABAC 적용 | 자동 드리프트 감지 | 실시간 알림 | 완전 자동화 |
| **5. Optimizing** | 최적화 | Zero Trust | AI 기반 품질 예측 | 예측적 모니터링 | 셀프서비스 |

### 단계별 행동 계획

**1단계 → 2단계 (기초 구축, 1~2개월)**

```
□ Unity Catalog 활성화 및 Metastore 생성
□ IdP 연동 및 SCIM 자동 프로비저닝
□ 역할 기반 그룹 설계 및 생성
□ 프로덕션 워크로드를 Service Principal으로 전환
□ 기본 비용 모니터링 대시보드 구축
```

**2단계 → 3단계 (표준화, 2~3개월)**

```
□ 데이터 분류 체계 수립 및 태깅
□ Row Filter / Column Mask 적용 (PII 테이블)
□ SDP Expectations으로 파이프라인 품질 검증
□ 감사 로그 모니터링 대시보드 구축
□ Databricks Asset Bundles + CI/CD 파이프라인 구축
□ 코드 리뷰 프로세스 정립
```

**3단계 → 4단계 (자동화, 3~6개월)**

```
□ Lakehouse Monitoring으로 자동 드리프트 감지
□ 비용 이상 감지 자동 알림 설정
□ PII 자동 태깅 파이프라인 구축
□ 데이터 품질 SLA 대시보드 운영
□ Predictive Optimization 전면 활성화
□ 규제 준수 자동 보고서 생성
```

**4단계 → 5단계 (최적화, 6개월+)**

```
□ 셀프서비스 데이터 플랫폼 구축
□ 데이터 메시 패턴 적용 (도메인별 소유권)
□ AI 기반 데이터 품질 이상 예측
□ 실시간 컴플라이언스 모니터링
□ 데이터 카탈로그 + Genie 기반 자연어 접근
```

### 성숙도 자가 진단 질문

다음 질문에 답하여 현재 성숙도를 평가해 보세요.

| 질문 | 예 | 아니오 |
|------|---|-------|
| Unity Catalog이 활성화되어 있습니까? | 2단계+ | 1단계 |
| 모든 권한이 그룹을 통해 관리됩니까? | 2단계+ | 1단계 |
| PII 데이터에 Row/Column 필터가 적용되어 있습니까? | 3단계+ | 2단계 |
| 데이터 품질 SLA가 정의되어 있습니까? | 3단계+ | 2단계 |
| CI/CD로 자동 배포되고 있습니까? | 3단계+ | 2단계 |
| 데이터 드리프트가 자동 감지됩니까? | 4단계+ | 3단계 |
| 비용 이상이 자동 알림됩니까? | 4단계+ | 3단계 |
| 셀프서비스 데이터 접근이 가능합니까? | 5단계 | 4단계 |

---

## 참고 링크

- [Unity Catalog 모범 사례](https://docs.databricks.com/aws/en/data-governance/unity-catalog/best-practices)
- [인증 및 접근 제어](https://docs.databricks.com/aws/en/security/auth/index)
- [Row Filter / Column Mask](https://docs.databricks.com/aws/en/tables/row-and-column-filters)
- [감사 로그 시스템 테이블](https://docs.databricks.com/aws/en/admin/system-tables/audit)
- [Lakehouse Monitoring](https://docs.databricks.com/aws/en/lakehouse-monitoring/index)
- [Databricks Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/index)
- [네트워크 보안](https://docs.databricks.com/aws/en/security/network/index)
- [Predictive Optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
