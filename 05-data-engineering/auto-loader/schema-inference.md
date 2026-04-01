# Auto Loader 스키마 추론과 진화

## 왜 스키마 관리가 중요한가?

실제 데이터 수집 환경에서는 소스 시스템의 스키마가 **사전 공지 없이 변경** 되는 일이 빈번합니다. 새 컬럼이 추가되거나, 기존 컬럼의 타입이 바뀌거나, 예상하지 못한 데이터 형식이 들어오기도 합니다. Auto Loader의 스키마 추론과 진화 기능은 이러한 변화를 **자동으로 감지하고 대응** 하여 파이프라인의 중단을 최소화합니다.

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

> 💡 **Parquet, Avro 포맷** 에서는 스키마가 파일에 내장되어 있으므로, `inferColumnTypes` 설정과 관계없이 정확한 타입이 자동으로 사용됩니다. 이 옵션은 주로 **JSON, CSV** 같은 스키마가 없는 포맷에서 유용합니다.

---

## schemaEvolutionMode (스키마 진화 모드)

소스 데이터에 새로운 컬럼이 추가되었을 때 Auto Loader가 어떻게 대응할지 결정하는 옵션입니다.

### 4가지 모드 비교

| 모드 | 새 컬럼 감지 시 동작 | 스트림 중단 | 데이터 유실 | 적합한 시나리오 |
|------|-------------------|---------|---------|-----------:|
| **`addNewColumns`**| 스키마에 새 컬럼을 자동 추가합니다 | 예 (재시작 시 반영) | 없음 | 소스 스키마가 점진적으로 확장되는 경우 |
| **`rescue`**| `_rescued_data` 컬럼에 저장합니다 | 아니요 | 없음 | 스키마 변경을 수동으로 검토하고 싶은 경우 |
| **`failOnNewColumns`**| 즉시 스트림을 중단합니다 | 예 (오류 발생) | 없음 | 스키마가 엄격히 통제되어야 하는 경우 |
| **`none`**| 새 컬럼을 무시합니다 | 아니요 | **새 컬럼 데이터 유실**| 고정 스키마로 운영하는 경우 |

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

기존 스키마에 맞지 않는 데이터가 `_rescued_data` 컬럼에 JSON 형태로 저장됩니다. 스트림이 중단되지 않으므로 **데이터 유실 없이 안정적으로 운영** 할 수 있습니다.

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

Auto Loader의 타입 추론이 부정확할 때, 특정 컬럼의 타입을 **명시적으로 지정** 할 수 있습니다.

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

스키마에 맞지 않는 데이터를 **별도 컬럼에 보존** 하는 기능입니다. 데이터를 유실하지 않으면서도 파이프라인을 중단 없이 운영할 수 있습니다.

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

> 💡 **SDP에서의 자동 재시작**: SDP는 스키마 진화로 인한 스트림 중단 시 **자동으로 재시작** 합니다. 따라서 `addNewColumns` 모드를 사용하면 사람의 개입 없이 스키마가 자동으로 진화합니다.

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

## 현업 사례: JSON 스키마가 갑자기 바뀌어서 새벽 3시에 전화가 온 이야기

> 🔥 **데이터 엔지니어라면 한 번쯤 겪는 사고입니다.**

현업에서 가장 흔한 장애 시나리오를 살펴보겠습니다. 소스 시스템(모바일 앱, 외부 API 등)의 개발자가 JSON 스키마를 **사전 공지 없이 변경** 합니다. 이것이 데이터 파이프라인에 어떤 결과를 가져오는지, 어떻게 방어해야 하는지를 상세히 알아보겠습니다.

### 실제 장애 시나리오 타임라인

```
수요일 14:00 — 모바일 앱 팀이 주문 API에 새 필드 추가
  - 기존: {"order_id": 123, "amount": 5000, "user_id": "abc"}
  - 변경: {"order_id": 123, "amount": 5000, "user_id": "abc",
            "coupon_code": "SUMMER25", "discount_rate": 0.15}
  - 앱 팀은 "하위 호환이니까 데이터팀에 안 알려도 되겠지" 라고 판단

수요일 14:05 — Auto Loader가 새 파일을 읽기 시작
  - schemaEvolutionMode = "none" 이었다면 → 새 필드가 조용히 사라짐 (데이터 유실!)
  - schemaEvolutionMode = "failOnNewColumns" 이었다면 → 스트림 즉시 중단
  - schemaEvolutionMode = "addNewColumns" 이었다면 → 스트림 중단 후 재시작 시 자동 반영

수요일 14:10 — 더 심각한 상황: 타입 변경
  - 앱 팀이 amount를 정수에서 문자열로 변경 (쿠폰 적용 시 "5000원 - 750원 할인")
  - 기존: {"amount": 5000}
  - 변경: {"amount": "5000원 - 750원 할인"}
  - → inferColumnTypes로 BIGINT로 추론된 컬럼에 STRING이 들어옴
  - → 파싱 실패! 파이프라인 장애!

목요일 새벽 03:00 — 하루치 데이터가 없다는 알림
  - 마케팅팀의 일간 매출 대시보드가 빈 값을 보여줌
  - 당직 엔지니어에게 전화가 옴 📞
```

