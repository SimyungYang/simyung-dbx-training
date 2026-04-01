# 고급 패턴과 성능 최적화

## 현업 사례: 월별 매출 크로스탭을 매번 수동으로 만들던 시절

> 🔥 **이것은 거의 모든 데이터 분석 조직에서 겪는 비효율입니다.**

현업에서는 "월별 매출을 제품별로 보고 싶어요"라는 요청이 매주, 때로는 매일 옵니다. PIVOT을 모르면 어떤 일이 벌어지는지 살펴보겠습니다.

### PIVOT 없이 리포트 만드는 고통

```sql
-- 😫 PIVOT을 모를 때의 접근법: CASE WHEN을 수동으로 나열
-- 제품이 10개면 CASE WHEN을 10개 써야 합니다
-- 제품이 100개면? 포기하고 Excel로 갑니다

SELECT
    DATE_FORMAT(order_date, 'yyyy-MM') AS month,
    SUM(CASE WHEN product_name = 'Laptop' THEN amount END) AS laptop,
    SUM(CASE WHEN product_name = 'Mouse' THEN amount END) AS mouse,
    SUM(CASE WHEN product_name = 'Keyboard' THEN amount END) AS keyboard,
    SUM(CASE WHEN product_name = 'Monitor' THEN amount END) AS monitor,
    SUM(CASE WHEN product_name = 'Headset' THEN amount END) AS headset,
    SUM(CASE WHEN product_name = 'Webcam' THEN amount END) AS webcam,
    SUM(CASE WHEN product_name = 'Charger' THEN amount END) AS charger,
    -- ... 제품이 추가될 때마다 이 쿼리를 수정해야 합니다
    SUM(amount) AS total
FROM orders
GROUP BY 1
ORDER BY month;

-- 매번 이런 쿼리를 수동으로 작성하면:
-- - 제품 추가 시 쿼리 수정 필요 (운영 부담)
-- - 오타로 인한 잘못된 집계 (품질 리스크)
-- - 쿼리 길이가 수백 줄이 되면 유지보수 불가
```

> 💡 PIVOT을 사용하면 위 코드가 10줄 이내로 줄어듭니다. 하지만 **PIVOT에도 한계가 있습니다.**

---

## 동적 PIVOT의 한계와 우회 방법

### SQL PIVOT의 가장 큰 제약: 값을 미리 알아야 합니다

Databricks SQL의 `PIVOT`은 `IN` 절에 피벗할 값을 ** 명시적으로 나열** 해야 합니다. 즉, 제품명이 뭐가 있는지 미리 알아야 합니다. 새 제품이 추가되면 쿼리를 수정해야 합니다.

```sql
-- 이것은 동작하지 않습니다! (동적 서브쿼리 불가)
-- ❌ 에러 발생
SELECT *
FROM orders
PIVOT (
    SUM(amount)
    FOR product_name IN (SELECT DISTINCT product_name FROM orders)
    -- → "Subquery not supported in PIVOT IN clause" 에러
);
```

### 우회 방법 1: 2단계 쿼리 (가장 실용적)

```python
# Python에서 동적으로 PIVOT 쿼리를 생성하는 패턴
# 현업에서 가장 많이 사용하는 방법입니다

# 1단계: 피벗할 값 목록을 동적으로 가져옴
products = spark.sql("""
    SELECT DISTINCT product_name
    FROM orders
    WHERE order_date >= '2025-01-01'
    ORDER BY product_name
""").collect()

# 2단계: 동적 PIVOT 쿼리 생성
product_list = ", ".join([
    f"'{row.product_name}' AS {row.product_name.lower().replace(' ', '_')}"
    for row in products
])

pivot_query = f"""
SELECT *
FROM (
    SELECT DATE_FORMAT(order_date, 'yyyy-MM') AS month, product_name, amount
    FROM orders
    WHERE order_date >= '2025-01-01'
)
PIVOT (
    SUM(amount) FOR product_name IN ({product_list})
)
ORDER BY month
"""

result = spark.sql(pivot_query)
display(result)
```

