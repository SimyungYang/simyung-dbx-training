# 운영 대시보드와 활성화 방법

## 1. 왜 시스템 테이블 대시보드가 필요한가

Databricks 플랫폼이 커질수록 비용, 보안, 성능을 개별 도구로 따로 추적하는 방식은 한계에 부딪힙니다. **시스템 테이블 (System Tables)** 은 Unity Catalog 내에 자동으로 적재되는 운영 메타데이터로, 단일 SQL 인터페이스로 세 가지 핵심 관심사를 동시에 다룰 수 있습니다.

| 관심사 | 문제 | 시스템 테이블 해법 |
|--------|------|-----------------|
| **비용 가시성 (Cost Visibility)** | 어떤 팀/프로젝트가 얼마나 쓰는지 모름 | `billing.usage` + `billing.list_prices` |
| **보안 감사 (Security Audit)** | 누가 민감 데이터에 접근했는지 추적 불가 | `access.audit` |
| **운영 모니터링 (Ops Monitoring)** | 실패 Job, 유휴 클러스터 자동 탐지 불가 | `compute.clusters`, `lakeflow.job_run_timeline` |

> 핵심 장점: 외부 로그 수집 파이프라인 없이 **Databricks SQL + AI/BI 대시보드** 만으로 운영 단일 창구(Single Pane of Glass)를 구성할 수 있습니다.

---

## 2. 시스템 테이블 활성화

### 필요 권한

- **계정 관리자 (Account Admin)** 권한 필요
- Metastore에 Unity Catalog 활성화 필수
- 조회 권한 부여 시 `system` 카탈로그에 대한 `USE CATALOG` 권한 및 각 스키마에 `SELECT` 권한 필요

### UI로 활성화

1. **Account Console** (accounts.azuredatabricks.net 또는 accounts.cloud.databricks.com) 접속
2. **Settings** → **Feature Enablement**
3. **System Tables** 항목에서 **Enable** 클릭
4. 활성화 후 데이터 수집 시작까지 최대 **24시간** 소요

### REST API로 활성화

특정 스키마만 선택적으로 활성화하거나 자동화 파이프라인에서 관리할 때 REST API를 사용합니다.

```bash
# 활성화 가능한 스키마 목록 조회
curl -X GET \
  "https://<account-console-host>/api/2.0/accounts/<account-id>/metastores/<metastore-id>/systemschemas" \
  -H "Authorization: Bearer <token>"

# 특정 스키마 활성화 (예: billing)
curl -X PUT \
  "https://<account-console-host>/api/2.0/accounts/<account-id>/metastores/<metastore-id>/systemschemas/billing" \
  -H "Authorization: Bearer <token>"
```

### 조회 권한 부여

```sql
-- 특정 사용자에게 billing 스키마 조회 권한 부여
GRANT USE SCHEMA ON SCHEMA system.billing TO `analyst@example.com`;
GRANT SELECT ON SCHEMA system.billing TO `analyst@example.com`;

-- 그룹 단위 권한 부여 (권장)
GRANT USE SCHEMA ON SCHEMA system.access TO `security-team`;
GRANT SELECT ON SCHEMA system.access TO `security-team`;
```

> 시스템 테이블 데이터는 **읽기 전용 (Read-Only)** 이며, `system` 카탈로그 내 각 스키마에 위치합니다. 활성화 이후 생성된 이벤트부터 데이터가 적재되며 소급 적용되지 않습니다.

---

## 3. 비용 모니터링 대시보드

### billing.usage + list_prices 조인

`billing.usage` 단독으로는 DBU 수량만 알 수 있습니다. 실제 달러 비용을 계산하려면 `billing.list_prices` 와 조인해야 합니다.

```sql
-- 일별 비용 추이 (최근 30일)
SELECT
    u.usage_date,
    SUM(u.usage_quantity * p.pricing.default) AS cost_usd
FROM system.billing.usage u
JOIN system.billing.list_prices p
    ON u.sku_name = p.sku_name
    AND u.usage_date BETWEEN p.price_start_time::DATE
                         AND COALESCE(p.price_end_time::DATE, CURRENT_DATE())
WHERE u.usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY u.usage_date
ORDER BY u.usage_date;
```

### 팀별/프로젝트별 비용 분석

