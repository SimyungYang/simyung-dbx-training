# Delta Lake & Apache Iceberg — 상호 운용성

## 왜 이 주제를 다루나요?

현재 레이크하우스 생태계에서는 두 가지 주요 오픈 테이블 포맷이 경쟁하고 있습니다.

| 포맷 | 주도 | 설명 |
|------|------|------|
| **Delta Lake**| Databricks | Spark 기반, 가장 오래된 레이크하우스 포맷입니다 |
| **Apache Iceberg**| Netflix → Apache Foundation → **Databricks**| 엔진 독립적, 빠르게 채택 확산 중입니다 |

> 💡 **Apache Iceberg와 Databricks의 관계**: Iceberg는 Netflix에서 시작되어 Apache Foundation의 오픈소스 프로젝트로 성장했습니다. **2025년 Databricks가 Iceberg의 핵심 기여 기업인 Tabular를 인수**하면서, Iceberg의 주요 창시자(Ryan Blue 등)가 Databricks에 합류했습니다. 이로 인해 Databricks는 Delta Lake와 Iceberg ** 두 포맷 모두의 핵심 기여자**가 되었으며, UniForm을 통한 두 포맷의 통합을 더욱 강화하고 있습니다. 이것은 "Delta vs Iceberg" 경쟁 구도가 아니라, ** 하나의 데이터에 두 가지 접근 경로를 제공하는 통합**전략입니다.

실무에서는 Databricks(Delta Lake)와 다른 플랫폼(Snowflake, Athena 등 Iceberg 사용)이 공존하는 경우가 많습니다. 이 문서에서는 두 포맷 간의 ** 상호 운용성(Interoperability)**을 어떻게 확보하는지 살펴보겠습니다.

---

## Delta Lake vs Apache Iceberg 비교

| 비교 항목 | Delta Lake | Apache Iceberg |
|-----------|------------|----------------|
| ** 파일 포맷**| Parquet | Parquet (Avro, ORC도 지원) |
| ** 트랜잭션 로그**| JSON 기반 (_delta_log/) | JSON + Avro 기반 (metadata/) |
| **ACID**| ✅ | ✅ |
| ** 타임 트래블**| ✅ | ✅ |
| ** 스키마 진화**| ✅ | ✅ |
| ** 파티션 진화**| CLUSTER BY로 변경 | Hidden Partitioning |
| ** 주요 엔진**| Spark, Databricks | Spark, Trino, Flink, Dremio |
| ** 주도 벤더**| Databricks | Netflix → 오픈소스 커뮤니티 |

두 포맷은 매우 유사한 기능을 제공하며, 실제 데이터는 모두 **Parquet 파일**로 저장됩니다. 차이는 주로 **메타데이터(트랜잭션 로그)를 관리하는 방식**에 있습니다.

---

## UniForm: Delta Lake의 호환성 솔루션

### 개념

> 💡 **UniForm(유니폼)**은 Delta Lake 테이블을 **Iceberg 포맷으로도 읽을 수 있게**해주는 Databricks의 호환성 기능입니다. 하나의 Delta 테이블을 생성하면, Iceberg 클라이언트에서도 동일한 테이블을 읽을 수 있습니다.

| 구성 요소 | Delta 테이블 (원본) | UniForm 적용 후 |
|-----------|-------------------|----------------|
| 메타데이터 | `_delta_log/` (Delta 메타데이터) | `_delta_log/` + `metadata/` (Iceberg 메타데이터 자동 생성) |
| 데이터 파일 | Parquet 파일 (공유) | 동일한 Parquet 파일 (복사 없음) |

핵심은 ** 데이터 파일(Parquet)은 하나**이고, 메타데이터만 Delta와 Iceberg 두 가지 형식으로 유지한다는 것입니다. 따라서 데이터 복사가 발생하지 않습니다.

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
| **Snowflake에서 읽기**| Databricks에서 Delta로 관리하는 테이블을 Snowflake에서 Iceberg로 읽습니다 |
| **AWS Athena에서 쿼리**| S3에 저장된 Delta 테이블을 Athena에서 Iceberg로 조회합니다 |
| ** 멀티 엔진 분석**| Trino, Flink 등 Iceberg를 지원하는 다양한 엔진에서 동일 데이터를 분석합니다 |

