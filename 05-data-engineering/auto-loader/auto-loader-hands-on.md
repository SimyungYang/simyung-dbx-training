# Auto Loader 실습

## 시나리오

S3에 매시간 JSON 형태의 주문 데이터가 도착합니다. Auto Loader를 사용하여 이 데이터를 자동으로 수집하고, Bronze 테이블에 저장하는 파이프라인을 구축해 보겠습니다.

---

## 실습: SDP 기반 파이프라인

### Step 1: Bronze 테이블 (Auto Loader로 수집)

```sql
CREATE OR REFRESH STREAMING TABLE bronze_orders
COMMENT '원본 주문 데이터 (JSON)'
AS SELECT
    *,
    _metadata.file_path AS _source_file,
    _metadata.file_modification_time AS _file_modified_at,
    current_timestamp() AS _ingested_at
FROM STREAM read_files(
    's3://my-bucket/raw/orders/',
    format => 'json',
    inferColumnTypes => true,
    rescuedDataColumn => '_rescued_data'
);
```

### Step 2: Silver 테이블 (정제)

```sql
CREATE OR REFRESH STREAMING TABLE silver_orders (
    CONSTRAINT valid_id EXPECT (order_id IS NOT NULL) ON VIOLATION DROP ROW,
    CONSTRAINT valid_amount EXPECT (amount > 0) ON VIOLATION DROP ROW
)
COMMENT '정제된 주문 데이터'
AS SELECT
    CAST(order_id AS BIGINT) AS order_id,
    CAST(customer_id AS BIGINT) AS customer_id,
    TRIM(product_name) AS product_name,
    CAST(amount AS DECIMAL(12,2)) AS amount,
    CAST(order_date AS TIMESTAMP) AS order_date,
    UPPER(TRIM(status)) AS status
FROM STREAM(bronze_orders)
WHERE _rescued_data IS NULL;  -- 파싱 오류 없는 데이터만
```

### Step 3: Gold 테이블 (집계)

```sql
CREATE OR REFRESH MATERIALIZED VIEW gold_hourly_sales
COMMENT '시간별 매출 요약'
AS SELECT
    DATE_TRUNC('HOUR', order_date) AS sale_hour,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value
FROM silver_orders
WHERE status = 'COMPLETED'
GROUP BY DATE_TRUNC('HOUR', order_date);
```

---

## 파이프라인 생성 및 실행

1. Workspace에서 **Pipelines** → **Create Pipeline** 클릭
2. Pipeline 이름: `orders-ingestion-pipeline`
3. Source Code: 위 SQL이 포함된 노트북 경로 지정
4. Target Schema: `catalog.schema` 지정
5. **Start** 클릭

파이프라인이 시작되면 Auto Loader가 S3 디렉토리를 감시하기 시작합니다. 새 JSON 파일이 도착하면 자동으로 Bronze → Silver → Gold로 처리됩니다.

---

## 참고 링크

- [Databricks: Auto Loader tutorial](https://docs.databricks.com/aws/en/ingestion/auto-loader/tutorial.html)
