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
