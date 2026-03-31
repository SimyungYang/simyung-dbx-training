# 성능 최적화 모범 사례

## 이 문서에서 다루는 내용

Databricks 플랫폼에서 쿼리, 파이프라인, ML 워크로드의 성능을 극대화하는 방법을 다룹니다. Delta Lake 데이터 레이아웃부터 Spark 쿼리 튜닝, SQL Warehouse 최적화, 파이프라인 처리량 개선, ML/AI 지연시간 최적화까지 실전 트러블슈팅 가이드를 제공합니다.

---

## 1. Delta Lake 성능 최적화

### 1.1 Liquid Clustering 키 선택 전략

Liquid Clustering은 기존의 파티셔닝과 Z-ORDER를 대체하는 현대적인 데이터 레이아웃 기술입니다. 올바른 키 선택이 성능을 좌우합니다.

**키 선택 의사결정 매트릭스**

| 고려 요소 | 좋은 키 | 나쁜 키 |
|----------|--------|--------|
| **쿼리 필터 빈도** | WHERE절에 자주 등장 | 거의 필터링되지 않음 |
| **카디널리티** | 중~고 (수천~수백만) | 극저 (boolean, 2-3값) 또는 극고 (UUID) |
| **상관관계** | 다른 키와 독립적 | 다른 키와 높은 상관 (둘 다 선택 불필요) |
| **데이터 타입** | Date, String, Integer | Array, Map, Struct |
| **키 개수** | 1~4개 | 5개 이상 (최대 4개 제한) |

```sql
-- 사례 1: 이커머스 트랜잭션 테이블
-- 쿼리 패턴: 날짜 범위 + 지역 필터가 대부분
ALTER TABLE sales.silver.transactions
CLUSTER BY (transaction_date, region);

-- 사례 2: IoT 센서 데이터
-- 쿼리 패턴: 디바이스별 + 시간 범위 조회
ALTER TABLE iot.silver.sensor_readings
CLUSTER BY (device_id, event_timestamp);

-- 사례 3: 쿼리 패턴이 자주 변경되는 테이블
-- Databricks가 자동으로 최적 키를 분석하여 결정
ALTER TABLE analytics.gold.user_behavior
CLUSTER BY AUTO;
```

**기존 테이블 마이그레이션 가이드**

| 현재 방식 | 마이그레이션 전략 |
|----------|----------------|
| `PARTITIONED BY (date)` | `CLUSTER BY (date)` — 파티션 키를 클러스터링 키로 |
| `ZORDER BY (col_a, col_b)` | `CLUSTER BY (col_a, col_b)` — Z-ORDER 키를 그대로 |
| 파티션 + Z-ORDER | `CLUSTER BY (partition_col, zorder_col)` — 둘 다 포함 |
| 없음 (최적화 안 함) | `CLUSTER BY AUTO` — 자동 분석 후 적용 |

> 💡 **핵심 장점**: Liquid Clustering은 키를 변경해도 **기존 데이터를 즉시 다시 쓸 필요가 없습니다**. 이후 OPTIMIZE 실행 시 점진적으로 새 키에 맞게 재배치됩니다.

### 1.2 파일 사이즈 최적화

Delta Lake의 최적 파일 크기는 **128MB ~ 1GB** 입니다. 이 범위를 벗어나면 성능이 저하됩니다.

| 문제 | 원인 | 증상 | 해결 방법 |
|------|------|------|----------|
| **Small File 문제** | 잦은 스트리밍 쓰기, 작은 배치 | 파일 수 과다, 메타데이터 오버헤드 | `OPTIMIZE` 실행 |
| **Large File 문제** | 과도한 OPTIMIZE, 큰 배치 | 프루닝 효과 감소, 메모리 압박 | 파일 크기 상한 설정 |

