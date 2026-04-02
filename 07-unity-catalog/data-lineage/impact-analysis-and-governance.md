# 영향도 분석과 거버넌스

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

리니지 정보를 활용하면 데이터 품질 이슈를 **자동으로 추적하고 전파** 할 수 있습니다.

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
