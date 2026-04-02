# 실습 2: JSON 수집과 스키마 진화

## 목적과 학습 목표

JSON (JavaScript Object Notation) 파일을 수집하면서, 소스 시스템이 **새 컬럼을 추가** 했을 때 Auto Loader가 어떻게 자동으로 대응하는지를 실습합니다. 실제 프로덕션 환경에서 가장 빈번하게 발생하는 문제인 **스키마 드리프트 (Schema Drift)** 를 안전하게 처리하는 방법을 익힙니다.

### 학습 목표

| 목표 | 설명 |
|------|------|
| **JSON 수집** | JSON Lines 포맷의 파일을 Auto Loader로 수집합니다 |
| **스키마 추론** | `inferColumnTypes`로 중첩 구조를 포함한 타입을 자동 추론합니다 |
| **스키마 진화** | `schemaEvolutionMode=addNewColumns`로 새 컬럼이 자동으로 테이블에 추가되는 것을 확인합니다 |
| **기존 데이터 호환성** | 새 컬럼이 추가되어도 기존 레코드는 `NULL`로 채워져 손상 없이 유지됩니다 |
| **Rescue Data** | 스키마에 맞지 않는 데이터가 `_rescued_data` 컬럼에 보존되는 것을 확인합니다 |

> **스키마 드리프트(Schema Drift) 란?**: 소스 시스템이 API 업데이트, 서비스 변경 등으로 기존 스키마에 새 필드를 추가하거나 타입을 바꾸는 현상입니다. 기존 파이프라인은 이를 인식하지 못해 에러가 발생하거나 새 데이터가 무시됩니다. Auto Loader의 스키마 진화 기능은 이 문제를 자동으로 처리합니다.

---

## 사전 준비

[실습 1: 사전 준비와 CSV 수집](setup-and-csv.md) 을 완료해야 합니다.

- 카탈로그 `training`, 스키마 `auto_loader_lab`, Volume `raw_data` 가 존재해야 합니다.
- 다음 Python 임포트를 노트북 첫 셀에서 실행합니다:

```python
import json
import random
from datetime import datetime
from pyspark.sql.functions import current_timestamp, col

random.seed(42)
```

---

## 실습 2A: JSON 파일 초기 수집

### Step 1: JSON 샘플 데이터 생성

JSON Lines 포맷 (한 줄에 하나의 JSON 객체) 으로 고객 데이터를 생성합니다.

```python
# --- JSON 고객 데이터 생성 (Batch 1: 50건, 기본 스키마) ---
# 스키마: customer_id, name, email, city, registered_at
customers_v1 = []
for i in range(1000, 1050):
    customers_v1.append(json.dumps({
        "customer_id": i,
        "name": f"고객_{i}",
        "email": f"customer{i}@example.com",
        "city": random.choice(["서울", "부산", "대구", "인천", "광주"]),
        "registered_at": f"2025-{random.randint(1,3):02d}-{random.randint(1,28):02d}T10:00:00Z"
    }, ensure_ascii=False))

json_content_v1 = "\n".join(customers_v1)  # JSON Lines 형식

dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/json/customers/customers_batch1.json",
    json_content_v1,
    overwrite=True
)

print(f"JSON 배치 1 (v1 스키마) 생성 완료: {len(customers_v1)}건")
print("컬럼: customer_id, name, email, city, registered_at")
```

---

### Step 2: Auto Loader로 JSON 읽기

```python
source_path     = "/Volumes/training/auto_loader_lab/raw_data/json/customers/"
checkpoint_path = "/Volumes/training/auto_loader_lab/checkpoints/json_customers/"
schema_path     = "/Volumes/training/auto_loader_lab/schema/json_customers/"

df_customers = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    # addNewColumns: 새 컬럼 발견 시 스키마에 자동 추가 (기본값)
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
    .withColumn("_source_file", col("_metadata.file_path"))
    .withColumn("_ingested_at", current_timestamp())
)

# 스키마 확인
df_customers.printSchema()
```

