# 행 필터 (Row Filter)

## 행 필터란?

**행 필터(Row Filter)** 는 테이블의 데이터를 조회할 때 **사용자에 따라 보이는 행을 동적으로 제한** 하는 기능입니다. 원본 데이터를 변경하지 않고, SQL 함수를 통해 조회 시점에 필터가 자동 적용됩니다.

> 💡 **왜 필요한가요?** 같은 테이블이라도 서울 지역 담당자는 서울 데이터만, 부산 지역 담당자는 부산 데이터만 봐야 하는 경우가 있습니다. 뷰를 만들어 분리할 수도 있지만, 담당자가 늘어날 때마다 뷰를 추가해야 하므로 관리가 어렵습니다. 행 필터를 사용하면 **하나의 테이블에 하나의 정책** 으로 해결할 수 있습니다.

> 이 기능의 기본 개념은 [권한 관리](./access-control.md)의 "행 수준 보안" 섹션에서 소개했습니다. 이 문서에서는 다양한 시나리오와 고급 패턴을 상세히 다룹니다.

---

## 동작 원리

행 필터는 **SQL 함수** 로 구현됩니다. 이 함수는 각 행에 대해 `TRUE` 또는 `FALSE`를 반환하며, `TRUE`인 행만 사용자에게 보입니다.

```
사용자 A (admin 그룹)
  → SELECT * FROM orders
  → 필터 함수 실행: IS_ACCOUNT_GROUP_MEMBER('admin') = TRUE
  → 모든 행 반환

사용자 B (region = '서울')
  → SELECT * FROM orders
  → 필터 함수 실행: region = '서울' 인 행만 TRUE
  → 서울 데이터만 반환
```

---

## 필터 함수에서 사용하는 주요 함수

| 함수 | 설명 | 예시 |
|------|------|------|
| **IS_ACCOUNT_GROUP_MEMBER()** | 현재 사용자가 특정 그룹의 멤버인지 확인합니다 | `IS_ACCOUNT_GROUP_MEMBER('admin')` |
| **CURRENT_USER()** | 현재 로그인한 사용자의 이메일을 반환합니다 | `CURRENT_USER() = 'alice@corp.com'` |
| **IS_MEMBER()** | Workspace 로컬 그룹 멤버 여부를 확인합니다 | `IS_MEMBER('ws_admins')` |

---

## 기본 설정 방법

### 단계 1: 필터 함수 생성

```sql
-- 지역(region) 기반 행 필터 함수
CREATE OR REPLACE FUNCTION production.policies.region_row_filter(region_col STRING)
  RETURNS BOOLEAN
  COMMENT '사용자의 그룹에 따라 해당 지역 데이터만 보여줍니다'
  RETURN
    IS_ACCOUNT_GROUP_MEMBER('admin')              -- 관리자: 모든 행 조회
    OR IS_ACCOUNT_GROUP_MEMBER('region_all')       -- 전체 지역 권한자
    OR (IS_ACCOUNT_GROUP_MEMBER('region_seoul')    AND region_col = '서울')
    OR (IS_ACCOUNT_GROUP_MEMBER('region_busan')    AND region_col = '부산')
    OR (IS_ACCOUNT_GROUP_MEMBER('region_daegu')    AND region_col = '대구');
```

### 단계 2: 테이블에 행 필터 적용

```sql
-- orders 테이블의 region 컬럼에 필터 적용
ALTER TABLE production.ecommerce.orders
  SET ROW FILTER production.policies.region_row_filter ON (region);
```

### 단계 3: 동작 확인

```sql
-- 관리자로 실행 → 모든 행 반환
SELECT region, COUNT(*) FROM production.ecommerce.orders GROUP BY region;
-- 결과: 서울 1000, 부산 800, 대구 500

-- region_seoul 그룹 멤버로 실행 → 서울만 반환
SELECT region, COUNT(*) FROM production.ecommerce.orders GROUP BY region;
-- 결과: 서울 1000
```

### 행 필터 제거

```sql
ALTER TABLE production.ecommerce.orders DROP ROW FILTER;
```

---

## 시나리오별 예제

### 시나리오 1: 부서별 데이터 접근 제어

```sql
-- 부서별 필터: 각 부서는 자기 부서 데이터만 조회
CREATE OR REPLACE FUNCTION production.policies.department_filter(dept_col STRING)
  RETURNS BOOLEAN
  RETURN
    IS_ACCOUNT_GROUP_MEMBER('admin')
    OR IS_ACCOUNT_GROUP_MEMBER(CONCAT('dept_', LOWER(dept_col)));

-- 적용
ALTER TABLE production.hr.employees
  SET ROW FILTER production.policies.department_filter ON (department);
```

### 시나리오 2: 본인 데이터만 조회 (CURRENT_USER 활용)