```sql
-- Small File 진단: 파일 수와 평균 크기 확인
DESCRIBE DETAIL catalog.schema.table_name;
-- numFiles, sizeInBytes를 확인하여 평균 파일 크기 계산
-- 평균 < 32MB면 OPTIMIZE 필요

-- OPTIMIZE로 파일 병합
OPTIMIZE catalog.schema.table_name;

-- 특정 파티션만 OPTIMIZE (대용량 테이블)
OPTIMIZE catalog.schema.table_name
WHERE event_date >= current_date() - INTERVAL 7 DAYS;
```

```python
# 스트리밍 쓰기 시 파일 크기 제어
(df.writeStream
  .option("maxBytesPerTrigger", "100m")  # 트리거당 최대 100MB 읽기
  .option("maxFilesPerTrigger", "1000")   # 트리거당 최대 파일 수
  .trigger(processingTime="30 seconds")   # 30초 간격 → 적당한 파일 크기
  .table("catalog.schema.streaming_table")
)
```

### 1.3 통계 수집 및 Data Skipping

Data Skipping은 쿼리 실행 시 불필요한 파일을 자동으로 건너뛰는 기능입니다. 정확한 통계 정보가 필수적입니다.

```sql
-- Delta 통계 재수집 (Runtime 14.3 LTS+)
ANALYZE TABLE catalog.schema.table_name COMPUTE DELTA STATISTICS;

-- 특정 컬럼에 대한 통계 수집
ALTER TABLE catalog.schema.table_name
SET TBLPROPERTIES('delta.dataSkippingStatsColumns' = 'event_date, user_id, region');

-- 통계 수집 대상 컬럼 수 확인 (기본: 처음 32개 컬럼)
-- Unity Catalog 관리형 테이블은 Predictive Optimization이 자동으로 관리
```

| 기능 | 동작 방식 | 성능 효과 |
|------|----------|----------|
| **Min/Max 통계** | 파일별 최소/최대값 기록 | WHERE절 조건으로 파일 단위 스킵 |
| **Null Count** | 파일별 NULL 수 기록 | IS NOT NULL 필터 최적화 |
| **Liquid Clustering** | 관련 데이터를 같은 파일에 배치 | Data Skipping 효과 극대화 |
| **ANALYZE TABLE** | 통계 강제 재수집 | 테이블 속성 변경 후 반드시 실행 |

---

## 2. Spark 쿼리 튜닝

### 2.1 Query Profile 읽는 법

Databricks SQL 및 노트북의 Query Profile은 성능 병목을 시각적으로 보여줍니다.

**병목 식별 우선순위**

```text
1단계: 스캔(Scan) 확인
  → 읽은 데이터량이 예상보다 크면 → 클러스터링/프루닝 부족

2단계: 셔플(Exchange) 확인
  → Shuffle Write/Read가 크면 → 조인 전략 변경 필요

3단계: 스필(Spill) 확인
  → Spill to Disk 발생하면 → 메모리 부족, 인스턴스 업그레이드

4단계: 스큐(Skew) 확인
  → 특정 태스크만 오래 걸리면 → 데이터 편향 해결 필요
```

### 2.2 Shuffle 최소화 전략

Shuffle은 네트워크를 통해 데이터를 재분배하는 가장 비싼 연산입니다.

**Broadcast Join (작은 테이블 조인)**

```sql
-- 방법 1: 힌트 사용 (한쪽 테이블이 작을 때)
SELECT /*+ BROADCAST(dim_product) */
  f.order_id,
  f.revenue,
  d.product_name,
  d.category
FROM fact_orders f
JOIN dim_product d ON f.product_id = d.product_id;

-- 방법 2: 자동 Broadcast 임계값 조정 (기본 10MB)
SET spark.sql.autoBroadcastJoinThreshold = 100m;  -- 100MB까지 자동 Broadcast
```

