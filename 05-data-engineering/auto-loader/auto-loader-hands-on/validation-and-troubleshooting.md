# 실습 4: 데이터 검증과 트러블슈팅

## 목적과 학습 목표

파이프라인이 실행된 후 수집된 데이터가 올바른지 검증하고, 실제 운영 환경에서 발생하는 장애를 진단하고 해결하는 방법을 익힙니다.

### 학습 목표

| 목표 | 설명 |
|------|------|
| **종합 품질 리포트** | Bronze/Silver/Gold 전 계층의 데이터 정합성을 한 번에 확인합니다 |
| **체크포인트 관리** | 손상된 체크포인트를 안전하게 리셋하는 방법을 익힙니다 |
| **스키마 충돌 해결** | 타입 불일치, 컬럼 삭제 등 스키마 충돌 상황을 처리합니다 |
| **에러 패턴 진단** | 일반적인 에러 메시지와 원인, 해결 방법을 이해합니다 |
| **성능 진단** | 수집 지연, 메모리 압박 등 성능 문제를 진단합니다 |

---

## 사전 준비

[실습 1~3](README.md) 을 완료한 후 진행합니다. 다음 테이블이 존재해야 합니다:
- `training.auto_loader_lab.bronze_orders`
- `training.auto_loader_lab.bronze_customers`
- `training.auto_loader_lab.silver_orders` (실습 3)
- `training.auto_loader_lab.silver_customers` (실습 3)

---

## 데이터 검증

### 검증 체크리스트

| 검증 항목 | 확인 방법 | 통과 기준 |
|-----------|-----------|-----------|
| **행 수 정합** | 원본 파일 행 수 vs 테이블 행 수 비교 | 완전 일치 또는 허용 범위 내 |
| **중복 없음** | 기본 키 `GROUP BY`로 중복 확인 | 중복 레코드 0건 |
| **NULL 비율** | 필수 컬럼 NULL 비율 | 0% (필수 컬럼), 허용 범위 내 |
| **`_rescued_data`** | 파싱 실패 건수 확인 | 0건 또는 허용 범위 내 |
| **타입 정합** | `DESCRIBE`로 컬럼 타입 확인 | 예상 타입과 일치 |
| **Silver 필터링** | Bronze 행 수 - Silver 행 수 = 품질 위반 건수 | 예상 범위 내 |
| **Gold 집계 정합** | Silver 합계 = Gold 합계 | 일치 |

---

### 종합 데이터 품질 리포트

```sql
-- ==============================================
-- Bronze 계층 품질 리포트
-- ==============================================
SELECT
    'bronze_orders' AS table_name,
    COUNT(*)                                                           AS total_rows,
    COUNT(DISTINCT order_id)                                           AS unique_keys,
    COUNT(DISTINCT _source_file)                                       AS source_files,
    SUM(CASE WHEN _rescued_data IS NOT NULL THEN 1 ELSE 0 END)         AS rescued_rows,
    SUM(CASE WHEN order_id IS NULL          THEN 1 ELSE 0 END)         AS null_key_rows,
    SUM(CASE WHEN amount IS NULL OR amount <= 0 THEN 1 ELSE 0 END)     AS invalid_amount_rows,
    MIN(_ingested_at)                                                  AS first_ingestion,
    MAX(_ingested_at)                                                  AS last_ingestion
FROM training.auto_loader_lab.bronze_orders

UNION ALL

SELECT
    'bronze_customers',
    COUNT(*),
    COUNT(DISTINCT customer_id),
    COUNT(DISTINCT _source_file),
    SUM(CASE WHEN _rescued_data IS NOT NULL THEN 1 ELSE 0 END),
    SUM(CASE WHEN customer_id IS NULL       THEN 1 ELSE 0 END),
    SUM(CASE WHEN email IS NULL             THEN 1 ELSE 0 END),
    MIN(_ingested_at),
    MAX(_ingested_at)
FROM training.auto_loader_lab.bronze_customers;
```