---

## Iceberg REST Catalog (IRC)

> 🆕 **Unity Catalog as Iceberg REST Catalog**: Databricks의 Unity Catalog는 **Iceberg REST Catalog**표준을 지원합니다. 이를 통해 외부 엔진(Snowflake, Trino, Apache Flink 등)이 Unity Catalog에 직접 연결하여 Delta 테이블을 Iceberg 테이블처럼 조회할 수 있습니다.

| 엔진 | 프로토콜 | Unity Catalog 역할 |
|------|----------|-------------------|
| Databricks | Delta 프로토콜 | Iceberg REST Catalog로 동작 |
| Snowflake | Iceberg 프로토콜 | 동일한 데이터에 읽기 접근 |
| Apache Spark (외부) | Iceberg 프로토콜 | 동일한 데이터에 읽기 접근 |
| Trino / Presto | Iceberg 프로토콜 | 동일한 데이터에 읽기 접근 |

> 💡 **REST Catalog란?**테이블 포맷의 메타데이터(어떤 테이블이 있고, 어디에 저장되어 있는지)를 HTTP REST API로 관리하는 표준 인터페이스입니다. 이전에는 각 엔진이 자체적으로 메타데이터를 관리했지만, REST Catalog를 통해 ** 하나의 카탈로그로 모든 엔진이 동일한 테이블 목록을 공유**할 수 있게 되었습니다.

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
| Databricks를 주 플랫폼으로 사용 | **Delta Lake**(기본) + UniForm으로 Iceberg 호환 제공 |
| 멀티 엔진 환경 (Databricks + Snowflake + Trino) | **Delta Lake + UniForm**활성화 |
| Databricks를 사용하지 않는 환경 | **Apache Iceberg**|
| 기존 Iceberg 테이블을 Databricks에서 분석 | **Foreign Iceberg Tables**로 등록 |

> 💡 **핵심 메시지**: Databricks에서는 Delta Lake를 기본으로 사용하되, UniForm과 Iceberg REST Catalog를 활용하면 외부 엔진과의 호환성을 확보할 수 있습니다. "하나의 데이터, 여러 엔진에서 조회" 전략이 가능합니다.

---

## Iceberg v3 및 최신 포맷 발전

### Apache Iceberg 버전별 발전

| 버전 | 주요 기능 | 비고 |
|------|----------|------|
| **v1**| 기본 테이블 포맷, 스냅샷, 타임 트래블 | 초기 릴리즈 |
| **v2**| Row-level deletes (Positional Delete, Equality Delete), Sort Order 진화 | 대부분의 엔진이 v2를 지원합니다 |
| **v3**| Multi-argument Transform, Default Value, Variant Type, View 지원 | 2025년 기준 GA 진행 중 |

> 🆕 **Iceberg v3의 핵심 변화**: v3에서는 **Variant 타입**(반정형 데이터 네이티브 지원), **Default Column Values**, **Multi-argument Transform**(파티셔닝 유연성 강화)이 추가되었습니다. Delta Lake의 Variant 타입과 유사한 기능이 Iceberg 표준에도 포함된 것입니다.

### UniForm v2: 양방향 호환성

> 🆕 **UniForm v2**는 기존의 단방향(Delta → Iceberg 읽기)에서 발전하여, **Iceberg 프로토콜로 쓰기**도 가능하게 하는 차세대 호환성 레이어입니다.

| 기능 | UniForm v1 | UniForm v2 |
|------|------------|------------|
| **외부 엔진에서 읽기**| ✅ | ✅ |
| ** 외부 엔진에서 쓰기**| ❌ | ✅ (Iceberg 프로토콜) |
| ** 메타데이터 동기**| 비동기 (약간의 지연) | 동기/준실시간 |
| ** 지원 포맷**| Iceberg | Iceberg + Hudi |

---

## Iceberg REST Catalog (IRC) 아키텍처 심화

### IRC 동작 원리

