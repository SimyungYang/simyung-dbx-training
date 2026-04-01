# 활용 시나리오와 내부 동작

## 리니지 활용 시나리오

### 시나리오 1: 스키마 변경 영향도 분석

`silver_orders` 테이블의 `amount` 컬럼을 `total_amount`로 이름을 변경하려 합니다.

```sql
-- 이 컬럼을 참조하는 모든 downstream 확인
SELECT DISTINCT
    target_table_full_name,
    target_column_name
FROM system.lineage.column_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
    AND source_column_name = 'amount';

-- 결과:
-- gold_daily_revenue.total_revenue
-- gold_customer_360.lifetime_revenue
-- 매출 대시보드 (dataset query)
-- → 이 3곳도 함께 변경해야 합니다
```

### 시나리오 2: 데이터 품질 이슈 역추적

`gold_daily_revenue` 테이블에서 특정 날짜의 매출이 비정상적으로 낮습니다.

```sql
-- upstream 추적: 이 테이블의 소스 확인
SELECT source_table_full_name
FROM system.lineage.table_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue';
-- → silver_orders

-- silver_orders의 upstream 확인
SELECT source_table_full_name
FROM system.lineage.table_lineage
WHERE target_table_full_name = 'production.ecommerce.silver_orders';
-- → bronze_orders

-- → bronze_orders에서 해당 날짜의 원본 데이터를 확인하여 원인 파악
```

---

## 테이블 리니지 vs 컬럼 리니지 내부 동작

### 리니지 수집 메커니즘

Unity Catalog의 리니지는 **쿼리 실행 시 자동으로**수집됩니다. 별도의 설정이나 에이전트 설치가 필요하지 않습니다.

| 단계 | 설명 |
|------|------|
| 1. ** 쿼리 파싱**| SQL/Spark 쿼리를 파싱하여 AST(Abstract Syntax Tree)를 생성합니다 |
| 2. ** 논리 계획 분석**| Catalyst Optimizer의 논리 계획에서 소스/타겟 테이블과 컬럼을 추출합니다 |
| 3. ** 변환 추적**| 각 컬럼이 어떤 소스 컬럼에서 파생되었는지 (SELECT, JOIN, CASE WHEN 등) 추적합니다 |
| 4. ** 메타데이터 저장**| 추출된 리니지 정보를 UC 메타스토어의 리니지 테이블에 저장합니다 |
| 5. ** 주기적 집계** | 시스템 테이블(system.lineage.*)에 집계하여 조회 가능하게 합니다 |

### 테이블 리니지 상세

```sql
-- system.lineage.table_lineage 테이블 스키마
SELECT * FROM system.lineage.table_lineage LIMIT 0;
-- 주요 컬럼:
--   source_table_full_name    : 소스 테이블 (catalog.schema.table)
--   source_type               : TABLE, VIEW, STREAMING_TABLE 등
--   target_table_full_name    : 타겟 테이블
--   target_type               : TABLE, VIEW, MODEL, DASHBOARD 등
--   created_by                : 리니지를 생성한 사용자/서비스 프린시펄
--   event_time                : 리니지가 기록된 시간
```

### 컬럼 리니지 상세

컬럼 리니지는 테이블 리니지보다 더 세밀합니다. SQL의 **표현식을 분석** 하여 컬럼 간 관계를 추적합니다.

| SQL 패턴 | 리니지 추적 여부 | 설명 |
|----------|---------------|------|
| `SELECT a FROM t` | ✅ 직접 매핑 | t.a → result.a |
| `SELECT a + b AS total FROM t` | ✅ 복합 매핑 | t.a, t.b → result.total |
| `SELECT CASE WHEN ... END AS status` | ✅ 조건부 매핑 | 관련 컬럼 모두 추적 |
| `SELECT * FROM t1 JOIN t2 ON ...` | ✅ JOIN 추적 | 양쪽 테이블의 컬럼 모두 추적 |
| `SELECT a FROM t WHERE b > 10` | ⚠️ 부분 추적 | a의 리니지는 추적, WHERE의 b는 필터로 기록 |
| UDF(User Defined Function) | ❌ 제한적 | UDF 내부의 컬럼 변환은 추적하기 어려움 |
| Dynamic SQL (문자열 조합) | ❌ 불가 | 런타임에 생성되는 SQL은 파싱 불가 |

```sql
-- 컬럼 리니지 상세 조회
SELECT
    source_table_full_name,
    source_column_name,
    target_table_full_name,
    target_column_name,
    event_time
FROM system.lineage.column_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue'
ORDER BY target_column_name, event_time DESC;
```

---

## system.lineage 테이블 활용 고급 쿼리

### 전체 리니지 그래프 탐색

```sql
-- 특정 테이블의 전체 downstream 재귀 탐색 (3단계까지)
WITH RECURSIVE downstream AS (
    -- 1단계: 직접 downstream
    SELECT
        source_table_full_name,
        target_table_full_name,
        target_type,
        1 AS depth
    FROM system.lineage.table_lineage
    WHERE source_table_full_name = 'production.ecommerce.bronze_orders'

    UNION ALL

    -- 2~3단계: 간접 downstream
    SELECT
        d.target_table_full_name AS source_table_full_name,
        l.target_table_full_name,
        l.target_type,
        d.depth + 1 AS depth
    FROM downstream d
    JOIN system.lineage.table_lineage l
        ON d.target_table_full_name = l.source_table_full_name
    WHERE d.depth < 3
)
SELECT DISTINCT
    source_table_full_name,
    target_table_full_name,
    target_type,
    depth
FROM downstream
ORDER BY depth, target_table_full_name;
```

### 미사용 테이블 식별

```sql
-- 최근 90일간 downstream이 없는 테이블 (미사용 후보)
SELECT t.table_catalog, t.table_schema, t.table_name, t.created
FROM system.information_schema.tables t
LEFT JOIN system.lineage.table_lineage l
    ON t.table_catalog || '.' || t.table_schema || '.' || t.table_name = l.source_table_full_name
    AND l.event_time > DATEADD(day, -90, CURRENT_TIMESTAMP())
WHERE l.source_table_full_name IS NULL
  AND t.table_catalog = 'production'
  AND t.table_schema = 'ecommerce'
ORDER BY t.created;
```

### 가장 많이 참조되는 테이블 (핵심 테이블 식별)

```sql
-- 가장 많은 downstream을 가진 테이블 Top 20
SELECT
    source_table_full_name,
    COUNT(DISTINCT target_table_full_name) AS downstream_count,
    COLLECT_SET(target_type) AS target_types
FROM system.lineage.table_lineage
WHERE event_time > DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY source_table_full_name
ORDER BY downstream_count DESC
LIMIT 20;
```

---
