# PIVOT / UNPIVOT 완전 가이드

## 1. 왜 PIVOT / UNPIVOT이 필요한가

데이터는 저장 목적과 분석 목적이 다를 때 형태를 바꿔야 합니다. 트랜잭션 데이터는 보통 **긴 형식(long format)** — 즉, 한 행에 하나의 관측값 — 으로 저장됩니다. 하지만 보고서나 대시보드는 **넓은 형식(wide format)** — 카테고리별 컬럼이 나란히 — 을 요구합니다.

| 상황 | 필요한 변환 |
|------|------------|
| 월별 제품 매출을 컬럼으로 나열한 리포트 | PIVOT (행 → 열) |
| BI 툴에 크로스탭(Cross-tab) 피드 | PIVOT |
| 넓은 테이블을 ML 피처 테이블로 정규화 | UNPIVOT (열 → 행) |
| 레거시 Wide 테이블을 ELT 파이프라인에 적재 | UNPIVOT |

**ETL/ELT 파이프라인** 에서도 PIVOT/UNPIVOT은 필수입니다. 외부 시스템(ERP, 스프레드시트)에서 넓은 형식으로 넘어온 데이터를 정규화된 형식으로 변환하거나, 반대로 다운스트림 소비자를 위해 요약 테이블을 넓게 펼쳐야 하는 경우가 자주 발생합니다.

> **비유:** PIVOT은 세로로 나열된 성적표를 과목별 컬럼이 있는 표로 변환하는 것이고, UNPIVOT은 그 반대입니다.

---

## 2. PIVOT 기본 문법

### 구문 구조

```sql
SELECT *
FROM 소스_테이블
PIVOT (
    집계함수(값_컬럼)
    FOR 피벗_컬럼 IN (값1 AS 별칭1, 값2 AS 별칭2, ...)
)
```

**핵심 구성 요소:**
- `집계함수(값_컬럼)` — SUM, COUNT, AVG, MAX, MIN 등 모두 사용 가능
- `FOR 피벗_컬럼` — 어느 컬럼의 값을 새 컬럼 이름으로 만들지 지정
- `IN (...)` — 새 컬럼으로 생성할 값 목록 (정적)

### 예제: 월별 제품 매출 PIVOT

```sql
-- 원본 데이터 (long format)
-- | order_date | product_name | amount    |
-- |------------|--------------|-----------|
-- | 2025-01-05 | Laptop       | 3,300,000 |
-- | 2025-01-12 | Mouse        |    25,000 |
-- | 2025-02-03 | Laptop       | 1,500,000 |
-- | 2025-02-18 | Mouse        |    50,000 |

SELECT *
FROM (
    SELECT
        DATE_FORMAT(order_date, 'yyyy-MM') AS month,
        product_name,
        amount
    FROM orders
)
PIVOT (
    SUM(amount)
    FOR product_name IN (
        'Laptop'   AS laptop,
        'Mouse'    AS mouse,
        'Keyboard' AS keyboard,
        'Monitor'  AS monitor
    )
)
ORDER BY month;
```

결과 (wide format):

| month   | laptop    | mouse  | keyboard | monitor |
|---------|-----------|--------|----------|---------|
| 2025-01 | 3,300,000 | 25,000 | 89,000   | NULL    |
| 2025-02 | 1,500,000 | 50,000 | 178,000  | 550,000 |

### 여러 집계 함수 동시 사용

```sql
-- 매출 합계(total)와 주문 건수(cnt)를 동시에 피벗
SELECT *
FROM (
    SELECT region, product_name, amount
    FROM orders
)
PIVOT (
    SUM(amount) AS total,
    COUNT(*)    AS cnt
    FOR product_name IN ('Laptop' AS laptop, 'Mouse' AS mouse)
);
-- 결과 컬럼: region, laptop_total, laptop_cnt, mouse_total, mouse_cnt
```

---

## 3. UNPIVOT 기본 문법

**UNPIVOT** 은 넓은 테이블(wide format)을 긴 테이블(long format)로 변환합니다.