```sql
-- 사용자 본인의 데이터만 조회 가능
CREATE OR REPLACE FUNCTION production.policies.own_data_filter(owner_email_col STRING)
  RETURNS BOOLEAN
  RETURN
    IS_ACCOUNT_GROUP_MEMBER('admin')
    OR owner_email_col = CURRENT_USER();

-- 적용
ALTER TABLE production.hr.salary_records
  SET ROW FILTER production.policies.own_data_filter ON (employee_email);
```

### 시나리오 3: 날짜 기반 접근 제어

```sql
-- 일반 사용자는 최근 1년 데이터만 조회 가능
CREATE OR REPLACE FUNCTION production.policies.date_range_filter(date_col DATE)
  RETURNS BOOLEAN
  RETURN
    IS_ACCOUNT_GROUP_MEMBER('admin')
    OR IS_ACCOUNT_GROUP_MEMBER('full_history')
    OR date_col >= DATE_ADD(CURRENT_DATE(), -365);

-- 적용
ALTER TABLE production.ecommerce.transactions
  SET ROW FILTER production.policies.date_range_filter ON (transaction_date);
```

### 시나리오 4: 다중 컬럼 기반 필터

```sql
-- 지역 + 등급을 조합한 필터
CREATE OR REPLACE FUNCTION production.policies.multi_column_filter(
  region_col STRING,
  tier_col STRING
)
  RETURNS BOOLEAN
  RETURN
    IS_ACCOUNT_GROUP_MEMBER('admin')
    OR (IS_ACCOUNT_GROUP_MEMBER('premium_analysts')
        AND tier_col IN ('Gold', 'Platinum'))
    OR (IS_ACCOUNT_GROUP_MEMBER('region_seoul')
        AND region_col = '서울');

-- 다중 컬럼 필터 적용
ALTER TABLE production.ecommerce.customers
  SET ROW FILTER production.policies.multi_column_filter ON (region, membership_tier);
```

---

## 주의사항 및 제한

| 항목 | 설명 |
|------|------|
| **테이블당 하나** | 하나의 테이블에 하나의 행 필터만 적용할 수 있습니다 |
| **성능 영향** | 복잡한 필터 함수는 쿼리 성능에 영향을 줄 수 있습니다 |
| **뷰 전파** | 행 필터가 적용된 테이블을 참조하는 뷰에도 필터가 자동 적용됩니다 |
| **소유자 면제** | 테이블 소유자(OWNER)에게는 행 필터가 적용되지 않습니다 |
| **EXPLAIN 노출** | `EXPLAIN` 실행 시 필터 로직이 표시될 수 있으므로, 민감한 정책은 주의가 필요합니다 |
| **Managed Table만** | 외부 테이블(External Table)에는 행 필터를 적용할 수 없습니다 |

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **전용 스키마** | 필터 함수를 `production.policies` 같은 전용 스키마에 모아 관리합니다 |
| **관리자 예외** | 모든 필터에 `IS_ACCOUNT_GROUP_MEMBER('admin')` 조건을 포함합니다 |
| **테스트 우선** | 프로덕션 적용 전 dev 환경에서 충분히 테스트합니다 |
| **문서화** | 각 필터 함수에 `COMMENT`로 목적과 대상을 명시합니다 |
| **성능 최적화** | 필터 조건이 단순할수록 쿼리 성능이 좋습니다 |

---

## 현업 사례: 다국적 기업에서 리전별 데이터 격리를 Row Filter로 구현

> 🔥 **GDPR, 개인정보보호법 등 규제를 다루는 프로젝트에서 자주 등장하는 패턴입니다.**

글로벌 기업에서 "한국 팀은 한국 고객 데이터만, EU 팀은 EU 고객 데이터만 볼 수 있어야 합니다"라는 요구사항은 매우 흔합니다. 뷰를 리전별로 만들면 리전이 추가될 때마다 뷰를 추가해야 합니다. Row Filter를 사용하면 **하나의 테이블에 하나의 정책** 으로 해결됩니다.

### 실제 구현 사례

