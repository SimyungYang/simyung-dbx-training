# Auto Loader 실습

## 개요

이 실습에서는 Auto Loader를 사용하여 다양한 포맷의 데이터를 수집하는 파이프라인을 직접 구축해 봅니다. CSV, JSON 파일을 스트리밍으로 수집하고, 스키마 진화를 처리하며, SDP(선언적 파이프라인)와 통합하는 과정을 단계별로 진행합니다.

> 💡 **실습 목표**: Auto Loader의 핵심 기능(자동 수집, 스키마 추론, 스키마 진화, 에러 처리)을 직접 체험하고, 프로덕션에서 사용할 수 있는 패턴을 익힙니다.

---

## 사전 준비

### 1단계: 카탈로그 및 스키마 생성

```sql
-- Unity Catalog에 실습용 카탈로그와 스키마 생성
CREATE CATALOG IF NOT EXISTS training;
USE CATALOG training;

CREATE SCHEMA IF NOT EXISTS auto_loader_lab;
USE SCHEMA auto_loader_lab;
```

### 2단계: 볼륨 생성 및 샘플 데이터 업로드

```sql
-- 데이터 저장용 볼륨 생성
CREATE VOLUME IF NOT EXISTS training.auto_loader_lab.raw_data;
```

### 3단계: 샘플 데이터 생성 (Python)

```python
# 노트북에서 실행 - CSV 샘플 데이터 생성
import json
from datetime import datetime, timedelta
import random

# --- CSV 주문 데이터 생성 ---
csv_header = "order_id,customer_id,product_name,amount,order_date,status\n"
csv_rows = []
for i in range(1, 101):
    date = (datetime(2025, 3, 1) + timedelta(days=random.randint(0, 30))).strftime("%Y-%m-%d")
    status = random.choice(["COMPLETED", "PENDING", "CANCELLED"])
    product = random.choice(["노트북", "키보드", "마우스", "모니터", "헤드셋"])
    amount = round(random.uniform(10000, 500000), 2)
    csv_rows.append(f"{i},{random.randint(1000,9999)},{product},{amount},{date},{status}")

csv_content = csv_header + "\n".join(csv_rows)

# 볼륨에 CSV 파일 저장
dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/csv/orders/orders_batch1.csv",
    csv_content,
    overwrite=True
)

# --- JSON 고객 데이터 생성 ---
customers = []
for i in range(1000, 1050):
    customers.append(json.dumps({
        "customer_id": i,
        "name": f"고객_{i}",
        "email": f"customer{i}@example.com",
        "city": random.choice(["서울", "부산", "대구", "인천", "광주"]),
        "registered_at": f"2025-{random.randint(1,3):02d}-{random.randint(1,28):02d}T10:00:00Z"
    }, ensure_ascii=False))

json_content = "\n".join(customers)  # JSON Lines 형식

dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/json/customers/customers_batch1.json",
    json_content,
    overwrite=True
)

print("샘플 데이터 생성 완료!")
```

---

## 실습 1: CSV 파일 스트리밍 수집

### 목표
S3/볼륨에 도착하는 CSV 주문 데이터를 Auto Loader로 자동 수집하여 Bronze 테이블에 저장합니다.

### Step 1: Auto Loader로 CSV 읽기

```python
# Auto Loader로 CSV 파일 스트리밍 읽기
source_path = "/Volumes/training/auto_loader_lab/raw_data/csv/orders/"
checkpoint_path = "/Volumes/training/auto_loader_lab/checkpoints/csv_orders/"
schema_path = "/Volumes/training/auto_loader_lab/schema/csv_orders/"

df_orders = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    .option("header", "true")
    .option("delimiter", ",")
    .option("encoding", "UTF-8")
    # 에러 처리
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)

# 메타데이터 컬럼 추가
from pyspark.sql.functions import current_timestamp, col

df_with_metadata = (df_orders
    .withColumn("_source_file", col("_metadata.file_path"))
    .withColumn("_file_modified_at", col("_metadata.file_modification_time"))
    .withColumn("_ingested_at", current_timestamp())
)

# 스키마 확인
df_with_metadata.printSchema()
```

### Step 2: Bronze 테이블에 저장

```python
# Bronze 테이블에 스트리밍 쓰기
(df_with_metadata.writeStream
    .option("checkpointLocation", checkpoint_path)
    .option("mergeSchema", "true")
    .trigger(availableNow=True)    # 현재 파일만 처리 후 종료
    .toTable("training.auto_loader_lab.bronze_orders")
)
```

