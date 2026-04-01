# 수집 파이프라인 구성

## 왜 올바른 파이프라인 구성이 중요한가요?

Lakeflow Connect의 수집 파이프라인은 한번 설정하면 지속적으로 운영되는 시스템입니다. 초기 구성을 올바르게 하지 않으면 **데이터 누락, 성능 저하, 불필요한 비용 증가**등의 문제가 발생할 수 있습니다. 이 문서에서는 수집 파이프라인의 각 구성 요소를 상세히 설명하고, 운영 환경에서의 모범 사례를 안내합니다.

---

## Unity Catalog Connection 설정

### Connection이란?

Unity Catalog **Connection**은 외부 데이터 소스에 접속하기 위한 자격증명과 연결 정보를 **중앙에서 안전하게 관리**하는 객체입니다. 한번 생성하면 여러 파이프라인에서 재사용할 수 있습니다.

| 구성 요소 | 역할 | 사용하는 파이프라인 |
|-----------|------|-------------------|
| **Connection: mysql_prod**| MySQL 프로덕션 연결 | 고객 데이터 수집, 주문 데이터 수집 |
| **Connection: salesforce_crm**| Salesforce CRM 연결 | CRM 동기화 |
| **Connection: postgres_analytics**| PostgreSQL 분석 DB 연결 | 분석 DB 수집 |

### Connection 생성 옵션 상세

#### MySQL Connection

```sql
CREATE CONNECTION mysql_prod
TYPE MYSQL
OPTIONS (
    host = 'mysql-prod.example.com',    -- MySQL 서버 주소
    port = '3306',                       -- 포트 (기본값: 3306)
    user = 'databricks_reader',          -- 접속 계정
    password = secret('scope', 'mysql-password'),  -- Secret 참조
    useSSL = 'true',                     -- SSL 암호화 사용
    trustServerCertificate = 'false'     -- 서버 인증서 검증
);
```

#### PostgreSQL Connection

```sql
CREATE CONNECTION postgres_analytics
TYPE POSTGRESQL
OPTIONS (
    host = 'postgres.example.com',
    port = '5432',
    user = 'databricks_cdc',
    password = secret('scope', 'pg-password'),
    database = 'analytics',              -- 대상 데이터베이스 지정
    sslmode = 'require'                  -- SSL 모드 (require/verify-full)
);
```

#### SQL Server Connection

```sql
CREATE CONNECTION sqlserver_erp
TYPE SQLSERVER
OPTIONS (
    host = 'sqlserver.example.com',
    port = '1433',
    user = 'databricks_reader',
    password = secret('scope', 'mssql-password'),
    database = 'ERP',
    encrypt = 'true',
    trustServerCertificate = 'false'
);
```

#### Salesforce Connection

```sql
CREATE CONNECTION salesforce_crm
TYPE SALESFORCE
OPTIONS (
    sfUrl = 'https://mycompany.my.salesforce.com',
    oauthClientId = secret('scope', 'sf-client-id'),
    oauthClientSecret = secret('scope', 'sf-client-secret'),
    oauthRefreshToken = secret('scope', 'sf-refresh-token')
);
```

### Connection 보안 모범 사례

| 항목 | 권장 사항 | 이유 |
|------|---------|------|
| ** 비밀번호 관리**| `secret()` 함수 사용 | 평문 노출 방지 |
| ** 계정 권한**| 최소 필요 권한만 부여 | 보안 원칙 준수 |
| ** 전용 계정**| Databricks 전용 계정 생성 | 감사 추적 용이 |
| **SSL 사용**| SSL/TLS 활성화 | 통신 구간 암호화 |
| ** 네트워크 제한**| IP 화이트리스트 설정 | 비인가 접근 차단 |
| ** 비밀번호 로테이션**| 주기적 갱신 (90일 권장) | 자격증명 유출 대비 |

---

## 테이블 선택 및 필터링

### 전체 스키마 수집 vs 개별 테이블 선택

