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

## 증분 새로고침 내부 동작 — Change Tracking

MV의 증분 갱신(Incremental Refresh)이 어떻게 동작하는지 내부 메커니즘을 이해하면, 성능 문제를 진단하고 최적의 MV를 설계할 수 있습니다.

### Change Tracking 원리

Delta Lake의 **트랜잭션 로그(_delta_log)**는 테이블에 대한 모든 변경 사항을 기록합니다. MV의 증분 갱신은 이 로그를 활용하여 동작합니다.

```
1. MV 새로고침 시작
2. 마지막 새로고침 시점의 Delta 버전 확인 (예: version 42)
3. 소스 테이블의 현재 버전 확인 (예: version 50)
4. version 42 ~ 50 사이의 변경 사항(AddFile, RemoveFile) 분석
5. 변경된 파일에서 영향받는 행만 추출
6. 해당 행에 대해서만 집계를 재계산
7. MV 결과에 증분 적용
```

### 증분 갱신이 가능한 쿼리 패턴

| 패턴 | 증분 갱신 | 설명 |
|------|:---------:|------|
| `SELECT ... GROUP BY ... (SUM, COUNT, MIN, MAX, AVG)` | ✅ | 단순 집계는 변경분만 반영 가능 |
| `SELECT ... INNER JOIN ... GROUP BY` | ✅ | INNER JOIN + 집계 |
| `SELECT ... WHERE ... GROUP BY` | ✅ | 필터 + 집계 |
| `SELECT DISTINCT ...` | ✅ | 중복 제거 |
| `SELECT ... UNION ALL ...` | ✅ | 단순 합집합 |

### 증분 갱신이 불가능한 경우

다음 패턴에서는 **전체 재계산(Full Recomputation)**으로 폴백됩니다.

| 패턴 | 이유 | 대안 |
|------|------|------|
| **OUTER JOIN (LEFT/RIGHT/FULL)** | NULL 매칭 변경 추적이 복잡합니다 | INNER JOIN으로 변환하거나, 별도 테이블로 분리 |
| **Non-deterministic 함수** | `rand()`, `uuid()`, `current_timestamp()` 등은 재실행 시 결과가 다름 | 소스 테이블에 미리 계산하여 저장 |
| **Window 함수 (RANK, ROW_NUMBER)** | 전체 순위가 변경될 수 있어 증분 불가 | Gold 레이어에서 별도 처리 |
| **EXISTS / NOT EXISTS 서브쿼리** | 서브쿼리 결과 변경 추적 어려움 | JOIN으로 변환 |
| **HAVING 절** | 그룹 필터 변경 추적이 복잡 | WHERE로 이동 가능한지 검토 |

> ⚠️ **Gotcha**: MV가 증분 갱신을 사용하는지 전체 재계산을 사용하는지는 **이벤트 로그에서 확인**할 수 있습니다. `DESCRIBE HISTORY`로 각 새로고침의 `operationMetrics`를 조회하면 `numOutputRows`(출력 행)와 `numSourceRows`(소스 행)를 비교하여 증분 여부를 판단할 수 있습니다.

```sql
-- 새로고침 방식 확인
DESCRIBE HISTORY catalog.schema.mv_daily_revenue;
-- operationMetrics의 numTargetRowsInserted, numTargetRowsUpdated 확인
-- 전체 재계산: numSourceRows ≈ 소스 전체 행 수
-- 증분 갱신: numSourceRows << 소스 전체 행 수
```

---

## MV vs 사전 집계 테이블(Pre-aggregation Table) 비교

"MV 대신 그냥 집계 테이블을 CTAS로 만들면 안 되나요?"라는 질문을 자주 받습니다. 두 접근법의 차이를 명확히 비교합니다.

| 비교 항목 | Materialized View | 사전 집계 테이블 (CTAS + 스케줄) |
|-----------|-------------------|-------------------------------|
| **갱신 방식** | 증분/자동 | 전체 재생성 (DROP + CREATE 또는 MERGE) |
| **스키마 관리** | 자동 (소스 변경 시 자동 반영) | 수동 (ALTER TABLE 필요) |
| **의존성 추적** | Unity Catalog 리니지에 자동 반영 | 별도 관리 필요 |
| **최적화** | Predictive Optimization 자동 적용 | 수동 OPTIMIZE 필요 |
| **쿼리 리라이트** | 쿼리 최적화기가 MV를 자동 활용 가능 (향후 기능) | 불가 |
| **실패 복구** | 이전 버전 유지 (새로고침 실패해도 기존 데이터 유효) | 실패 시 데이터 부재 위험 |
| **권한 관리** | MV에 직접 GRANT 가능 | 테이블에 GRANT (동일) |
| **비용** | 증분 갱신으로 비용 절감 | 매번 전체 재계산 비용 |

