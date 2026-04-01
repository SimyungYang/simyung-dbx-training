# CDC 처리 — APPLY CHANGES

## 왜 CDC가 필요한가?

운영 데이터베이스(OLTP)에서는 데이터가 끊임없이 INSERT, UPDATE, DELETE됩니다. 이러한 변경사항을 분석 시스템(데이터 레이크하우스)에 반영해야 하는데, 매번 전체 테이블을 복사하는 것은 비효율적입니다.

> 💡 **CDC(Change Data Capture)** 는 소스 데이터베이스에서 발생하는 변경사항만 감지하여 타겟 시스템에 반영하는 기술입니다. 전체 데이터를 매번 복사하는 대신, ** 변경된 부분만** 캡처하므로 네트워크 대역폭, 처리 시간, 비용을 크게 절감할 수 있습니다.

### 전체 복사 vs CDC 비교

| 비교 항목 | 전체 복사 (Full Load) | CDC (변경분만) |
|----------|---------------------|---------------|
| ** 데이터 전송량** | 전체 테이블 크기 | 변경분만 (전체의 1~5% 수준) |
| ** 처리 시간** | 테이블 크기에 비례 | 변경량에 비례 |
| ** 소스 부하** | 높음 (전체 스캔) | 낮음 (로그 기반) |
| ** 실시간성** | 낮음 (배치 주기에 의존) | 높음 (실시간/준실시간 가능) |
| ** 이력 추적** | 불가능 (최신 스냅샷만) | 가능 (변경 이력 보존) |

---

## CDC 데이터 흐름

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | 운영 DB (MySQL, PostgreSQL 등) | 변경 로그(Binlog, WAL)를 생성합니다 |
| 2 | CDC 도구 (Debezium, Lakeflow Connect) | CDC 이벤트(JSON/Avro)를 전달합니다 |
| 3 | Bronze (원본 CDC 레코드) | 원본 CDC 레코드를 저장합니다 |
| 4 | Silver (최신 상태 또는 이력) | `APPLY CHANGES INTO`로 최신 상태 또는 이력을 관리합니다 |
| 5 | Gold (비즈니스 뷰) | 비즈니스 요구에 맞는 뷰를 생성합니다 |

---

## SCD(Slowly Changing Dimension) 유형

CDC로 캡처한 변경사항을 타겟 테이블에 ** 어떻게 반영**할지 결정하는 전략을 SCD라고 합니다.

> 💡 **SCD(Slowly Changing Dimension)** 는 데이터 웨어하우스에서 "서서히 변하는 데이터"(예: 고객 주소, 직원 부서)를 어떻게 관리할지 정의하는 설계 패턴입니다. Ralph Kimball이 제시한 개념으로, Type 0부터 Type 4까지 다양한 유형이 있습니다.

### SCD 유형 비교

| 유형 | 전략 | 이력 보존 | 설명 | 예시 |
|------|------|----------|------|------|
| **Type 0** | 변경 금지 | 해당 없음 | 최초 값을 유지하며, 절대 변경하지 않습니다 | 고객 최초 가입일 |
| **Type 1** | 최신값 덮어쓰기 | ❌ | 기존 값을 새 값으로 교체합니다. 이전 값은 사라집니다 | 고객 현재 주소 |
| **Type 2** | 이력 추적 | ✅ | 변경 시 새 행을 추가하고, 기존 행을 만료 처리합니다 | 고객 주소 변경 이력 |
| **Type 3** | 이전/현재 컬럼 | 부분적 | 이전 값과 현재 값을 별도 컬럼에 저장합니다 | 이전 부서 + 현재 부서 |
| **Type 4** | 별도 이력 테이블 | ✅ | 현재 테이블과 이력 테이블을 분리합니다 | 현재 테이블 + 이력 테이블 |

> 💡 SDP에서는 가장 많이 사용되는 **Type 1** 과 **Type 2** 를 `APPLY CHANGES INTO` 구문으로 직접 지원합니다.