태그(Tag)를 클러스터와 Job에 부착하면 `custom_tags` 컬럼으로 그룹화할 수 있습니다.

```sql
-- 팀별 누적 비용 (이번 달)
SELECT
    u.custom_tags['team']      AS team,
    u.custom_tags['project']   AS project,
    u.sku_name,
    ROUND(SUM(u.usage_quantity * p.pricing.default), 2) AS cost_usd
FROM system.billing.usage u
JOIN system.billing.list_prices p
    ON u.sku_name = p.sku_name
    AND u.usage_date BETWEEN p.price_start_time::DATE
                         AND COALESCE(p.price_end_time::DATE, CURRENT_DATE())
WHERE DATE_TRUNC('month', u.usage_date) = DATE_TRUNC('month', CURRENT_DATE())
GROUP BY ALL
ORDER BY cost_usd DESC;
```

### 주별 비용 추이 (이상 탐지용)

```sql
-- 주별 비용 비교: 전주 대비 증감률
WITH weekly AS (
    SELECT
        DATE_TRUNC('week', u.usage_date) AS week_start,
        SUM(u.usage_quantity * p.pricing.default) AS cost_usd
    FROM system.billing.usage u
    JOIN system.billing.list_prices p
        ON u.sku_name = p.sku_name
        AND u.usage_date BETWEEN p.price_start_time::DATE
                             AND COALESCE(p.price_end_time::DATE, CURRENT_DATE())
    WHERE u.usage_date >= CURRENT_DATE() - INTERVAL 90 DAYS
    GROUP BY 1
)
SELECT
    week_start,
    cost_usd,
    LAG(cost_usd) OVER (ORDER BY week_start) AS prev_week_cost,
    ROUND(
        (cost_usd - LAG(cost_usd) OVER (ORDER BY week_start))
        / NULLIF(LAG(cost_usd) OVER (ORDER BY week_start), 0) * 100,
        1
    ) AS wow_change_pct
FROM weekly
ORDER BY week_start;
```

---

## 4. 클러스터 운영 대시보드

### 비효율 클러스터 탐지

**Auto-termination (자동 종료)** 이 설정되지 않은 클러스터는 유휴 상태에서도 비용이 계속 발생합니다.

```sql
-- Auto-termination 미설정 또는 종료 시간이 과도하게 긴 클러스터
SELECT
    cluster_id,
    cluster_name,
    cluster_source,
    autotermination_minutes,
    creator_id,
    change_time
FROM system.compute.clusters
WHERE cluster_source = 'UI'          -- 사용자가 수동 생성한 클러스터
  AND (
        autotermination_minutes IS NULL
     OR autotermination_minutes > 120  -- 2시간 초과
  )
  AND change_time >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY autotermination_minutes DESC NULLS FIRST;
```

### 장기 실행 클러스터 목록

```sql
-- 현재 Running 상태이면서 생성 후 48시간 이상 경과한 클러스터
SELECT
    cluster_id,
    cluster_name,
    state,
    creator_id,
    change_time,
    TIMESTAMPDIFF(HOUR, change_time, NOW()) AS running_hours
FROM system.compute.clusters
WHERE state = 'RUNNING'
  AND change_time <= NOW() - INTERVAL 48 HOURS
ORDER BY running_hours DESC;
```

---

## 5. 보안 감사 대시보드

### 로그인 실패 및 비정상 접근 패턴

```sql
-- 최근 24시간 내 로그인 실패 상위 계정
SELECT
    user_identity.email                AS user_email,
    COUNT(*)                           AS failed_attempts,
    MIN(event_time)                    AS first_attempt,
    MAX(event_time)                    AS last_attempt
FROM system.access.audit
WHERE action_name = 'login'
  AND response.status_code != 200
  AND event_time >= NOW() - INTERVAL 24 HOURS
GROUP BY user_identity.email
HAVING failed_attempts >= 5
ORDER BY failed_attempts DESC;
```

### 민감 데이터 접근 로그

```sql
-- 특정 테이블에 대한 접근 로그 (최근 7일)
SELECT
    event_time,
    user_identity.email     AS user_email,
    action_name,
    request_params.table_full_name AS target_table,
    source_ip_address
FROM system.access.audit
WHERE request_params.table_full_name LIKE '%pii%'  -- PII 포함 테이블 필터
  AND event_time >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY event_time DESC
LIMIT 500;
```

