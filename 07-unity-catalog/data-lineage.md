# 데이터 리니지

## 리니지란?

> 💡 **데이터 리니지(Data Lineage)** 란 데이터가 **어디서 왔고(upstream), 어디로 가는지(downstream)** 를 추적하는 기능입니다. "이 테이블의 데이터는 어떤 소스에서 왔지?", "이 컬럼을 변경하면 어디에 영향이 가지?"라는 질문에 답할 수 있습니다.

---

## 왜 리니지가 중요한가요?

| 사용 사례 | 설명 |
|-----------|------|
| **영향도 분석** | 소스 테이블의 스키마를 변경할 때, 어떤 downstream 테이블과 대시보드에 영향이 있는지 확인합니다 |
| **데이터 품질 추적** | Gold 테이블에 이상이 발견되면, 어느 단계(Bronze? Silver?)에서 문제가 생겼는지 역추적합니다 |
| **규정 준수** | 개인정보(PII)가 어떤 경로로 흘러가는지 파악하여 GDPR, 개인정보보호법 등을 준수합니다 |
| **문서화 자동화** | 수동으로 데이터 흐름을 문서화하는 대신, 자동으로 추적됩니다 |
| **마이그레이션 계획** | 테이블을 변경/삭제할 때, 영향 받는 모든 객체를 사전에 파악합니다 |

---

## 리니지의 범위

Unity Catalog는 **테이블 수준**과 **컬럼 수준** 리니지를 모두 자동으로 추적합니다.

### 테이블 수준 리니지

| 단계 | 소스/대상 | 설명 |
|------|-----------|------|
| 소스 | MySQL (외부 소스) | 원본 데이터입니다 |
| Bronze | bronze_orders | 원본 데이터를 수집합니다 |
| Silver | silver_orders | 정제된 데이터입니다 |
| Gold | gold_daily_revenue | 일별 매출 집계 → 매출 대시보드에 활용 |
| Gold | gold_customer_360 | 고객 통합 뷰 → 고객 분석 대시보드 및 이탈 예측 모델에 활용 |

### 컬럼 수준 리니지

```
silver_orders.total_amount
  ← bronze_orders.raw_data:amount (JSON에서 추출)

gold_daily_revenue.total_revenue
  ← silver_orders.total_amount (SUM 집계)

gold_customer_360.lifetime_revenue
  ← silver_orders.total_amount (고객별 SUM)
```

---

## 리니지 시각화

