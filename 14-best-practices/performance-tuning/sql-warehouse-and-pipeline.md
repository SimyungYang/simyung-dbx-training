# SQL Warehouse와 파이프라인 최적화

## 3. SQL Warehouse 성능 최적화

### 3.1 쿼리 프로필 분석

SQL Warehouse의 Query Profile에서 주목해야 할 핵심 지표입니다.

| 단계 | 확인 항목 | 최적화 방향 |
|------|----------|-----------|
| **Scan** | 읽은 바이트 수 vs 테이블 크기 | 비율 높으면 클러스터링 부족 |
| **Filter** | 필터 전후 행 수 비율 | 비율 높으면 프루닝 미작동 |
| **Join** | 조인 유형 (Broadcast vs SortMerge) | 작은 테이블은 Broadcast로 |
| **Aggregate** | 집계 전후 행 수 | Pre-aggregate 고려 |
| **Exchange (Shuffle)** | Shuffle 데이터량 | 조인 키 최적화, 클러스터링 |
| **Spill** | Disk Spill 발생 여부 | Warehouse 사이즈 업 |

### 3.2 인덱스 없이 성능 높이기

Databricks는 전통적인 B-Tree 인덱스 대신 다음 기법들로 성능을 달성합니다.

```sql
-- 1. Liquid Clustering: 쿼리 패턴에 맞는 데이터 배치
ALTER TABLE catalog.schema.orders
CLUSTER BY (order_date, customer_id);

-- 2. 통계 수집: 옵티마이저에게 정확한 정보 제공
ANALYZE TABLE catalog.schema.orders COMPUTE DELTA STATISTICS;

-- 3. Materialized View: 복잡한 집계 사전 계산
CREATE MATERIALIZED VIEW catalog.schema.mv_daily_orders AS
SELECT order_date, region, COUNT(*) AS order_count, SUM(amount) AS total
FROM catalog.schema.orders
GROUP BY order_date, region;

-- 4. 적절한 데이터 타입 사용
-- ❌ 나쁜 예: 날짜를 문자열로 저장
-- CREATE TABLE ... (order_date STRING, ...)
-- ✅ 좋은 예: 네이티브 날짜 타입 사용 (프루닝/통계 활용)
-- CREATE TABLE ... (order_date DATE, ...)
```

### 3.3 쿼리 캐싱 전략

```text
요청 → Result Cache 확인 → 히트? → 즉시 반환 (0초)
                        ↓ 미스
        Disk Cache 확인 → 히트? → 로컬 SSD에서 읽기 (빠름)
                        ↓ 미스
        리모트 스토리지 (S3/ADLS) 에서 읽기 (느림)
```

| 캐시 유형 | 유효 기간 | 무효화 조건 | 설정 방법 |
|----------|----------|-----------|----------|
| **Result Cache** | 기본 활성화 | 테이블 데이터 변경 시 자동 무효화 | 자동 (수동 비활성화 가능) |
| **Disk Cache** | Warehouse 재시작까지 | Warehouse 재시작 또는 만료 | Storage-optimized 인스턴스 자동 |

```sql
-- Result Cache 확인: 동일 쿼리 재실행 시 Query Profile에서
-- "Result Cache Hit" 표시 여부 확인

-- Disk Cache 상태 확인 (노트북에서)
-- spark.catalog.isCached("table_name")

-- Cache를 최대 활용하는 쿼리 패턴
-- ✅ 동일한 쿼리를 파라미터만 바꿔 실행 (Result Cache 히트율 향상)
-- ✅ 자주 조인하는 디멘션 테이블은 자동으로 Disk Cache에 로드
-- ❌ CURRENT_TIMESTAMP() 등 비결정적 함수 사용 시 Cache 무효화
```

---

## 4. Query Profile 읽는 법 상세

### 4.1 Query Profile 핵심 지표

```text
[Query Profile 읽기 순서]

1. 상단 요약 확인
   - Total Execution Time: 전체 실행 시간
   - Rows Returned: 반환 행 수
   - Data Read: 읽은 데이터량

2. DAG(유향 비순환 그래프) 확인
   - 각 노드(연산)의 실행 시간 비율
   - 가장 넓은(시간 비중 큰) 노드가 병목

3. 상세 지표 확인
   - Rows In vs Rows Out: 필터링 효과
   - Shuffle Bytes: 네트워크 전송량
   - Spill Bytes: 디스크 스필량
```

