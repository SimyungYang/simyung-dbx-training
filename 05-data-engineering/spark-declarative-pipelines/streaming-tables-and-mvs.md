# Streaming Tables & Materialized Views

## 왜 두 가지 테이블 유형이 필요한가?

데이터 파이프라인에서 처리하는 데이터의 성격은 크게 두 가지입니다:

1. **계속 추가되는 데이터**— 로그, 이벤트, 트랜잭션처럼 새 데이터가 지속적으로 들어오는 경우
2. **집계/변환이 필요한 데이터**— 기존 데이터를 GROUP BY, JOIN 등으로 요약하거나 변환하는 경우

SDP(Spark Declarative Pipelines)는 이 두 가지 상황에 최적화된 서로 다른 테이블 유형을 제공합니다.

> 💡 쉽게 비유하면, **Streaming Table** 은 "받은 편지함"처럼 새 메일이 계속 쌓이는 곳이고, **Materialized View** 는 "대시보드"처럼 전체 데이터를 요약해서 보여주는 곳입니다.

---

## Streaming Table (스트리밍 테이블)

### 개념

**Streaming Table** 은 새로 도착한 데이터만 **증분(Incremental)** 으로 처리하여 추가하는 테이블입니다. 한 번 처리된 데이터는 다시 처리하지 않으며, **Append-Only** 방식으로 동작합니다.

### 핵심 특성

| 특성 | 설명 |
|------|------|
| **처리 방식** | 증분 처리 (Incremental). 마지막 처리 이후의 새 데이터만 읽습니다 |
| **데이터 모델** | 기본적으로 Append-Only. `APPLY CHANGES`로 UPDATE/DELETE 가능 |
| **소스 요구사항** | 스트리밍 소스 필요 (`STREAM()` 함수 사용) |
| **상태 관리** | 체크포인트를 통해 "어디까지 읽었는지" 자동 추적합니다 |
| **적합한 계층** | Medallion의 **Bronze**, **Silver** 계층 |

### 지원하는 소스 유형

**Streaming Table 소스 유형**

| 소스 유형 | 설명 |
|-----------|------|
| 클라우드 스토리지 (Auto Loader) | S3, ADLS, GCS 등에서 파일을 자동 수집합니다 |
| Kafka / Event Hub (메시지 큐) | 실시간 메시지 스트림을 수집합니다 |
| 다른 Streaming Table (체이닝) | 이전 Streaming Table의 출력을 입력으로 사용합니다 |
| CDC 소스 (APPLY CHANGES) | CDC 변경 데이터를 처리합니다 |

### SQL 예제

```sql
-- 기본 Streaming Table: 새 데이터를 증분 처리
CREATE OR REFRESH STREAMING TABLE silver_orders
AS SELECT
    CAST(order_id AS BIGINT) AS order_id,
    CAST(customer_id AS BIGINT) AS customer_id,
    CAST(amount AS DECIMAL(12,2)) AS amount,
    CAST(order_date AS TIMESTAMP) AS order_date,
    CAST(status AS STRING) AS status
FROM STREAM(bronze_orders);
```

### Python 예제

```python
import dlt
from pyspark.sql.functions import col

@dlt.table(
    comment="정제된 주문 데이터 (증분 처리)",
    table_properties={"quality": "silver"}
)
def silver_orders():
    return (
        dlt.read_stream("bronze_orders")
           .select(
               col("order_id").cast("bigint"),
               col("customer_id").cast("bigint"),
               col("amount").cast("decimal(12,2)"),
               col("order_date").cast("timestamp"),
               col("status").cast("string")
           )
    )
```

### 내부 동작 원리

1. **첫 번째 실행**: 소스의 모든 데이터를 읽어 테이블에 저장합니다
2. **이후 실행**: 체크포인트를 기준으로 "마지막으로 읽은 위치" 이후의 새 데이터만 읽습니다
3. **결과**: 이전에 처리한 데이터를 다시 읽지 않으므로 매우 효율적입니다

> ⚠️ Streaming Table의 소스는 반드시 `STREAM()` 함수로 감싸야 합니다. 이를 빠뜨리면 일반 배치 읽기가 되어 매번 전체 데이터를 다시 처리하게 됩니다.

---

## Materialized View (구체화된 뷰)

### 개념

**Materialized View** 는 쿼리의 결과를 미리 계산하여 저장하는 테이블입니다. 소스 데이터가 변경되면 결과를 **자동으로 재계산** 합니다. 일반 VIEW와 달리 결과가 물리적으로 저장되어 있으므로 조회가 빠릅니다.

