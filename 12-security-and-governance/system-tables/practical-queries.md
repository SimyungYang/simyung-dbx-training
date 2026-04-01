# 실무 활용 쿼리 모음

## 실무 활용 쿼리 모음

### 비용 관리 쿼리

```sql
-- 📊 월별 비용 추이 및 전월 대비 증감률
WITH monthly AS (
    SELECT
        DATE_TRUNC('MONTH', usage_date) AS month,
        billing_origin_product AS product,
        SUM(usage_quantity) AS total_dbus
    FROM system.billing.usage
    WHERE usage_date >= CURRENT_DATE() - INTERVAL 6 MONTHS
    GROUP BY 1, 2
)
SELECT
    month,
    product,
    total_dbus,
    LAG(total_dbus) OVER (PARTITION BY product ORDER BY month) AS prev_month_dbus,
    ROUND((total_dbus - LAG(total_dbus) OVER (PARTITION BY product ORDER BY month))
        / NULLIF(LAG(total_dbus) OVER (PARTITION BY product ORDER BY month), 0) * 100, 1) AS growth_pct
FROM monthly
ORDER BY month DESC, total_dbus DESC;

-- 💰 팀/프로젝트별 비용 분배 (클러스터 태그 기반)
SELECT
    custom_tags.Team AS team,
    custom_tags.Project AS project,
    billing_origin_product,
    SUM(usage_quantity) AS total_dbus,
    ROUND(SUM(usage_quantity) * 0.22, 2) AS estimated_cost_usd
FROM system.billing.usage
WHERE usage_date >= DATE_TRUNC('MONTH', CURRENT_DATE())
    AND custom_tags.Team IS NOT NULL
GROUP BY 1, 2, 3
ORDER BY total_dbus DESC;

-- 🔍 비용 급증 감지 (전일 대비 2배 이상 증가)
WITH daily AS (
    SELECT
        usage_date,
        SUM(usage_quantity) AS daily_dbus
    FROM system.billing.usage
    WHERE usage_date >= CURRENT_DATE() - INTERVAL 14 DAYS
    GROUP BY 1
)
SELECT
    usage_date,
    daily_dbus,
    LAG(daily_dbus) OVER (ORDER BY usage_date) AS prev_day_dbus,
    ROUND(daily_dbus / NULLIF(LAG(daily_dbus) OVER (ORDER BY usage_date), 0), 2) AS ratio
FROM daily
HAVING ratio > 2.0
ORDER BY usage_date DESC;

-- 🏷️ SQL Warehouse별 비용 및 효율성
SELECT
    usage_metadata.warehouse_id,
    sku_name,
    SUM(usage_quantity) AS total_dbus,
    COUNT(DISTINCT usage_date) AS active_days,
    ROUND(SUM(usage_quantity) / COUNT(DISTINCT usage_date), 2) AS avg_daily_dbus
FROM system.billing.usage
WHERE billing_origin_product = 'SQL'
    AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY total_dbus DESC;
```

### 보안 감사 쿼리

```sql
-- 🔐 로그인 실패 추적 (브루트포스 감지)
SELECT
    user_identity.email,
    source_ip_address,
    COUNT(*) AS failed_attempts,
    MIN(event_time) AS first_attempt,
    MAX(event_time) AS last_attempt
FROM system.access.audit
WHERE action_name = 'login'
    AND response.status_code != 200
    AND event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY 1, 2
HAVING COUNT(*) > 5
ORDER BY failed_attempts DESC;

-- 🛡️ 권한 변경 추적 (GRANT/REVOKE 이력)
SELECT
    event_time,
    user_identity.email AS changed_by,
    action_name,
    request_params.securable_type AS object_type,
    request_params.securable_full_name AS object_name,
    request_params.changes AS permission_changes
FROM system.access.audit
WHERE action_name IN ('updatePermissions', 'metastoreAssignment')
    AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
ORDER BY event_time DESC;

-- 📥 대량 데이터 다운로드 감지
SELECT
    user_identity.email,
    DATE(event_time) AS event_date,
    COUNT(*) AS download_count
FROM system.access.audit
WHERE action_name IN ('downloadQueryResult', 'downloadNotebookResult')
    AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2
HAVING COUNT(*) > 20
ORDER BY download_count DESC;

-- 🔑 PAT 토큰 생성/사용 추적
SELECT
    event_time,
    user_identity.email,
    action_name,
    source_ip_address,
    request_params
FROM system.access.audit
WHERE action_name IN ('createToken', 'deleteToken', 'listTokens')
    AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
ORDER BY event_time DESC;
```

