# Databricks vs 경쟁사

## 1. 시장 컨텍스트 — 왜 비교가 중요한가?

2024~2026년 현재, 데이터 플랫폼 시장은 **"모던 데이터 스택(Modern Data Stack)"** 에서 **"데이터 인텔리전스 플랫폼(Data Intelligence Platform)"** 으로 빠르게 진화하고 있습니다.

과거에는 데이터 웨어하우스(DW), 데이터 레이크(Data Lake), ML 플랫폼이 각각 별도 도구로 분리되어 있었습니다. 그러나 AI/ML 워크로드가 전사적으로 확산되면서, 조직들은 다음과 같은 과제에 직면하게 되었습니다.

- **데이터 사일로 (Data Silo)**: ETL 팀과 분석 팀, ML 팀이 서로 다른 플랫폼을 사용하며 데이터가 분산
- **거버넌스 공백**: 여러 플랫폼에 걸친 데이터 접근 추적과 계보(Lineage) 관리 어려움
- **운영 비용 증가**: 여러 플랫폼의 운영비와 라이선스 중복

이 맥락에서 **Databricks**, **Snowflake**, **Microsoft Fabric**, **AWS 네이티브 스택** 등이 각자의 방향으로 통합 플랫폼화를 추진하고 있습니다. 따라서 플랫폼 선택은 단순한 기술 비교가 아니라, **조직의 데이터 전략 방향과의 정합성** 을 따져야 하는 결정입니다.

---

## 2. 핵심 차이 3가지 (심화)

### 차이 1: 오픈소스 기반 vs 독자 포맷

| 플랫폼 | 데이터 포맷 | 벤더 락인 수준 |
|--------|------------|----------------|
| **Databricks** | Delta Lake (오픈소스, Parquet 기반) | 낮음 — 데이터를 S3/ADLS에 직접 소유합니다 |
| **Snowflake** | 독자 포맷 (내부 스토리지) | 높음 — 데이터를 꺼내려면 COPY INTO로 내보내야 합니다 |
| **BigQuery** | 독자 포맷 (Capacitor) | 높음 — 데이터가 Google 내부에 저장됩니다 |
| **Microsoft Fabric** | OneLake (Delta/Parquet 기반, 오픈 지향) | 중간 — Azure 종속성은 있으나 포맷은 개방적 |

**포맷 전쟁의 현재 상황:**

- **Delta Lake**: Linux Foundation 오픈소스 프로젝트. ACID 트랜잭션, CDC(Change Data Capture), DML(UPDATE/DELETE/MERGE) 지원. Databricks Runtime에서 가장 최적화됨
- **Apache Iceberg**: Netflix에서 시작된 오픈소스 테이블 포맷. Snowflake, AWS도 지원을 확대 중. 대규모 파티션 관리와 스키마 진화(Schema Evolution)에 강점
- **UniForm (Universal Format)**: Databricks가 2024년에 발표한 기능으로, 동일한 데이터를 Delta, Iceberg, Hudi 형식으로 동시에 노출. 포맷 간 상호운용성 확보

> **핵심 포인트:**Databricks는 Delta Lake를 통해 "데이터는 고객 소유, 처리는 Databricks에서" 모델을 지향합니다. 3년 후 플랫폼을 변경하더라도 데이터는 S3/ADLS의 Parquet 파일 그대로이므로, 다른 도구에서 즉시 읽을 수 있습니다.

---

### 차이 2: 통합 플랫폼 vs 특화 도구

각 플랫폼이 최근 영역을 확장하고 있지만, 출발점과 강점의 차이는 여전히 유효합니다.

| 플랫폼 | 출발점 | 최근 확장 | 잔존 약점 |
|--------|--------|----------|-----------|
| **Databricks** | 데이터 엔지니어링 + Spark | SQL 강화 (DBT native, Serverless SQL), AI/BI 대시보드, Lakebase (OLTP), AI Agent Framework | SQL 편의성이 Snowflake보다 다소 뒤처진 측면 있음 |
| **Snowflake** | SQL 분석 DW | Snowpark ML, Snowflake Cortex (LLM), Iceberg 지원, Streamlit 내장 | Python/Spark 워크로드는 여전히 제한적, 스트리밍은 성숙도 낮음 |
| **BigQuery** | 서버리스 SQL 분석 | Vertex AI 통합, BigQuery ML, Dataform 내장 | 세밀한 인프라 제어 어려움, 스트리밍 비용 높음 |
| **AWS 네이티브** | 클라우드 인프라 | SageMaker, Glue 4.0, Redshift Serverless | 각 서비스를 직접 조합해야 해 운영 복잡도 높음 |