### 핵심 특성

| 특성 | 설명 |
|------|------|
| **처리 방식** | 소스 변경 시 결과를 재계산합니다. 가능한 경우 증분 갱신을 시도합니다 |
| **데이터 모델** | 전체 결과를 유지합니다. UPDATE/DELETE가 자동 반영됩니다 |
| **소스 요구사항** | 모든 테이블/뷰를 소스로 사용 가능합니다 (`STREAM()` 불필요) |
| **적합한 연산** | GROUP BY, JOIN, UNION, 윈도우 함수 등 모든 SQL 연산 |
| **적합한 계층** | Medallion의 **Silver**(일부), **Gold** 계층 |

### SQL 예제

```sql
-- 일별 매출 집계 (Materialized View)
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_revenue
AS SELECT
    DATE(order_date) AS sale_date,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM silver_orders
GROUP BY DATE(order_date);
```

### Python 예제

```python
import dlt
from pyspark.sql.functions import col, count, sum as _sum, avg, countDistinct, date_trunc

@dlt.table(
    comment="일별 매출 집계",
    table_properties={"quality": "gold"}
)
def gold_daily_revenue():
    return (
        dlt.read("silver_orders")  # STREAM이 아닌 일반 read
           .groupBy(date_trunc("day", col("order_date")).alias("sale_date"))
           .agg(
               count("*").alias("order_count"),
               _sum("amount").alias("total_revenue"),
               avg("amount").alias("avg_order_value"),
               countDistinct("customer_id").alias("unique_customers")
           )
    )
```

### 내부 동작 원리

1. **첫 번째 실행**: 소스의 전체 데이터를 읽어 쿼리 결과를 계산하고 저장합니다
2. **이후 실행**: 소스 데이터의 변경사항을 감지하여 결과를 갱신합니다
   - **증분 갱신 가능**: 일부 쿼리 패턴(단순 필터, 집계 등)은 변경분만으로 결과를 갱신합니다
   - **전체 재계산 필요**: 복잡한 JOIN, 윈도우 함수 등은 전체 데이터를 다시 계산합니다
3. **결과**: 항상 소스의 최신 상태를 반영한 정확한 결과를 제공합니다

> 💡 Materialized View는 소스에서 데이터가 UPDATE되거나 DELETE되어도 결과에 자동으로 반영됩니다. 이것이 Streaming Table과의 가장 큰 차이점입니다.

---

## 핵심 차이 비교

### 상세 비교표

| 비교 항목 | Streaming Table | Materialized View |
|----------|----------------|-------------------|
| **처리 모델** | 증분 처리 (Append 기반) | 전체 결과 재계산 (또는 증분 갱신) |
| **소스 읽기** | `STREAM()` 함수 필수 | 일반 테이블 참조 |
| **INSERT** | ✅ 새 행 추가 | ✅ 결과에 반영 |
| **UPDATE** | ❌ 기본 미지원 (APPLY CHANGES 필요) | ✅ 자동 반영 |
| **DELETE** | ❌ 기본 미지원 (APPLY CHANGES 필요) | ✅ 자동 반영 |
| **적합한 연산** | 필터, 변환, 정제 | 집계, JOIN, 복잡한 연산 |
| **Medallion 계층** | Bronze, Silver | Silver(일부), Gold |
| **갱신 비용** | 낮음 (새 데이터만) | 중간~높음 (변경 범위에 따라) |
| **실시간성** | 높음 | 중간 |
| **상태 관리** | 체크포인트로 진행 위치 추적 | 소스-타겟 간 차이 감지 |

### 언제 무엇을 선택할까?

**데이터 처리 유형 선택 가이드**

| 질문 | 조건 | 권장 유형 |
|------|------|----------|
| 새 데이터만 계속 추가되는가? | Yes + 단순 변환/필터만 필요 | Streaming Table |
| 새 데이터만 계속 추가되는가? | Yes + 집계/JOIN 필요 | Materialized View |
| UPDATE/DELETE 포함? | CDC 처리 필요 | Streaming Table + APPLY CHANGES |
| UPDATE/DELETE 포함? | CDC 불필요 | Materialized View |

### 의사결정 가이드

