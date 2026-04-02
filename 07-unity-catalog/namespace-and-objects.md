# 네임스페이스 구조

## Unity Catalog의 3-Level 네임스페이스

> 💡 Unity Catalog는 **Catalog → Schema → Object**3단계 계층 구조로 모든 데이터 자산을 관리합니다. 이 구조는 ANSI SQL 표준의 3-Level 네임스페이스를 따릅니다.

전통적인 데이터베이스에서 `database.table` 형식으로 2단계 네임스페이스를 사용하는 것과 달리, Unity Catalog는 **Catalog** 라는 최상위 계층을 추가하여 3단계로 확장했습니다. 이를 통해 환경별(production/staging/dev), 팀별, 도메인별 데이터를 명확하게 분리하고 독립적으로 권한을 관리할 수 있습니다.

| 수준 | 이름 | 유형 |
|------|------|------|
| Metastore | (최상위 - 리전당 1개) | 컨테이너 |
| └ Catalog | catalog_1 | 카탈로그 |
|   └ Schema | schema_a | 스키마 |
|     - | table_1 | 테이블 |
|     - | view_1 | 뷰 |
|     - | volume_1 | 볼륨 (파일) |
|     - | function_1 | 함수 |
|     - | model_1 | ML 모델 |
|   └ Schema | schema_b | 스키마 |
| └ Catalog | catalog_2 | 카탈로그 |

### 완전한 이름 (Fully Qualified Name)

Unity Catalog에서 모든 객체는 **3단계 이름** 으로 고유하게 식별됩니다. 이 이름 규칙은 SQL 표준을 따르므로, 다른 SQL 도구나 플랫폼에서도 동일한 방식으로 접근할 수 있습니다.

```sql
-- 3-Level 이름: catalog.schema.object
SELECT * FROM production.ecommerce.orders;
--               ^^^^^^^^^^  ^^^^^^^^^  ^^^^^^
--               카탈로그      스키마     테이블
```

---

## Metastore — 최상위 컨테이너

> 💡 **Metastore** 는 Unity Catalog의 **최상위 컨테이너** 입니다. 하나의 클라우드 리전에 하나의 Metastore가 존재하며, 해당 리전의 모든 Workspace가 이 Metastore를 공유합니다.

Metastore는 직접 데이터를 담지 않고, 그 아래의 Catalog → Schema → Object를 **메타데이터(이름, 위치, 권한, 리니지 등)** 로 관리하는 카탈로그 역할을 합니다.

### Metastore 설정

Metastore는 계정 관리자가 생성하며, 보통 클라우드 리전당 하나만 필요합니다. 여러 Workspace를 하나의 Metastore에 연결하면, Workspace 간에 데이터를 안전하게 공유할 수 있습니다.

| 설정 항목 | 설명 |
|-----------|------|
| **이름** | Metastore의 식별 이름입니다 (예: `ap-northeast-2-metastore`) |
| **리전** | 클라우드 리전입니다. Workspace와 동일한 리전이어야 합니다 |
| **기본 스토리지** | 관리형 테이블의 기본 저장 위치입니다 (S3, ADLS, GCS) |
| **소유자** | 계정 관리자 또는 Metastore 관리자 그룹입니다 |

```sql
-- Metastore 정보 확인
SELECT * FROM system.information_schema.catalogs;
```

---

## 각 레벨의 역할

| 레벨 | 역할 | 일반적인 구성 전략 | 예시 |
|------|------|-----------------|------|
| **Metastore** | 최상위 컨테이너. **리전별 하나** 만 존재하며, 여러 Workspace에서 공유됩니다 | 계정 수준 관리 | US-East Metastore |
| **Catalog** | 데이터 자산의 최상위 논리 그룹입니다. **환경별** 또는 **팀별** 로 분리합니다 | `production`, `staging`, `dev`, `sandbox` | 환경별 분리 |
| **Schema** | 관련 객체를 논리적으로 그룹화합니다. **도메인별** 또는 **Medallion 계층별** 로 분리합니다 | `ecommerce`, `hr`, `finance` 또는 `bronze`, `silver`, `gold` | 도메인별 분리 |
| **Object** | 실제 데이터 자산입니다. 테이블, 뷰, 볼륨, 함수, 모델 등 10가지 유형이 있습니다 | 의미 있는 이름 사용 | `orders`, `customers` |

---

## 객체 유형 상세

