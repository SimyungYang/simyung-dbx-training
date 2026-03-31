# SQL 주요 기능

## Databricks SQL의 고급 기능

Databricks SQL은 표준 SQL(ANSI SQL)을 지원하며, 추가로 분석에 유용한 고급 기능들을 제공합니다. 이 문서에서는 Databricks SQL만의 차별화된 기능과 고급 SQL 패턴을 살펴보겠습니다.

> 💡 **Databricks SQL은 Apache Spark SQL을 기반**으로 하며, 여기에 파이프 구문, QUALIFY, SQL 스크립팅 등 생산성을 높이는 확장 기능을 더했습니다.

---

## 파이프 구문 (Pipe Syntax)

> 🆕 **파이프 구문(`|>`)** 은 SQL 쿼리를 **위에서 아래로 순서대로** 읽을 수 있게 해 주는 Databricks의 SQL 구문입니다. 전통 SQL에서는 실행 순서(FROM → WHERE → GROUP BY → SELECT)와 작성 순서(SELECT → FROM → WHERE)가 다르지만, 파이프 구문은 **작성 순서 = 실행 순서**입니다.

### 전통 SQL vs 파이프 구문 비교

```sql
-- 전통적 SQL: 실행 순서와 작성 순서가 다릅니다
SELECT city, COUNT(*) AS cnt
FROM customers
WHERE signup_date >= '2025-01-01'
GROUP BY city
HAVING cnt >= 10
ORDER BY cnt DESC
LIMIT 5;

-- 파이프 구문: 위에서 아래로 자연스럽게 읽힙니다
FROM customers
|> WHERE signup_date >= '2025-01-01'
|> AGGREGATE COUNT(*) AS cnt GROUP BY city
|> WHERE cnt >= 10
|> ORDER BY cnt DESC
|> LIMIT 5;
```

### 파이프 구문의 핵심 규칙

| 규칙 | 설명 |
|------|------|
| **시작** | 반드시 `FROM` 절로 시작합니다 |
| **파이프 연산자** | 각 단계를 `\|>`로 연결합니다 |
| **집계** | `GROUP BY` 대신 `AGGREGATE ... GROUP BY`를 사용합니다 |
| **HAVING 대체** | 집계 후 필터는 `\|> WHERE`로 작성합니다 (HAVING 불필요) |
| **SELECT** | `\|> SELECT`로 컬럼을 선택하거나 변환합니다 |
| **중간 결과 활용** | 각 단계의 결과가 다음 단계의 입력이 됩니다 |

### 복잡한 분석에서의 파이프 구문

```sql
-- 고객별 월간 매출 추이에서 상위 10명 추출
FROM orders
|> JOIN customers ON orders.customer_id = customers.customer_id
|> WHERE order_date >= '2025-01-01'
|> AGGREGATE
    SUM(amount) AS total_revenue,
    COUNT(*) AS order_count
   GROUP BY customers.customer_id, customers.name, DATE_TRUNC('MONTH', order_date) AS month
|> WHERE total_revenue >= 100000
|> ORDER BY total_revenue DESC
|> LIMIT 10
|> SELECT name, month, total_revenue, order_count;
```

> 💡 **파이프 구문의 장점**: 쿼리가 길어질수록 전통 SQL은 읽기 어려워지지만, 파이프 구문은 **데이터가 흐르는 순서대로** 읽을 수 있어 가독성이 유지됩니다. 특히 여러 단계의 집계와 필터가 필요한 분석에서 효과적입니다.

---

## CTE (Common Table Expression)

CTE는 `WITH` 절을 사용하여 **쿼리 내에서 이름 붙은 임시 결과 집합**을 정의하는 기능입니다. 복잡한 쿼리를 논리적 단계로 나누어 **가독성과 재사용성**을 높입니다.

### 기본 CTE

```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('MONTH', order_date) AS month,
        SUM(amount) AS revenue
    FROM silver_orders
    GROUP BY 1
),
growth AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
        ROUND((revenue - LAG(revenue) OVER (ORDER BY month))
            / LAG(revenue) OVER (ORDER BY month) * 100, 1) AS growth_pct
    FROM monthly_sales
)
SELECT * FROM growth ORDER BY month;
```

### Recursive CTE (재귀 CTE)