### 우회 방법 2: 임시 뷰 + 매개변수화

```sql
-- 위젯이나 파라미터를 사용하여 반자동화하는 패턴
-- Databricks SQL 대시보드에서 유용합니다

-- 먼저 카테고리 목록을 확인
SELECT DISTINCT category FROM products ORDER BY category;

-- 확인된 카테고리로 PIVOT 실행
SELECT *
FROM (
    SELECT
        region,
        category,
        amount
    FROM orders o JOIN products p ON o.product_id = p.id
)
PIVOT (
    SUM(amount)
    FOR category IN (
        'Electronics' AS electronics,
        'Clothing' AS clothing,
        'Food' AS food,
        'Furniture' AS furniture
    )
);
```

> 💡 **현업 팁**: 대시보드에서 크로스탭이 필요할 때는 PIVOT 쿼리를 쓰는 것보다 **Lakeview의 피벗 테이블 시각화** 를 사용하는 것이 훨씬 편합니다. SQL PIVOT은 ETL 파이프라인에서 데이터를 넓은(wide) 형태로 변환할 때 주로 사용합니다.

---

## 실전 리포트 생성 패턴

### 패턴 1: YoY(전년 대비) 비교 크로스탭

```sql
-- 현업에서 매우 자주 요청받는 리포트: "올해 vs 작년 월별 매출 비교"
SELECT
    month_name,
    FORMAT_NUMBER(y2024, 0) AS `2024`,
    FORMAT_NUMBER(y2025, 0) AS `2025`,
    CONCAT(
        CASE
            WHEN y2024 IS NULL OR y2024 = 0 THEN 'N/A'
            ELSE CONCAT(
                ROUND((y2025 - y2024) / y2024 * 100, 1), '%'
            )
        END
    ) AS yoy_growth
FROM (
    SELECT
        MONTH(order_date) AS month_num,
        DATE_FORMAT(order_date, 'MMM') AS month_name,
        YEAR(order_date) AS yr,
        SUM(amount) AS total
    FROM orders
    WHERE YEAR(order_date) IN (2024, 2025)
    GROUP BY 1, 2, 3
)
PIVOT (
    SUM(total)
    FOR yr IN (2024 AS y2024, 2025 AS y2025)
)
ORDER BY month_num;
```

결과 예시:

| month_name | 2024 | 2025 | yoy_growth |
|-----------|------|------|------------|
| Jan | 1,200,000 | 1,500,000 | 25.0% |
| Feb | 1,100,000 | 1,400,000 | 27.3% |
| Mar | 1,300,000 | 1,250,000 | -3.8% |

### 패턴 2: EAV(Entity-Attribute-Value) 데이터 정규화 (UNPIVOT)

```sql
-- 현업에서 자주 만나는 패턴: 설정 테이블이나 속성 테이블이
-- 넓은 형태로 되어 있어서 정규화가 필요한 경우

-- 원본: 고객 속성이 컬럼으로 펼쳐짐 (100개 이상의 컬럼)
-- | customer_id | attr_age | attr_gender | attr_city | attr_tier | ...
-- 이것을 EAV 형태로 변환하면 유연한 분석이 가능합니다

SELECT customer_id, attribute, value
FROM customer_profiles
UNPIVOT (
    value FOR attribute IN (
        attr_age AS 'age',
        attr_gender AS 'gender',
        attr_city AS 'city',
        attr_tier AS 'tier',
        attr_signup_channel AS 'signup_channel'
    )
)
ORDER BY customer_id, attribute;

-- 결과: 세로로 길어지지만, 속성이 추가되어도 스키마가 안 변합니다
-- | customer_id | attribute      | value    |
-- |-------------|----------------|----------|
-- | C001        | age            | 32       |
-- | C001        | city           | 서울     |
-- | C001        | gender         | M        |
-- | C001        | signup_channel | mobile   |
-- | C001        | tier           | Gold     |
```