### 권한 변경 감사

```sql
-- GRANT / REVOKE 이벤트 추적
SELECT
    event_time,
    user_identity.email    AS performed_by,
    action_name,
    request_params
FROM system.access.audit
WHERE action_name IN ('grantPermission', 'revokePermission',
                      'createGroup', 'addPrincipalToGroup')
  AND event_time >= CURRENT_DATE() - INTERVAL 30 DAYS
ORDER BY event_time DESC;
```

---

## 6. Job 성능 대시보드

### 실패 Job 탐지

```sql
-- 최근 7일 실패 Job 목록 (실패율 포함)
SELECT
    job_id,
    job_name,
    COUNT(*)                                                  AS total_runs,
    SUM(CASE WHEN result_state = 'FAILED' THEN 1 ELSE 0 END) AS failed_runs,
    ROUND(
        SUM(CASE WHEN result_state = 'FAILED' THEN 1 ELSE 0 END)
        / COUNT(*) * 100,
        1
    ) AS failure_rate_pct
FROM system.lakeflow.job_run_timeline
WHERE period_start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY job_id, job_name
HAVING failure_rate_pct > 10
ORDER BY failure_rate_pct DESC;
```

### 장기 실행 Job 탐지 (SLA 위반 탐지)

```sql
-- 평균 실행 시간 대비 2배 이상 소요된 이상 실행 감지
WITH job_stats AS (
    SELECT
        job_id,
        job_name,
        AVG(TIMESTAMPDIFF(MINUTE, period_start_time, period_end_time)) AS avg_duration_min,
        STDDEV(TIMESTAMPDIFF(MINUTE, period_start_time, period_end_time)) AS std_duration_min
    FROM system.lakeflow.job_run_timeline
    WHERE period_start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
      AND result_state = 'SUCCEEDED'
    GROUP BY job_id, job_name
)
SELECT
    t.job_id,
    t.job_name,
    TIMESTAMPDIFF(MINUTE, t.period_start_time, t.period_end_time) AS run_duration_min,
    s.avg_duration_min,
    ROUND(
        TIMESTAMPDIFF(MINUTE, t.period_start_time, t.period_end_time)
        / NULLIF(s.avg_duration_min, 0),
        2
    ) AS duration_ratio
FROM system.lakeflow.job_run_timeline t
JOIN job_stats s ON t.job_id = s.job_id
WHERE t.period_start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
  AND TIMESTAMPDIFF(MINUTE, t.period_start_time, t.period_end_time)
      > s.avg_duration_min + 2 * s.std_duration_min
ORDER BY duration_ratio DESC;
```

---

## 7. Databricks SQL Alert 설정

**SQL Alert** 는 Databricks SQL 쿼리 결과가 특정 조건을 충족할 때 Slack, 이메일, 웹훅으로 알림을 발송하는 기능입니다.

### 비용 임계값 초과 알림

```sql
-- SQL Alert 등록용 쿼리: 일일 비용이 $1,000 초과 시 알림
SELECT
    usage_date,
    ROUND(SUM(u.usage_quantity * p.pricing.default), 2) AS daily_cost_usd
FROM system.billing.usage u
JOIN system.billing.list_prices p
    ON u.sku_name = p.sku_name
    AND u.usage_date BETWEEN p.price_start_time::DATE
                         AND COALESCE(p.price_end_time::DATE, CURRENT_DATE())
WHERE u.usage_date = CURRENT_DATE() - INTERVAL 1 DAY
GROUP BY usage_date
HAVING daily_cost_usd > 1000;
```

### Alert 설정 절차

1. Databricks SQL → **Alerts** → **New Alert**
2. 위 쿼리를 연결하고 **Trigger condition**: `Value` > `0` (결과가 1행 이상 반환되면 트리거)
3. **Refresh schedule**: 매일 오전 9시 (UTC+9 기준 `0 0 * * *`)
4. **Notification destination** 에서 Slack 채널 또는 이메일 그룹 지정

> Alert는 **SQL Warehouse** 에서 실행됩니다. 비용 절감을 위해 Serverless SQL Warehouse 또는 Auto Stop이 짧은 Warehouse를 지정하세요.

### 보안 이상 탐지 Alert