재귀 CTE는 **자기 자신을 참조**하여 계층적 데이터(조직도, 카테고리 트리 등)를 처리합니다.

```sql
-- 조직도에서 특정 관리자의 모든 하위 직원 조회
WITH RECURSIVE org_tree AS (
    -- 기본 케이스: 최상위 관리자
    SELECT employee_id, name, manager_id, 1 AS level
    FROM employees
    WHERE employee_id = 1001  -- CEO

    UNION ALL

    -- 재귀 케이스: 하위 직원 탐색
    SELECT e.employee_id, e.name, e.manager_id, t.level + 1
    FROM employees e
    INNER JOIN org_tree t ON e.manager_id = t.employee_id
)
SELECT
    REPEAT('  ', level - 1) || name AS org_chart,
    level
FROM org_tree
ORDER BY level, name;
```

> ⚠️ **재귀 CTE 주의사항**: 무한 루프를 방지하기 위해 Databricks는 기본적으로 최대 재귀 깊이를 제한합니다. `spark.sql.cte.maxRecursionDepth` 설정으로 조정할 수 있습니다.

---

## 윈도우 함수 심화

> 💡 **윈도우 함수(Window Function)** 는 GROUP BY 없이도 그룹 내에서 집계, 순위, 누적 계산을 할 수 있는 함수입니다. 행의 개수를 줄이지 않고 각 행에 분석 결과를 추가합니다.

### 순위 함수 비교

```sql
SELECT
    department,
    employee_name,
    salary,
    -- ROW_NUMBER: 동일 값이 있어도 고유한 순번 부여
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    -- RANK: 동일 값이면 같은 순위, 다음 순위 건너뜀 (1, 2, 2, 4)
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
    -- DENSE_RANK: 동일 값이면 같은 순위, 다음 순위 연속 (1, 2, 2, 3)
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank,
    -- NTILE: N개의 버킷으로 균등 분배
    NTILE(4)     OVER (PARTITION BY department ORDER BY salary DESC) AS quartile
FROM employees;
```

| 함수 | 동일 값 처리 | 예시 (값: 100, 90, 90, 80) |
|------|-------------|--------------------------|
| `ROW_NUMBER()` | 고유한 순번 | 1, 2, 3, 4 |
| `RANK()` | 같은 순위, 건너뜀 | 1, 2, 2, 4 |
| `DENSE_RANK()` | 같은 순위, 연속 | 1, 2, 2, 3 |
| `NTILE(2)` | N개 그룹으로 분배 | 1, 1, 2, 2 |

### LAG / LEAD (전후 행 참조)

```sql
SELECT
    order_date,
    daily_revenue,
    -- 이전 행 값 (전일 매출)
    LAG(daily_revenue, 1) OVER (ORDER BY order_date) AS prev_day_revenue,
    -- 다음 행 값 (익일 매출)
    LEAD(daily_revenue, 1) OVER (ORDER BY order_date) AS next_day_revenue,
    -- 전일 대비 증감률
    ROUND(
        (daily_revenue - LAG(daily_revenue) OVER (ORDER BY order_date))
        / LAG(daily_revenue) OVER (ORDER BY order_date) * 100, 1
    ) AS day_over_day_pct
FROM daily_sales;
```

### 윈도우 프레임 지정

윈도우 프레임은 현재 행을 기준으로 **계산 범위를 세밀하게 제어**합니다.

```sql
SELECT
    order_date,
    daily_revenue,
    -- 최근 7일 이동 평균
    AVG(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d,
    -- 월초부터 현재까지 누적 합계
    SUM(daily_revenue) OVER (
        PARTITION BY DATE_TRUNC('MONTH', order_date)
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS mtd_revenue,
    -- 전체 기간 대비 비율
    ROUND(daily_revenue / SUM(daily_revenue) OVER () * 100, 2) AS pct_of_total
FROM daily_sales;
```

| 프레임 지정 | 의미 |
|------------|------|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | 파티션 시작부터 현재 행까지 |
| `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | 현재 행 포함 최근 7행 |
| `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` | 현재 행부터 파티션 끝까지 |
| `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` | 이전 행, 현재 행, 다음 행 (3행) |

---

## SQL 스크립팅

> 💡 **SQL 스크립팅**을 사용하면 변수, 조건문, 반복문을 포함한 **절차적 SQL 코드**를 작성할 수 있습니다. 복잡한 데이터 처리 로직을 SQL만으로 구현할 수 있습니다.

### 변수 선언과 제어문

```sql
-- 변수 선언
DECLARE target_date DATE DEFAULT CURRENT_DATE() - INTERVAL 1 DAY;
DECLARE row_count INT;
DECLARE status_msg STRING;

