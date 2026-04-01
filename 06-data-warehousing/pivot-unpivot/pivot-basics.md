# PIVOT 기본과 UNPIVOT

## PIVOT / UNPIVOT이란?

데이터 분석에서는 같은 데이터를 **다른 형태로 바라볼** 필요가 자주 있습니다. **PIVOT**은 행(Row)을 열(Column)로 변환하여 **크로스탭(교차표)** 형태로 만들고, **UNPIVOT**은 반대로 열을 행으로 변환합니다.

> 💡 **비유**: PIVOT은 세로로 나열된 성적표를 과목별 컬럼이 있는 표로 변환하는 것이고, UNPIVOT은 그 반대입니다.

---

## PIVOT: 행을 열로 변환

### 기본 구문

```sql
SELECT *
FROM 소스_테이블
PIVOT (
    집계함수(값_컬럼)
    FOR 피벗_컬럼 IN (값1 AS 별칭1, 값2 AS 별칭2, ...)
)
```

### 예제: 월별 제품 매출 피벗

```sql
-- 원본 데이터
-- | month   | product  | amount    |
-- |---------|----------|-----------|
-- | 2025-01 | Laptop   | 3,300,000 |
-- | 2025-01 | Mouse    | 25,000    |
-- | 2025-02 | Laptop   | 1,500,000 |
-- | 2025-02 | Mouse    | 50,000    |

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

결과:

| month | laptop | mouse | keyboard | monitor |
|-------|--------|-------|----------|---------|
| 2025-01 | 3,300,000 | 25,000 | 89,000 | NULL |
| 2025-02 | 1,500,000 | 50,000 | 178,000 | 550,000 |

### 여러 집계 함수 사용

```sql
-- 제품별 매출 합계와 건수를 동시에 피벗
SELECT *
FROM (
    SELECT
        region,
        product_name,
        amount
    FROM orders
)
PIVOT (
    SUM(amount) AS total,
    COUNT(*) AS cnt
    FOR product_name IN ('Laptop' AS laptop, 'Mouse' AS mouse)
);
```

결과에는 `laptop_total`, `laptop_cnt`, `mouse_total`, `mouse_cnt` 컬럼이 생성됩니다.

### CASE WHEN으로 수동 피벗

`PIVOT` 구문 대신 `CASE WHEN`으로도 같은 결과를 얻을 수 있습니다. 동적 컬럼이 필요하거나 더 복잡한 로직이 필요할 때 유용합니다.

```sql
SELECT
    DATE_FORMAT(order_date, 'yyyy-MM') AS month,
    SUM(CASE WHEN product_name = 'Laptop' THEN amount END) AS laptop,
    SUM(CASE WHEN product_name = 'Mouse' THEN amount END) AS mouse,
    SUM(CASE WHEN product_name = 'Keyboard' THEN amount END) AS keyboard,
    SUM(amount) AS total
FROM orders
GROUP BY 1
ORDER BY month;
```

---

## UNPIVOT: 열을 행으로 변환

### 기본 구문

```sql
SELECT *
FROM 소스_테이블
UNPIVOT (
    값_컬럼 FOR 카테고리_컬럼 IN (컬럼1, 컬럼2, ...)
)
```

### 예제: 분기별 매출 언피벗

```sql
-- 원본 데이터 (피벗된 형태)
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

| year | quarter | revenue |
|------|---------|---------|
| 2024 | q1 | 1,200,000 |
| 2024 | q2 | 1,500,000 |
| 2024 | q3 | 1,300,000 |
| 2024 | q4 | 1,800,000 |
| 2025 | q1 | 1,400,000 |
| 2025 | q2 | 1,600,000 |

> 💡 **UNPIVOT은 NULL을 자동으로 제외합니다.** 위 예제에서 2025년 Q3, Q4는 NULL이므로 결과에 포함되지 않습니다. NULL도 포함하려면 `UNPIVOT INCLUDE NULLS`를 사용하세요.

### INCLUDE NULLS

```sql
SELECT *
FROM quarterly_revenue
UNPIVOT INCLUDE NULLS (
    revenue FOR quarter IN (q1, q2, q3, q4)
)
ORDER BY year, quarter;
```

---
