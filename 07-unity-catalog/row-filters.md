# 행 필터 (Row Filter)

## 행 필터란?

**행 필터(Row Filter)** 는 테이블의 데이터를 조회할 때 **사용자에 따라 보이는 행을 동적으로 제한**하는 기능입니다. 원본 데이터를 변경하지 않고, SQL 함수를 통해 조회 시점에 필터가 자동 적용됩니다.

> 💡 **왜 필요한가요?** 같은 테이블이라도 서울 지역 담당자는 서울 데이터만, 부산 지역 담당자는 부산 데이터만 봐야 하는 경우가 있습니다. 뷰를 만들어 분리할 수도 있지만, 담당자가 늘어날 때마다 뷰를 추가해야 하므로 관리가 어렵습니다. 행 필터를 사용하면 **하나의 테이블에 하나의 정책**으로 해결할 수 있습니다.

> 이 기능의 기본 개념은 [권한 관리](./access-control.md)의 "행 수준 보안" 섹션에서 소개했습니다. 이 문서에서는 다양한 시나리오와 고급 패턴을 상세히 다룹니다.

---

## 동작 원리

행 필터는 **SQL 함수**로 구현됩니다. 이 함수는 각 행에 대해 `TRUE` 또는 `FALSE`를 반환하며, `TRUE`인 행만 사용자에게 보입니다.

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
