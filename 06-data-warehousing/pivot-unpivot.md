# PIVOT과 UNPIVOT

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

## 실전 활용 사례

### 1. 크로스탭 리포트 생성 (PIVOT)

```sql
-- 지역별-카테고리별 매출 크로스탭
SELECT
    COALESCE(region, '** 전체 **') AS region,
    FORMAT_NUMBER(electronics, 0) AS electronics,
    FORMAT_NUMBER(accessories, 0) AS accessories,
    FORMAT_NUMBER(furniture, 0) AS furniture,
    FORMAT_NUMBER(COALESCE(electronics, 0) + COALESCE(accessories, 0) + COALESCE(furniture, 0), 0) AS total
FROM (
    SELECT region, category, amount
    FROM sales
)
PIVOT (
    SUM(amount)
    FOR category IN ('Electronics' AS electronics, 'Accessories' AS accessories, 'Furniture' AS furniture)
)
ORDER BY region NULLS LAST;
```

### 2. 설문 데이터 정규화 (UNPIVOT)

```sql
-- 설문 응답이 컬럼으로 펼쳐진 형태를 정규화
-- 원본: | user_id | q1_answer | q2_answer | q3_answer |
SELECT *
FROM survey_responses
UNPIVOT (
    answer FOR question IN (q1_answer AS 'Q1', q2_answer AS 'Q2', q3_answer AS 'Q3')
)
ORDER BY user_id, question;
-- 결과: | user_id | question | answer |
```

### 3. 시계열 데이터 변환 (PIVOT + UNPIVOT)

```sql
-- 센서 데이터를 센서별 컬럼으로 피벗
SELECT *
FROM (
    SELECT
        DATE_TRUNC('HOUR', timestamp) AS hour,
        sensor_id,
        AVG(value) AS avg_value
    FROM sensor_readings
    GROUP BY 1, 2
)
PIVOT (
    AVG(avg_value)
    FOR sensor_id IN ('temp_01' AS temperature, 'humid_01' AS humidity, 'press_01' AS pressure)
)
ORDER BY hour;
```

### 4. 요일별 매출 히트맵 (PIVOT)

```sql
-- 시간대별-요일별 매출 히트맵
SELECT *
FROM (
    SELECT
        HOUR(order_time) AS hour_of_day,
        DAYOFWEEK(order_date) AS day_of_week,
        amount
    FROM orders
)
PIVOT (
    SUM(amount)
    FOR day_of_week IN (
        1 AS sunday,
        2 AS monday,
        3 AS tuesday,
        4 AS wednesday,
        5 AS thursday,
        6 AS friday,
        7 AS saturday
    )
)
ORDER BY hour_of_day;
```

---

## PIVOT vs UNPIVOT 비교

| 비교 항목 | PIVOT | UNPIVOT |
|-----------|-------|---------|
| **변환 방향** | 행 → 열 | 열 → 행 |
| **결과 형태** | 넓은(Wide) 테이블 | 긴(Long) 테이블 |
| **집계 함수** | 필수 (SUM, AVG, COUNT 등) | 불필요 |
| **NULL 처리** | 값이 없으면 NULL | 기본적으로 NULL 행 제외 |
| **사용 사례** | 크로스탭 리포트, 대시보드 | 데이터 정규화, ETL 변환 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **PIVOT** | 특정 컬럼의 고유 값을 새 컬럼으로 변환하여 크로스탭을 생성합니다 |
| **UNPIVOT** | 여러 컬럼을 하나의 값 컬럼과 카테고리 컬럼으로 합칩니다 |
| **집계 함수** | PIVOT에서는 SUM, COUNT, AVG 등 집계 함수가 필수입니다 |
| **NULL 처리** | UNPIVOT은 기본적으로 NULL을 제외합니다 (INCLUDE NULLS로 변경 가능) |
| **CASE WHEN 대안** | PIVOT 구문 대신 수동으로 CASE WHEN을 사용할 수도 있습니다 |

---

## 참고 링크

- [Databricks: PIVOT clause](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-pivot.html)
- [Databricks: UNPIVOT clause](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-unpivot.html)
- [Azure Databricks: PIVOT](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-syntax-qry-select-pivot)
