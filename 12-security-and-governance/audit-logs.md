# 감사 로그 (Audit Logs) 상세

## 감사 로그란?

Databricks의 **감사 로그(Audit Logs)** 는 플랫폼에서 발생하는 모든 사용자 활동과 시스템 이벤트를 기록합니다. "누가, 언제, 무엇을, 어디서" 했는지를 추적할 수 있어 보안 모니터링, 규정 준수, 사고 조사에 필수적입니다.

> 💡 기본 개념은 [보안 개요](./security-overview.md)의 "감사 로그" 섹션에서 소개했습니다. 이 문서에서는 감사 테이블의 구조, 실전 쿼리, 대시보드 구축을 상세히 다룹니다.

---

## system.access.audit 테이블 구조

감사 로그는 `system.access.audit` 시스템 테이블에 저장됩니다.

### 주요 컬럼

| 컬럼 | 타입 | 설명 |
|------|------|------|
| **event_time** | TIMESTAMP | 이벤트 발생 시각 |
| **event_date** | DATE | 이벤트 발생 날짜 (파티션 키) |
| **workspace_id** | BIGINT | 이벤트가 발생한 워크스페이스 ID |
| **service_name** | STRING | 서비스 이름 (예: `notebook`, `clusters`, `unityCatalog`) |
| **action_name** | STRING | 수행된 작업 (예: `runCommand`, `create`, `delete`) |
| **user_identity** | STRUCT | 사용자 정보 (이메일, IP 등) |
| **user_identity.email** | STRING | 작업을 수행한 사용자/SP의 이메일 |
| **source_ip_address** | STRING | 요청 출발 IP 주소 |
| **request_params** | MAP | 요청 파라미터 (테이블 이름, 쿼리 등) |
| **response** | STRUCT | 응답 정보 (상태 코드, 에러 메시지) |
| **response.status_code** | INT | HTTP 응답 코드 |
| **response.error_message** | STRING | 에러 메시지 (실패 시) |
| **audit_level** | STRING | 감사 수준 (`ACCOUNT_LEVEL` 또는 `WORKSPACE_LEVEL`) |

### 주요 service_name 값

| service_name | 설명 |
|-------------|------|
| `accounts` | 계정 관리 (사용자 추가, 그룹 변경) |
| `clusters` | 클러스터 생성, 시작, 종료 |
| `notebook` | 노트북 생성, 실행, 수정 |
| `jobs` | Job 생성, 실행, 스케줄 |
| `unityCatalog` | UC 객체 관리 (테이블, 권한 등) |
| `databrickssql` | SQL 웨어하우스 쿼리 실행 |
| `iamRole` | IAM 관련 이벤트 |
| `secrets` | Secret 접근 |

---

## 주요 감사 쿼리

### 1. 로그인 실패 이벤트

```sql
-- 최근 24시간 내 로그인 실패 조회
SELECT
  event_time,
  user_identity.email AS user_email,
  source_ip_address,
  response.error_message,
  response.status_code
FROM system.access.audit
WHERE action_name IN ('login', 'tokenLogin', 'samlLogin')
  AND response.status_code != 200
  AND event_date >= CURRENT_DATE() - INTERVAL 1 DAY
ORDER BY event_time DESC;
```

### 2. 데이터 접근 이력

```sql
-- 특정 테이블에 대한 접근 이력
SELECT
  event_time,
  user_identity.email AS user_email,
  action_name,
  request_params.full_name_arg AS object_name,
  source_ip_address
FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND request_params.full_name_arg LIKE 'production.ecommerce.customers%'
  AND event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY event_time DESC;
```

### 3. 권한 변경 추적

```sql
-- GRANT/REVOKE 이벤트 추적
SELECT
  event_time,
  user_identity.email AS changed_by,
  action_name,
  request_params.securable_type AS object_type,
  request_params.securable_full_name AS object_name,
  request_params.principal AS granted_to,
  request_params.changes AS permission_changes
FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND action_name IN ('updatePermissions', 'grantPermission', 'revokePermission')
  AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
ORDER BY event_time DESC;
```