-- 변수에 값 할당
SET row_count = (SELECT COUNT(*) FROM silver_orders WHERE order_date = target_date);

-- 조건문
IF row_count = 0 THEN
    SET status_msg = '어제 주문 데이터가 없습니다!';
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = status_msg;
ELSEIF row_count < 100 THEN
    SET status_msg = '주문 데이터가 예상보다 적습니다: ' || CAST(row_count AS STRING);
    -- 로그 테이블에 기록
    INSERT INTO audit_log (message, created_at) VALUES (status_msg, CURRENT_TIMESTAMP());
ELSE
    SET status_msg = '정상 처리 완료. 주문 수: ' || CAST(row_count AS STRING);
END IF;

SELECT status_msg AS result;
```

### 반복문과 예외 처리

```sql
-- 반복문: 최근 12개월에 대해 월별 집계 테이블 생성
DECLARE i INT DEFAULT 0;

WHILE i < 12 DO
    DECLARE target_month DATE;
    SET target_month = DATE_TRUNC('MONTH', CURRENT_DATE()) - MAKE_INTERVAL(0, i);

    INSERT INTO monthly_summary
    SELECT
        target_month AS month,
        COUNT(*) AS order_count,
        SUM(amount) AS total_revenue
    FROM orders
    WHERE DATE_TRUNC('MONTH', order_date) = target_month;

    SET i = i + 1;
END WHILE;
```

> 🆕 **Multi-Table Transactions**: `BEGIN ATOMIC ... END;` 구문으로 여러 테이블에 걸친 원자적 트랜잭션을 수행할 수 있게 되었습니다.

```sql
-- 원자적 트랜잭션: 두 테이블을 동시에 업데이트
BEGIN ATOMIC
    UPDATE accounts SET balance = balance - 50000 WHERE account_id = 'A001';
    UPDATE accounts SET balance = balance + 50000 WHERE account_id = 'A002';
    INSERT INTO transactions (from_acc, to_acc, amount, txn_time)
    VALUES ('A001', 'A002', 50000, CURRENT_TIMESTAMP());
END;
```

---

## PIVOT / UNPIVOT

### PIVOT: 행을 열로 변환

```sql
-- 월별-제품별 매출을 피벗 테이블로 변환
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
    FOR product_name IN ('Laptop' AS laptop, 'Mouse' AS mouse, 'Keyboard' AS keyboard)
)
ORDER BY month;
```

결과 예시:

| month | laptop | mouse | keyboard |
|-------|--------|-------|----------|
| 2025-01 | 3,300,000 | 25,000 | 89,000 |
| 2025-02 | 1,500,000 | 50,000 | 178,000 |

### UNPIVOT: 열을 행으로 변환

```sql
-- 피벗된 데이터를 다시 행으로 변환
SELECT *
FROM quarterly_revenue
UNPIVOT (
    revenue FOR quarter IN (q1, q2, q3, q4)
)
ORDER BY year, quarter;
```

---

## QUALIFY 절

> 💡 **QUALIFY**는 윈도우 함수의 결과를 기준으로 행을 필터링하는 절입니다. 서브쿼리 없이 윈도우 함수 결과를 바로 필터링할 수 있어 쿼리가 훨씬 간결해집니다.

```sql
-- ❌ 전통적 방식: 서브쿼리 필요
SELECT * FROM (
    SELECT
        customer_id,
        order_date,
        amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
WHERE rn = 1;

-- ✅ QUALIFY 사용: 서브쿼리 불필요
SELECT
    customer_id,
    order_date,
    amount
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;
```

```sql
-- 부서별 급여 상위 3명 조회
SELECT
    department,
    employee_name,
    salary
FROM employees
QUALIFY DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) <= 3;
```

---

## LATERAL VIEW / EXPLODE (반구조화 데이터)

배열(Array)이나 맵(Map) 같은 반구조화 데이터를 행으로 펼칠 때 사용합니다.

```sql
-- 배열 데이터를 행으로 펼치기
SELECT
    order_id,
    customer_id,
    item