### 4.2 Query Profile 실전 분석 사례

```text
[사례 1: 스캔 병목]
Query Profile:
  Scan: 읽은 바이트 = 50GB, 테이블 크기 = 50GB
  Filter: 입력 행 = 10억, 출력 행 = 100만

진단: 50GB 전체를 읽고 1%만 사용 → 프루닝 미작동
해결:
  1. CLUSTER BY (필터 컬럼) 적용
  2. OPTIMIZE 실행
  3. ANALYZE TABLE로 통계 갱신

[사례 2: Shuffle 병목]
Query Profile:
  Exchange (Shuffle Write): 20GB
  Join Type: SortMergeJoin

진단: 양쪽 테이블 모두 Shuffle → 네트워크 비용 높음
해결:
  1. 작은 테이블(< 100MB)이면 BROADCAST 힌트 추가
  2. 조인 키에 Liquid Clustering 적용
  3. 중간 결과를 Materialized View로 사전 계산

[사례 3: Skew 병목]
Query Profile:
  Stage Duration: 최소 2초, 최대 180초, 중앙값 5초

진단: 1개 태스크가 180초 (나머지 대비 36배) → 데이터 편향
해결:
  1. AQE Skew Join 설정 확인 (skewedPartitionFactor 낮추기)
  2. 조인 키의 NULL 값 사전 필터링
  3. 극심한 Skew → Salting 기법 적용
```

---

## 5. Query History로 병목 분석

### 5.1 시스템 테이블 활용

```sql
-- 가장 비용이 높은 쿼리 Top 20 (최근 7일)
SELECT
  statement_id,
  executed_by,
  statement_text,
  total_duration_ms,
  rows_produced,
  read_bytes,
  shuffle_write_bytes,
  spill_to_disk_bytes,
  warehouse_id
FROM system.query.history
WHERE start_time >= current_date() - INTERVAL 7 DAYS
ORDER BY total_duration_ms DESC
LIMIT 20;

-- 시간대별 쿼리 패턴 분석
SELECT
  date_trunc('hour', start_time) AS hour,
  COUNT(*) AS query_count,
  AVG(total_duration_ms) / 1000 AS avg_duration_sec,
  SUM(read_bytes) / 1024 / 1024 / 1024 AS total_read_gb
FROM system.query.history
WHERE start_time >= current_date() - INTERVAL 7 DAYS
GROUP BY 1
ORDER BY 1;
```

### 5.2 반복 최적화 대상 쿼리 찾기

```sql
-- 동일 쿼리가 반복 실행되는 패턴 (MV 후보)
SELECT
  statement_text,
  COUNT(*) AS execution_count,
  AVG(total_duration_ms) / 1000 AS avg_sec,
  SUM(read_bytes) / 1024 / 1024 / 1024 AS total_read_gb
FROM system.query.history
WHERE start_time >= current_date() - INTERVAL 7 DAYS
AND status = 'FINISHED'
GROUP BY statement_text
HAVING COUNT(*) >= 10  -- 10회 이상 반복
ORDER BY total_read_gb DESC
LIMIT 10;
-- 이 쿼리들은 Materialized View로 전환하면 비용 대폭 절감

-- Spill이 발생한 쿼리 (메모리 부족)
SELECT
  statement_id,
  executed_by,
  LEFT(statement_text, 200) AS query_preview,
  total_duration_ms / 1000 AS duration_sec,
  spill_to_disk_bytes / 1024 / 1024 AS spill_mb
FROM system.query.history
WHERE spill_to_disk_bytes > 0
AND start_time >= current_date() - INTERVAL 7 DAYS
ORDER BY spill_to_disk_bytes DESC
LIMIT 10;
```

{% hint style="info" %}
**Query History 활용 팁**: 주 1회 Query History를 분석하여 상위 10개 비용 쿼리를 최적화하면, 전체 SQL Warehouse 비용의 20~40%를 절감할 수 있습니다.
{% endhint %}

---

## 6. 파이프라인 성능 최적화

### 6.1 SDP(Spark Declarative Pipelines) 처리량 최적화