### 패턴 3: 멀티 메트릭 대시보드용 PIVOT

```sql
-- "지역별로 매출, 주문수, 평균 단가를 한 표로 보여주세요"
-- 여러 집계 함수를 동시에 PIVOT하는 실전 패턴

SELECT
    region,
    FORMAT_NUMBER(Electronics_revenue, 0) AS elec_revenue,
    Electronics_orders AS elec_orders,
    FORMAT_NUMBER(Electronics_avg_price, 0) AS elec_avg,
    FORMAT_NUMBER(Clothing_revenue, 0) AS cloth_revenue,
    Clothing_orders AS cloth_orders,
    FORMAT_NUMBER(Clothing_avg_price, 0) AS cloth_avg
FROM (
    SELECT region, category, amount
    FROM orders o JOIN products p ON o.product_id = p.id
)
PIVOT (
    SUM(amount) AS revenue,
    COUNT(*) AS orders,
    ROUND(AVG(amount), 0) AS avg_price
    FOR category IN ('Electronics', 'Clothing')
);
```

> 💡 **현업에서 가장 중요한 팁**: PIVOT/UNPIVOT은 ** 도구** 일 뿐입니다. 최종 사용자에게 보여줄 리포트라면 Lakeview 대시보드의 피벗 테이블 시각화가 훨씬 빠르고 유연합니다. SQL PIVOT은 ** 데이터 파이프라인에서 형태를 변환** 하거나, ** 다른 시스템에 넓은 형태로 내보낼 때** 진가를 발휘합니다.

---

## PIVOT 성능 최적화 팁

현업에서 대용량 데이터에 PIVOT을 적용할 때 알아두면 좋은 성능 팁입니다.

| 상황 | 문제 | 해결 방법 |
|------|------|----------|
| 피벗 대상 값이 1,000개 이상 | 컬럼이 너무 많아 메모리 부족 | 상위 N개만 필터링 후 PIVOT |
| 원본 테이블이 10억 행 이상 | PIVOT이 매우 느림 | 먼저 GROUP BY로 집계한 결과에 PIVOT |
| 매일 돌아가는 리포트 | 매번 전체 데이터를 PIVOT | 집계 테이블(Gold Table)을 만들고 증분 업데이트 |

```sql
-- ✅ 대용량 데이터에서의 올바른 PIVOT 순서:
-- 1. 먼저 집계 → 2. 그 다음 PIVOT (행 수를 줄인 후 피벗)

-- ❌ 느림: 원본 10억 행에 직접 PIVOT
SELECT * FROM billion_row_table PIVOT (...);

-- ✅ 빠름: 먼저 집계해서 행을 줄인 후 PIVOT
SELECT *
FROM (
    SELECT month, product, SUM(amount) AS total
    FROM billion_row_table
    GROUP BY month, product
)  -- 이 결과는 수천 행
PIVOT (
    SUM(total) FOR product IN ('A' AS a, 'B' AS b, 'C' AS c)
);
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **PIVOT**| 특정 컬럼의 고유 값을 새 컬럼으로 변환하여 크로스탭을 생성합니다 |
| **UNPIVOT**| 여러 컬럼을 하나의 값 컬럼과 카테고리 컬럼으로 합칩니다 |
| ** 집계 함수**| PIVOT에서는 SUM, COUNT, AVG 등 집계 함수가 필수입니다 |
| **NULL 처리**| UNPIVOT은 기본적으로 NULL을 제외합니다 (INCLUDE NULLS로 변경 가능) |
| **CASE WHEN 대안** | PIVOT 구문 대신 수동으로 CASE WHEN을 사용할 수도 있습니다 |

---

## 참고 링크

- [Databricks: PIVOT clause](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-pivot.html)
- [Databricks: UNPIVOT clause](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-unpivot.html)
- [Azure Databricks: PIVOT](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-syntax-qry-select-pivot)