### Step 3: 결과 확인

```sql
-- 수집된 데이터 확인
SELECT * FROM training.auto_loader_lab.bronze_orders LIMIT 10;

-- 수집 통계
SELECT
    COUNT(*) AS total_rows,
    COUNT(DISTINCT _source_file) AS source_files,
    MIN(_ingested_at) AS first_ingested,
    MAX(_ingested_at) AS last_ingested,
    SUM(CASE WHEN _rescued_data IS NOT NULL THEN 1 ELSE 0 END) AS rescued_rows
FROM training.auto_loader_lab.bronze_orders;
```

### Step 4: 새 파일 추가 및 증분 처리 확인

```python
# 두 번째 배치 CSV 생성
csv_rows_2 = []
for i in range(101, 201):
    date = (datetime(2025, 3, 15) + timedelta(days=random.randint(0, 15))).strftime("%Y-%m-%d")
    status = random.choice(["COMPLETED", "PENDING", "CANCELLED"])
    product = random.choice(["노트북", "키보드", "마우스", "모니터", "헤드셋"])
    amount = round(random.uniform(10000, 500000), 2)
    csv_rows_2.append(f"{i},{random.randint(1000,9999)},{product},{amount},{date},{status}")

csv_content_2 = csv_header + "\n".join(csv_rows_2)

dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/csv/orders/orders_batch2.csv",
    csv_content_2,
    overwrite=True
)

print("두 번째 배치 파일 생성 완료!")
```

```python
# 동일한 스트리밍 쿼리를 다시 실행하면 새 파일만 처리됩니다
(df_with_metadata.writeStream
    .option("checkpointLocation", checkpoint_path)
    .option("mergeSchema", "true")
    .trigger(availableNow=True)
    .toTable("training.auto_loader_lab.bronze_orders")
)
```

```sql
-- 증분 처리 확인: 행 수가 200으로 증가했는지 확인
SELECT
    COUNT(*) AS total_rows,
    COUNT(DISTINCT _source_file) AS source_files
FROM training.auto_loader_lab.bronze_orders;
-- 예상 결과: total_rows = 200, source_files = 2
```

> 💡 **핵심 포인트**: Auto Loader는 체크포인트를 통해 이미 처리한 파일을 기억합니다. 스트림을 다시 시작해도 기존 파일은 건너뛰고, 새로 도착한 `orders_batch2.csv`만 처리합니다.

---

## 실습 2: JSON 파일 + 스키마 진화

### 목표
JSON 고객 데이터를 수집하면서, 소스 스키마가 변경되었을 때 Auto Loader가 자동으로 대응하는 것을 확인합니다.

### Step 1: 초기 JSON 수집

```python
source_path = "/Volumes/training/auto_loader_lab/raw_data/json/customers/"
checkpoint_path = "/Volumes/training/auto_loader_lab/checkpoints/json_customers/"
schema_path = "/Volumes/training/auto_loader_lab/schema/json_customers/"

df_customers = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
    .withColumn("_source_file", col("_metadata.file_path"))
    .withColumn("_ingested_at", current_timestamp())
)

# Bronze 테이블에 저장
(df_customers.writeStream
    .option("checkpointLocation", checkpoint_path)
    .option("mergeSchema", "true")
    .trigger(availableNow=True)
    .toTable("training.auto_loader_lab.bronze_customers")
)
```

```sql
-- 초기 데이터 확인
SELECT * FROM training.auto_loader_lab.bronze_customers LIMIT 5;

-- 스키마 확인
DESCRIBE training.auto_loader_lab.bronze_customers;
```

### Step 2: 스키마가 변경된 새 파일 생성

```python
# 새 컬럼(phone, membership_level)이 추가된 JSON 생성
customers_v2 = []
for i in range(1050, 1080):
    customers_v2.append(json.dumps({
        "customer_id": i,
        "name": f"고객_{i}",
        "email": f"customer{i}@example.com",
        "city": random.choice(["서울", "부산", "대구", "인천", "광주"]),
        "registered_at": f"2025-03-{random.randint(1,28):02d}T10:00:00Z",
        # 새로 추가된 컬럼
        "phone": f"010-{random.randint(1000,9999)}-{random.randint(1000,9999)}",
        "membership_level": random.choice(["BRONZE", "SILVER", "GOLD", "PLATINUM"])
    }, ensure_ascii=False))

json_content_v2 = "\n".join(customers_v2)

dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/json/customers/customers_batch2.json",
    json_content_v2,
    overwrite=True
)

print("스키마 변경된 두 번째 배치 생성 완료!")
```

