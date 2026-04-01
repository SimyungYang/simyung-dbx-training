# UniForm (Universal Format) 상세

## 왜 UniForm이 필요한가요?

현대 데이터 생태계에서는 하나의 플랫폼만 사용하는 경우가 드뭅니다. Databricks에서 데이터를 관리하면서, Snowflake에서 분석하고, Trino로 애드혹 쿼리를 실행하고, Apache Flink로 실시간 처리를 하는 **멀티 엔진 환경** 이 일반적입니다.

문제는 각 엔진이 서로 다른 테이블 포맷을 사용한다는 것입니다. Databricks는 Delta Lake, Snowflake와 Trino는 주로 Apache Iceberg를 지원합니다. 동일한 데이터를 여러 포맷으로 복사하면 ** 데이터 일관성 유지**, ** 스토리지 비용**, ** 파이프라인 복잡도** 문제가 발생합니다.

> 💡 **UniForm(Universal Format)** 은 하나의 Delta Lake 테이블에 **Iceberg 호환 메타데이터를 자동 생성** 하여, 데이터 복사 없이 여러 엔진에서 동일한 데이터를 읽을 수 있게 하는 기능입니다.

---

## 핵심 개념

UniForm의 핵심은 ** 데이터 파일은 하나**, ** 메타데이터만 두 가지** 라는 점입니다.

**Delta 테이블 디렉토리 구조 (UniForm 활성화 시)**:

| 경로 | 설명 |
|------|------|
| `_delta_log/*.json` | Delta 프로토콜 메타데이터 |
| `metadata/*.metadata.json`, `snap-*.avro` | Iceberg 프로토콜 메타데이터 (UniForm이 자동 생성) |
| `part-*.parquet` | 실제 데이터 파일 (Delta와 Iceberg가 공유) |

- **Databricks/Spark** 는 `_delta_log/`를 읽어 Delta 프로토콜로 데이터에 접근합니다
- **Snowflake/Trino/Flink** 는 `metadata/`를 읽어 Iceberg 프로토콜로 동일한 데이터에 접근합니다
- **Parquet 파일은 공유** 되므로 데이터 복사가 발생하지 않습니다

---

## 활성화 방법

### 새 테이블 생성 시

```sql
-- Iceberg 호환성이 있는 Delta 테이블 생성
CREATE TABLE catalog.schema.shared_events (
    event_id BIGINT,
    user_id BIGINT,
    event_type STRING,
    event_time TIMESTAMP,
    properties MAP<STRING, STRING>
)
TBLPROPERTIES (
    'delta.universalFormat.enabledFormats' = 'iceberg'
);
```

### 기존 테이블에 활성화

```sql
-- 기존 Delta 테이블에 UniForm 활성화
ALTER TABLE catalog.schema.existing_table
SET TBLPROPERTIES ('delta.universalFormat.enabledFormats' = 'iceberg');
```

### 비활성화

```sql
-- UniForm 비활성화
ALTER TABLE catalog.schema.shared_events
UNSET TBLPROPERTIES ('delta.universalFormat.enabledFormats');
```

### 활성화 시 자동 설정

UniForm을 활성화하면 다음 속성이 ** 자동으로 함께 설정** 됩니다.

| 속성 | 자동 설정값 | 설명 |
|------|------------|------|
| `delta.columnMapping.mode` | `name` | 컬럼 이름 기반 매핑 활성화 |
| `delta.minReaderVersion` | 2 이상 | Reader 프로토콜 버전 |
| `delta.minWriterVersion` | 7 이상 | Writer 프로토콜 버전 |

---

## Iceberg REST Catalog (IRC)

외부 엔진이 UniForm이 활성화된 테이블을 읽으려면, 테이블의 ** 위치(메타데이터 경로)** 를 알아야 합니다. Unity Catalog는 **Iceberg REST Catalog** 표준을 지원하여, 외부 엔진이 HTTP API를 통해 테이블 목록을 조회하고 메타데이터 위치를 확인할 수 있습니다.

### 연결 정보

외부 엔진에서 Unity Catalog의 Iceberg REST Catalog에 연결하려면 다음 정보가 필요합니다.

| 항목 | 값 |
|------|---|
| **URI**| `https://<workspace-url>/api/2.1/unity-catalog/iceberg` |
| ** 인증**| Databricks Personal Access Token 또는 OAuth M2M |
| **Warehouse**| Unity Catalog의 카탈로그 이름 |

