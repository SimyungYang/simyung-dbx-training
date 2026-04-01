# 개요와 주요 시스템 테이블

## 시스템 테이블이란?

> 💡 **시스템 테이블(System Tables)**은 Databricks 플랫폼의 운영 데이터(감사 로그, 빌링, 리니지, 사용량 등)를 **SQL로 직접 조회** 할 수 있는 내장 테이블입니다. `system` 카탈로그 아래에 위치하며, Unity Catalog가 활성화된 모든 워크스페이스에서 사용할 수 있습니다.

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