**예상 스키마** (v1 추론):
```
root
 |-- city: string (nullable = true)
 |-- customer_id: long (nullable = true)
 |-- email: string (nullable = true)
 |-- name: string (nullable = true)
 |-- registered_at: timestamp (nullable = true)
 |-- _rescued_data: string (nullable = true)
 |-- _source_file: string (nullable = true)
 |-- _ingested_at: timestamp (nullable = true)
```

---

### Step 3: Bronze 테이블에 저장

```python
query = (
    df_customers
    .writeStream
    .option("checkpointLocation", checkpoint_path)
    .option("mergeSchema", "true")
    .trigger(availableNow=True)
    .toTable("training.auto_loader_lab.bronze_customers")
)

query.awaitTermination()
print("JSON 배치 1 수집 완료!")
```

```sql
-- 초기 데이터 및 스키마 확인
DESCRIBE training.auto_loader_lab.bronze_customers;
```

```sql
SELECT customer_id, name, city, registered_at, _ingested_at
FROM training.auto_loader_lab.bronze_customers
LIMIT 5;
```

---

## 실습 2B: 스키마 진화 시뮬레이션

소스 시스템이 업그레이드되어 **새 필드 2개** (`phone`, `membership_level`) 가 추가된 상황을 시뮬레이션합니다.

### Step 1: v2 스키마 JSON 파일 생성

```python
# Batch 2: 새 컬럼이 추가된 v2 스키마 (30건)
# 스키마: customer_id, name, email, city, registered_at, phone(신규), membership_level(신규)
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

print(f"JSON 배치 2 (v2 스키마) 생성 완료: {len(customers_v2)}건")
print("추가된 컬럼: phone, membership_level")
```

---

### Step 2: 스키마 진화 트리거

동일한 스트리밍 쿼리를 다시 실행합니다. Auto Loader가 새 컬럼을 감지하고 스키마를 자동으로 확장합니다.

```python
# 중요: 동일한 checkpoint_path를 사용합니다
# → batch2.json만 처리됩니다 (batch1은 이미 처리됨)
# → 새 컬럼 phone, membership_level이 자동 감지됩니다
query2 = (
    df_customers
    .writeStream
    .option("checkpointLocation", checkpoint_path)   # 동일 체크포인트
    .option("mergeSchema", "true")                   # Delta 테이블 스키마도 확장
    .trigger(availableNow=True)
    .toTable("training.auto_loader_lab.bronze_customers")
)

query2.awaitTermination()
print("스키마 진화 처리 완료!")
```

> **내부 동작**: `schemaEvolutionMode=addNewColumns`는 신규 파일에서 기존 스키마에 없는 컬럼을 발견하면, `schemaLocation`의 스키마 파일을 업데이트하고 스트림을 재시작합니다. `mergeSchema=true`는 Delta 테이블 자체의 스키마도 새 컬럼을 포함하도록 확장합니다.

---

### Step 3: 스키마 진화 결과 확인

```sql
-- 스키마 진화 확인: phone, membership_level 컬럼이 추가되었는지 확인
DESCRIBE training.auto_loader_lab.bronze_customers;
```

```sql
-- 버전별 데이터 분포 확인
SELECT
    CASE WHEN phone IS NOT NULL THEN 'v2 (phone+membership)' ELSE 'v1 (original)' END AS schema_version,
    COUNT(*)  AS row_count,
    MIN(customer_id) AS min_id,
    MAX(customer_id) AS max_id
FROM training.auto_loader_lab.bronze_customers
GROUP BY 1
ORDER BY 1;
```

**예상 결과**:
```
schema_version          | row_count | min_id | max_id
------------------------+-----------+--------+-------
v1 (original)           | 50        | 1000   | 1049
v2 (phone+membership)   | 30        | 1050   | 1079
```