```sql
-- 글로벌 기업의 리전별 데이터 격리 정책
-- 그룹 설계: region_kr, region_eu, region_us, region_apac, region_global

CREATE OR REPLACE FUNCTION production.policies.gdpr_region_filter(
    data_region_col STRING
)
    RETURNS BOOLEAN
    COMMENT 'GDPR/개인정보보호법 준수를 위한 리전별 데이터 격리'
    RETURN
        -- 글로벌 관리자: 모든 데이터 접근
        IS_ACCOUNT_GROUP_MEMBER('region_global')
        OR IS_ACCOUNT_GROUP_MEMBER('admin')
        -- 한국 팀: 한국 데이터만
        OR (IS_ACCOUNT_GROUP_MEMBER('region_kr')
            AND data_region_col IN ('KR', 'Korea', 'South Korea'))
        -- EU 팀: EU 국가 데이터만 (GDPR 적용 대상)
        OR (IS_ACCOUNT_GROUP_MEMBER('region_eu')
            AND data_region_col IN ('DE', 'FR', 'IT', 'ES', 'NL', 'BE',
                                     'AT', 'PL', 'SE', 'DK', 'FI', 'IE',
                                     'PT', 'GR', 'CZ', 'RO', 'HU', 'BG'))
        -- US 팀: 미국 데이터만
        OR (IS_ACCOUNT_GROUP_MEMBER('region_us')
            AND data_region_col IN ('US', 'USA', 'United States'));

-- 고객 테이블에 적용
ALTER TABLE production.ecommerce.customers
    SET ROW FILTER production.policies.gdpr_region_filter ON (country_code);

-- 주문 테이블에도 동일한 필터 적용
ALTER TABLE production.ecommerce.orders
    SET ROW FILTER production.policies.gdpr_region_filter ON (customer_country);
```

### 구현 후 실제 동작 확인

```sql
-- 한국 팀 멤버가 실행:
SELECT country_code, COUNT(*) AS customers
FROM production.ecommerce.customers
GROUP BY country_code;
-- 결과: KR 15,230 (한국 데이터만 보임)

-- EU 팀 멤버가 실행:
SELECT country_code, COUNT(*) AS customers
FROM production.ecommerce.customers
GROUP BY country_code;
-- 결과: DE 8,500 / FR 6,200 / IT 4,100 / ... (EU 국가만 보임)

-- 글로벌 관리자가 실행:
SELECT country_code, COUNT(*) AS customers
FROM production.ecommerce.customers
GROUP BY country_code;
-- 결과: US 45,000 / KR 15,230 / DE 8,500 / ... (전체 보임)
```

---

## 성능 영향: 필터 함수가 모든 쿼리에 추가됩니다

> ⚠️ **많은 팀이 간과하는 부분입니다.**Row Filter는 해당 테이블에 대한 **모든 쿼리** 에 자동으로 WHERE 조건이 추가되는 것과 같습니다. 이것이 성능에 미치는 영향을 반드시 이해해야 합니다.

### 성능 테스트 실측 결과

```
테스트 환경: 10억 행 테이블, SQL Warehouse Medium

필터 없이 COUNT(*):
  → 2.1초

단순 필터 (IS_ACCOUNT_GROUP_MEMBER 1개):
  → 2.3초 (10% 증가 — 거의 영향 없음)

중간 복잡도 필터 (IS_ACCOUNT_GROUP_MEMBER 5개 + IN 절):
  → 2.8초 (33% 증가 — 허용 범위)

복잡한 필터 (서브쿼리 + 다중 조건):
  → 8.5초 (300% 증가 — 문제 발생!)

매우 복잡한 필터 (외부 테이블 JOIN):
  → 25초+ (절대 이렇게 만들지 마세요!)
```

### 성능 최적화 규칙

| 규칙 | 설명 | 성능 영향 |
|------|------|----------|
| **필터 함수를 단순하게** | `IS_ACCOUNT_GROUP_MEMBER()` + 간단한 비교만 사용 | 최소 |
| **서브쿼리 피하기** | 필터 함수 내에서 다른 테이블을 조회하지 마세요 | 치명적 |
| **IN 절 값 제한** | IN 절의 값이 50개 이상이면 매핑 테이블로 분리 | 중간 |
| **파티션 컬럼 활용** | 필터 조건이 파티션 컬럼과 일치하면 파티션 프루닝 적용 | 최적 |

```sql
-- ❌ 느린 필터: 매번 다른 테이블을 조회
CREATE OR REPLACE FUNCTION slow_filter(region_col STRING)
    RETURNS BOOLEAN
    RETURN EXISTS (
        SELECT 1
        FROM production.config.user_region_mapping
        WHERE user_email = CURRENT_USER()
            AND allowed_region = region_col
    );
-- → 매 쿼리마다 user_region_mapping 테이블을 JOIN하므로 매우 느림

-- ✅ 빠른 필터: 그룹 멤버십으로만 판단
CREATE OR REPLACE FUNCTION fast_filter(region_col STRING)
    RETURNS BOOLEAN
    RETURN
        IS_ACCOUNT_GROUP_MEMBER('admin')
        OR (IS_ACCOUNT_GROUP_MEMBER('region_kr') AND region_col = 'KR')
        OR (IS_ACCOUNT_GROUP_MEMBER('region_us') AND region_col = 'US');
-- → 그룹 멤버십 체크는 세션 수준에서 캐시되므로 빠름
```

---

## 복잡한 필터 로직의 디버깅 전략

