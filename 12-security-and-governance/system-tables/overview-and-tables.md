# 개요와 주요 시스템 테이블

## 1. 왜 시스템 테이블이 필요한가

### 관찰가능성 (Observability) 부재 문제

데이터 플랫폼을 운영하다 보면 다음과 같은 질문이 반복적으로 생깁니다.

- "이번 달 클라우드 비용이 갑자기 왜 늘었지?"
- "누가 프로덕션 테이블을 실수로 삭제했을까?"
- "이 파이프라인 Job이 점점 느려지는 것 같은데, 언제부터였지?"
- "우리 팀에서 실제로 사용하지 않는 클러스터가 얼마나 되지?"

전통적인 데이터 플랫폼에서는 이런 질문에 답하려면 **CloudWatch, CloudTrail, 별도 APM 도구**, 혹은 수동 로그 분석이 필요했습니다. 도구가 분산되어 있고, 데이터 형식도 제각각이며, SQL로 바로 분석할 수 없다는 문제가 있었습니다.

**시스템 테이블(System Tables)** 은 이 문제를 해결합니다. Databricks 플랫폼이 스스로 생산하는 운영 데이터를 **`system` 카탈로그** 아래의 표준 테이블로 제공하여, 익숙한 SQL 한 줄로 플랫폼 전반을 관찰할 수 있게 합니다.

### 시스템 테이블로 해결하는 주요 과제

| 과제 | 기존 방식 | 시스템 테이블 방식 |
|------|-----------|-------------------|
| **클라우드 비용 관리** | AWS Cost Explorer + 수동 태깅 | `system.billing.usage` SQL 분석 |
| **보안 감사 (Security Audit)** | CloudTrail JSON 파싱 | `system.access.audit` 직접 조회 |
| **사용량 분석** | 워크스페이스별 개별 확인 | 계정 전체 통합 뷰 |
| **성능 / SLA 모니터링** | Job 로그 수동 확인 | `system.lakeflow.job_run_timeline` 집계 |
| **데이터 리니지 (Lineage)** | 별도 메타데이터 도구 | `system.lineage.table_lineage` |

---

## 2. 시스템 테이블 개요

### system 카탈로그 구조

```
system (카탈로그)
├── access           ← 보안 감사 로그
│   ├── audit
│   └── table_lineage (access 스키마의 lineage 뷰)
├── billing          ← 비용 / DBU 사용량
│   ├── usage
│   └── list_prices
├── compute          ← 클러스터 / 웨어하우스
│   ├── clusters
│   └── warehouse_events
├── lakeflow         ← Job / 워크플로우 실행
│   ├── jobs
│   ├── job_tasks
│   └── job_run_timeline
├── lineage          ← 데이터 리니지
│   ├── table_lineage
│   └── column_lineage
└── storage          ← 스토리지 최적화
    └── predictive_optimization_operations_history
```

### 활성화 방법

시스템 테이블은 **Unity Catalog** 가 활성화된 계정에서만 사용할 수 있습니다. 대부분의 테이블은 자동으로 활성화되지만, 일부 스키마는 Account Admin이 명시적으로 활성화해야 합니다.

**Account Console → Settings → System tables** 메뉴에서 각 스키마를 활성화합니다. 또는 REST API로도 가능합니다.

```bash
# REST API로 특정 스키마 활성화 (예: billing)
curl -X PUT "https://accounts.azuredatabricks.net/api/2.0/accounts/{account_id}/metastores/{metastore_id}/systemschemas/billing" \
  -H "Authorization: Bearer <token>"
```

