# Auto Loader 스키마 추론과 진화

## 왜 스키마 관리가 중요한가?

실제 데이터 수집 환경에서는 소스 시스템의 스키마가 **사전 공지 없이 변경**되는 일이 빈번합니다. 새 컬럼이 추가되거나, 기존 컬럼의 타입이 바뀌거나, 예상하지 못한 데이터 형식이 들어오기도 합니다. Auto Loader의 스키마 추론과 진화 기능은 이러한 변화를 **자동으로 감지하고 대응**하여 파이프라인의 중단을 최소화합니다.

> 💡 **스키마 추론(Schema Inference)** 은 데이터에서 컬럼 이름과 타입을 자동으로 파악하는 것이고, **스키마 진화(Schema Evolution)** 는 소스 스키마가 변경되었을 때 자동으로 대응하는 것입니다.

---

## schemaLocation 설정

스키마 추론의 핵심은 `cloudFiles.schemaLocation`입니다. Auto Loader가 추론한 스키마를 이 경로에 저장하고, 이후 실행 시 저장된 스키마를 기반으로 데이터를 처리합니다.

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/orders/")
    .load("s3://bucket/raw/orders/")
)
```

### schemaLocation에 저장되는 파일

| 파일 | 설명 |
|------|------|
| `_schemas/0` | 초기 추론된 스키마 (JSON 형식) |
| `_schemas/1`, `2`, ... | 스키마 진화 시 새로 추론된 스키마 버전 |

> ⚠️ **schemaLocation 경로는 스트림별로 고유해야 합니다.** 여러 스트림이 같은 schemaLocation을 공유하면 스키마 충돌이 발생할 수 있습니다.

---

## inferColumnTypes (컬럼 타입 추론)

기본적으로 Auto Loader는 모든 컬럼을 `STRING`으로 추론합니다. `inferColumnTypes`를 `true`로 설정하면 실제 데이터 값을 분석하여 **적절한 타입(INT, DOUBLE, TIMESTAMP 등)** 을 추론합니다.

```python
# inferColumnTypes 비활성화 (기본값) — 모든 컬럼이 STRING
df_string = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "false")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/v1/")
    .load("s3://bucket/data/")
)
# 결과 스키마: order_id STRING, amount STRING, order_date STRING

# inferColumnTypes 활성화 — 데이터 기반 타입 추론
df_typed = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/v2/")
    .load("s3://bucket/data/")
)
# 결과 스키마: order_id BIGINT, amount DOUBLE, order_date TIMESTAMP
```

| 설정 | 동작 | 권장 사용 시나리오 |
|------|------|----------------|
| `false` (기본값) | 모든 컬럼을 STRING으로 추론합니다 | 스키마를 직접 제어하고 싶을 때 |
| `true` | 데이터를 샘플링하여 최적 타입을 추론합니다 | 빠른 프로토타이핑, 타입이 명확한 데이터 |

> 💡 **Parquet, Avro 포맷**에서는 스키마가 파일에 내장되어 있으므로, `inferColumnTypes` 설정과 관계없이 정확한 타입이 자동으로 사용됩니다. 이 옵션은 주로 **JSON, CSV** 같은 스키마가 없는 포맷에서 유용합니다.

---

## schemaEvolutionMode (스키마 진화 모드)

소스 데이터에 새로운 컬럼이 추가되었을 때 Auto Loader가 어떻게 대응할지 결정하는 옵션입니다.

### 4가지 모드 비교

| 모드 | 새 컬럼 감지 시 동작 | 스트림 중단 | 데이터 유실 | 적합한 시나리오 |
|------|-------------------|---------|---------|-----------:|
| **`addNewColumns`** | 스키마에 새 컬럼을 자동 추가합니다 | 예 (재시작 시 반영) | 없음 | 소스 스키마가 점진적으로 확장되는 경우 |
| **`rescue`** | `_rescued_data` 컬럼에 저장합니다 | 아니요 | 없음 | 스키마 변경을 수동으로 검토하고 싶은 경우 |
| **`failOnNewColumns`** | 즉시 스트림을 중단합니다 | 예 (오류 발생) | 없음 | 스키마가 엄격히 통제되어야 하는 경우 |
| **`none`** | 새 컬럼을 무시합니다 | 아니요 | **새 컬럼 데이터 유실** | 고정 스키마로 운영하는 경우 |

### addNewColumns 모드

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/orders/")
    .load("s3://bucket/raw/orders/")
)
```

이 모드에서 새 컬럼이 감지되면 다음과 같은 과정이 발생합니다.

1. 스트림이 `UnknownFieldException`으로 중단됩니다
2. `schemaLocation`에 새 스키마 버전이 저장됩니다
3. 스트림을 재시작하면 새 스키마가 자동으로 적용됩니다

> 💡 **SDP(선언적 파이프라인)** 에서는 스트림 재시작이 자동으로 처리되므로, `addNewColumns` 모드를 사용하면 개입 없이 스키마가 진화합니다.

### rescue 모드

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaEvolutionMode", "rescue")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/orders/")
    .load("s3://bucket/raw/orders/")
)
```

기존 스키마에 맞지 않는 데이터가 `_rescued_data` 컬럼에 JSON 형태로 저장됩니다. 스트림이 중단되지 않으므로 **데이터 유실 없이 안정적으로 운영**할 수 있습니다.

### failOnNewColumns 모드

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaEvolutionMode", "failOnNewColumns")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/orders/")
    .load("s3://bucket/raw/orders/")
)
```

