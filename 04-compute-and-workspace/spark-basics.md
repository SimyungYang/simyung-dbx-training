# Apache Spark 기초

## Apache Spark란?

> 💡 **Apache Spark**는 대용량 데이터를 여러 대의 컴퓨터에 분산하여 **동시에 빠르게 처리**할 수 있는 오픈소스 분산 컴퓨팅 엔진입니다.

Spark는 **분산 처리(Distributed Processing)** 엔진입니다. 단일 서버로 10시간 걸리는 ETL 작업을 10대의 서버에 분산하면 1시간으로 줄어듭니다. Databricks는 이 Spark 위에 클러스터 관리, 모니터링, 최적화(Photon 엔진)를 추가한 관리형 플랫폼입니다.

현업에서 Spark를 써야 하는 가장 현실적인 이유는 **데이터 크기**입니다. 단일 서버의 메모리(보통 64~256GB)에 담을 수 없는 데이터를 처리해야 할 때 Spark가 필요합니다. pandas로 처리하던 데이터가 수십 GB를 넘어가기 시작하면, `MemoryError`가 발생하기 시작하고, 그때가 Spark로 전환해야 할 시점입니다. 반대로 데이터가 10GB 이하라면 pandas나 Polars가 오히려 빠를 수 있습니다 — Spark는 분산 환경의 오버헤드(네트워크, 직렬화)가 있기 때문입니다.

---

## Spark의 아키텍처: Driver와 Executor

Spark는 **Driver**와 **Executor**라는 두 가지 역할로 구성됩니다.

