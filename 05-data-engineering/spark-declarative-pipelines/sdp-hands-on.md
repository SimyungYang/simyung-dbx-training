# SDP 실습 — Medallion 파이프라인 구축

## 시나리오

온라인 쇼핑몰의 **주문 데이터(JSON)**와 ** 고객 마스터 데이터(CDC)**를 수집하여, Medallion 아키텍처 기반의 분석 파이프라인을 구축합니다. Bronze(원본 수집) → Silver(정제·검증) → Gold(비즈니스 집계) 3계층 파이프라인을 SDP(Spark Declarative Pipelines)로 구현합니다.

| 계층 | 테이블 | 소스 | 설명 |
|------|--------|------|------|
| **Bronze**| bronze_orders | JSON 파일 (주문) | Auto Loader로 원본 수집 |
|  | bronze_customers | CDC 스트림 (고객) | CDC 데이터를 수집 |
| **Silver**| silver_orders | bronze_orders | 정제 + Expectations 적용 |
|  | silver_customers | bronze_customers | SCD Type 1 처리 |
| **Gold**| gold_customer_revenue | silver_orders + silver_customers | 고객별 매출 집계 |
|  | gold_daily_kpi | silver_orders | 일별 KPI 집계 |

---

## 사전 준비

### 1. 카탈로그 및 스키마 생성

```sql
-- Unity Catalog에 실습용 카탈로그와 스키마 생성
CREATE CATALOG IF NOT EXISTS training;
CREATE SCHEMA IF NOT EXISTS training.ecommerce;
```

### 2. 샘플 데이터 준비

실습에 사용할 JSON 데이터를 Volumes에 업로드합니다.

```sql
-- Volumes 생성
CREATE VOLUME IF NOT EXISTS training.ecommerce.raw_data;
```

```python
# 샘플 주문 데이터 생성 (JSON)
import json
from datetime import datetime, timedelta
import random

orders = []
for i in range(1, 101):
    orders.append({
        "order_id": i,
        "customer_id": random.randint(1, 20),
        "product": random.choice(["노트북", "키보드", "마우스", "모니터", "헤드셋"]),
        "amount": round(random.uniform(10000, 500000), 2),
        "order_date": (datetime(2025, 1, 1) + timedelta(days=random.randint(0, 90))).isoformat(),
        "status": random.choice(["COMPLETED", "PENDING", "CANCELLED"])
    })

# JSON 파일로 저장
path = "/Volumes/training/ecommerce/raw_data/orders/"
dbutils.fs.mkdirs(path)
dbutils.fs.put(f"{path}orders_batch1.json", "\n".join(json.dumps(o) for o in orders[:50]), True)
dbutils.fs.put(f"{path}orders_batch2.json", "\n".join(json.dumps(o) for o in orders[50:]), True)
```

```python
# 샘플 고객 데이터 생성 (CDC 형식)
customers = []
for i in range(1, 21):
    customers.append({
        "customer_id": i,
        "name": f"고객_{i:03d}",
        "email": f"customer{i}@example.com",
        "city": random.choice(["서울", "부산", "대구", "인천", "광주"]),
        "updated_at": datetime.now().isoformat(),
        "_change_type": "INSERT"
    })

path = "/Volumes/training/ecommerce/raw_data/customers/"
dbutils.fs.mkdirs(path)
dbutils.fs.put(f"{path}customers_initial.json", "\n".join(json.dumps(c) for c in customers), True)
```

---

## Step 1: 파이프라인 노트북 작성

하나의 SQL 노트북에 전체 파이프라인을 작성합니다. SDP에서는 **"무엇을 만들지"만 선언** 하면, 실행 순서와 의존성은 자동으로 관리됩니다.

### Bronze Layer — 원본 수집

```sql
-- ===== Bronze Layer =====
-- 원본 데이터를 변환 없이 그대로 수집합니다

-- 주문 데이터 수집 (Auto Loader)
CREATE OR REFRESH STREAMING TABLE bronze_orders
COMMENT 'Auto Loader로 수집한 원본 주문 데이터'
AS SELECT
    *,
    _metadata.file_path AS _source_file,     -- 원본 파일 경로
    _metadata.file_modification_time AS _file_time  -- 파일 수정 시간
FROM STREAM read_files(
    '/Volumes/training/ecommerce/raw_data/orders/',
    format => 'json'
);

-- 고객 CDC 데이터 수집
CREATE OR REFRESH STREAMING TABLE bronze_customers
COMMENT 'CDC 형식의 고객 마스터 데이터'
AS SELECT *
FROM STREAM read_files(
    '/Volumes/training/ecommerce/raw_data/customers/',
    format => 'json'
);
```