### Job/파이프라인 모니터링 쿼리

```sql
-- ⚠️ 반복 실패하는 Job 식별 (SLA 위험)
SELECT
    job_id,
    COUNT(*) AS total_runs,
    SUM(CASE WHEN result_state = 'FAILED' THEN 1 ELSE 0 END) AS failed_runs,
    ROUND(SUM(CASE WHEN result_state = 'FAILED' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) AS failure_rate_pct,
    AVG(TIMESTAMPDIFF(MINUTE, start_time, end_time)) AS avg_duration_min
FROM system.lakeflow.jobs
WHERE start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY job_id
HAVING failure_rate_pct > 20
ORDER BY failure_rate_pct DESC;

-- ⏱️ Job 실행 시간 추이 (성능 퇴화 감지)
SELECT
    job_id,
    DATE(start_time) AS run_date,
    AVG(TIMESTAMPDIFF(MINUTE, start_time, end_time)) AS avg_min,
    MAX(TIMESTAMPDIFF(MINUTE, start_time, end_time)) AS max_min,
    MIN(TIMESTAMPDIFF(MINUTE, start_time, end_time)) AS min_min
FROM system.lakeflow.jobs
WHERE start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
    AND result_state = 'SUCCESS'
GROUP BY job_id, DATE(start_time)
ORDER BY job_id, run_date;

-- 🕐 현재 실행 중인 Job (장시간 실행 감지)
SELECT
    job_id,
    run_id,
    TIMESTAMPDIFF(MINUTE, start_time, CURRENT_TIMESTAMP()) AS running_minutes,
    start_time
FROM system.lakeflow.jobs
WHERE result_state IS NULL  -- 아직 실행 중
    AND TIMESTAMPDIFF(MINUTE, start_time, CURRENT_TIMESTAMP()) > 60
ORDER BY running_minutes DESC;
```

### 클러스터 최적화 쿼리

```sql
-- 💤 유휴 클러스터 감지 (비용 낭비)
SELECT
    cluster_id,
    cluster_name,
    driver_node_type,
    autotermination_minutes,
    TIMESTAMPDIFF(HOUR, state_start_time, CURRENT_TIMESTAMP()) AS idle_hours
FROM system.compute.clusters
WHERE state = 'RUNNING'
    AND TIMESTAMPDIFF(HOUR, state_start_time, CURRENT_TIMESTAMP()) > 4
ORDER BY idle_hours DESC;

-- 📊 클러스터 사이징 분석 (과다/과소 프로비저닝)
SELECT
    cluster_id,
    cluster_name,
    driver_node_type,
    num_workers,
    SUM(usage_quantity) AS total_dbus,
    COUNT(DISTINCT usage_date) AS active_days,
    ROUND(SUM(usage_quantity) / COUNT(DISTINCT usage_date), 2) AS avg_daily_dbus
FROM system.billing.usage u
JOIN system.compute.clusters c ON u.usage_metadata.cluster_id = c.cluster_id
WHERE usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2, 3, 4
ORDER BY total_dbus DESC
LIMIT 20;
```

### 데이터 거버넌스 쿼리

```sql
-- 🔗 테이블 영향도 분석 (이 테이블을 변경하면 어디에 영향?)
WITH RECURSIVE downstream AS (
    SELECT source_table_full_name, target_table_full_name, 1 AS depth
    FROM system.lineage.table_lineage
    WHERE source_table_full_name = 'production.bronze.raw_orders'

    UNION ALL

    SELECT d.target_table_full_name, l.target_table_full_name, d.depth + 1
    FROM downstream d
    JOIN system.lineage.table_lineage l ON d.target_table_full_name = l.source_table_full_name
    WHERE d.depth < 5
)
SELECT DISTINCT target_table_full_name, MIN(depth) AS distance
FROM downstream
GROUP BY target_table_full_name
ORDER BY distance;

-- 📋 사용되지 않는 테이블 (30일 이상 미접근)
SELECT
    t.table_catalog,
    t.table_schema,
    t.table_name,
    t.created AS table_created,
    MAX(a.event_date) AS last_accessed
FROM system.information_schema.tables t
LEFT JOIN system.access.audit a
    ON a.request_params.full_name_arg = CONCAT(t.table_catalog, '.', t.table_schema, '.', t.table_name)
    AND a.action_name IN ('getTable', 'commandSubmit')
WHERE t.table_catalog != 'system'
GROUP BY 1, 2, 3, 4
HAVING last_accessed IS NULL OR last_accessed < CURRENT_DATE() - INTERVAL 30 DAYS
ORDER BY table_created;
```

---