Unity Catalog가 Iceberg REST Catalog로 동작할 때의 내부 흐름을 이해하면, 외부 엔진 연동 시 트러블슈팅에 도움이 됩니다.

| 단계 | API 호출 | 응답 |
|------|---------|------|
| 1 | `GET /v1/config` | UC: 카탈로그 설정 반환 |
| 2 | `GET /v1/namespaces` | UC: 스키마 목록 반환 |
| 3 | `GET /v1/tables/{table}` | UC: 테이블 메타데이터 반환 (metadata_location, snapshot 정보) |
| 4 | Credential Vending | UC: 임시 클라우드 자격증명 발급 (S3 STS Token / Azure SAS Token) |
| 5 | 데이터 파일 직접 읽기 | 클라우드 스토리지 (S3/ADLS) — Parquet 파일 직접 접근 |

> 외부 엔진 (Snowflake/Trino)이 위 순서로 Iceberg REST Catalog API를 호출합니다.

### Credential Vending (자격증명 발급)

> 💡 **Credential Vending**은 외부 엔진이 데이터 파일에 접근할 때, Unity Catalog가 **일시적인 클라우드 자격증명**을 발급하는 메커니즘입니다. 이를 통해 장기 키(Long-lived Key)를 공유하지 않고도 안전하게 데이터에 접근할 수 있습니다.

| 클라우드 | 발급되는 자격증명 | 유효 기간 | 권한 범위 |
|----------|-----------------|----------|----------|
| **AWS**| STS Temporary Credentials (AssumeRole) | 1시간 (기본) | 해당 테이블의 S3 경로만 접근 가능 |
| **Azure**| SAS Token 또는 Azure AD Token | 1시간 (기본) | 해당 테이블의 ADLS 경로만 접근 가능 |
| **GCP**| OAuth2 Access Token | 1시간 (기본) | 해당 테이블의 GCS 경로만 접근 가능 |

```python
# Credential Vending을 통한 외부 접근 예시 (PyIceberg)
catalog = load_catalog(
    "unity",
    **{
        "type": "rest",
        "uri": "https://<workspace-url>/api/2.1/unity-catalog/iceberg",
        "credential": "<oauth-token>",       # UC 인증
        "warehouse": "<catalog-name>",
        # Credential Vending 자동 처리 — 별도 S3/ADLS 키 불필요
    }
)
# UC가 자동으로 임시 자격증명을 발급하여 데이터 파일에 접근합니다
```

> ⚠️ **보안 이점**: Credential Vending 덕분에 외부 엔진에 클라우드 스토리지의 장기 접근 키를 제공할 필요가 없습니다. UC가 접근을 중개하므로, **UC 권한을 회수하면 즉시 접근이 차단**됩니다.

---

## 외부 엔진에서 Delta 테이블 읽기 패턴

### 엔진별 연동 방법

| 엔진 | 연동 방법 | UniForm 필요 | 비고 |
|------|----------|-------------|------|
| **Snowflake**| Iceberg REST Catalog (IRC) | ✅ UniForm 활성화 | Snowflake의 Iceberg Catalog Integration 사용 |
| **Trino**| Iceberg Connector + IRC | ✅ UniForm 활성화 | `iceberg.catalog-type=rest` 설정 |
| **Apache Spark (외부)**| Delta Sharing 또는 IRC | 양쪽 가능 | Delta Sharing은 UniForm 없이도 사용 가능 |
| **dbt**| Databricks Adapter (네이티브) | ❌ 불필요 | dbt-databricks 어댑터가 Delta를 직접 지원 |
| **Presto**| Iceberg Connector + IRC | ✅ UniForm 활성화 | Trino와 유사한 설정 |
| **Apache Flink**| Iceberg Connector + IRC | ✅ UniForm 활성화 | 실시간 스트리밍 읽기 가능 |
| **DuckDB** | delta-rs (Delta 네이티브) 또는 PyIceberg | 양쪽 가능 | 로컬 분석에 적합 |

### Snowflake에서 연동하는 예시