| 조인 전략 | 적용 조건 | Shuffle | 성능 |
|----------|----------|---------|------|
| **Broadcast Hash Join** | 한쪽 < 100MB | 없음 | ★★★★★ |
| **Sort Merge Join** | 양쪽 모두 대용량 | 양쪽 Shuffle | ★★★☆☆ |
| **Shuffle Hash Join** | 한쪽 중간 크기 | 한쪽 Shuffle | ★★★★☆ |

```python
# Python에서 Broadcast 힌트
from pyspark.sql.functions import broadcast

result = (
    fact_orders
    .join(broadcast(dim_product), "product_id")
    .select("order_id", "revenue", "product_name")
)
```

### 2.3 Skew 처리

데이터 편향(Skew)은 특정 키에 데이터가 집중되어 일부 태스크만 과도하게 오래 걸리는 현상입니다.

```sql
-- AQE(Adaptive Query Execution) 기반 자동 Skew 처리
-- Databricks에서는 기본 활성화 (spark.sql.adaptive.enabled = true)
-- 추가 설정으로 민감도 조정 가능

SET spark.sql.adaptive.skewJoin.enabled = true;            -- 기본 true
SET spark.sql.adaptive.skewJoin.skewedPartitionFactor = 5; -- 중앙값 대비 5배 이상이면 skew
SET spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes = 256m;  -- 256MB 이상이면 skew
```

```python
# Salting 기법 (극심한 Skew에 대한 수동 해결)
from pyspark.sql.functions import concat, lit, rand, floor

num_salt_buckets = 10

# 큰 테이블에 salt 추가
orders_salted = (
    orders
    .withColumn("salt", floor(rand() * num_salt_buckets))
    .withColumn("salted_key", concat("customer_id", lit("_"), "salt"))
)

# 작은 테이블을 salt 수만큼 복제
customers_exploded = (
    customers
    .crossJoin(spark.range(num_salt_buckets).withColumnRenamed("id", "salt"))
    .withColumn("salted_key", concat("customer_id", lit("_"), "salt"))
)

# Salted 조인 (Skew 분산)
result = orders_salted.join(customers_exploded, "salted_key")
```

### 2.4 Spill to Disk 진단 및 해결

Spill은 메모리 부족으로 데이터를 디스크에 임시 저장하는 현상으로, 성능을 크게 저하시킵니다.

| 진단 지표 | 확인 방법 | 임계값 |
|----------|----------|-------|
| Spill (Memory) | Query Profile Stage Details | 0보다 크면 주의 |
| Spill (Disk) | Query Profile Stage Details | 0보다 크면 경고 |
| GC Time | Executor Metrics | 총 시간의 10% 초과 시 위험 |

```python
# 해결 방법 1: 메모리 증가
# Memory-optimized 인스턴스로 전환 (r5, r6i 계열)

# 해결 방법 2: 파티션 수 증가 (데이터 분산)
spark.conf.set("spark.sql.shuffle.partitions", "400")  # 기본 200 → 400

# 해결 방법 3: 조인 전략 변경
# Sort Merge Join 대신 Broadcast Join 사용 (가능한 경우)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "200m")
```

### 2.5 Photon 활용 가이드

Photon은 Databricks의 벡터화된 쿼리 엔진으로, 특정 워크로드에서 큰 성능 향상을 제공합니다.

| 워크로드 | Photon 효과 | 권장 여부 |
|---------|-----------|----------|
| SQL 집계/조인 (대용량) | 2~8x 빠름 | ✅ 강력 추천 |
| Delta MERGE/UPDATE/DELETE | 2~5x 빠름 | ✅ 강력 추천 |
| 넓은 테이블 (100+ 컬럼) 스캔 | 2~4x 빠름 | ✅ 추천 |
| Small File이 많은 테이블 | 2~3x 빠름 | ✅ 추천 |
| 단순 SELECT (< 2초) | 거의 효과 없음 | ⚠️ 불필요 |
| Python UDF 중심 로직 | 효과 없음 | ❌ Photon 미지원 |
| RDD API, Dataset API | 미지원 | ❌ 사용 불가 |
| 스트리밍 (Stateful) | 미지원 | ❌ Stateless만 가능 |