### 이 사고를 방지하는 방어 전략

현업에서는 "스키마가 절대 안 바뀔 거야"라는 가정은 하지 않습니다. **반드시 바뀐다고 가정하고** 방어적으로 파이프라인을 설계해야 합니다.

---

## rescuedDataColumn 실전 활용 패턴

`rescuedDataColumn`은 단순히 "구조되지 않은 데이터를 저장하는 컬럼"이 아닙니다. 현업에서는 이 컬럼을 **스키마 변경 감지 시스템** 으로 활용합니다.

### 패턴 1: rescued_data 모니터링 알림 구축

```sql
-- rescued_data에 데이터가 쌓이기 시작하면 = 스키마가 변경되었다는 신호
-- 이 쿼리를 DBSQL Alert로 등록하세요

SELECT
    DATE(ingested_at) AS dt,
    COUNT(*) AS total_rows,
    COUNT(_rescued_data) AS rescued_rows,
    ROUND(COUNT(_rescued_data) * 100.0 / COUNT(*), 2) AS rescued_pct
FROM catalog.bronze.orders
WHERE ingested_at >= CURRENT_DATE() - INTERVAL 1 DAY
GROUP BY 1;

-- rescued_pct가 0%에서 갑자기 올라가면 → 소스 스키마가 변경된 것!
-- 알림 조건: rescued_pct > 0
```

### 패턴 2: rescued_data 내용 분석으로 새 필드 발견

```sql
-- rescued_data에 어떤 필드가 들어오는지 분석
SELECT
    json_key,
    COUNT(*) AS occurrences,
    MIN(ingested_at) AS first_seen,
    MAX(ingested_at) AS last_seen
FROM (
    SELECT
        explode(map_keys(from_json(_rescued_data,
            'MAP<STRING, STRING>'))) AS json_key,
        _ingested_at AS ingested_at
    FROM catalog.bronze.orders
    WHERE _rescued_data IS NOT NULL
)
GROUP BY json_key
ORDER BY occurrences DESC;

-- 결과 예시:
-- | json_key      | occurrences | first_seen          | last_seen           |
-- |---------------|-------------|---------------------|---------------------|
-- | coupon_code   | 15,230      | 2025-03-15 14:05:00 | 2025-03-15 23:59:00 |
-- | discount_rate | 15,230      | 2025-03-15 14:05:00 | 2025-03-15 23:59:00 |
```

### 패턴 3: rescued_data에서 복구하여 정식 컬럼으로 승격

```sql
-- Silver 레이어에서 rescued_data를 파싱하여 정식 컬럼으로 추출
CREATE OR REFRESH STREAMING TABLE silver_orders AS
SELECT
    *,
    -- rescued_data에서 새 필드를 추출
    COALESCE(
        coupon_code,
        get_json_object(_rescued_data, '$.coupon_code')
    ) AS coupon_code_final,
    COALESCE(
        discount_rate,
        CAST(get_json_object(_rescued_data, '$.discount_rate') AS DOUBLE)
    ) AS discount_rate_final
FROM STREAM(catalog.bronze.orders);
```

---

## schemaHints로 안전망 구축하기

`schemaHints`는 "추론이 틀려도 괜찮게" 만드는 안전장치입니다. 현업에서는 **핵심 비즈니스 컬럼** 에 반드시 schemaHints를 적용합니다.

### 실전 schemaHints 설계 원칙

```python
# ❌ 위험한 설정: schemaHints 없이 모든 것을 추론에 맡김
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/")
    .load("s3://bucket/data/")
)
# → 첫 번째 파일에 amount가 5000(정수)이면 BIGINT로 추론
# → 나중에 amount가 5000.50(소수)이면 타입 불일치로 rescued_data로 빠짐

# ✅ 안전한 설정: 핵심 컬럼에 schemaHints 적용
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaHints", """
        order_id BIGINT,
        amount DECIMAL(18,4),
        user_id STRING,
        created_at TIMESTAMP,
        items ARRAY<STRUCT<product_id:STRING, qty:INT, price:DECIMAL(12,2)>>
    """)
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/")
    .option("rescuedDataColumn", "_rescued_data")
    .load("s3://bucket/data/")
)
```