Unity Catalog에서 관리하는 객체는 10가지 유형이 있습니다. 각 객체 유형의 특성과 사용 시나리오를 이해하면 데이터 자산을 효과적으로 구성할 수 있습니다.

### Table (테이블)

테이블은 행(Row)과 열(Column)로 구성된 구조화된 데이터입니다. Unity Catalog에서 가장 기본적이고 중요한 객체 유형입니다.

| 테이블 유형 | 설명 | DROP 시 동작 | 적합한 경우 |
|------------|------|-------------|-----------|
| **Managed Table** | Databricks가 데이터 위치를 관리합니다 | 메타데이터 + 데이터 모두 삭제 | 신규 테이블 (권장) |
| **External Table** | 사용자가 지정한 외부 경로에 데이터가 저장됩니다 | 메타데이터만 삭제, 데이터 유지 | 기존 데이터 위치 유지 필요 |

```sql
-- Managed Table 생성
CREATE TABLE production.ecommerce.orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    total_amount DECIMAL(10,2)
) USING DELTA
COMMENT '주문 데이터';

-- External Table 생성
CREATE TABLE production.ecommerce.legacy_orders (
    order_id BIGINT,
    order_date DATE
) USING DELTA
LOCATION 's3://my-bucket/legacy/orders';
```

### View (뷰)

뷰는 SQL 쿼리를 저장한 **가상 테이블** 입니다. 데이터를 물리적으로 저장하지 않고, 조회할 때마다 정의된 쿼리를 실행합니다. 복잡한 조인이나 필터를 미리 정의해두면 사용자가 간단하게 데이터에 접근할 수 있습니다.

```sql
-- 일반 View
CREATE VIEW production.ecommerce.v_active_customers AS
SELECT customer_id, name, email
FROM production.ecommerce.customers
WHERE status = 'active';

-- Dynamic View (행 수준 보안)
CREATE VIEW production.ecommerce.v_orders_secure AS
SELECT *
FROM production.ecommerce.orders
WHERE region = current_user_region();  -- 사용자별로 다른 데이터 표시
```

### Materialized View (구체화된 뷰)

Materialized View는 쿼리 결과를 **물리적으로 저장** 하고, 원본 데이터가 변경되면 자동으로 증분 갱신합니다. 복잡한 집계 쿼리를 매번 실행하는 대신 미리 계산된 결과를 사용하므로 조회 성능이 크게 향상됩니다.

```sql
CREATE MATERIALIZED VIEW production.ecommerce.mv_daily_sales AS
SELECT
    order_date,
    region,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_sales
FROM production.ecommerce.orders
GROUP BY order_date, region;
```

### Streaming Table (스트리밍 테이블)

Streaming Table은 스트리밍 데이터를 **증분 처리** 하는 특수 테이블입니다. SDP(Streaming Data Pipeline)에서 주로 사용되며, 새로 도착한 데이터만 처리하여 효율적으로 테이블을 갱신합니다.

```sql
CREATE STREAMING TABLE production.ecommerce.bronze_orders AS
SELECT * FROM STREAM read_files('/Volumes/raw/orders/*.json');
```

### Volume (볼륨)

Volume은 **비테이블 형태의 파일**(PDF, 이미지, CSV, JSON 등)을 저장하는 공간입니다. 기존의 DBFS(Databricks File System)를 대체하며, Unity Catalog의 거버넌스(권한, 감사)가 적용됩니다.

| Volume 유형 | 설명 | 적합한 용도 |
|------------|------|-----------|
| **Managed Volume** | Databricks가 스토리지 위치를 관리합니다 | 일반적인 파일 저장 (권장) |
| **External Volume** | 사용자가 지정한 외부 경로를 사용합니다 | 기존 스토리지 경로 유지 |

```sql
-- Managed Volume 생성
CREATE VOLUME production.ecommerce.raw_files
COMMENT '원본 파일 저장소';

-- Volume에 파일 경로 접근
-- /Volumes/production/ecommerce/raw_files/invoice.pdf
```

### Function (함수)

SQL 또는 Python으로 정의한 사용자 정의 함수(UDF)입니다. Unity Catalog에 등록하면 어떤 Workspace에서든 동일한 함수를 사용할 수 있으며, 함수에 대한 접근 권한도 관리됩니다.