FROM orders
LATERAL VIEW EXPLODE(items) AS item;

-- EXPLODE를 직접 SELECT에서 사용 (더 간결한 방식)
SELECT
    order_id,
    customer_id,
    EXPLODE(items) AS item
FROM orders;

-- 배열의 인덱스도 함께 추출
SELECT
    order_id,
    pos AS item_index,
    item
FROM orders
LATERAL VIEW POSEXPLODE(items) AS pos, item;
```

### 중첩 JSON 처리

```sql
-- 중첩 JSON 데이터 처리 예시
-- 원본: {"user": "Alice", "purchases": [{"item": "A", "qty": 2}, {"item": "B", "qty": 1}]}
SELECT
    raw_data:user::STRING AS user_name,
    purchase.item::STRING AS item,
    purchase.qty::INT AS quantity
FROM events
LATERAL VIEW EXPLODE(FROM_JSON(
    raw_data:purchases,
    'ARRAY<STRUCT<item: STRING, qty: INT>>'
)) AS purchase;
```

---

## 고급 집계: GROUPING SETS, CUBE, ROLLUP

### GROUPING SETS

여러 GROUP BY 조합의 결과를 **한 번의 쿼리**로 계산합니다.

```sql
-- 지역별, 제품별, 전체 매출을 한 번에 계산
SELECT
    region,
    product,
    SUM(amount) AS total_revenue
FROM sales
GROUP BY GROUPING SETS (
    (region, product),  -- 지역 + 제품별
    (region),           -- 지역별 소계
    (product),          -- 제품별 소계
    ()                  -- 전체 합계
)
ORDER BY region NULLS LAST, product NULLS LAST;
```

### CUBE와 ROLLUP

```sql
-- CUBE: 모든 가능한 조합의 집계
SELECT region, product, SUM(amount) AS revenue
FROM sales
GROUP BY CUBE(region, product);
-- → (region, product), (region), (product), () 네 가지 조합

-- ROLLUP: 계층적 집계 (왼쪽에서 오른쪽으로 줄여가며 집계)
SELECT year, quarter, month, SUM(amount) AS revenue
FROM sales
GROUP BY ROLLUP(year, quarter, month);
-- → (year, quarter, month), (year, quarter), (year), () 네 가지 조합
```

| 함수 | 생성되는 집계 조합 | 사용 사례 |
|------|-----------------|----------|
| `GROUPING SETS` | 명시적으로 지정한 조합만 | 특정 조합의 소계가 필요할 때 |
| `CUBE` | 모든 가능한 조합 (2^n) | 다차원 분석, 다양한 각도의 집계 |
| `ROLLUP` | 계층적 조합 (n+1) | 시간 계층(연/분기/월), 지역 계층 |

---

## 날짜/시간 함수 심화

```sql
SELECT
    CURRENT_DATE()                              AS today,
    CURRENT_TIMESTAMP()                         AS now,
    DATE_TRUNC('MONTH', CURRENT_DATE())         AS month_start,
    LAST_DAY(CURRENT_DATE())                    AS month_end,
    DATE_ADD(CURRENT_DATE(), 30)                AS plus_30_days,
    DATEDIFF(CURRENT_DATE(), '2025-01-01')      AS days_since_jan1,
    MONTHS_BETWEEN(CURRENT_DATE(), '2025-01-01') AS months_diff,
    DATE_FORMAT(CURRENT_TIMESTAMP(), 'yyyy-MM-dd HH:mm:ss') AS formatted,
    EXTRACT(YEAR FROM CURRENT_DATE())           AS year,
    EXTRACT(QUARTER FROM CURRENT_DATE())        AS quarter,
    DAYOFWEEK(CURRENT_DATE())                   AS day_of_week,
    WEEKOFYEAR(CURRENT_DATE())                  AS week_number;