**주목할 트렌드:**Snowflake는 "SQL 회사에서 AI 회사로" 전환을 선언했고, Databricks는 역으로 "엔지니어링 회사에서 분석/AI 플랫폼으로" 영역을 확대 중입니다. 두 플랫폼이 서로의 영역을 공략하며 점점 유사해지고 있으나, **아키텍처 철학의 차이(섹션 4 참조)** 는 여전히 중요합니다.

---

### 차이 3: Spark 네이티브 성능 최적화

Databricks는 Apache Spark를 창시한 팀(UC Berkeley AMPLab)이 설립한 회사입니다. 이 차이는 단순한 마케팅이 아니라 기술적 깊이에서 나타납니다.

| 항목 | 오픈소스 Spark (EMR 등) | Databricks Runtime (DBR) |
|------|------------------------|--------------------------|
| SQL 엔진 | Catalyst 옵티마이저 (JVM) | **Photon**— C++로 재작성된 벡터화 실행 엔진, SQL 워크로드 2~8배 빠름 |
| I/O 최적화 | 기본 파일 스캔 | **Predictive I/O**— 읽기 패턴을 학습해 사전 로드, **Data Skipping**— 파일 수준 통계로 불필요한 파일 제외 |
| 인덱싱 | 없음 (파티션만) | **Z-Order**(다차원 클러스터링), **Liquid Clustering**(2024년, 자동 최적화 클러스터링) |
| 쿼리 최적화 | 수동 `ANALYZE TABLE` | **Adaptive Query Execution (AQE)**— 런타임 실행 계획 재최적화, Enhanced AQE with Runtime Statistics |
| 스트리밍 | Structured Streaming | **Enhanced Structured Streaming**+ **DLT (Delta Live Tables)**— 선언형 파이프라인 |
| ML 통합 | 별도 라이브러리 설치 | MLflow 내장, Feature Store, Model Serving 일체형 |

> **Photon과 DBR의 차이:**Photon은 JVM 기반 Spark가 아닌 C++ 네이티브 코드로 실행됩니다. SIMD 인스트럭션을 활용한 벡터화 처리로, 특히 집계(aggregation)와 조인(join) 연산에서 획기적인 성능을 발휘합니다.

---

## 3. 기능별 상세 비교표

| 기능 영역 | Databricks | Snowflake | AWS 네이티브 | Microsoft Fabric |
|-----------|-----------|-----------|-------------|-----------------|
| **데이터 포맷** | Delta Lake (오픈소스) | 독자 포맷 + Iceberg 지원 | S3 + Iceberg/Delta 선택적 | OneLake (Delta 기반) |
| **SQL 분석** | Serverless SQL Warehouse, AI/BI | 최상급 SQL, 자동 스케일링 | Redshift (좋음) / Athena (서버리스) | Direct Lake 모드 |
| **데이터 엔지니어링** | Spark, DLT, Workflows | Snowpark, Tasks, Streams | Glue, Step Functions | Data Factory, Pipelines |
| **ML/AI** | MLflow, Feature Store, Model Serving, AI Agent | Cortex ML, Snowpark ML | SageMaker (업계 최고 수준) | Azure ML 연동 |
| **스트리밍** | Structured Streaming, Kafka 직접 연동 | Snowpipe Streaming (제한적) | Kinesis + Glue Streaming | Eventhouse (KQL) |
| **거버넌스** | Unity Catalog (통합 카탈로그, Lineage 포함) | Snowflake Access Control | AWS Lake Formation + Glue Catalog | Microsoft Purview 연동 |
| **보안** | 행/열 수준 보안, Dynamic View, Private Link | 행/열 수준 보안, Tri-Secret Secure | IAM + Lake Formation, VPC Endpoint | Entra ID, Private Link |
| **가격 모델** | DBU(Databricks Unit) 소비 기반, Serverless 옵션 | Credit 소비 기반, 스토리지 별도 | 각 서비스별 독립 과금 | Microsoft Capacity (F SKU) |
| **멀티클라우드** | AWS, Azure, GCP 동일 경험 | AWS, Azure, GCP 지원 | AWS 전용 | Azure 중심 (Multi-cloud 제한) |
| **오픈소스 친화성** | 높음 (Delta, MLflow, Spark 기여) | 중간 (Iceberg 지원 추가) | 중간 (EMR은 높음) | 중간 |