---

### Bronze → Silver 데이터 손실 분석

```sql
-- Bronze에서 Silver로 승격되지 않은 행의 수와 이유를 분석합니다
SELECT
    'orders' AS domain,
    b.total_bronze,
    s.total_silver,
    (b.total_bronze - s.total_silver) AS dropped_rows,
    ROUND((b.total_bronze - s.total_silver) * 100.0 / b.total_bronze, 2) AS drop_rate_pct
FROM
    (SELECT COUNT(*) AS total_bronze FROM training.auto_loader_lab.bronze_orders) b,
    (SELECT COUNT(*) AS total_silver FROM training.auto_loader_lab.silver_orders) s;
```

```sql
-- 드롭된 행의 원인 분석 (Bronze에는 있지만 Silver에는 없는 행)
-- 참고: 이 쿼리는 실습 데이터의 규모에서는 빠르게 실행됩니다.
-- 대규모 프로덕션에서는 SDP UI의 Data Quality 탭을 사용하세요.
SELECT
    CASE
        WHEN order_id IS NULL   THEN 'NULL order_id'
        WHEN amount <= 0        THEN 'Invalid amount (<= 0)'
        WHEN status NOT IN ('COMPLETED', 'PENDING', 'CANCELLED') THEN 'Invalid status: ' || status
        WHEN _rescued_data IS NOT NULL THEN 'Parse error'
        ELSE 'Other'
    END AS drop_reason,
    COUNT(*) AS count
FROM training.auto_loader_lab.bronze_orders
WHERE order_id NOT IN (SELECT order_id FROM training.auto_loader_lab.silver_orders WHERE order_id IS NOT NULL)
GROUP BY 1
ORDER BY count DESC;
```

---

### Gold 집계 정합성 검증

```sql
-- Silver의 합계와 Gold의 합계가 일치하는지 확인합니다
SELECT
    'Silver total revenue' AS metric,
    SUM(amount) AS value
FROM training.auto_loader_lab.silver_orders
WHERE status = 'COMPLETED'

UNION ALL

SELECT
    'Gold sum of completed_revenue',
    SUM(completed_revenue)
FROM training.auto_loader_lab.gold_daily_sales;
-- 두 값이 같아야 합니다
```

```sql
-- 날짜별 Silver vs Gold 비교
SELECT
    s.order_date,
    s.silver_count,
    g.order_count AS gold_count,
    s.silver_revenue,
    g.total_revenue AS gold_revenue,
    (s.silver_count - g.order_count) AS count_diff
FROM (
    SELECT order_date, COUNT(*) AS silver_count, SUM(amount) AS silver_revenue
    FROM training.auto_loader_lab.silver_orders
    GROUP BY order_date
) s
LEFT JOIN training.auto_loader_lab.gold_daily_sales g
    ON s.order_date = g.order_date
ORDER BY s.order_date;
```

---

### `_rescued_data` 심층 분석

`_rescued_data` 컬럼에 데이터가 있다면 원인을 파악합니다.

```sql
-- _rescued_data가 있는 행의 원본 내용 확인
SELECT
    _source_file,
    _rescued_data,
    COUNT(*) AS count
FROM training.auto_loader_lab.bronze_orders
WHERE _rescued_data IS NOT NULL
GROUP BY _source_file, _rescued_data
ORDER BY count DESC
LIMIT 20;
```

```python
# rescued_data를 파싱하여 어떤 필드가 문제인지 확인
from pyspark.sql.functions import from_json, col, schema_of_json

# _rescued_data 샘플 수집
rescued_samples = spark.sql("""
    SELECT _rescued_data
    FROM training.auto_loader_lab.bronze_orders
    WHERE _rescued_data IS NOT NULL
    LIMIT 5
""").collect()

for row in rescued_samples:
    print(row['_rescued_data'])
# 출력 예: {"order_id":"ABC", "amount": "not_a_number"}
# → order_id가 STRING으로 도착했거나, amount가 숫자가 아님을 알 수 있습니다
```