```sql
-- SQL UDF
CREATE FUNCTION production.ecommerce.calculate_tax(price DECIMAL(10,2), rate DECIMAL(4,2))
RETURNS DECIMAL(10,2)
RETURN price * rate;

-- Python UDF
CREATE FUNCTION production.ecommerce.clean_text(text STRING)
RETURNS STRING
LANGUAGE PYTHON AS $$
    import re
    return re.sub(r'[^가-힣a-zA-Z0-9\s]', '', text).strip()
$$;
```

### Model (ML 모델)

MLflow로 학습하고 등록한 ML 모델입니다. Unity Catalog에 등록되면 데이터 테이블과 동일한 거버넌스(권한, 리니지, 태그)가 적용되며, Model Serving으로 직접 배포할 수 있습니다.

```python
import mlflow

# 모델을 Unity Catalog에 등록
mlflow.register_model(
    "runs:/<run_id>/model",
    "production.ml_models.churn_prediction"
)

# 등록된 모델 로드
model = mlflow.pyfunc.load_model("models:/production.ml_models.churn_prediction@champion")
```

### Connection (연결)

외부 시스템(MySQL, PostgreSQL, Snowflake 등)의 **연결 정보** 를 저장합니다. Lakeflow Connect에서 외부 데이터 소스를 연결할 때 사용합니다.

```sql
CREATE CONNECTION mysql_prod TYPE MYSQL
OPTIONS (
    host 'mysql.example.com',
    port '3306',
    user '{{secrets/db/mysql-user}}',
    password '{{secrets/db/mysql-pass}}'
);
```

### 기타 보안 관련 객체

| 객체 유형 | 설명 |
|-----------|------|
| **External Location** | 외부 클라우드 스토리지 경로(S3, ADLS, GCS)를 등록합니다. External Table이나 External Volume이 이 위치를 참조합니다 |
| **Storage Credential** | 클라우드 스토리지에 접근하기 위한 자격 증명(IAM Role, Service Principal 등)을 저장합니다 |

---

## 네임스페이스 구성 전략

### 전략 1: 환경별 카탈로그 + 도메인별 스키마 (가장 일반적)

가장 널리 사용되는 패턴입니다. 환경(production/staging/dev)을 카탈로그로 분리하고, 각 환경 내에서 비즈니스 도메인으로 스키마를 나눕니다. 환경별 접근 권한을 카탈로그 수준에서 쉽게 관리할 수 있다는 것이 장점입니다.

| 카탈로그 | 스키마 | 설명 |
|---------|--------|------|
| **production** | ecommerce | 주문, 고객, 상품 |
| | hr | 직원, 급여 |
| | finance | 매출, 비용 |
| **staging** | ecommerce | - |
| | hr | - |
| **dev** | ecommerce | - |
| | sandbox | 개인 실험용 |

### 전략 2: 환경별 카탈로그 + Medallion별 스키마

데이터 엔지니어링 관점에서 데이터의 처리 단계를 중심으로 스키마를 구성하는 패턴입니다. ETL 파이프라인의 흐름이 스키마 구조에 자연스럽게 반영됩니다.

| 카탈로그 | 스키마 | 설명 |
|---------|--------|------|
| **production** | bronze | 원본 데이터 |
| | silver | 정제된 데이터 |
| | gold | 비즈니스 집계 |
| **dev** | bronze | - |
| | silver | - |
| | gold | - |

### 전략 3: 팀별 카탈로그 (대규모 조직)

대규모 조직에서 팀 간 데이터 독립성을 최대화할 때 사용합니다. 각 팀이 자체 카탈로그를 소유하고, 공유 데이터는 별도의 shared 카탈로그에 공개합니다.

| 카탈로그 | 스키마 | 설명 |
|---------|--------|------|
| **data_engineering** | bronze, silver, gold | 데이터 파이프라인 |
| **data_science** | features, models, experiments | ML/AI |
| **shared** | reference_data | 코드 테이블 등 |
| | cross_team | 팀 간 공유 데이터 |

> 💡 **권장**: 대부분의 조직에서는 **전략 1 (환경별 카탈로그 + 도메인별 스키마)** 로 시작하는 것을 권장합니다. 조직이 커지고 요구사항이 복잡해지면 전략 3으로 확장할 수 있습니다.

### SQL로 네임스페이스 생성