```sql
-- Snowflake에서 Databricks UC를 Iceberg Catalog로 등록
CREATE OR REPLACE CATALOG INTEGRATION databricks_uc_catalog
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  CATALOG_NAMESPACE = 'default'
  REST_CONFIG = (
    CATALOG_URI = 'https://<workspace-url>/api/2.1/unity-catalog/iceberg'
    WAREHOUSE = '<catalog-name>'
  )
  REST_AUTHENTICATION = (
    TYPE = OAUTH
    OAUTH_CLIENT_ID = '<client-id>'
    OAUTH_CLIENT_SECRET = '<client-secret>'
    OAUTH_ALLOWED_SCOPES = ('all-apis')
    OAUTH_TOKEN_URI = 'https://<workspace-url>/oidc/v1/token'
  );

-- Databricks의 Delta 테이블을 Iceberg로 읽기
CREATE ICEBERG TABLE snowflake_db.schema.orders
  CATALOG = 'databricks_uc_catalog'
  EXTERNAL_VOLUME = 'my_external_volume'
  CATALOG_TABLE_NAME = 'schema.orders';

SELECT * FROM snowflake_db.schema.orders LIMIT 100;
```

### Trino에서 연동하는 예시

```properties
# Trino catalog 설정 (etc/catalog/databricks.properties)
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=https://<workspace-url>/api/2.1/unity-catalog/iceberg
iceberg.rest-catalog.warehouse=<catalog-name>
iceberg.rest-catalog.security=OAUTH2
iceberg.rest-catalog.oauth2.token=<access-token>
```

---

## 성능 비교 및 최적화

### Delta Lake vs Iceberg 성능 특성

| 성능 항목 | Delta Lake (Databricks) | Apache Iceberg (오픈소스 엔진) |
|-----------|------------------------|-------------------------------|
| **읽기 성능 (Scan)**| Photon 엔진 활용으로 매우 빠름 | 엔진에 따라 다름 (Trino 등) |
| ** 쓰기 성능**| Optimized Write + Auto Compaction | Compaction 직접 관리 필요 |
| ** 소규모 파일 문제**| Auto Compaction으로 자동 해결 | Rewrite Data Files API로 수동 관리 |
| ** 메타데이터 성능**| Delta Log Checkpoint (10 커밋마다) | Manifest List → Manifest File 2단계 |
| ** 파티션 진화**| Liquid Clustering (CLUSTER BY) | Hidden Partitioning (자동 파티션) |
| **Z-Order/정렬**| Z-ORDER BY 또는 Liquid Clustering | Sort Order 진화 |

### UniForm 사용 시 성능 고려사항

| 항목 | 영향 | 설명 |
|------|------|------|
| ** 쓰기 오버헤드**| 5~15% 증가 | Iceberg 메타데이터를 추가로 생성해야 합니다 |
| ** 스토리지 오버헤드**| 매우 작음 (< 1%) | Parquet 데이터 파일은 공유, 메타데이터만 추가 |
| ** 읽기 성능 (Delta)**| 영향 없음 | Delta 프로토콜로 읽을 때는 성능 차이 없습니다 |
| ** 읽기 성능 (Iceberg)**| Delta 대비 약간 느림 | Iceberg 메타데이터 경유 + Credential Vending 오버헤드 |
| ** 메타데이터 동기 지연**| 수 초 ~ 수십 초 | Delta 쓰기 후 Iceberg 메타데이터 생성까지 약간의 지연 |

> 💡 **UniForm 쓰기 오버헤드 최소화**: UniForm은 **비동기적**으로 Iceberg 메타데이터를 생성합니다. 따라서 쓰기 작업의 크리티컬 패스에 미치는 영향은 제한적입니다. 다만, 매우 빈번한 소규모 쓰기(초당 수십 커밋)가 발생하는 경우에는 오버헤드가 누적될 수 있으므로 주의가 필요합니다.

---

## 엔터프라이즈 멀티 플랫폼 전략

### 전형적인 엔터프라이즈 데이터 아키텍처

대규모 조직에서는 여러 데이터 플랫폼이 공존하는 것이 현실입니다. 이때의 권장 패턴을 정리합니다.