| 상황 | 권장 테이블 유형 | 이유 |
|------|----------------|------|
| 클라우드 스토리지에서 파일 수집 | **Streaming Table** | Auto Loader와 함께 증분 처리가 효율적입니다 |
| Kafka에서 이벤트 소비 | **Streaming Table** | 스트리밍 소스를 자연스럽게 처리합니다 |
| 로그/이벤트 데이터 적재 | **Streaming Table** | Append-Only 특성에 잘 맞습니다 |
| CDC 데이터 반영 | **Streaming Table**+ APPLY CHANGES | CDC의 INSERT/UPDATE/DELETE를 모두 처리합니다 |
| 일별/월별 매출 집계 | **Materialized View** | GROUP BY 집계는 전체 데이터 기반이 필요합니다 |
| 여러 테이블 JOIN | **Materialized View** | JOIN 결과를 미리 계산하여 조회 성능을 높입니다 |
| 중복 제거 (DISTINCT) | **Materialized View** | 전체 데이터에서 중복을 제거해야 합니다 |
| 윈도우 함수 (순위, 누적합) | **Materialized View** | 전체 데이터 맥락이 필요한 연산입니다 |

---

## Streaming Table + Materialized View 조합 패턴

실제 파이프라인에서는 두 유형을 **함께 사용** 하여 Medallion 아키텍처를 구현합니다.

### Medallion 아키텍처 적용 예제

**Medallion 아키텍처에서의 Streaming Table과 Materialized View 활용**

| 계층 | 테이블 | 유형 | 설명 |
|------|--------|------|------|
| **Bronze (원본 수집)** | bronze_orders | Streaming Table | Auto Loader로 수집 |
|  | bronze_customers | Streaming Table | CDC로 수집 |
| **Silver (정제/통합)** | silver_orders | Streaming Table | 필터 + 타입 변환 |
|  | silver_customers | Streaming Table | APPLY CHANGES 처리 |
| **Gold (비즈니스 집계)** | gold_daily_revenue | Materialized View | 일별 매출 집계 |
|  | gold_customer_orders | Materialized View | 고객별 주문 요약 |

### 전체 파이프라인 코드 (SQL)

```sql
-- ============================================
-- Bronze: Streaming Table (원본 데이터 수집)
-- ============================================
CREATE OR REFRESH STREAMING TABLE bronze_orders
AS SELECT * FROM STREAM read_files(
    '/volumes/catalog/schema/landing/orders/',
    format => 'json'
);

CREATE OR REFRESH STREAMING TABLE bronze_customers
AS SELECT * FROM STREAM read_files(
    '/volumes/catalog/schema/landing/customers/',
    format => 'json'
);

-- ============================================
-- Silver: Streaming Table (데이터 정제)
-- ============================================
CREATE OR REFRESH STREAMING TABLE silver_orders (
    CONSTRAINT valid_order_id EXPECT (order_id IS NOT NULL) ON VIOLATION DROP ROW,
    CONSTRAINT positive_amount EXPECT (amount > 0) ON VIOLATION DROP ROW
)
AS SELECT
    CAST(order_id AS BIGINT) AS order_id,
    CAST(customer_id AS BIGINT) AS customer_id,
    CAST(amount AS DECIMAL(12,2)) AS amount,
    CAST(order_date AS TIMESTAMP) AS order_date,
    CAST(status AS STRING) AS status
FROM STREAM(bronze_orders);

-- Silver: CDC 처리
CREATE OR REFRESH STREAMING TABLE silver_customers;

APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customers)
KEYS (customer_id)
SEQUENCE BY updated_at
STORED AS SCD TYPE 1;

-- ============================================
-- Gold: Materialized View (비즈니스 집계)
-- ============================================
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_revenue
AS SELECT
    DATE(o.order_date) AS sale_date,
    COUNT(*) AS order_count,
    SUM(o.amount) AS total_revenue,
    AVG(o.amount) AS avg_order_value
FROM silver_orders o
GROUP BY DATE(o.order_date);

CREATE OR REFRESH MATERIALIZED VIEW gold_customer_orders
AS SELECT
    c.customer_id,
    c.name AS customer_name,
    c.city,
    COUNT(o.order_id) AS total_orders,
    SUM(o.amount) AS total_spent,
    MAX(o.order_date) AS last_order_date
FROM silver_customers c
LEFT JOIN silver_orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city;
```

---

## 새로고침 전략

파이프라인의 갱신 방식을 상황에 맞게 설정할 수 있습니다.

### 새로고침 모드 비교

