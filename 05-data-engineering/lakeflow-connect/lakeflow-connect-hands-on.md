# Lakeflow Connect 실습

## 시나리오

운영 중인 MySQL 데이터베이스의 **고객(customers) 테이블**과 **주문(orders) 테이블**을 Databricks 레이크하우스로 **실시간 CDC 수집** 합니다. 수집된 Bronze 데이터를 SDP 파이프라인으로 변환하여 Silver 계층까지 구성하는 전체 흐름을 실습합니다.

| 계층 | 구성 요소 | 설명 |
|------|-----------|------|
| **소스 MySQL**| ecommerce.customers | 고객 테이블 |
|  | ecommerce.orders | 주문 테이블 |
| **Lakeflow Connect**| Connection: mysql_ecommerce | MySQL 연결 정보 |
|  | Ingestion Pipeline: ecommerce_ingestion | 수집 파이프라인 |
| **Databricks (Bronze)**| analytics.bronze.customers | 수집된 고객 데이터 |
|  | analytics.bronze.orders | 수집된 주문 데이터 |
| **Databricks (Silver)**| analytics.silver.customers | 정제된 고객 데이터 |
|  | analytics.silver.orders | 정제된 주문 데이터 |

---

## 사전 준비

### 1. 소스 MySQL 데이터베이스 설정

Lakeflow Connect의 MySQL CDC는 **Binlog(Binary Log)** 를 사용합니다. 소스 MySQL에서 다음 설정이 필요합니다.

```sql
-- MySQL 서버 설정 확인 (my.cnf 또는 파라미터 그룹)
-- 아래 설정이 활성화되어 있어야 합니다
SHOW VARIABLES LIKE 'log_bin';           -- ON 이어야 함
SHOW VARIABLES LIKE 'binlog_format';     -- ROW 이어야 함
SHOW VARIABLES LIKE 'binlog_row_image';  -- FULL 이어야 함
```

> ⚠️ **AWS RDS 사용 시**: RDS 파라미터 그룹에서 `binlog_format=ROW`를 설정해야 합니다. Aurora MySQL은 기본적으로 Binlog가 비활성화되어 있으므로 `binlog_format` 파라미터를 활성화해야 합니다.

**Databricks 전용 계정 생성:**

```sql
-- CDC 수집용 계정 생성 (최소 권한 원칙)
CREATE USER 'databricks_cdc'@'%' IDENTIFIED BY '<strong-password>';

-- 필수 권한 부여
GRANT SELECT ON ecommerce.* TO 'databricks_cdc'@'%';
GRANT REPLICATION SLAVE ON *.* TO 'databricks_cdc'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'databricks_cdc'@'%';
FLUSH PRIVILEGES;
```

### 2. 네트워크 연결 설정

Databricks에서 소스 MySQL에 접속할 수 있어야 합니다.

| 네트워크 구성 | 방법 | 보안 수준 |
|-------------|------|:--------:|
| **퍼블릭 인터넷**| MySQL의 퍼블릭 IP + 보안 그룹으로 Databricks IP 허용 | 낮음 |
| **VPC Peering**| Databricks VPC와 소스 DB VPC 간 피어링 | 중간 |
| **AWS PrivateLink**| Private Endpoint를 통한 프라이빗 연결 | 높음 |

> 💡 **프로덕션 환경에서는 PrivateLink 또는 VPC Peering** 을 권장합니다. 퍼블릭 인터넷 접속은 개발/테스트 환경에서만 사용하시기 바랍니다.

### 3. Unity Catalog 설정

대상 카탈로그와 스키마가 미리 생성되어 있어야 합니다.

```sql
-- 대상 카탈로그 및 스키마 생성
CREATE CATALOG IF NOT EXISTS analytics;
CREATE SCHEMA IF NOT EXISTS analytics.bronze;
CREATE SCHEMA IF NOT EXISTS analytics.silver;

-- 파이프라인 실행 서비스 프린시펄에 권한 부여
GRANT USE CATALOG ON CATALOG analytics TO `lakeflow-connect-service`;
GRANT USE SCHEMA ON SCHEMA analytics.bronze TO `lakeflow-connect-service`;
GRANT CREATE TABLE ON SCHEMA analytics.bronze TO `lakeflow-connect-service`;
```

---

## Step 1: Connection 생성

Unity Catalog Connection은 소스 시스템의 접속 정보를 안전하게 관리하는 객체입니다.

### 방법 A: SQL로 Connection 생성

```sql
-- MySQL Connection 생성
CREATE CONNECTION mysql_ecommerce
TYPE MYSQL
OPTIONS (
    host = 'mysql-prod.company.com',
    port = '3306',
    user = 'databricks_cdc',
    password = secret('db-secrets', 'mysql-ecommerce-pwd')
);
```