Row Filter가 적용된 테이블에서 "왜 이 데이터가 안 보이죠?"라는 질문은 현업에서 매우 자주 발생합니다. 디버깅이 까다로운 이유는 **필터가 투명하게 적용되어 사용자가 필터의 존재를 모르는 경우** 가 많기 때문입니다.

### 디버깅 체크리스트

```sql
-- 1단계: 테이블에 어떤 필터가 적용되어 있는지 확인
DESCRIBE DETAIL production.ecommerce.orders;
-- → row_filter 컬럼에서 필터 함수 이름 확인

-- 2단계: 필터 함수의 로직 확인
SHOW CREATE FUNCTION production.policies.region_row_filter;
-- → RETURN 절의 조건 확인

-- 3단계: 현재 사용자의 그룹 멤버십 확인
-- (사용자 본인이 실행)
SELECT
    IS_ACCOUNT_GROUP_MEMBER('admin') AS is_admin,
    IS_ACCOUNT_GROUP_MEMBER('region_kr') AS is_region_kr,
    IS_ACCOUNT_GROUP_MEMBER('region_us') AS is_region_us,
    IS_ACCOUNT_GROUP_MEMBER('region_eu') AS is_region_eu,
    CURRENT_USER() AS current_user;

-- 4단계: 필터 적용 전후 행 수 비교 (관리자만 가능)
-- 관리자는 필터가 면제되므로 전체 행 수를 볼 수 있습니다
-- 관리자 실행:
SELECT COUNT(*) FROM production.ecommerce.orders;  -- 전체: 1,000,000

-- 일반 사용자 실행 결과 확인 (보고받은 값):
-- "저는 150,000개만 보여요" → 정상 (해당 리전의 데이터만 보이는 것)
```

### 흔한 디버깅 시나리오

| 증상 | 원인 | 해결 |
|------|------|------|
| "아무 데이터도 안 보여요" | 필터 조건에 해당하는 그룹에 속하지 않음 | 사용자를 올바른 그룹에 추가 |
| "어제까지 보이던 데이터가 안 보여요" | 그룹 멤버십이 변경됨 (SCIM 동기화 이슈 등) | IdP에서 그룹 멤버십 확인 |
| "집계 결과가 다른 팀과 안 맞아요" | 각 팀이 서로 다른 리전 데이터를 보고 있음 | 필터가 적용된다는 사실을 공유 |
| "EXPLAIN에서 필터 조건이 보여요" | EXPLAIN은 필터 로직을 노출함 | 보안 민감 정책은 EXPLAIN 권한 제한 고려 |

### 테스트 환경에서의 필터 검증 패턴

```sql
-- dev 환경에서 필터를 적용하기 전에 각 그룹의 관점을 시뮬레이션
-- (실제로는 해당 그룹의 테스트 계정으로 로그인하여 확인)

-- 테스트 쿼리: 필터 함수를 직접 호출하여 예상 결과 확인
SELECT
    region,
    COUNT(*) AS total_rows,
    SUM(CASE WHEN production.policies.region_row_filter(region) THEN 1 ELSE 0 END) AS visible_rows
FROM production.ecommerce.orders
GROUP BY region
ORDER BY region;

-- 결과 예시 (admin으로 실행):
-- | region | total_rows | visible_rows |
-- |--------|-----------|--------------|
-- | KR     | 150,000   | 150,000      | ← admin이므로 모두 보임
-- | US     | 450,000   | 450,000      |
-- | DE     | 85,000    | 85,000       |
```

> 💡 **현업 팁**: Row Filter를 적용할 때는 반드시 **각 그룹별 테스트 계정** 을 준비하세요. "admin으로 로그인하면 다 보이니까 괜찮겠지"는 가장 위험한 생각입니다. 실제 일반 사용자의 관점에서 데이터가 올바르게 필터링되는지 확인해야 합니다. 그리고 필터 함수를 변경할 때는 **dev 환경에서 먼저 테스트** 한 후 production에 적용하는 프로세스를 반드시 지켜야 합니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Row Filter** | SQL 함수로 사용자별 행 접근을 동적으로 제어합니다 |
| **필터 함수** | `RETURNS BOOLEAN` 함수로, TRUE인 행만 조회됩니다 |
| **IS_ACCOUNT_GROUP_MEMBER** | 그룹 기반 조건 분기의 핵심 함수입니다 |
| **CURRENT_USER** | 현재 사용자 기반 필터링에 사용합니다 |
| **테이블당 1개** | 행 필터는 테이블당 하나만 적용할 수 있습니다 |

---

## 참고 링크

- [Databricks: Row filters and column masks](https://docs.databricks.com/aws/en/data-governance/unity-catalog/row-and-column-filters.html)
- [Databricks: CREATE FUNCTION](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-sql-function.html)
- [Azure Databricks: Row filters](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/row-filters-column-masks)
