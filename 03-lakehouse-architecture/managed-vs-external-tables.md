# Managed Table vs External Table vs Foreign Table

## 왜 테이블 유형을 이해해야 하나요?

Databricks에서 테이블을 생성할 때, 데이터가 **어디에 저장되고**, **누가 관리하는지**에 따라 테이블 유형이 달라집니다. 잘못된 유형을 선택하면 데이터를 의도치 않게 삭제하거나, 반대로 불필요한 데이터가 영원히 남을 수 있습니다.

이 문서에서는 세 가지 테이블 유형의 차이점과 선택 기준을 자세히 살펴보겠습니다.

---

## 테이블 유형 한눈에 비교

| 비교 항목 | Managed Table | External Table | Foreign Table |
|-----------|--------------|----------------|---------------|
| **데이터 저장 위치** | Unity Catalog가 관리하는 스토리지 | 사용자가 지정한 외부 경로 | 외부 데이터 소스(MySQL, PostgreSQL 등) |
| **메타데이터 관리** | Unity Catalog | Unity Catalog | Unity Catalog (메타데이터만) |
| **DROP TABLE 시 데이터** | 함께 삭제됨 | 메타데이터만 삭제, 데이터 유지 | 메타데이터만 삭제, 원본 유지 |
| **Predictive Optimization** | ✅ 지원 | ❌ 미지원 | ❌ 미지원 |
| **포맷** | Delta Lake (기본) | Delta, Parquet, CSV, JSON 등 | 원본 시스템의 포맷 |
| **주요 용도** | 일반적인 데이터 관리 | 기존 데이터 레이크 연동, 멀티 플랫폼 공유 | 외부 DBMS 데이터 조회 |

---

## Managed Table (관리형 테이블)

### 개념

> 💡 **Managed Table(관리형 테이블)** 은 Unity Catalog가 데이터의 **저장 위치와 생명주기를 모두 관리**하는 테이블입니다. 테이블을 삭제(DROP)하면 데이터 파일까지 함께 삭제됩니다.

마치 **회사 소유 창고**에 물건을 보관하는 것과 같습니다. 창고를 해체하면 안에 있는 물건도 모두 처분됩니다.

### 생성 방법

```sql
-- LOCATION을 지정하지 않으면 자동으로 Managed Table이 됩니다
CREATE TABLE catalog.schema.customers (
    customer_id BIGINT,
    name STRING,
    email STRING,
    signup_date DATE
);

-- USING DELTA를 명시할 수도 있지만, 기본값이 Delta이므로 생략 가능합니다
```

### 데이터 저장 위치

Managed Table의 데이터는 카탈로그 또는 스키마에 설정된 **Managed Storage Location**에 자동 저장됩니다.

```sql
-- 스키마의 관리형 스토리지 위치 확인
DESCRIBE SCHEMA EXTENDED catalog.my_schema;
```

### 특징

- **Predictive Optimization** 자동 적용 가능 (OPTIMIZE, VACUUM 자동화)
- Unity Catalog의 거버넌스 기능을 **100% 활용** 가능
- 데이터 생명주기를 Unity Catalog가 완전히 관리
- 신규 프로젝트에서 **권장되는 기본 유형**

---

## External Table (외부 테이블)

### 개념

> 💡 **External Table(외부 테이블)** 은 사용자가 지정한 클라우드 스토리지 경로에 데이터가 저장되는 테이블입니다. Unity Catalog는 메타데이터만 관리하며, 테이블을 삭제해도 **실제 데이터 파일은 그대로 남습니다**.

마치 **외부 임대 창고**에 물건을 보관하는 것과 같습니다. Unity Catalog 등록을 해제해도 창고의 물건은 그대로입니다.

### 생성 방법

```sql
-- LOCATION을 지정하면 External Table이 됩니다
CREATE TABLE catalog.schema.sales_external (
    sale_id BIGINT,
    product STRING,
    amount DECIMAL(10, 2),
    sale_date DATE
)
LOCATION 's3://my-bucket/data/sales/';

-- 기존 데이터가 있는 경로를 가리킬 수도 있습니다
CREATE TABLE catalog.schema.existing_data
LOCATION 's3://my-bucket/existing-delta-table/';
```

### External Location 설정

외부 테이블을 생성하려면, 해당 스토리지 경로가 Unity Catalog의 **External Location**으로 등록되어 있어야 합니다.

```sql
-- External Location 생성 (관리자가 수행)
CREATE EXTERNAL LOCATION my_external_location
URL 's3://my-bucket/data/'
WITH (STORAGE CREDENTIAL my_credential);
```

