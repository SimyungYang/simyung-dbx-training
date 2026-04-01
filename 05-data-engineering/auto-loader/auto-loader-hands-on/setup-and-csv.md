# 사전 준비와 CSV 수집 실습

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
