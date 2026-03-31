# 컬럼 마스킹 (Column Masking)

## 컬럼 마스킹이란?

**컬럼 마스킹(Column Masking)**은 민감한 컬럼의 값을 사용자 권한에 따라 **동적으로 가림 처리**하는 기능입니다. 원본 데이터를 변경하지 않고, 조회 시점에 SQL 함수가 실행되어 마스킹된 값이 반환됩니다.

> 💡 **왜 필요한가요?** 고객 정보 테이블에는 이메일, 전화번호, 주민번호 같은 개인정보가 포함되어 있습니다. 분석가는 고객 구매 패턴을 분석해야 하지만, 개인 식별 정보는 볼 필요가 없습니다. 컬럼 마스킹을 사용하면 역할에 따라 적절한 수준의 데이터만 노출할 수 있습니다.

> 기본 개념은 [권한 관리](./access-control.md)의 "열 수준 보안" 섹션에서 소개했습니다. 이 문서에서는 다양한 마스킹 패턴과 고급 활용법을 상세히 다룹니다.

---

## 동작 원리

컬럼 마스킹은 **SQL 함수**로 구현됩니다. 함수는 원본 값을 입력받아, 사용자 권한에 따라 원본 또는 마스킹된 값을 반환합니다.

```
관리자 (admin 그룹):
  SELECT email FROM customers → alice@company.com

분석가 (analysts 그룹):
  SELECT email FROM customers → al***@company.com

일반 사용자:
  SELECT email FROM customers → ***@***.***
```

---

## 다단계 마스킹 패턴

### 관리자 / 분석가 / 일반 사용자 3단계

대부분의 기업에서는 다음 3단계 마스킹을 적용합니다.

| 수준 | 대상 | 이메일 예시 | 전화번호 예시 |
|------|------|-----------|-------------|
| **원본** | 관리자, 데이터 소유자 | alice@company.com | 010-1234-5678 |
| **부분 마스킹** | 분석가, 운영자 | al***@company.com | 010-****-5678 |
| **완전 마스킹** | 일반 사용자 | ***@***.*** | ***-****-**** |

---

## 마스킹 함수 작성

### 이메일 마스킹

```sql
CREATE OR REPLACE FUNCTION production.policies.mask_email(email_col STRING)
  RETURNS STRING
  COMMENT '이메일 주소를 역할에 따라 마스킹합니다'
  RETURN
    CASE
      WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN email_col
      WHEN IS_ACCOUNT_GROUP_MEMBER('analysts') THEN
        CONCAT(LEFT(email_col, 2), '***@', SPLIT(email_col, '@')[1])
      ELSE '***@***.***'
    END;
```

### 전화번호 마스킹

```sql
CREATE OR REPLACE FUNCTION production.policies.mask_phone(phone_col STRING)
  RETURNS STRING
  COMMENT '전화번호를 역할에 따라 마스킹합니다'
  RETURN
    CASE
      WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN phone_col
      WHEN IS_ACCOUNT_GROUP_MEMBER('analysts') THEN
        CONCAT(LEFT(phone_col, 3), '-****-', RIGHT(phone_col, 4))
      ELSE '***-****-****'
    END;
```

### 주민번호 마스킹

```sql
CREATE OR REPLACE FUNCTION production.policies.mask_ssn(ssn_col STRING)
  RETURNS STRING
  COMMENT '주민등록번호를 역할에 따라 마스킹합니다'
  RETURN
    CASE
      WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN ssn_col
      WHEN IS_ACCOUNT_GROUP_MEMBER('analysts') THEN
        CONCAT(LEFT(ssn_col, 8), '******')  -- 생년월일만 노출
      ELSE '******-*******'
    END;
```

### 카드번호 마스킹

```sql
CREATE OR REPLACE FUNCTION production.policies.mask_card_number(card_col STRING)
  RETURNS STRING
  COMMENT '카드번호를 마스킹합니다 (마지막 4자리만 노출)'
  RETURN
    CASE
      WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN card_col
      ELSE CONCAT('****-****-****-', RIGHT(REGEXP_REPLACE(card_col, '[^0-9]', ''), 4))
    END;
```

---

## 테이블에 마스킹 적용

### 마스킹 함수 연결

```sql
-- 이메일 컬럼에 마스킹 적용
ALTER TABLE production.ecommerce.customers
  ALTER COLUMN email SET MASK production.policies.mask_email;

-- 전화번호 컬럼에 마스킹 적용
ALTER TABLE production.ecommerce.customers
  ALTER COLUMN phone SET MASK production.policies.mask_phone;

-- 주민번호 컬럼에 마스킹 적용
ALTER TABLE production.ecommerce.customers
  ALTER COLUMN ssn SET MASK production.policies.mask_ssn;
```

### 마스킹 해제

```sql
-- 특정 컬럼의 마스킹 제거
ALTER TABLE production.ecommerce.customers
  ALTER COLUMN email DROP MASK;
```

---

## 고급 패턴

### 패턴 1: CURRENT_USER 기반 마스킹