### 구문 구조

```sql
SELECT *
FROM 소스_테이블
UNPIVOT (
    값_컬럼 FOR 카테고리_컬럼 IN (컬럼1, 컬럼2, ...)
)
```

### 예제: 분기별 매출 UNPIVOT

```sql
-- 원본 데이터 (wide format)
-- | year | q1        | q2        | q3        | q4        |
-- |------|-----------|-----------|-----------|-----------|
-- | 2024 | 1,200,000 | 1,500,000 | 1,300,000 | 1,800,000 |
-- | 2025 | 1,400,000 | 1,600,000 | NULL      | NULL      |

SELECT *
FROM quarterly_revenue
UNPIVOT (
    revenue FOR quarter IN (q1, q2, q3, q4)
)
ORDER BY year, quarter;
```

결과:

| year | quarter | revenue   |
|------|---------|-----------|
| 2024 | q1      | 1,200,000 |
| 2024 | q2      | 1,500,000 |
| 2024 | q3      | 1,300,000 |
| 2024 | q4      | 1,800,000 |
| 2025 | q1      | 1,400,000 |
| 2025 | q2      | 1,600,000 |

> **UNPIVOT은 기본적으로 NULL을 자동 제외합니다.** 2025년 Q3, Q4는 NULL이므로 결과에 포함되지 않습니다. NULL도 포함하려면 `UNPIVOT INCLUDE NULLS`를 사용하세요.

```sql
SELECT *
FROM quarterly_revenue
UNPIVOT INCLUDE NULLS (
    revenue FOR quarter IN (q1, q2, q3, q4)
)
ORDER BY year, quarter;
```

---

## 4. 동적 PIVOT — 값을 미리 모를 때

정적 PIVOT은 IN 절에 값을 직접 나열해야 합니다. 하지만 제품 목록이 매달 바뀌거나, 컬럼 값을 미리 알 수 없을 때는 **동적 PIVOT(Dynamic PIVOT)** 이 필요합니다.

Databricks SQL에서는 **SQL Scripting** (Databricks Runtime 16.0+ / SQL Warehouse 2024.50+) 을 활용해 동적으로 IN 절을 구성할 수 있습니다.

```sql
-- SQL Scripting을 활용한 동적 PIVOT
DECLARE pivot_cols STRING;
DECLARE sql_stmt  STRING;

-- 1) 피벗할 값 목록을 동적으로 수집
SET VAR pivot_cols = (
    SELECT CONCAT_WS(', ',
        COLLECT_LIST(CONCAT("'", product_name, "' AS ", REPLACE(product_name, ' ', '_')))
    )
    FROM (SELECT DISTINCT product_name FROM orders ORDER BY product_name)
);

-- 2) 완성된 PIVOT SQL 조립 후 실행
SET VAR sql_stmt = CONCAT(
    'SELECT * FROM (SELECT DATE_FORMAT(order_date, ''yyyy-MM'') AS month,
      product_name, amount FROM orders)
     PIVOT (SUM(amount) FOR product_name IN (', pivot_cols, '))
     ORDER BY month'
);

EXECUTE IMMEDIATE sql_stmt;
```

> **주의:** 동적 PIVOT은 SQL Scripting 지원 환경(Serverless SQL Warehouse 또는 DBR 16.0+)에서만 동작합니다. 일반 클러스터에서는 Python + `spark.sql()`로 동일하게 구현하세요.

```python
# PySpark로 동적 PIVOT
from pyspark.sql import functions as F

products = [row[0] for row in spark.sql(
    "SELECT DISTINCT product_name FROM orders ORDER BY product_name"
).collect()]

df = spark.sql("""
    SELECT DATE_FORMAT(order_date, 'yyyy-MM') AS month, product_name, amount
    FROM orders
""")

df.groupBy("month").pivot("product_name", products).sum("amount").orderBy("month").display()
```

---

## 5. 실전 활용 패턴

### 5-1. 크로스탭(Cross-tab) 리포트

