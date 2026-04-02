# 실습 1: 사전 준비와 CSV 수집

## 목적과 학습 목표

이 실습은 Auto Loader (자동 로더) 전체 시리즈의 첫 번째 단계입니다. 클라우드 스토리지에 도착하는 파일을 **자동으로 감지하고 증분 수집** 하는 Auto Loader의 핵심 메커니즘을 직접 체험합니다.

### 학습 목표

| 목표 | 설명 |
|------|------|
| **환경 구성** | Unity Catalog (유니티 카탈로그), Volume (볼륨), 샘플 데이터를 준비합니다 |
| **기본 파이프라인** | `cloudFiles` 포맷으로 CSV를 스트리밍 수집하고 Delta 테이블에 저장합니다 |
| **증분 처리 확인** | 새 파일 추가 시 기존 파일은 건너뛰고 신규 파일만 처리되는 것을 확인합니다 |
| **Checkpoint 이해** | Checkpoint (체크포인트) 가 어떻게 이미 처리한 파일을 추적하는지 이해합니다 |
| **메타데이터 활용** | `_metadata` 컬럼으로 파일 출처를 Bronze 테이블에 기록합니다 |

> **Auto Loader의 핵심 가치**: 일반 `spark.read`는 매번 모든 파일을 다시 읽습니다. Auto Loader는 체크포인트에 처리 상태를 기록하여 **새로 도착한 파일만 처리** 합니다. 파일이 수십억 개로 늘어나도 성능이 일정하게 유지됩니다.

---

## 사전 준비

### 필요 환경

| 항목 | 요구 사항 |
|------|----------|
| **Databricks Runtime** | DBR 12.2 LTS 이상 (Auto Loader는 DBR 8.x부터 GA) |
| **Unity Catalog** | 활성화된 UC 환경 (Metastore가 Workspace에 연결되어 있어야 함) |
| **권한** | `CREATE CATALOG`, `CREATE SCHEMA`, `CREATE VOLUME`, `CREATE TABLE` 권한 |
| **클러스터** | Single Node 또는 Multi Node (본 실습은 Single Node로 충분) |

### 주의 사항

- 실습 전 클러스터가 **Unity Catalog 모드** 로 실행되어야 합니다 (Access Mode: Single User 또는 Shared).
- DBFS (`/dbfs/`) 대신 **UC Volume** (`/Volumes/`) 경로를 사용합니다. 이것이 현재 Databricks 권장 방식입니다.
- `availableNow=True` Trigger (트리거) 를 사용하므로 스트림은 처리 완료 후 자동 종료됩니다.

---

### 1단계: 카탈로그 및 스키마 생성

```sql
-- Unity Catalog에 실습용 카탈로그와 스키마 생성
-- 이미 존재하는 카탈로그가 있다면 해당 카탈로그를 사용해도 됩니다
CREATE CATALOG IF NOT EXISTS training;
USE CATALOG training;

CREATE SCHEMA IF NOT EXISTS auto_loader_lab;
USE SCHEMA auto_loader_lab;

-- 생성 결과 확인
SHOW SCHEMAS IN training;
```

**예상 결과**: `auto_loader_lab` 스키마가 목록에 나타납니다.

---

### 2단계: Volume 생성

Volume (볼륨) 은 Unity Catalog가 관리하는 파일 스토리지 경로입니다. Auto Loader의 소스 파일과 체크포인트를 모두 Volume에 저장합니다.

```sql
-- 데이터 저장용 볼륨 생성
CREATE VOLUME IF NOT EXISTS training.auto_loader_lab.raw_data;

-- 볼륨 경로 확인
-- 실제 경로: /Volumes/training/auto_loader_lab/raw_data/
SHOW VOLUMES IN training.auto_loader_lab;
```

