# 레이크하우스란?

## 왜 레이크하우스가 등장했나요?

[이전 섹션](../01-data-fundamentals/data-warehouse-vs-data-lake.md)에서 데이터 웨어하우스와 데이터 레이크의 장단점을 살펴보았습니다. 이 두 가지 접근법에는 각각 분명한 한계가 있었습니다.

### 기존 방식의 문제점

| 기존 2-Platform 아키텍처 | 문제점 |
|--------------------------|--------|
| 소스 데이터 → 데이터 레이크 (원본 저장, ML 학습용) | 데이터 중복 저장 |
| 데이터 레이크 → ETL → 데이터 웨어하우스 (정제, BI/SQL 분석용) | 복사 비용, 데이터 불일치 |

*출처: [Databricks Docs](https://docs.databricks.com)*

| 문제점 | 설명 |
|--------|------|
| **데이터 중복** | 동일한 데이터가 레이크와 웨어하우스에 이중으로 저장되어 스토리지 비용이 증가합니다 |
| **데이터 불일치** | 복사 과정에서 시간 차이가 발생하여 두 시스템의 데이터가 달라질 수 있습니다 |
| **복잡한 관리** | 두 개의 시스템을 각각 관리하고, 사이의 ETL 파이프라인도 관리해야 합니다 |
| **거버넌스 분산** | 레이크와 웨어하우스에서 각각 별도로 권한을 관리해야 합니다 |
| **높은 비용** | 스토리지 중복 + 컴퓨팅 중복 + 관리 인력 중복 |

### 레이크하우스의 해결 방법

> 💡 **레이크하우스(Lakehouse)** 란 **데이터 레이크의 유연성과 저비용**에 **데이터 웨어하우스의 성능과 안정성**을 결합한 차세대 데이터 아키텍처입니다.

![Lakehouse Architecture](https://docs.databricks.com/aws/en/assets/images/object-hierarchy-966dee8fa239626134ed6b7278beab2c.png)

*출처: [Databricks Docs](https://docs.databricks.com)*

쉽게 말하면, 레이크하우스는 **"데이터 레이크 위에 웨어하우스의 기능을 얹은 것"**입니다.

---

## 레이크하우스의 핵심 원리

레이크하우스가 "최고의 장점만 결합"할 수 있는 비결은 다음 세 가지 기술적 혁신에 있습니다.

### 1. 오픈 포맷 기반 저장

데이터를 Parquet 같은 **오픈 파일 포맷**으로 클라우드 오브젝트 스토리지에 저장합니다. 특정 벤더의 독점 포맷이 아니기 때문에 어떤 도구로도 읽을 수 있습니다.

> 💡 **Parquet(파케이)란?** Apache에서 개발한 오픈소스 **컬럼 기반 파일 포맷**입니다. 행 단위가 아니라 열 단위로 데이터를 저장하기 때문에, 분석 쿼리에서 필요한 컬럼만 효율적으로 읽을 수 있어 매우 빠르고 저장 공간도 적게 차지합니다. 현재 빅데이터 생태계에서 가장 널리 사용되는 파일 포맷 중 하나입니다.

### 2. 트랜잭션 레이어 (Delta Lake)

오픈 파일 포맷 위에 **트랜잭션 관리 레이어**를 추가합니다. 이것이 바로 **Delta Lake**의 역할입니다. 데이터 레이크에 없던 ACID 트랜잭션, 스키마 관리, 타임 트래블 등의 기능을 제공합니다. (다음 문서에서 자세히 다룹니다.)

### 3. 고성능 쿼리 엔진

데이터 레이크 위에서도 데이터 웨어하우스 수준의 SQL 쿼리 성능을 제공하는 최적화된 엔진을 사용합니다. Databricks에서는 **Photon** 엔진이 이 역할을 수행합니다.

> 💡 **Photon이란?** Databricks가 개발한 **차세대 쿼리 엔진**입니다. C++로 작성되어 기존 Spark SQL 엔진보다 최대 수 배 빠른 성능을 제공합니다. 특별한 설정 없이 Photon이 활성화된 클러스터를 사용하면 자동으로 적용됩니다.

---

## 레이크하우스 vs 기존 방식 비교

| 비교 항목 | 데이터 레이크 | 데이터 웨어하우스 | 레이크하우스 |
|-----------|-------------|-----------------|------------|
| **데이터 유형** | 모든 유형 | 정형만 | 모든 유형 ✅ |
| **ACID 트랜잭션** | ❌ | ✅ | ✅ |
| **SQL 성능** | 느림 | 빠름 | 빠름 ✅ |
| **ML/AI 지원** | ✅ | 제한적 | ✅ |
| **스키마 관리** | 없음 | 강력 | 강력 ✅ |
| **저장 비용** | 저렴 | 비쌈 | 저렴 ✅ |
| **거버넌스** | 약함 | 강력 | 강력 ✅ |
| **오픈 포맷** | ✅ | ❌ (보통 독점) | ✅ |

---

## Databricks 레이크하우스의 구성 요소

아래는 Databricks 공식 문서의 레이크하우스 아키텍처 다이어그램입니다.

![Databricks Lakehouse Architecture](https://docs.databricks.com/aws/en/assets/images/lakehouse-diagram-865ba5c041f60df99ff6bee1ebaad26d.png)

*출처: [Databricks 공식 문서 — What is a data lakehouse?](https://docs.databricks.com/aws/en/lakehouse/)*

Databricks의 레이크하우스는 다음 요소들로 구성됩니다.

| 레이어 | 구성 요소 | 역할 |
|--------|-----------|------|
| **스토리지** | 클라우드 스토리지 + Delta Lake | 데이터를 오픈 포맷으로 저렴하게 저장하면서도, 트랜잭션과 스키마를 관리합니다 |
| **컴퓨팅** | Spark + Photon | 대규모 데이터를 빠르게 처리하고, SQL 쿼리 성능을 극대화합니다 |
| **분석** | SQL, ML, BI | 하나의 데이터로 SQL 분석, ML 모델 학습, 대시보드를 모두 수행합니다 |
| **거버넌스** | Unity Catalog | 모든 레이어에 걸쳐 통합적인 접근 제어와 감사를 수행합니다 |

---

## 레이크하우스의 실제 장점 사례

### 사례 1: 데이터 복사 제거

**기존**: 데이터 레이크에 원본 저장 → ETL → 웨어하우스에 복사 → BI 도구에서 조회
**레이크하우스**: 데이터 레이크에 Delta Lake로 저장 → SQL로 바로 조회 (복사 불필요!)

### 사례 2: ML과 BI의 통합

**기존**: 분석가는 웨어하우스에서 SQL, 데이터 과학자는 레이크에서 Python → 서로 다른 데이터를 봄
**레이크하우스**: 동일한 Delta 테이블에서 SQL 분석과 ML 학습을 모두 수행 → 일관된 데이터

### 사례 3: 실시간 + 배치 통합

**기존**: 배치 파이프라인과 스트리밍 파이프라인을 별도 시스템에서 운영
**레이크하우스**: Spark Structured Streaming으로 실시간 데이터를 Delta 테이블에 적재 → 같은 테이블에서 배치 분석도 수행

---

## 업계 동향: 레이크하우스의 보편화

레이크하우스 아키텍처는 Databricks가 처음 제안한 개념이지만, 현재는 업계 전반에서 채택되고 있는 추세입니다.

| 벤더 | 레이크하우스 관련 제품 |
|------|---------------------|
| **Databricks** | Delta Lake 기반 Lakehouse Platform (선도) |
| **Snowflake** | Iceberg Tables, Unistore |
| **AWS** | S3 + Glue + Athena (레이크하우스 패턴) |
| **Microsoft** | OneLake (Microsoft Fabric) |
| **Google** | BigLake |

> 💡 **Apache Iceberg란?** Delta Lake와 유사한 목적을 가진 또 다른 오픈소스 테이블 포맷입니다. Netflix가 주도하여 개발했으며, 현재 많은 데이터 플랫폼에서 지원하고 있습니다. Databricks는 Delta Lake를 기본으로 사용하면서도, **UniForm** 기능을 통해 Iceberg 호환성을 제공하고 있어, 외부 엔진에서도 Delta 테이블을 Iceberg 테이블처럼 읽을 수 있습니다.

---

## 심화: 레이크하우스의 한계와 실전 트레이드오프

레이크하우스가 만능은 아닙니다. Principal SA로서 고객에게 정직하게 전달해야 할 한계와 트레이드오프를 살펴보겠습니다.

### 레이크하우스의 한계

| 한계 영역 | 상세 | 해결 방향 |
|----------|------|----------|
| **OLTP 워크로드** | 레이크하우스는 분석(OLAP)에 최적화되어 있습니다. 웹 앱의 건 단위 INSERT/UPDATE(sub-ms 지연시간)에는 부적합합니다 | **Lakebase**로 해결: PostgreSQL 호환 OLTP를 레이크하우스와 통합합니다 |
| **실시간 저지연(<100ms)** | Structured Streaming도 마이크로배치 기반이므로 수백 ms~수 초의 지연이 발생합니다. Delta Lake의 commit 오버헤드도 있습니다 | Kafka/Flink 조합으로 실시간 레이어를 별도 구성하거나, Lakebase + Change Data Capture를 활용합니다 |
| **동시성 제한** | Delta Lake의 Optimistic Concurrency Control은 동일 파일에 대한 동시 쓰기 충돌 시 재시도가 필요합니다 | 파티셔닝으로 쓰기 충돌을 분산하고, `delta.isolationLevel`을 `WriteSerializable`로 설정합니다 |
| **소규모 데이터** | 수 MB~GB 수준의 데이터는 오히려 PostgreSQL이나 MySQL이 더 효율적입니다. 레이크하우스의 오버헤드(메타스토어, 파일 관리)가 과합니다 | 데이터 규모가 수십 GB 이상일 때 레이크하우스의 이점이 본격적으로 나타납니다 |
| **그래프 쿼리** | 관계형/컬럼형 스토리지 기반이므로, 그래프 탐색(친구의 친구, 최단 경로 등)에는 비효율적입니다 | Neo4j 등 전용 그래프 DB와 연동합니다 |

---

### 멀티 클라우드 레이크하우스 전략

엔터프라이즈 고객의 80% 이상이 2개 이상의 클라우드를 사용합니다. 멀티 클라우드 환경에서의 레이크하우스 전략은 다음과 같습니다.

| 과제 | 상세 | Databricks 해법 |
|------|------|----------------|
| **데이터 레지던시** | GDPR(EU), 개인정보보호법(한국) 등으로 데이터가 특정 리전을 벗어날 수 없습니다 | 리전별 Workspace를 분리하고, Unity Catalog Metastore를 리전별로 운영합니다 |
| **크로스 클라우드 거버넌스** | AWS와 Azure의 Workspace에서 동일한 거버넌스 정책을 적용해야 합니다 | Unity Catalog의 Account-level 관리로 복수 클라우드의 Workspace를 단일 계정에서 통합 관리합니다 |
| **데이터 공유** | AWS의 데이터를 Azure에서 분석해야 하는 경우 | **Delta Sharing**으로 크로스 클라우드 공유합니다. 데이터 복사 없이 읽기 권한만 부여합니다 |
| **재해 복구** | 한 클라우드 리전 전체 장애 시 복구 | Deep Clone으로 다른 리전/클라우드에 복제본을 유지합니다. RPO/RTO는 클론 주기에 의존합니다 |

> 💡 **실무 조언**: 멀티 클라우드는 비용과 복잡성이 크게 증가합니다. "정말 필요한가?"를 먼저 검증하세요. 규제 요건이 아니라면 단일 클라우드 + 멀티 리전이 대부분의 경우 더 효율적입니다.

---

### 거버넌스 부채 (Governance Debt) 관리

기존 데이터 웨어하우스(Teradata, Oracle, Netezza 등)에서 레이크하우스로 마이그레이션할 때, 거버넌스 정책 마이그레이션이 가장 큰 과제입니다.

| 거버넌스 항목 | 기존 DW | 레이크하우스 (Unity Catalog) | 마이그레이션 전략 |
|-------------|---------|--------------------------|----------------|
| **접근 제어** | GRANT/REVOKE (DB 레벨) | Unity Catalog GRANT (3-level namespace) | 기존 권한 매핑 스크립트 작성, `INFORMATION_SCHEMA`에서 기존 권한 추출 |
| **데이터 마스킹** | DB 내장 함수, View | Row Filter + Column Mask | 마스킹 정책을 SQL 함수로 재작성 |
| **감사 로그** | DB 자체 감사 | `system.access.audit` 테이블 | 기존 감사 보고서 쿼리를 시스템 테이블 기반으로 변환 |
| **데이터 분류** | 수동 태깅 또는 없음 | UC Tags + AI Classification | 마이그레이션 시점에 민감도 분류 일괄 적용 |
| **리니지** | 없음 또는 외부 도구 | UC 자동 리니지 | 마이그레이션 후 자동 수집 시작 |

> ⚠️ **실무 경험**: 거버넌스 마이그레이션을 데이터 마이그레이션 이후로 미루면 "거버넌스 부채"가 눈덩이처럼 불어납니다. **Day 1부터 Unity Catalog를 활성화**하고, 데이터 마이그레이션과 거버넌스 마이그레이션을 동시에 진행하는 것이 핵심입니다.

---

### 비용 모델 분석

레이크하우스의 비용 구조를 정확히 이해하면, 기존 DW 대비 40~70%의 비용 절감이 가능합니다.

#### 스토리지 계층별 비용 (AWS 기준, 2025)

| 스토리지 계층 | 월 비용/TB | 접근 빈도 | 레이크하우스 활용 |
|-------------|----------|----------|-----------------|
| **S3 Standard** | ~$23/TB | 자주 접근 (Gold) | Hot 데이터, 최근 파티션 |
| **S3 Intelligent-Tiering** | $23~2.3/TB | 자동 분류 | 대부분의 Silver/Gold 테이블에 권장 |
| **S3 Glacier Instant Retrieval** | ~$4/TB | 분기 1회 | Bronze 히스토리, 규제 보존 데이터 |
| **S3 Glacier Deep Archive** | ~$1/TB | 연 1회 | 7년 이상 보존 의무 데이터 |

#### 컴퓨트-스토리지 분리의 비용 이점

```
[기존 DW: 컴퓨트+스토리지 결합]
├─ 피크 시간 기준으로 프로비저닝 → 70%는 유휴 상태
├─ 스토리지 확장 = 컴퓨트도 함께 확장 (불필요한 비용)
└─ 연간 라이선스: 수억~수십억 원

[레이크하우스: 컴퓨트+스토리지 분리]
├─ 스토리지: S3/ADLS에 독립 저장 ($23/TB/월)
├─ 컴퓨트: 필요할 때만 시작, 사용한 만큼 과금 (DBU)
├─ 오토스케일링: 피크 시 확장, 비피크 시 축소
└─ Serverless: 프로비저닝 없이 즉시 실행, 유휴 비용 0
```

#### Serverless의 경제성

| 워크로드 패턴 | Classic 클러스터 | Serverless | 절감률 |
|-------------|----------------|-----------|-------|
| **배치 ETL (1일 2시간)** | 24시간 프로비저닝 or 시작 대기 시간 | 실행 시간만 과금 | 50~70% |
| **애드훅 쿼리 (간헐적)** | 최소 1노드 상시 대기 | 쿼리 실행 시간만 과금 | 60~80% |
| **피크 타임 대응** | 수동 스케일업 (10분+ 소요) | 즉시 확장 (초 단위) | 가용성 향상 |

> 💡 **비용 최적화 공식**: `총 비용 = (스토리지 비용) + (컴퓨트 DBU x 단가) + (네트워크 egress)`. 대부분의 경우 컴퓨트가 70~80%를 차지합니다. **Serverless 전환, 클러스터 자동 종료 정책(idle 10분), Photon 활성화**가 가장 효과적인 비용 절감 수단입니다.

---

### 경쟁사 비교 심화

| 비교 항목 | Databricks Lakehouse | Snowflake | Microsoft Fabric | Google BigLake |
|----------|---------------------|-----------|-----------------|---------------|
| **테이블 포맷** | Delta Lake (UniForm으로 Iceberg 호환) | Iceberg Tables (자체 포맷 + Iceberg) | Delta Lake (OneLake) | BigLake (다중 포맷) |
| **오픈소스 기반** | ✅ Apache Spark, Delta Lake, MLflow | ❌ 독점 엔진 | 부분적 (Delta Lake은 오픈) | 부분적 |
| **벤더 종속** | 낮음 (오픈 포맷, 포터블) | 높음 (데이터 export 필요) | 중간 (Azure 종속) | 중간 (GCP 종속) |
| **ML/AI 통합** | ✅ 네이티브 (MLflow, Feature Store, Model Serving) | 제한적 (Snowpark ML) | 중간 (Azure ML 연동) | 중간 (Vertex AI 연동) |
| **실시간 스트리밍** | ✅ Structured Streaming | ❌ Snowpipe (마이크로배치만) | 중간 (Eventstream) | 중간 (Dataflow) |
| **멀티 클라우드** | ✅ AWS/Azure/GCP 모두 지원 | ✅ AWS/Azure/GCP | ❌ Azure 전용 | ❌ GCP 전용 |
| **거버넌스** | Unity Catalog (통합) | Horizon (자체) | Purview 연동 | Dataplex |
| **가격 모델** | DBU (사용량 기반) | Credit (사용량 기반) | CU (용량 기반) | Slot (용량 기반) |
| **OLTP 통합** | ✅ Lakebase (PostgreSQL) | ✅ Unistore (제한적 OLTP) | ❌ 별도 Azure DB | ❌ 별도 Cloud SQL |

> 💡 **Databricks의 핵심 차별점**: (1) **오픈소스 기반**으로 벤더 종속이 가장 낮습니다. (2) **ML/AI 워크로드가 네이티브**로 통합되어 있습니다. (3) **멀티 클라우드 지원**이 가장 성숙합니다. Snowflake는 SQL/BI에 강하고, Fabric은 Microsoft 생태계 내에서 강점을 가집니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **레이크하우스** | 데이터 레이크의 유연성 + 웨어하우스의 안정성을 결합한 아키텍처입니다 |
| **Delta Lake** | 레이크하우스를 가능하게 하는 핵심 기술. 오픈 파일 포맷 위에 트랜잭션을 추가합니다 |
| **Photon** | Databricks의 고성능 쿼리 엔진으로, 웨어하우스 수준의 SQL 성능을 제공합니다 |
| **오픈 포맷** | Parquet 등 독점 포맷이 아닌 오픈 포맷을 사용하여 벤더 종속을 방지합니다 |
| **단일 플랫폼** | 하나의 데이터로 BI, ML, 스트리밍 등 모든 워크로드를 지원합니다 |

다음 문서에서는 레이크하우스의 핵심 기술인 **Delta Lake**를 자세히 살펴보겠습니다.

---

## 참고 링크

- [Databricks: What is a Data Lakehouse?](https://docs.databricks.com/aws/en/lakehouse/)
- [Azure Databricks: What is a data lakehouse?](https://learn.microsoft.com/en-us/azure/databricks/lakehouse/)
- [Databricks Blog: What is a Lakehouse?](https://www.databricks.com/blog/2020/01/30/what-is-a-data-lakehouse.html)
- [Delta Lake Official Site](https://delta.io/)
