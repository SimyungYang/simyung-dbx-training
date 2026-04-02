# 윈도우 함수 상세 가이드

## 윈도우 함수란?

**윈도우 함수(Window Function)** 는 `GROUP BY` 없이도 각 행에 대해 **관련된 행들의 집합(윈도우)** 을 기준으로 계산을 수행하는 함수입니다. 일반 집계 함수와 달리 행의 개수를 줄이지 않고, 각 행에 분석 결과를 추가합니다.

> 💡 **비유**: 윈도우 함수는 "나의 성적이 반 전체에서 몇 등인지" 알려주는 것과 같습니다. 반 전체 성적표가 유지되면서, 각 학생 옆에 순위가 추가됩니다.

---

## OVER 절 기본 구문

```sql
함수() OVER (
    [PARTITION BY 파티션_컬럼]    -- 그룹 나누기
    [ORDER BY 정렬_컬럼]          -- 그룹 내 정렬
    [프레임 지정]                  -- 계산 범위 제한
)
```

| 요소 | 설명 | 필수 여부 |
|------|------|----------|
| `PARTITION BY` | 데이터를 논리적 그룹으로 나눕니다 | 선택 (생략 시 전체가 하나의 파티션) |
| `ORDER BY` | 파티션 내에서 행의 순서를 결정합니다 | 함수에 따라 다름 |
| `프레임 지정` | 현재 행 기준으로 계산 범위를 지정합니다 | 선택 |

---

## 순위 함수

### ROW_NUMBER, RANK, DENSE_RANK, NTILE

```sql
SELECT
    department,
    employee_name,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank,
    NTILE(4)     OVER (PARTITION BY department ORDER BY salary DESC) AS quartile
FROM employees;
```

### 순위 함수 비교

| 함수 | 동일 값 처리 | 예시 (값: 100, 90, 90, 80) |
|------|-------------|--------------------------|
| `ROW_NUMBER()` | 항상 고유 순번 | 1, 2, 3, 4 |
| `RANK()` | 같은 순위, 다음 순위 건너뜀 | 1, 2, 2, 4 |
| `DENSE_RANK()` | 같은 순위, 다음 순위 연속 | 1, 2, 2, 3 |
| `NTILE(N)` | N개 그룹으로 균등 분배 | NTILE(2) → 1, 1, 2, 2 |

### 실전 예제: 부서별 급여 순위

```sql
-- 부서별 급여 상위 3명 조회
SELECT department, employee_name, salary
FROM employees
QUALIFY ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) <= 3;
```

---

## 집계 윈도우 함수

`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`를 윈도우 함수로 사용하면 행을 줄이지 않고 집계 결과를 추가합니다.

```sql
SELECT
    order_date,
    amount,
    -- 전체 매출 대비 비율
    ROUND(amount / SUM(amount) OVER () * 100, 2) AS pct_of_total,
    -- 월별 평균 대비 차이
    amount - AVG(amount) OVER (
        PARTITION BY DATE_TRUNC('MONTH', order_date)
    ) AS diff_from_monthly_avg,
    -- 누적 합계
    SUM(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM daily_sales;
```

### 누적 합계와 이동 평균

```sql
SELECT
    sale_date,
    daily_revenue,
    -- 누적 합계 (Cumulative Sum)
    SUM(daily_revenue) OVER (
        ORDER BY sale_date
    ) AS cumulative_revenue,
    -- 7일 이동 평균 (Moving Average)
    ROUND(AVG(daily_revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 0) AS moving_avg_7d,
    -- 30일 이동 합계
    SUM(daily_revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS moving_sum_30d
FROM daily_sales;
```

---

## 오프셋 함수: LAG, LEAD, FIRST_VALUE, LAST_VALUE

### LAG / LEAD (전후 행 참조)

```sql
SELECT
    sale_date,
    daily_revenue,
    -- 전일 매출
    LAG(daily_revenue, 1, 0) OVER (ORDER BY sale_date) AS prev_day,
    -- 익일 매출
    LEAD(daily_revenue, 1, 0) OVER (ORDER BY sale_date) AS next_day,
    -- 전일 대비 변화율
    ROUND(
        (daily_revenue - LAG(daily_revenue) OVER (ORDER BY sale_date))
        / LAG(daily_revenue) OVER (ORDER BY sale_date) * 100, 1
    ) AS day_over_day_pct,
    -- 전주 동일 요일 매출 (7일 전)
    LAG(daily_revenue, 7) OVER (ORDER BY sale_date) AS same_day_last_week
FROM daily_sales;
```

| 함수 | 설명 | 인자 |
|------|------|------|
| `LAG(col, n, default)` | n행 이전의 값을 반환합니다 | n: 오프셋(기본 1), default: 기본값 |
| `LEAD(col, n, default)` | n행 이후의 값을 반환합니다 | n: 오프셋(기본 1), default: 기본값 |

### FIRST_VALUE / LAST_VALUE