> **DBFS vs UC Volume**: DBFS (`/dbfs/user/hive/...`) 는 레거시 경로입니다. 신규 워크로드는 반드시 **UC Volume** 을 사용하세요. Unity Catalog의 접근 제어(ACL), 감사 로그, 데이터 리니지가 Volume을 통해 작동합니다.

---

### 3단계: 샘플 데이터 생성

```python
# 노트북 셀에서 실행 - CSV 주문 데이터 생성
import json
from datetime import datetime, timedelta
import random

random.seed(42)  # 재현성을 위해 시드 고정

# --- CSV 주문 데이터 생성 (Batch 1: 100건) ---
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

print(f"CSV 배치 1 생성 완료: {len(csv_rows)}건")
print("경로: /Volumes/training/auto_loader_lab/raw_data/csv/orders/orders_batch1.csv")
```

**예상 출력**:
```
CSV 배치 1 생성 완료: 100건
경로: /Volumes/training/auto_loader_lab/raw_data/csv/orders/orders_batch1.csv
```

---

## 실습 1: CSV 파일 스트리밍 수집

### Step 1: Auto Loader로 CSV 읽기

Auto Loader는 Spark Structured Streaming (구조적 스트리밍) 의 `cloudFiles` 소스 포맷으로 구현되어 있습니다.

```python
from pyspark.sql.functions import current_timestamp, col

# 경로 설정
source_path      = "/Volumes/training/auto_loader_lab/raw_data/csv/orders/"
checkpoint_path  = "/Volumes/training/auto_loader_lab/checkpoints/csv_orders/"
schema_path      = "/Volumes/training/auto_loader_lab/schema/csv_orders/"

# Auto Loader로 CSV 파일 스트리밍 읽기
df_orders = (
    spark.readStream
    .format("cloudFiles")                               # Auto Loader 포맷
    .option("cloudFiles.format", "csv")                 # 소스 파일 포맷
    .option("cloudFiles.schemaLocation", schema_path)   # 추론된 스키마 저장 위치
    .option("cloudFiles.inferColumnTypes", "true")      # 컬럼 타입 자동 추론 (기본값: 모두 STRING)
    .option("header", "true")                           # CSV 첫 줄을 헤더로 사용
    .option("delimiter", ",")                           # 구분자
    .option("encoding", "UTF-8")                        # 인코딩
    .option("rescuedDataColumn", "_rescued_data")       # 파싱 실패 데이터 보존
    .load(source_path)
)

# _metadata 컬럼으로 파일 출처 정보 추가 (데이터 리니지 추적)
df_with_metadata = (
    df_orders
    .withColumn("_source_file",      col("_metadata.file_path"))
    .withColumn("_file_modified_at", col("_metadata.file_modification_time"))
    .withColumn("_ingested_at",      current_timestamp())
)

# 스키마 확인 (스트리밍이므로 실제 데이터를 읽지 않고 스키마만 출력)
df_with_metadata.printSchema()
```

**예상 출력** (스키마):
```
root
 |-- order_id: long (nullable = true)
 |-- customer_id: long (nullable = true)
 |-- product_name: string (nullable = true)
 |-- amount: double (nullable = true)
 |-- order_date: date (nullable = true)
 |-- status: string (nullable = true)
 |-- _rescued_data: string (nullable = true)
 |-- _source_file: string (nullable = true)
 |-- _file_modified_at: timestamp (nullable = true)
 |-- _ingested_at: timestamp (nullable = true)
```

> **`inferColumnTypes=true` 효과**: 이 옵션이 없으면 모든 컬럼이 `STRING` 타입으로 수집됩니다. `true`로 설정하면 `amount`는 `double`, `order_date`는 `date`로 자동 추론됩니다. 단, 처음 수집 시 일부 파일을 샘플링하여 추론하므로 핵심 컬럼은 `schemaHints`로 명시적으로 지정하는 것을 권장합니다.

---

### Step 2: Bronze 테이블에 저장