| 방식 | 장점 | 단점 | 적합한 경우 |
|------|------|------|-----------|
| ** 전체 스키마**| 설정 간단, 새 테이블 자동 수집 | 불필요한 테이블까지 수집 | 소스 스키마의 모든 데이터가 필요한 경우 |
| ** 개별 테이블 선택**| 필요한 데이터만 수집, 비용 절감 | 새 테이블 추가 시 설정 변경 필요 | 특정 테이블만 필요한 경우 |

### 테이블 선택 패턴

```python
# 방법 1: 전체 스키마 수집
@dlt.table
def ingest_all():
    return mysql.read_changefeed(
        connection="mysql_prod",
        source_schema="ecommerce",
        # source_table 생략 시 전체 스키마 수집
    )

# 방법 2: 특정 테이블만 선택
@dlt.table
def ingest_customers():
    return mysql.read_changefeed(
        connection="mysql_prod",
        source_schema="ecommerce",
        source_table="customers"
    )
```

### UI에서 테이블 필터링

Pipeline UI에서 테이블을 선택할 때 다음과 같은 필터링 옵션을 사용할 수 있습니다:

| 필터 옵션 | 설명 | 예시 |
|----------|------|------|
| ** 스키마 선택**| 특정 스키마의 모든 테이블 | `ecommerce` 스키마 전체 |
| ** 테이블 개별 선택**| 체크박스로 필요한 테이블만 선택 | `customers`, `orders`만 선택 |
| ** 와일드카드 패턴** | 이름 패턴으로 필터 (일부 커넥터) | `dim_*`, `fact_*` |
| **제외 패턴**| 특정 테이블 제외 | 임시 테이블, 로그 테이블 제외 |

---

## 스키마 매핑 규칙 (소스 → 타겟)

Lakeflow Connect는 소스 테이블의 스키마를 ** 자동으로 대상 Delta 테이블에 매핑**합니다.

### 기본 매핑 구조

```
소스: <connection>.<source_schema>.<source_table>
타겟: <target_catalog>.<target_schema>.<source_table>

예시:
소스: mysql_prod.ecommerce.customers
타겟: analytics.bronze.customers
```

### 네이밍 규칙

| 소스 객체 | 대상 매핑 | 변환 규칙 |
|----------|---------|---------|
| **스키마 이름** | 대상 스키마 직접 지정 | UI에서 선택 |
| **테이블 이름** | 소스 이름 그대로 사용 | 변경 불가 (대소문자 유지) |
| **컬럼 이름** | 소스 이름 그대로 사용 | 대소문자 유지 |

### 자동 추가 메타데이터 컬럼

Lakeflow Connect는 수집 시 다음 메타데이터 컬럼을 **자동으로 추가**합니다:

| 메타데이터 컬럼 | 타입 | 설명 |
|---------------|------|------|
| `_ingested_at` | TIMESTAMP | Databricks에 수집된 시간 |
| `_change_type` | STRING | CDC 변경 유형 (INSERT/UPDATE/DELETE) |
| `_commit_timestamp` | TIMESTAMP | 소스에서의 변경 시간 |
| `_source_schema` | STRING | 소스 스키마 이름 |
| `_source_table` | STRING | 소스 테이블 이름 |

```sql
-- 메타데이터 컬럼 활용 예시
SELECT
    _change_type,
    _commit_timestamp,
    customer_id,
    name
FROM analytics.bronze.customers
WHERE _change_type = 'UPDATE'
  AND _commit_timestamp >= CURRENT_DATE - INTERVAL 1 DAY
ORDER BY _commit_timestamp DESC;
```

---

## 데이터 타입 변환 규칙

소스 데이터베이스의 데이터 타입은 Delta Lake 호환 타입으로 자동 변환됩니다.

### MySQL → Delta 타입 매핑

