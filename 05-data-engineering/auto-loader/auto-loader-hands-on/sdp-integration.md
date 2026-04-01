# SDP와 Auto Loader 통합

## 실습 3: SDP와 Auto Loader 통합

### 목표
SDP(선언적 파이프라인)에서 Auto Loader를 사용하여 Bronze → Silver → Gold Medallion 파이프라인을 구축합니다.

### 파이프라인 구조

| 단계 | 계층 | 설명 |
|------|------|------|
| 1 | CSV/JSON 파일 | 원본 파일 소스입니다 |
| 2 | Bronze (원본 수집) | 원본 데이터를 그대로 수집합니다 |
| 3 | Silver (정제/검증) | 데이터를 정제하고 검증합니다 |
| 4 | Gold (비즈니스 집계) | 비즈니스 요구에 맞게 집계합니다 |

### SDP 노트북 코드

아래 SQL을 하나의 노트북에 작성합니다.

```sql
-- =====================================================
-- SDP 파이프라인: E-Commerce 주문 데이터 처리
-- =====================================================

-- [Bronze] Auto Loader로 CSV 주문 데이터 수집
CREATE OR REFRESH STREAMING TABLE bronze_orders
COMMENT '원본 주문 데이터 (CSV from Auto Loader)'
AS SELECT
    *,
    _metadata.file_path AS _source_file,
    _metadata.file_name AS _source_file_name,
    _metadata.file_modification_time AS _file_modified_at,
    current_timestamp() AS _ingested_at
FROM STREAM read_files(
    '/Volumes/training/auto_loader_lab/raw_data/csv/orders/',
    format => 'csv',
    header => true,
    inferColumnTypes => true,
    rescuedDataColumn => '_rescued_data'
);
```

```sql
-- [Bronze] Auto Loader로 JSON 고객 데이터 수집
CREATE OR REFRESH STREAMING TABLE bronze_customers
COMMENT '원본 고객 데이터 (JSON from Auto Loader)'
AS SELECT
    *,
    _metadata.file_path AS _source_file,
    _metadata.file_modification_time AS _file_modified_at,
    current_timestamp() AS _ingested_at
FROM STREAM read_files(
    '/Volumes/training/auto_loader_lab/raw_data/json/customers/',
    format => 'json',
    inferColumnTypes => true,
    rescuedDataColumn => '_rescued_data'
);
```

```sql
-- [Silver] 주문 데이터 정제 및 검증
CREATE OR REFRESH STREAMING TABLE silver_orders (
    -- 데이터 품질 규칙
    CONSTRAINT valid_order_id EXPECT (order_id IS NOT NULL) ON VIOLATION DROP ROW,
    CONSTRAINT valid_amount EXPECT (amount > 0) ON VIOLATION DROP ROW,
    CONSTRAINT valid_status EXPECT (status IN ('COMPLETED', 'PENDING', 'CANCELLED')) ON VIOLATION DROP ROW
)
COMMENT '정제된 주문 데이터'
AS SELECT
    CAST(order_id AS BIGINT) AS order_id,
    CAST(customer_id AS BIGINT) AS customer_id,
    TRIM(product_name) AS product_name,
    CAST(amount AS DECIMAL(12,2)) AS amount,
    CAST(order_date AS DATE) AS order_date,
    UPPER(TRIM(status)) AS status,
    _source_file,
    _ingested_at
FROM STREAM(bronze_orders)
WHERE _rescued_data IS NULL;   -- 파싱 오류가 없는 데이터만 통과
```

```sql
-- [Silver] 고객 데이터 정제
CREATE OR REFRESH STREAMING TABLE silver_customers (
    CONSTRAINT valid_customer_id EXPECT (customer_id IS NOT NULL) ON VIOLATION DROP ROW,
    CONSTRAINT valid_email EXPECT (email IS NOT NULL AND email LIKE '%@%') ON VIOLATION DROP ROW
)
COMMENT '정제된 고객 데이터'
AS SELECT
    CAST(customer_id AS BIGINT) AS customer_id,
    TRIM(name) AS name,
    LOWER(TRIM(email)) AS email,
    TRIM(city) AS city,
    CAST(registered_at AS TIMESTAMP) AS registered_at,
    _source_file,
    _ingested_at
FROM STREAM(bronze_customers)
WHERE _rescued_data IS NULL;
```

```sql
-- [Gold] 일별 매출 요약
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_sales
COMMENT '일별 매출 요약'
AS SELECT
    order_date,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(CASE WHEN status = 'COMPLETED' THEN amount ELSE 0 END) AS completed_revenue,
    SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_count
FROM silver_orders
GROUP BY order_date
ORDER BY order_date;
```

```sql
-- [Gold] 도시별 고객 통계
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_by_city
COMMENT '도시별 고객 및 매출 통계'
AS SELECT
    c.city,
    COUNT(DISTINCT c.customer_id) AS customer_count,
    COUNT(o.order_id) AS order_count,
    COALESCE(SUM(o.amount), 0) AS total_revenue,
    COALESCE(AVG(o.amount), 0) AS avg_order_value
FROM silver_customers c
LEFT JOIN silver_orders o
    ON c.customer_id = o.customer_id
GROUP BY c.city
ORDER BY total_revenue DESC;
```

### 파이프라인 생성 및 실행

1. Workspace에서 **Pipelines**→ **Create Pipeline** 클릭
2. Pipeline 이름: `auto-loader-lab-pipeline`
3. Source Code: 위 SQL이 포함된 노트북 경로 지정
4. Target Catalog: `training`
5. Target Schema: `auto_loader_lab`
6. **Start** 클릭

### 결과 검증

```sql
-- Gold 일별 매출 확인
SELECT * FROM training.auto_loader_lab.gold_daily_sales
ORDER BY order_date DESC;

-- Gold 도시별 고객 확인
SELECT * FROM training.auto_loader_lab.gold_customer_by_city;

-- 데이터 품질 확인: 드롭된 행 수는 SDP UI의 Data Quality 탭에서 확인
```

---