> 💡 **SA 권장**: 단순 집계(SUM, COUNT, AVG)이고 소스가 Delta 테이블이면 **MV가 항상 유리**합니다. 사전 집계 테이블은 MV가 지원하지 않는 복잡한 로직(OUTER JOIN, Window 함수 등)이 필요한 경우에만 사용합니다.

---

## 대규모 MV 관리 전략

수십~수백 개의 MV를 운영하는 엔터프라이즈 환경에서의 관리 전략입니다.

### MV 분류 체계

| 분류 | 갱신 빈도 | 예시 | 스케줄 |
|------|----------|------|--------|
| **핫 MV** | 실시간~1시간 | 실시간 대시보드 백엔드 | SDP Continuous 또는 1시간 스케줄 |
| **웜 MV** | 일 1~4회 | 일일 KPI, 운영 리포트 | CRON 스케줄 (새벽/업무 시작 전) |
| **콜드 MV** | 주 1회~월 1회 | 월간 재무 리포트, 감사 보고서 | 수동 또는 월간 스케줄 |

### 스케줄 최적화 전략

```sql
-- 핫 MV: SDP 파이프라인에서 관리 (자동 갱신)
-- SDP 파이프라인 노트북 내:
CREATE MATERIALIZED VIEW mv_realtime_metrics
AS SELECT ...;

-- 웜 MV: SQL Warehouse 스케줄 (비즈니스 시간 외)
ALTER MATERIALIZED VIEW mv_daily_summary
SET SCHEDULE CRON '0 5 * * *' AT TIME ZONE 'Asia/Seoul';  -- 매일 05:00

-- 콜드 MV: 수동 새로고침 또는 장기 스케줄
ALTER MATERIALIZED VIEW mv_monthly_report
SET SCHEDULE CRON '0 6 1 * *' AT TIME ZONE 'Asia/Seoul';  -- 매월 1일 06:00
```

### MV 새로고침 순서 관리

MV 간에 의존성이 있는 경우(MV_A를 소스로 MV_B가 참조), 새로고침 순서가 중요합니다.

```
방법 1: SDP 파이프라인에서 관리 (권장)
  → SDP가 의존성을 자동으로 분석하여 올바른 순서로 갱신

방법 2: Lakeflow Jobs로 순서 관리
  → Task 1: REFRESH MV_A → Task 2: REFRESH MV_B (의존성 설정)

방법 3: 독립 스케줄 (비권장)
  → MV_A 05:00, MV_B 06:00 (시간차로 순서 보장, 실패 시 문제)
```

---

## MV와 Predictive Optimization 연동

Predictive Optimization이 활성화되면 MV에도 자동 최적화가 적용됩니다.

| 자동 최적화 | MV에 대한 동작 |
|------------|--------------|
| **자동 OPTIMIZE** | MV의 소형 파일을 자동으로 병합합니다. 빈번한 증분 갱신으로 소형 파일이 쌓이는 문제를 해결합니다 |
| **자동 VACUUM** | MV의 이전 버전 파일을 자동으로 정리합니다 |
| **자동 ANALYZE** | MV의 통계를 자동으로 갱신하여 MV를 조회하는 쿼리의 실행 계획을 최적화합니다 |

> 💡 **실무 팁**: MV를 많이 사용하는 환경에서는 **반드시 Predictive Optimization을 활성화**하는 것을 권장합니다. MV의 증분 갱신은 소형 파일을 누적시키는 경향이 있는데, 자동 OPTIMIZE가 이를 해결합니다.

```sql
-- MV가 속한 스키마에 Predictive Optimization 활성화
ALTER SCHEMA catalog.analytics ENABLE PREDICTIVE OPTIMIZATION;
```

---

## 비용 분석 — MV 유지 비용 vs 쿼리 비용 절감

MV 도입을 결정할 때 가장 중요한 것은 **유지 비용과 쿼리 비용 절감의 손익 분기점**입니다.

### MV 유지 비용 구성