---

## 4. Databricks vs Snowflake 심화

가장 빈번하게 비교되는 두 플랫폼입니다. 단순한 기능 비교를 넘어 **아키텍처 철학의 차이** 를 이해하는 것이 중요합니다.

### 아키텍처 철학 비교

**Snowflake의 철학:**"데이터 웨어하우스의 클라우드 재발명"
- 컴퓨팅과 스토리지의 완전 분리, 하지만 데이터는 Snowflake 내부 스토리지에 보관
- SQL 중심의 단일 인터페이스, 사용의 단순함과 관리형 경험 우선
- 내부적으로 마이크로파티션(Micro-partition) 구조로 자동 최적화

**Databricks의 철학:**"데이터 레이크하우스(Lakehouse)"
- 데이터는 고객 클라우드 스토리지(S3/ADLS/GCS)에 직접 저장
- 오픈 포맷(Delta Lake) 위에서 DW와 ML을 동시에 처리
- 다양한 언어(Python, SQL, Scala, R)와 워크로드(배치, 스트리밍, ML)를 하나에서

### 어떤 상황에서 어떤 것을 선택할까?

| 상황 | 추천 |
|------|------|
| 비개발자(비즈니스 애널리스트)가 많고 SQL 분석이 주 목적 | Snowflake |
| Python 데이터 엔지니어/과학자 중심 팀 | Databricks |
| 대규모 ML 모델 훈련 + 서빙이 필요 | Databricks |
| 데이터를 내부 스토리지에 두고 싶지 않음 (컴플라이언스) | Databricks |
| 빠른 도입과 낮은 운영 복잡도 우선 | Snowflake |
| 스트리밍 + 배치 통합 파이프라인 | Databricks |
| 사용량이 예측 가능하고 비용 통제가 중요 | Snowflake (Credit 모델 예측 쉬움) |

> **현실적 조언:** 두 플랫폼을 동시에 도입하는 기업도 있습니다. 예: Snowflake는 비즈니스 분석용, Databricks는 ML/데이터 엔지니어링용으로 분리 운영. 단, 데이터 이동 비용과 거버넌스 복잡도가 증가합니다.

---

## 5. Databricks vs AWS 네이티브 (Glue + Redshift + SageMaker)

AWS 환경에 이미 깊이 투자한 조직에서 "굳이 Databricks를 써야 하나?"라는 질문이 자주 나옵니다.

### AWS 네이티브 스택의 장점

- **AWS 서비스 간 네이티브 통합**: S3, IAM, VPC, CloudWatch 등과 자연스러운 연동
- **세밀한 제어**: 인스턴스 유형, 네트워크 구성, 스팟 인스턴스 활용 등 유연성
- **비용 효율**: 적절히 최적화하면 관리형 플랫폼 대비 저렴할 수 있음
- **SageMaker의 ML 깊이**: 엔드포인트 배포, A/B 테스팅, 파이프라인 등 MLOps 기능 성숙

### AWS 네이티브 스택의 한계

- **조합 복잡도**: Glue(ETL) + Redshift(DW) + SageMaker(ML) + Lake Formation(거버넌스) + Kinesis(스트리밍)을 직접 연결하고 운영해야 함
- **거버넌스 단절**: 각 서비스 간 데이터 계보(Lineage) 추적이 불완전
- **개발 경험 비일관성**: 서비스마다 개발 방식이 다름 (Glue Studio vs SageMaker Studio vs Redshift Query Editor)
- **Spark 버전 지연**: EMR의 Spark 버전 업데이트가 Databricks Runtime보다 느림

### 비교 요약

| 관점 | Databricks on AWS | AWS 네이티브 |
|------|------------------|-------------|
| 통합 경험 | 단일 플랫폼, 일관된 UI/API | 서비스별 분리, 다양한 콘솔 |
| 운영 부담 | 낮음 (완전 관리형) | 높음 (서비스 간 연동 직접 구성) |
| 비용 | DBU + 클라우드 인프라 | 서비스별 독립 과금 (최적화 가능) |
| 성능 (Spark) | Photon + 최신 DBR | 표준 Spark (최적화 어려움) |
| ML/AI 깊이 | MLflow 중심, 좋음 | SageMaker 중심, 업계 최고 수준 |
| AWS 서비스 연동 | 양호 (API 통합) | 최상 (네이티브 통합) |

---

## 6. Databricks vs Microsoft Fabric