---

## 트러블슈팅

### 체크포인트 리셋

체크포인트가 손상되었거나 처음부터 다시 처리해야 할 때 사용합니다.

```python
# ⚠️ 주의: 체크포인트 삭제 전 반드시 아래 사항을 확인하세요:
# 1. 대상 테이블을 함께 초기화하지 않으면 데이터가 중복됩니다.
# 2. 파일 수가 많으면 전체 재처리에 오랜 시간이 걸립니다.

# [안전한 방법] 테이블 + 체크포인트 + 스키마를 함께 초기화합니다
def reset_pipeline(table_name, checkpoint_path, schema_path):
    """Auto Loader 파이프라인을 안전하게 초기화합니다."""
    print(f"[1/3] 테이블 삭제: {table_name}")
    spark.sql(f"DROP TABLE IF EXISTS {table_name}")

    print(f"[2/3] 체크포인트 삭제: {checkpoint_path}")
    dbutils.fs.rm(checkpoint_path, recurse=True)

    print(f"[3/3] 스키마 캐시 삭제: {schema_path}")
    dbutils.fs.rm(schema_path, recurse=True)

    print("초기화 완료. 스트림을 다시 시작하면 모든 파일을 처음부터 처리합니다.")

# 사용 예
reset_pipeline(
    table_name      = "training.auto_loader_lab.bronze_orders",
    checkpoint_path = "/Volumes/training/auto_loader_lab/checkpoints/csv_orders/",
    schema_path     = "/Volumes/training/auto_loader_lab/schema/csv_orders/"
)
```

> **체크포인트를 삭제하면 벌어지는 일**: Auto Loader는 "처음 본 디렉토리"로 인식하여 모든 파일을 다시 읽습니다. `APPEND` 모드로 쓰고 있다면 데이터가 중복됩니다. 반드시 대상 테이블도 함께 초기화하거나, `MERGE INTO` (멱등 처리) 로직을 사용하세요.

---

### 스키마 충돌 해결

```python
# 문제 1: 스키마 덮어쓰기가 필요한 경우 (기존 추론 스키마가 잘못된 경우)
df_fixed = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.allowOverwrites", "true")   # 기존 스키마 파일 덮어쓰기 허용
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    # 타입이 잘못 추론된 컬럼을 힌트로 교정
    .option("cloudFiles.schemaHints",
            "order_id BIGINT, customer_id BIGINT, amount DOUBLE, order_date DATE")
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
```

```python
# 문제 2: 컬럼 타입이 BIGINT → STRING으로 변경된 경우
# 해결: Bronze에서는 모두 STRING으로 수집하고, Silver에서 CAST합니다
df_all_string = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path + "_v2/")  # 새 스키마 경로
    .option("cloudFiles.inferColumnTypes", "false")             # 모두 STRING으로 수집
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
# Silver에서: CAST(order_id AS BIGINT) → 변환 실패 시 NULL이 되고 CONSTRAINT로 필터링됩니다
```

**스키마 관련 문제 대응 매트릭스**:

| 상황 | 원인 | 해결 방법 |
|------|------|-----------|
| 새 컬럼 추가됨 | 소스 스키마 확장 | `schemaEvolutionMode=addNewColumns` + `mergeSchema=true` |
| 기존 컬럼 타입 변경 | 소스 타입 변경 | `schemaHints`로 타입 고정 또는 Bronze를 전체 STRING으로 수집 |
| 기존 컬럼 삭제됨 | 소스에서 컬럼 제거 | Bronze 스키마 유지 (NULL로 채워짐), Silver에서 처리 |
| 스키마 추론 결과가 잘못됨 | 샘플 편향 | `schemaHints`로 핵심 컬럼 타입 명시 |
| 스키마 파일이 손상됨 | 체크포인트 오류 | `cloudFiles.allowOverwrites=true` + 스키마 경로 삭제 후 재실행 |

---

### 일반적인 에러와 해결 방법

