# Delta Lake & Apache Iceberg — 상호 운용성

## 왜 이 주제를 다루나요?

현재 레이크하우스 생태계에서는 두 가지 주요 오픈 테이블 포맷이 경쟁하고 있습니다.

| 포맷 | 주도 | 설명 |
|------|------|------|
| **Delta Lake** | Databricks | Spark 기반, 가장 오래된 레이크하우스 포맷입니다 |
| **Apache Iceberg** | Netflix → Apache Foundation | 엔진 독립적, 빠르게 채택 확산 중입니다 |

실무에서는 Databricks(Delta Lake)와 다른 플랫폼(Snowflake, Athena 등 Iceberg 사용)이 공존하는 경우가 많습니다. 이 문서에서는 두 포맷 간의 **상호 운용성(Interoperability)** 을 어떻게 확보하는지 살펴보겠습니다.

---

## Delta Lake vs Apache Iceberg 비교

| 비교 항목 | Delta Lake | Apache Iceberg |
|-----------|------------|----------------|
| **파일 포맷** | Parquet | Parquet (Avro, ORC도 지원) |
| **트랜잭션 로그** | JSON 기반 (_delta_log/) | JSON + Avro 기반 (metadata/) |
| **ACID** | ✅ | ✅ |
| **타임 트래블** | ✅ | ✅ |
| **스키마 진화** | ✅ | ✅ |
| **파티션 진화** | CLUSTER BY로 변경 | Hidden Partitioning |
| **주요 엔진** | Spark, Databricks | Spark, Trino, Flink, Dremio |
| **주도 벤더** | Databricks | Netflix → 오픈소스 커뮤니티 |

두 포맷은 매우 유사한 기능을 제공하며, 실제 데이터는 모두 **Parquet 파일**로 저장됩니다. 차이는 주로 **메타데이터(트랜잭션 로그)를 관리하는 방식**에 있습니다.

---

## UniForm: Delta Lake의 호환성 솔루션

### 개념

> 💡 **UniForm(유니폼)** 은 Delta Lake 테이블을 **Iceberg 포맷으로도 읽을 수 있게** 해주는 Databricks의 호환성 기능입니다. 하나의 Delta 테이블을 생성하면, Iceberg 클라이언트에서도 동일한 테이블을 읽을 수 있습니다.

| 구성 요소 | Delta 테이블 (원본) | UniForm 적용 후 |
|-----------|-------------------|----------------|
| 메타데이터 | `_delta_log/` (Delta 메타데이터) | `_delta_log/` + `metadata/` (Iceberg 메타데이터 자동 생성) |
| 데이터 파일 | Parquet 파일 (공유) | 동일한 Parquet 파일 (복사 없음) |

핵심은 **데이터 파일(Parquet)은 하나**이고, 메타데이터만 Delta와 Iceberg 두 가지 형식으로 유지한다는 것입니다. 따라서 데이터 복사가 발생하지 않습니다.

### 설정 방법

```sql
-- UniForm이 활성화된 테이블 생성
CREATE TABLE catalog.schema.shared_sales (
    sale_id BIGINT,
    product STRING,
    amount DECIMAL(10, 2),
    sale_date DATE
)
USING DELTA
TBLPROPERTIES (
    'delta.universalFormat.enabledFormats' = 'iceberg'
);
```

기존 테이블에 활성화하는 것도 가능합니다.

```sql
ALTER TABLE catalog.schema.existing_table
SET TBLPROPERTIES ('delta.universalFormat.enabledFormats' = 'iceberg');
```

### 활용 시나리오

| 시나리오 | 설명 |
|----------|------|
| **Snowflake에서 읽기** | Databricks에서 Delta로 관리하는 테이블을 Snowflake에서 Iceberg로 읽습니다 |
| **AWS Athena에서 쿼리** | S3에 저장된 Delta 테이블을 Athena에서 Iceberg로 조회합니다 |
| **멀티 엔진 분석** | Trino, Flink 등 Iceberg를 지원하는 다양한 엔진에서 동일 데이터를 분석합니다 |

---

## Iceberg REST Catalog (IRC)

> 🆕 **Unity Catalog as Iceberg REST Catalog**: Databricks의 Unity Catalog는 **Iceberg REST Catalog** 표준을 지원합니다. 이를 통해 외부 엔진(Snowflake, Trino, Apache Flink 등)이 Unity Catalog에 직접 연결하여 Delta 테이블을 Iceberg 테이블처럼 조회할 수 있습니다.