> 참고: [System tables - Databricks Documentation](https://docs.databricks.com/en/admin/system-tables/index.html)

### 데이터 보존 기간 (Data Retention)

| 스키마 | 보존 기간 |
|--------|-----------|
| `system.billing.usage` | **365일** |
| `system.access.audit` | **365일** |
| `system.compute.clusters` | **365일** |
| `system.lakeflow.*` | **365일** |
| `system.lineage.*` | **90일** |
| `system.storage.*` | **365일** |

> 주의: 보존 기간은 Databricks 정책에 따라 변경될 수 있습니다. 장기 보관이 필요하다면 별도 테이블로 복사하는 파이프라인 구축을 권장합니다.

---

## 3. 주요 시스템 테이블 상세

### 3-1. system.billing.usage — 비용 분석

**`system.billing.usage`** 는 Databricks의 모든 워크로드에서 소비한 **DBU (Databricks Unit)** 사용량을 행 단위로 기록합니다. 클러스터, SQL Warehouse, Jobs, Serverless, DLT 파이프라인 등 모든 제품의 사용량이 포함됩니다.

**주요 컬럼**

| 컬럼 | 설명 |
|------|------|
| `usage_date` | 사용 날짜 |
| `sku_name` | SKU 이름 (예: `STANDARD_ALL_PURPOSE_COMPUTE`) |
| `billing_origin_product` | 제품 유형 (`JOBS`, `SQL`, `ALL_PURPOSE`, `DLT` 등) |
| `usage_quantity` | 소비한 DBU 수량 |
| `usage_metadata.cluster_id` | 관련 클러스터 ID |
| `usage_metadata.warehouse_id` | 관련 웨어하우스 ID |
| `usage_metadata.job_id` | 관련 Job ID |
| `identity_metadata.run_as` | 실행한 사용자 이메일 |
| `custom_tags` | 워크로드에 붙은 커스텀 태그 |

**`system.billing.list_prices`** 를 조인하면 DBU를 실제 USD 비용으로 변환할 수 있습니다.

```sql
-- 지난 30일간 워크로드 유형별 DBU 사용량 및 비용 추정
SELECT
    billing_origin_product,
    sku_name,
    SUM(u.usage_quantity)                              AS total_dbus,
    ROUND(SUM(u.usage_quantity * p.pricing.default), 2) AS estimated_cost_usd
FROM system.billing.usage u
LEFT JOIN system.billing.list_prices p
    ON u.sku_name = p.sku_name
    AND u.usage_date BETWEEN p.price_start_time::DATE AND COALESCE(p.price_end_time::DATE, CURRENT_DATE())
WHERE u.usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY estimated_cost_usd DESC;

-- 일별 비용 추이 (급증 감지용)
SELECT
    usage_date,
    billing_origin_product,
    SUM(usage_quantity) AS daily_dbus
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE() - INTERVAL 90 DAYS
GROUP BY 1, 2
ORDER BY usage_date, billing_origin_product;

-- 사용자별 DBU 소비 TOP 10
SELECT
    identity_metadata.run_as   AS user_email,
    billing_origin_product     AS product,
    SUM(usage_quantity)        AS total_dbus
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY total_dbus DESC
LIMIT 10;
```

---

### 3-2. system.access.audit — 감사 로그

**`system.access.audit`** 는 Databricks 플랫폼에서 발생하는 **모든 사용자 활동** 을 기록합니다. 테이블 조회, 클러스터 생성, 권한 변경, 토큰 발급 등 모든 API 호출이 포함됩니다. 보안 감사(Security Audit), GDPR/규정 준수, 내부 보안 정책 이행 증빙에 핵심적으로 사용됩니다.

**주요 컬럼**

| 컬럼 | 설명 |
|------|------|
| `event_time` | 이벤트 발생 시각 (UTC) |
| `event_date` | 이벤트 날짜 |
| `workspace_id` | 워크스페이스 ID |
| `action_name` | 수행된 액션 (예: `getTable`, `createCluster`, `deletePermissions`) |
| `user_identity.email` | 행위자 이메일 |
| `source_ip_address` | 요청 발생 IP |
| `request_params` | 요청 파라미터 (JSON) |
| `response.status_code` | 응답 상태 코드 |
| `service_name` | 서비스 이름 (예: `clusters`, `unityCatalog`) |

```sql
-- 최근 7일간 누가 어떤 테이블에 접근했는지 확인
SELECT
    event_date,
    user_identity.email     AS user_email,
    action_name,
    request_params.full_name_arg AS table_name,
    source_ip_address
FROM system.access.audit
WHERE action_name IN ('getTable', 'commandSubmit')
    AND event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY event_date DESC
LIMIT 100;

-- 비정상 접근 패턴 감지: 야간 시간대(22시~06시) 활동
SELECT
    user_identity.email,
    COUNT(*)                    AS late_night_actions,
    COLLECT_SET(action_name)    AS action_types
FROM system.access.audit
WHERE HOUR(event_time) NOT BETWEEN 6 AND 22
    AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY user_identity.email
HAVING COUNT(*) > 10
ORDER BY late_night_actions DESC;

-- 권한 변경 이력 (중요 보안 이벤트)
SELECT
    event_time,
    user_identity.email     AS actor,
    action_name,
    request_params
FROM system.access.audit
WHERE action_name IN (
    'updatePermissions', 'grantPrivileges', 'revokePrivileges',
    'createCredential', 'deleteCredential'
)
    AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
ORDER BY event_time DESC;
```

---

### 3-3. system.compute.clusters — 클러스터 사용 현황

**`system.compute.clusters`** 는 계정 내 모든 클러스터의 **설정, 상태, 이벤트 이력** 을 제공합니다. 비용 누수(자동 종료 미설정, 과도한 노드 수)를 발견하고, 클러스터 사용 패턴을 최적화하는 데 활용합니다.

**주요 컬럼**

| 컬럼 | 설명 |
|------|------|
| `cluster_id` | 클러스터 고유 ID |
| `cluster_name` | 클러스터 이름 |
| `cluster_source` | 생성 주체 (`UI`, `JOB`, `API`) |
| `driver_node_type` | 드라이버 노드 타입 |
| `node_type_id` | 워커 노드 타입 |
| `autoscale` | 오토스케일 설정 (min/max 워커 수) |
| `autotermination_minutes` | 자동 종료까지의 유휴 시간(분) |
| `state` | 현재 상태 (`RUNNING`, `TERMINATED`, `PENDING`) |
| `state_start_time` | 상태 시작 시각 |
| `owned_by` | 클러스터 소유자 |
| `custom_tags` | 커스텀 태그 |

```sql
-- 자동 종료 설정이 없거나 너무 긴 클러스터 (비용 누수 리스크)
SELECT
    cluster_id,
    cluster_name,
    owned_by,
    autotermination_minutes,
    state,
    TIMESTAMPDIFF(HOUR, state_start_time, CURRENT_TIMESTAMP()) AS running_hours
FROM system.compute.clusters
WHERE state = 'RUNNING'
    AND (autotermination_minutes IS NULL OR autotermination_minutes > 120)
ORDER BY running_hours DESC;

-- 클러스터 유형별 분포 파악
SELECT
    cluster_source,
    node_type_id,
    COUNT(DISTINCT cluster_id)  AS cluster_count,
    AVG(autotermination_minutes) AS avg_autotermination_min
FROM system.compute.clusters
WHERE state_start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY cluster_count DESC;
```

---

### 3-4. system.lakeflow.job_run_timeline — Job 실행 분석

**`system.lakeflow`** 스키마는 Databricks Workflows (Job)의 실행 이력을 상세히 기록합니다. `job_run_timeline` 은 각 Run의 시작/종료 시각, 상태, 컴퓨트 사용 정보를 제공하며 **SLA 모니터링과 성능 추이 분석** 에 적합합니다.

**주요 테이블 & 컬럼**

| 테이블 | 주요 컬럼 |
|--------|-----------|
| `system.lakeflow.jobs` | `job_id`, `run_id`, `result_state`, `start_time`, `end_time`, `error_message` |
| `system.lakeflow.job_tasks` | `task_key`, `run_id`, `result_state`, `start_time`, `end_time` |
| `system.lakeflow.job_run_timeline` | `job_id`, `run_id`, `period_start_time`, `period_end_time`, `cluster_id`, `dbu_consumed` |

```sql
-- 최근 7일간 실패한 Job (에러 메시지 포함)
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
    DATE(start_time)                                        AS run_date,
    AVG(TIMESTAMPDIFF(MINUTE, start_time, end_time))        AS avg_duration_min,
    COUNT(*)                                                AS run_count,
    SUM(CASE WHEN result_state = 'FAILED' THEN 1 ELSE 0 END) AS fail_count
FROM system.lakeflow.jobs
WHERE start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY job_id, DATE(start_time)
ORDER BY job_id, run_date;

-- Job별 DBU 소비량 (비용 배분 분석)
SELECT
    t.job_id,
    SUM(t.dbu_consumed)                                 AS total_dbu,
    ROUND(SUM(t.dbu_consumed * p.pricing.default), 2)   AS estimated_cost_usd
FROM system.lakeflow.job_run_timeline t
LEFT JOIN system.billing.list_prices p
    ON p.sku_name LIKE '%JOBS%'
    AND t.period_start_time::DATE BETWEEN p.price_start_time::DATE AND COALESCE(p.price_end_time::DATE, CURRENT_DATE())
WHERE t.period_start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY t.job_id
ORDER BY total_dbu DESC
LIMIT 20;
```

---

### 3-5. system.access.table_lineage — 리니지

**`system.lineage`** 스키마는 테이블과 컬럼 간의 **데이터 흐름(Lineage)** 을 기록합니다. 특정 테이블을 변경할 때 downstream 영향을 파악하거나, 컬럼 출처를 추적하는 데 활용합니다.

```sql
-- 특정 테이블에 의존하는 모든 downstream 테이블
SELECT
    target_table_full_name,
    target_type,
    source_table_full_name
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
ORDER BY target_table_full_name;

-- 특정 컬럼의 upstream 출처 추적
SELECT
    source_table_full_name,
    source_column_name,
    target_table_full_name,
    target_column_name
FROM system.lineage.column_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_revenue'
    AND target_column_name = 'total_amount';
```

---

## 4. 실전 활용 시나리오

### 시나리오 1: 비용 대시보드 구축

**목표**: 태그 기반으로 팀/프로젝트별 비용을 일별로 시각화합니다.

```sql
-- 커스텀 태그 기반 팀별 비용 집계
SELECT
    usage_date,
    custom_tags.Team                                        AS team,
    custom_tags.Project                                     AS project,
    billing_origin_product,
    SUM(u.usage_quantity)                                   AS total_dbus,
    ROUND(SUM(u.usage_quantity * p.pricing.default), 2)     AS estimated_cost_usd
FROM system.billing.usage u
LEFT JOIN system.billing.list_prices p
    ON u.sku_name = p.sku_name
    AND u.usage_date BETWEEN p.price_start_time::DATE
        AND COALESCE(p.price_end_time::DATE, CURRENT_DATE())
WHERE u.usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
    AND custom_tags.Team IS NOT NULL
GROUP BY 1, 2, 3, 4
ORDER BY usage_date, team, estimated_cost_usd DESC;
```

> 이 쿼리를 Databricks SQL Dashboard의 차트 위젯에 연결하면 팀별 실시간 비용 대시보드가 됩니다.

---

### 시나리오 2: 비정상 접근 감지 (Security Monitoring)

**목표**: 야간 대량 접근, 해외 IP, 짧은 시간 내 대량 삭제 등 비정상 패턴을 탐지합니다.

```sql
-- 1시간 내 DROP/DELETE 액션 다수 수행한 사용자 탐지
SELECT
    user_identity.email,
    DATE_TRUNC('hour', event_time)  AS hour_bucket,
    COUNT(*)                        AS action_count,
    COLLECT_SET(action_name)        AS actions
FROM system.access.audit
WHERE action_name IN ('deleteTable', 'dropSchema', 'deleteCatalog', 'deleteVolume')
    AND event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY 1, 2
HAVING COUNT(*) >= 5
ORDER BY action_count DESC;

-- 신규 IP에서 처음 접근한 사용자 (기존 접근 IP 목록과 비교)
WITH known_ips AS (
    SELECT user_identity.email, source_ip_address
    FROM system.access.audit
    WHERE event_date BETWEEN CURRENT_DATE() - INTERVAL 60 DAYS
        AND CURRENT_DATE() - INTERVAL 8 DAYS
    GROUP BY 1, 2
),
recent AS (
    SELECT user_identity.email, source_ip_address, MAX(event_time) AS last_seen
    FROM system.access.audit
    WHERE event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
    GROUP BY 1, 2
)
SELECT r.email, r.source_ip_address, r.last_seen
FROM recent r
LEFT JOIN known_ips k
    ON r.email = k.email AND r.source_ip_address = k.source_ip_address
WHERE k.source_ip_address IS NULL;
```

---

### 시나리오 3: 미사용 리소스 식별

**목표**: 30일 이상 사용되지 않은 클러스터나 웨어하우스를 찾아 비용을 절감합니다.

```sql
-- 최근 30일간 billing.usage에 나타나지 않은 클러스터 (미사용 클러스터)
SELECT
    c.cluster_id,
    c.cluster_name,
    c.owned_by,
    c.state,
    c.autotermination_minutes
FROM system.compute.clusters c
LEFT JOIN (
    SELECT DISTINCT usage_metadata.cluster_id AS cluster_id
    FROM system.billing.usage
    WHERE usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
        AND usage_metadata.cluster_id IS NOT NULL
) active ON c.cluster_id = active.cluster_id
WHERE active.cluster_id IS NULL
    AND c.cluster_source = 'UI'   -- 수동 생성 클러스터만
ORDER BY c.owned_by;
```

---

### 시나리오 4: SLA 모니터링

**목표**: 중요 파이프라인 Job의 완료 시각이 SLA를 넘겼는지 매일 체크합니다.

```sql
-- 특정 Job이 목표 완료 시각(예: 매일 오전 7시) 이전에 완료되었는지 확인
SELECT
    job_id,
    DATE(start_time)                                            AS run_date,
    MIN(start_time)                                             AS first_start,
    MAX(end_time)                                               AS last_end,
    HOUR(MAX(end_time))                                         AS end_hour,
    TIMESTAMPDIFF(MINUTE, MIN(start_time), MAX(end_time))       AS total_duration_min,
    CASE WHEN HOUR(MAX(end_time)) >= 7 THEN 'SLA_BREACH' ELSE 'OK' END AS sla_status
FROM system.lakeflow.jobs
WHERE job_id IN (123456, 234567)   -- 모니터링 대상 Job ID
    AND result_state = 'SUCCEEDED'
    AND start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY job_id, DATE(start_time)
ORDER BY run_date DESC;
```

---

## 5. 시스템 테이블 전체 목록

| 스키마 | 테이블 | 내용 | 보존 기간 |
|--------|--------|------|-----------|
| `system.access` | `audit` | 사용자 활동 감사 로그 | 365일 |
| `system.billing` | `usage` | DBU 사용량 상세 | 365일 |
| `system.billing` | `list_prices` | SKU별 DBU 단가 | 365일 |
| `system.compute` | `clusters` | 클러스터 설정 및 이벤트 | 365일 |
| `system.compute` | `warehouse_events` | SQL Warehouse 이벤트 로그 | 365일 |
| `system.lakeflow` | `jobs` | Job 실행 이력 | 365일 |
| `system.lakeflow` | `job_tasks` | Task 수준 실행 이력 | 365일 |
| `system.lakeflow` | `job_run_timeline` | Run 타임라인 및 DBU | 365일 |
| `system.lineage` | `table_lineage` | 테이블 간 데이터 흐름 | 90일 |
| `system.lineage` | `column_lineage` | 컬럼 간 데이터 흐름 | 90일 |
| `system.storage` | `predictive_optimization_operations_history` | 자동 OPTIMIZE/VACUUM 이력 | 365일 |
| `system.information_schema` | 다양한 뷰 | 테이블·컬럼·권한 메타데이터 | - |

---

## 6. 장단점과 한계

### 장점

- **별도 에이전트 불필요**: 플랫폼이 자동으로 수집하므로 설정 비용이 없습니다.
- **SQL 직접 조회**: BI 도구, Databricks SQL Dashboard, Notebook 어디서든 즉시 분석 가능합니다.
- **계정 전체 통합 뷰**: 여러 워크스페이스의 데이터가 단일 `system` 카탈로그로 통합됩니다.
- **Unity Catalog 권한 통합**: 기존 UC 권한 모델로 시스템 테이블 접근을 제어할 수 있습니다.

### 한계 및 고려사항

| 항목 | 내용 |
|------|------|
| **데이터 지연 (Latency)** | 대부분의 테이블은 수 분~수십 분 지연이 있습니다. 실시간 모니터링에는 적합하지 않습니다. |
| **보존 기간** | `system.lineage.*` 는 90일만 보관됩니다. 장기 분석이 필요하면 별도 복사 필요합니다. |
| **쿼리 성능** | 감사 로그(`audit`)는 데이터량이 매우 많아 `event_date` 파티션 필터를 반드시 사용해야 합니다. |
| **스키마 변경 가능성** | Databricks가 GA 전 Preview 테이블의 스키마를 변경할 수 있으므로 프로덕션 파이프라인 전에 GA 여부를 확인합니다. |
| **Unity Catalog 전용** | Hive Metastore(레거시) 환경에서는 사용 불가합니다. |
| **비용** | 시스템 테이블 조회도 SQL Warehouse DBU를 소비합니다. 대규모 집계 쿼리는 서버리스 웨어하우스 활용을 권장합니다. |

---

## 7. 베스트 프랙티스

### 7-1. 정기 모니터링 Job 설정

시스템 테이블을 매일 집계하여 별도 **Delta 테이블** 로 저장하는 Job을 구성하면 이력 관리와 대시보드 성능을 모두 개선할 수 있습니다.

```python
# 매일 전날 비용 집계를 별도 테이블에 적재하는 패턴
spark.sql("""
INSERT INTO main.monitoring.daily_cost_summary
SELECT
    usage_date,
    billing_origin_product,
    custom_tags.Team    AS team,
    SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE usage_date = CURRENT_DATE() - INTERVAL 1 DAY
GROUP BY 1, 2, 3
""")
```

> Databricks Workflows에서 이 노트북을 매일 오전 1시에 실행하도록 스케줄링합니다.

### 7-2. 알림 설정 (Alerting)

**Databricks SQL Alert** 를 활용하면 시스템 테이블 기반 이상 감지를 슬랙/이메일로 알림 받을 수 있습니다.

1. SQL 쿼리를 저장 (예: 야간 비정상 접근 쿼리)
2. **Databricks SQL → Alerts → New Alert** 에서 해당 쿼리 선택
3. 조건 설정: 결과 행 수 > 0 이면 트리거
4. 알림 대상(Slack Webhook, 이메일) 설정

### 7-3. 태그 기반 분석

비용 분석의 정확도를 높이려면 클러스터, Job, SQL Warehouse에 일관된 **커스텀 태그(Custom Tags)** 를 붙이는 태깅 정책을 먼저 수립해야 합니다.

권장 태그 키 예시:

```
Team       = data-platform
Project    = crm-analytics
Env        = production
CostCenter = CC-1234
```

**Cluster Policies** 를 통해 태그 입력을 강제할 수 있습니다.

> 참고: [Cluster policies - Databricks Documentation](https://docs.databricks.com/en/compute/cluster-policies.html)

### 7-4. 접근 권한 관리

시스템 테이블은 민감한 운영 정보를 포함하므로, 필요한 그룹에만 읽기 권한을 부여합니다.

```sql
-- 보안 팀에게 감사 로그 읽기 권한 부여
GRANT SELECT ON TABLE system.access.audit TO `security-team`;

-- 재무 팀에게 빌링 테이블 읽기 권한 부여
GRANT SELECT ON TABLE system.billing.usage TO `finance-team`;
GRANT SELECT ON TABLE system.billing.list_prices TO `finance-team`;
```

---

## 참고 자료

| 항목 | 링크 |
|------|------|
| 시스템 테이블 공식 문서 | [docs.databricks.com/en/admin/system-tables](https://docs.databricks.com/en/admin/system-tables/index.html) |
| billing.usage 스키마 참조 | [docs.databricks.com/en/admin/system-tables/billing](https://docs.databricks.com/en/admin/system-tables/billing.html) |
| access.audit 스키마 참조 | [docs.databricks.com/en/admin/system-tables/audit-logs](https://docs.databricks.com/en/admin/system-tables/audit-logs.html) |
| lakeflow 스키마 참조 | [docs.databricks.com/en/admin/system-tables/lakeflow](https://docs.databricks.com/en/admin/system-tables/lakeflow.html) |
| lineage 스키마 참조 | [docs.databricks.com/en/admin/system-tables/lineage](https://docs.databricks.com/en/admin/system-tables/lineage.html) |
| Cluster Policies | [docs.databricks.com/en/compute/cluster-policies](https://docs.databricks.com/en/compute/cluster-policies.html) |