### 설정 예시

```python
# PyIceberg에서 Unity Catalog 연결
from pyiceberg.catalog import load_catalog

catalog = load_catalog(
    "databricks_unity",
    **{
        "type": "rest",
        "uri": "https://my-workspace.cloud.databricks.com/api/2.1/unity-catalog/iceberg",
        "token": "<personal-access-token>",
        "warehouse": "my_catalog"
    }
)

# 테이블 조회
table = catalog.load_table("my_schema.shared_events")
df = table.scan().to_pandas()
```

---

## 외부 엔진에서 읽기

### Snowflake에서 읽기

```sql
-- Snowflake에서 Databricks의 UniForm 테이블 읽기
CREATE OR REPLACE ICEBERG TABLE snowflake_db.public.shared_events
  CATALOG = 'ICEBERG_REST'
  EXTERNAL_VOLUME = 'my_external_volume'
  CATALOG_TABLE_NAME = 'my_schema.shared_events'
  CATALOG_NAMESPACE = 'my_catalog';

-- 조회
SELECT * FROM snowflake_db.public.shared_events;
```

### Trino에서 읽기

```properties
# Trino Iceberg 카탈로그 설정 (etc/catalog/databricks.properties)
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=https://my-workspace.cloud.databricks.com/api/2.1/unity-catalog/iceberg
iceberg.rest-catalog.warehouse=my_catalog
iceberg.rest-catalog.security=OAUTH2
iceberg.rest-catalog.oauth2.token=<token>
```

```sql
-- Trino에서 조회
SELECT * FROM databricks.my_schema.shared_events;
```

### Apache Spark (외부)에서 읽기

```python
# 외부 Spark에서 Iceberg REST Catalog를 통해 읽기
spark = SparkSession.builder \
    .config("spark.sql.catalog.unity", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.unity.type", "rest") \
    .config("spark.sql.catalog.unity.uri",
            "https://my-workspace.cloud.databricks.com/api/2.1/unity-catalog/iceberg") \
    .config("spark.sql.catalog.unity.token", "<token>") \
    .config("spark.sql.catalog.unity.warehouse", "my_catalog") \
    .getOrCreate()

df = spark.table("unity.my_schema.shared_events")
```

---

## UniForm 사용 시 고려사항

| 항목 | 설명 |
|------|------|
| **쓰기는 Delta만**| UniForm 테이블에 데이터를 쓰는 것은 **Databricks(Delta 프로토콜)** 에서만 가능합니다. 외부 엔진은 읽기 전용입니다 |
| ** 메타데이터 동기화 지연**| Delta 쓰기 후 Iceberg 메타데이터가 자동 생성되지만, 약간의 지연이 있을 수 있습니다 |
| **Deletion Vectors 호환**| UniForm은 Deletion Vectors와 함께 사용 가능합니다 |
| **Liquid Clustering 호환**| UniForm과 Liquid Clustering을 동시에 사용할 수 있습니다 |
| ** 스토리지 오버헤드**| Iceberg 메타데이터 파일이 추가로 생성되지만, 데이터 파일은 공유되므로 오버헤드는 최소화됩니다 |

---

## 언제 UniForm을 사용해야 하나요?

| 상황 | UniForm 필요 여부 |
|------|------------------|
| Databricks만 사용 | ❌ 불필요 (Delta Lake만으로 충분) |
| Databricks + Snowflake에서 같은 데이터 조회 | ✅ 필요 |
| Databricks + Trino/Presto 멀티 엔진 분석 | ✅ 필요 |
| Databricks + Apache Flink 실시간 처리 | ✅ 필요 |
| 데이터를 ETL로 복사하여 다른 시스템에 제공 | ⚠️ UniForm으로 복사 없이 해결 가능 |

> 💡 ** 핵심 메시지**: "하나의 데이터, 여러 엔진에서 조회"가 필요하다면 UniForm을 활성화하세요. 데이터 복사 없이 멀티 엔진 호환성을 확보할 수 있습니다.

---

## 심화: UniForm 구성 가이드

### Step-by-Step: UniForm 활성화 및 외부 엔진 연결

#### Step 1: UniForm 활성화

