# 데이터 분류 및 태깅

## 3. 데이터 분류 및 태깅

### 3.1 데이터 분류 체계

| 분류 등급 | 설명 | 예시 데이터 | 접근 제어 |
|----------|------|-----------|----------|
| **Public**| 공개 가능 정보 | 제품 카탈로그, 공시 데이터 | 전체 열람 가능 |
| **Internal**| 사내 한정 정보 | 매출 집계, 내부 리포트 | 임직원 전체 |
| **Confidential**| 기밀 정보 | 고객 개인정보, 계약 조건 | 관련 팀만 |
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
| **GDPR**| 개인정보 접근 권한, 삭제 권한, 이동 권한 | Row/Column Filter, VACUUM, Delta Sharing |
| **HIPAA**| PHI 암호화, 접근 감사, 최소 권한 | Column Mask, 감사 로그, 그룹 기반 접근 |
| **SOC 2**| 접근 제어, 변경 관리, 모니터링 | Unity Catalog, 감사 로그, Compute Policy |
| **PCI-DSS**| 카드 데이터 암호화, 접근 제한, 로깅 | Column Mask, Row Filter, 감사 로그 |
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