### 특징

- 테이블 삭제 시 **데이터가 보존**되므로 안전합니다
- 여러 플랫폼에서 **동일한 데이터에 접근**해야 할 때 유용합니다
- Delta 외에 Parquet, CSV, JSON 등 다양한 포맷을 지원합니다
- Predictive Optimization이 지원되지 않으므로 OPTIMIZE/VACUUM을 **수동으로 관리**해야 합니다

---

## Foreign Table (외래 테이블)

### 개념

> 💡 **Foreign Table(외래 테이블)** 은 **Lakehouse Federation**을 통해 외부 데이터베이스(MySQL, PostgreSQL, Snowflake 등)의 테이블을 Databricks에서 직접 조회할 수 있게 하는 테이블입니다. 데이터를 복사하지 않고 **원본 시스템에 쿼리를 전달**합니다.

> 💡 **Lakehouse Federation이란?** 외부 데이터 소스에 저장된 데이터를 Databricks로 복사하지 않고도, Unity Catalog를 통해 거버넌스를 유지하면서 직접 조회할 수 있는 기능입니다.

### 생성 방법

```sql
-- 1. 먼저 Connection 생성 (관리자가 수행)
CREATE CONNECTION my_mysql_conn
TYPE MYSQL
OPTIONS (
    host 'mysql-server.example.com',
    port '3306',
    user 'readonly_user',
    password secret('scope', 'mysql_password')
);

-- 2. Foreign Catalog 생성
CREATE FOREIGN CATALOG mysql_catalog
USING CONNECTION my_mysql_conn;

-- 3. Foreign Table 조회 (자동으로 등록됨)
SELECT * FROM mysql_catalog.my_database.orders;
```

### 지원하는 외부 소스

| 데이터 소스 | 연결 유형 |
|------------|----------|
| MySQL | `MYSQL` |
| PostgreSQL | `POSTGRESQL` |
| SQL Server | `SQLSERVER` |
| Oracle | `ORACLE` |
| Snowflake | `SNOWFLAKE` |
| Google BigQuery | `BIGQUERY` |
| Amazon Redshift | `REDSHIFT` |

### 특징

- 데이터 복사 없이 **실시간으로 외부 데이터를 조회**합니다
- Unity Catalog의 **접근 제어(GRANT/REVOKE)** 가 그대로 적용됩니다
- 대량 데이터 조회 시에는 네트워크 지연이 발생할 수 있어, **필요한 데이터만 선별적으로 조회**하는 것이 좋습니다
- 읽기 전용(SELECT)만 가능하며, INSERT/UPDATE/DELETE는 지원되지 않습니다

---

## 선택 가이드

| 상황 | 권장 테이블 유형 |
|------|-----------------|
| 새 프로젝트에서 테이블 생성 | **Managed Table** |
| Predictive Optimization 활용 | **Managed Table** |
| 기존 S3/ADLS 데이터를 등록하고 싶을 때 | **External Table** |
| Databricks 외 플랫폼에서도 같은 데이터 접근 | **External Table** |
| 테이블 삭제 시 데이터 보존 필요 | **External Table** |
| 외부 DBMS 데이터를 실시간 조회 | **Foreign Table** |
| ETL 없이 외부 소스 탐색/분석 | **Foreign Table** |

> 💡 **핵심 메시지**: 특별한 이유가 없다면 **Managed Table**을 기본으로 사용하세요. Databricks가 스토리지 관리, 최적화를 자동으로 처리해주므로 운영 부담이 가장 적습니다. 외부 시스템과의 연동이 필요한 경우에만 External Table이나 Foreign Table을 고려하시기 바랍니다.

---

## 테이블 유형 확인 방법

```sql
-- 테이블의 상세 정보 확인
DESCRIBE DETAIL catalog.schema.my_table;

-- 결과에서 'is_managed_location' 컬럼으로 Managed/External 구분 가능
-- 또는 SHOW CREATE TABLE로 LOCATION 유무 확인
SHOW CREATE TABLE catalog.schema.my_table;
```

---

## 현업 사례: External Table 남발로 거버넌스를 잃어버린 팀

> 🔥 **실제 프로젝트에서 흔한 문제입니다.**

현업에서는 "데이터를 잃으면 안 되니까"라는 이유로 **모든 테이블을 External Table로 생성하는** 팀을 자주 만납니다. 처음에는 안전해 보이지만, 6개월~1년이 지나면 다음과 같은 상황이 벌어집니다.

### 전형적인 문제 시나리오