```sql
-- 지역 × 제품 카테고리 매출 크로스탭
SELECT *
FROM (
    SELECT region, category, revenue
    FROM sales_summary
)
PIVOT (
    SUM(revenue) AS rev,
    COUNT(*)     AS txn
    FOR category IN ('Electronics' AS elec, 'Apparel' AS aprl, 'Food' AS food)
);
```

### 5-2. 시계열 데이터 변환 (Long → Wide)

KPI 지표를 날짜별 컬럼으로 펼쳐 BI 툴에 전달하는 패턴입니다.

```sql
SELECT *
FROM (
    SELECT metric_name, report_date, metric_value
    FROM daily_kpi
    WHERE report_date BETWEEN '2025-01-01' AND '2025-01-07'
)
PIVOT (
    AVG(metric_value)
    FOR report_date IN (
        '2025-01-01' AS d0101,
        '2025-01-02' AS d0102,
        '2025-01-03' AS d0103
        -- ...
    )
);
```

### 5-3. 피처 엔지니어링 (Feature Engineering)

ML 모델 학습 전, 범주형 이벤트 로그를 사용자별 피처 벡터로 변환할 때 PIVOT이 유용합니다.

```sql
-- 사용자별 이벤트 유형 카운트 → 피처 테이블
SELECT *
FROM (
    SELECT user_id, event_type, 1 AS occurred
    FROM user_events
)
PIVOT (
    COUNT(occurred)
    FOR event_type IN (
        'page_view'    AS f_page_view,
        'add_to_cart'  AS f_add_to_cart,
        'purchase'     AS f_purchase,
        'refund'       AS f_refund
    )
);
```

---

## 6. 성능 고려사항

### 데이터 크기 폭발 (Data Explosion)

PIVOT은 행 수가 줄고 컬럼 수가 늘어납니다. 하지만 **카디널리티(Cardinality)** 가 높은 컬럼을 PIVOT하면 컬럼 수가 수백~수천 개로 폭발할 수 있습니다.

| 피벗 컬럼 카디널리티 | 결과 컬럼 수 | 위험성 |
|---------------------|-------------|--------|
| 낮음 (< 50)         | 안전        | 낮음   |
| 중간 (50~200)       | 주의 필요   | 중간   |
| 높음 (> 200)        | 위험        | 높음   |

- Spark/Delta는 컬럼 수가 많아질수록 **스키마 관리 비용**, **직렬화 오버헤드**, **메모리 사용량** 이 증가합니다.
- 컬럼 수가 많은 Delta 테이블은 `OPTIMIZE` 성능도 저하됩니다.

### 셔플(Shuffle) 비용

PIVOT 내부는 GROUP BY + 집계로 동작하므로 **셔플(Shuffle)** 이 발생합니다. 데이터가 클수록:

```sql
-- 파티션 필터를 먼저 적용해 셔플 대상 데이터 최소화
SELECT *
FROM (
    SELECT month, product_name, amount
    FROM orders
    WHERE order_date >= '2025-01-01'  -- 파티션 프루닝 먼저!
)
PIVOT (
    SUM(amount) FOR product_name IN ('Laptop' AS laptop, 'Mouse' AS mouse)
);
```

---

## 7. 장단점과 대안 — PIVOT vs CASE WHEN

### CASE WHEN 수동 피벗

```sql
SELECT
    DATE_FORMAT(order_date, 'yyyy-MM') AS month,
    SUM(CASE WHEN product_name = 'Laptop'   THEN amount END) AS laptop,
    SUM(CASE WHEN product_name = 'Mouse'    THEN amount END) AS mouse,
    SUM(CASE WHEN product_name = 'Keyboard' THEN amount END) AS keyboard,
    SUM(amount)                                               AS total
FROM orders
GROUP BY 1
ORDER BY month;
```

### 비교표