> 💡 **Bronze 계층의 원칙**: 소스 데이터를 **변환 없이 원본 그대로** 저장합니다. 메타데이터(`_metadata`)를 함께 저장하면 데이터 출처를 추적할 수 있습니다.

### Silver Layer — 정제 및 검증

```sql
-- ===== Silver Layer =====
-- 데이터 타입 변환, 정제, 품질 검증을 수행합니다

-- 주문 데이터 정제 + Expectations
CREATE OR REFRESH STREAMING TABLE silver_orders (
    -- 데이터 품질 규칙 (Expectations)
    CONSTRAINT valid_order_id EXPECT (order_id IS NOT NULL)
        ON VIOLATION DROP ROW,
    CONSTRAINT valid_amount EXPECT (amount > 0)
        ON VIOLATION DROP ROW,
    CONSTRAINT valid_status EXPECT (status IN ('COMPLETED', 'PENDING', 'CANCELLED'))
        ON VIOLATION DROP ROW
)
COMMENT '정제된 주문 데이터 (품질 검증 통과)'
AS SELECT
    CAST(order_id AS BIGINT) AS order_id,
    CAST(customer_id AS BIGINT) AS customer_id,
    TRIM(product) AS product,
    CAST(amount AS DECIMAL(12,2)) AS amount,
    CAST(order_date AS TIMESTAMP) AS order_date,
    UPPER(status) AS status,
    current_timestamp() AS _processed_at    -- 처리 시각 기록
FROM STREAM(bronze_orders);

-- 고객 마스터 (CDC → SCD Type 1: 최신값 유지)
CREATE OR REFRESH STREAMING TABLE silver_customers
COMMENT '고객 마스터 데이터 (SCD Type 1)';

APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customers)
KEYS (customer_id)
SEQUENCE BY updated_at
STORED AS SCD TYPE 1;
```

> 💡 **Expectations의 역할**: `ON VIOLATION DROP ROW`는 품질 규칙을 위반하는 행을 자동으로 제거합니다. 제거된 행의 수는 Pipeline UI에서 확인할 수 있어, 데이터 품질을 모니터링할 수 있습니다.

### Gold Layer — 비즈니스 집계

```sql
-- ===== Gold Layer =====
-- 비즈니스 분석에 직접 사용하는 집계 테이블입니다

-- 고객별 누적 매출
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_revenue
COMMENT '고객별 매출 요약 (완료된 주문만)'
AS SELECT
    c.customer_id,
    c.name,
    c.city,
    COUNT(o.order_id) AS total_orders,
    SUM(o.amount) AS total_revenue,
    AVG(o.amount) AS avg_order_value,
    MAX(o.order_date) AS last_order_date
FROM silver_orders o
JOIN silver_customers c ON o.customer_id = c.customer_id
WHERE o.status = 'COMPLETED'
GROUP BY c.customer_id, c.name, c.city;

-- 일별 KPI
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_kpi
COMMENT '일별 주요 성과 지표'
AS SELECT
    DATE(order_date) AS sale_date,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value
FROM silver_orders
WHERE status = 'COMPLETED'
GROUP BY DATE(order_date);
```

---

## Step 2: 파이프라인 생성 (UI)

1. 좌측 메뉴 **Workflows**→ **Pipelines**클릭
2. **Create Pipeline**버튼 클릭
3. 설정 입력:

| 설정 항목 | 값 |
|----------|-----|
| **Pipeline name**| `shop-medallion-pipeline` |
| **Source code**| 위에서 작성한 노트북 경로 |
| **Destination**| Catalog: `training`, Schema: `ecommerce` |
| **Compute**| Serverless 선택 |
| **Pipeline mode**| Triggered (실습용) |

4. **Create**클릭

---

## Step 3: 파이프라인 실행