```python
# Bronze 테이블에 스트리밍 쓰기
# trigger(availableNow=True): 현재 도착한 파일을 모두 처리하고 스트림 종료
query = (
    df_with_metadata
    .writeStream
    .option("checkpointLocation", checkpoint_path)
    .option("mergeSchema", "true")         # 스키마 변경 시 자동으로 테이블 스키마 확장
    .trigger(availableNow=True)            # 배치 스타일 실행 (처리 후 자동 종료)
    .toTable("training.auto_loader_lab.bronze_orders")
)

# 스트림 완료까지 대기
query.awaitTermination()
print("수집 완료!")
```

**`trigger` 옵션 비교**:

| Trigger | 동작 | 사용 시나리오 |
|---------|------|--------------|
| `availableNow=True` | 현재 파일 모두 처리 후 종료 | 스케줄 배치 (Databricks Jobs) |
| `processingTime='30 seconds'` | 30초마다 신규 파일 처리 | Near Real-Time 수집 |
| 없음 (Continuous) | 신규 파일 즉시 처리, 계속 실행 | Real-Time 수집 (상시 클러스터) |

---

### Step 3: 결과 확인

```sql
-- 수집된 데이터 확인
SELECT * FROM training.auto_loader_lab.bronze_orders LIMIT 5;
```

```sql
-- 수집 통계 - 핵심 품질 지표
SELECT
    COUNT(*)                                                   AS total_rows,
    COUNT(DISTINCT _source_file)                               AS source_files,
    MIN(_ingested_at)                                          AS first_ingested,
    MAX(_ingested_at)                                          AS last_ingested,
    SUM(CASE WHEN _rescued_data IS NOT NULL THEN 1 ELSE 0 END) AS rescued_rows,
    SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END)          AS null_order_id
FROM training.auto_loader_lab.bronze_orders;
```

**예상 결과**:
```
total_rows | source_files | rescued_rows | null_order_id
-----------+--------------+--------------+--------------
100        | 1            | 0            | 0
```

> **`_rescued_data` 모니터링**: `rescued_rows`가 0이면 모든 데이터가 정상적으로 파싱되었습니다. 0보다 크면 해당 행을 조회하여 원인을 파악해야 합니다: `SELECT _rescued_data FROM bronze_orders WHERE _rescued_data IS NOT NULL LIMIT 5`

---

### Step 4: 새 파일 추가 및 증분 처리 확인

Auto Loader의 핵심인 **증분 처리** 를 직접 확인합니다.

```python
# 두 번째 배치 CSV 생성 (order_id 101~200)
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

print(f"배치 2 생성 완료: {len(csv_rows_2)}건")
```

```python
# 동일한 스트리밍 쿼리를 다시 실행
# → Auto Loader는 체크포인트를 확인하여 batch1은 건너뛰고 batch2만 처리합니다
query2 = (
    df_with_metadata
    .writeStream
    .option("checkpointLocation", checkpoint_path)  # 같은 체크포인트 경로!
    .option("mergeSchema", "true")
    .trigger(availableNow=True)
    .toTable("training.auto_loader_lab.bronze_orders")
)

query2.awaitTermination()
print("증분 처리 완료!")
```

```sql
-- 증분 처리 검증: 행 수가 200으로 늘었는지 확인
SELECT
    COUNT(*)                    AS total_rows,
    COUNT(DISTINCT _source_file) AS source_files,
    MIN(order_id)               AS min_order_id,
    MAX(order_id)               AS max_order_id
FROM training.auto_loader_lab.bronze_orders;
-- 예상 결과: total_rows=200, source_files=2, min=1, max=200
```

---

## 심화 학습

### 변형 시나리오 1: 잘못된 형식의 행 처리

실제 환경에서는 일부 행이 스키마에 맞지 않을 수 있습니다.

