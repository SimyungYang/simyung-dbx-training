# 구체화된 뷰(Materialized View) 상세

## 구체화된 뷰란?

**구체화된 뷰(Materialized View, MV)** 는 SQL 쿼리의 결과를 **미리 계산하여 물리적으로 저장**하는 데이터베이스 객체입니다. 일반 뷰는 매번 쿼리 시 재계산하지만, 구체화된 뷰는 저장된 결과를 즉시 반환하므로 **쿼리 성능이 비약적으로 향상**됩니다.

> 💡 **비유**: 일반 뷰는 "레시피(쿼리)"만 저장해 두고 매번 요리하는 것이고, 구체화된 뷰는 "완성된 요리"를 냉장고에 보관해 두고 바로 내놓는 것입니다.

---

## 일반 뷰와 구체화된 뷰 비교

| 비교 항목 | 일반 뷰 (View) | 구체화된 뷰 (MV) |
|-----------|--------------|----------------|
| **데이터 저장** | 저장하지 않습니다 (쿼리만 저장) | 결과를 물리적으로 저장합니다 |
| **쿼리 성능** | 매번 재계산하므로 느릴 수 있습니다 | 사전 계산된 결과로 빠릅니다 |
| **스토리지 비용** | 없음 | 결과 데이터 크기만큼 발생합니다 |
| **데이터 신선도** | 항상 최신 (쿼리 시 재계산) | 마지막 새로고침 시점의 데이터 |
| **생성 환경** | 모든 컴퓨트 | SDP 또는 SQL Warehouse |
| **적합한 사용** | 단순 필터링, 보안 뷰 | 복잡한 집계, 자주 조회되는 요약 |

---

## CREATE MATERIALIZED VIEW 문법

### 기본 생성

```sql
-- 일별 매출 집계를 구체화된 뷰로 생성
CREATE MATERIALIZED VIEW catalog.schema.mv_daily_revenue
AS
SELECT
    DATE(order_date) AS sale_date,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM catalog.schema.orders
GROUP BY DATE(order_date);
```

### 코멘트와 속성 지정

```sql
CREATE MATERIALIZED VIEW catalog.schema.mv_product_summary
COMMENT '제품별 매출 요약 — 매일 새벽 2시 갱신'
TBLPROPERTIES (
    'pipelines.autoOptimize.zOrderCols' = 'category'
)
AS
SELECT
    category,
    product_name,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_price
FROM catalog.schema.orders
GROUP BY category, product_name;
```

### 파티션 지정

```sql
CREATE MATERIALIZED VIEW catalog.schema.mv_monthly_sales
PARTITIONED BY (sale_month)
AS
SELECT
    DATE_TRUNC('MONTH', order_date) AS sale_month,
    region,
    SUM(amount) AS revenue
FROM catalog.schema.orders
GROUP BY 1, 2;
```

---

## 새로고침 (Refresh)

구체화된 뷰의 데이터를 최신 상태로 업데이트하는 과정을 **새로고침(Refresh)** 이라고 합니다.

### 수동 새로고침

```sql
-- 특정 구체화된 뷰를 수동으로 새로고침
REFRESH MATERIALIZED VIEW catalog.schema.mv_daily_revenue;
```

### 스케줄 새로고침

SQL Warehouse에서 생성한 구체화된 뷰는 스케줄을 설정하여 주기적으로 새로고침할 수 있습니다.

```sql
-- 매일 새벽 2시에 자동 새로고침 (CRON 표현식)
ALTER MATERIALIZED VIEW catalog.schema.mv_daily_revenue
SET SCHEDULE CRON '0 2 * * *' AT TIME ZONE 'Asia/Seoul';

-- 스케줄 확인
DESCRIBE MATERIALIZED VIEW catalog.schema.mv_daily_revenue;

-- 스케줄 제거
ALTER MATERIALIZED VIEW catalog.schema.mv_daily_revenue
DROP SCHEDULE;
```

### SDP 파이프라인에서의 새로고침

SDP 파이프라인 내에서 구체화된 뷰를 정의하면, 파이프라인 실행 시 **자동으로 새로고침**됩니다.

```sql
-- SDP 파이프라인 노트북 내
CREATE MATERIALIZED VIEW mv_customer_360
AS
SELECT
    c.customer_id,
    c.name,
    COUNT(o.order_id) AS total_orders,
    SUM(o.amount) AS lifetime_value,
    MAX(o.order_date) AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;
```

---

## 증분 갱신 vs 전체 재계산

Databricks는 가능한 경우 **증분 갱신(Incremental Refresh)** 을 수행하여 변경된 부분만 업데이트합니다.