```

| 함수 | 설명 | 예시 결과 |
|------|------|----------|
| `DATE_TRUNC(unit, date)` | 지정 단위로 절삭합니다 | `DATE_TRUNC('MONTH', '2025-03-15')` → `2025-03-01` |
| `DATE_ADD(date, days)` | 날짜에 일수를 더합니다 | `DATE_ADD('2025-01-01', 30)` → `2025-01-31` |
| `DATEDIFF(end, start)` | 두 날짜 간 일수 차이입니다 | `DATEDIFF('2025-03-01', '2025-01-01')` → `59` |
| `LAST_DAY(date)` | 해당 월의 마지막 날입니다 | `LAST_DAY('2025-02-15')` → `2025-02-28` |
| `MONTHS_BETWEEN(d1, d2)` | 두 날짜 간 월 차이입니다 | `MONTHS_BETWEEN('2025-06-01', '2025-01-01')` → `5.0` |

---

## JSON / 반구조화 데이터 처리

Databricks SQL은 JSON 데이터를 **네이티브로 지원**합니다. 콜론(`:`) 구문으로 JSON 필드에 직접 접근할 수 있습니다.

```sql
-- JSON 컬럼에서 값 추출 (: 구문)
SELECT
    event_id,
    raw_data:user_id::INT          AS user_id,
    raw_data:event_type::STRING    AS event_type,
    raw_data:metadata.browser::STRING AS browser,
    raw_data:metadata.os::STRING   AS os,
    raw_data:tags[0]::STRING       AS first_tag
FROM raw_events;

-- GET 함수로 안전하게 접근 (키가 없으면 NULL 반환)
SELECT
    GET(raw_data, 'user_id')  AS user_id,
    GET(raw_data, 'optional_field') AS optional_field  -- NULL if missing
FROM raw_events;

-- JSON 배열을 행으로 펼치기
SELECT
    event_id,
    tag.value::STRING AS tag_name
FROM raw_events,
LATERAL VIEW EXPLODE(FROM_JSON(raw_data:tags, 'ARRAY<STRING>')) tag;

-- SCHEMA_OF_JSON: JSON 문자열의 스키마를 자동 추론
SELECT SCHEMA_OF_JSON('{"name": "Alice", "age": 30, "scores": [95, 87, 92]}');
-- 결과: STRUCT<age: BIGINT, name: STRING, scores: ARRAY<BIGINT>>
```

> 💡 **콜론(`:`) 구문의 장점**: JSON 파싱 함수를 호출할 필요 없이, SQL 컬럼처럼 자연스럽게 JSON 필드에 접근할 수 있습니다. 중첩 필드는 점(`.`)으로, 배열 요소는 대괄호(`[]`)로 접근합니다.

---

## 실습: 복잡한 분석 쿼리 (매출 분석 시나리오)

온라인 쇼핑몰의 매출 데이터를 분석하는 종합 시나리오입니다.

### 시나리오 설정

```sql
-- 샘플 데이터 준비
CREATE OR REPLACE TABLE catalog.schema.sales_analysis (
    sale_id      BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id  INT,
    product_name STRING,
    category     STRING,
    region       STRING,
    amount       DECIMAL(10,2),
    sale_date    DATE
) USING DELTA;

INSERT INTO catalog.schema.sales_analysis
    (customer_id, product_name, category, region, amount, sale_date)
VALUES
    (1, 'Laptop Pro', 'Electronics', 'Seoul', 1500000, '2025-01-05'),
    (1, 'Mouse', 'Accessories', 'Seoul', 35000, '2025-01-10'),
    (2, 'Laptop Air', 'Electronics', 'Busan', 1200000, '2025-01-08'),
    (2, 'Keyboard', 'Accessories', 'Busan', 89000, '2025-02-15'),
    (3, 'Monitor 4K', 'Electronics', 'Seoul', 550000, '2025-02-20'),
    (3, 'Laptop Pro', 'Electronics', 'Seoul', 1500000, '2025-03-01'),
    (1, 'Webcam', 'Accessories', 'Seoul', 120000, '2025-03-10'),
    (4, 'Laptop Air', 'Electronics', 'Daegu', 1200000, '2025-03-15'),
    (2, 'Monitor 4K', 'Electronics', 'Busan', 550000, '2025-03-20'),
    (5, 'Laptop Pro', 'Electronics', 'Incheon', 1500000, '2025-03-25');
```

### 분석 1: 월별 매출 추이 및 전월 대비 성장률

```sql
WITH monthly AS (
    SELECT
        DATE_TRUNC('MONTH', sale_date) AS month,
        COUNT(*)    AS order_count,
        SUM(amount) AS revenue
    FROM catalog.schema.sales_analysis
    GROUP BY 1
)
SELECT
    DATE_FORMAT(month, 'yyyy-MM') AS month,
    order_count,
    FORMAT_NUMBER(revenue, 0) AS revenue,
    FORMAT_NUMBER(LAG(revenue) OVER (ORDER BY month), 0) AS prev_month,
    CONCAT(
        ROUND((revenue - LAG(revenue) OVER (ORDER BY month))
            / LAG(revenue) OVER (ORDER BY month) * 100, 1),
        '%'
    ) AS growth