### 4. 비정상 접근 패턴 탐지

```sql
-- 비정상적으로 많은 테이블을 조회한 사용자
SELECT
  user_identity.email AS user_email,
  COUNT(DISTINCT request_params.full_name_arg) AS unique_tables_accessed,
  COUNT(*) AS total_queries
FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND action_name = 'getTable'
  AND event_date = CURRENT_DATE()
GROUP BY user_identity.email
HAVING unique_tables_accessed > 50
ORDER BY unique_tables_accessed DESC;
```

### 5. 클러스터 생성/삭제 이력

```sql
-- 클러스터 생성/삭제 이벤트
SELECT
  event_time,
  user_identity.email AS user_email,
  action_name,
  request_params.cluster_name,
  request_params.num_workers,
  request_params.node_type_id
FROM system.access.audit
WHERE service_name = 'clusters'
  AND action_name IN ('create', 'permanentDelete', 'edit')
  AND event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY event_time DESC;
```

### 6. Secret 접근 이력

```sql
-- Secret 접근 이벤트 (누가 어떤 Secret을 읽었는지)
SELECT
  event_time,
  user_identity.email AS user_email,
  action_name,
  request_params.scope AS secret_scope,
  request_params.key AS secret_key,
  source_ip_address
FROM system.access.audit
WHERE service_name = 'secrets'
  AND event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY event_time DESC;
```

### 7. 야간/주말 비정상 활동

```sql
-- 업무 시간 외 활동 탐지 (한국 기준 22시~06시, 주말)
SELECT
  event_time,
  user_identity.email AS user_email,
  service_name,
  action_name,
  source_ip_address
FROM system.access.audit
WHERE event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
  AND (
    HOUR(event_time) < 6 OR HOUR(event_time) >= 22  -- 야간
    OR DAYOFWEEK(event_time) IN (1, 7)               -- 주말
  )
  AND user_identity.email NOT LIKE '%service-principal%'  -- SP 제외
ORDER BY event_time DESC
LIMIT 100;
```

---

## 감사 대시보드 구축

### 핵심 위젯 구성

감사 대시보드에 포함할 핵심 위젯은 다음과 같습니다.

| 위젯 | 쿼리 목적 | 시각화 유형 |
|------|----------|-----------|
| **일별 이벤트 추이** | 전체 감사 이벤트 수 추이 | 라인 차트 |
| **로그인 실패 현황** | 실패한 인증 시도 | 숫자 카운터 + 테이블 |
| **서비스별 활동** | 서비스별 이벤트 분포 | 파이 차트 |
| **Top 활동 사용자** | 가장 활발한 사용자 | 바 차트 |
| **권한 변경 로그** | 최근 GRANT/REVOKE | 테이블 |
| **비정상 접근** | 야간/과다 접근 | 테이블 + 알림 |

### 일별 이벤트 추이 쿼리

```sql
SELECT
  event_date,
  service_name,
  COUNT(*) AS event_count
FROM system.access.audit
WHERE event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY event_date, service_name
ORDER BY event_date;
```

### 서비스별 활동 분포

```sql
SELECT
  service_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_identity.email) AS unique_users
FROM system.access.audit
WHERE event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY service_name
ORDER BY event_count DESC;
```

---

## 알림 설정

Databricks SQL 알림(Alert)을 설정하여 보안 이벤트를 실시간 감지할 수 있습니다.

### 알림 1: 로그인 실패 급증

```sql
-- 최근 1시간 내 로그인 실패가 10건 이상이면 알림
SELECT COUNT(*) AS failed_logins
FROM system.access.audit
WHERE action_name IN ('login', 'samlLogin')
  AND response.status_code != 200
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 1 HOUR;
-- 알림 조건: failed_logins > 10
```

### 알림 2: 관리자 권한 변경

```sql
-- Account Admin 또는 Workspace Admin 권한 변경 감지
SELECT
  event_time,
  user_identity.email AS changed_by,
  request_params
FROM system.access.audit
WHERE action_name IN ('setAdmin', 'removeAdmin', 'updatePermissions')
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 1 HOUR
  AND response.status_code = 200;
-- 알림 조건: 결과가 1건 이상
```