```sql
-- 카탈로그 생성
CREATE CATALOG IF NOT EXISTS production
COMMENT '프로덕션 데이터';

-- 스키마 생성
CREATE SCHEMA IF NOT EXISTS production.ecommerce
COMMENT '이커머스 도메인'
MANAGED LOCATION 's3://my-bucket/production/ecommerce';

-- 테이블 생성
CREATE TABLE production.ecommerce.orders (
    order_id BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id BIGINT NOT NULL,
    order_date DATE,
    total_amount DECIMAL(10,2),
    status STRING
) USING DELTA
COMMENT '주문 데이터'
TBLPROPERTIES ('quality' = 'gold');

-- 볼륨 생성
CREATE VOLUME production.ecommerce.raw_files
COMMENT '원본 파일 저장소';

-- 기본 카탈로그/스키마 설정 (매번 3-Level 이름을 쓰지 않아도 됨)
USE CATALOG production;
USE SCHEMA ecommerce;
SELECT * FROM orders;  -- = production.ecommerce.orders
```

---

## Information Schema — 메타데이터 조회

Unity Catalog는 **ANSI SQL 표준의 information_schema** 를 제공하여, 카탈로그, 스키마, 테이블 등의 메타데이터를 SQL로 조회할 수 있습니다.

```sql
-- 카탈로그 목록 조회
SELECT catalog_name, comment, created
FROM system.information_schema.catalogs;

-- 특정 카탈로그의 스키마 목록
SELECT schema_name, comment, created
FROM production.information_schema.schemata;

-- 특정 스키마의 테이블 목록
SELECT table_name, table_type, comment, created
FROM production.information_schema.tables
WHERE table_schema = 'ecommerce';

-- 특정 테이블의 컬럼 목록
SELECT column_name, data_type, is_nullable, comment
FROM production.information_schema.columns
WHERE table_schema = 'ecommerce' AND table_name = 'orders';

-- 볼륨 목록
SELECT volume_name, volume_type, comment
FROM production.information_schema.volumes
WHERE volume_schema = 'ecommerce';
```

---

## Managed vs External

| 비교 | Managed (관리형) | External (외부) |
|------|-----------------|----------------|
| **데이터 위치** | Databricks가 관리하는 위치에 자동 저장 | 사용자가 지정한 외부 경로 (S3, ADLS) |
| **DROP 시** | 테이블 + 데이터 **모두 삭제** | 테이블 정의만 삭제, **데이터는 유지** |
| **Predictive Optimization** | ✅ 지원 | ❌ 미지원 |
| **스토리지 관리** | 자동 (VACUUM, 압축 등) | 수동 또는 제한적 |
| **마이그레이션** | 카탈로그 간 이동 가능 | 데이터 위치 고정 |
| **권장** | ✅ 신규 테이블에 권장 | 기존 데이터 위치를 유지해야 할 때 |

> 💡 **Managed를 권장하는 이유**: Managed Table은 Databricks가 데이터 레이아웃 최적화(Predictive Optimization), 파일 정리(VACUUM), 소형 파일 병합 등을 자동으로 수행합니다. External Table에서는 이러한 최적화 기능이 제한되므로, 특별한 이유가 없다면 Managed Table을 사용하는 것이 좋습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **3-Level 네임스페이스** | Catalog → Schema → Object 계층 구조입니다. ANSI SQL 표준을 따릅니다 |
| **Metastore** | 리전당 하나의 최상위 컨테이너입니다. 여러 Workspace가 공유합니다 |
| **Catalog** | 최상위 그룹. 환경별(prod/dev) 또는 팀별로 분리합니다 |
| **Schema** | 관련 객체를 묶는 논리 그룹. 도메인별로 분리합니다 |
| **10가지 객체 유형** | Table, View, MV, Streaming Table, Volume, Function, Model, Connection, External Location, Storage Credential |
| **Managed vs External** | Managed는 Databricks가 관리하고 최적화합니다. 신규 테이블에 권장합니다 |
| **information_schema** | SQL로 메타데이터(카탈로그, 스키마, 테이블, 컬럼 등)를 조회할 수 있습니다 |

---

## 참고 링크

- [Databricks: Unity Catalog objects](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)
- [Databricks: Create catalogs and schemas](https://docs.databricks.com/aws/en/data-governance/unity-catalog/create-catalogs.html)
- [Databricks: Managed and external tables](https://docs.databricks.com/aws/en/data-governance/unity-catalog/create-tables.html)
- [Databricks: Volumes](https://docs.databricks.com/aws/en/connect/unity-catalog/volumes.html)
- [Azure Databricks: Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/)