> 💡 **Secret 사용**: 비밀번호를 평문으로 입력하는 대신 `secret()` 함수를 사용하면 Databricks Secret Scope에 저장된 값을 안전하게 참조할 수 있습니다. 프로덕션 환경에서는 반드시 Secret을 사용하시기 바랍니다.

### 방법 B: UI로 Connection 생성

1. **Catalog Explorer** 열기
2. 좌측 메뉴에서 **External Data**→ **Connections** 클릭
3. **Create Connection** 클릭
4. 다음 정보 입력:

| 필드 | 값 | 설명 |
|------|---|------|
| Connection name | `mysql_ecommerce` | 연결 이름 (영문, 밑줄 가능) |
| Connection type | `MySQL` | 소스 시스템 유형 |
| Host | `mysql-prod.company.com` | MySQL 서버 주소 |
| Port | `3306` | MySQL 포트 |
| User | `databricks_cdc` | 접속 계정 |
| Password | `(비밀번호)` | 또는 Secret 참조 |

5. **Test Connection** 클릭하여 연결 확인
6. **Create** 클릭

### 연결 테스트 및 확인

```sql
-- 생성된 Connection 확인
SHOW CONNECTIONS;

-- Connection을 통해 소스 테이블 목록 확인
SHOW TABLES IN mysql_ecommerce.ecommerce;
```

---

## Step 2: Ingestion Pipeline 생성 및 설정

### 방법 A: UI로 파이프라인 생성 (권장)

1. 좌측 메뉴에서 **Pipelines** 클릭
2. **Create Pipeline**→ **ETL pipeline** 선택
3. 파이프라인 설정:

| 설정 항목 | 값 | 설명 |
|---------|---|------|
| Pipeline name | `ecommerce_ingestion` | 파이프라인 이름 |
| Pipeline type | `Ingestion (Lakeflow Connect)` | 수집 파이프라인 |
| Source | `MySQL` | 소스 유형 |
| Connection | `mysql_ecommerce` | 앞서 생성한 Connection |

4. **테이블 선택**:
   - Source schema: `ecommerce`
   - Tables: `customers`, `orders` 선택 (또는 전체 스키마 선택)

5. **대상 설정**:
   - Target catalog: `analytics`
   - Target schema: `bronze`

6. **실행 모드 설정**:
   - Mode: `Continuous` (실시간 CDC) 또는 `Triggered` (주기적)

7. **Create** 클릭

### 방법 B: Python API로 파이프라인 생성

```python
import dlt
from dlt.sources import mysql

# Lakeflow Connect MySQL 수집 정의
@dlt.table(
    name="bronze_customers",
    comment="MySQL ecommerce.customers 실시간 CDC 수집"
)
def ingest_customers():
    return mysql.read_changefeed(
        connection="mysql_ecommerce",
        source_schema="ecommerce",
        source_table="customers"
    )

@dlt.table(
    name="bronze_orders",
    comment="MySQL ecommerce.orders 실시간 CDC 수집"
)
def ingest_orders():
    return mysql.read_changefeed(
        connection="mysql_ecommerce",
        source_schema="ecommerce",
        source_table="orders"
    )
```

---

## Step 3: 파이프라인 실행 및 모니터링

### 파이프라인 시작

1. Pipeline UI에서 **Start** 버튼 클릭
2. 초기 스냅샷이 시작됩니다 (소스 테이블 크기에 따라 시간 소요)
3. 스냅샷 완료 후 자동으로 CDC 모드로 전환됩니다

### 모니터링 대시보드 확인

Pipeline UI에서 다음 메트릭을 실시간으로 확인할 수 있습니다:

| 메트릭 | 설명 | 정상 기준 |
|-------|------|---------|
| **Ingestion Latency**| 소스 변경 → 대상 반영까지 지연 시간 | 수초~수분 |
| **Records Processed**| 처리된 레코드 수 | 지속적으로 증가 |
| **Pipeline Status**| 파이프라인 상태 | `RUNNING` 또는 `IDLE` |
| **Error Count**| 에러 발생 건수 | 0 |
| **Last Update Time** | 마지막 업데이트 시간 | 최근 시간 |

### SQL로 수집 결과 확인

```sql
-- 수집된 고객 데이터 확인
SELECT * FROM analytics.bronze.customers LIMIT 10;

-- 수집된 주문 데이터 확인
SELECT COUNT(*) AS total_orders,
       MIN(_ingested_at) AS first_ingested,
       MAX(_ingested_at) AS last_ingested
FROM analytics.bronze.orders;

-- 시스템 메타데이터 컬럼 확인
-- Lakeflow Connect가 자동으로 추가하는 메타데이터
SELECT _ingested_at,        -- 수집 시간
       _change_type,        -- INSERT/UPDATE/DELETE
       _commit_timestamp,   -- 소스 변경 시간
       customer_id, name, email
FROM analytics.bronze.customers
ORDER BY _ingested_at DESC
LIMIT 20;
```