### schemaHints를 적용해야 하는 컬럼 유형

| 컬럼 유형 | 위험 | schemaHints 예시 |
|----------|------|-----------------|
| **금액/가격**| 정수로 추론되면 소수점 이하 유실 | `"amount DECIMAL(18,4)"` |
| **ID 컬럼**| 숫자 ID가 나중에 UUID로 변경될 수 있음 | `"order_id STRING"` (가장 안전) |
| **날짜/시간**| 포맷이 일관되지 않으면 STRING으로 추론 | `"created_at TIMESTAMP"` |
| **전화번호**| 숫자로 추론되면 앞자리 0이 사라짐 | `"phone STRING"` |
| **우편번호**| 숫자로 추론되면 00123이 123이 됨 | `"zip_code STRING"` |

> 💡 **20년 경험에서 나온 규칙**: "이 컬럼의 타입이 잘못 추론되면 비즈니스에 영향이 있는가?" 라고 자문하세요. 답이 "예"이면 반드시 schemaHints를 적용합니다. **모든 컬럼에 힌트를 줄 필요는 없지만, 돈/시간/식별자 컬럼은 예외 없이 지정하세요.**

---

## 스키마 진화 모드 선택: 실전 의사결정

현업에서 어떤 `schemaEvolutionMode`를 써야 하는지 자주 질문을 받습니다. 정답은 **소스의 성격** 에 따라 다릅니다.

| 소스 유형 | 권장 모드 | 이유 |
|----------|----------|------|
| **자사 백엔드 API**| `addNewColumns` | 스키마 변경을 통제할 수 있고, 새 필드 추가가 자연스러움 |
| **외부 벤더 API**| `rescue` | 스키마 변경을 통제할 수 없으므로, 일단 받아두고 검토 |
| **IoT 센서 데이터**| `rescue` + 모니터링 | 펌웨어 업데이트로 필드가 갑자기 바뀌는 경우가 많음 |
| **규제 대상 데이터**(금융, 의료) | `failOnNewColumns` | 스키마 변경 시 반드시 사람이 검토해야 함 |
| **로그 데이터**| `addNewColumns` | 필드가 자주 추가되고, 유연성이 중요함 |

### 가장 안전한 프로덕션 조합 (현업 추천)

```python
# 20년 경험에서 나온 "황금 조합"
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    # 1. 타입 추론 활성화 (편의성)
    .option("cloudFiles.inferColumnTypes", "true")
    # 2. 핵심 컬럼은 schemaHints로 고정 (안전성)
    .option("cloudFiles.schemaHints",
            "order_id STRING, amount DECIMAL(18,4), created_at TIMESTAMP")
    # 3. 새 컬럼은 자동 추가 (유연성)
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    # 4. 파싱 실패 데이터는 rescued_data에 보존 (데이터 무결성)
    .option("rescuedDataColumn", "_rescued_data")
    # 5. 스키마 위치 고유하게 설정
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/orders_v1/")
    .load("s3://bucket/raw/orders/")
)
```

> 💡 **현업 팁**: 이 조합을 사용하면 새벽 3시에 전화받는 횟수가 **대폭** 줄어듭니다. 스키마가 변경되어도 파이프라인이 멈추지 않고, rescued_data 모니터링으로 다음 날 출근해서 차분하게 대응할 수 있습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **schemaLocation**| 추론된 스키마를 저장하는 경로입니다. 스트림별로 고유해야 합니다 |
| **inferColumnTypes**| `true`로 설정하면 데이터를 분석하여 최적 타입을 추론합니다 |
| **schemaEvolutionMode**| 새 컬럼 감지 시 동작을 결정합니다 (addNewColumns, rescue, failOnNewColumns, none) |
| **schemaHints**| 특정 컬럼의 타입을 명시적으로 지정하여 추론을 보정합니다 |
| **rescuedDataColumn** | 스키마에 맞지 않는 데이터를 별도 컬럼에 보존하여 유실을 방지합니다 |

---

## 참고 링크

- [Databricks: Auto Loader schema inference](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema.html)
- [Databricks: Schema evolution with Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema.html#schema-evolution)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options.html)
- [Azure Databricks: Schema inference and evolution](https://learn.microsoft.com/en-us/azure/databricks/ingestion/cloud-object-storage/auto-loader/schema)