| MySQL 타입 | Delta (Spark) 타입 | 비고 |
|-----------|-------------------|------|
| `TINYINT` | `TINYINT` / `BOOLEAN` | TINYINT(1)은 BOOLEAN으로 변환 가능 |
| `SMALLINT` | `SMALLINT` | |
| `INT` | `INT` | |
| `BIGINT` | `BIGINT` | |
| `FLOAT` | `FLOAT` | |
| `DOUBLE` | `DOUBLE` | |
| `DECIMAL(p,s)` | `DECIMAL(p,s)` | 정밀도 유지 |
| `VARCHAR(n)` | `STRING` | 길이 제한 해제 |
| `TEXT` | `STRING` | |
| `DATE` | `DATE` | |
| `DATETIME` | `TIMESTAMP` | |
| `TIMESTAMP` | `TIMESTAMP` | |
| `BLOB` | `BINARY` | |
| `JSON` | `STRING` | JSON 문자열로 저장 |
| `ENUM` | `STRING` | 열거값이 문자열로 변환 |

### PostgreSQL → Delta 타입 매핑

| PostgreSQL 타입 | Delta (Spark) 타입 | 비고 |
|----------------|-------------------|------|
| `SMALLINT` | `SMALLINT` | |
| `INTEGER` | `INT` | |
| `BIGINT` | `BIGINT` | |
| `NUMERIC(p,s)` | `DECIMAL(p,s)` | |
| `VARCHAR(n)` | `STRING` | |
| `TEXT` | `STRING` | |
| `BOOLEAN` | `BOOLEAN` | |
| `DATE` | `DATE` | |
| `TIMESTAMP` | `TIMESTAMP` | |
| `TIMESTAMPTZ` | `TIMESTAMP` | 타임존 정보가 UTC로 변환 |
| `JSONB` | `STRING` | JSON 문자열로 저장 |
| `UUID` | `STRING` | UUID가 문자열로 변환 |
| `ARRAY` | `ARRAY` | 중첩 배열 지원 |

> ⚠️ **타입 변환 주의사항**: `TIMESTAMPTZ`(타임존 포함 타임스탬프)는 UTC로 변환됩니다. 소스 데이터의 타임존을 유지하고 싶다면, Silver 변환 시 별도로 타임존 변환 로직을 추가해야 합니다.

---

## 파티셔닝 전략

수집된 Delta 테이블의 파티셔닝은 이후 쿼리 성능에 큰 영향을 미칩니다.

### 파티셔닝 옵션

| 방식 | 설명 | 적합한 경우 |
|------|------|-----------|
| **파티셔닝 없음** (기본) | 데이터가 그대로 적재됨 | 소규모 테이블, 다양한 쿼리 패턴 |
| **Liquid Clustering**(권장) | 자동 데이터 레이아웃 최적화 | 대부분의 경우에 최적 |
| **Hive 스타일 파티셔닝** | 특정 컬럼 기준으로 물리적 분할 | 날짜 기반 시계열 데이터 |

```sql
-- Silver 테이블에서 Liquid Clustering 적용 (권장)
CREATE OR REFRESH STREAMING TABLE analytics.silver.orders
CLUSTER BY (order_date, customer_id)
AS SELECT * FROM STREAM(analytics.bronze.orders);

-- 기존 테이블에 Liquid Clustering 추가
ALTER TABLE analytics.silver.orders
CLUSTER BY (order_date, customer_id);
```

> 💡 **Liquid Clustering 권장**: Databricks는 전통적인 Hive 스타일 파티셔닝보다 **Liquid Clustering**을 권장합니다. 파티션 키를 잘못 선택하면 성능이 오히려 저하되지만, Liquid Clustering은 쿼리 패턴에 맞게 자동으로 데이터를 재배치합니다.

---

## 모니터링 설정

### Pipeline UI 메트릭

수집 파이프라인 실행 시 Pipeline UI에서 다음 정보를 실시간으로 모니터링할 수 있습니다.

| 모니터링 항목 | 확인 위치 | 설명 |
|-------------|---------|------|
| **파이프라인 상태**| Pipeline 개요 페이지 | RUNNING, IDLE, FAILED, STOPPED |
| ** 테이블별 수집 현황**| 각 테이블 노드 클릭 | 처리 건수, 지연시간, 에러 |
| ** 업데이트 이력**| Updates 탭 | 과거 실행 기록, 소요 시간 |
| ** 이벤트 로그** | Events 탭 | 에러, 경고, 스키마 변경 이벤트 |

### 시스템 테이블을 활용한 커스텀 모니터링

