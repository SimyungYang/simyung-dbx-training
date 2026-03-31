# 시스템 테이블

## 시스템 테이블이란?

> 💡 **시스템 테이블(System Tables)** 은 Databricks 플랫폼의 운영 데이터(감사 로그, 빌링, 리니지, 사용량 등)를 **SQL로 직접 조회**할 수 있는 내장 테이블입니다. `system` 카탈로그 아래에 위치하며, Unity Catalog가 활성화된 모든 워크스페이스에서 사용할 수 있습니다.

시스템 테이블을 활용하면 별도의 모니터링 도구 없이도, SQL 쿼리와 대시보드만으로 플랫폼 전체의 운영 상황을 파악할 수 있습니다.

---

## 주요 시스템 테이블

### 감사 (Audit)

| 테이블 | 설명 |
|--------|------|
| `system.access.audit` | **모든 사용자 활동 로그**. 누가, 언제, 어떤 작업을 했는지 기록합니다. 테이블 조회, 클러스터 생성, 권한 변경 등 모든 API 호출이 포함됩니다 |

```sql
-- 최근 7일간 누가 어떤 테이블에 접근했는지 확인
SELECT
    event_date,
    user_identity.email AS user_email,
    action_name,
    request_params.full_name_arg AS table_name,
    source_ip_address
FROM system.access.audit
WHERE action_name IN ('getTable', 'commandSubmit')
    AND event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY event_date DESC
LIMIT 100;

-- 비정상 접근 패턴 감지: 야간 시간대(22시~06시) 접근
SELECT
    user_identity.email,
    COUNT(*) AS late_night_actions,
    COLLECT_SET(action_name) AS action_types
FROM system.access.audit
WHERE HOUR(event_time) NOT BETWEEN 6 AND 22
    AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY user_identity.email
HAVING COUNT(*) > 10
ORDER BY late_night_actions DESC;
```

### 빌링 (Billing)

| 테이블 | 설명 |
|--------|------|
| `system.billing.usage` | **DBU 사용량** 상세 기록. 클러스터, Warehouse, Jobs, Serverless 등 워크로드별 소비량입니다 |
| `system.billing.list_prices` | **DBU 단가** 정보입니다 |

```sql
-- 지난 30일간 워크로드 유형별 DBU 사용량
SELECT
    usage_metadata.warehouse_id,
    sku_name,
    billing_origin_product,
    SUM(usage_quantity) AS total_dbus,
    ROUND(SUM(usage_quantity) * 0.22, 2) AS estimated_cost_usd  -- 대략적 비용 추정
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2, 3
ORDER BY total_dbus DESC;

-- 사용자별 비용 TOP 10
SELECT
    identity_metadata.run_as AS user_email,
    billing_origin_product AS product,
    SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY total_dbus DESC
LIMIT 10;

-- 일별 비용 추이 (급증 감지)
SELECT
    usage_date,
    billing_origin_product,
    SUM(usage_quantity) AS daily_dbus
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE() - INTERVAL 90 DAYS
GROUP BY 1, 2
ORDER BY usage_date;
```

### 컴퓨트 (Compute)

| 테이블 | 설명 |
|--------|------|
| `system.compute.clusters` | 클러스터 설정, 상태, 이벤트 기록입니다 |
| `system.compute.warehouse_events` | SQL Warehouse의 이벤트 로그입니다 |

```sql
-- 가장 오래 실행 중인 클러스터 (비용 누수 감지)
SELECT
    cluster_id,
    cluster_name,
    driver_node_type,
    autotermination_minutes,
    state,
    TIMESTAMPDIFF(HOUR, state_start_time, CURRENT_TIMESTAMP()) AS running_hours
FROM system.compute.clusters
WHERE state = 'RUNNING'
ORDER BY running_hours DESC;
```

### 워크플로우 (Lakeflow)

| 테이블 | 설명 |
|--------|------|
| `system.lakeflow.jobs` | Job 실행 이력, 결과, 실행 시간입니다 |
| `system.lakeflow.job_tasks` | 개별 Task 수준의 실행 기록입니다 |

```sql
-- 최근 실패한 Job TOP 10
SELECT
    job_id,
    run_id,
    result_state,
    TIMESTAMPDIFF(MINUTE, start_time, end_time) AS duration_minutes,
    error_message,
    start_time
FROM system.lakeflow.jobs
WHERE result_state = 'FAILED'
    AND start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY start_time DESC
LIMIT 10;

-- Job별 평균 실행 시간 추이 (성능 저하 감지)
SELECT
    job_id,
    DATE(start_time) AS run_date,
    AVG(TIMESTAMPDIFF(MINUTE, start_time, end_time)) AS avg_duration_min,
    COUNT(*) AS run_count,
    SUM(CASE WHEN result_state = 'FAILED' THEN 1 ELSE 0 END) AS fail_count
FROM system.lakeflow.jobs
WHERE start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY job_id, DATE(start_time)
ORDER BY job_id, run_date;
```

### 리니지 (Lineage)

| 테이블 | 설명 |
|--------|------|
| `system.lineage.table_lineage` | 테이블 간 데이터 흐름(upstream/downstream)입니다 |
| `system.lineage.column_lineage` | 컬럼 간 데이터 흐름입니다 |