| 에러 메시지 | 원인 | 해결 방법 |
|-------------|------|-----------|
| `PERMISSION_DENIED on volume/path` | UC Volume 또는 S3 접근 권한 부족 | Unity Catalog 권한 (`GRANT READ VOLUME`) 또는 IAM 역할 확인 |
| `AnalysisException: path does not exist` | 소스 경로가 없음 | 경로 오타 확인, Volume이 생성되었는지 확인 |
| `StreamingQueryException: ... checkpoint` | 체크포인트 손상 또는 버전 불호환 | 체크포인트 삭제 후 재시작 |
| `DELTA_SCHEMA_CHANGE_SINCE_ANALYSIS` | 스트리밍 중 스키마 변경 감지 | `mergeSchema=true` 옵션 추가 |
| `MALFORMED_RECORD_IN_PARSING` | JSON/CSV 파싱 불가능한 레코드 | `rescuedDataColumn` 또는 `badRecordsPath` 설정 |
| `SchemaInferenceException: new columns found` | `failOnNewColumns` 모드에서 새 컬럼 감지 | 스키마 업데이트 또는 `addNewColumns` 모드로 변경 |
| `OutOfMemoryError` | 트리거당 처리 파일이 너무 많음 | `maxFilesPerTrigger` 또는 `maxBytesPerTrigger` 설정 |
| `OperationNotAllowedException: Cannot overwrite` | APPEND 모드에서 덮어쓰기 시도 | `outputMode("append")`와 체크포인트 일관성 확인 |

---

### 성능 진단

```python
# 스트리밍 쿼리 상태 확인 (실행 중인 경우)
# Spark UI의 Structured Streaming 탭에서도 확인 가능합니다
for query in spark.streams.active:
    print(f"쿼리 이름: {query.name}")
    print(f"상태: {query.status}")
    recent = query.recentProgress
    if recent:
        last = recent[-1]
        print(f"마지막 배치 입력 행수: {last.get('numInputRows', 'N/A')}")
        print(f"마지막 배치 처리 시간: {last.get('batchDuration', 'N/A')}ms")
    print("---")
```

```sql
-- Delta 테이블 최적화 (수집 후 파일 통합)
-- 소규모 파일이 많이 생성되었을 때 읽기 성능을 개선합니다
OPTIMIZE training.auto_loader_lab.bronze_orders;

-- 통계 갱신 (쿼리 최적화에 사용됩니다)
ANALYZE TABLE training.auto_loader_lab.bronze_orders COMPUTE STATISTICS;
```

```python
# 성능 튜닝: 트리거당 처리량 제어
# 파일이 매우 많거나 크다면 아래 옵션으로 배치 크기를 조절합니다
df_tuned = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    # 성능 튜닝 옵션
    .option("cloudFiles.maxFilesPerTrigger", "500")    # 트리거당 최대 500파일
    .option("cloudFiles.maxBytesPerTrigger", "1g")     # 트리거당 최대 1GB
    .option("cloudFiles.fetchParallelism",  "4")       # 파일 목록 조회 병렬도
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
```

---

### 파이프라인 모니터링 자동화

프로덕션에서는 데이터 품질을 자동으로 모니터링하는 뷰를 생성합니다.

```sql
-- 데이터 신선도 모니터링 뷰 (마지막 수집이 얼마나 오래 됐는지)
CREATE OR REPLACE VIEW training.auto_loader_lab.v_pipeline_health AS
SELECT
    'bronze_orders' AS table_name,
    MAX(_ingested_at) AS last_ingested_at,
    DATEDIFF(MINUTE, MAX(_ingested_at), current_timestamp()) AS minutes_since_last_ingestion,
    COUNT(*) AS total_rows
FROM training.auto_loader_lab.bronze_orders

UNION ALL

SELECT
    'bronze_customers',
    MAX(_ingested_at),
    DATEDIFF(MINUTE, MAX(_ingested_at), current_timestamp()),
    COUNT(*)
FROM training.auto_loader_lab.bronze_customers;

-- 30분 이상 수집이 없으면 알람을 발생시킬 수 있습니다
SELECT *,
    CASE WHEN minutes_since_last_ingestion > 30 THEN '⚠️ 지연 감지' ELSE '✅ 정상' END AS health_status
FROM training.auto_loader_lab.v_pipeline_health;
```

