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
| **Broadcast Hash Join**| 한쪽 < 100MB | 없음 | ★★★★★ |
| **Sort Merge Join**| 양쪽 모두 대용량 | 양쪽 Shuffle | ★★★☆☆ |
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
