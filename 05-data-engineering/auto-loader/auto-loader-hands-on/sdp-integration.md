# 실습 3: SDP와 Auto Loader 통합

## 목적과 학습 목표

SDP (Spark Declarative Pipelines, 선언적 파이프라인) 는 Databricks에서 데이터 파이프라인을 선언적 SQL/Python으로 정의하는 프레임워크입니다. SDP 안에서 Auto Loader를 사용하면 체크포인트, 스키마 위치, 오류 처리를 수동으로 관리할 필요 없이 **자동으로 관리** 됩니다.

### 학습 목표

| 목표 | 설명 |
|------|------|
| **SDP + Auto Loader 통합** | `read_files()` 함수로 파일 소스를 SDP 파이프라인에 연결합니다 |
| **Medallion Architecture** | Bronze → Silver → Gold 3계층 파이프라인을 구축합니다 |
| **Data Quality Constraints** | `CONSTRAINT ... EXPECT` 구문으로 데이터 품질 규칙을 선언합니다 |
| **Streaming Table vs Materialized View** | 두 테이블 유형의 차이와 적합한 사용 시나리오를 이해합니다 |
| **파이프라인 모니터링** | SDP UI에서 Data Lineage (데이터 리니지) 와 Data Quality 결과를 확인합니다 |

> **왜 SDP + Auto Loader를 함께 사용하는가?** Auto Loader를 단독으로 사용하면 개발자가 체크포인트 경로, 스키마 위치, `mergeSchema` 옵션 등을 직접 관리해야 합니다. SDP가 이 모든 것을 자동으로 처리합니다. 또한 SDP는 파이프라인 DAG (방향 비순환 그래프), 데이터 품질 모니터링, 자동 재시작 등을 내장합니다.

---

## 사전 준비

### 필요 환경

- [실습 1](setup-and-csv.md) 과 [실습 2](json-and-schema-evolution.md) 에서 생성한 샘플 데이터가 존재해야 합니다.
- **Serverless Pipeline** 또는 **Classic Pipeline** 클러스터를 사용할 수 있어야 합니다.
- Databricks Runtime: **Delta Live Tables** 가 활성화된 환경 (DBR 12.x LTS 이상 권장).

### SDP 파이프라인 개요

```
소스 파일
  ├── /csv/orders/       → [Bronze] bronze_orders (Streaming Table)
  └── /json/customers/   → [Bronze] bronze_customers (Streaming Table)
                               ↓
                    [Silver] silver_orders (Streaming Table, Data Quality 적용)
                    [Silver] silver_customers (Streaming Table, Data Quality 적용)
                               ↓
                    [Gold] gold_daily_sales (Materialized View)
                    [Gold] gold_customer_by_city (Materialized View)
```

---

## SDP 파이프라인 코드

아래 SQL 코드를 **하나의 새 노트북** 에 작성합니다 (셀 단위로 분리하거나 모두 하나의 셀에 작성해도 됩니다).

---

### Bronze Layer: 원본 데이터 수집

```sql
-- =========================================================
-- [Bronze] CSV 주문 데이터 수집
-- Auto Loader(read_files)로 CSV 파일을 스트리밍 수집합니다
-- SDP가 checkpointLocation과 schemaLocation을 자동 관리합니다
-- =========================================================
CREATE OR REFRESH STREAMING TABLE bronze_orders
COMMENT '원본 주문 데이터 - CSV 파일에서 Auto Loader로 수집'
TBLPROPERTIES (
  'quality' = 'bronze',
  'pipelines.reset.allowed' = 'true'
)
AS SELECT
    *,
    _metadata.file_path          AS _source_file,
    _metadata.file_name          AS _source_file_name,
    _metadata.file_size          AS _source_file_size,
    _metadata.file_modification_time AS _file_modified_at,
    current_timestamp()          AS _ingested_at
FROM STREAM read_files(
    '/Volumes/training/auto_loader_lab/raw_data/csv/orders/',
    format            => 'csv',
    header            => true,
    inferColumnTypes  => true,
    rescuedDataColumn => '_rescued_data'
);
```