---

## Step 4: CDC 변경사항 확인

소스 MySQL에서 데이터를 변경하고, Databricks에서 변경 사항이 반영되는지 확인합니다.

### 소스에서 변경 실행

```sql
-- [MySQL에서 실행] 새 고객 추가
INSERT INTO ecommerce.customers (customer_id, name, email, city)
VALUES (1001, '김테스트', 'test@example.com', 'Seoul');

-- [MySQL에서 실행] 기존 고객 정보 수정
UPDATE ecommerce.customers
SET city = 'Busan'
WHERE customer_id = 1001;

-- [MySQL에서 실행] 새 주문 추가
INSERT INTO ecommerce.orders (order_id, customer_id, amount, order_date)
VALUES (5001, 1001, 50000, NOW());
```

### Databricks에서 변경 확인

```sql
-- CDC 변경사항 확인 (수초~수분 후)
SELECT _change_type, _commit_timestamp, customer_id, name, city
FROM analytics.bronze.customers
WHERE customer_id = 1001
ORDER BY _commit_timestamp DESC;

-- 결과 예시:
-- 예상 결과:
-- | _change_type | _commit_timestamp  | customer_id | name   | city  |
-- |-------------|-------------------|-------------|--------|-------|
-- | UPDATE      | 2025-01-15 10:01  | 1001        | 김테스트 | Busan |
-- | INSERT      | 2025-01-15 10:00  | 1001        | 김테스트 | Seoul |

-- 주문 테이블 변경 확인
SELECT * FROM analytics.bronze.orders
WHERE order_id = 5001;
```

> 💡 **CDC 이력 보존**: Bronze 테이블에는 모든 변경 이력이 보존됩니다. INSERT, UPDATE, DELETE 모두 별도의 행으로 기록되므로, 특정 시점의 데이터 상태를 재현할 수 있습니다.

---

## Step 5: 스키마 변경 시나리오

운영 DB에서 스키마가 변경되었을 때 Lakeflow Connect가 어떻게 대응하는지 확인합니다.

### 시나리오: 소스에서 컬럼 추가

```sql
-- [MySQL에서 실행] 고객 테이블에 새 컬럼 추가
ALTER TABLE ecommerce.customers ADD COLUMN phone VARCHAR(20);

-- [MySQL에서 실행] 새 컬럼에 데이터 입력
UPDATE ecommerce.customers
SET phone = '010-1234-5678'
WHERE customer_id = 1001;
```

### Databricks에서 스키마 변경 확인

```sql
-- 대상 테이블에 자동으로 phone 컬럼이 추가되었는지 확인
DESCRIBE analytics.bronze.customers;

-- 새 컬럼의 데이터 확인
SELECT customer_id, name, phone
FROM analytics.bronze.customers
WHERE phone IS NOT NULL;

-- 기존 행에서 phone은 NULL (스키마 추가 이전 데이터)
SELECT customer_id, name, phone
FROM analytics.bronze.customers
WHERE phone IS NULL
LIMIT 5;
```

> 💡 **스키마 진화 자동 처리**: Lakeflow Connect는 소스에서 컬럼이 추가되면 대상 Delta 테이블에도 자동으로 컬럼을 추가합니다. 기존 행의 새 컬럼 값은 NULL로 채워집니다.

---

## Step 6: SDP 파이프라인으로 Silver 변환 연결

수집된 Bronze 데이터를 **SDP 파이프라인으로 정제하여 Silver 계층** 을 구축합니다.

```sql
-- SDP 파이프라인에서 Silver 변환 정의

-- 고객 테이블: CDC 이력에서 최신 상태만 유지 (SCD Type 1)
CREATE OR REFRESH STREAMING TABLE analytics.silver.customers;

APPLY CHANGES INTO analytics.silver.customers
FROM STREAM(analytics.bronze.customers)
KEYS (customer_id)
SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 1;

-- 주문 테이블: 데이터 품질 검증 + 정제
CREATE OR REFRESH STREAMING TABLE analytics.silver.orders (
    CONSTRAINT valid_amount EXPECT (amount > 0) ON VIOLATION DROP ROW,
    CONSTRAINT valid_customer EXPECT (customer_id IS NOT NULL) ON VIOLATION DROP ROW
)
AS
SELECT
    order_id,
    customer_id,
    amount,
    order_date,
    _ingested_at AS ingested_at
FROM STREAM(analytics.bronze.orders)
WHERE _change_type != 'DELETE';
```