```sql
-- Photon이 사용되고 있는지 확인
-- Query Profile에서 "PhotonGroupingScan", "PhotonShuffleExchange" 등의
-- Photon 연산자가 표시되면 Photon으로 실행 중

-- Photon 활성화 확인
SELECT current_catalog(), current_schema();
-- 클러스터 설정에서 runtime_engine = PHOTON 확인
```

> 💡 **비용 대비 효과**: Photon 인스턴스는 DBU 단가가 약 2배이지만, 실행 시간이 3~5배 빨라져서 **총 비용이 오히려 40~60% 절감**되는 경우가 많습니다.

---

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

## 4. 파이프라인 성능 최적화

### 4.1 SDP(Spark Declarative Pipelines) 처리량 최적화

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

### 4.2 Auto Loader 파일 감지 모드 선택

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

### 4.3 Structured Streaming 지연시간 최적화

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

## 5. ML/AI 성능 최적화

### 5.1 Model Serving 지연시간 최적화

| 최적화 영역 | 전략 | 효과 |
|-----------|------|------|
| **콜드 스타트** | Minimum Instances >= 1 | 첫 요청 지연 제거 |
| **모델 크기** | ONNX 변환, 양자화 | 로드 시간 50~80% 감소 |
| **GPU 선택** | 모델 크기에 맞는 GPU | 과도한 GPU는 비용 낭비 |
| **배치 요청** | Batch Inference 활용 | 개별 요청 대비 10~100x 처리량 |
| **Feature Lookup** | Online Table 활용 | Feature 조회 < 10ms |

```python
# Model Serving 엔드포인트 설정 (최적화 버전)
import mlflow

# 모델 등록 시 최적화 힌트
mlflow.pyfunc.log_model(
    artifact_path="model",
    python_model=my_model,
    pip_requirements=["scikit-learn==1.3.0"],  # 최소 의존성
    metadata={"serving_optimization": "latency"}
)
```

### 5.2 배치 추론 병렬화

```python
import mlflow
from pyspark.sql.functions import struct

# Spark UDF로 배치 추론 병렬화
model_uri = "models:/fraud_detection/production"
predict_udf = mlflow.pyfunc.spark_udf(spark, model_uri)

# 대용량 데이터셋에 병렬 추론 적용
predictions = (
    spark.table("catalog.schema.transactions")
    .withColumn("prediction", predict_udf(struct("feature1", "feature2", "feature3")))
)

# 최적화 팁: 파티션 수를 코어 수의 2~3배로 설정
predictions.repartition(spark.sparkContext.defaultParallelism * 2)
```

### 5.3 Vector Search 검색 속도 최적화

| 최적화 항목 | 권장 설정 | 이유 |
|-----------|----------|------|
| **임베딩 차원** | 768~1536 (모델에 따라) | 불필요하게 큰 차원은 속도 저하 |
| **인덱스 유형** | Delta Sync Index | 자동 동기화, 운영 부담 최소 |
| **엔드포인트 크기** | 데이터 크기에 비례 | 1M 문서 미만: Small, 이상: Medium+ |
| **필터링** | 메타데이터 필터 활용 | 검색 범위 축소로 속도/정확도 향상 |

```python
# Vector Search 쿼리 최적화
results = index.similarity_search(
    query_text="고객 이탈 예측 방법",
    columns=["content", "source", "page"],  # 필요한 컬럼만 반환
    filters={"category": "ml", "year": 2025},  # 메타데이터 필터로 범위 축소
    num_results=5  # 필요한 만큼만 요청
)
```

---

## 6. 성능 진단 체크리스트

단계별로 성능 문제를 진단하고 해결하는 가이드입니다.

### Step 1: 쿼리가 느린가?

```text
□ Query Profile에서 가장 시간이 오래 걸리는 단계는?
  → Scan이 오래 걸림 → Step 2로
  → Join이 오래 걸림 → Step 3으로
  → Aggregate가 오래 걸림 → Step 4로
```