---

## APPLY CHANGES INTO 구문 상세

### 기본 구조

```sql
APPLY CHANGES INTO target_table
FROM source
KEYS (key_columns)           -- 레코드를 식별하는 기본 키
SEQUENCE BY sequence_column  -- 변경 순서를 결정하는 컬럼
[COLUMNS ...]                -- 포함/제외할 컬럼
[WHERE ...]                  -- 필터 조건
[APPLY AS DELETE WHEN ...]   -- 삭제 조건
[APPLY AS TRUNCATE WHEN ...] -- 전체 삭제 조건
STORED AS SCD TYPE {1|2}     -- SCD 유형
[TRACK HISTORY ON ...]       -- (Type 2) 이력 추적 대상 컬럼
```

### 주요 절(Clause) 설명

| 절 | 필수 여부 | 설명 |
|----|----------|------|
| `KEYS` | ✅ 필수 | 소스와 타겟에서 레코드를 매칭하는 키 컬럼입니다. 복합 키 가능합니다 |
| `SEQUENCE BY` | ✅ 필수 | 동일 키에 대한 여러 변경 이벤트의 순서를 결정합니다. 타임스탬프 또는 시퀀스 번호를 사용합니다 |
| `COLUMNS` | 선택 | `COLUMNS *`(전체), `COLUMNS * EXCEPT (...)`, `COLUMNS (a, b, c)` 형태로 지정합니다 |
| `WHERE` | 선택 | 소스 데이터에서 특정 조건의 레코드만 처리합니다 |
| `APPLY AS DELETE WHEN` | 선택 | 조건이 참인 레코드를 삭제로 처리합니다 |
| `APPLY AS TRUNCATE WHEN` | 선택 | 조건이 참이면 타겟 테이블 전체를 삭제합니다 |
| `STORED AS SCD TYPE` | ✅ 필수 | `1` (최신값 덮어쓰기) 또는 `2` (이력 추적)를 지정합니다 |
| `TRACK HISTORY ON` | 선택 | (Type 2 전용) 특정 컬럼이 변경될 때만 새 이력 행을 생성합니다 |

---

## SCD Type 1 구현: 최신값 덮어쓰기

SCD Type 1은 변경이 발생하면 기존 값을 **최신 값으로 교체** 합니다. 이력을 보존하지 않으며, 항상 **현재 상태** 만 유지합니다.

### SQL 예제

```sql
-- 1단계: 타겟 Streaming Table 생성
CREATE OR REFRESH STREAMING TABLE silver_customers;

-- 2단계: CDC 적용 (SCD Type 1)
APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS * EXCEPT (_rescued_data, _metadata)
STORED AS SCD TYPE 1;
```

### Python 예제

```python
import dlt

dlt.create_streaming_table("silver_customers")

@dlt.apply_changes(
    target="silver_customers",
    source="bronze_customer_cdc",
    keys=["customer_id"],
    sequence_by=col("updated_at"),
    except_column_list=["_rescued_data", "_metadata"],
    stored_as_scd_type=1
)
def apply_customer_changes():
    pass
```

### 동작 예시

소스 CDC 레코드:

| customer_id | name | city | updated_at | operation |
|-------------|------|------|-----------|-----------|
| 101 | 김철수 | 서울 | 2024-01-01 | INSERT |
| 101 | 김철수 | 부산 | 2024-03-15 | UPDATE |
| 101 | 김철수 | 대구 | 2024-06-20 | UPDATE |

SCD Type 1 결과 (silver_customers):

| customer_id | name | city | updated_at |
|-------------|------|------|-----------|
| 101 | 김철수 | **대구** | 2024-06-20 |

> 항상 최신 상태만 유지됩니다. "김철수가 서울에 살았다"는 이력은 사라집니다.

---

## SCD Type 2 구현: 이력 추적