```sql
-- 파이프라인 실행 이력 조회 (시스템 테이블 활용)
SELECT
    pipeline_id,
    update_id,
    state,
    creation_time,
    TIMESTAMPDIFF(MINUTE, creation_time, end_time) AS duration_minutes
FROM system.lakeflow.pipeline_event_log
WHERE pipeline_id = '<pipeline-id>'
ORDER BY creation_time DESC
LIMIT 20;

-- 일별 수집 건수 추이
SELECT
    DATE(_ingested_at) AS ingest_date,
    COUNT(*) AS records_ingested,
    COUNT(DISTINCT _source_table) AS tables_updated
FROM analytics.bronze.orders
GROUP BY DATE(_ingested_at)
ORDER BY ingest_date DESC
LIMIT 30;
```

### 알림 설정

Lakeflow Jobs와 연동하면 다양한 조건에서 알림을 받을 수 있습니다.

| 알림 조건 | 설정 방법 | 알림 채널 |
|----------|---------|---------|
| **파이프라인 실패**| Jobs UI → Notifications | 이메일, Slack, PagerDuty |
| ** 수집 지연 임계치 초과**| SQL Alert (수동 설정) | 이메일, Slack |
| ** 데이터 건수 이상**| SQL Alert (수동 설정) | 이메일, Slack |
| **SLA 위반**| Jobs SLA 설정 | 이메일, Slack |

```sql
-- SQL Alert 예시: 수집 지연이 30분 이상일 때 알림
-- (Databricks SQL Alert로 등록)
SELECT
    MAX(TIMESTAMPDIFF(MINUTE, _commit_timestamp, _ingested_at)) AS max_latency_min
FROM analytics.bronze.orders
WHERE _ingested_at >= CURRENT_TIMESTAMP - INTERVAL 1 HOUR
HAVING max_latency_min > 30;
```

---

## 스케줄링 옵션

### 실행 모드 비교

| 실행 모드 | 동작 방식 | 지연시간 | 비용 | 적합한 경우 |
|----------|---------|:-------:|:----:|-----------|
| **Continuous**| 항상 실행, 변경 즉시 수집 | 초~분 | 높음 | 실시간 대시보드, 실시간 분석 |
| **Triggered**| 수동 또는 스케줄로 실행 | 분~시간 | 낮음 | 일일 배치, 주기적 동기화 |

### Triggered 모드 스케줄 설정

Lakeflow Jobs에서 파이프라인의 실행 스케줄을 설정할 수 있습니다:

| 스케줄 유형 | 설정 예시 | 사용 사례 |
|-----------|---------|---------|
| ** 크론 표현식** | `0 */2 * * *` (2시간마다) | 정기적 수집 |
| **파일 도착 트리거**| S3 경로 감시 | 파일 기반 연동 |
| ** 수동 실행** | API / UI 버튼 | 임시 수집, 테스트 |

```python
# Lakeflow Jobs로 파이프라인 스케줄링 (API 예시)
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 매 2시간마다 파이프라인 트리거
w.jobs.create(
    name="ecommerce_ingestion_schedule",
    tasks=[{
        "task_key": "trigger_ingestion",
        "pipeline_task": {
            "pipeline_id": "<pipeline-id>",
            "full_refresh": False
        }
    }],
    schedule={
        "quartz_cron_expression": "0 0 */2 * * ?",
        "timezone_id": "Asia/Seoul"
    }
)
```

---

## 고급 설정

### 병렬 수집 설정

하나의 파이프라인에서 여러 테이블을 동시에 수집할 때, 병렬도를 조정할 수 있습니다.

| 설정 | 기본값 | 설명 |
|------|-------|------|
| **Max concurrent tables**| 자동 | 동시 수집 테이블 수 (서버리스에서 자동 조정) |
| **Batch size**| 자동 | CDC 이벤트 배치 크기 |
| **Checkpoint interval**| 자동 | 체크포인트 저장 주기 |

### 초기 스냅샷 최적화

대규모 테이블의 초기 스냅샷 시간을 줄이기 위한 전략입니다.