### 알림 3: 대량 데이터 다운로드

```sql
-- 단기간 내 다수의 테이블 SELECT 감지
SELECT
  user_identity.email,
  COUNT(DISTINCT request_params.full_name_arg) AS tables_accessed
FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND action_name = 'getTable'
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 1 HOUR
GROUP BY user_identity.email
HAVING tables_accessed > 20;
-- 알림 조건: 결과가 1건 이상
```

---

## 감사 로그 보존 정책

| 항목 | 설명 |
|------|------|
| **기본 보존 기간** | 시스템 테이블은 365일간 데이터를 보존합니다 |
| **장기 보존** | 규제 요건에 따라 별도 테이블에 복제하여 장기 보존합니다 |
| **외부 SIEM** | Splunk, Sentinel 등 SIEM 도구로 감사 로그를 전송할 수 있습니다 |

```sql
-- 장기 보존을 위한 감사 로그 복제
CREATE TABLE IF NOT EXISTS compliance.audit.long_term_audit AS
SELECT * FROM system.access.audit WHERE 1=0;

-- 매일 증분 복제 (Job으로 스케줄)
INSERT INTO compliance.audit.long_term_audit
SELECT *
FROM system.access.audit
WHERE event_date = CURRENT_DATE() - INTERVAL 1 DAY;
```

---

## 쿼리 성능 최적화

감사 테이블은 대규모 데이터를 포함하므로, 쿼리 성능을 위해 다음 사항을 지켜야 합니다.

| 원칙 | 설명 |
|------|------|
| **event_date 필터 필수** | `event_date` 파티션 키를 항상 WHERE 절에 포함합니다 |
| **service_name 필터** | 특정 서비스만 조회하면 스캔 범위가 줄어듭니다 |
| **LIMIT 사용** | 탐색적 쿼리에는 LIMIT을 설정합니다 |
| **집계 우선** | 상세 조회 전 COUNT/GROUP BY로 규모를 파악합니다 |

```sql
-- ✅ 좋은 예: 파티션 필터 + 서비스 필터
SELECT * FROM system.access.audit
WHERE event_date >= '2025-03-01' AND event_date <= '2025-03-31'
  AND service_name = 'unityCatalog';

-- ❌ 나쁜 예: 파티션 필터 없이 전체 스캔
SELECT * FROM system.access.audit
WHERE action_name = 'login';
```

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **대시보드 구축** | 핵심 보안 지표를 시각화하는 감사 대시보드를 운영합니다 |
| **알림 설정** | 로그인 실패, 권한 변경 등 주요 이벤트에 알림을 설정합니다 |
| **장기 보존** | 규제 요건에 따라 감사 로그를 장기 보존합니다 |
| **정기 리뷰** | 월간 보안 리뷰에서 감사 로그를 분석합니다 |
| **SIEM 연동** | 기존 보안 도구(Splunk, Sentinel)와 연동합니다 |
| **event_date 필터** | 쿼리 시 반드시 날짜 필터를 포함합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **system.access.audit** | 모든 감사 이벤트가 저장되는 시스템 테이블입니다 |
| **service_name** | 이벤트가 발생한 서비스를 식별합니다 |
| **action_name** | 수행된 작업을 식별합니다 |
| **user_identity** | 누가 작업을 수행했는지 기록합니다 |
| **알림** | SQL Alert로 보안 이벤트를 실시간 감지합니다 |
| **장기 보존** | 규제 요건에 따라 별도 테이블에 복제합니다 |

---

## 참고 링크

- [Databricks: Audit log system table](https://docs.databricks.com/aws/en/admin/system-tables/audit.html)
- [Databricks: Monitor account activity](https://docs.databricks.com/aws/en/admin/account-settings/audit-logs.html)
- [Databricks: System tables reference](https://docs.databricks.com/aws/en/admin/system-tables/)
- [Azure Databricks: Audit logs](https://learn.microsoft.com/en-us/azure/databricks/admin/system-tables/audit)