SCD Type 2는 변경이 발생하면 기존 레코드를 만료 처리하고 ** 새 레코드를 추가**합니다. 전체 변경 이력이 보존됩니다.

### SQL 예제

```sql
-- 1단계: 타겟 Streaming Table 생성
CREATE OR REFRESH STREAMING TABLE silver_customers_history;

-- 2단계: CDC 적용 (SCD Type 2)
APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS * EXCEPT (_rescued_data, _metadata)
STORED AS SCD TYPE 2;
```

### 자동 생성되는 메타 컬럼

SCD Type 2로 저장하면 다음 시스템 컬럼이 자동으로 추가됩니다.

| 컬럼 | 데이터 타입 | 설명 |
|------|-----------|------|
| `__START_AT` | TIMESTAMP | 해당 버전의 유효 시작 시간 (SEQUENCE BY 컬럼의 값) |
| `__END_AT` | TIMESTAMP | 해당 버전의 유효 종료 시간. **NULL이면 현재 유효한 레코드** 입니다 |

### 동작 예시

소스 CDC 레코드 (동일한 데이터):

| customer_id | name | city | updated_at | operation |
|-------------|------|------|-----------|-----------|
| 101 | 김철수 | 서울 | 2024-01-01 | INSERT |
| 101 | 김철수 | 부산 | 2024-03-15 | UPDATE |
| 101 | 김철수 | 대구 | 2024-06-20 | UPDATE |

SCD Type 2 결과 (silver_customers_history):

| customer_id | name | city | __START_AT | __END_AT |
|-------------|------|------|-----------|---------|
| 101 | 김철수 | 서울 | 2024-01-01 | 2024-03-15 |
| 101 | 김철수 | 부산 | 2024-03-15 | 2024-06-20 |
| 101 | 김철수 | **대구** | 2024-06-20 | **NULL** |

> `__END_AT`이 NULL인 행이 ** 현재 유효한 최신 레코드**입니다.

### 현재 유효한 레코드만 조회

```sql
-- 현재 유효한 레코드만 필터링
SELECT *
FROM silver_customers_history
WHERE __END_AT IS NULL;

-- 특정 시점의 고객 정보 조회 (Time Travel 스타일)
SELECT *
FROM silver_customers_history
WHERE __START_AT <= '2024-02-01'
  AND (__END_AT > '2024-02-01' OR __END_AT IS NULL);
```

### TRACK HISTORY ON: 선택적 이력 추적

모든 컬럼 변경이 아닌, **특정 컬럼이 변경될 때만** 이력을 기록하고 싶다면 `TRACK HISTORY ON` 절을 사용합니다.

```sql
APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS * EXCEPT (_rescued_data)
STORED AS SCD TYPE 2
TRACK HISTORY ON (city, email);  -- city나 email이 변경될 때만 이력 생성
```

> 💡 `TRACK HISTORY ON *` (기본값)은 모든 컬럼 변경을 추적합니다. `TRACK HISTORY ON * EXCEPT (last_login_at)` 형태로 특정 컬럼을 제외할 수도 있습니다.

---

## DELETE 처리

### Soft Delete (삭제 표시만)

CDC 소스에서 DELETE 이벤트를 받으면, 타겟에서 해당 레코드를 실제로 삭제하는 대신 **삭제 표시** 만 할 수 있습니다. SCD Type 2에서는 `__END_AT`이 설정되어 만료 처리됩니다.

```sql
APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS * EXCEPT (_rescued_data)
APPLY AS DELETE WHEN operation = 'DELETE'
STORED AS SCD TYPE 2;
```

> SCD Type 2에서 DELETE를 적용하면, 해당 레코드의 `__END_AT`이 설정되어 "만료됨"으로 표시됩니다. 물리적으로 삭제되지는 않습니다.

### Hard Delete (물리적 삭제)

SCD Type 1에서 DELETE를 적용하면, 해당 레코드가 타겟 테이블에서 **물리적으로 삭제** 됩니다.