### Step 2: 스캔 최적화

```text
□ 읽은 데이터량 vs 필요한 데이터량 비교
  → 과도한 스캔 → Liquid Clustering 적용했는가?
  → 클러스터링 적용됨 → OPTIMIZE를 최근에 실행했는가?
  → OPTIMIZE 실행됨 → 클러스터링 키가 쿼리 패턴과 맞는가?
□ ANALYZE TABLE로 통계를 수집했는가?
□ 파일 수가 과도하지 않은가? (Small File 문제)
```

### Step 3: 조인 최적화

```text
□ 조인 유형 확인 (Broadcast vs SortMerge)
  → 한쪽 테이블이 작은데 SortMerge? → Broadcast 힌트 추가
  → 양쪽 모두 대용량? → Shuffle Partition 수 조정
□ 데이터 편향(Skew) 존재하는가?
  → 특정 태스크만 오래 걸림 → AQE Skew Join 설정 확인
  → 극심한 Skew → Salting 기법 적용
□ 조인 키에 NULL이 많은가?
  → NULL 키는 사전 필터링
```

### Step 4: 집계/셔플 최적화

```text
□ Shuffle 데이터량이 과도한가?
  → Pre-aggregate 가능한가? (서브쿼리에서 먼저 집계)
  → Shuffle Partition 수 조정 (기본 200, 데이터에 따라 조절)
□ Spill to Disk 발생하는가?
  → 메모리 확대 (Memory-optimized 인스턴스)
  → Partition 수 증가로 단위 데이터 축소
□ Photon이 활성화되어 있는가?
  → SQL 집계 워크로드에 Photon 적용 시 2~8x 향상
```

### Step 5: 리소스 최적화

```text
□ 클러스터/Warehouse 사이즈가 적절한가?
  → CPU 사용률 > 80% 지속 → 스케일 업
  → 메모리 사용률 > 80% → Memory-optimized 인스턴스
  → I/O 대기 높음 → Storage-optimized 인스턴스
□ Autoscaling이 적절히 동작하는가?
  → Min/Max 워커 수 검토
□ 네트워크 병목이 있는가?
  → 리전 간 데이터 전송 최소화
  → 같은 리전에 컴퓨트와 스토리지 배치
```

### 빠른 참조: 증상별 해결 매트릭스

| 증상 | 1순위 확인 | 2순위 확인 | 3순위 확인 |
|------|----------|----------|----------|
| 쿼리 시작이 느림 | Warehouse Auto Stop 설정 | Instance Pool 사용 | Serverless 전환 |
| 스캔이 느림 | Liquid Clustering | 통계 수집 | 파일 크기 최적화 |
| 조인이 느림 | Broadcast 가능 여부 | Skew 확인 | 클러스터 사이즈 |
| 집계가 느림 | Photon 활성화 | MV 사전 계산 | Partition 수 조정 |
| Spill 발생 | 메모리 확대 | Partition 수 증가 | 쿼리 분리 |
| 스트리밍 지연 | 트리거 간격 | Checkpoint 위치 | 파일 수 제한 |
| ML 추론 느림 | 모델 최적화 | GPU 선택 | 배치 크기 조정 |

---

## 참고 링크

- [Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering)
- [Data Skipping](https://docs.databricks.com/aws/en/delta/data-skipping)
- [Photon 엔진](https://docs.databricks.com/aws/en/compute/photon)
- [클러스터 구성 모범 사례](https://docs.databricks.com/aws/en/compute/cluster-config-best-practices)
- [SQL Warehouse 사이징 가이드](https://docs.databricks.com/aws/en/compute/sql-warehouse/warehouse-behavior)
- [AQE(Adaptive Query Execution)](https://docs.databricks.com/aws/en/optimizations/aqe)
- [Predictive Optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
- [Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/index)
