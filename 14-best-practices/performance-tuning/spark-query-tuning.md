# Spark 쿼리 튜닝

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

> 💡 **비용 대비 효과**: Photon 인스턴스는 DBU 단가가 약 2배이지만, 실행 시간이 3~5배 빨라져서 **총 비용이 오히려 40~60% 절감** 되는 경우가 많습니다.

---

## 3. AQE (Adaptive Query Execution) 심화

### 3.1 AQE 핵심 기능

AQE는 쿼리 실행 중 런타임 통계를 기반으로 실행 계획을 동적으로 최적화합니다.

| AQE 기능 | 동작 | 효과 | 설정 |
|---------|------|------|------|
| **동적 파티션 병합** | 작은 파티션을 런타임에 병합 | Small File → 적절한 크기 | `coalescePartitions.enabled = true` |
| **동적 조인 전환** | SortMerge → Broadcast 자동 전환 | Shuffle 제거 | `localShuffleReader.enabled = true` |
| **Skew Join 처리** | 편향된 파티션 자동 분할 | 특정 태스크 병목 해소 | `skewJoin.enabled = true` |
| **동적 파티션 수 조정** | 실제 데이터 크기에 맞게 파티션 수 조정 | 과다/과소 파티션 방지 | `advisoryPartitionSizeInBytes` |

```sql
-- AQE 상세 설정 (Databricks에서는 대부분 기본 활성화)
SET spark.sql.adaptive.enabled = true;                    -- AQE 활성화 (기본)
SET spark.sql.adaptive.coalescePartitions.enabled = true; -- 파티션 병합 (기본)
SET spark.sql.adaptive.localShuffleReader.enabled = true; -- 로컬 셔플 (기본)
SET spark.sql.adaptive.skewJoin.enabled = true;           -- Skew 처리 (기본)

-- 파티션 크기 조정 (기본 64MB)
SET spark.sql.adaptive.advisoryPartitionSizeInBytes = 128m;  -- 대용량 데이터에서 128MB

-- 파티션 병합 최소 크기
SET spark.sql.adaptive.coalescePartitions.minPartitionSize = 1m;  -- 1MB 미만 병합
```

### 3.2 AQE와 Shuffle Partition 튜닝

```text
[AQE 없이 수동 튜닝]
  spark.sql.shuffle.partitions = 200 (기본값)
  → 데이터가 1TB면 파티션당 5GB → 메모리 부족 (Spill)
  → 데이터가 100MB면 파티션당 0.5MB → 과다 파티션 (오버헤드)
  → 수동으로 워크로드마다 조정 필요

[AQE 활성화]
  spark.sql.shuffle.partitions = auto
  → 런타임에 실제 데이터 크기를 보고 자동 조정
  → advisoryPartitionSizeInBytes 기준으로 병합/분할
  → 수동 튜닝 불필요
```

{% hint style="info" %}
**AQE 최대 활용 팁**: `spark.sql.shuffle.partitions` 을 높게 설정(예: 2000)하고 AQE에게 병합을 맡기세요. AQE는 파티션을 나눌 수는 없고 **병합만** 가능합니다. 초기 파티션 수가 너무 적으면 AQE가 최적화할 여지가 없습니다.
{% endhint %}

---

## 4. 데이터 스큐 처리 전략 상세

### 4.1 Skew 진단 방법

```python
# 방법 1: 조인 키의 분포 확인
key_distribution = (
    orders
    .groupBy("customer_id")
    .count()
    .orderBy("count", ascending=False)
)

# 상위 10개 키 확인
key_distribution.show(10)
# customer_id | count
# NULL        | 5,000,000  ← NULL 키가 대량! (Skew 원인)
# VIP_001     | 2,000,000  ← 대형 고객
# VIP_002     | 1,500,000

# 방법 2: Query Profile에서 확인
# Task Duration 분포를 보면 특정 태스크만 10x 이상 오래 걸림
```

### 4.2 Skew 해결 전략 비교

| 전략 | 적용 조건 | 구현 복잡도 | 효과 |
|------|----------|-----------|------|
| **AQE Skew Join** | 대부분의 Skew | 자동 (설정만) | ★★★★☆ |
| **NULL 키 사전 필터링** | NULL이 대량인 경우 | 낮음 | ★★★★★ |
| **Salting** | 극심한 Skew (AQE 부족) | 중간 | ★★★★★ |
| **Repartition** | 특정 키로 편향 | 낮음 | ★★★☆☆ |
| **Broadcast Join** | 한쪽이 작은 경우 | 낮음 | ★★★★★ |