| 패턴 | 아키텍처 | 적합한 경우 |
|------|----------|-----------|
| **Databricks 중심**| Databricks(Delta) + UniForm → 외부 엔진 읽기 전용 | Databricks가 주 플랫폼이고, 일부 팀이 Snowflake/Trino 사용 |
| **Snowflake 공존**| Databricks: ETL/ML, Snowflake: BI/분석 → IRC로 데이터 공유 | 양쪽 투자가 모두 큰 경우 |
| **Open Lakehouse**| Delta Lake(오픈소스) + 멀티 엔진 → IRC로 통합 카탈로그 | 벤더 종속을 최소화하려는 경우 |
| ** 마이그레이션 중**| 기존 Iceberg → Databricks로 점진적 전환 | Foreign Iceberg Tables로 기존 데이터 유지하며 전환 |

### 비용 최적화 관점

| 전략 | 설명 | 예상 효과 |
|------|------|----------|
| ** 데이터 복사 제거**| UniForm/IRC로 같은 데이터를 여러 엔진에서 읽기 | 스토리지 비용 50%+ 절감 |
| **ETL 파이프라인 통합**| 여러 포맷으로의 변환 파이프라인 제거 | 컴퓨트 비용 절감 + 운영 복잡도↓ |
| ** 카탈로그 통합**| UC를 중앙 카탈로그로, IRC로 외부 엔진 연동 | 거버넌스 비용↓, 정합성↑ |

---

## Edge Case와 주의사항

| 주의사항 | 설명 |
|---------|------|
| **UniForm은 읽기 전용**| 외부 엔진에서 UniForm 테이블에 ** 쓰기는 불가**합니다 (v1 기준). 쓰기는 반드시 Databricks/Delta 프로토콜로 수행해야 합니다 |
| **일부 Delta 기능 미지원** | Deletion Vectors, Column Mapping 등 고급 Delta 기능은 Iceberg 메타데이터에 완전히 반영되지 않을 수 있습니다 |
| **메타데이터 동기 타이밍** | Delta 커밋과 Iceberg 메타데이터 생성 사이에 수 초의 지연이 있어, 실시간성이 극도로 중요한 경우 주의가 필요합니다 |
| **Foreign Iceberg 제약**| Databricks에서 Foreign Iceberg 테이블은 ** 읽기 전용**이며, DML(INSERT/UPDATE/DELETE)은 지원되지 않습니다 |
| **스키마 진화 호환성** | Delta에서 스키마를 변경하면 Iceberg 메타데이터에도 반영되지만, 복잡한 중첩 스키마 변경은 호환성 확인이 필요합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Delta Lake**| Databricks의 기본 테이블 포맷입니다. ACID, 타임 트래블 등을 제공합니다 |
| **Apache Iceberg**| 엔진 독립적인 오픈 테이블 포맷입니다. Snowflake, Trino 등에서 사용됩니다 |
| **UniForm**| Delta 테이블을 Iceberg로도 읽을 수 있게 하는 호환성 기능입니다 |
| **Iceberg REST Catalog**| Unity Catalog가 표준 Iceberg REST API를 제공하여 외부 엔진 연동을 지원합니다 |
| **Credential Vending**| 외부 엔진에 임시 클라우드 자격증명을 발급하여 안전한 데이터 접근을 보장합니다 |
| **Foreign Iceberg Tables**| 외부 Iceberg 테이블을 Databricks에서 읽는 기능입니다 |

이것으로 ** 레이크하우스 아키텍처** 섹션을 마치겠습니다. 다음 섹션에서는 코드를 실행하는 [컴퓨트와 워크스페이스](../04-compute-and-workspace/README.md)에 대해 알아보겠습니다.

---

## 참고 링크

- [Databricks: UniForm](https://docs.databricks.com/aws/en/delta/uniform.html)
- [Databricks: Iceberg REST Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/iceberg-rest-catalog.html)
- [Databricks: Read Iceberg tables](https://docs.databricks.com/aws/en/delta/iceberg.html)
- [Apache Iceberg Official](https://iceberg.apache.org/)
- [Databricks Blog: Delta Lake Universal Format](https://www.databricks.com/blog/delta-lake-universal-format-unifrom)