| 단계 | 상황 | 결과 |
|------|------|------|
| **1개월 차** | "안전하니까 전부 External로!" | 문제 없음 |
| **3개월 차** | 팀원 10명이 각자 External Table 생성 | S3 버킷에 수백 개 경로가 산재 |
| **6개월 차** | 누가 어떤 경로를 쓰는지 파악 불가 | 스토리지 비용 청구서에 놀람 |
| **1년 차** | 테이블을 DROP해도 데이터가 남아 있음 | **좀비 데이터**가 TB 단위로 누적 |

```
실제 비용 사례:
- External Table 500개 운영
- 각 테이블에 VACUUM을 수동으로 안 돌림
- 오래된 파일 버전이 계속 누적
- 월 스토리지 비용: $3,200 → Managed Table로 전환 후 $1,100
- 60% 이상 절감 (Predictive Optimization이 자동으로 VACUUM 수행)
```

### 가장 치명적인 실수: "External Location 없이 직접 경로 지정"

초기에 Unity Catalog를 도입하면서 External Location을 체계적으로 설정하지 않고, 팀원들이 각자 `s3://my-bucket/team-a/project-x/...` 같은 경로를 자유롭게 지정하면 다음 문제가 발생합니다.

- **경로 충돌**: 두 팀이 같은 경로를 가리키는 서로 다른 External Table을 생성
- **권한 사각지대**: Unity Catalog 밖에서 직접 S3에 접근하면 거버넌스가 우회됨
- **리니지 단절**: 데이터 파일이 Unity Catalog 관리 영역 밖에 있어 추적 불가

> 💡 **현업 교훈**: External Table을 쓸 때는 반드시 **External Location을 중앙에서 관리**하고, 팀원이 임의의 경로에 테이블을 만들 수 없도록 권한을 제한해야 합니다.

---

## External Table에서 Managed Table로의 마이그레이션 전략

기존에 External Table로 운영하던 환경을 Managed Table로 전환하려면, 단계적으로 접근해야 합니다. 한 번에 전환하면 데이터 유실이나 파이프라인 장애가 발생할 수 있습니다.

### 마이그레이션 4단계 프로세스

```sql
-- ===== 1단계: 현황 파악 =====
-- External Table 목록과 데이터 크기 확인
SELECT
    table_catalog,
    table_schema,
    table_name,
    table_type,
    data_source_format,
    storage_path
FROM system.information_schema.tables
WHERE table_type = 'EXTERNAL'
ORDER BY table_catalog, table_schema;

-- ===== 2단계: Managed Table로 데이터 복사 (CTAS) =====
-- 가장 안전한 방법: CREATE TABLE AS SELECT
CREATE TABLE catalog.schema.orders_managed AS
SELECT * FROM catalog.schema.orders_external;

-- ===== 3단계: 다운스트림 파이프라인을 새 테이블로 전환 =====
-- 뷰를 사용하면 전환 기간 동안 하위 호환성을 유지할 수 있습니다
CREATE OR REPLACE VIEW catalog.schema.orders AS
SELECT * FROM catalog.schema.orders_managed;  -- 포인터만 변경

-- ===== 4단계: 안정성 확인 후 External Table 및 원본 데이터 정리 =====
-- 최소 1~2주 병행 운영 후 정리
DROP TABLE catalog.schema.orders_external;  -- 메타데이터만 삭제됨
-- S3/ADLS의 원본 데이터는 별도로 삭제해야 합니다
```

### 마이그레이션 시 주의사항

| 항목 | 주의점 |
|------|--------|
| **대용량 테이블** | TB 단위 테이블은 CTAS가 오래 걸립니다. `DEEP CLONE`을 고려하세요 |
| **스트리밍 테이블** | 실시간 파이프라인이 연결된 테이블은 다운타임 계획이 필요합니다 |
| **외부 연동** | Databricks 외 플랫폼에서도 접근하는 테이블은 External로 유지해야 합니다 |
| **히스토리 보존** | CTAS는 Delta 히스토리를 초기화합니다. `DEEP CLONE`을 쓰면 히스토리도 복사됩니다 |

```sql
-- DEEP CLONE: 히스토리를 포함한 전체 복사 (대용량에 적합)
CREATE TABLE catalog.schema.orders_managed
DEEP CLONE catalog.schema.orders_external;
```

> 💡 **현업 팁**: 마이그레이션은 **중요도가 낮은 테이블부터** 시작하세요. dev/staging 환경에서 먼저 검증하고, 프로덕션은 마지막에 진행합니다. 많은 팀이 "한 번에 전부 전환"을 시도하다가 장애를 겪습니다.

---