Microsoft Fabric은 2023년 GA 된 Microsoft의 통합 데이터 분석 플랫폼으로, OneLake + Lakehouse + Data Factory + Power BI + Synapse를 하나로 묶었습니다.

### Fabric의 강점

- **Microsoft 생태계 통합**: Azure Active Directory/Entra ID, Teams, Power BI, Office 365와 네이티브 연동
- **OneLake**: 테넌트당 하나의 논리적 데이터 레이크. Delta/Parquet 기반으로 오픈 포맷 지향
- **Power BI 직접 통합**: Direct Lake 모드로 Parquet 파일에서 직접 Power BI 쿼리 (Import 불필요)
- **가격 모델**: F-SKU(용량 기반) 구매로 예측 가능한 비용
- **Copilot 통합**: Microsoft 365 Copilot과 연계한 자연어 데이터 쿼리

### Fabric의 한계 (2025년 기준)

- **Azure 종속**: 멀티클라우드 전략을 쓰는 조직에는 제약
- **Spark 성능**: Databricks Runtime + Photon 대비 성능 최적화 수준이 낮음
- **ML/AI 성숙도**: Azure ML과의 연동은 있으나, MLflow 수준의 통합 경험은 부족
- **스트리밍**: Eventhouse(KQL 기반)와 Lakehouse 스트리밍 간 통합이 아직 성숙 중
- **엔터프라이즈 도입 사례**: 상대적으로 최신 플랫폼으로 레퍼런스가 적음

### 비교 요약

| 관점 | Databricks | Microsoft Fabric |
|------|-----------|-----------------|
| 클라우드 지원 | AWS, Azure, GCP 동일 경험 | Azure 중심 |
| 데이터 포맷 | Delta Lake (성숙) | OneLake Delta (성장 중) |
| BI 통합 | Databricks AI/BI (2024 출시) | Power BI (업계 최고) |
| ML/AI | MLflow + Model Serving (성숙) | Azure ML 연동 (성장 중) |
| Microsoft 생태계 | 양호 | 최상 |
| 가격 예측성 | DBU 기반 (변동성 있음) | F-SKU 용량 기반 (예측 쉬움) |

> **주목할 경쟁 상황:**Microsoft는 Fabric에 OpenAI(ChatGPT) 투자 역량을 연계하고 있어, Databricks의 AI/ML 포지셔닝에 직접 도전장을 던지고 있습니다. 반면 Databricks는 모델 독립성과 오픈소스 기반의 유연성으로 차별화를 유지하고 있습니다.

---

## 7. "우리 회사에 Databricks가 필요한가?" 판단 체크리스트

모든 회사에 Databricks가 최선은 아닙니다. 아래 체크리스트로 판단해 보시기 바랍니다.

### Databricks가 적합한 경우

- [ ] 데이터 파이프라인(ETL)과 SQL 분석을 **하나의 플랫폼** 에서 하고 싶다
- [ ] **머신러닝/AI** 워크로드가 있거나 향후 계획이 있다
- [ ] 현재 **여러 도구를 조합** 하여 사용하고 있으며, 통합하고 싶다
- [ ] 데이터 **거버넌스(누가, 어떤 데이터를, 왜 접근했는지)** 가 중요하다
- [ ] **스트리밍(실시간)** 데이터 처리가 필요하다
- [ ] **벤더 락인** 을 최소화하고 싶다 (오픈소스 기반)
- [ ] 데이터 팀이 **5명 이상** 이며, 다양한 역할(엔지니어, 분석가, 과학자)이 협업한다
- [ ] **멀티클라우드** 전략을 쓰거나 향후 클라우드 이동 가능성이 있다
- [ ] LLM/생성형 AI 를 데이터 파이프라인에 통합하려 한다

### Databricks보다 다른 선택이 나을 수 있는 경우

- [ ] SQL 분석만 필요하고, ML/AI 계획이 없다 → **Snowflake** 또는 **BigQuery** 검토
- [ ] 예산이 매우 제한적이고, 자체 운영 역량이 있다 → 오픈소스(Spark + Iceberg + Trino) 검토
- [ ] AWS 서비스에 이미 깊이 통합되어 있다 → **AWS Glue + Redshift + SageMaker** 검토
- [ ] Microsoft 생태계(Azure, Power BI, Teams) 중심이다 → **Microsoft Fabric** 검토
- [ ] 데이터 팀이 1~2명이고, 데이터 규모가 작다 → PostgreSQL + dbt로 충분할 수 있습니다
- [ ] 비개발자 중심 조직으로 SQL 외 코딩을 지양한다 → **Snowflake** 또는 **BigQuery** 검토