```sql
SELECT
    department,
    employee_name,
    salary,
    -- 부서 내 최고 급여자 이름
    FIRST_VALUE(employee_name) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS top_earner,
    -- 부서 내 최저 급여자 이름
    LAST_VALUE(employee_name) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_earner
FROM employees;
```

> ⚠️ **LAST_VALUE 주의사항**: 프레임을 명시하지 않으면 기본 프레임(`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`)이 적용되어, 현재 행까지만 보입니다. 전체 파티션의 마지막 값을 얻으려면 `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`을 명시해야 합니다.

---

## 프레임 지정: ROWS BETWEEN, RANGE BETWEEN

### 프레임 구문

```sql
{ROWS | RANGE} BETWEEN 시작점 AND 끝점
```

| 프레임 경계 | 의미 |
|-----------|------|
| `UNBOUNDED PRECEDING` | 파티션의 첫 행입니다 |
| `N PRECEDING` | 현재 행의 N행 이전입니다 |
| `CURRENT ROW` | 현재 행입니다 |
| `N FOLLOWING` | 현재 행의 N행 이후입니다 |
| `UNBOUNDED FOLLOWING` | 파티션의 마지막 행입니다 |

### ROWS vs RANGE

| 구분 | ROWS | RANGE |
|------|------|-------|
| **기준** | 물리적 행 수 | 논리적 값 범위 |
| **동일 값 처리** | 각 행을 개별로 취급합니다 | 같은 값의 행을 하나로 취급합니다 |
| **사용 빈도** | 더 자주 사용됩니다 | 날짜/시간 범위에 유용합니다 |

```sql
-- ROWS: 물리적으로 전후 1행씩 (총 3행)
AVG(value) OVER (ORDER BY date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)

-- RANGE: 논리적으로 같은 날짜 범위
SUM(value) OVER (ORDER BY date RANGE BETWEEN INTERVAL 7 DAYS PRECEDING AND CURRENT ROW)
```

---

## QUALIFY 절

QUALIFY는 윈도우 함수의 결과를 **서브쿼리 없이 직접 필터링** 하는 Databricks SQL 전용 절입니다.

```sql
-- ❌ 서브쿼리 방식 (전통적)
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
) WHERE rn = 1;

-- ✅ QUALIFY 방식 (간결)
SELECT *
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;
```

### QUALIFY 활용 패턴

```sql
-- 고객별 최근 주문만 조회
SELECT customer_id, order_date, amount
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;

-- 카테고리별 매출 상위 5개 제품
SELECT category, product_name, total_sales
FROM product_sales
QUALIFY DENSE_RANK() OVER (PARTITION BY category ORDER BY total_sales DESC) <= 5;
```

---

## 실전 예제

### 매출 순위와 점유율 분석

```sql
WITH monthly_sales AS (
    SELECT
        product_name,
        DATE_TRUNC('MONTH', sale_date) AS month,
        SUM(amount) AS revenue
    FROM sales
    GROUP BY 1, 2
)
SELECT
    month,
    product_name,
    FORMAT_NUMBER(revenue, 0) AS revenue,
    RANK() OVER (PARTITION BY month ORDER BY revenue DESC) AS rank,
    ROUND(revenue / SUM(revenue) OVER (PARTITION BY month) * 100, 1) AS market_share_pct,
    FORMAT_NUMBER(
        SUM(revenue) OVER (PARTITION BY month ORDER BY revenue DESC), 0
    ) AS cumulative_revenue
FROM monthly_sales
ORDER BY month, rank;
```

### 고객 이탈 분석 (마지막 구매 이후 경과일)

```sql
SELECT
    customer_id,
    order_date,
    amount,
    LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_date,
    DATEDIFF(
        order_date,
        LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)
    ) AS days_between_orders
FROM orders
QUALIFY days_between_orders > 90 OR days_between_orders IS NULL;
```

---

## 정리

| 함수 카테고리 | 함수 | 핵심 포인트 |
|-------------|------|-----------|
| **순위** | ROW_NUMBER, RANK, DENSE_RANK, NTILE | 파티션 내 행에 순위를 부여합니다 |
| **집계** | SUM, AVG, COUNT OVER | 행을 줄이지 않고 집계를 추가합니다 |
| **오프셋** | LAG, LEAD, FIRST_VALUE, LAST_VALUE | 전후 행이나 첫/끝 값을 참조합니다 |
| **프레임** | ROWS BETWEEN, RANGE BETWEEN | 계산 범위를 세밀하게 제어합니다 |
| **QUALIFY** | QUALIFY 절 | 윈도우 함수 결과를 서브쿼리 없이 필터링합니다 |

---

## 참고 링크

- [Databricks: Window functions](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-functions-builtin.html#window-functions)
- [Databricks: QUALIFY clause](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify.html)
- [Databricks: Analytic functions](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-window-functions.html)
- [Azure Databricks: Window functions](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-window-functions)