| 모드 | 설명 | 적합한 경우 |
|------|------|-----------|
| **Triggered**(트리거) | 수동으로 또는 스케줄에 의해 실행합니다. 실행 후 종료됩니다 | 배치 처리, 비용 최적화가 필요한 경우 |
| **Continuous**(연속) | 파이프라인이 항상 실행 상태를 유지하며, 새 데이터를 즉시 처리합니다 | 실시간/준실시간 처리가 필요한 경우 |
| **Scheduled**(스케줄) | cron 표현식으로 주기적 실행을 설정합니다 | 정기 배치(시간별, 일별) |

### 전체 새로고침 (Full Refresh)

기존 데이터를 모두 삭제하고 처음부터 다시 처리합니다.

```sql
-- SQL에서 전체 새로고침 (소스부터 재처리)
CREATE OR REFRESH STREAMING TABLE silver_orders
AS SELECT ...
FROM STREAM(bronze_orders);
-- Pipeline UI에서 "Full Refresh" 옵션 선택
```

> ⚠️ **전체 새로고침 주의사항**:
> - Streaming Table의 전체 새로고침은 체크포인트를 초기화하고, 소스의 모든 데이터를 다시 처리합니다.
> - 대량의 데이터가 있는 경우 시간과 비용이 크게 증가할 수 있습니다.
> - Materialized View는 매 실행마다 자동으로 필요한 범위를 재계산하므로 전체 새로고침의 영향이 상대적으로 적습니다.

---

## 성능 특성 비교

| 성능 항목 | Streaming Table | Materialized View |
|----------|----------------|-------------------|
| **초기 로드** | 소스 전체 읽기 | 소스 전체 계산 |
| **증분 갱신** | 매우 빠름 (새 데이터만) | 쿼리 복잡도에 따라 다름 |
| **스토리지** | 원본 데이터 크기에 비례 | 집계 결과 크기 (보통 원본보다 작음) |
| **조회 성능** | 원본 데이터 스캔 필요 | 미리 계산된 결과 반환 (빠름) |
| **동시성** | 높음 | 높음 |

---

## 제한사항 및 주의사항

### Streaming Table 제한사항

| 제한사항 | 설명 |
|---------|------|
| Append-Only 기본 | UPDATE/DELETE는 `APPLY CHANGES INTO`를 통해서만 가능합니다 |
| 소스 순서 의존 | 소스 데이터의 도착 순서가 바뀌면 중복 처리가 발생할 수 있습니다 |
| 체크포인트 손실 | 체크포인트가 손상되면 전체 새로고침이 필요합니다 |
| 집계 불가 | `GROUP BY`, `DISTINCT` 등 전체 데이터 연산은 사용할 수 없습니다 |

### Materialized View 제한사항

| 제한사항 | 설명 |
|---------|------|
| 재계산 비용 | 소스 데이터가 매우 크면 갱신 시간이 오래 걸릴 수 있습니다 |
| 스트리밍 소스 미지원 | `STREAM()` 함수를 사용할 수 없습니다. 배치 소스만 가능합니다 |
| 실시간성 한계 | 소스 변경이 결과에 반영되려면 파이프라인 갱신이 필요합니다 |

### 공통 주의사항

> ⚠️ Streaming Table과 Materialized View 모두 SDP 파이프라인 안에서 정의됩니다. 파이프라인 외부에서 직접 데이터를 수정(INSERT/UPDATE/DELETE)하면 데이터 일관성이 깨질 수 있습니다. 항상 파이프라인을 통해 데이터를 관리하세요.

---

## 정리

| 핵심 포인트 | 설명 |
|------------|------|
| **Streaming Table** | 새 데이터만 증분 처리합니다. Append-Only 데이터에 최적화되어 있습니다 |
| **Materialized View** | 전체 데이터를 대상으로 결과를 계산합니다. 집계, JOIN에 적합합니다 |
| **선택 기준** | 데이터가 계속 추가만 되면 ST, 집계/변환이 필요하면 MV를 사용합니다 |
| **조합 패턴** | Bronze/Silver에 ST, Gold에 MV를 사용하는 것이 일반적인 Medallion 패턴입니다 |
| **새로고침** | Triggered(수동/스케줄), Continuous(실시간), Full Refresh(전체 재처리) |

---

## 참고 링크

- [Databricks: Streaming tables](https://docs.databricks.com/aws/en/sdp/streaming-tables.html)
- [Databricks: Materialized views](https://docs.databricks.com/aws/en/sdp/materialized-views.html)
- [Databricks: SDP pipeline settings](https://docs.databricks.com/aws/en/sdp/updates.html)
- [Databricks Blog: Streaming tables vs materialized views](https://www.databricks.com/blog/delta-live-tables-announces-general-availability)