1. Pipeline 상세 페이지에서 **Start**클릭
2. 실행 과정을 실시간으로 모니터링:

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 초기화 | 인프라를 프로비저닝합니다 |
| 2 | Bronze 수집 | Auto Loader로 데이터를 수집합니다 |
| 3 | Silver 변환 | 타입 변환, 정제를 수행합니다 |
| 4 | Gold 집계 | JOIN + GROUP BY로 비즈니스 집계를 생성합니다 |
| 5 | 완료 | 파이프라인 실행이 완료됩니다 |

---

## Step 4: 결과 확인 및 모니터링

### Pipeline UI에서 확인

| 확인 항목 | 위치 | 설명 |
|-----------|------|------|
| **DAG 그래프**| Pipeline 메인 화면 | 테이블 간 의존성과 데이터 흐름을 시각적으로 표시합니다 |
| **Expectations 결과**| 각 테이블 클릭 → Quality 탭 | 통과/실패 행 수를 확인합니다 |
| ** 처리 건수**| 각 테이블 클릭 → Details | INSERT/UPDATE/DELETE 건수를 확인합니다 |
| ** 이벤트 로그** | Updates 탭 | 실행 이력, 에러, 경고를 확인합니다 |

### SQL로 결과 확인

```sql
-- Bronze: 수집된 원본 데이터 확인
SELECT COUNT(*) AS total_rows, MIN(order_date), MAX(order_date)
FROM training.ecommerce.bronze_orders;

-- Silver: 정제된 데이터 확인
SELECT status, COUNT(*) AS cnt
FROM training.ecommerce.silver_orders
GROUP BY status;

-- Silver: Expectations로 제거된 행 확인
-- (Pipeline UI의 Quality 탭에서도 확인 가능)

-- Gold: 비즈니스 집계 확인
SELECT * FROM training.ecommerce.gold_customer_revenue
ORDER BY total_revenue DESC
LIMIT 10;

SELECT * FROM training.ecommerce.gold_daily_kpi
ORDER BY sale_date;
```

---

## Step 5: 증분 데이터 추가 및 재실행

SDP의 핵심 장점 중 하나는 **증분 처리**입니다. 새 데이터를 추가하고 파이프라인을 다시 실행하면, 새 데이터만 처리합니다.

### 새 주문 데이터 추가

```python
# 추가 주문 데이터 생성 (batch3)
new_orders = []
for i in range(101, 151):
    new_orders.append({
        "order_id": i,
        "customer_id": random.randint(1, 20),
        "product": random.choice(["태블릿", "스피커", "충전기"]),
        "amount": round(random.uniform(15000, 300000), 2),
        "order_date": (datetime(2025, 4, 1) + timedelta(days=random.randint(0, 30))).isoformat(),
        "status": random.choice(["COMPLETED", "PENDING"])
    })

path = "/Volumes/training/ecommerce/raw_data/orders/"
dbutils.fs.put(f"{path}orders_batch3.json", "\n".join(json.dumps(o) for o in new_orders), True)
```

### 고객 데이터 변경 (CDC 시뮬레이션)

```python
# 기존 고객 정보 업데이트 (도시 변경)
updates = [
    {"customer_id": 1, "name": "고객_001", "email": "customer1@example.com",
     "city": "제주", "updated_at": datetime.now().isoformat(), "_change_type": "UPDATE"},
    {"customer_id": 5, "name": "고객_005", "email": "vip5@example.com",
     "city": "서울", "updated_at": datetime.now().isoformat(), "_change_type": "UPDATE"},
]

path = "/Volumes/training/ecommerce/raw_data/customers/"
dbutils.fs.put(f"{path}customers_update1.json", "\n".join(json.dumps(u) for u in updates), True)
```

### 파이프라인 재실행

Pipeline UI에서 **Start**클릭 (또는 CLI: `databricks pipelines start-update --pipeline-id <id>`)

> 💡 ** 증분 처리 확인**: 재실행 후 Bronze 테이블의 행 수가 증가했는지, Gold 테이블의 집계가 업데이트되었는지 확인하세요. Auto Loader는 이전에 처리한 파일을 건너뛰고 **새 파일(batch3, update1)만 처리**합니다.

---

## Step 6: Full Refresh vs 증분 업데이트

| 모드 | 설명 | 사용 시기 |
|------|------|----------|
| **증분 업데이트** (기본) | 새 데이터만 처리합니다 | 일반적인 운영 |
| **Full Refresh**| 모든 데이터를 처음부터 다시 처리합니다 | 로직 변경, 데이터 복구 |

