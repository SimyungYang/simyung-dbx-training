# 스키마 진화(Schema Evolution)와 스키마 강제(Schema Enforcement)

## 왜 스키마 관리가 중요한가요?

데이터 시스템에서 테이블의 구조(스키마)가 변하는 것은 매우 자연스러운 일입니다. 새로운 비즈니스 요구사항이 생기면 컬럼이 추가되고, 불필요한 컬럼은 제거되며, 컬럼 이름이 변경되기도 합니다.

일반 데이터 레이크에서는 스키마 변경이 혼란을 초래하기 쉽습니다. 새 컬럼이 추가된 데이터와 이전 데이터가 섞이면서 분석 결과가 틀어지거나, 잘못된 데이터 타입이 들어와 파이프라인이 깨지는 문제가 발생합니다.

Delta Lake는 이러한 문제를 **스키마 강제(Schema Enforcement)**와 **스키마 진화(Schema Evolution)** 두 가지 메커니즘으로 해결합니다.

---

## 스키마 강제 (Schema Enforcement)

### 개념

> 💡 **스키마 강제(Schema Enforcement)**란 테이블에 정의된 스키마(컬럼 이름, 데이터 타입)와 **일치하지 않는 데이터를 자동으로 거부** 하는 기능입니다. "Schema-on-Write" 방식이라고도 합니다.

### 동작 방식

Delta Lake는 데이터를 쓸 때 다음 규칙을 검사합니다.

| 검사 항목 | 동작 |
|-----------|------|
| 테이블에 없는 컬럼이 들어올 때 | ❌ 에러 발생 |
| 데이터 타입이 다를 때 | ❌ 에러 발생 (암시적 형변환 불가 시) |
| 테이블 컬럼이 데이터에 없을 때 | ✅ NULL로 채움 |

### 예제

```sql
-- 테이블 정의
CREATE TABLE catalog.schema.products (
    product_id BIGINT,
    name STRING,
    price DECIMAL(10, 2)
);

-- ✅ 정상: 스키마가 일치
INSERT INTO catalog.schema.products VALUES (1, '노트북', 1500000.00);

-- ❌ 에러: 'category' 컬럼이 테이블에 없음
INSERT INTO catalog.schema.products (product_id, name, price, category)
VALUES (2, '키보드', 89000.00, '전자제품');
-- AnalysisException: Cannot write to table because the column 'category' is not found

-- ❌ 에러: price에 문자열 삽입 시도
INSERT INTO catalog.schema.products VALUES (3, '마우스', 'cheap');
-- AnalysisException: Cannot safely cast 'price': string to decimal(10,2)
```

> 💡 스키마 강제는 **데이터 품질의 첫 번째 방어선** 입니다. 잘못된 데이터가 테이블에 들어오는 것을 원천 차단하여, 다운스트림 분석의 신뢰성을 보장합니다.

---

## 스키마 진화 (Schema Evolution)

### 개념

> 💡 **스키마 진화(Schema Evolution)**란 테이블의 스키마를 **데이터 유실 없이 안전하게 변경** 할 수 있는 기능입니다. 새 컬럼 추가, 컬럼 이름 변경, 컬럼 삭제 등이 가능합니다.

### DDL을 통한 스키마 변경

```sql
-- 컬럼 추가
ALTER TABLE catalog.schema.products
ADD COLUMN category STRING;

-- 컬럼 추가 (기본값 포함)
ALTER TABLE catalog.schema.products
ADD COLUMN created_at TIMESTAMP DEFAULT current_timestamp();

-- 컬럼 이름 변경 (Column Mapping 필요)
ALTER TABLE catalog.schema.products
RENAME COLUMN name TO product_name;

-- 컬럼 삭제 (Column Mapping 필요)
ALTER TABLE catalog.schema.products
DROP COLUMN category;

-- 컬럼 타입 변경 (안전한 확장만 가능)
ALTER TABLE catalog.schema.products
ALTER COLUMN product_id TYPE BIGINT;  -- INT → BIGINT은 가능
```

### mergeSchema 옵션 (쓰기 시 자동 스키마 진화)

DDL을 수동으로 실행하는 대신, 데이터를 쓸 때 **자동으로 스키마를 확장** 할 수 있습니다.

```sql
-- SQL: INSERT 시 mergeSchema 활성화
SET spark.databricks.delta.schema.autoMerge.enabled = true;

INSERT INTO catalog.schema.products VALUES
(4, '태블릿', 800000.00, '전자제품', current_timestamp());
-- 'category' 컬럼이 자동으로 추가됩니다
```