> 💡 **SCD Type 1 vs Type 2**: SCD Type 1은 변경 시 기존 데이터를 덮어씁니다(항상 최신 상태만 유지). SCD Type 2는 변경 이력을 모두 보존합니다. 자세한 내용은 [SDP: CDC와 SCD](../spark-declarative-pipelines/cdc-and-scd.md) 문서를 참고하시기 바랍니다.

---

## 트러블슈팅 가이드

### 자주 발생하는 오류와 해결 방법

| 오류 메시지 | 원인 | 해결 방법 |
|-----------|------|----------|
| `Connection failed: Access denied` | MySQL 계정 권한 부족 | `GRANT` 문으로 필요한 권한 부여 |
| `Connection timed out` | 네트워크 연결 불가 | 보안 그룹, VPC Peering, PrivateLink 설정 확인 |
| `Binlog not enabled` | MySQL Binlog 비활성화 | `log_bin=ON`, `binlog_format=ROW` 설정 |
| `Schema evolution error` | 비호환 스키마 변경 | 대상 테이블 스키마 수동 조정 후 파이프라인 재시작 |
| `Secret not found` | Secret Scope 미설정 | Databricks CLI로 Secret 생성 확인 |
| `Insufficient permissions on target` | 대상 스키마 권한 부족 | Unity Catalog 권한 설정 확인 |

### 파이프라인 재시작 방법

```python
# 파이프라인 상태 확인 (Databricks CLI)
# databricks pipelines get --pipeline-id <pipeline-id>

# 파이프라인이 FAILED 상태인 경우:
# 1. Pipeline UI에서 에러 메시지 확인
# 2. 원인 해결 (위의 표 참고)
# 3. Pipeline UI에서 "Start" 클릭하여 재시작
#    - "Full Refresh": 초기 스냅샷부터 다시 시작
#    - "Start": 마지막 체크포인트에서 이어서 시작
```

### 성능 최적화 팁

| 팁 | 설명 |
|----|------|
| **테이블 필터링**| 필요한 테이블만 선택하여 불필요한 수집 방지 |
| **연속 vs 트리거**| 실시간이 필요 없으면 트리거 모드로 비용 절감 |
| **대규모 초기 스냅샷**| 테이블이 매우 크면 오프피크 시간에 시작 |
| **병렬 수집**| 여러 테이블을 하나의 파이프라인에서 동시 수집 |

---

## 클린업

실습이 끝나면 생성한 리소스를 정리합니다.

```sql
-- 1. 파이프라인 중지 (UI에서 Stop 클릭 또는 API 사용)

-- 2. 수집된 테이블 삭제
DROP TABLE IF EXISTS analytics.bronze.customers;
DROP TABLE IF EXISTS analytics.bronze.orders;
DROP TABLE IF EXISTS analytics.silver.customers;
DROP TABLE IF EXISTS analytics.silver.orders;

-- 3. Connection 삭제
DROP CONNECTION IF EXISTS mysql_ecommerce;

-- 4. 스키마/카탈로그 삭제 (필요 시)
-- DROP SCHEMA IF EXISTS analytics.bronze CASCADE;
-- DROP SCHEMA IF EXISTS analytics.silver CASCADE;
-- DROP CATALOG IF EXISTS analytics CASCADE;
```

> ⚠️ **주의**: `CASCADE` 옵션은 스키마/카탈로그 내의 모든 테이블을 삭제합니다. 프로덕션 환경에서는 절대 사용하지 마시기 바랍니다. 실습용 리소스만 삭제하시기 바랍니다.

---

## 정리

이 실습에서 다음을 수행하였습니다:

1. **MySQL Connection 생성**: Unity Catalog에 소스 접속 정보를 안전하게 등록
2. **Ingestion Pipeline 생성**: 코드 없이 UI에서 CDC 수집 파이프라인 구성
3. **실시간 CDC 확인**: 소스 변경사항이 수초 내에 Delta 테이블에 반영됨을 검증
4. **스키마 진화 확인**: 소스에서 컬럼 추가 시 대상 테이블에 자동 반영됨을 검증
5. **SDP 연동**: Bronze 데이터를 Silver로 변환하는 후처리 파이프라인 구성

---

## 참고 링크

- [Databricks: Lakeflow Connect Tutorial](https://docs.databricks.com/aws/en/lakeflow-connect/)
- [Databricks: Create a Connection](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-connection.html)
- [Databricks: MySQL CDC Requirements](https://docs.databricks.com/aws/en/lakeflow-connect/)
- [Databricks: SDP APPLY CHANGES](https://docs.databricks.com/aws/en/delta-live-tables/cdc.html)