```sql
-- 특정 테이블에 의존하는 모든 downstream 테이블
SELECT
    target_table_full_name,
    target_type,
    source_table_full_name
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
ORDER BY target_table_full_name;

-- 특정 컬럼이 어디에서 왔는지 (upstream 추적)
SELECT
    source_table_full_name,
    source_column_name,
    target_table_full_name,
    target_column_name
FROM system.lineage.column_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_revenue'
    AND target_column_name = 'total_amount';
```

### Predictive Optimization

| 테이블 | 설명 |
|--------|------|
| `system.storage.predictive_optimization_operations_history` | 자동 OPTIMIZE/VACUUM 실행 이력입니다 |

```sql
-- Predictive Optimization이 자동으로 수행한 작업 확인
SELECT
    table_name,
    operation_type,
    operation_status,
    usage_quantity AS dbus_used,
    start_time
FROM system.storage.predictive_optimization_operations_history
WHERE start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY start_time DESC;
```

---

## 시스템 테이블 전체 목록

| 스키마 | 테이블 | 내용 |
|--------|--------|------|
| `system.access` | `audit` | 사용자 활동 감사 로그 |
| `system.billing` | `usage` | DBU 사용량 |
| `system.billing` | `list_prices` | DBU 단가 |
| `system.compute` | `clusters` | 클러스터 정보 |
| `system.compute` | `warehouse_events` | SQL Warehouse 이벤트 |
| `system.lakeflow` | `jobs` | Job 실행 이력 |
| `system.lakeflow` | `job_tasks` | Task 실행 이력 |
| `system.lakeflow` | `job_run_timeline` | Run 타임라인 |
| `system.lineage` | `table_lineage` | 테이블 리니지 |
| `system.lineage` | `column_lineage` | 컬럼 리니지 |
| `system.storage` | `predictive_optimization_operations_history` | 자동 최적화 이력 |
| `system.information_schema` | 다양한 뷰 | 테이블/컬럼/권한 메타데이터 |

---

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

## 운영 대시보드 구성 가이드

시스템 테이블을 활용한 AI/BI 대시보드를 구축하면, 한눈에 플랫폼 운영 상황을 파악할 수 있습니다.

### 권장 대시보드 패널 구성

| 대시보드 | 주요 패널 | 데이터 소스 |
|---------|----------|-----------|
| **비용 관리** | 월별 DBU 추이, 팀별 비용, 비용 급증 알림 | `system.billing.usage` |
| **보안 감사** | 로그인 실패, 권한 변경, 대량 다운로드 | `system.access.audit` |
| **Job 모니터링** | 실패율, 실행시간 추이, SLA 위반 | `system.lakeflow.jobs` |
| **클러스터 현황** | 유휴 클러스터, 활용률, 비용 | `system.compute.clusters` |
| **데이터 거버넌스** | 리니지 맵, 미사용 테이블, 접근 패턴 | `system.lineage.*`, `system.access.audit` |

### 알림 설정 예시

```sql
-- SQL Alert: 일일 비용이 $500을 초과하면 알림
SELECT
    usage_date,
    SUM(usage_quantity) AS total_dbus,
    ROUND(SUM(usage_quantity) * 0.22, 2) AS estimated_cost_usd
FROM system.billing.usage
WHERE usage_date = CURRENT_DATE() - INTERVAL 1 DAY
GROUP BY usage_date
HAVING estimated_cost_usd > 500;
```

> 💡 위 쿼리를 **SQL Alert**로 등록하고, Slack/이메일로 알림을 설정하면 비용 이상을 즉시 감지할 수 있습니다.

---

## 활성화 방법

시스템 테이블은 **계정 관리자(Account Admin)** 가 활성화해야 합니다.

1. **Account Console** → **Settings** → **Feature Enablement**
2. **System Tables** → **Enable**
3. 활성화 후 데이터가 채워지기까지 최대 24시간이 소요될 수 있습니다

> 💡 시스템 테이블의 데이터는 **읽기 전용**이며, Unity Catalog의 `system` 카탈로그에 위치합니다. 조회하려면 `USE CATALOG system` 권한이 필요합니다.

---

## 정리

| 핵심 카테고리 | 대표 테이블 | 활용 |
|-------------|-----------|------|
| **감사** | `system.access.audit` | 누가 무엇을 했는지 추적 |
| **빌링** | `system.billing.usage` | 비용 분석, 예산 관리 |
| **컴퓨트** | `system.compute.clusters` | 유휴 리소스 감지 |
| **워크플로우** | `system.lakeflow.jobs` | Job 성공/실패 모니터링 |
| **리니지** | `system.lineage.table_lineage` | 데이터 흐름 추적 |

---

## 참고 링크

- [Databricks: System tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/)
- [Databricks: Billing usage](https://docs.databricks.com/aws/en/administration-guide/system-tables/billing.html)
- [Databricks: Audit logs](https://docs.databricks.com/aws/en/administration-guide/system-tables/audit-logs.html)
- [Azure Databricks: System tables](https://learn.microsoft.com/en-us/azure/databricks/administration-guide/system-tables/)