```sql
APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS * EXCEPT (_rescued_data)
APPLY AS DELETE WHEN operation = 'DELETE'
STORED AS SCD TYPE 1;
```

### DELETE 처리 비교

| 항목 | SCD Type 1 + DELETE | SCD Type 2 + DELETE |
|------|-------------------|-------------------|
| **물리적 삭제** | ✅ 행이 실제로 삭제됩니다 | ❌ 행은 유지됩니다 |
| ** 삭제 이력** | ❌ 추적 불가 | ✅ `__END_AT` 설정으로 추적 가능 |
| ** 복원 가능** | ❌ 원본 CDC 로그에서만 가능 | ✅ 이력에서 확인 가능 |

---

## 소스별 CDC 패턴

CDC 데이터를 생성하는 도구에 따라 이벤트 형식이 다릅니다.

### Debezium 형식 처리

[Debezium](https://debezium.io/)은 오픈소스 CDC 도구로, Kafka를 통해 변경 이벤트를 전달합니다.

```sql
-- Debezium CDC 이벤트 처리
APPLY CHANGES INTO silver_orders
FROM STREAM(bronze_debezium_orders)
KEYS (payload.after.order_id)
SEQUENCE BY payload.source.ts_ms
COLUMNS
    payload.after.order_id AS order_id,
    payload.after.customer_id AS customer_id,
    payload.after.amount AS amount,
    payload.after.status AS status,
    payload.after.updated_at AS updated_at
APPLY AS DELETE WHEN payload.op = 'd'
STORED AS SCD TYPE 1;
```

### Lakeflow Connect

Databricks의 관리형 커넥터인 Lakeflow Connect를 사용하면 CDC 데이터가 표준화된 형식으로 자동 수집됩니다.

```sql
-- Lakeflow Connect가 수집한 CDC 데이터 처리
APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_mysql_customers)
KEYS (customer_id)
SEQUENCE BY _metadata.ingestion_timestamp
COLUMNS * EXCEPT (_metadata, _rescued_data)
APPLY AS DELETE WHEN _change_type = 'delete'
STORED AS SCD TYPE 2;
```

### 커스텀 CDC (Application 레벨)

애플리케이션에서 직접 변경 로그를 생성하는 경우:

```sql
-- 커스텀 변경 로그 테이블 처리
APPLY CHANGES INTO silver_products
FROM STREAM(bronze_product_changelog)
KEYS (product_id)
SEQUENCE BY change_sequence_no
COLUMNS product_id, name, price, category, status
APPLY AS DELETE WHEN change_type = 'D'
WHERE change_type IN ('I', 'U', 'D')  -- INSERT, UPDATE, DELETE만 처리
STORED AS SCD TYPE 1;
```

---

## 실습: CDC 파이프라인 전체 예제

고객 데이터의 CDC 처리를 Bronze → Silver → Gold 전체 과정으로 구현하는 예제입니다.

```sql
-- ============================================
-- Bronze: 원본 CDC 이벤트 수집 (Auto Loader)
-- ============================================
CREATE OR REFRESH STREAMING TABLE bronze_customer_cdc
AS SELECT
    *,
    _metadata.file_path AS source_file,
    _metadata.file_modification_time AS file_time
FROM STREAM read_files(
    '/volumes/catalog/schema/landing/customers/',
    format => 'json',
    schema => '
        customer_id BIGINT,
        name STRING,
        email STRING,
        city STRING,
        phone STRING,
        operation STRING,
        updated_at TIMESTAMP
    '
);

-- ============================================
-- Silver (SCD Type 1): 고객 현재 상태
-- ============================================
CREATE OR REFRESH STREAMING TABLE silver_customers;

APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS customer_id, name, email, city, phone
APPLY AS DELETE WHEN operation = 'DELETE'
STORED AS SCD TYPE 1;

-- ============================================
-- Silver (SCD Type 2): 고객 이력
-- ============================================
CREATE OR REFRESH STREAMING TABLE silver_customers_history;

APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS customer_id, name, email, city, phone
APPLY AS DELETE WHEN operation = 'DELETE'
STORED AS SCD TYPE 2
TRACK HISTORY ON (city, email);  -- 주소, 이메일 변경만 추적

-- ============================================
-- Gold: 도시별 고객 분포 (현재 기준)
-- ============================================
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_by_city
AS SELECT
    city,
    COUNT(*) AS customer_count,
    COUNT(DISTINCT email) AS unique_emails
FROM silver_customers
GROUP BY city
ORDER BY customer_count DESC;

-- ============================================
-- Gold: 고객 이동 이력 분석
-- ============================================
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_city_changes
AS SELECT
    customer_id,
    name,
    city AS current_city,
    __START_AT AS moved_at,
    LAG(city) OVER (PARTITION BY customer_id ORDER BY __START_AT) AS previous_city
FROM silver_customers_history
WHERE __END_AT IS NULL OR __END_AT IS NOT NULL
ORDER BY customer_id, __START_AT;
```

---

## 성능 고려사항

### 대용량 CDC 처리 팁

| 항목 | 권장 사항 |
|------|----------|
| **파티셔닝** | 타겟 테이블에 적절한 파티션을 설정하면 MERGE 성능이 향상됩니다 |
| ** 키 선택** | 키 컬럼의 카디널리티가 높을수록 효율적입니다. 복합 키는 최소화합니다 |
| ** 시퀀스 컬럼** | 타임스탬프보다 단조 증가하는 시퀀스 번호가 더 안정적입니다 |
| ** 배치 크기** | Auto Loader의 `maxFilesPerTrigger` 또는 `maxBytesPerTrigger`로 배치 크기를 조절합니다 |
| ** 불필요한 컬럼 제외** | `COLUMNS * EXCEPT (...)`로 메타데이터 컬럼을 제외하여 저장 공간을 절약합니다 |

### 순서 보장

> ⚠️ `SEQUENCE BY` 컬럼은 동일한 키에 대해 **변경 순서를 올바르게 반영** 해야 합니다. 동일한 키에 대해 같은 시퀀스 값을 가진 여러 레코드가 있으면, 하나만 임의로 선택됩니다. 타임스탬프의 정밀도가 충분하지 않은 경우 별도의 시퀀스 번호 컬럼 사용을 권장합니다.

---

## 정리

| 핵심 포인트 | 설명 |
|------------|------|
| **CDC의 목적** | 소스 DB의 변경사항만 캡처하여 효율적으로 레이크하우스에 반영합니다 |
| **SCD Type 1** | 최신값으로 덮어쓰기. 이력 보존이 불필요한 경우에 사용합니다 |
| **SCD Type 2** | 이력 추적. `__START_AT`/`__END_AT` 컬럼으로 유효 기간을 관리합니다 |
| **APPLY CHANGES INTO** | SDP에서 CDC를 처리하는 핵심 구문입니다 |
| **SEQUENCE BY** | 동일 키에 대한 변경 순서를 보장합니다. 필수입니다 |
| **DELETE 처리** | Type 1은 물리적 삭제, Type 2는 만료 처리(soft delete)로 동작합니다 |

---

## 참고 링크

- [Databricks: APPLY CHANGES — CDC 처리](https://docs.databricks.com/aws/en/sdp/cdc.html)
- [Databricks: SCD Type 2 with SDP](https://docs.databricks.com/aws/en/sdp/cdc.html#scd-type-2)
- [Databricks: Lakeflow Connect](https://docs.databricks.com/aws/en/connect/index.html)
- [Debezium Documentation](https://debezium.io/documentation/)
- [Databricks Blog: Simplify CDC with SDP](https://www.databricks.com/blog/2022/04/25/simplifying-change-data-capture-with-databricks-delta-live-tables.html)