새 컬럼이 감지되면 즉시 오류가 발생합니다. 관리자가 수동으로 스키마를 검토하고 업데이트해야 합니다.

---

## schemaHints (스키마 힌트)

Auto Loader의 타입 추론이 부정확할 때, 특정 컬럼의 타입을 **명시적으로 지정**할 수 있습니다.

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaHints",
            "order_id BIGINT, amount DECIMAL(12,2), order_date DATE, tags ARRAY<STRING>")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/orders/")
    .load("s3://bucket/raw/orders/")
)
```

### schemaHints 활용 사례

| 상황 | 문제 | schemaHints 해결 |
|------|------|-----------------|
| ID 컬럼이 STRING으로 추론됨 | 숫자 ID가 문자열로 저장 | `"id BIGINT"` |
| 금액이 DOUBLE로 추론됨 | 부동소수점 오차 발생 | `"amount DECIMAL(12,2)"` |
| 날짜가 STRING으로 추론됨 | 날짜 연산 불가 | `"created_at TIMESTAMP"` |
| 중첩 구조 타입 지정 | 복잡한 JSON 구조 | `"address STRUCT<city:STRING, zip:STRING>"` |

> 💡 **schemaHints는 추론 결과를 덮어씁니다.** 추론된 타입보다 schemaHints에 지정한 타입이 우선적으로 적용됩니다.

---

## rescuedDataColumn (구조 데이터 컬럼)

스키마에 맞지 않는 데이터를 **별도 컬럼에 보존**하는 기능입니다. 데이터를 유실하지 않으면서도 파이프라인을 중단 없이 운영할 수 있습니다.

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("rescuedDataColumn", "_rescued_data")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/orders/")
    .load("s3://bucket/raw/orders/")
)
```

### rescuedDataColumn에 저장되는 데이터

| 상황 | 동작 |
|------|------|
| 알 수 없는 새 컬럼 | 새 컬럼 데이터가 JSON으로 저장됩니다 |
| 타입 불일치 | 예상 타입으로 변환 실패한 값이 저장됩니다 |
| 대소문자 불일치 | 컬럼명 대소문자가 다른 경우 저장됩니다 |

```sql
-- rescued_data 컬럼 분석: 어떤 데이터가 구조되었는지 확인
SELECT
    _rescued_data,
    COUNT(*) AS rescue_count
FROM catalog.bronze.orders
WHERE _rescued_data IS NOT NULL
GROUP BY _rescued_data
ORDER BY rescue_count DESC
LIMIT 20;
```

---

## SDP(선언적 파이프라인)에서의 스키마 진화

SDP에서는 `read_files()` 함수를 통해 Auto Loader의 스키마 추론과 진화 기능을 그대로 활용할 수 있습니다.

```sql
-- SDP에서 스키마 추론 + 진화 + 구조 데이터 컬럼 활용
CREATE OR REFRESH STREAMING TABLE bronze_orders
AS SELECT
    *,
    _rescued_data,
    _metadata.file_path AS _source_file,
    current_timestamp() AS _ingested_at
FROM STREAM read_files(
    's3://bucket/raw/orders/',
    format => 'json',
    inferColumnTypes => true,
    schemaEvolutionMode => 'addNewColumns',
    rescuedDataColumn => '_rescued_data'
);
```

> 💡 **SDP에서의 자동 재시작**: SDP는 스키마 진화로 인한 스트림 중단 시 **자동으로 재시작**합니다. 따라서 `addNewColumns` 모드를 사용하면 사람의 개입 없이 스키마가 자동으로 진화합니다.

---

## 프로덕션 권장 설정

```python
# 프로덕션 환경 권장 스키마 설정
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    # 스키마 추론
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/orders/")
    # 스키마 진화
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    # 스키마 힌트 (중요 컬럼)
    .option("cloudFiles.schemaHints",
            "order_id BIGINT, amount DECIMAL(12,2), created_at TIMESTAMP")
    # 구조 데이터 보존
    .option("rescuedDataColumn", "_rescued_data")
    .load("s3://bucket/raw/orders/")
)
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **schemaLocation** | 추론된 스키마를 저장하는 경로입니다. 스트림별로 고유해야 합니다 |
| **inferColumnTypes** | `true`로 설정하면 데이터를 분석하여 최적 타입을 추론합니다 |
| **schemaEvolutionMode** | 새 컬럼 감지 시 동작을 결정합니다 (addNewColumns, rescue, failOnNewColumns, none) |
| **schemaHints** | 특정 컬럼의 타입을 명시적으로 지정하여 추론을 보정합니다 |
| **rescuedDataColumn** | 스키마에 맞지 않는 데이터를 별도 컬럼에 보존하여 유실을 방지합니다 |

---

## 참고 링크

- [Databricks: Auto Loader schema inference](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema.html)
- [Databricks: Schema evolution with Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema.html#schema-evolution)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options.html)
- [Azure Databricks: Schema inference and evolution](https://learn.microsoft.com/en-us/azure/databricks/ingestion/cloud-object-storage/auto-loader/schema)