```python
# PySpark: DataFrame 쓰기 시 mergeSchema 옵션
df_new = spark.createDataFrame([
    (5, "스피커", 250000.00, "전자제품", "2025-03-20")
], ["product_id", "name", "price", "category", "launch_date"])

df_new.write \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable("catalog.schema.products")
```

### MERGE 문에서의 스키마 진화

```sql
-- MERGE에서도 스키마 진화가 가능합니다
SET spark.databricks.delta.schema.autoMerge.enabled = true;

MERGE INTO catalog.schema.products AS target
USING new_products AS source
ON target.product_id = source.product_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
-- source에 새 컬럼이 있으면 자동으로 추가됩니다
```

---

## Column Mapping Mode

### 개념

> 💡 **Column Mapping**은 Delta Lake가 물리적 파일의 컬럼과 논리적 테이블의 컬럼을 **별도로 매핑** 하는 기능입니다. 이 기능이 활성화되면 컬럼 이름 변경과 컬럼 삭제가 가능해집니다.

### 활성화 방법

```sql
-- Column Mapping 활성화
ALTER TABLE catalog.schema.products
SET TBLPROPERTIES ('delta.columnMapping.mode' = 'name');
```

| 모드 | 설명 |
|------|------|
| `none` | 기본값. 컬럼 이름 변경/삭제 불가 |
| `name` | 컬럼 이름 기반 매핑. 이름 변경/삭제 가능 |

> ⚠️ **주의**: Databricks Runtime 13.4 이상에서 생성된 테이블은 기본적으로 Column Mapping이 `name` 모드로 활성화되어 있습니다. 따라서 별도 설정 없이 바로 컬럼 이름 변경과 삭제가 가능합니다.

### 컬럼 이름 변경과 삭제 예제

```sql
-- Column Mapping이 활성화된 테이블에서

-- 컬럼 이름 변경
ALTER TABLE catalog.schema.products
RENAME COLUMN name TO product_name;

-- 컬럼 삭제
ALTER TABLE catalog.schema.products
DROP COLUMN old_unused_column;

-- 여러 컬럼 동시 삭제
ALTER TABLE catalog.schema.products
DROP COLUMNS (temp_col1, temp_col2);
```

---

## Auto Loader에서의 스키마 진화

Auto Loader는 클라우드 스토리지에서 파일을 자동으로 수집하는 Databricks의 기능입니다. 소스 데이터의 스키마가 시간에 따라 변할 수 있으므로, Auto Loader는 **스키마 진화를 자동으로 처리** 할 수 있습니다.

```python
# Auto Loader에서 스키마 진화 활성화
(spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/checkpoints/schema")
    # 스키마 진화 모드 설정
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .load("s3://my-bucket/raw-data/")
    .writeStream
    .option("checkpointLocation", "/checkpoints/stream")
    .option("mergeSchema", "true")
    .table("catalog.schema.bronze_events")
)
```

| 스키마 진화 모드 | 동작 |
|-----------------|------|
| `addNewColumns` | 새 컬럼 발견 시 자동 추가 (기본값) |
| `rescue` | 새 컬럼을 `_rescued_data` 컬럼에 JSON으로 저장 |
| `failOnNewColumns` | 새 컬럼 발견 시 스트림 중단 |
| `none` | 스키마 변경 무시 |

> 💡 **rescue 모드** 는 예상치 못한 컬럼이 들어왔을 때 데이터를 잃지 않으면서도 기존 스키마를 유지할 수 있어, 프로덕션 환경에서 안전한 선택입니다.

---

## 모범 사례

| 항목 | 권장사항 |
|------|---------|
| **프로덕션 테이블**| 스키마 강제를 기본으로 유지하고, 스키마 변경은 DDL로 명시적으로 수행 |
| **Bronze 레이어**| `mergeSchema` 또는 Auto Loader의 `addNewColumns` 모드 활용 |
| **Silver/Gold 레이어**| 스키마 강제를 엄격하게 적용하여 데이터 품질 보장 |
| **Column Mapping**| 컬럼 이름 변경/삭제가 필요한 테이블에 활성화 |

---

## Column Mapping Mode 내부 동작 상세

Column Mapping은 Delta Lake에서 스키마 유연성을 크게 높이는 핵심 기능입니다. 내부 동작 원리를 이해하면 제한사항과 문제를 예방할 수 있습니다.

### 물리적 이름 vs 논리적 이름

Column Mapping이 활성화되면, 테이블의 **논리적 컬럼 이름**(사용자가 보는 이름)과 **물리적 컬럼 이름**(Parquet 파일 내 실제 이름)이 분리됩니다.

