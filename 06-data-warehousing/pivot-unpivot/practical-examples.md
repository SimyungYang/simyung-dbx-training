# 실전 활용 사례

## 실전 활용 사례

### 1. 크로스탭 리포트 생성 (PIVOT)

```sql
-- 지역별-카테고리별 매출 크로스탭
SELECT
    COALESCE(region, '**전체 **') AS region,
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
| **변환 방향**| 행 → 열 | 열 → 행 |
| **결과 형태**| 넓은(Wide) 테이블 | 긴(Long) 테이블 |
| **집계 함수**| 필수 (SUM, AVG, COUNT 등) | 불필요 |
| **NULL 처리**| 값이 없으면 NULL | 기본적으로 NULL 행 제외 |
| **사용 사례** | 크로스탭 리포트, 대시보드 | 데이터 정규화, ETL 변환 |

---