```sql
-- v1 레코드는 새 컬럼이 NULL로 채워집니다
SELECT customer_id, name, phone, membership_level
FROM training.auto_loader_lab.bronze_customers
ORDER BY customer_id
LIMIT 10;
```

> **스키마 진화 결과**: `phone`과 `membership_level` 컬럼이 테이블에 추가되었습니다. 이전 배치 (v1) 레코드는 해당 컬럼이 `NULL`로 표시됩니다. 이것이 **Backward Compatible Schema Evolution (하위 호환 스키마 진화)** 입니다.

---

## 실습 2C: 중첩 JSON 구조 처리

실제 API 응답에서 자주 등장하는 중첩 JSON 구조를 처리합니다.

```python
# 중첩 JSON 데이터 생성
nested_customers = []
for i in range(1080, 1100):
    nested_customers.append(json.dumps({
        "customer_id": i,
        "name": f"고객_{i}",
        "contact": {                    # 중첩 객체
            "email": f"customer{i}@example.com",
            "phone": f"010-{random.randint(1000,9999)}-{random.randint(1000,9999)}"
        },
        "address": {                    # 중첩 객체
            "city": random.choice(["서울", "부산", "대구"]),
            "zip": f"{random.randint(10000,99999)}"
        },
        "tags": random.sample(["VIP", "신규", "활성", "휴면"], k=2)  # 배열
    }, ensure_ascii=False))

json_nested_content = "\n".join(nested_customers)

dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/json/customers/customers_nested.json",
    json_nested_content,
    overwrite=True
)

print("중첩 JSON 생성 완료")
```

```python
# 중첩 구조는 StructType으로 자동 추론됩니다
# contact.email, contact.phone, address.city 등으로 접근합니다
from pyspark.sql.functions import col

nested_path = "/Volumes/training/auto_loader_lab/raw_data/json/nested_customers/"
schema_path_nested = "/Volumes/training/auto_loader_lab/schema/nested_customers/"
checkpoint_path_nested = "/Volumes/training/auto_loader_lab/checkpoints/nested_customers/"

dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/json/nested_customers/batch1.json",
    json_nested_content,
    overwrite=True
)

df_nested = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path_nested)
    .option("cloudFiles.inferColumnTypes", "true")
    .option("rescuedDataColumn", "_rescued_data")
    .load(nested_path)
    # 중첩 컬럼 펼치기 (Flatten)
    .select(
        col("customer_id"),
        col("name"),
        col("contact.email").alias("email"),
        col("contact.phone").alias("phone"),
        col("address.city").alias("city"),
        col("address.zip").alias("zip"),
        col("tags"),
        col("_metadata.file_path").alias("_source_file"),
        current_timestamp().alias("_ingested_at")
    )
)

df_nested.printSchema()
```

**예상 스키마** (중첩 구조가 자동으로 `StructType`으로 추론됨):
```
root
 |-- customer_id: long
 |-- name: string
 |-- email: string
 |-- phone: string
 |-- city: string
 |-- zip: string
 |-- tags: array<string>
 |-- _source_file: string
 |-- _ingested_at: timestamp
```

---

## 심화 학습

### 스키마 진화 모드 비교 실습

4가지 `schemaEvolutionMode`를 상황별로 비교합니다.

| 모드 | 새 컬럼 발견 시 동작 | 사용 시나리오 |
|------|---------------------|--------------|
| `addNewColumns` (기본값) | 스키마에 자동 추가 | 소스 스키마가 자주 확장되는 경우 |
| `rescue` | `_rescued_data`에 JSON으로 보존 | 변경 사항을 사람이 검토 후 반영하는 경우 |
| `failOnNewColumns` | 스트림을 중단하고 에러 발생 | 스키마 변경이 절대 허용되지 않는 경우 |
| `none` | 새 컬럼 무시 (데이터 손실) | 고정 스키마로 엄격히 운영하는 경우 |