```
| Column Mapping 모드 | 논리적 이름 | 물리적 이름 | 설명 |
|--------------------|-----------|-----------|------|
| `none` (기본, 비활성) | customer_name | customer_name | 동일 |
| `name` (활성화) | customer_name | col-a1b2c3d4-e5f6 | UUID 기반 물리적 이름 |
```

| 모드 | 논리적 이름 | 물리적 이름 | 이름 변경 시 |
|------|-----------|-----------|-------------|
| `none` | `customer_name` | `customer_name` (동일) | 불가 — 모든 Parquet 파일을 다시 써야 함 |
| `name` | `customer_name` | `col-a1b2c3d4` (UUID) | 메타데이터만 변경 — 파일 재작성 불필요 |

### 메타데이터 저장 위치

Column Mapping 정보는 Delta Log의 **메타데이터 액션** 에 저장됩니다.

```json
// Delta Log의 metadata action 내 column mapping 정보
{
  "metaData": {
    "schemaString": "{\"fields\":[{\"name\":\"customer_name\",\"metadata\":{\"delta.columnMapping.id\":1,\"delta.columnMapping.physicalName\":\"col-a1b2c3d4-e5f6\"}}]}",
    "configuration": {
      "delta.columnMapping.mode": "name",
      "delta.columnMapping.maxColumnId": "5"
    }
  }
}
```

### 이름 변경/삭제의 실제 동작

```sql
-- Column Mapping = name 일 때
ALTER TABLE products RENAME COLUMN customer_name TO client_name;

-- 실제 일어나는 일:
-- 1. Delta Log에 새 metadata action 기록
-- 2. 논리적 이름: customer_name → client_name (변경)
-- 3. 물리적 이름: col-a1b2c3d4 (변경 없음!)
-- 4. 기존 Parquet 파일: 수정 없음 (제로 카피 연산)
-- → 수 TB 테이블이라도 이름 변경이 밀리초 단위로 완료됩니다
```

---

## Column Mapping 활성화 시 제한사항

Column Mapping을 활성화하면 강력한 기능을 얻지만, 몇 가지 중요한 제한사항이 있습니다.

### 호환성 제한

| 제한사항 | 설명 | 해결 방법 |
|---------|------|----------|
| **되돌리기 불가**| `name` 모드를 `none`으로 되돌릴 수 없습니다 | 새 테이블을 생성하여 데이터를 복사 |
| **Delta 프로토콜 요구**| Reader Version 2, Writer Version 5 이상 필요 | 구버전 Delta 클라이언트에서 읽기 불가 |
| **Streaming 읽기**| Column Mapping 변경 후 기존 스트리밍 체크포인트가 무효화될 수 있음 | 스트림을 중지 → 체크포인트 삭제 → 재시작 |
| **Delta Sharing**| 일부 구버전 소비자가 Column Mapping 테이블을 읽지 못할 수 있음 | 소비자 클라이언트 버전 확인 |
| **Time Travel**| 이름 변경 전 버전을 조회할 때 현재 논리적 이름으로 접근해야 함 | `SELECT client_name FROM t VERSION AS OF 5` (이전에는 customer_name이었더라도) |
| **CLONE**| Column Mapping 테이블의 SHALLOW CLONE은 제한적 | DEEP CLONE 사용 권장 |

### Streaming과 Column Mapping의 상호작용

> ⚠️ **중요**: Column Mapping이 활성화된 테이블에서 컬럼을 **이름 변경하거나 삭제**하면, 해당 테이블을 읽는 **Structured Streaming 쿼리가 실패** 할 수 있습니다.

```python
# Column Mapping 변경 후 스트리밍 재시작 절차
# 1. 스트리밍 쿼리 중지
streaming_query.stop()

# 2. 체크포인트 초기화 (또는 삭제)
dbutils.fs.rm("/checkpoints/my_stream", recurse=True)

# 3. 스트리밍 쿼리 재시작 (새 체크포인트 경로)
(spark.readStream.table("source_table")
    .writeStream
    .option("checkpointLocation", "/checkpoints/my_stream_v2")  # 새 경로
    .table("target_table"))
```

---

## 대규모 스키마 변경 전략 (컬럼 100개+ 추가)

실무에서는 스키마가 한 번에 수십~수백 개의 컬럼이 추가되는 경우가 있습니다(예: IoT 센서 데이터, 설문 데이터 등).

### 대량 컬럼 추가 방법 비교