| 기준 | PIVOT 구문 | CASE WHEN |
|------|-----------|-----------|
| 가독성 | 높음 (선언적) | 낮음 (반복 코드) |
| 동적 컬럼 | SQL Scripting 필요 | Python/Jinja 조합 필요 |
| 성능 | 동일 (내부 동작 같음) | 동일 |
| 복잡한 조건 | 불가 | 가능 (`AND`, 범위 조건 등) |
| 집계 후 합계 컬럼 추가 | 불편 | 쉬움 (`SUM(amount) AS total`) |
| 표준 SQL 호환성 | 방언마다 다름 | ANSI 표준, 광범위 지원 |

**권장:** 컬럼 목록이 고정되고 가독성이 중요한 경우 PIVOT 구문을, 복잡한 필터 조건이나 합계 컬럼이 필요한 경우 CASE WHEN을 선택합니다.

---

## 8. 흔한 실수

### 8-1. NULL 처리 누락

PIVOT 결과에서 매칭되는 값이 없으면 NULL이 됩니다. 리포트에서 NULL 대신 0을 보여주려면 `COALESCE`를 사용합니다.

```sql
SELECT
    month,
    COALESCE(laptop,   0) AS laptop,
    COALESCE(mouse,    0) AS mouse,
    COALESCE(keyboard, 0) AS keyboard
FROM (
    SELECT DATE_FORMAT(order_date, 'yyyy-MM') AS month, product_name, amount
    FROM orders
)
PIVOT (SUM(amount) FOR product_name IN ('Laptop' AS laptop, 'Mouse' AS mouse, 'Keyboard' AS keyboard));
```

### 8-2. 집계 함수 누락

`FOR ... IN` 절의 값과 `집계함수(값_컬럼)` 을 모두 지정해야 합니다. 집계 함수 없이 단순 값을 나열하면 오류가 발생합니다.

```sql
-- 잘못된 예 (집계 함수 없음 → 오류)
PIVOT (amount FOR product_name IN ('Laptop', 'Mouse'))

-- 올바른 예
PIVOT (SUM(amount) FOR product_name IN ('Laptop' AS laptop, 'Mouse' AS mouse))
```

### 8-3. 컬럼 별칭의 특수문자

PIVOT 결과 컬럼명에 공백, 하이픈, 한글 등 특수문자가 포함되면 이후 쿼리에서 반드시 백틱(`)으로 감싸야 합니다.

```sql
-- 별칭에 공백/한글 포함 시
PIVOT (SUM(amount) FOR product_name IN ('Laptop Pro' AS `Laptop Pro`, '마우스' AS `마우스`))

-- 이후 참조 시
SELECT `Laptop Pro`, `마우스` FROM pivot_result;
```

**권장:** 영문 소문자 + 언더스코어 형식(`laptop_pro`)으로 별칭을 정의해 특수문자 문제를 원천 차단합니다.

### 8-4. UNPIVOT 대상 컬럼의 데이터 타입 불일치

UNPIVOT하는 컬럼들은 **동일한 데이터 타입** 이어야 합니다. 타입이 다르면 명시적 CAST가 필요합니다.

```sql
SELECT *
FROM (
    SELECT year,
           CAST(q1 AS BIGINT) AS q1,
           CAST(q2 AS BIGINT) AS q2,
           CAST(q3 AS BIGINT) AS q3
    FROM mixed_type_table
)
UNPIVOT (revenue FOR quarter IN (q1, q2, q3));
```

---

## 참고 링크

- [Databricks SQL — PIVOT 공식 문서](https://docs.databricks.com/en/sql/language-manual/sql-ref-syntax-qry-select-pivot.html)
- [Databricks SQL — UNPIVOT 공식 문서](https://docs.databricks.com/en/sql/language-manual/sql-ref-syntax-qry-select-unpivot.html)
- [Databricks SQL Scripting — EXECUTE IMMEDIATE](https://docs.databricks.com/en/sql/language-manual/sql-ref-syntax-ddl-execute-immediate.html)
- [PySpark DataFrame.pivot()](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.GroupedData.pivot.html)
- [Databricks Best Practices — Query Performance](https://docs.databricks.com/en/optimizations/index.html)