```sql
-- 본인 데이터는 원본, 타인 데이터는 마스킹
CREATE OR REPLACE FUNCTION production.policies.mask_salary(
  salary_col DECIMAL(10,2),
  owner_email STRING
)
  RETURNS DECIMAL(10,2)
  RETURN
    CASE
      WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN salary_col
      WHEN CURRENT_USER() = owner_email THEN salary_col  -- 본인 급여는 원본
      ELSE NULL  -- 타인 급여는 NULL
    END;

-- 적용 (마스킹 함수에 다른 컬럼도 전달 가능)
ALTER TABLE production.hr.employees
  ALTER COLUMN salary SET MASK production.policies.mask_salary
  USING COLUMNS (email);
```

> 💡 `USING COLUMNS` 절을 사용하면 마스킹 함수에 **같은 행의 다른 컬럼 값**을 추가 인자로 전달할 수 있습니다.

### 패턴 2: 해싱(Hashing) 마스킹

```sql
-- 원본 대신 해시값을 반환 (분석용 조인키로 활용 가능)
CREATE OR REPLACE FUNCTION production.policies.hash_email(email_col STRING)
  RETURNS STRING
  RETURN
    CASE
      WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN email_col
      ELSE SHA2(email_col, 256)  -- 해시값으로 대체
    END;
```

해싱 마스킹은 원본을 볼 수 없지만, **동일 값은 동일 해시를 생성**하므로 분석가가 고유 사용자 수 파악이나 테이블 간 조인에 활용할 수 있습니다.

### 패턴 3: 범위형 마스킹 (숫자)

```sql
-- 정확한 나이 대신 연령대로 표시
CREATE OR REPLACE FUNCTION production.policies.mask_age(age_col INT)
  RETURNS STRING
  RETURN
    CASE
      WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN CAST(age_col AS STRING)
      ELSE CONCAT(CAST(FLOOR(age_col / 10) * 10 AS STRING), '대')
    END;
```

---

## 마스킹 확인 및 모니터링

```sql
-- 테이블에 적용된 마스킹 함수 확인
DESCRIBE TABLE EXTENDED production.ecommerce.customers;

-- 마스킹이 올바르게 동작하는지 테스트
-- (관리자 계정으로 실행)
SELECT email, phone FROM production.ecommerce.customers LIMIT 5;

-- 특정 사용자가 보는 결과를 시뮬레이션하려면
-- 해당 사용자 계정으로 직접 쿼리를 실행해야 합니다
```

---

## 주의사항 및 제한

| 항목 | 설명 |
|------|------|
| **컬럼당 하나** | 하나의 컬럼에 하나의 마스킹 함수만 적용할 수 있습니다 |
| **반환 타입 일치** | 마스킹 함수의 반환 타입이 컬럼의 데이터 타입과 일치해야 합니다 |
| **소유자 면제** | 테이블 소유자(OWNER)에게는 마스킹이 적용되지 않습니다 |
| **성능** | 마스킹 함수는 행 단위로 실행되므로, 대량 데이터에서는 성능에 주의합니다 |
| **WHERE 절 영향** | 마스킹된 컬럼으로 WHERE 절 필터링 시, 마스킹된 값으로 비교됩니다 |
| **Managed Table만** | 외부 테이블에는 컬럼 마스킹을 적용할 수 없습니다 |

---

## 행 필터 + 컬럼 마스킹 조합

행 필터와 컬럼 마스킹을 **함께 적용**하면 더욱 정밀한 보안을 구현할 수 있습니다.

```sql
-- 행 필터: 지역별 데이터 제한
ALTER TABLE production.ecommerce.customers
  SET ROW FILTER production.policies.region_row_filter ON (region);

-- 컬럼 마스킹: 이메일, 전화번호 마스킹
ALTER TABLE production.ecommerce.customers
  ALTER COLUMN email SET MASK production.policies.mask_email;

ALTER TABLE production.ecommerce.customers
  ALTER COLUMN phone SET MASK production.policies.mask_phone;

-- 결과: 서울 담당 분석가는
-- 1) 서울 지역 고객만 보이고 (행 필터)
-- 2) 이메일/전화번호는 부분 마스킹됩니다 (컬럼 마스킹)
```

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **정책 스키마 분리** | 마스킹 함수를 `production.policies` 스키마에 모아 관리합니다 |
| **일관된 마스킹 수준** | 조직 전체에서 동일한 PII 유형에 동일한 마스킹 패턴을 적용합니다 |
| **COMMENT 필수** | 각 함수에 목적, 대상 그룹, 마스킹 수준을 명시합니다 |
| **태그 연동** | PII 태그가 있는 컬럼에 자동 마스킹을 적용합니다 ([태그](./tags.md) 참조) |
| **정기 검증** | 새 그룹이 추가되면 마스킹 함수도 업데이트합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Column Mask** | SQL 함수로 컬럼 값을 동적으로 마스킹합니다 |
| **다단계 마스킹** | 관리자/분석가/일반 사용자에게 다른 수준의 마스킹을 적용합니다 |
| **USING COLUMNS** | 같은 행의 다른 컬럼을 마스킹 함수의 추가 인자로 전달합니다 |
| **해싱 마스킹** | 원본 대신 해시값을 반환하여 분석 가능성을 유지합니다 |
| **행 필터 + 마스킹** | 두 기능을 조합하여 정밀한 보안을 구현합니다 |

---

## 참고 링크

- [Databricks: Row filters and column masks](https://docs.databricks.com/aws/en/data-governance/unity-catalog/row-and-column-filters.html)
- [Databricks: ALTER TABLE column masking](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-table.html)
- [Azure Databricks: Column masks](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/row-filters-column-masks)