```sql
-- =========================================================
-- [Bronze] JSON 고객 데이터 수집
-- =========================================================
CREATE OR REFRESH STREAMING TABLE bronze_customers
COMMENT '원본 고객 데이터 - JSON 파일에서 Auto Loader로 수집'
TBLPROPERTIES (
  'quality' = 'bronze',
  'pipelines.reset.allowed' = 'true'
)
AS SELECT
    *,
    _metadata.file_path              AS _source_file,
    _metadata.file_modification_time AS _file_modified_at,
    current_timestamp()              AS _ingested_at
FROM STREAM read_files(
    '/Volumes/training/auto_loader_lab/raw_data/json/customers/',
    format            => 'json',
    inferColumnTypes  => true,
    rescuedDataColumn => '_rescued_data'
);
```

> **`read_files()` vs `spark.readStream.format("cloudFiles")`**: `read_files()`는 SDP 전용 SQL 함수로, 내부적으로 `cloudFiles` 포맷을 사용합니다. SDP 환경에서는 `read_files()`를 사용하면 체크포인트와 스키마 위치를 직접 지정할 필요가 없습니다.

---

### Silver Layer: 데이터 정제와 품질 검증

Silver 레이어에서는 Bronze의 원시 데이터를 정제하고, **`CONSTRAINT ... EXPECT`** 구문으로 데이터 품질 규칙을 선언합니다.

```sql
-- =========================================================
-- [Silver] 주문 데이터 정제 및 품질 검증
-- CONSTRAINT: 위반 행에 대해 DROP ROW 또는 경고 처리합니다
-- =========================================================
CREATE OR REFRESH STREAMING TABLE silver_orders (
    -- 데이터 품질 규칙 (Data Quality Constraints)
    CONSTRAINT valid_order_id   EXPECT (order_id IS NOT NULL)                    ON VIOLATION DROP ROW,
    CONSTRAINT valid_amount     EXPECT (amount > 0)                              ON VIOLATION DROP ROW,
    CONSTRAINT valid_status     EXPECT (status IN ('COMPLETED', 'PENDING', 'CANCELLED')) ON VIOLATION DROP ROW,
    CONSTRAINT no_parse_errors  EXPECT (_rescued_data IS NULL)                   ON VIOLATION DROP ROW
)
COMMENT '정제된 주문 데이터 - 품질 검증 및 타입 변환 완료'
TBLPROPERTIES ('quality' = 'silver')
AS SELECT
    CAST(order_id    AS BIGINT)        AS order_id,
    CAST(customer_id AS BIGINT)        AS customer_id,
    TRIM(product_name)                 AS product_name,
    CAST(amount      AS DECIMAL(12,2)) AS amount,
    CAST(order_date  AS DATE)          AS order_date,
    UPPER(TRIM(status))                AS status,
    _source_file,
    _ingested_at
FROM STREAM(bronze_orders)
WHERE _rescued_data IS NULL;   -- 파싱 오류 행은 Silver로 승격하지 않습니다
```

```sql
-- =========================================================
-- [Silver] 고객 데이터 정제
-- =========================================================
CREATE OR REFRESH STREAMING TABLE silver_customers (
    CONSTRAINT valid_customer_id EXPECT (customer_id IS NOT NULL)          ON VIOLATION DROP ROW,
    CONSTRAINT valid_email       EXPECT (email IS NOT NULL AND email LIKE '%@%') ON VIOLATION DROP ROW,
    CONSTRAINT valid_city        EXPECT (city IS NOT NULL)                 ON VIOLATION WARN  -- WARN: 경고만 기록, 행은 유지
)
COMMENT '정제된 고객 데이터'
TBLPROPERTIES ('quality' = 'silver')
AS SELECT
    CAST(customer_id AS BIGINT)       AS customer_id,
    TRIM(name)                        AS name,
    LOWER(TRIM(email))                AS email,
    TRIM(city)                        AS city,
    CAST(registered_at AS TIMESTAMP)  AS registered_at,
    -- 스키마 진화로 추가된 컬럼 (없으면 NULL)
    phone,
    membership_level,
    _source_file,
    _ingested_at
FROM STREAM(bronze_customers)
WHERE _rescued_data IS NULL;
```