```python
# rescue 모드: 새 컬럼을 _rescued_data에 보존
df_rescue_mode = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path + "_rescue_test")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaEvolutionMode", "rescue")   # rescue 모드
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
# rescue 모드에서 v2 파일(phone, membership_level 포함)을 수집하면:
# phone과 membership_level 데이터가 _rescued_data에 JSON으로 저장됩니다
# 예: _rescued_data = '{"phone":"010-1234-5678","membership_level":"GOLD"}'
```

### 트러블슈팅: 타입 충돌 처리

소스 시스템이 `customer_id`를 갑자기 STRING으로 바꾼 경우:

```python
# 문제 상황: customer_id가 "C1080" 같은 문자열로 변경됨
conflicting_data = json.dumps({
    "customer_id": "C9999",   # 기존: INT, 변경: STRING
    "name": "테스트 고객",
    "email": "test@example.com",
    "city": "서울",
    "registered_at": "2025-04-01T00:00:00Z"
}, ensure_ascii=False)

# 해결책 1: schemaHints로 STRING으로 고정
# .option("cloudFiles.schemaHints", "customer_id STRING")

# 해결책 2: rescuedDataColumn으로 보존 후 수동 처리
# 타입 불일치 행은 _rescued_data에 저장되고, 해당 컬럼은 NULL이 됩니다

# 해결책 3: Bronze를 모두 STRING으로 수집 후 Silver에서 CAST
# .option("cloudFiles.inferColumnTypes", "false")  # 모두 STRING
```

### 성능 튜닝: 대용량 JSON 처리

```python
# 대용량 JSON 파일에 대한 최적화 설정
df_large = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    # 성능 튜닝
    .option("cloudFiles.maxFilesPerTrigger", "200")   # 트리거당 처리 파일 수 제한
    .option("cloudFiles.maxBytesPerTrigger", "2g")    # 트리거당 최대 2GB
    # 멀티라인 JSON (한 파일에 하나의 JSON 객체)
    # .option("multiLine", "true")    # JSON Lines가 아닌 경우 활성화
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
```

---

## 정리

### 핵심 요약

| 개념 | 설명 |
|------|------|
| **JSON Lines 포맷** | 한 줄에 하나의 JSON 객체. `multiLine=false` (기본값) 로 수집합니다 |
| **중첩 구조 자동 추론** | `StructType`, `ArrayType`으로 자동 추론됩니다. `col("a.b")`로 접근합니다 |
| **`schemaEvolutionMode=addNewColumns`** | 새 컬럼 자동 추가. `mergeSchema=true`와 함께 사용해야 Delta 테이블도 확장됩니다 |
| **기존 레코드 호환** | 새 컬럼 추가 후 기존 레코드는 해당 컬럼이 `NULL`로 채워집니다 |
| **타입 충돌** | 기존 컬럼의 타입이 변경되면 자동 처리되지 않습니다. `schemaHints` 또는 `_rescued_data`로 처리합니다 |

### 스키마 진화 의사결정 트리

```
소스에서 새 컬럼이 추가됨
    ├── 자동으로 수집하고 싶다 → addNewColumns
    ├── 수동 검토 후 반영하고 싶다 → rescue
    ├── 반드시 알람을 받고 싶다 → failOnNewColumns
    └── 무시해도 된다 → none (비권장, 데이터 손실)
```

### 다음 단계

- [SDP와 Auto Loader 통합](sdp-integration.md) — `read_files()` 함수로 Medallion Architecture (메달리온 아키텍처) 파이프라인을 구축합니다.

---

## 참고 링크

- [Databricks: Auto Loader schema inference and evolution](https://docs.databricks.com/aws/en/ingestion/auto-loader/schema.html)
- [Databricks: Schema evolution options](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html#schema-inference-options)
- [Databricks: Rescued data column](https://docs.databricks.com/aws/en/ingestion/auto-loader/schema.html#rescued-data-column)
- [Delta Lake: Schema evolution](https://docs.databricks.com/aws/en/delta/update-schema.html)