```sql
-- 새 테이블 생성 시 UniForm 활성화
CREATE TABLE catalog.schema.sales (
    sale_id BIGINT,
    product STRING,
    amount DECIMAL(10,2),
    sale_date DATE
)
TBLPROPERTIES (
    'delta.universalFormat.enabledFormats' = 'iceberg',
    'delta.enableIcebergCompatV2' = 'true'
);

-- 기존 테이블에 UniForm 활성화
ALTER TABLE catalog.schema.existing_table
SET TBLPROPERTIES (
    'delta.universalFormat.enabledFormats' = 'iceberg'
);
```

> ⚠️ **Column Mapping 필수**: UniForm을 활성화하려면 `delta.columnMapping.mode = 'name'`이 설정되어야 합니다. 최신 테이블은 기본값이지만, 레거시 테이블은 수동 활성화가 필요합니다.

```sql
-- 레거시 테이블: Column Mapping 먼저 활성화
ALTER TABLE catalog.schema.legacy_table
SET TBLPROPERTIES ('delta.columnMapping.mode' = 'name');

-- 이후 UniForm 활성화
ALTER TABLE catalog.schema.legacy_table
SET TBLPROPERTIES ('delta.universalFormat.enabledFormats' = 'iceberg');
```

#### Step 2: Iceberg REST Catalog (IRC) 연결 설정

Unity Catalog가 ** 표준 Iceberg REST Catalog API** 를 노출하므로, 외부 엔진은 이 엔드포인트에 연결하면 됩니다.

| 설정 항목 | 값 |
|----------|-----|
| **Catalog URI**| `https://<workspace-url>/api/2.1/unity-catalog/iceberg` |
| ** 인증**| OAuth M2M (Service Principal) 또는 PAT |
| **Warehouse**| Unity Catalog의 catalog 이름 |

#### Step 3: 외부 엔진에서 읽기

**Snowflake에서 읽기 (Iceberg Catalog Integration)**
```sql
-- Snowflake: Databricks Unity Catalog를 Iceberg 카탈로그로 등록
CREATE OR REPLACE CATALOG INTEGRATION databricks_iceberg
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  CATALOG_NAMESPACE = 'catalog.schema'
  REST_CONFIG = (
    CATALOG_URI = 'https://<workspace-url>/api/2.1/unity-catalog/iceberg'
    WAREHOUSE = 'catalog'
  )
  REST_AUTHENTICATION = (
    TYPE = OAUTH
    OAUTH_CLIENT_ID = '<service-principal-client-id>'
    OAUTH_CLIENT_SECRET = '<service-principal-secret>'
    OAUTH_TOKEN_URI = 'https://<workspace-url>/oidc/v1/token'
  );

-- Snowflake에서 Databricks 테이블 읽기
SELECT * FROM catalog.schema.sales WHERE sale_date >= '2025-01-01';
```

**Apache Spark OSS에서 읽기**
```python
# Spark OSS에서 Iceberg REST Catalog 연결
spark = SparkSession.builder \
    .config("spark.sql.catalog.databricks", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.databricks.type", "rest") \
    .config("spark.sql.catalog.databricks.uri",
            "https://<workspace-url>/api/2.1/unity-catalog/iceberg") \
    .config("spark.sql.catalog.databricks.credential", "<pat-token>") \
    .config("spark.sql.catalog.databricks.warehouse", "catalog") \
    .getOrCreate()

# Databricks Delta 테이블을 Iceberg로 읽기
df = spark.read.table("databricks.schema.sales")
```

**Trino/Presto에서 읽기**
```properties
# Trino: etc/catalog/databricks.properties
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=https://<workspace-url>/api/2.1/unity-catalog/iceberg
iceberg.rest-catalog.warehouse=catalog
iceberg.rest-catalog.security=OAUTH2
iceberg.rest-catalog.oauth2.token=<pat-token>
```

### Credential Vending (자격 증명 발급)

외부 엔진이 실제 Parquet 파일을 읽으려면 ** 클라우드 스토리지 접근 권한** 이 필요합니다. Unity Catalog의 **Credential Vending** 은 이 문제를 해결합니다.

| 방식 | 설명 |
|------|------|
| ** 자동 (Credential Vending)**| IRC 요청 시 Unity Catalog가 ** 임시 자격 증명(STS Token)을 자동 발급** 합니다. 외부 엔진이 S3/ADLS에 직접 접근할 수 있는 단기 토큰을 받습니다 |
| ** 수동** | 외부 엔진에 S3/ADLS 접근 권한을 직접 설정합니다 (IAM Role, Managed Identity) |