```python
# NULL 키 사전 필터링 (가장 간단하고 효과적)
# NULL 키는 조인에 매칭되지 않으므로, 사전에 제거
orders_filtered = orders.filter("customer_id IS NOT NULL")
result = orders_filtered.join(customers, "customer_id")

# 분리 처리 패턴 (NULL도 결과에 필요한 경우)
orders_with_customer = (
    orders.filter("customer_id IS NOT NULL")
    .join(customers, "customer_id")
)
orders_without_customer = (
    orders.filter("customer_id IS NULL")
    .withColumn("customer_name", lit("Unknown"))
)
result = orders_with_customer.union(orders_without_customer)
```

### 4.3 Repartition 전략

```python
# Repartition: 데이터를 특정 키로 균등 분배
# 조인 전에 양쪽 테이블을 동일 키로 repartition하면 Shuffle 감소

# 적합한 경우: 동일 키로 여러 번 조인하는 경우
orders_repartitioned = orders.repartition(200, "customer_id")
customers_repartitioned = customers.repartition(200, "customer_id")

# 첫 번째 조인
result1 = orders_repartitioned.join(customers_repartitioned, "customer_id")
# 두 번째 조인 (이미 같은 키로 파티셔닝되어 있으므로 Shuffle 없음)
result2 = orders_repartitioned.join(another_table, "customer_id")
```

---

## 5. Broadcast Join 임계값 튜닝

### 5.1 임계값 조정 가이드

```sql
-- 기본 임계값: 10MB
-- Databricks에서는 자동 Broadcast를 위한 임계값을 조정할 수 있음

-- 보수적 설정 (메모리가 제한적인 경우)
SET spark.sql.autoBroadcastJoinThreshold = 10m;

-- 적극적 설정 (메모리 여유가 있는 경우, 권장)
SET spark.sql.autoBroadcastJoinThreshold = 100m;

-- 매우 적극적 설정 (Memory-optimized 인스턴스)
SET spark.sql.autoBroadcastJoinThreshold = 500m;

-- Broadcast 비활성화 (특수한 경우, 예: OOM 발생 시)
SET spark.sql.autoBroadcastJoinThreshold = -1;
```

| Executor 메모리 | 권장 Broadcast 임계값 | 비고 |
|---------------|-------------------|------|
| 4GB 이하 | 10~30MB | 기본값 유지 |
| 4~16GB | 30~100MB | 중간 크기 디멘션 테이블 |
| 16~64GB | 100~500MB | 대부분의 디멘션 테이블 |
| 64GB 이상 | 500MB~1GB | 매우 큰 디멘션도 Broadcast |

{% hint style="warning" %}
**OOM 주의**: Broadcast 임계값을 너무 크게 설정하면 Driver 또는 Executor에서 OutOfMemoryError가 발생할 수 있습니다. Broadcast되는 테이블은 각 Executor의 메모리에 복사되므로, Executor 메모리의 30%를 넘지 않도록 설정하세요.
{% endhint %}

---

## 6. Spark UI로 실행 계획 읽는 법

### 6.1 핵심 지표 해석

| Spark UI 탭 | 확인 항목 | 의미 |
|-----------|---------|------|
| **SQL 탭** | DAG Visualization | 쿼리의 물리적 실행 계획 |
| **Stages 탭** | Input/Output Size | 단계별 데이터 이동량 |
| **Tasks 탭** | Duration Distribution | 태스크별 실행 시간 분포 (Skew 확인) |
| **Executors 탭** | GC Time, Spill | 리소스 사용 현황 |

### 6.2 실행 계획 읽기 순서

```sql
-- EXPLAIN으로 실행 계획 확인
EXPLAIN FORMATTED
SELECT /*+ BROADCAST(d) */
  f.order_id, f.revenue, d.product_name
FROM fact_orders f
JOIN dim_product d ON f.product_id = d.product_id
WHERE f.order_date >= '2026-01-01';
```

```text
[실행 계획 읽기 (아래에서 위로)]

== Physical Plan ==
*(3) Project [order_id, revenue, product_name]        ← 5. 최종 컬럼 선택
+- *(3) BroadcastHashJoin [product_id], [product_id]  ← 4. Broadcast Join 실행
   :- *(3) Filter (order_date >= 2026-01-01)          ← 3. 필터 적용
   :  +- *(3) ColumnarToRow                           ← 2. Photon → Row 변환
   :     +- PhotonScan [order_id, revenue, ...]       ← 1. 팩트 테이블 스캔
   +- BroadcastExchange                               ← Broadcast 배포
      +- PhotonScan dim_product [product_id, ...]     ← 디멘션 테이블 스캔

확인 포인트:
✅ BroadcastHashJoin 사용됨 (SortMergeJoin이 아님)
✅ PhotonScan 사용됨 (Photon 엔진 활성)
✅ Filter가 Scan에 가깝게 위치 (일찍 필터링 = 효율적)
```

