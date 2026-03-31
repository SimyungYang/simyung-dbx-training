# 스키마 진화(Schema Evolution)와 스키마 강제(Schema Enforcement)

## 왜 스키마 관리가 중요한가요?

데이터 시스템에서 테이블의 구조(스키마)가 변하는 것은 매우 자연스러운 일입니다. 새로운 비즈니스 요구사항이 생기면 컬럼이 추가되고, 불필요한 컬럼은 제거되며, 컬럼 이름이 변경되기도 합니다.

일반 데이터 레이크에서는 스키마 변경이 혼란을 초래하기 쉽습니다. 새 컬럼이 추가된 데이터와 이전 데이터가 섞이면서 분석 결과가 틀어지거나, 잘못된 데이터 타입이 들어와 파이프라인이 깨지는 문제가 발생합니다.

Delta Lake는 이러한 문제를 **스키마 강제(Schema Enforcement)** 와 **스키마 진화(Schema Evolution)** 두 가지 메커니즘으로 해결합니다.

---

## 스키마 강제 (Schema Enforcement)

### 개념

> 💡 **스키마 강제(Schema Enforcement)** 란 테이블에 정의된 스키마(컬럼 이름, 데이터 타입)와 **일치하지 않는 데이터를 자동으로 거부**하는 기능입니다. "Schema-on-Write" 방식이라고도 합니다.

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

> 💡 스키마 강제는 **데이터 품질의 첫 번째 방어선**입니다. 잘못된 데이터가 테이블에 들어오는 것을 원천 차단하여, 다운스트림 분석의 신뢰성을 보장합니다.

---

## 스키마 진화 (Schema Evolution)

### 개념

> 💡 **스키마 진화(Schema Evolution)** 란 테이블의 스키마를 **데이터 유실 없이 안전하게 변경**할 수 있는 기능입니다. 새 컬럼 추가, 컬럼 이름 변경, 컬럼 삭제 등이 가능합니다.

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

DDL을 수동으로 실행하는 대신, 데이터를 쓸 때 **자동으로 스키마를 확장**할 수 있습니다.

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

> 💡 **Column Mapping**은 Delta Lake가 물리적 파일의 컬럼과 논리적 테이블의 컬럼을 **별도로 매핑**하는 기능입니다. 이 기능이 활성화되면 컬럼 이름 변경과 컬럼 삭제가 가능해집니다.

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

Auto Loader는 클라우드 스토리지에서 파일을 자동으로 수집하는 Databricks의 기능입니다. 소스 데이터의 스키마가 시간에 따라 변할 수 있으므로, Auto Loader는 **스키마 진화를 자동으로 처리**할 수 있습니다.

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

> 💡 **rescue 모드**는 예상치 못한 컬럼이 들어왔을 때 데이터를 잃지 않으면서도 기존 스키마를 유지할 수 있어, 프로덕션 환경에서 안전한 선택입니다.

---

## 모범 사례

| 항목 | 권장사항 |
|------|---------|
| **프로덕션 테이블** | 스키마 강제를 기본으로 유지하고, 스키마 변경은 DDL로 명시적으로 수행 |
| **Bronze 레이어** | `mergeSchema` 또는 Auto Loader의 `addNewColumns` 모드 활용 |
| **Silver/Gold 레이어** | 스키마 강제를 엄격하게 적용하여 데이터 품질 보장 |
| **Column Mapping** | 컬럼 이름 변경/삭제가 필요한 테이블에 활성화 |

---

## 정리

| 기능 | 설명 |
|------|------|
| **스키마 강제** | 정의된 스키마와 다른 데이터를 거부하여 데이터 품질을 보장합니다 |
| **스키마 진화** | DDL 또는 `mergeSchema` 옵션으로 스키마를 안전하게 변경합니다 |
| **Column Mapping** | 물리적/논리적 컬럼을 분리하여 이름 변경/삭제를 가능하게 합니다 |
| **Auto Loader 스키마 진화** | 소스 데이터의 스키마 변경을 자동으로 감지하고 처리합니다 |

---

## 참고 링크

- [Databricks: Update Delta Lake table schema](https://docs.databricks.com/aws/en/delta/update-schema.html)
- [Databricks: Delta column mapping](https://docs.databricks.com/aws/en/delta/column-mapping.html)
- [Databricks: Auto Loader schema evolution](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema.html)
- [Azure Databricks: Schema enforcement and evolution](https://learn.microsoft.com/en-us/azure/databricks/delta/update-schema)
