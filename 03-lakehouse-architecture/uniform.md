# UniForm (Universal Format) 상세

## 왜 UniForm이 필요한가요?

현대 데이터 생태계에서는 하나의 플랫폼만 사용하는 경우가 드뭅니다. Databricks에서 데이터를 관리하면서, Snowflake에서 분석하고, Trino로 애드혹 쿼리를 실행하고, Apache Flink로 실시간 처리를 하는 **멀티 엔진 환경**이 일반적입니다.

문제는 각 엔진이 서로 다른 테이블 포맷을 사용한다는 것입니다. Databricks는 Delta Lake, Snowflake와 Trino는 주로 Apache Iceberg를 지원합니다. 동일한 데이터를 여러 포맷으로 복사하면 **데이터 일관성 유지**, **스토리지 비용**, **파이프라인 복잡도** 문제가 발생합니다.

> 💡 **UniForm(Universal Format)** 은 하나의 Delta Lake 테이블에 **Iceberg 호환 메타데이터를 자동 생성**하여, 데이터 복사 없이 여러 엔진에서 동일한 데이터를 읽을 수 있게 하는 기능입니다.

---

## 핵심 개념

UniForm의 핵심은 **데이터 파일은 하나**, **메타데이터만 두 가지**라는 점입니다.

```
Delta 테이블 디렉토리:
├── _delta_log/                  ← Delta 프로토콜 메타데이터
│   ├── 00000000000000000000.json
│   ├── 00000000000000000001.json
│   └── ...
├── metadata/                    ← Iceberg 프로토콜 메타데이터 (UniForm이 자동 생성)
│   ├── v1.metadata.json
│   ├── snap-xxxxx.avro
│   └── ...
├── part-00001.parquet          ← 실제 데이터 (공유)
├── part-00002.parquet
└── ...
```

- **Databricks/Spark**는 `_delta_log/`를 읽어 Delta 프로토콜로 데이터에 접근합니다
- **Snowflake/Trino/Flink**는 `metadata/`를 읽어 Iceberg 프로토콜로 동일한 데이터에 접근합니다
- **Parquet 파일은 공유**되므로 데이터 복사가 발생하지 않습니다

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

UniForm을 활성화하면 다음 속성이 **자동으로 함께 설정**됩니다.

| 속성 | 자동 설정값 | 설명 |
|------|------------|------|
| `delta.columnMapping.mode` | `name` | 컬럼 이름 기반 매핑 활성화 |
| `delta.minReaderVersion` | 2 이상 | Reader 프로토콜 버전 |
| `delta.minWriterVersion` | 7 이상 | Writer 프로토콜 버전 |

---

## Iceberg REST Catalog (IRC)

외부 엔진이 UniForm이 활성화된 테이블을 읽으려면, 테이블의 **위치(메타데이터 경로)** 를 알아야 합니다. Unity Catalog는 **Iceberg REST Catalog** 표준을 지원하여, 외부 엔진이 HTTP API를 통해 테이블 목록을 조회하고 메타데이터 위치를 확인할 수 있습니다.

### 연결 정보

외부 엔진에서 Unity Catalog의 Iceberg REST Catalog에 연결하려면 다음 정보가 필요합니다.

| 항목 | 값 |
|------|---|
| **URI** | `https://<workspace-url>/api/2.1/unity-catalog/iceberg` |
| **인증** | Databricks Personal Access Token 또는 OAuth M2M |
| **Warehouse** | Unity Catalog의 카탈로그 이름 |

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
| **쓰기는 Delta만** | UniForm 테이블에 데이터를 쓰는 것은 **Databricks(Delta 프로토콜)** 에서만 가능합니다. 외부 엔진은 읽기 전용입니다 |
| **메타데이터 동기화 지연** | Delta 쓰기 후 Iceberg 메타데이터가 자동 생성되지만, 약간의 지연이 있을 수 있습니다 |
| **Deletion Vectors 호환** | UniForm은 Deletion Vectors와 함께 사용 가능합니다 |
| **Liquid Clustering 호환** | UniForm과 Liquid Clustering을 동시에 사용할 수 있습니다 |
| **스토리지 오버헤드** | Iceberg 메타데이터 파일이 추가로 생성되지만, 데이터 파일은 공유되므로 오버헤드는 최소화됩니다 |

---

## 언제 UniForm을 사용해야 하나요?

| 상황 | UniForm 필요 여부 |
|------|------------------|
| Databricks만 사용 | ❌ 불필요 (Delta Lake만으로 충분) |
| Databricks + Snowflake에서 같은 데이터 조회 | ✅ 필요 |
| Databricks + Trino/Presto 멀티 엔진 분석 | ✅ 필요 |
| Databricks + Apache Flink 실시간 처리 | ✅ 필요 |
| 데이터를 ETL로 복사하여 다른 시스템에 제공 | ⚠️ UniForm으로 복사 없이 해결 가능 |

> 💡 **핵심 메시지**: "하나의 데이터, 여러 엔진에서 조회"가 필요하다면 UniForm을 활성화하세요. 데이터 복사 없이 멀티 엔진 호환성을 확보할 수 있습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **UniForm** | Delta 테이블에 Iceberg 호환 메타데이터를 자동 생성하여 멀티 엔진 읽기를 지원합니다 |
| **데이터 복사 없음** | Parquet 데이터 파일은 공유되고, 메타데이터만 두 형식으로 유지됩니다 |
| **Iceberg REST Catalog** | Unity Catalog가 표준 Iceberg REST API를 제공하여 외부 엔진의 카탈로그 역할을 합니다 |
| **읽기 전용** | 외부 엔진에서는 읽기만 가능하고, 쓰기는 Databricks에서만 수행합니다 |

---

## 참고 링크

- [Databricks: UniForm](https://docs.databricks.com/aws/en/delta/uniform.html)
- [Databricks: Iceberg REST Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/iceberg-rest-catalog.html)
- [Azure Databricks: UniForm](https://learn.microsoft.com/en-us/azure/databricks/delta/uniform)
- [Databricks Blog: Delta Lake Universal Format](https://www.databricks.com/blog/delta-lake-universal-format-unifrom)
- [Apache Iceberg REST Catalog Spec](https://iceberg.apache.org/concepts/catalog/#rest-catalog)