| 방법 | 장점 | 단점 | 적합한 경우 |
|------|------|------|------------|
| **ALTER TABLE ADD COLUMN 반복**| 명시적, 각 컬럼에 주석 가능 | 100개면 100번 DDL 실행. 느림 | 소규모 변경(10개 이내) |
| **mergeSchema로 자동 추가**| DataFrame 쓰기 한 번으로 모든 컬럼 추가 | 타입/이름 제어가 어려움 | 소스의 스키마를 그대로 수용할 때 |
| **CREATE TABLE AS SELECT**| 새 테이블에 원하는 스키마 구성 | 데이터 전체 복사 필요 | 대폭 스키마 재설계 시 |
| **REPLACE TABLE**| 스키마를 완전히 교체하면서 동일 테이블명 유지 | 기존 데이터 삭제됨 | 빈 테이블 재정의 |

### 대량 컬럼 추가 실습 (mergeSchema 활용)

```python
# 100개 이상의 새 컬럼이 있는 DataFrame을 기존 테이블에 병합
from pyspark.sql.types import StructType, StructField, StringType, DoubleType

# 새 스키마로 DataFrame 생성 (기존 컬럼 + 새 100개 컬럼)
new_columns = [StructField(f"sensor_{i:03d}", DoubleType(), True) for i in range(100)]
# ... DataFrame 준비 ...

# mergeSchema로 한 번에 추가
df_new_data.write \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable("catalog.schema.sensor_readings")

# 결과: 기존 테이블에 100개의 새 컬럼이 자동 추가됩니다
# 기존 행의 새 컬럼 값은 NULL로 채워집니다
```

### 스키마 변경 시 다운스트림 영향 관리

| 영향 | 설명 | 대응 |
|------|------|------|
| **하위 뷰/쿼리** | `SELECT *`을 사용하는 뷰는 새 컬럼이 자동 포함됨 | 명시적 컬럼 목록 사용 권장 |
| **BI 대시보드**| 새 컬럼이 대시보드에 예기치 않게 나타날 수 있음 | Gold 레이어에서 스키마를 고정하여 BI에 제공 |
| **ML 파이프라인**| 피처 수가 변경되면 모델 학습이 실패할 수 있음 | 피처 선택 로직에서 컬럼 목록을 명시적으로 지정 |
| **Delta Sharing**| Share에 추가된 테이블의 스키마가 변경되면 소비자에게 영향 | 스키마 변경 전 소비자에게 사전 공지 |

---

## Auto Loader 스키마 진화와의 관계

Auto Loader의 스키마 진화는 Delta Lake의 스키마 진화(`mergeSchema`)와 함께 동작하며, **소스 파일 → 타겟 테이블** 전체 흐름에서 스키마를 관리합니다.

### Auto Loader + Delta 스키마 진화 전체 흐름

```
[소스 파일]                  [Auto Loader]              [Delta 테이블]
새 JSON 파일 도착        →   스키마 감지               →  스키마 진화
(새 필드 "loyalty_tier"      (addNewColumns 모드)         (mergeSchema=true)
 포함)                       새 컬럼 발견!                loyalty_tier 컬럼 추가
                             스키마 파일 업데이트          기존 행은 NULL
                             스트림 재시작 (자동)
```

### schemaEvolutionMode와 Delta mergeSchema의 관계

| Auto Loader 모드 | Delta mergeSchema | 동작 |
|-----------------|-------------------|------|
| `addNewColumns` + `mergeSchema=true` | 활성 | 새 컬럼이 소스에서 타겟까지 자동 전파됩니다 |
| `addNewColumns` + `mergeSchema=false` | 비활성 | Auto Loader는 새 컬럼을 감지하지만 Delta 쓰기에서 에러 발생 |
| `rescue` + 어떤 설정이든 | 무관 | 새 컬럼은 `_rescued_data`에 JSON으로 저장. 테이블 스키마 변경 없음 |
| `failOnNewColumns` | 무관 | 새 컬럼 발견 시 스트림이 즉시 중단됩니다 |

### Rescue 모드 활용 패턴 (프로덕션 권장)