---

## 8. 마이그레이션 고려사항

다른 플랫폼에서 Databricks로 전환 시 반드시 검토해야 할 사항들입니다.

### Snowflake → Databricks

| 항목 | 고려사항 |
|------|---------|
| 데이터 이동 | COPY INTO로 S3에 내보낸 뒤 Delta 변환. `CONVERT TO DELTA` 명령어 활용 |
| SQL 호환성 | Snowflake 방언(dialect)과 Spark SQL의 차이 존재 (예: `QUALIFY`, 일부 날짜 함수) |
| Snowpipe → DLT | 실시간 수집 파이프라인을 DLT(Delta Live Tables)로 재구현 |
| 저장 프로시저 | Snowflake의 JavaScript 저장 프로시저를 Python/SQL로 전환 필요 |
| 예상 기간 | 데이터 규모와 복잡도에 따라 3개월~1년 이상 |

### AWS EMR → Databricks

| 항목 | 고려사항 |
|------|---------|
| 코드 호환성 | PySpark 코드는 대부분 호환. 단, EMR 전용 설정(예: EMRFS S3 설정)은 제거 필요 |
| 데이터 포맷 | S3의 Parquet/ORC 파일은 그대로 사용 가능. Delta 변환은 단계적으로 |
| 클러스터 관리 | 수동 클러스터 구성에서 Databricks 관리형 클러스터로 전환 (운영 부담 감소) |
| IAM | 기존 EMR IAM Role을 Databricks Instance Profile로 연결 가능 |
| 예상 기간 | 상대적으로 짧음 (1~3개월), Spark 코드 재사용 가능 |

### Microsoft Fabric / Synapse → Databricks

| 항목 | 고려사항 |
|------|---------|
| 데이터 포맷 | OneLake/ADLS의 Delta 파일은 Databricks에서 직접 읽기 가능 |
| Azure 연동 | Databricks on Azure는 동일한 Azure 자원(ADLS Gen2, Key Vault, AAD) 사용 가능 |
| Power BI 연결 | Databricks Partner Connect로 Power BI에서 Databricks SQL Warehouse 연결 |
| Synapse Spark → DBR | 코드 호환성 높음. DBR 전용 기능(Photon, DLT)으로 단계적 전환 |

### 공통 마이그레이션 권장 사항

1. **단계적 접근**: 모든 것을 한 번에 이전하지 말고, 워크로드별로 단계 분리
2. **Unity Catalog 설계 우선**: 마이그레이션 전 데이터 거버넌스 구조(카탈로그/스키마/테이블 계층) 설계
3. **병행 운영 기간 확보**: 신/구 시스템을 동시 운영하며 결과 비교 검증
4. **비용 모델 사전 시뮬레이션**: DBU 소비량을 사전에 추정해 예산 계획 수립
5. **Databricks Professional Services 또는 파트너 활용**: 복잡한 마이그레이션은 전문가 지원 권장

---

## 참고 링크

**Databricks 공식 문서**
- [Databricks: What is a Lakehouse?](https://docs.databricks.com/en/lakehouse/index.html)
- [Databricks: Delta Lake Guide](https://docs.databricks.com/en/delta/index.html)
- [Databricks: Unity Catalog Overview](https://docs.databricks.com/en/data-governance/unity-catalog/index.html)
- [Databricks: Photon Runtime](https://docs.databricks.com/en/runtime/photon.html)
- [Databricks: Delta Live Tables](https://docs.databricks.com/en/delta-live-tables/index.html)
- [Databricks: UniForm (Universal Format)](https://docs.databricks.com/en/delta/uniform.html)

**기술 비교 자료**
- [Databricks Blog: Lakehouse vs Data Warehouse](https://www.databricks.com/blog/2021/08/30/frequently-asked-questions-about-the-data-lakehouse.html)
- [Delta Lake vs Apache Iceberg](https://delta.io/blog/delta-lake-vs-apache-iceberg/)
- [Snowflake Documentation: Iceberg Tables](https://docs.snowflake.com/en/user-guide/tables-iceberg)

**마이그레이션 가이드**
- [Databricks Migration Guide](https://docs.databricks.com/en/migration/index.html)
- [Databricks Partner: 마이그레이션 가속기 프로그램](https://www.databricks.com/partners)
