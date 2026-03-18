# SQL 주요 기능

## Databricks SQL의 고급 기능

Databricks SQL은 표준 SQL(ANSI SQL)을 지원하며, 추가로 분석에 유용한 고급 기능들을 제공합니다.

---

## 파이프 구문 (Pipe Syntax)

> 🆕 **파이프 구문(`|>`)**은 SQL 쿼리를 **위에서 아래로 순서대로** 읽을 수 있게 해 주는 Databricks의 새로운 SQL 구문입니다.

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

---

## CTE (Common Table Expression)

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

---

## 윈도우 함수

> 💡 **윈도우 함수(Window Function)**는 GROUP BY 없이도 그룹 내에서 집계, 순위, 누적 계산을 할 수 있는 함수입니다.

```sql
SELECT
    customer_id,
    order_date,
    amount,
    -- 고객별 누적 매출
    SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS cumulative_revenue,
    -- 고객별 순위
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rank,
    -- 이전 주문 금액
    LAG(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_amount
FROM silver_orders;
```

---

## SQL 스크립팅

> 💡 **SQL 스크립팅**을 사용하면 변수, 조건문, 반복문을 포함한 절차적 SQL 코드를 작성할 수 있습니다.

```sql
DECLARE target_date DATE DEFAULT CURRENT_DATE() - INTERVAL 1 DAY;
DECLARE row_count INT;

SET row_count = (SELECT COUNT(*) FROM silver_orders WHERE order_date = target_date);

IF row_count = 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '어제 주문 데이터가 없습니다!';
END IF;
```

> 🆕 **Multi-Table Transactions**: `BEGIN ATOMIC ... END;` 구문으로 여러 테이블에 걸친 원자적 트랜잭션을 수행할 수 있게 되었습니다.

---

## 참고 링크

- [Databricks: SQL language reference](https://docs.databricks.com/aws/en/sql/language-manual/)