> 💡 **Credential Vending의 장점**: 외부 엔진에 영구 스토리지 자격 증명을 부여하지 않아도 됩니다. Unity Catalog가 요청마다 ** 최소 권한의 임시 토큰** 을 발급하므로 보안이 강화됩니다.

### UniForm 운영 시 주의사항

| 항목 | 주의 |
|------|------|
| ** 쓰기는 Databricks만** | 외부 엔진에서는 읽기만 가능합니다. 쓰기는 반드시 Databricks에서 수행합니다 |
| ** 메타데이터 동기화 지연** | Delta 커밋 후 Iceberg 메타데이터 생성까지 약간의 지연(수 초~수 분)이 있을 수 있습니다 |
| ** 스토리지 오버헤드** | Iceberg 메타데이터 파일이 추가로 생성되므로 스토리지가 약 1~5% 증가합니다 |
| **VACUUM 주의**| VACUUM이 Iceberg에서 참조하는 파일을 삭제하지 않도록 보존 기간을 충분히 설정합니다 |
| ** 지원 데이터 타입**| 대부분의 타입이 호환되지만, MAP 타입 등 일부 복잡한 타입은 제한될 수 있습니다 |

### Managed Iceberg Tables (Iceberg 네이티브 쓰기)

UniForm과 별도로, Databricks는 **Iceberg 형식으로 직접 쓰는 Managed Iceberg Tables** 도 지원합니다.

```sql
-- Managed Iceberg Table 생성 (Iceberg 네이티브 포맷)
CREATE TABLE catalog.schema.iceberg_sales (
    sale_id BIGINT,
    product STRING,
    amount DECIMAL(10,2)
) USING ICEBERG;
```

| 비교 | UniForm (Delta + Iceberg 읽기) | Managed Iceberg (Iceberg 네이티브) |
|------|-------------------------------|----------------------------------|
| ** 저장 포맷**| Delta Lake (Parquet + Delta Log) | Iceberg (Parquet + Iceberg Metadata) |
| **Databricks 기능**| ✅ Delta의 모든 기능 (Liquid Clustering, DV, Time Travel 등) | ⚠️ Iceberg 기능만 (일부 Delta 고유 기능 미지원) |
| ** 외부 엔진 읽기**| ✅ Iceberg 메타데이터 자동 생성 | ✅ 네이티브 Iceberg |
| ** 외부 엔진 쓰기**| ❌ 읽기 전용 | ✅ 외부에서도 쓰기 가능 |
| ** 적합한 경우**| Databricks가 주(Primary) 엔진, 외부는 읽기만 | 외부 엔진도 쓰기해야 하는 경우 |

> 💡 ** 선택 가이드**: 대부분의 경우 **UniForm(Delta + Iceberg 읽기)** 을 권장합니다. Delta Lake의 모든 고급 기능(Liquid Clustering, Deletion Vectors, Predictive Optimization 등)을 활용하면서 외부 호환성도 확보합니다. ** 외부 엔진에서 쓰기가 필수인 경우에만**Managed Iceberg를 사용합니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **UniForm**| Delta 테이블에 Iceberg 호환 메타데이터를 자동 생성하여 멀티 엔진 읽기를 지원합니다 |
| ** 데이터 복사 없음**| Parquet 데이터 파일은 공유되고, 메타데이터만 두 형식으로 유지됩니다 |
| **Iceberg REST Catalog**| Unity Catalog가 표준 Iceberg REST API를 제공하여 외부 엔진의 카탈로그 역할을 합니다 |
| ** 읽기 전용** | 외부 엔진에서는 읽기만 가능하고, 쓰기는 Databricks에서만 수행합니다 |

---

## 참고 링크

- [Databricks: UniForm](https://docs.databricks.com/aws/en/delta/uniform.html)
- [Databricks: Iceberg REST Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/iceberg-rest-catalog.html)
- [Azure Databricks: UniForm](https://learn.microsoft.com/en-us/azure/databricks/delta/uniform)
- [Databricks Blog: Delta Lake Universal Format](https://www.databricks.com/blog/delta-lake-universal-format-unifrom)
- [Apache Iceberg REST Catalog Spec](https://iceberg.apache.org/concepts/catalog/#rest-catalog)
