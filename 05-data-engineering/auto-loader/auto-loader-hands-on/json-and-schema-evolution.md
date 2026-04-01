# JSON 수집과 스키마 진화

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