**`CONSTRAINT` 위반 처리 옵션 비교**:

| 옵션 | 동작 | 사용 시나리오 |
|------|------|--------------|
| `ON VIOLATION DROP ROW` | 위반 행을 제거하고 카운트를 기록합니다 | 핵심 비즈니스 규칙 (NULL 키, 음수 금액) |
| `ON VIOLATION WARN` | 행을 유지하고 경고를 기록합니다 | 선택적 필드, 데이터 품질 모니터링 용도 |
| (없음, 기본) | 행을 유지하고 위반 건수만 집계합니다 | 단순 품질 지표 수집 |
| `ON VIOLATION FAIL UPDATE` | 파이프라인 실행을 중단합니다 | 심각한 품질 위반 시 파이프라인 중단 필요 |

---

### Gold Layer: 비즈니스 집계

Gold 레이어는 **Materialized View (구체화 뷰)** 로 정의합니다. Streaming Table과 달리 Materialized View는 전체 데이터를 재집계하며, Silver 레이어가 업데이트될 때 자동으로 갱신됩니다.

```sql
-- =========================================================
-- [Gold] 일별 매출 요약
-- Materialized View: Silver 데이터 전체를 집계합니다
-- =========================================================
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_sales
COMMENT '일별 매출 KPI - BI 대시보드 소비용'
TBLPROPERTIES ('quality' = 'gold')
AS SELECT
    order_date,
    COUNT(*)                                                     AS order_count,
    SUM(amount)                                                  AS total_revenue,
    ROUND(AVG(amount), 2)                                        AS avg_order_value,
    COUNT(DISTINCT customer_id)                                  AS unique_customers,
    SUM(CASE WHEN status = 'COMPLETED' THEN amount ELSE 0 END)   AS completed_revenue,
    SUM(CASE WHEN status = 'CANCELLED' THEN 1     ELSE 0 END)    AS cancelled_count,
    ROUND(
        SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        1
    )                                                            AS completion_rate_pct
FROM silver_orders
GROUP BY order_date
ORDER BY order_date;
```

```sql
-- =========================================================
-- [Gold] 도시별 고객 및 매출 통계
-- =========================================================
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_by_city
COMMENT '도시별 고객 및 매출 통계'
TBLPROPERTIES ('quality' = 'gold')
AS SELECT
    c.city,
    COUNT(DISTINCT c.customer_id)        AS customer_count,
    COUNT(o.order_id)                    AS order_count,
    COALESCE(SUM(o.amount), 0)           AS total_revenue,
    COALESCE(ROUND(AVG(o.amount), 2), 0) AS avg_order_value,
    -- membership_level 분포 (Silver에서 스키마 진화로 추가된 컬럼)
    COUNT(CASE WHEN c.membership_level = 'GOLD'     THEN 1 END) AS gold_members,
    COUNT(CASE WHEN c.membership_level = 'PLATINUM' THEN 1 END) AS platinum_members
FROM silver_customers c
LEFT JOIN silver_orders o ON c.customer_id = o.customer_id
GROUP BY c.city
ORDER BY total_revenue DESC;
```

**Streaming Table vs Materialized View**:

| 구분 | Streaming Table | Materialized View |
|------|----------------|-------------------|
| **처리 방식** | 증분 처리 (새 데이터만) | 전체 재계산 |
| **소스** | 스트리밍 소스 (`STREAM(...)`) | 배치 소스 (테이블 전체) |
| **적합한 계층** | Bronze, Silver | Gold (집계, 조인) |
| **체크포인트** | 자동 관리 | 불필요 |
| **지연** | 낮음 (Near Real-Time) | 업스트림 완료 후 갱신 |

---

## 파이프라인 생성 및 실행

### 1단계: 파이프라인 생성

1. 왼쪽 사이드바에서 **Pipelines** 클릭
2. **Create Pipeline** 클릭
3. 아래 설정 입력:

| 항목 | 값 |
|------|-----|
| Pipeline name | `auto-loader-lab-pipeline` |
| Pipeline mode | **Triggered** (스케줄 배치) 또는 Continuous |
| Source Code | 위 SQL이 포함된 노트북 경로 |
| Target catalog | `training` |
| Target schema | `auto_loader_lab` |
| Cluster | Serverless 또는 Legacy (Fixed size: 1 worker) |

4. **Save** 클릭 후 **Start** 클릭

### 2단계: 파이프라인 실행 모니터링

파이프라인 실행 중 UI에서 확인할 수 있는 항목:

- **DAG 뷰**: Bronze → Silver → Gold 의존성 그래프
- **Data Quality 탭**: 각 테이블의 Constraint 위반 건수
- **Event Log**: 각 단계의 처리 건수, 소요 시간
- **Lineage**: 데이터 출처부터 최종 테이블까지의 흐름

---

## 결과 검증

```sql
-- Bronze 수집 확인
SELECT COUNT(*) AS bronze_orders_count FROM training.auto_loader_lab.bronze_orders;
SELECT COUNT(*) AS bronze_customers_count FROM training.auto_loader_lab.bronze_customers;
```

```sql
-- Silver 데이터 품질 검증
-- SDP UI Data Quality 탭에서도 확인할 수 있습니다
SELECT
    COUNT(*)                                         AS silver_orders,
    SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_order_id,
    SUM(CASE WHEN amount <= 0     THEN 1 ELSE 0 END)  AS invalid_amount,
    MIN(amount) AS min_amount,
    MAX(amount) AS max_amount
FROM training.auto_loader_lab.silver_orders;
```

```sql
-- Gold 일별 매출 확인
SELECT
    order_date,
    order_count,
    total_revenue,
    completion_rate_pct
FROM training.auto_loader_lab.gold_daily_sales
ORDER BY order_date DESC
LIMIT 10;
```

```sql
-- Gold 도시별 통계
SELECT * FROM training.auto_loader_lab.gold_customer_by_city;
```

**예상 결과** (실습 1~2 데이터 기준):

| city | customer_count | order_count | total_revenue |
|------|---------------|-------------|---------------|
| 서울  | ~16           | ~40         | ~8,000,000   |
| 부산  | ~14           | ~35         | ~7,000,000   |
| 대구  | ~12           | ~30         | ~6,000,000   |

---

## 심화 학습

### 변형 시나리오 1: 새 파일 추가 후 파이프라인 재실행

파이프라인을 다시 **Start** 하면:
- Bronze: 새로 도착한 파일만 수집합니다 (Checkpoint 기반).
- Silver: Bronze의 새 행만 처리합니다.
- Gold: Materialized View 전체를 재집계합니다.

```python
# 새 파일 추가
import random, json
from datetime import datetime, timedelta

new_orders = [
    f"{i},{random.randint(1000,9999)},새제품,{random.randint(10000,200000)},2025-04-01,COMPLETED"
    for i in range(201, 251)
]
dbutils.fs.put(
    "/Volumes/training/auto_loader_lab/raw_data/csv/orders/orders_batch3.csv",
    "order_id,customer_id,product_name,amount,order_date,status\n" + "\n".join(new_orders),
    overwrite=True
)
print("배치 3 생성 완료. 파이프라인을 다시 Start 하세요.")
```

### 변형 시나리오 2: Python API로 SDP 정의

SQL 대신 Python Decorator (데코레이터) API로 동일한 파이프라인을 정의할 수 있습니다.