![Spark 클러스터 아키텍처 — Driver와 Executor](https://docs.databricks.com/en/_images/spark-cluster-overview.png)

> 출처: [Apache Spark 공식 문서 — Cluster Mode Overview](https://spark.apache.org/docs/latest/cluster-overview.html)

| 구성 요소 | 역할 | 비유 |
|-----------|------|------|
| **Driver** | 전체 작업을 계획하고, Executor에게 작업을 나눠줍니다. 최종 결과를 수집합니다 | 현장 감독. 누가 무슨 일을 할지 지시합니다 |
| **Executor** | 실제 데이터를 읽고, 변환하고, 계산하는 작업자입니다. 여러 대가 병렬로 동작합니다 | 작업자. 맡은 데이터를 처리하고 결과를 보고합니다 |
| **Cluster Manager** | Driver와 Executor에게 컴퓨팅 리소스(CPU, 메모리)를 할당합니다 | 인사부. 작업자를 배치합니다 |

> 💡 **노드(Node)란?** 클러스터를 구성하는 개별 컴퓨터(또는 가상 머신)를 말합니다. Driver가 실행되는 노드를 **Driver Node**, Executor가 실행되는 노드를 **Worker Node**라고 부릅니다.

---

## Spark의 핵심 개념: DataFrame

Spark에서 데이터를 다루는 가장 기본적인 구조는 **DataFrame**입니다.

> 💡 **DataFrame**은 행(Row)과 열(Column)로 구성된 분산 데이터 구조입니다. 엑셀의 시트나 SQL 테이블과 비슷하게 생겼지만, 내부적으로는 여러 Executor에 데이터가 나뉘어(Partition) 저장되어 있습니다.

```python
# DataFrame 생성 예시
df = spark.createDataFrame([
    ("김철수", 28, "서울", 4500000),
    ("이영희", 34, "부산", 5200000),
    ("박민수", 25, "대구", 3800000),
], ["이름", "나이", "도시", "연봉"])

# DataFrame 내용 확인
df.show()
# +------+---+----+-------+
# |  이름|나이|도시|   연봉|
# +------+---+----+-------+
# |김철수| 28|서울|4500000|
# |이영희| 34|부산|5200000|
# |박민수| 25|대구|3800000|
# +------+---+----+-------+
```

### 파티션(Partition) — 분산의 단위

> 💡 **파티션(Partition)** 이란 DataFrame의 데이터를 물리적으로 나눈 조각입니다. 각 Executor는 하나 이상의 파티션을 담당하여 병렬로 처리합니다.

| 파티션 | 데이터 | 처리 노드 |
|--------|--------|----------|
| 파티션 1 | 2,500만 건 | Worker 1 |
| 파티션 2 | 2,500만 건 | Worker 2 |
| 파티션 3 | 2,500만 건 | Worker 3 |
| 파티션 4 | 2,500만 건 | Worker 4 |

DataFrame (1억 건)을 4개 파티션으로 나누어 4개 Worker가 동시에 병렬 처리합니다.

---

## Transformation과 Action

Spark의 연산은 크게 **Transformation(변환)** 과 **Action(실행)** 으로 나뉩니다. 이 구분은 Spark의 **지연 실행(Lazy Evaluation)** 방식과 직접 관련됩니다.

### Transformation (변환) — "계획 세우기"

데이터를 어떻게 변환할지 **계획만 세우고**, 실제로 실행하지는 않습니다.

```python
# 이 코드들은 아직 실행되지 않습니다 (계획만 세움)
filtered = df.filter(df.나이 >= 30)           # 30세 이상 필터링
selected = filtered.select("이름", "연봉")      # 필요한 컬럼만 선택
sorted_df = selected.orderBy("연봉", ascending=False)  # 연봉 내림차순 정렬
```

### Action (실행) — "계획 실행하기"

실제로 데이터를 처리하고 결과를 반환합니다. Action이 호출되면 그때서야 위의 모든 Transformation이 실행됩니다.

```python
# Action: 이 시점에 위의 모든 변환이 실행됩니다
sorted_df.show()        # 결과를 화면에 출력 (Action!)
sorted_df.count()       # 건수를 반환 (Action!)
sorted_df.collect()     # 전체 데이터를 Driver로 가져옴 (Action!)
sorted_df.write.save()  # 파일로 저장 (Action!)
```

> 💡 **지연 실행(Lazy Evaluation)이란?** Spark가 Transformation을 즉시 실행하지 않고, Action이 호출될 때까지 기다리는 전략입니다. 이렇게 하면 Spark가 전체 실행 계획을 먼저 최적화한 후 실행할 수 있어서, 불필요한 계산을 줄이고 성능을 크게 향상시킬 수 있습니다.

### 주요 Transformation과 Action 목록

| Transformation (변환) | 설명 |
|----------------------|------|
| `filter()` / `where()` | 조건에 맞는 행만 남깁니다 |
| `select()` | 특정 컬럼만 선택합니다 |
| `groupBy()` | 그룹별로 묶습니다 |
| `join()` | 두 DataFrame을 결합합니다 |
| `orderBy()` | 정렬합니다 |
| `withColumn()` | 새 컬럼을 추가하거나 기존 컬럼을 변환합니다 |
| `drop()` | 컬럼을 제거합니다 |
| `distinct()` | 중복을 제거합니다 |

| Action (실행) | 설명 |
|--------------|------|
| `show()` | 결과를 화면에 출력합니다 |
| `count()` | 행 수를 반환합니다 |
| `collect()` | 전체 데이터를 리스트로 반환합니다 |
| `first()` / `head()` | 첫 번째 행을 반환합니다 |
| `write.save()` | 파일이나 테이블로 저장합니다 |
| `display()` | Databricks 노트북에서 시각화와 함께 출력합니다 |

---

## Spark SQL — SQL로 분산 처리하기

Spark의 강력한 장점 중 하나는 **SQL로도 분산 처리가 가능**하다는 것입니다. Python DataFrame API와 SQL을 자유롭게 혼합하여 사용할 수 있습니다.

```python
# DataFrame을 임시 뷰로 등록
df.createOrReplaceTempView("employees")

# SQL로 분석
result = spark.sql("""
    SELECT
        도시,
        COUNT(*) AS 인원수,
        AVG(연봉) AS 평균연봉
    FROM employees
    GROUP BY 도시
    ORDER BY 평균연봉 DESC
""")

result.show()
```

Databricks에서는 `%sql` 매직 커맨드를 사용하여 노트북 셀에서 직접 SQL을 실행할 수도 있습니다.

```sql
%sql
SELECT 도시, COUNT(*) AS 인원수, AVG(연봉) AS 평균연봉
FROM employees
GROUP BY 도시
ORDER BY 평균연봉 DESC
```

---

## 실습: 기본 데이터 처리

```python
from pyspark.sql.functions import col, sum, avg, count, when, upper

# 1. 데이터 읽기 (Delta 테이블에서)
orders = spark.read.table("catalog.schema.orders")

# 2. 필터링: 완료된 주문만
completed = orders.filter(col("status") == "COMPLETED")

# 3. 변환: 새 컬럼 추가
enriched = completed.withColumn(
    "order_size",
    when(col("amount") >= 100000, "대형")
    .when(col("amount") >= 50000, "중형")
    .otherwise("소형")
)

# 4. 집계: 주문 크기별 통계
summary = enriched.groupBy("order_size").agg(
    count("*").alias("주문건수"),
    sum("amount").alias("총매출"),
    avg("amount").alias("평균주문액")
)

# 5. 결과 출력
display(summary)

# 6. 결과를 Delta 테이블로 저장
summary.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("catalog.schema.order_summary")
```

---

## Databricks에서의 Spark

Databricks에서 Spark를 사용할 때 알아두면 좋은 점들을 정리하겠습니다.

| 항목 | 설명 |
|------|------|
| **자동 설정** | 클러스터를 생성하면 Spark가 자동으로 구성됩니다. 별도 설치가 필요 없습니다 |
| **spark 변수** | 노트북에서 `spark` 변수가 자동으로 사용 가능합니다 |
| **Photon 엔진** | Databricks 전용 고성능 엔진으로, Spark SQL 쿼리를 자동으로 가속합니다 |
| **Adaptive Query Execution** | 실행 중에 자동으로 쿼리 계획을 최적화합니다 |
| **display() 함수** | Databricks 전용 함수로, `show()` 대신 시각화와 함께 결과를 표시합니다 |

> 🆕 **Databricks Runtime 18.x**: 최신 Databricks Runtime은 **Apache Spark 4.1.0**을 기반으로 하며, 성능 개선과 새로운 기능들이 포함되어 있습니다. 클러스터 생성 시 최신 Runtime을 선택하시면 최적의 성능을 경험하실 수 있습니다.

---

## 심화: Spark 내부 동작과 성능 최적화

이 섹션에서는 Spark의 내부 동작 원리와 프로덕션 환경에서의 성능 최적화 기법을 다룹니다. 대규모 데이터 처리에서 발생하는 실제 문제와 해결 방법을 이해하면, 데이터 파이프라인을 훨씬 효율적으로 설계하실 수 있습니다.

### Catalyst Optimizer — Spark의 두뇌

Spark가 빠른 이유 중 하나는 **Catalyst Optimizer**라는 쿼리 최적화 엔진입니다. SQL이든 DataFrame API든, 모든 연산은 Catalyst를 거쳐 최적화된 실행 계획으로 변환됩니다.

#### 실행 계획 변환 과정

```
사용자 코드 (SQL/DataFrame API)
    ↓
[1] Unresolved Logical Plan — 파싱된 논리 계획 (테이블/컬럼 미확인)
    ↓ (Catalog에서 테이블/컬럼 확인)
[2] Resolved Logical Plan — 확인된 논리 계획
    ↓ (최적화 규칙 적용)
[3] Optimized Logical Plan — 최적화된 논리 계획
    ↓ (물리 전략 선택)
[4] Physical Plan — 물리 실행 계획
    ↓ (코드 생성)
[5] RDD 연산 — 실제 분산 실행
```

#### 주요 최적화 규칙

| 최적화 규칙 | 설명 | 효과 |
|------------|------|------|
| **Predicate Pushdown** | WHERE 조건을 가능한 한 데이터 소스에 가깝게 내려보냅니다 | Parquet/Delta에서 필요한 행만 읽음 → I/O 대폭 감소 |
| **Column Pruning** | SELECT에 필요한 컬럼만 읽습니다 | 100개 컬럼 테이블에서 3개만 필요하면 3개만 읽음 |
| **Constant Folding** | `1 + 2`와 같은 상수 표현식을 컴파일 시점에 `3`으로 치환합니다 | 런타임 계산 제거 |
| **Join Reordering** | 조인 순서를 최적화합니다 (작은 테이블을 먼저 조인) | Shuffle 데이터량 감소 |
| **Partition Pruning** | 파티션 컬럼 조건으로 불필요한 파티션을 건너뜁니다 | 전체 테이블 대신 필요 파티션만 스캔 |

```python
# 실행 계획 확인 방법
df = spark.sql("""
    SELECT customer_id, SUM(amount)
    FROM catalog.schema.orders
    WHERE order_date >= '2025-01-01'
    GROUP BY customer_id
""")

# 논리 + 물리 실행 계획 출력
df.explain(mode="extended")

# Databricks에서는 Spark UI의 SQL 탭에서 시각적으로 확인 가능
```

#### Cost-Based Optimization (CBO)

Catalyst는 테이블 통계(행 수, 컬럼 분포, 데이터 크기)를 활용하여 **비용 기반** 으로 최적의 실행 계획을 선택합니다.

```sql
-- 테이블 통계 수집 (CBO 활성화의 핵심)
ANALYZE TABLE catalog.schema.orders COMPUTE STATISTICS;

-- 컬럼 단위 통계 수집 (더 정밀한 최적화)
ANALYZE TABLE catalog.schema.orders
COMPUTE STATISTICS FOR COLUMNS customer_id, order_date, amount;
```

> ⚠️ **Gotcha**: CBO는 통계가 **최신 상태일 때만** 효과적입니다. 대량 데이터 적재 후 ANALYZE를 실행하지 않으면, Catalyst가 잘못된 통계로 비효율적인 조인 전략을 선택할 수 있습니다. Delta Lake의 Predictive Optimization은 이를 자동화합니다.

---

### Shuffle 메커니즘 심화

> 💡 **Shuffle**이란 데이터를 키(key) 기준으로 재분배하는 과정입니다. GROUP BY, JOIN, ORDER BY 등의 연산에서 발생하며, **네트워크를 통해 데이터가 이동**하므로 Spark에서 가장 비용이 큰 연산입니다.

#### 조인 전략 비교

| 조인 전략 | 조건 | 데이터 이동 | 성능 |
|-----------|------|-----------|------|
| **Broadcast Hash Join** | 한쪽 테이블이 작음 (기본 ≤ 10MB) | 작은 테이블을 모든 Executor에 복제 | ⭐⭐⭐ 가장 빠름 |
| **Sort-Merge Join** | 양쪽 테이블 모두 큼 | 양쪽 데이터를 키별로 셔플 + 정렬 | ⭐⭐ 안정적 |
| **Shuffle Hash Join** | 한쪽이 상대적으로 작음 | 양쪽 데이터를 키별로 셔플 | ⭐⭐ 정렬 불필요 |
| **Cartesian Join** | 조인 조건 없음 | 전체 데이터 교차 결합 | ⭐ 매우 느림 (주의!) |

```python
from pyspark.sql.functions import broadcast

# Broadcast Join 강제 (작은 테이블을 명시적으로 브로드캐스트)
result = large_orders.join(
    broadcast(small_stores),  # small_stores를 모든 Executor로 복제
    "store_id"
)

# Broadcast 임계값 조정 (기본 10MB → 100MB)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "104857600")  # 100MB
```

> ⚠️ **Gotcha — Broadcast Join 메모리**: Broadcast되는 테이블은 **각 Executor의 메모리에 적재**됩니다. 100MB 테이블을 20개 Executor에 브로드캐스트하면 총 2GB의 메모리를 사용합니다. 임계값을 너무 크게 설정하면 OOM(Out of Memory)이 발생할 수 있습니다.

#### Shuffle 파티션 수 튜닝

```python
# 기본값: 200 (대부분의 워크로드에 부적합!)
spark.conf.get("spark.sql.shuffle.partitions")  # "200"

# 데이터 규모에 맞게 조정
# 일반 가이드: 셔플 후 각 파티션이 100~200MB가 되도록 설정
# 예: 셔플 데이터 100GB → 100GB / 128MB ≈ 800 파티션
spark.conf.set("spark.sql.shuffle.partitions", "800")
```

| 데이터 규모 | 권장 Shuffle 파티션 수 | 파티션당 크기 |
|------------|----------------------|-------------|
| < 1GB | 20~50 | 20~50MB |
| 1~10GB | 50~200 | 50~200MB |
| 10~100GB | 200~1,000 | 100~200MB |
| 100GB~1TB | 1,000~5,000 | 100~200MB |
| > 1TB | 5,000~20,000 | 100~200MB |

> ⚠️ **Gotcha**: 파티션 수가 너무 많으면 **스케줄링 오버헤드**와 **소형 파일 문제**가 발생합니다. 반대로 너무 적으면 각 Task가 처리하는 데이터가 많아져 **OOM**이나 **GC 지연**이 발생합니다. AQE를 활성화하면 이를 자동으로 조정합니다.

---

### AQE (Adaptive Query Execution) 심화

> 💡 **AQE**는 Spark 3.x부터 도입된 기능으로, 쿼리 **실행 중에** 런타임 통계를 수집하여 실행 계획을 동적으로 수정합니다. Databricks Runtime에서는 기본 활성화되어 있습니다.

#### AQE의 세 가지 핵심 기능

| 기능 | 설명 | 효과 |
|------|------|------|
| **Coalescing Post-Shuffle Partitions** | 셔플 후 작은 파티션들을 자동으로 합칩니다 | 소형 파일/태스크 감소, 오버헤드 절감 |
| **Converting Sort-Merge Join to Broadcast Hash Join** | 런타임에 한쪽 테이블이 작으면 Broadcast Join으로 전환합니다 | 불필요한 셔플 제거 |
| **Optimizing Skew Join** | 데이터 편향(Skew)이 있는 파티션을 자동 분할합니다 | Skew로 인한 지연 해소 |

```python
# AQE 관련 설정 (Databricks에서는 기본 활성화)
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

# Skew Join 감지 임계값 (기본: 256MB, 파티션 중앙값의 5배)
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "268435456")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")

# 셔플 파티션 합치기 목표 크기 (기본: 64MB)
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "134217728")  # 128MB
```

> ⚠️ **Gotcha**: AQE는 **Shuffle 또는 Broadcast Exchange 경계**에서만 재최적화합니다. 단일 Stage 내에서는 초기 계획이 유지됩니다. 따라서 AQE가 있어도 `ANALYZE TABLE`로 통계를 수집하는 것이 여전히 중요합니다.

---

### 데이터 Skew 처리

> 💡 **데이터 Skew(편향)** 란 특정 키에 데이터가 집중되어, 하나의 파티션이 다른 파티션보다 훨씬 큰 상태를 말합니다. 예를 들어, 전체 주문의 40%가 "서울" 지역인 경우, GROUP BY 지역 시 "서울" 파티션이 병목이 됩니다.

#### Skew 진단 방법

```
Spark UI → Stages 탭 → Task Duration 확인
- 대부분의 Task: 10초
- 하나의 Task: 10분  ← Skew!
```

#### 해결 방법 1: AQE Skew Join (자동)

위에서 설명한 AQE의 Skew Join 최적화가 자동으로 처리합니다. Databricks에서는 기본 활성화되어 있으므로, 대부분의 경우 별도 조치 없이 해결됩니다.

#### 해결 방법 2: Skew Hint (수동)

```sql
-- Databricks에서 Skew Join Hint 사용
SELECT /*+ SKEW('orders', 'region') */
    o.*, s.store_name
FROM orders o
JOIN stores s ON o.region = s.region;

-- 특정 Skew 값을 명시
SELECT /*+ SKEW('orders', 'region', ('서울', '경기')) */
    o.*, s.store_name
FROM orders o
JOIN stores s ON o.region = s.region;
```

#### 해결 방법 3: Salting 기법 (수동)

Skew가 심한 키에 랜덤 접미사(salt)를 추가하여 데이터를 강제로 분산시킵니다.

```python
from pyspark.sql.functions import col, concat, lit, floor, rand, explode, array

SALT_BUCKETS = 10

# 큰 테이블: 키에 salt 추가
orders_salted = orders.withColumn(
    "salted_key",
    concat(col("region"), lit("_"), floor(rand() * SALT_BUCKETS).cast("string"))
)

# 작은 테이블: salt 값 전체를 explode로 복제
salt_values = [str(i) for i in range(SALT_BUCKETS)]
stores_exploded = stores.withColumn(
    "salt", explode(array([lit(s) for s in salt_values]))
).withColumn(
    "salted_key", concat(col("region"), lit("_"), col("salt"))
)

# Salted 키로 조인
result = orders_salted.join(stores_exploded, "salted_key")
```

> ⚠️ **Gotcha**: Salting은 작은 테이블을 N배(salt 수)로 복제하므로 메모리를 많이 사용합니다. AQE의 자동 Skew 처리가 가능하면 먼저 시도하고, 효과가 부족할 때만 Salting을 사용하세요.

---

### 메모리 관리

Spark Executor의 메모리는 세 영역으로 나뉩니다. 메모리 구조를 이해하면 OOM 오류를 진단하고 해결하는 데 큰 도움이 됩니다.

#### Executor 메모리 구조

```
spark.executor.memory (예: 8GB)
├─ Unified Memory (spark.memory.fraction = 0.6 → 4.8GB)
│  ├─ Execution Memory — Shuffle, Sort, Aggregation 등 연산에 사용
│  └─ Storage Memory — cache(), persist() 등 캐시에 사용
│  (두 영역은 동적으로 서로 차용 가능)
├─ User Memory (0.4 → 3.2GB) — UDF, 사용자 데이터 구조
└─ Reserved Memory (300MB 고정) — Spark 내부 사용
```

| 메모리 영역 | 비율 | 용도 | OOM 원인 |
|------------|------|------|---------|
| **Execution** | 동적 (Unified 내) | Shuffle, Sort, Join, Aggregation | 셔플 데이터가 메모리 초과 → Spill to Disk |
| **Storage** | 동적 (Unified 내) | DataFrame 캐시 | 캐시가 너무 많으면 Execution 영역 부족 |
| **User** | 40% | UDF, 브로드캐스트 변수 | 대형 컬렉션을 Driver로 collect() |

#### Spill to Disk

Execution Memory가 부족하면 Spark는 데이터를 디스크로 **Spill(유출)** 합니다. Spill 자체는 OOM을 방지하는 안전 장치이지만, **디스크 I/O로 인해 성능이 크게 저하**됩니다.

```
Spark UI → Stages 탭에서 확인:
- Shuffle Spill (Memory): 메모리에서 처리된 데이터
- Shuffle Spill (Disk): 디스크로 유출된 데이터
- Disk Spill 비율이 높으면 메모리 부족 신호!
```

#### OOM 대응 체크리스트

| 순서 | 조치 | 설명 |
|------|------|------|
| 1 | **파티션 수 늘리기** | 파티션당 데이터량을 줄여 메모리 사용 감소 |
| 2 | **Broadcast Join 확인** | 너무 큰 테이블이 브로드캐스트되고 있지 않은지 확인 |
| 3 | **collect() 제거** | Driver로 대량 데이터를 가져오는 collect() 사용 자제 |
| 4 | **캐시 정리** | 불필요한 cache/persist 제거 (unpersist()) |
| 5 | **Executor 메모리 증가** | 클러스터 설정에서 메모리가 큰 인스턴스 타입 선택 |
| 6 | **Executor 수 증가** | Worker 노드를 추가하여 데이터를 더 분산 |

```python
# OOM 디버깅에 유용한 설정
spark.conf.set("spark.sql.adaptive.enabled", "true")  # AQE로 자동 최적화
spark.conf.set("spark.sql.shuffle.partitions", "auto")  # Databricks에서 자동 조정

# 메모리 사용량 모니터링 (Spark UI 외)
print(f"Storage Memory Used: {spark.sparkContext._jsc.sc().getExecutorMemoryStatus()}")
```

> ⚠️ **Gotcha — collect()의 위험**: `df.collect()`는 전체 DataFrame 데이터를 **Driver 한 대의 메모리**로 가져옵니다. 1억 건의 데이터를 collect하면 Driver가 OOM으로 죽습니다. 반드시 `.limit()`, `.head()`, `.take()` 등으로 필요한 데이터만 가져오세요.

> ⚠️ **Gotcha — Python UDF 메모리**: Python UDF(User Defined Function)는 JVM과 Python 프로세스 사이에 데이터를 직렬화/역직렬화(SerDe)합니다. 이 과정에서 메모리를 2배 이상 사용하며, 성능도 크게 저하됩니다. 가능하면 **Spark 내장 함수**나 **Pandas UDF**(Arrow 기반)를 사용하세요.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Apache Spark** | 대용량 데이터를 분산 처리하는 오픈소스 엔진입니다 |
| **Driver** | 작업을 계획하고 지휘하는 프로세스입니다 |
| **Executor** | 실제 데이터를 처리하는 작업자 프로세스입니다 |
| **DataFrame** | 분산 환경에서 데이터를 다루는 기본 구조입니다 |
| **Partition** | DataFrame을 물리적으로 나눈 단위. 병렬 처리의 기본 단위입니다 |
| **Lazy Evaluation** | Transformation은 계획만 세우고, Action 호출 시 실행합니다 |
| **Catalyst Optimizer** | SQL/DataFrame 연산을 자동으로 최적화하는 쿼리 엔진입니다 |
| **AQE** | 런타임 통계를 활용하여 실행 중 계획을 동적으로 최적화합니다 |
| **Shuffle** | 키 기반 데이터 재분배. 가장 비용이 큰 연산이므로 최적화가 중요합니다 |

다음 문서에서는 Spark를 실행하는 컴퓨팅 리소스인 **클러스터**의 종류와 설정 방법을 살펴보겠습니다.

---

## 참고 링크

- [Databricks: Apache Spark on Databricks](https://docs.databricks.com/aws/en/spark/)
- [Azure Databricks: Apache Spark](https://learn.microsoft.com/en-us/azure/databricks/spark/)
- [Apache Spark Official](https://spark.apache.org/)