## Foreign Table(Lakehouse Federation)의 실전 활용과 한계

### Foreign Table이 빛나는 순간

현업에서 Foreign Table은 다음 시나리오에서 매우 유용합니다.

| 시나리오 | 설명 | 효과 |
|---------|------|------|
| **탐색적 분석** | "MySQL에 있는 데이터를 한번 살펴보고 싶어요" | ETL 구축 없이 즉시 조회 |
| **ETL 타당성 검증** | 데이터를 가져올 가치가 있는지 판단 | 비용 없이 데이터 품질 확인 |
| **실시간 참조 데이터** | 자주 변경되는 코드 테이블(부서명, 제품코드) | 동기화 파이프라인 불필요 |
| **크로스 플랫폼 JOIN** | Snowflake 데이터와 Delta Lake 데이터를 조인 | 데이터 이동 없이 분석 |

### Foreign Table의 현실적인 한계

그러나 실무에서 Foreign Table을 주력으로 사용하면 반드시 부딪히는 문제가 있습니다.

```
실제 성능 측정 사례 (1,000만 행 테이블):
- Delta Lake (Managed Table): SELECT COUNT(*) → 0.8초
- Foreign Table (PostgreSQL): SELECT COUNT(*) → 12.5초
- Foreign Table + WHERE 절: → 3.2초 (Predicate Pushdown 적용)
- Foreign Table + 복잡한 JOIN: → 45초+ (네트워크 지연 + 원본 DB 부하)
```

| 한계 | 설명 | 대안 |
|------|------|------|
| **성능** | 대량 데이터 조회 시 10~50배 느립니다 | 자주 쓰는 데이터는 Delta Lake로 복사 |
| **원본 DB 부하** | Databricks 쿼리가 원본 OLTP DB에 부하를 줍니다 | Read Replica에 연결하세요 |
| **읽기 전용** | INSERT/UPDATE/DELETE 불가 | 양방향 동기화가 필요하면 Lakeflow Connect 사용 |
| **스키마 변경 감지** | 원본 스키마 변경 시 Foreign Catalog 새로고침 필요 | 정기적으로 `REFRESH FOREIGN CATALOG` 실행 |
| **네트워크 의존** | 원본 DB와 네트워크 연결이 끊기면 조회 실패 | 중요한 데이터는 로컬 복사본 유지 |

### 실전 패턴: Foreign Table로 탐색 → Managed Table로 정착

가장 많이 사용되는 패턴은 "**먼저 Foreign Table로 탐색하고, 가치가 확인되면 Managed Table로 가져오는**" 흐름입니다.

```sql
-- 1단계: Foreign Table로 데이터 탐색
SELECT
    COUNT(*) AS total_rows,
    COUNT(DISTINCT customer_id) AS unique_customers,
    MIN(order_date) AS earliest,
    MAX(order_date) AS latest
FROM mysql_catalog.ecommerce.orders;

-- 2단계: 가치가 확인되면 Managed Table로 초기 적재
CREATE TABLE production.bronze.mysql_orders AS
SELECT * FROM mysql_catalog.ecommerce.orders
WHERE order_date >= '2024-01-01';

-- 3단계: 이후부터는 Lakeflow Connect나 Auto Loader로 증분 동기화
```

> 💡 **현업 팁**: Foreign Table을 "영구적인 데이터 소스"로 사용하려는 유혹을 피하세요. 분석 쿼리 하나가 프로덕션 MySQL을 느리게 만들어서 CS팀에서 전화가 오는 일이 실제로 발생합니다. **탐색용으로 시작하되, 반복적으로 사용하는 데이터는 반드시 Delta Lake로 가져오세요.**

---

## 정리

| 테이블 유형 | 핵심 특징 |
|------------|----------|
| **Managed Table** | Unity Catalog가 데이터와 메타데이터를 모두 관리합니다. DROP 시 데이터도 삭제됩니다 |
| **External Table** | 사용자 지정 경로에 데이터가 저장됩니다. DROP 시 데이터는 보존됩니다 |
| **Foreign Table** | 외부 DBMS의 데이터를 복사 없이 실시간으로 조회합니다 |

---

## 참고 링크

- [Databricks: Create tables](https://docs.databricks.com/aws/en/data-governance/unity-catalog/create-tables.html)
- [Databricks: Managed vs External tables](https://docs.databricks.com/aws/en/data-governance/unity-catalog/create-tables.html#managed-tables)
- [Databricks: Lakehouse Federation](https://docs.databricks.com/aws/en/query-federation/index.html)
- [Azure Databricks: Unity Catalog tables](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/create-tables)