---

## 7. 자주 발생하는 성능 문제 Top 5 + 해결법

### Top 1: SELECT * 사용

```sql
-- ❌ 문제: 불필요한 컬럼까지 모두 읽기 (넓은 테이블에서 치명적)
SELECT * FROM catalog.schema.wide_table WHERE date = '2026-03-01';

-- ✅ 해결: 필요한 컬럼만 선택
SELECT order_id, customer_id, amount
FROM catalog.schema.wide_table WHERE date = '2026-03-01';
-- 100+ 컬럼 테이블에서 3개 컬럼만 선택하면 97% I/O 절감
```

### Top 2: 비효율적 조인 순서

```sql
-- ❌ 문제: 큰 테이블끼리 먼저 조인
SELECT *
FROM fact_orders f           -- 10억 행
JOIN fact_returns r ON f.order_id = r.order_id  -- 1억 행
JOIN dim_product d ON f.product_id = d.product_id;  -- 10만 행

-- ✅ 해결: 작은 테이블(디멘션)을 먼저 조인
SELECT /*+ BROADCAST(d) */ *
FROM fact_orders f
JOIN dim_product d ON f.product_id = d.product_id  -- Broadcast (10만 행)
JOIN fact_returns r ON f.order_id = r.order_id;
```

### Top 3: 데이터 타입 불일치

```sql
-- ❌ 문제: 조인 키의 데이터 타입이 다름 (암묵적 캐스팅 → 인덱스 무효)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
-- orders.customer_id: STRING, customers.customer_id: INT
-- → 암묵적으로 CAST 발생 → Data Skipping 무효화

-- ✅ 해결: 명시적 타입 일치
SELECT * FROM orders o
JOIN customers c ON CAST(o.customer_id AS INT) = c.customer_id;
-- 또는 소스에서 타입을 통일
```

### Top 4: 파티션 과다 (Over-partitioning)

```sql
-- ❌ 문제: 고카디널리티 컬럼으로 파티셔닝 (100만+ 파티션)
-- CREATE TABLE ... PARTITIONED BY (user_id)  -- 100만 유저 → 100만 파티션!

-- ✅ 해결: Liquid Clustering으로 전환
ALTER TABLE catalog.schema.user_events
CLUSTER BY (event_date, user_id);
-- 파티셔닝 없이도 효과적인 Data Skipping
```

### Top 5: Collect/toPandas 대용량 데이터

```python
# ❌ 문제: 대용량 DataFrame을 Driver로 수집 → OOM
df = spark.table("catalog.schema.big_table")  # 10GB
pandas_df = df.toPandas()  # Driver 메모리 부족!

# ✅ 해결 1: 필터/집계 후 수집
summary = (
    df.filter("date >= '2026-03-01'")
    .groupBy("region")
    .agg({"revenue": "sum"})
)
pandas_df = summary.toPandas()  # 작은 결과만 수집

# ✅ 해결 2: Spark에서 처리 완료 후 저장
df.filter("date >= '2026-03-01'").write.mode("overwrite").saveAsTable("result_table")
```

{% hint style="info" %}
**성능 문제 진단 순서**: 1) Query Profile에서 가장 느린 단계 확인 → 2) Scan이 느리면 클러스터링/통계 확인 → 3) Shuffle이 크면 조인 전략 확인 → 4) Spill이 있으면 메모리/파티션 확인 → 5) Skew가 있으면 키 분포 확인.
{% endhint %}

---

## 참고 링크

- [Databricks: AQE](https://docs.databricks.com/aws/en/optimizations/aqe)
- [Databricks: Photon](https://docs.databricks.com/aws/en/compute/photon)
- [Databricks: Query Profile](https://docs.databricks.com/aws/en/optimizations/query-profile)
- [Databricks: Broadcast Join](https://docs.databricks.com/aws/en/optimizations/join-optimization)
- [Databricks: Skew Join](https://docs.databricks.com/aws/en/optimizations/skew-join)
- [Spark: Adaptive Query Execution](https://spark.apache.org/docs/latest/sql-performance-tuning.html#adaptive-query-execution)

---