---

## 실습 정리 (리소스 삭제)

실습이 끝나면 생성한 리소스를 정리합니다.

```sql
-- 테이블 및 뷰 삭제
DROP TABLE  IF EXISTS training.auto_loader_lab.bronze_orders;
DROP TABLE  IF EXISTS training.auto_loader_lab.bronze_customers;
DROP TABLE  IF EXISTS training.auto_loader_lab.silver_orders;
DROP TABLE  IF EXISTS training.auto_loader_lab.silver_customers;
DROP VIEW   IF EXISTS training.auto_loader_lab.gold_daily_sales;
DROP VIEW   IF EXISTS training.auto_loader_lab.gold_customer_by_city;
DROP VIEW   IF EXISTS training.auto_loader_lab.v_pipeline_health;

-- 볼륨 삭제 (파일 포함)
DROP VOLUME IF EXISTS training.auto_loader_lab.raw_data;

-- 스키마 삭제 (CASCADE: 남은 객체 모두 삭제)
DROP SCHEMA IF EXISTS training.auto_loader_lab CASCADE;
```

```python
# 체크포인트 및 스키마 캐시 정리
dbutils.fs.rm("/Volumes/training/auto_loader_lab/", recurse=True)
print("모든 실습 리소스 정리 완료!")
```

---

## 정리

### 전체 실습 학습 포인트 요약

| 실습 | 주요 학습 내용 |
|------|---------------|
| **실습 1: CSV 수집** | `cloudFiles` 포맷으로 CSV 수집, Checkpoint 기반 증분 처리, `_metadata` 컬럼 활용 |
| **실습 2: JSON + 스키마 진화** | JSON Lines 수집, `schemaEvolutionMode=addNewColumns`, 중첩 구조 Flatten |
| **실습 3: SDP 통합** | `read_files()`로 Medallion 파이프라인, `CONSTRAINT`로 데이터 품질 선언, Streaming Table vs Materialized View |
| **실습 4: 검증 & 트러블슈팅** | 종합 품질 리포트, 체크포인트 안전 리셋, 스키마 충돌 해결, 에러 패턴 진단 |

### Auto Loader 운영 핵심 원칙

1. **체크포인트는 소중하다**: 절대 단독으로 삭제하지 마세요. 테이블과 함께 초기화해야 합니다.
2. **`_rescued_data`는 필수**: 설정하지 않으면 스키마 불일치 데이터가 조용히 사라집니다.
3. **Bronze는 날것 그대로**: 변환 없이 수집하고, `_source_file`과 `_ingested_at`을 항상 추가합니다.
4. **SDP와 함께 사용**: 단독 사용 대신 SDP를 활용하면 체크포인트/스키마 관리가 자동화됩니다.
5. **`availableNow=True`로 비용 절감**: 상시 실행이 필요 없다면 이 트리거로 배치 처리합니다.

---

## 참고 링크

- [Databricks: Auto Loader tutorial](https://docs.databricks.com/aws/en/ingestion/auto-loader/tutorial.html)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html)
- [Databricks: Auto Loader schema inference](https://docs.databricks.com/aws/en/ingestion/auto-loader/schema.html)
- [Databricks: Auto Loader production best practices](https://docs.databricks.com/aws/en/ingestion/auto-loader/production.html)
- [Databricks: read_files in SDP](https://docs.databricks.com/aws/en/ingestion/auto-loader/unity-catalog.html)
- [Databricks: Streaming tables DDL](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-streaming-table.html)
- [Databricks: Delta Live Tables expectations](https://docs.databricks.com/aws/en/delta-live-tables/expectations.html)