```python
# 프로덕션에서 안전한 스키마 진화 패턴
# 1단계: rescue 모드로 안전하게 수집
(spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/checkpoints/schema")
    .option("cloudFiles.schemaEvolutionMode", "rescue")  # 새 컬럼은 _rescued_data에 저장
    .load("s3://bucket/raw/")
    .writeStream
    .option("checkpointLocation", "/checkpoints/bronze")
    .table("catalog.schema.bronze_events"))

# 2단계: _rescued_data 분석하여 새 컬럼 확인
rescued = spark.sql("""
    SELECT _rescued_data, COUNT(*) as cnt
    FROM catalog.schema.bronze_events
    WHERE _rescued_data IS NOT NULL
    GROUP BY _rescued_data
    ORDER BY cnt DESC
""")

# 3단계: 확인 후 DDL로 명시적으로 스키마 변경
# ALTER TABLE catalog.schema.bronze_events ADD COLUMN loyalty_tier STRING;

# 4단계: 스트림 재시작 (체크포인트 유지, 새 컬럼 포함)
```

> 💡 **프로덕션 권장 패턴**: Bronze 레이어에서는 `rescue` 모드를 사용하여 예기치 않은 스키마 변경을 안전하게 캡처하고, Silver/Gold 레이어에서는 DDL로 명시적 스키마 관리를 하는 것이 가장 안정적입니다.

---

## SDP(선언적 파이프라인)에서의 스키마 진화 처리

SDP(Structured Data Pipelines, 구 DLT)에서도 스키마 진화를 처리할 수 있습니다.

### SDP에서의 스키마 진화 설정

```python
# SDP 파이프라인에서 Auto Loader + 스키마 진화
import dlt

@dlt.table(
    comment="Bronze: 원본 이벤트 데이터"
)
def bronze_events():
    return (
        spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
        .option("cloudFiles.schemaLocation", "/checkpoints/schema")
        .load("s3://bucket/raw/events/")
    )

@dlt.table(
    comment="Silver: 정제된 이벤트"
)
@dlt.expect_or_drop("valid_event_type", "event_type IS NOT NULL")
def silver_events():
    return dlt.read_stream("bronze_events").select(
        "event_id",
        "event_type",
        "user_id",
        "timestamp"
        # 명시적 컬럼 선택 → Bronze에 새 컬럼이 추가되어도 Silver에는 영향 없음
    )
```

### SDP 스키마 진화 시 주의사항

| 주의사항 | 설명 | 대응 |
|---------|------|------|
| **Full Refresh 트리거**| 스키마 변경 시 SDP가 전체 재처리를 할 수 있음 | `pipelines.reset.allowed = false` 설정으로 방지 |
| **Expectations 깨짐**| 새 컬럼에 의존하는 Expectation이 없으면 괜찮지만, 기존 컬럼이 변경되면 문제 | 컬럼 타입 변경은 별도 마이그레이션으로 처리 |
| **Bronze → Silver 전파** | Bronze에서 `SELECT *`을 사용하면 새 컬럼이 자동 전파됨 | Silver에서는 명시적 컬럼 목록 사용 |
| **파이프라인 재시작 필요**| 일부 스키마 변경 후 파이프라인을 수동 재시작해야 할 수 있음 | 모니터링 및 자동 재시작 설정 |

---

## 정리

| 기능 | 설명 |
|------|------|
| **스키마 강제**| 정의된 스키마와 다른 데이터를 거부하여 데이터 품질을 보장합니다 |
| **스키마 진화**| DDL 또는 `mergeSchema` 옵션으로 스키마를 안전하게 변경합니다 |
| **Column Mapping**| 물리적/논리적 컬럼을 분리하여 이름 변경/삭제를 가능하게 합니다 |
| **Auto Loader 스키마 진화**| 소스 데이터의 스키마 변경을 자동으로 감지하고 처리합니다 |
| **Column Mapping 내부**| UUID 기반 물리적 이름을 사용하여 논리적 이름과 분리합니다 |
| **Column Mapping 제한**| 되돌리기 불가, 스트리밍 체크포인트 무효화, SHALLOW CLONE 제한이 있습니다 |
| **대량 스키마 변경**| mergeSchema로 한 번에 수백 개 컬럼 추가가 가능합니다 |
| **rescue 모드**| 새 컬럼을 _rescued_data에 저장하여 프로덕션에서 안전하게 처리합니다 |
| **SDP 스키마 진화** | Bronze에서 자동 진화, Silver/Gold에서 명시적 컬럼 선택이 권장됩니다 |

---

## 참고 링크

- [Databricks: Update Delta Lake table schema](https://docs.databricks.com/aws/en/delta/update-schema.html)
- [Databricks: Delta column mapping](https://docs.databricks.com/aws/en/delta/column-mapping.html)
- [Databricks: Auto Loader schema evolution](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema.html)
- [Azure Databricks: Schema enforcement and evolution](https://learn.microsoft.com/en-us/azure/databricks/delta/update-schema)