```python
# 의도적으로 잘못된 형식의 CSV 생성
bad_csv = """order_id,customer_id,product_name,amount,order_date,status
201,5001,모니터,150000,2025-04-01,COMPLETED
202,INVALID_ID,키보드,NOT_A_NUMBER,INVALID_DATE,UNKNOWN_STATUS
203,5003,마우스,25000,2025-04-01,COMPLETED
"""

dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/csv/orders/orders_batch_bad.csv",
    bad_csv,
    overwrite=True
)

# 스트림 재실행 후 rescued_data 확인
# 타입 불일치 행은 _rescued_data에 원본 JSON으로 보존됩니다
```

```sql
-- 구조적으로 잘못된 행 조회
SELECT order_id, customer_id, amount, _rescued_data
FROM training.auto_loader_lab.bronze_orders
WHERE _rescued_data IS NOT NULL;
```

### 변형 시나리오 2: Schema Hint (스키마 힌트) 로 타입 강제 지정

Auto Loader가 `amount`를 `STRING`으로 잘못 추론하는 경우 힌트로 교정합니다.

```python
df_with_hints = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    # 핵심 컬럼 타입을 명시적으로 고정
    .option("cloudFiles.schemaHints",
            "order_id BIGINT, customer_id BIGINT, amount DECIMAL(12,2), order_date DATE")
    .option("header", "true")
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
```

### 변형 시나리오 3: 특정 파일 패턴만 수집

```python
# 파일명 패턴으로 필터링 (예: orders_*.csv 만 수집)
df_filtered = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("pathGlobFilter", "orders_*.csv")    # glob 패턴
    .option("header", "true")
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
```

### 성능 튜닝 포인트

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `cloudFiles.maxFilesPerTrigger` | 1000 | 트리거당 처리할 최대 파일 수입니다. 파일이 크면 줄이고 작으면 늘립니다 |
| `cloudFiles.maxBytesPerTrigger` | 없음 | 트리거당 최대 데이터 크기 (`10g` 형식). `maxFilesPerTrigger`와 함께 사용합니다 |
| `cloudFiles.useIncrementalListing` | `auto` | 시간순 파티션 디렉토리 (`/year=2025/month=03/`)에서 `true`로 설정하면 스캔 범위를 줄여줍니다 |

---

## 정리

### 핵심 요약

| 개념 | 설명 |
|------|------|
| **`cloudFiles` 포맷** | Auto Loader의 Spark 소스 포맷. `spark.readStream.format("cloudFiles")`로 사용합니다 |
| **Checkpoint (체크포인트)** | 처리 완료한 파일 목록을 기록. 동일 체크포인트로 재실행하면 새 파일만 처리합니다 |
| **`inferColumnTypes=true`** | 파일을 샘플링하여 컬럼 타입을 자동 추론합니다. `schemaHints`로 특정 컬럼을 덮어쓸 수 있습니다 |
| **`rescuedDataColumn`** | 스키마 불일치 데이터를 유실 없이 보존합니다. 프로덕션에서 필수 설정입니다 |
| **`_metadata` 컬럼** | 파일 경로, 수정 시간 등 파일 출처 정보. 데이터 리니지 추적에 활용합니다 |
| **`trigger(availableNow=True)`** | 배치 스타일 실행. 현재 파일을 모두 처리하고 스트림을 종료합니다 |

### 다음 단계

- [JSON 수집과 스키마 진화](json-and-schema-evolution.md) — JSON 포맷 수집과 새 컬럼이 추가되었을 때의 자동 대응을 실습합니다.

---

## 참고 링크

- [Databricks: Auto Loader tutorial](https://docs.databricks.com/aws/en/ingestion/auto-loader/tutorial.html)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html)
- [Databricks: cloudFiles format reference](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html#common-auto-loader-options)
- [Databricks: Unity Catalog Volumes](https://docs.databricks.com/aws/en/connect/unity-catalog/volumes.html)
- [Databricks: Structured Streaming triggers](https://docs.databricks.com/aws/en/structured-streaming/triggers.html)