```python
# SDP 파이프라인 성능 설정
import dlt

@dlt.table(
  comment="최적화된 실버 테이블",
  table_properties={
    "pipelines.autoOptimize.managed": "true",      # 자동 최적화
    "pipelines.autoOptimize.zOrderCols": "user_id"  # 자동 Z-ORDER (레거시)
  }
)
def silver_events():
    return (
        dlt.read_stream("bronze_events")
        .withWatermark("event_time", "1 hour")  # 늦은 데이터 처리 한도
        .dropDuplicatesWithinWatermark(["event_id"])  # 중복 제거
    )
```

| 설정 | 기본값 | 성능 팁 |
|------|-------|--------|
| **pipelines.trigger.interval** | `"once"` 또는 연속 | 배치: `"once"`, 실시간: `"5 seconds"` |
| **Serverless 스케일링** | 자동 | Enhanced Autoscaling이 워크로드에 맞게 자동 조절 |
| **테이블 속성** | 기본값 | Liquid Clustering 적용 시 읽기 성능 대폭 향상 |

### 6.2 Auto Loader 파일 감지 모드 선택

| 모드 | 동작 방식 | 장점 | 단점 | 적합한 상황 |
|------|----------|------|------|-----------|
| **Directory Listing** | 주기적 디렉토리 스캔 | 설정 간단, 추가 인프라 불필요 | 파일 수 증가 시 스캔 비용 증가 | 파일 수 < 100만, 간단한 구조 |
| **File Notification** | SNS/SQS 이벤트 기반 | 즉시 감지, 대규모에 효율적 | 클라우드 리소스 설정 필요 | 파일 수 > 100만, 실시간 요구 |

```python
# Directory Listing 모드 (기본, 간단한 설정)
df = (spark.readStream
  .format("cloudFiles")
  .option("cloudFiles.format", "json")
  .option("cloudFiles.schemaLocation", "/checkpoints/schema")
  .load("s3://bucket/incoming/")
)

# File Notification 모드 (대규모, 실시간)
df = (spark.readStream
  .format("cloudFiles")
  .option("cloudFiles.format", "json")
  .option("cloudFiles.useNotifications", "true")  # 이벤트 기반
  .option("cloudFiles.schemaLocation", "/checkpoints/schema")
  .load("s3://bucket/incoming/")
)
```

### 6.3 Structured Streaming 지연시간 최적화

| 지연시간 목표 | 트리거 설정 | 적합한 워크로드 |
|-------------|-----------|---------------|
| **< 1초** | `trigger(processingTime="0 seconds")` | 실시간 대시보드, 알림 |
| **1~10초** | `trigger(processingTime="5 seconds")` | 준실시간 분석 |
| **10초~5분** | `trigger(processingTime="1 minute")` | 마이크로 배치 ETL |
| **5분+** | `trigger(availableNow=True)` | 스케줄 기반 배치 |

```python
# 지연시간 최적화 설정 조합
(df.writeStream
  .trigger(processingTime="5 seconds")
  .option("checkpointLocation", "/checkpoints/my_stream")
  .option("maxFilesPerTrigger", "100")        # 트리거당 파일 수 제한
  .option("maxBytesPerTrigger", "50m")         # 트리거당 데이터량 제한
  .outputMode("append")
  .toTable("catalog.schema.realtime_events")
)
```

---

## 7. SDP 성능 튜닝 상세

### 7.1 파이프라인 클러스터 사이징

| 파이프라인 유형 | 데이터 규모 | 권장 구성 | 비고 |
|-------------|-----------|----------|------|
| **소규모 배치** | < 10GB/실행 | Serverless (기본) | 관리 불필요 |
| **중규모 배치** | 10~100GB/실행 | Serverless Enhanced | 자동 스케일링 |
| **대규모 배치** | 100GB+/실행 | Classic, 8+ workers | Spot 활용 |
| **실시간 스트리밍** | 연속 | Serverless | 자동 확장/축소 |

```python
# SDP에서 Liquid Clustering 적용
import dlt

@dlt.table(
  comment="클러스터링된 골드 테이블",
  cluster_by=["order_date", "region"],  # Liquid Clustering
  table_properties={
    "quality": "gold"
  }
)
def gold_daily_revenue():
    return (
        dlt.read("silver_orders")
        .groupBy("order_date", "region")
        .agg(
            F.sum("amount").alias("total_revenue"),
            F.countDistinct("customer_id").alias("unique_customers")
        )
    )
```

### 7.2 SDP Expectations 성능 영향