![Unity Catalog 데이터 리니지 그래프 개요](https://docs.databricks.com/aws/en/assets/images/uc-lineage-overview-d654e26bc9af96d30f971da157eb1497.png)

![컬럼 수준 리니지 예시](https://docs.databricks.com/aws/en/assets/images/uc-lineage-column-lineage-89a091244240d677fe7a6e0503076089.png)

*출처: [Databricks 공식 문서](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-lineage.html)*

---

## 리니지 확인 방법

### Catalog Explorer UI

1. **Catalog** → 테이블 선택
2. **Lineage** 탭 클릭
3. Upstream(이 테이블의 소스)과 Downstream(이 테이블을 참조하는 객체)을 시각적으로 확인합니다

### 시스템 테이블로 SQL 조회

```sql
-- 특정 테이블의 downstream (이 테이블을 참조하는 모든 객체)
SELECT
    target_table_full_name AS downstream_table,
    target_type,
    target_column_name
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
ORDER BY target_table_full_name;

-- 특정 테이블의 upstream (이 테이블이 참조하는 소스)
SELECT
    source_table_full_name AS upstream_table,
    source_type,
    source_column_name
FROM system.lineage.table_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue'
ORDER BY source_table_full_name;

-- 컬럼 수준 리니지: total_revenue가 어디서 왔는지
SELECT
    source_table_full_name,
    source_column_name,
    target_column_name
FROM system.lineage.column_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue'
    AND target_column_name = 'total_revenue';

-- PII 컬럼의 전파 경로 추적
SELECT
    source_table_full_name,
    source_column_name,
    target_table_full_name,
    target_column_name
FROM system.lineage.column_lineage
WHERE source_column_name IN ('email', 'phone', 'ssn', 'name')
ORDER BY source_table_full_name, target_table_full_name;
```

---

## 리니지가 추적되는 작업

| 작업 유형 | 추적 여부 | 설명 |
|-----------|----------|------|
| **SQL 쿼리** (CTAS, INSERT, MERGE) | ✅ | SQL Warehouse, Notebook에서 실행한 쿼리 |
| **SDP Pipeline** | ✅ | Streaming Table, Materialized View의 소스 추적 |
| **Spark DataFrame** | ✅ | `df.write.saveAsTable()` 등 |
| **MLflow 모델** | ✅ | 모델이 어떤 Feature Table에서 학습되었는지 |
| **AI/BI Dashboard** | ✅ | 대시보드가 어떤 테이블을 조회하는지 |
| **외부 도구 (JDBC)** | ⚠️ 부분적 | SQL Warehouse를 통한 외부 BI 도구 접근 |

---

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

Unity Catalog의 리니지는 **쿼리 실행 시 자동으로** 수집됩니다. 별도의 설정이나 에이전트 설치가 필요하지 않습니다.

| 단계 | 설명 |
|------|------|
| 1. **쿼리 파싱** | SQL/Spark 쿼리를 파싱하여 AST(Abstract Syntax Tree)를 생성합니다 |
| 2. **논리 계획 분석** | Catalyst Optimizer의 논리 계획에서 소스/타겟 테이블과 컬럼을 추출합니다 |
| 3. **변환 추적** | 각 컬럼이 어떤 소스 컬럼에서 파생되었는지 (SELECT, JOIN, CASE WHEN 등) 추적합니다 |
| 4. **메타데이터 저장** | 추출된 리니지 정보를 UC 메타스토어의 리니지 테이블에 저장합니다 |
| 5. **주기적 집계** | 시스템 테이블(system.lineage.*)에 집계하여 조회 가능하게 합니다 |

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

컬럼 리니지는 테이블 리니지보다 더 세밀합니다. SQL의 **표현식을 분석**하여 컬럼 간 관계를 추적합니다.

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

## 영향도 분석 (Impact Analysis) 실전 패턴

### 패턴 1: 스키마 변경 전 전방위 영향도 확인

```sql
-- 1. 테이블 레벨 영향도
SELECT
    target_table_full_name,
    target_type,
    COUNT(*) AS reference_count
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
GROUP BY target_table_full_name, target_type
ORDER BY reference_count DESC;

-- 2. 컬럼 레벨 영향도 (특정 컬럼 변경 시)
SELECT
    target_table_full_name,
    target_column_name,
    source_column_name
FROM system.lineage.column_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
  AND source_column_name = 'amount'
ORDER BY target_table_full_name;

-- 3. 대시보드 영향도
SELECT DISTINCT
    target_table_full_name AS dashboard_or_query,
    target_type
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
  AND target_type IN ('DASHBOARD', 'QUERY');
```

### 패턴 2: 모델 영향도 — Feature Table 변경 시

```sql
-- Feature Table이 변경되면 영향받는 ML 모델 확인
SELECT
    target_table_full_name AS affected_model,
    target_type
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'production.ml.customer_features'
  AND target_type = 'MODEL';
-- → 이 Feature Table로 학습된 모든 모델이 재학습 대상입니다
```

### 패턴 3: SDP 파이프라인 영향도

```sql
-- SDP에서 사용하는 소스 테이블이 변경되면 영향받는 Streaming Table
SELECT
    target_table_full_name AS streaming_table,
    target_type
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'production.ecommerce.raw_events'
  AND target_type = 'STREAMING_TABLE';
```

---

## 리니지 기반 데이터 품질 자동화

리니지 정보를 활용하면 데이터 품질 이슈를 **자동으로 추적하고 전파**할 수 있습니다.

### 품질 이슈 전파 알림

```python
from databricks.sdk import WorkspaceClient

def check_upstream_quality(table_name):
    """
    특정 테이블의 upstream 품질을 확인하고,
    문제가 있으면 downstream 소유자에게 알림을 보냅니다.
    """
    w = WorkspaceClient()

    # 1. upstream 테이블 확인
    upstream_tables = spark.sql(f"""
        SELECT DISTINCT source_table_full_name
        FROM system.lineage.table_lineage
        WHERE target_table_full_name = '{table_name}'
    """).collect()

    quality_issues = []
    for row in upstream_tables:
        upstream = row.source_table_full_name

        # 2. 각 upstream 테이블의 품질 메트릭 확인
        try:
            quality = spark.sql(f"""
                SELECT *
                FROM {upstream}_profile_metrics
                WHERE log_type = 'DATA_QUALITY'
                  AND quality_check_passed = false
                  AND window_end > DATEADD(hour, -24, CURRENT_TIMESTAMP())
            """).count()

            if quality > 0:
                quality_issues.append(upstream)
        except:
            pass

    if quality_issues:
        # 3. downstream 영향받는 테이블/대시보드 확인
        downstream = spark.sql(f"""
            SELECT DISTINCT target_table_full_name, target_type
            FROM system.lineage.table_lineage
            WHERE source_table_full_name = '{table_name}'
        """).collect()

        print(f"⚠️ 품질 이슈 감지: {quality_issues}")
        print(f"영향받는 downstream: {[r.target_table_full_name for r in downstream]}")
        # Slack/이메일 알림 전송
```

### 리니지 기반 SLA 모니터링

```sql
-- 핵심 Gold 테이블의 upstream이 제시간에 업데이트되었는지 확인
WITH gold_tables AS (
    SELECT 'production.ecommerce.gold_daily_revenue' AS table_name, 6 AS sla_hour
    UNION ALL
    SELECT 'production.ecommerce.gold_customer_360', 8
),
upstream_freshness AS (
    SELECT
        g.table_name AS gold_table,
        g.sla_hour,
        l.source_table_full_name,
        MAX(h.timestamp) AS last_update
    FROM gold_tables g
    JOIN system.lineage.table_lineage l
        ON g.table_name = l.target_table_full_name
    JOIN (
        -- 각 소스 테이블의 마지막 업데이트 시간 (DESCRIBE HISTORY 대용)
        SELECT table_full_name, MAX(event_time) AS timestamp
        FROM system.lineage.table_lineage
        GROUP BY table_full_name
    ) h ON l.source_table_full_name = h.table_full_name
    GROUP BY g.table_name, g.sla_hour, l.source_table_full_name
)
SELECT
    gold_table,
    source_table_full_name,
    last_update,
    CASE
        WHEN last_update < DATEADD(hour, -sla_hour, CURRENT_TIMESTAMP())
        THEN '🔴 SLA 위반'
        ELSE '🟢 정상'
    END AS sla_status
FROM upstream_freshness;
```

---

## 규제 감사에서의 리니지 활용

### GDPR / 개인정보보호법 준수

| 규제 요건 | 리니지 활용 방법 |
|-----------|----------------|
| **정보 접근권** | 특정 고객 데이터가 어떤 테이블에 존재하는지 컬럼 리니지로 추적 |
| **삭제권 (Right to be Forgotten)** | PII 컬럼의 downstream을 추적하여 모든 복사본을 삭제 |
| **처리 목적 제한** | 데이터가 원래 수집 목적 외에 사용되지 않는지 리니지로 검증 |
| **데이터 이전** | 데이터 흐름을 문서화하여 국가 간 데이터 이전 규정 준수 |

### PII 전파 추적 대시보드 쿼리

```sql
-- PII 태그가 붙은 컬럼의 전파 경로 전체 추적
WITH pii_columns AS (
    SELECT
        catalog_name || '.' || schema_name || '.' || table_name AS table_full_name,
        column_name
    FROM system.information_schema.column_tags
    WHERE tag_name = 'pii' AND tag_value = 'true'
)
SELECT
    p.table_full_name AS pii_source,
    p.column_name AS pii_column,
    cl.target_table_full_name AS propagated_to,
    cl.target_column_name AS target_column
FROM pii_columns p
JOIN system.lineage.column_lineage cl
    ON p.table_full_name = cl.source_table_full_name
    AND p.column_name = cl.source_column_name
ORDER BY pii_source, propagated_to;
```

### 감사 보고서 자동 생성

```sql
-- 분기별 데이터 리니지 감사 보고서
SELECT
    source_table_full_name AS data_source,
    target_table_full_name AS data_destination,
    target_type AS destination_type,
    created_by AS responsible_user,
    MIN(event_time) AS first_seen,
    MAX(event_time) AS last_seen,
    COUNT(*) AS operation_count
FROM system.lineage.table_lineage
WHERE event_time BETWEEN '2025-01-01' AND '2025-03-31'
GROUP BY source_table_full_name, target_table_full_name, target_type, created_by
ORDER BY operation_count DESC;
```

---

## Edge Case와 주의사항

| 주의사항 | 설명 |
|---------|------|
| **리니지 지연** | 리니지 정보는 실시간이 아닙니다. 쿼리 실행 후 system.lineage 테이블에 반영되기까지 수 분 ~ 수 시간이 소요될 수 있습니다 |
| **UDF 리니지 제한** | Python/Scala UDF 내부의 컬럼 변환은 추적되지 않습니다. UDF의 입출력만 기록됩니다 |
| **Dynamic SQL** | 문자열로 조합한 SQL (`spark.sql(f"SELECT {cols} FROM ...")`)은 리니지 파싱이 불완전할 수 있습니다 |
| **외부 도구 리니지** | JDBC로 접근하는 외부 BI 도구의 쿼리는 SQL Warehouse를 경유해야 리니지가 기록됩니다. 직접 스토리지 접근은 추적 불가합니다 |
| **리니지 보존 기간** | system.lineage 테이블의 데이터는 일정 기간(기본 365일) 동안 보존됩니다. 장기 보존이 필요하면 별도 백업이 필요합니다 |
| **뷰의 리니지** | 뷰를 통한 접근은 뷰의 정의에서 소스 테이블까지 투명하게 추적됩니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **리니지** | 데이터의 출처(upstream)와 행선지(downstream)를 추적합니다 |
| **테이블 리니지** | 테이블 간의 데이터 흐름을 추적합니다 |
| **컬럼 리니지** | 컬럼 간의 데이터 흐름을 추적합니다 |
| **자동 추적** | SQL, Spark, SDP 등의 작업에서 자동으로 추적됩니다 |
| **영향도 분석** | 스키마 변경, 데이터 품질 이슈 시 downstream 영향을 파악합니다 |
| **규제 준수** | PII 전파 추적, 감사 보고서 등 규제 요건을 지원합니다 |
| **시스템 테이블** | `system.lineage.table_lineage`, `column_lineage`로 SQL 조회가 가능합니다 |

---

## 참고 링크

- [Databricks: Data lineage](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-lineage.html)
- [Azure Databricks: Data lineage](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/data-lineage)