```python
import dlt
from pyspark.sql.functions import current_timestamp, col, upper, trim

@dlt.table(
    name="bronze_orders_py",
    comment="Python API로 정의한 Bronze 주문 테이블"
)
def bronze_orders_py():
    return (
        spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "csv")
        .option("cloudFiles.inferColumnTypes", "true")
        .option("header", "true")
        .option("rescuedDataColumn", "_rescued_data")
        .load("/Volumes/training/auto_loader_lab/raw_data/csv/orders/")
        .withColumn("_source_file", col("_metadata.file_path"))
        .withColumn("_ingested_at", current_timestamp())
    )

@dlt.table(
    name="silver_orders_py",
    comment="Python API로 정의한 Silver 주문 테이블"
)
@dlt.expect_or_drop("valid_order_id", "order_id IS NOT NULL")
@dlt.expect_or_drop("valid_amount",   "amount > 0")
def silver_orders_py():
    return (
        dlt.read_stream("bronze_orders_py")
        .filter("_rescued_data IS NULL")
        .select(
            col("order_id").cast("bigint"),
            col("customer_id").cast("bigint"),
            trim(col("product_name")).alias("product_name"),
            col("amount").cast("decimal(12,2)"),
            col("order_date").cast("date"),
            upper(trim(col("status"))).alias("status"),
            col("_source_file"),
            col("_ingested_at")
        )
    )
```

### 트러블슈팅: 파이프라인 리셋

파이프라인을 처음부터 재실행해야 하는 경우:

1. 파이프라인 UI에서 우측 상단 `...` 메뉴 → **Full refresh all**
2. 모든 체크포인트가 초기화되고 전체 파일을 다시 처리합니다.
3. 대상 테이블도 초기화됩니다.

> **주의**: Full Refresh는 모든 데이터를 재처리하므로 시간이 오래 걸립니다. 특정 테이블만 리셋하려면 해당 테이블 노드를 우클릭하고 **Full refresh selected** 를 선택합니다.

---

## 정리

### 핵심 요약

| 개념 | 설명 |
|------|------|
| **`read_files()`** | SDP 전용 SQL 함수. Auto Loader(`cloudFiles`)를 내부적으로 사용합니다 |
| **Streaming Table** | 증분 처리. Bronze, Silver에 적합합니다 |
| **Materialized View** | 전체 재집계. Gold (집계, 조인) 에 적합합니다 |
| **`CONSTRAINT ... EXPECT`** | 선언적 데이터 품질 규칙. DROP ROW / WARN / FAIL 3가지 동작을 지원합니다 |
| **자동 체크포인트 관리** | SDP가 체크포인트, 스키마 위치를 자동으로 관리합니다 |
| **DAG 자동 생성** | 테이블 간 의존성을 선언하면 SDP가 실행 순서를 결정합니다 |

### SDP + Auto Loader 모범 사례

1. **Bronze는 원본 그대로**: 스키마 변환 없이 `*`로 수집합니다. `_metadata`, `_rescued_data`, `_ingested_at` 만 추가합니다.
2. **Silver에서 품질 적용**: `CONSTRAINT`로 품질 규칙을 선언합니다. Bronze에 적용하지 않습니다.
3. **Gold는 Materialized View**: 집계와 조인은 Materialized View로 정의합니다.
4. **`rescuedDataColumn` 필수**: Bronze에서 반드시 `_rescued_data`를 설정합니다.
5. **TBLPROPERTIES로 메타데이터**: `quality=bronze/silver/gold`, 소유자 등을 기록합니다.

### 다음 단계

- [데이터 검증과 트러블슈팅](validation-and-troubleshooting.md) — 파이프라인 전체의 데이터 품질을 검증하고 장애 상황을 해결합니다.

---

## 참고 링크

- [Databricks: Auto Loader with Delta Live Tables](https://docs.databricks.com/aws/en/ingestion/auto-loader/unity-catalog.html)
- [Databricks: `read_files()` function](https://docs.databricks.com/aws/en/sql/language-manual/functions/read_files.html)
- [Databricks: Delta Live Tables — Streaming Tables](https://docs.databricks.com/aws/en/delta-live-tables/streaming-tables.html)
- [Databricks: Data quality with Delta Live Tables](https://docs.databricks.com/aws/en/delta-live-tables/expectations.html)
- [Databricks: CREATE STREAMING TABLE](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-streaming-table.html)
- [Databricks: CREATE MATERIALIZED VIEW](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-materialized-view.html)