### Step 3: 스키마 진화 확인

```python
# 동일한 스트리밍 쿼리 재실행
(df_customers.writeStream
    .option("checkpointLocation", checkpoint_path)
    .option("mergeSchema", "true")
    .trigger(availableNow=True)
    .toTable("training.auto_loader_lab.bronze_customers")
)
```

```sql
-- 스키마 진화 확인: phone, membership_level 컬럼이 추가되었는지 확인
DESCRIBE training.auto_loader_lab.bronze_customers;

-- 데이터 확인: 기존 레코드는 새 컬럼이 NULL
SELECT
    customer_id, name, phone, membership_level
FROM training.auto_loader_lab.bronze_customers
ORDER BY customer_id
LIMIT 10;

-- 스키마 진화 통계
SELECT
    CASE WHEN phone IS NOT NULL THEN 'v2 (with phone)' ELSE 'v1 (original)' END AS schema_version,
    COUNT(*) AS row_count
FROM training.auto_loader_lab.bronze_customers
GROUP BY 1;
```

> 💡 **스키마 진화 결과**: `schemaEvolutionMode=addNewColumns`와 `mergeSchema=true` 설정 덕분에, 새 컬럼(`phone`, `membership_level`)이 자동으로 테이블에 추가되었습니다. 기존 레코드는 해당 컬럼이 `NULL`로 표시됩니다.

---

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

1. Workspace에서 **Pipelines** → **Create Pipeline** 클릭
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

## 데이터 검증

각 실습 후 데이터가 올바르게 수집되었는지 검증하는 단계입니다.

### 검증 체크리스트

| 검증 항목 | 확인 방법 | 예상 결과 |
|-----------|-----------|-----------|
| **행 수 정합** | `COUNT(*)`로 원본 파일과 테이블 비교 | 파일의 행 수와 테이블 행 수 일치 |
| **중복 없음** | `GROUP BY`로 기본 키 중복 확인 | 중복 레코드 없음 |
| **NULL 처리** | 필수 컬럼 NULL 비율 확인 | 허용 범위 내 |
| **_rescued_data** | 파싱 실패 건 확인 | 0 또는 허용 범위 내 |
| **스키마 일치** | `DESCRIBE`로 컬럼 타입 확인 | 예상 스키마와 일치 |

```sql
-- 종합 데이터 품질 리포트
SELECT
    'bronze_orders' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT order_id) AS unique_keys,
    SUM(CASE WHEN _rescued_data IS NOT NULL THEN 1 ELSE 0 END) AS rescued_rows,
    SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_key_rows,
    MIN(_ingested_at) AS first_ingestion,
    MAX(_ingested_at) AS last_ingestion
FROM training.auto_loader_lab.bronze_orders

UNION ALL

SELECT
    'bronze_customers' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT customer_id) AS unique_keys,
    SUM(CASE WHEN _rescued_data IS NOT NULL THEN 1 ELSE 0 END) AS rescued_rows,
    SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_key_rows,
    MIN(_ingested_at) AS first_ingestion,
    MAX(_ingested_at) AS last_ingestion
FROM training.auto_loader_lab.bronze_customers;
```

---

## 트러블슈팅

### 체크포인트 리셋

체크포인트가 손상되었거나, 처음부터 다시 처리해야 할 때 사용합니다.

```python
# 체크포인트 삭제 (주의: 모든 파일을 처음부터 다시 처리합니다)
dbutils.fs.rm("/Volumes/training/auto_loader_lab/checkpoints/csv_orders/", recurse=True)

# 스키마 위치도 리셋 (필요한 경우)
dbutils.fs.rm("/Volumes/training/auto_loader_lab/schema/csv_orders/", recurse=True)

print("체크포인트 및 스키마 리셋 완료. 스트림을 다시 시작하면 모든 파일을 처리합니다.")
```

> ⚠️ **체크포인트 리셋 시 주의**: 체크포인트를 삭제하면 모든 파일을 처음부터 다시 처리합니다. 대상 테이블에 `MERGE` 대신 `APPEND`를 사용하고 있다면 데이터 중복이 발생합니다. 리셋 전에 대상 테이블도 함께 초기화하세요.

```python
# 테이블과 체크포인트를 함께 초기화하는 안전한 방법
spark.sql("DROP TABLE IF EXISTS training.auto_loader_lab.bronze_orders")
dbutils.fs.rm("/Volumes/training/auto_loader_lab/checkpoints/csv_orders/", recurse=True)
dbutils.fs.rm("/Volumes/training/auto_loader_lab/schema/csv_orders/", recurse=True)
print("테이블, 체크포인트, 스키마 모두 초기화 완료.")
```