FROM monthly
ORDER BY month;
```

### 분석 2: 고객별 RFM 분석 (Recency, Frequency, Monetary)

```sql
WITH rfm_base AS (
    SELECT
        customer_id,
        DATEDIFF(CURRENT_DATE(), MAX(sale_date)) AS recency,
        COUNT(*) AS frequency,
        SUM(amount) AS monetary
    FROM catalog.schema.sales_analysis
    GROUP BY customer_id
),
rfm_scored AS (
    SELECT
        customer_id,
        recency,
        frequency,
        monetary,
        NTILE(3) OVER (ORDER BY recency ASC)   AS r_score,  -- 최근일수록 높은 점수
        NTILE(3) OVER (ORDER BY frequency DESC) AS f_score,
        NTILE(3) OVER (ORDER BY monetary DESC)  AS m_score
    FROM rfm_base
)
SELECT
    customer_id,
    recency || '일 전' AS last_purchase,
    frequency || '회' AS purchase_count,
    FORMAT_NUMBER(monetary, 0) AS total_spent,
    CONCAT(r_score, f_score, m_score) AS rfm_score,
    CASE
        WHEN r_score >= 2 AND f_score >= 2 AND m_score >= 2 THEN 'VIP'
        WHEN r_score >= 2 AND f_score >= 2 THEN '충성 고객'
        WHEN r_score >= 2 THEN '신규/활성 고객'
        WHEN f_score >= 2 THEN '휴면 위험'
        ELSE '이탈 고객'
    END AS segment
FROM rfm_scored
ORDER BY monetary DESC;
```

### 분석 3: 지역-카테고리별 크로스탭 (PIVOT + ROLLUP)

```sql
-- 지역별 카테고리 매출 크로스탭 + 소계
SELECT
    COALESCE(region, '** 전체 **') AS region,
    FORMAT_NUMBER(SUM(CASE WHEN category = 'Electronics' THEN amount END), 0) AS electronics,
    FORMAT_NUMBER(SUM(CASE WHEN category = 'Accessories' THEN amount END), 0) AS accessories,
    FORMAT_NUMBER(SUM(amount), 0) AS total
FROM catalog.schema.sales_analysis
GROUP BY ROLLUP(region)
ORDER BY region NULLS LAST;
```

---

## 정리

| 기능 | 핵심 포인트 |
|------|------------|
| **파이프 구문** | `\|>`로 쿼리를 위에서 아래로 읽을 수 있게 합니다 |
| **CTE / Recursive CTE** | 복잡한 쿼리를 단계별로 분리하고, 계층 데이터를 처리합니다 |
| **윈도우 함수** | GROUP BY 없이 순위, 누적, 이동 평균을 계산합니다 |
| **SQL 스크립팅** | 변수, 조건문, 반복문으로 절차적 로직을 SQL로 작성합니다 |
| **PIVOT / UNPIVOT** | 행/열을 변환하여 크로스탭 분석을 수행합니다 |
| **QUALIFY** | 윈도우 함수 결과를 서브쿼리 없이 바로 필터링합니다 |
| **EXPLODE** | 배열, 맵 데이터를 행으로 펼칩니다 |
| **GROUPING SETS / CUBE / ROLLUP** | 여러 집계 조합을 한 번에 계산합니다 |
| **JSON 처리** | 콜론(`:`) 구문으로 JSON 필드에 직접 접근합니다 |

---

## 참고 링크

- [Databricks: SQL language reference](https://docs.databricks.com/aws/en/sql/language-manual/)
- [Databricks: Pipe syntax](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-pipe-syntax.html)
- [Databricks: Window functions](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-window-functions.html)
- [Databricks: SQL scripting](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-scripting.html)
- [Databricks: QUALIFY clause](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify.html)
- [Databricks: Semi-structured data](https://docs.databricks.com/aws/en/query/semi-structured.html)
- [Azure Databricks: SQL reference](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/)