| 갱신 방식 | 동작 | 성능 |
|----------|------|------|
| **증분 갱신** | 원본 데이터의 변경분만 계산하여 반영합니다 | 빠르고 효율적 |
| **전체 재계산** | 처음부터 전체 쿼리를 다시 실행합니다 | 느리지만 항상 정확 |

### 증분 갱신이 지원되는 조건

| 조건 | 설명 |
|------|------|
| **소스가 Delta 테이블** | 소스 테이블이 Delta Lake 형식이어야 합니다 |
| **단순 집계** | SUM, COUNT, MIN, MAX, AVG 같은 단순 집계가 포함된 경우 |
| **INNER/LEFT JOIN** | 특정 조인 패턴에서 증분 갱신이 가능합니다 |

증분 갱신이 불가능한 경우 Databricks는 자동으로 전체 재계산으로 전환합니다.

---

## Streaming Table과의 비교

구체화된 뷰와 스트리밍 테이블은 모두 데이터를 물리적으로 저장하지만, 사용 목적이 다릅니다.

| 비교 항목 | 구체화된 뷰 (MV) | 스트리밍 테이블 (ST) |
|-----------|----------------|-------------------|
| **데이터 처리** | 배치 기반 (전체/증분 재계산) | 증분 기반 (새 데이터만 추가) |
| **UPDATE/DELETE** | 소스의 변경사항을 반영합니다 | INSERT만 처리합니다 (Append-only) |
| **소스 유형** | 모든 테이블/뷰 | 스트리밍 소스 (Auto Loader, Kafka 등) |
| **주요 용도** | 집계/요약, 자주 조회되는 결과 | 실시간 이벤트 수집, 로그 처리 |
| **데이터 수정** | 소스 수정이 MV에 반영됩니다 | 한번 입력된 데이터는 수정 불가 |

### 선택 기준

```
이미 존재하는 데이터를 "요약/집계"하고 싶다 → Materialized View
새로 들어오는 데이터를 "계속 수집"하고 싶다 → Streaming Table
```

---

## 적합한 사용 사례

| 사용 사례 | 설명 | 예시 |
|----------|------|------|
| **대시보드 백엔드** | 자주 조회되는 집계를 미리 계산합니다 | 일별/월별 매출 요약 |
| **비용이 큰 조인** | 여러 테이블의 조인 결과를 캐시합니다 | Customer 360 뷰 |
| **실시간 리포팅** | 복잡한 쿼리를 즉시 응답합니다 | 실시간 KPI 대시보드 |
| **데이터 마트** | Gold 레이어의 최종 소비 테이블로 사용합니다 | 비즈니스 부서별 마트 |

### 비적합한 사용 사례

| 사용 사례 | 이유 | 대안 |
|----------|------|------|
| 실시간 이벤트 수집 | MV는 배치 기반으로 지연이 있습니다 | Streaming Table |
| 원본과 항상 동기화 | 새로고침 사이에 지연이 있습니다 | 일반 뷰 |
| 자주 변경되는 소스 | 빈번한 새로고침으로 비용이 증가합니다 | 일반 뷰 또는 캐시 |

---

## 관리 명령

```sql
-- 구체화된 뷰 정보 조회
DESCRIBE MATERIALIZED VIEW catalog.schema.mv_daily_revenue;

-- 새로고침 이력 확인
DESCRIBE HISTORY catalog.schema.mv_daily_revenue;

-- 구체화된 뷰 삭제
DROP MATERIALIZED VIEW IF EXISTS catalog.schema.mv_daily_revenue;

-- 스케줄 설정
ALTER MATERIALIZED VIEW catalog.schema.mv_daily_revenue
SET SCHEDULE CRON '0 */6 * * *' AT TIME ZONE 'Asia/Seoul';  -- 6시간마다

-- 스케줄 제거
ALTER MATERIALIZED VIEW catalog.schema.mv_daily_revenue
DROP SCHEDULE;
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **구체화된 뷰** | 쿼리 결과를 물리적으로 저장하여 빠른 조회를 제공하는 객체입니다 |
| **새로고침** | 수동, 스케줄, 또는 SDP 파이프라인을 통해 데이터를 최신화합니다 |
| **증분 갱신** | 가능한 경우 변경분만 계산하여 효율적으로 새로고침합니다 |
| **vs 일반 뷰** | 일반 뷰는 매번 재계산, MV는 사전 계산된 결과를 반환합니다 |
| **vs Streaming Table** | MV는 집계/요약, ST는 실시간 수집에 적합합니다 |

---

## 참고 링크

- [Databricks: Materialized views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-materialized-views.html)
- [Databricks: CREATE MATERIALIZED VIEW](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-materialized-view.html)
- [Databricks: Schedule materialized views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-materialized-view.html)
- [Azure Databricks: Materialized views](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-materialized-views)