| 비용 항목 | 설명 |
|-----------|------|
| **새로고침 컴퓨트** | 증분/전체 재계산에 사용되는 DBU |
| **스토리지** | MV 결과 데이터 저장 비용 |
| **자동 OPTIMIZE** | Predictive Optimization에 의한 파일 병합 DBU |

### 쿼리 비용 절감 구성

| 절감 항목 | 설명 |
|-----------|------|
| **쿼리 DBU 절감** | 매번 전체 테이블 집계 대신 MV 직접 조회 |
| **SQL Warehouse 시간 절감** | 복잡한 쿼리 실행 시간 감소 → Warehouse 시간 절약 |
| **사용자 대기 시간 절감** | 대시보드 로딩 시간 감소 → 생산성 향상 (정량화 어려움) |

### 손익 분기점 계산 예시

```
시나리오: 일별 매출 집계 MV
  - 소스 테이블: 1억 행, 50GB
  - 집계 결과: 365행, 10KB
  - 쿼리 빈도: 일 200회 (대시보드 자동 갱신 포함)
  - SQL Warehouse: Small (8 DBU/시간)

MV 없이 매번 쿼리:
  200회/일 × 5초/쿼리 × (8 DBU/3600초) ≈ 2.22 DBU/일

MV 유지:
  새로고침: 1회/일 × 30초 × (8 DBU/3600초) ≈ 0.07 DBU/일
  스토리지: 무시할 수준

절감 효과: 2.22 - 0.07 = 2.15 DBU/일 = ~64.5 DBU/월
→ MV가 월 64.5 DBU (약 $45)를 절약합니다
```

> 💡 **판단 기준**: 소스 데이터가 크고, 집계 결과가 작으며, 쿼리 빈도가 높을수록 MV의 ROI가 높아집니다. 반대로 소스가 작거나 쿼리가 드문 경우에는 MV 유지가 오히려 비용을 증가시킬 수 있습니다.

---

## Edge Cases와 주의사항

### MV가 예상대로 동작하지 않는 경우

| 상황 | 증상 | 해결 방법 |
|------|------|----------|
| **소스 스키마 변경** | 새로고침 실패 | MV DROP + 재생성, 또는 소스 스키마 호환성 유지 |
| **소스 테이블 삭제** | MV 조회 시 에러 | MV 삭제 후 새 소스로 재생성 |
| **매우 큰 소스 + 전체 재계산** | 새로고침 타임아웃 | 쿼리 최적화, 파티셔닝 적용, 또는 Warehouse 크기 증가 |
| **동시 새로고침 충돌** | 두 새로고침이 동시에 트리거됨 | 스케줄 간격 조정, SDP 파이프라인 사용 |
| **MV 위에 MV (체이닝)** | 지원되지만 의존성 관리 주의 | SDP 파이프라인에서 의존성 자동 관리 |

### MV 새로고침 시 잠금(Lock) 동작

MV 새로고침 중에도 **기존 데이터에 대한 읽기는 계속 가능**합니다(MVCC 덕분). 새로고침이 완료되면 원자적(atomic)으로 새 버전으로 전환됩니다. 따라서 새로고침 중 대시보드가 중단되지 않습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **구체화된 뷰** | 쿼리 결과를 물리적으로 저장하여 빠른 조회를 제공하는 객체입니다 |
| **새로고침** | 수동, 스케줄, 또는 SDP 파이프라인을 통해 데이터를 최신화합니다 |
| **증분 갱신** | 가능한 경우 변경분만 계산하여 효율적으로 새로고침합니다 |
| **증분 갱신 불가** | OUTER JOIN, Window 함수, Non-deterministic 함수 등은 전체 재계산 |
| **vs 일반 뷰** | 일반 뷰는 매번 재계산, MV는 사전 계산된 결과를 반환합니다 |
| **vs Streaming Table** | MV는 집계/요약, ST는 실시간 수집에 적합합니다 |
| **Predictive Optimization** | MV의 소형 파일 병합과 통계 갱신을 자동화합니다 |
| **비용 분석** | 쿼리 빈도가 높고 소스가 큰 경우 MV의 ROI가 높습니다 |

---

## 참고 링크

- [Databricks: Materialized views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-materialized-views.html)
- [Databricks: CREATE MATERIALIZED VIEW](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-materialized-view.html)
- [Databricks: Schedule materialized views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-materialized-view.html)
- [Azure Databricks: Materialized views](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-materialized-views)