| 엔진 | 프로토콜 | Unity Catalog 역할 |
|------|----------|-------------------|
| Databricks | Delta 프로토콜 | Iceberg REST Catalog로 동작 |
| Snowflake | Iceberg 프로토콜 | 동일한 데이터에 읽기 접근 |
| Apache Spark (외부) | Iceberg 프로토콜 | 동일한 데이터에 읽기 접근 |
| Trino / Presto | Iceberg 프로토콜 | 동일한 데이터에 읽기 접근 |

> 💡 **REST Catalog란?** 테이블 포맷의 메타데이터(어떤 테이블이 있고, 어디에 저장되어 있는지)를 HTTP REST API로 관리하는 표준 인터페이스입니다. 이전에는 각 엔진이 자체적으로 메타데이터를 관리했지만, REST Catalog를 통해 **하나의 카탈로그로 모든 엔진이 동일한 테이블 목록을 공유**할 수 있게 되었습니다.

### 외부 엔진에서 연결하는 방법

```python
# PyIceberg에서 Unity Catalog에 연결하는 예시
from pyiceberg.catalog import load_catalog

catalog = load_catalog(
    "unity",
    **{
        "type": "rest",
        "uri": "https://<workspace-url>/api/2.1/unity-catalog/iceberg",
        "token": "<access-token>",
        "warehouse": "<catalog-name>"
    }
)

# Delta 테이블을 Iceberg 테이블처럼 조회
table = catalog.load_table("schema.shared_sales")
df = table.scan().to_pandas()
```

---

## Foreign Iceberg Tables (외부 Iceberg 테이블 읽기)

반대로, 외부에서 생성된 Iceberg 테이블을 Databricks에서 읽는 것도 가능합니다.

```sql
-- 외부 Iceberg 테이블을 Unity Catalog에 등록
CREATE TABLE catalog.schema.external_iceberg_table
USING ICEBERG
LOCATION 's3://external-bucket/iceberg-tables/sales/'
TBLPROPERTIES (
    'metadata_location' = 's3://external-bucket/iceberg-tables/sales/metadata/v1.metadata.json'
);

-- 일반 테이블처럼 조회
SELECT * FROM catalog.schema.external_iceberg_table;
```

---

## 실무 가이드: 어떤 포맷을 선택해야 하나요?

| 상황 | 권장 |
|------|------|
| Databricks를 주 플랫폼으로 사용 | **Delta Lake** (기본) + UniForm으로 Iceberg 호환 제공 |
| 멀티 엔진 환경 (Databricks + Snowflake + Trino) | **Delta Lake + UniForm** 활성화 |
| Databricks를 사용하지 않는 환경 | **Apache Iceberg** |
| 기존 Iceberg 테이블을 Databricks에서 분석 | **Foreign Iceberg Tables**로 등록 |

> 💡 **핵심 메시지**: Databricks에서는 Delta Lake를 기본으로 사용하되, UniForm과 Iceberg REST Catalog를 활용하면 외부 엔진과의 호환성을 확보할 수 있습니다. "하나의 데이터, 여러 엔진에서 조회" 전략이 가능합니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Delta Lake** | Databricks의 기본 테이블 포맷입니다. ACID, 타임 트래블 등을 제공합니다 |
| **Apache Iceberg** | 엔진 독립적인 오픈 테이블 포맷입니다. Snowflake, Trino 등에서 사용됩니다 |
| **UniForm** | Delta 테이블을 Iceberg로도 읽을 수 있게 하는 호환성 기능입니다 |
| **Iceberg REST Catalog** | Unity Catalog가 표준 Iceberg REST API를 제공하여 외부 엔진 연동을 지원합니다 |
| **Foreign Iceberg Tables** | 외부 Iceberg 테이블을 Databricks에서 읽는 기능입니다 |

이것으로 **레이크하우스 아키텍처** 섹션을 마치겠습니다. 다음 섹션에서는 코드를 실행하는 [컴퓨트와 워크스페이스](../04-compute-and-workspace/README.md)에 대해 알아보겠습니다.

---

## 참고 링크

- [Databricks: UniForm](https://docs.databricks.com/aws/en/delta/uniform.html)
- [Databricks: Iceberg REST Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/iceberg-rest-catalog.html)
- [Databricks: Read Iceberg tables](https://docs.databricks.com/aws/en/delta/iceberg.html)
- [Apache Iceberg Official](https://iceberg.apache.org/)
- [Databricks Blog: Delta Lake Universal Format](https://www.databricks.com/blog/delta-lake-universal-format-unifrom)