| 전략 | 설명 | 효과 |
|------|------|------|
| ** 오프피크 시간 실행**| 소스 DB 부하가 적은 시간대에 시작 | 소스 영향 최소화 |
| ** 테이블 분할 수집**| 큰 테이블을 별도 파이프라인으로 분리 | 실패 시 영향 범위 축소 |
| ** 필요한 테이블만 선택**| 불필요한 테이블 제외 | 전체 수집 시간 단축 |

---

## 운영 모범 사례

### 파이프라인 구성 모범 사례

| 영역 | 모범 사례 | 이유 |
|------|---------|------|
| **Connection**| 소스 시스템별 1개 Connection 생성 | 관리 용이성 |
| ** 파이프라인 분리**| 업무 도메인별 파이프라인 분리 | 장애 격리, 독립적 운영 |
| ** 네이밍 규칙**| `<source>_<domain>_ingestion` 패턴 | 일관성, 검색 용이 |
| ** 대상 스키마**| `bronze` 스키마에 원시 데이터 적재 | Medallion 아키텍처 준수 |
| ** 알림 설정**| 모든 프로덕션 파이프라인에 알림 설정 | 장애 조기 감지 |
| ** 문서화**| 파이프라인 설명(description) 필수 작성 | 팀 협업 시 이해도 향상 |

### 보안 모범 사례

| 영역 | 모범 사례 | 설명 |
|------|---------|------|
| ** 최소 권한**| 수집에 필요한 최소 권한만 부여 | SELECT + REPLICATION 권한만 |
| ** 전용 계정**| Databricks 전용 DB 계정 사용 | 감사 추적, 권한 분리 |
| ** 네트워크 격리**| PrivateLink 또는 VPC Peering 사용 | 데이터 전송 보안 |
| **Secret 관리**| 모든 자격증명을 Secret Scope에 저장 | 평문 노출 방지 |
| ** 접근 제어**| Unity Catalog 권한으로 테이블 접근 관리 | 역할 기반 접근 제어 |

### 비용 최적화 모범 사례

| 전략 | 상세 내용 | 예상 절감 효과 |
|------|---------|:-------------:|
| ** 실행 모드 최적화**| 실시간 불필요 시 Triggered 모드 사용 | 30~70% |
| ** 불필요 테이블 제외**| 분석에 사용되지 않는 테이블 수집 중지 | 10~50% |
| ** 보관 정책**| Bronze 테이블에 보관 기간 설정 | 스토리지 비용 절감 |
| ** 모니터링**| 시스템 테이블로 비용 추적 | 이상 비용 조기 감지 |

```sql
-- Bronze 테이블에 보관 기간 설정 (예: 90일)
ALTER TABLE analytics.bronze.events
SET TBLPROPERTIES (
    'delta.deletedFileRetentionDuration' = 'interval 90 days',
    'delta.logRetentionDuration' = 'interval 90 days'
);

-- 오래된 데이터 정리 (VACUUM)
VACUUM analytics.bronze.events RETAIN 90 HOURS;
```

---

## 정리

수집 파이프라인의 핵심 구성 요소를 정리하면 다음과 같습니다:

- **Connection**: 소스 접속 정보를 Unity Catalog에서 안전하게 관리합니다
- **테이블 선택**: 필요한 테이블만 선택하여 비용을 최적화합니다
- **스키마 매핑**: 소스 타입이 Delta 호환 타입으로 자동 변환됩니다
- **모니터링**: Pipeline UI, 시스템 테이블, SQL Alert로 파이프라인을 관찰합니다
- **스케줄링**: Continuous(실시간) 또는 Triggered(배치) 모드를 선택합니다
- **보안**: Secret, 최소 권한, 네트워크 격리를 적용합니다

---

## 참고 링크

- [Databricks: Configure Lakeflow Connect](https://docs.databricks.com/aws/en/lakeflow-connect/)
- [Databricks: Unity Catalog Connections](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-connection.html)
- [Databricks: Delta Lake Table Properties](https://docs.databricks.com/aws/en/delta/table-properties.html)
- [Databricks: Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering.html)
- [Databricks: System Tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/)