| Expectation 유형 | 성능 영향 | 설명 |
|----------------|---------|------|
| **EXPECT** | 낮음 | 위반 행을 기록하되 통과시킴 |
| **EXPECT OR DROP** | 낮음~중간 | 위반 행을 필터링 (추가 연산) |
| **EXPECT OR FAIL** | 낮음 | 위반 시 파이프라인 중단 |
| **복잡한 Expectation** | 중간~높음 | 서브쿼리, JOIN 포함 시 추가 비용 |

```python
# 성능 친화적 Expectation 패턴
@dlt.table
@dlt.expect_or_drop("valid_amount", "amount > 0 AND amount < 1000000")
@dlt.expect("valid_email", "email RLIKE '^[a-zA-Z0-9._%+-]+@'")
def silver_orders():
    return dlt.read_stream("bronze_orders")

# ❌ 성능 비친화적: 외부 테이블 JOIN으로 유효성 검사
# @dlt.expect("valid_product", "product_id IN (SELECT id FROM dim_products)")
# → 매 마이크로배치마다 서브쿼리 실행 → 비용 높음

# ✅ 대안: Broadcast 변수로 유효 목록 캐싱
valid_products = spark.table("dim_products").select("id").collect()
valid_set = {row.id for row in valid_products}
# UDF로 검증
```

---

## 8. 파이프라인 관측 가능성 (Observability)

### 8.1 SDP 이벤트 로그 활용

```sql
-- SDP 파이프라인 실행 이력 조회
SELECT
  id,
  origin.pipeline_name,
  event_type,
  message,
  timestamp,
  details
FROM event_log(TABLE(pipeline_id))
WHERE event_type IN ('flow_progress', 'planning', 'driver_exception')
ORDER BY timestamp DESC
LIMIT 50;

-- 파이프라인 처리량 모니터링
SELECT
  origin.pipeline_name,
  origin.flow_name,
  details:flow_progress:metrics:num_output_rows AS output_rows,
  details:flow_progress:metrics:num_output_bytes AS output_bytes,
  timestamp
FROM event_log(TABLE(pipeline_id))
WHERE event_type = 'flow_progress'
AND timestamp >= current_date() - INTERVAL 7 DAYS
ORDER BY timestamp DESC;
```

### 8.2 모니터링 대시보드 구축

| 모니터링 지표 | 데이터 소스 | Alert 조건 |
|-------------|-----------|-----------|
| **파이프라인 실행 시간** | Event Log | 평균 대비 2배 초과 |
| **데이터 품질 위반** | Expectation 결과 | 위반율 > 5% |
| **파일 수 증가** | DESCRIBE DETAIL | numFiles > 임계값 |
| **스트리밍 지연** | Streaming Metrics | 처리 지연 > 5분 |
| **실패/재시도** | Event Log | 실패 발생 시 즉시 알림 |

```python
# 스트리밍 파이프라인 지연시간 모니터링
# Structured Streaming의 진행 상황 확인
query = (
    df.writeStream
    .trigger(processingTime="10 seconds")
    .option("checkpointLocation", "/checkpoints/stream")
    .queryName("my_stream")
    .toTable("catalog.schema.events")
)

# 스트리밍 상태 확인
progress = query.lastProgress
if progress:
    input_rows = progress['numInputRows']
    processing_time = progress['batchDuration']
    print(f"입력 행: {input_rows}, 처리 시간: {processing_time}ms")
```

{% hint style="info" %}
**관측 가능성 모범 사례**: AI/BI Dashboard로 파이프라인 모니터링 대시보드를 구축하고, Alert를 설정하여 이상 발생 시 Slack으로 즉시 알림을 받으세요. 시스템 테이블 (`system.query.history`, Event Log)을 데이터 소스로 활용합니다.
{% endhint %}

---

## 참고 링크

- [Databricks: Query Profile](https://docs.databricks.com/aws/en/optimizations/query-profile)
- [Databricks: Query History](https://docs.databricks.com/aws/en/sql/admin/query-history)
- [Databricks: SDP Performance](https://docs.databricks.com/aws/en/delta-live-tables/performance)
- [Databricks: Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/index)
- [Databricks: Streaming Metrics](https://docs.databricks.com/aws/en/structured-streaming/stream-monitoring)
- [Databricks: Event Log](https://docs.databricks.com/aws/en/delta-live-tables/observability)

---