```bash
# CLI로 Full Refresh 실행
databricks pipelines start-update --pipeline-id <id> --full-refresh
```

> ⚠️ **Full Refresh 주의**: Streaming Table의 Full Refresh는 모든 데이터를 다시 수집합니다. 대용량 데이터의 경우 시간과 비용이 많이 소요될 수 있으므로 신중하게 사용하세요.

---

## 트러블슈팅

### 자주 발생하는 오류

| 증상 | 원인 | 해결 방법 |
|------|------|----------|
| "**Table not found**" | 소스 테이블이 아직 생성되지 않음 | SDP가 의존성을 자동 관리하므로, 전체 파이프라인을 다시 실행합니다 |
| "**Schema mismatch**" | JSON 스키마가 변경됨 | `cloudFiles.schemaEvolutionMode`를 `addNewColumns`로 설정합니다 |
| **Expectation으로 모든 행이 제거됨**| 품질 규칙이 너무 엄격함 | 규칙을 완화하거나, `ON VIOLATION FAIL UPDATE`로 변경하여 원인을 조사합니다 |
| "**Column not found**" | Silver 쿼리에서 참조한 컬럼이 Bronze에 없음 | Bronze 스키마를 `DESCRIBE TABLE`로 확인합니다 |
| **파이프라인이 시작되지 않음** | 권한 부족 | 카탈로그/스키마에 대한 `CREATE TABLE` 권한을 확인합니다 |
| "**Auto Loader detected new files but schema changed**" | 새 파일의 스키마가 다름 | 스키마 힌트를 제공하거나, 스키마 위치를 리셋합니다 |

### 유용한 디버깅 쿼리

```sql
-- 파이프라인 이벤트 로그 확인
SELECT * FROM event_log(TABLE(training.ecommerce.bronze_orders))
ORDER BY timestamp DESC
LIMIT 20;

-- Expectations 결과 상세
SELECT * FROM event_log(TABLE(training.ecommerce.silver_orders))
WHERE event_type = 'flow_progress'
ORDER BY timestamp DESC;
```

---

## 클린업

실습이 끝나면 리소스를 정리합니다.

```sql
-- 파이프라인 삭제: Pipeline UI에서 Delete 또는
-- databricks pipelines delete --pipeline-id <id>

-- 테이블 삭제
DROP TABLE IF EXISTS training.ecommerce.bronze_orders;
DROP TABLE IF EXISTS training.ecommerce.bronze_customers;
DROP TABLE IF EXISTS training.ecommerce.silver_orders;
DROP TABLE IF EXISTS training.ecommerce.silver_customers;
DROP MATERIALIZED VIEW IF EXISTS training.ecommerce.gold_customer_revenue;
DROP MATERIALIZED VIEW IF EXISTS training.ecommerce.gold_daily_kpi;

-- 볼륨의 데이터 삭제
-- dbutils.fs.rm("/Volumes/training/ecommerce/raw_data/", True)

-- 스키마 삭제 (선택)
-- DROP SCHEMA IF EXISTS training.ecommerce CASCADE;
```

---

## 정리

| 단계 | 배운 내용 |
|------|----------|
| **Bronze**| Auto Loader + `read_files()`로 파일을 자동 수집합니다. 원본을 그대로 저장합니다 |
| **Silver**| 데이터 타입 변환, TRIM, UPPER 등으로 정제합니다. Expectations로 품질을 검증합니다 |
| **Gold**| JOIN, GROUP BY로 비즈니스 집계를 만듭니다. Materialized View로 성능을 최적화합니다 |
| **CDC**| `APPLY CHANGES INTO`로 SCD Type 1 테이블을 자동 관리합니다 |
| ** 증분 처리**| 새 파일만 자동으로 감지하여 처리합니다. Full Refresh도 가능합니다 |
| ** 모니터링** | Pipeline UI에서 DAG, Expectations, 처리 건수를 확인합니다 |

---

## 참고 링크

- [Databricks: SDP Tutorial](https://docs.databricks.com/aws/en/delta-live-tables/tutorial.html)
- [Databricks: SDP SQL Reference](https://docs.databricks.com/aws/en/delta-live-tables/sql-ref.html)
- [Databricks: Expectations](https://docs.databricks.com/aws/en/delta-live-tables/expectations.html)
- [Databricks: APPLY CHANGES](https://docs.databricks.com/aws/en/delta-live-tables/cdc.html)