```sql
-- 10분 내 동일 IP에서 10회 이상 로그인 실패 시 알림
SELECT
    source_ip_address,
    COUNT(*) AS attempt_count,
    MAX(event_time) AS last_attempt
FROM system.access.audit
WHERE action_name = 'login'
  AND response.status_code != 200
  AND event_time >= NOW() - INTERVAL 10 MINUTES
GROUP BY source_ip_address
HAVING attempt_count >= 10;
```

---

## 8. 베스트 프랙티스

### 대시보드 새로고침 주기

| 대시보드 유형 | 권장 새로고침 주기 | 이유 |
|-------------|----------------|------|
| **비용 모니터링** | 1일 1회 (오전) | `billing.usage` 데이터 반영 지연 최대 3시간 |
| **보안 감사** | 1시간마다 | 이상 접근 패턴 조기 탐지 |
| **Job 성능** | 30분마다 | 실패 Job 신속 대응 |
| **클러스터 현황** | 1시간마다 | 유휴 클러스터 비용 최소화 |

### 데이터 보존 (Data Retention) 관리

- 시스템 테이블 기본 보존 기간은 스키마마다 다릅니다 (`audit`: 365일, `billing.usage`: 365일, `compute`: 365일)
- 장기 보존이 필요한 경우 **DEEP CLONE** 또는 **CTAS (CREATE TABLE AS SELECT)** 로 별도 테이블에 스냅샷을 보관하세요

```sql
-- 월별 billing 스냅샷 보관 예시
CREATE TABLE IF NOT EXISTS ops.billing_snapshots.usage_2025_03
AS
SELECT *
FROM system.billing.usage
WHERE DATE_TRUNC('month', usage_date) = '2025-03-01';
```

### 접근 권한 관리

- **최소 권한 원칙 (Least Privilege)**: 팀별로 필요한 스키마만 선택 부여
- **서비스 프린시펄 (Service Principal)** 을 통한 대시보드 실행 권장 (개인 계정 의존성 제거)
- 접근 권한 변경은 `system.access.audit` 로 자동 감사됨

### 권장 대시보드 패널 구성 요약

| 대시보드 | 주요 패널 | 데이터 소스 |
|---------|----------|-----------|
| **비용 관리** | 월별 DBU 추이, 팀별 비용, 비용 급증 알림 | `system.billing.usage` + `system.billing.list_prices` |
| **보안 감사** | 로그인 실패, 권한 변경, 대량 다운로드 | `system.access.audit` |
| **Job 모니터링** | 실패율, 실행시간 추이, SLA 위반 | `system.lakeflow.job_run_timeline` |
| **클러스터 현황** | 유휴 클러스터, Auto-termination 미설정 | `system.compute.clusters` |
| **데이터 거버넌스** | 리니지 맵, 미사용 테이블, 접근 패턴 | `system.lineage.*`, `system.access.audit` |

---

## 정리

| 핵심 카테고리 | 대표 테이블 | 활용 |
|-------------|-----------|------|
| **감사 (Audit)** | `system.access.audit` | 누가 무엇을 했는지 추적 |
| **빌링 (Billing)** | `system.billing.usage` | 비용 분석, 예산 관리 |
| **컴퓨트 (Compute)** | `system.compute.clusters` | 유휴 리소스 감지 |
| **워크플로우 (Workflow)** | `system.lakeflow.job_run_timeline` | Job 성공/실패 모니터링 |
| **리니지 (Lineage)** | `system.lineage.table_lineage` | 데이터 흐름 추적 |

---

## 참고 링크

- [Databricks: System tables overview](https://docs.databricks.com/aws/en/administration-guide/system-tables/)
- [Databricks: Enable system tables (REST API)](https://docs.databricks.com/api/account/systemschemas/enable)
- [Databricks: Billing usage schema](https://docs.databricks.com/aws/en/administration-guide/system-tables/billing.html)
- [Databricks: Audit logs schema](https://docs.databricks.com/aws/en/administration-guide/system-tables/audit-logs.html)
- [Databricks: Compute system tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/compute.html)
- [Databricks: Lakeflow jobs system table](https://docs.databricks.com/aws/en/administration-guide/system-tables/jobs.html)
- [Databricks: SQL Alerts](https://docs.databricks.com/aws/en/sql/user/alerts/index.html)