### 스키마 충돌 해결

스키마 진화 과정에서 충돌이 발생할 수 있습니다.

| 문제 | 원인 | 해결 방법 |
|------|------|-----------|
| **UnknownFieldException** | `schemaEvolutionMode=failOnNewColumns`에서 새 컬럼 감지 | 스키마를 수동으로 업데이트하거나, `addNewColumns` 모드로 변경 |
| **타입 불일치** | 같은 컬럼이 다른 타입으로 도착 (예: STRING → INT) | `schemaHints`로 타입을 명시하거나, `rescuedDataColumn`으로 처리 |
| **스키마 덮어쓰기 필요** | 스키마 위치의 기존 스키마가 잘못된 경우 | `cloudFiles.allowOverwrites=true` 설정 후 스키마 위치 리셋 |

```python
# 스키마 충돌 해결 예제: 타입 힌트로 강제 지정
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    # 문제가 되는 컬럼의 타입을 명시적으로 지정
    .option("cloudFiles.schemaHints",
            "order_id BIGINT, amount DOUBLE, zip_code STRING")
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
```

### 일반적인 에러와 해결 방법

| 에러 메시지 | 원인 | 해결 방법 |
|-------------|------|-----------|
| `PERMISSION_DENIED` | 볼륨/S3 경로 접근 권한 부족 | Unity Catalog 권한 또는 IAM 역할 확인 |
| `AnalysisException: path does not exist` | 소스 경로가 존재하지 않음 | 경로 오타 확인, 볼륨이 생성되었는지 확인 |
| `StreamingQueryException: checkpoint` | 체크포인트 손상 또는 호환 불가 | 체크포인트 리셋 후 재시작 |
| `DELTA_SCHEMA_CHANGE_SINCE_ANALYSIS` | 스트리밍 중 스키마 변경 | `mergeSchema=true` 옵션 추가 |
| `MALFORMED_RECORD_IN_PARSING` | 파싱할 수 없는 레코드 존재 | `rescuedDataColumn` 또는 `badRecordsPath` 설정 |

---

## 실습 정리 (리소스 삭제)

실습이 끝나면 생성한 리소스를 정리합니다.

```sql
-- 테이블 삭제
DROP TABLE IF EXISTS training.auto_loader_lab.bronze_orders;
DROP TABLE IF EXISTS training.auto_loader_lab.bronze_customers;
DROP TABLE IF EXISTS training.auto_loader_lab.silver_orders;
DROP TABLE IF EXISTS training.auto_loader_lab.silver_customers;
DROP VIEW IF EXISTS training.auto_loader_lab.gold_daily_sales;
DROP VIEW IF EXISTS training.auto_loader_lab.gold_customer_by_city;

-- 볼륨 삭제 (데이터 포함)
DROP VOLUME IF EXISTS training.auto_loader_lab.raw_data;

-- 스키마 삭제
DROP SCHEMA IF EXISTS training.auto_loader_lab CASCADE;
```

```python
# 체크포인트 정리
dbutils.fs.rm("/Volumes/training/auto_loader_lab/", recurse=True)
print("모든 실습 리소스 정리 완료!")
```

---

## 정리

| 실습 | 학습 포인트 |
|------|-------------|
| **실습 1: CSV 수집** | Auto Loader 기본 사용법, 체크포인트 기반 증분 처리 |
| **실습 2: JSON + 스키마 진화** | `schemaEvolutionMode=addNewColumns`로 자동 스키마 확장 |
| **실습 3: SDP 통합** | `read_files()` 함수로 Medallion 파이프라인 구축 |
| **데이터 검증** | `_rescued_data`, 행 수 정합, NULL 비율 확인 |
| **트러블슈팅** | 체크포인트 리셋, 스키마 충돌 해결, 일반 에러 대응 |

---

## 참고 링크

- [Databricks: Auto Loader tutorial](https://docs.databricks.com/aws/en/ingestion/auto-loader/tutorial.html)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html)
- [Databricks: Auto Loader schema inference](https://docs.databricks.com/aws/en/ingestion/auto-loader/schema.html)
- [Databricks: read_files in SDP](https://docs.databricks.com/aws/en/ingestion/auto-loader/unity-catalog.html)
- [Databricks: Streaming tables](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-streaming-table.html)
