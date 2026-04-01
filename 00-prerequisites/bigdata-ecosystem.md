# 빅데이터 생태계 — 주요 솔루션과 빅 플레이어

## 현재 빅데이터 생태계의 전체 그림

빅데이터 생태계는 매우 넓고 다양한 도구들이 존재합니다. 이 문서에서는 각 영역별로 어떤 솔루션이 있고, 주요 벤더들이 어떤 포지션을 차지하고 있는지 정리해 드리겠습니다.

> 💡 **왜 생태계를 알아야 하나요?**Databricks를 사용하더라도 주변 도구들과 연동하는 경우가 많습니다. "우리 팀은 이미 Kafka를 쓰고 있는데, Databricks와 어떻게 연결하나요?" "기존 Tableau 대시보드를 Databricks SQL과 연결할 수 있나요?" 같은 질문에 답하려면, 전체 생태계를 알아야 합니다. 또한 기술 선택 회의에서 "왜 Databricks인가?"를 설명하려면, 경쟁 도구들의 강점과 약점을 이해하고 있어야 합니다.

---

## 영역별 기술 지도

| 영역 | 기술 | 설명 |
|------|------|------|
| ** 데이터 수집**| Apache Kafka | 실시간 스트리밍 수집 |
|  | Apache Flink | 스트림 처리 |
|  | AWS Kinesis | AWS 스트리밍 서비스 |
|  | Fivetran / Airbyte | ELT 수집 도구 |
|  | Lakeflow Connect | Databricks 수집 서비스 |
| ** 데이터 저장**| Delta Lake | 레이크하우스 테이블 포맷 |
|  | Apache Iceberg | 오픈 테이블 포맷 |
|  | Apache Hudi | 오픈 테이블 포맷 |
|  | AWS S3 / ADLS / GCS | 클라우드 오브젝트 스토리지 |
| ** 데이터 처리**| Apache Spark | 대규모 분산 처리 |
|  | SQL (Databricks SQL 등) | SQL 기반 분석 |
| ** 데이터 분석/시각화**| Databricks AI/BI | 대시보드 + Genie |
|  | Tableau / Power BI / Looker | 외부 BI 도구 |
| **ML/AI**| MLflow | ML 실험 관리 |
|  | PyTorch / TensorFlow | 딥러닝 프레임워크 |
|  | Agent Framework | AI 에이전트 개발 |

---

## 데이터 수집 (Ingestion) 솔루션

### 스트리밍 수집

| 솔루션 | 개발사 | 설명 |
|--------|--------|------|
| **Apache Kafka**| LinkedIn → Apache | 가장 널리 사용되는 분산 메시지 스트리밍 플랫폼입니다. 초당 수백만 건의 이벤트를 처리할 수 있습니다 |
| **Confluent**| Confluent | Kafka의 상용 관리형 서비스입니다. Kafka 창시자가 설립했습니다 |
| **Amazon Kinesis**| AWS | AWS의 실시간 스트리밍 서비스입니다 |
| **Azure Event Hubs**| Microsoft | Azure의 이벤트 스트리밍 서비스입니다. Kafka API와 호환됩니다 |
| **Google Pub/Sub**| Google | Google Cloud의 메시지 서비스입니다 |

### 배치/CDC 수집

| 솔루션 | 유형 | 설명 |
|--------|------|------|
| **Fivetran**| SaaS | 400+ 소스에서 자동으로 데이터를 수집하는 관리형 ELT 서비스입니다 |
| **Airbyte**| 오픈소스 | Fivetran의 오픈소스 대안입니다. 300+ 커넥터를 제공합니다 |
| **Lakeflow Connect**| Databricks | Databricks의 관리형 수집 서비스입니다. DB, SaaS에서 CDC 기반 수집을 지원합니다 |
| **AWS DMS**| AWS | 데이터베이스 마이그레이션 및 CDC 복제 서비스입니다 |
| **Debezium**| 오픈소스 | CDC(Change Data Capture)에 특화된 오픈소스 커넥터입니다 |
| **Apache NiFi**| 오픈소스 | 시각적 데이터 흐름 관리. 드래그&드롭으로 파이프라인을 구성합니다 |

> 💡 ** 데이터 수집 도구 선택의 현실**: 현업에서 가장 많이 쓰이는 조합은 **(1) Kafka(실시간 이벤트)**+ **(2) Fivetran 또는 Lakeflow Connect(SaaS/DB 배치 수집)**+ **(3) Auto Loader(파일 수집)** 입니다. 세 가지를 용도에 맞게 조합하면 대부분의 수집 요구사항을 커버할 수 있습니다.

> 💡 **CDC(Change Data Capture)란?** 데이터베이스에서 발생하는 변경 사항(INSERT, UPDATE, DELETE)을 실시간으로 감지하여 다른 시스템에 전달하는 기술입니다. 전체 데이터를 매번 복사하는 대신, 변경된 부분만 전달하므로 효율적입니다.

---

## 데이터 저장 (Storage) — 테이블 포맷

현대 데이터 레이크에서는 ** 오픈 테이블 포맷** 이 핵심 기술입니다.

| 포맷 | 개발사 | 특징 |
|------|--------|------|
| **Delta Lake**| Databricks | Spark 생태계와 깊이 통합. 가장 성숙한 포맷입니다 |
| **Apache Iceberg**| Netflix → Apache | 엔진 독립적. Snowflake, Trino 등 다양한 엔진에서 지원합니다 |
| **Apache Hudi**| Uber → Apache | 증분 처리(Incremental Processing)에 강점이 있습니다 |

> 💡 세 포맷 모두 **ACID 트랜잭션**, ** 스키마 진화**, ** 타임 트래블** 을 지원하며, 실제 데이터는 **Parquet** 파일로 저장합니다. 업계는 점차 **Delta Lake + Iceberg** 조합(UniForm)으로 수렴하는 추세입니다.

---

## 데이터 처리 (Processing) 엔진

| 엔진 | 유형 | 설명 |
|------|------|------|
| **Apache Spark**| 배치 + 스트리밍 | 가장 널리 사용되는 분산 처리 엔진입니다. Databricks의 핵심입니다 |
| **Apache Flink**| 스트리밍 우선 | 진정한 이벤트 단위 스트리밍 처리에 강점이 있습니다 |
| **Trino (구 PrestoSQL)**| 대화형 쿼리 | 빠른 SQL 쿼리 엔진입니다. 여러 데이터 소스를 연합 쿼리할 수 있습니다 |
| **dbt**| SQL 변환 | SQL 기반 데이터 변환 도구입니다. ELT의 T(Transform)에 특화되어 있습니다 |
| **Apache Beam**| 통합 모델 | Google이 개발. 배치/스트리밍 통합 프로그래밍 모델을 제공합니다 |
| **Polars**| 로컬 분석 | Rust 기반의 고성능 DataFrame 라이브러리. 단일 머신에서 Pandas보다 10~100배 빠릅니다 |

> 💡 **Spark vs Polars**: "우리 데이터가 수 GB인데도 Spark를 써야 하나요?"라는 질문을 자주 받습니다. 데이터가 수십 GB 이하이고 단일 머신에서 처리 가능하다면, Polars나 DuckDB가 더 가볍고 빠를 수 있습니다. Spark는 수 TB~PB 규모의 데이터, 또는 여러 사용자가 동시에 접근하는 환경에서 진가를 발휘합니다. Databricks의 Serverless 노트북에서는 소규모 데이터도 빠르게 처리할 수 있으므로, 실무에서는 "일관된 환경"의 이점을 위해 Databricks를 사용하는 경우가 많습니다.

---

## 분석 플랫폼 & 데이터 웨어하우스

### 클라우드 데이터 플랫폼 비교

| 플랫폼 | 제공사 | 핵심 강점 | 테이블 포맷 |
|--------|--------|-----------|------------|
| **Databricks**| Databricks | 통합 레이크하우스, Spark/ML/AI | Delta Lake |
| **Snowflake**| Snowflake | 사용 편의성, SQL 성능, 멀티 클라우드 | 독자 + Iceberg |
| **BigQuery**| Google | 서버리스, 자동 확장, ML 통합 | 독자 + BigLake |
| **Redshift**| AWS | AWS 생태계 통합, 비용 효율 | 독자 |
| **Synapse**| Microsoft | Azure 통합, Spark + SQL 결합 | 독자 |
| **Microsoft Fabric**| Microsoft | OneLake 기반 통합 플랫폼 | Delta Lake |

> 💡 ** 클라우드 데이터 플랫폼 선택의 현실**: 실무에서 플랫폼을 선택할 때, 순수 기술력보다 "** 우리 팀이 가장 잘 쓸 수 있는 도구**"가 더 중요합니다. SQL 분석가가 대다수인 팀에서는 Snowflake가, ML/AI 엔지니어가 많은 팀에서는 Databricks가, Microsoft 생태계에 깊이 투자한 조직에서는 Fabric이 자연스러운 선택입니다. "어떤 플랫폼이 객관적으로 가장 좋은가?"보다 "우리 팀의 기술 스택과 역량에 가장 맞는 플랫폼은 무엇인가?"를 먼저 물어보세요.

### BI 도구

| 도구 | 특징 |
|------|------|
| **Tableau**| 데이터 시각화의 선도 기업. 직관적인 드래그&드롭 인터페이스입니다 |
| **Power BI**| Microsoft 생태계와 긴밀하게 통합됩니다. 가격이 경쟁력 있습니다 |
| **Looker**| Google Cloud 소속. LookML 기반의 시맨틱 레이어를 제공합니다 |
| **Databricks AI/BI**| Databricks 내장 대시보드. Genie(자연어 분석)를 지원합니다 |
| **Apache Superset**| 오픈소스 BI 도구. 웹 기반의 대시보드를 제공합니다 |

---

## AI & ML 플랫폼

| 플랫폼 | 제공사 | 설명 |
|--------|--------|------|
| **MLflow**| Databricks (오픈소스) | ML 실험 추적, 모델 관리, GenAI 트레이싱을 지원합니다 |
| **SageMaker**| AWS | AWS의 관리형 ML 플랫폼입니다 |
| **Vertex AI**| Google | Google Cloud의 ML 플랫폼입니다 |
| **Azure ML**| Microsoft | Azure의 ML 플랫폼입니다 |
| **Hugging Face**| Hugging Face | 오픈소스 ML 모델 허브. LLM 생태계의 중심입니다 |
| **Weights & Biases**| W&B | 실험 추적에 특화된 ML 도구입니다 |

> 💡 **MLflow vs 다른 ML 플랫폼**: MLflow는 Databricks가 만든 오픈소스이지만, Databricks 없이도 사용할 수 있습니다. SageMaker나 Vertex AI는 각 클라우드에 종속되지만, MLflow는 어디서든 실행 가능합니다. 이러한 오픈소스 전략은 Databricks의 핵심 철학 중 하나입니다. 현업에서는 MLflow를 "실험 추적의 표준"으로 사용하는 팀이 매우 많으며, Databricks를 쓰지 않는 회사에서도 MLflow만 독립적으로 사용하는 경우가 있습니다.

---

## 데이터 거버넌스

| 솔루션 | 유형 | 설명 |
|--------|------|------|
| **Unity Catalog**| Databricks (오픈소스) | Databricks의 통합 거버넌스 솔루션입니다 |
| **Collibra**| 상용 | 엔터프라이즈 데이터 거버넌스 플랫폼입니다 |
| **Alation**| 상용 | 데이터 카탈로그 + 거버넌스 플랫폼입니다 |
| **Apache Atlas**| 오픈소스 | Hadoop 생태계의 메타데이터 관리 도구입니다 |
| **Purview**| Microsoft | Azure의 데이터 거버넌스 및 카탈로그 서비스입니다 |
| **AWS Glue Data Catalog**| AWS | AWS의 관리형 메타데이터 카탈로그입니다 |

> 💡 ** 데이터 거버넌스가 왜 중요한가요?**GDPR(유럽 개인정보보호법), 개인정보보호법 등 규제가 강화되면서, "누가, 어떤 데이터를, 언제, 왜 접근했는지"를 추적할 수 있어야 합니다. 거버넌스 도구 없이 운영하면 감사(Audit) 시 대응이 불가능합니다. Unity Catalog는 이를 Databricks 플랫폼 안에서 자연스럽게 해결합니다.

---

## 빅 플레이어 포지션 맵

** 데이터 플랫폼 포지션 비교**

| 플랫폼 | SQL/분석 ↔ 엔지니어링/ML | 온프레미스 ↔ 클라우드 | 특징 |
|--------|----------------------|---------------------|------|
| **Databricks**| 엔지니어링/ML 중심 | 클라우드 네이티브 | Spark 기반 통합 플랫폼 |
| **Snowflake**| SQL/분석 중심 | 클라우드 네이티브 | 클라우드 데이터 웨어하우스 |
| **BigQuery**| SQL/분석 중심 | 클라우드 네이티브 | Google 서버리스 분석 |
| **Redshift**| SQL/분석 중심 | 클라우드 | AWS 데이터 웨어하우스 |
| **Microsoft Fabric**| 중간 | 클라우드 네이티브 | Microsoft 통합 분석 |
| **Hadoop**| 엔지니어링 중심 | 온프레미스 | 레거시 분산 처리 |
| **Teradata**| SQL/분석 중심 | 온프레미스 | 레거시 데이터 웨어하우스 |

---

## 실무 경험으로 본 각 도구 — 써본 사람만 아는 이야기

### Hadoop: 설치에 3일, 운영에 평생

2010년대 초반, Hadoop은 빅데이터의 대명사였습니다. 하지만 실제로 운영해본 사람이라면 이런 기억이 있을 것입니다.

| 항목 | 현실 |
|------|------|
| ** 설치**| NameNode, DataNode, ResourceManager, NodeManager, ZooKeeper, Hive Metastore... 각각 설정 파일이 수백 줄입니다. 클러스터 한 대를 띄우는 데 3~5일이 걸렸습니다 |
| ** 운영**| HDFS의 DataNode가 하나 죽으면 리밸런싱이 시작되는데, 대용량 데이터를 복제하느라 네트워크 대역폭을 잡아먹어 다른 작업이 느려집니다 |
| **YARN 튜닝**| 메모리 설정을 잘못하면 "Container killed by YARN" 에러가 끊이질 않습니다. `yarn.nodemanager.resource.memory-mb`, `mapreduce.map.memory.mb` 등 수십 개의 파라미터를 조정해야 합니다 |
| **Hive**| SQL처럼 보이지만, 간단한 SELECT도 MapReduce 잡으로 변환되어 30초~수 분이 걸렸습니다 |

### Spark: 30분이면 시작, 하지만 튜닝은 별도

Spark는 Hadoop의 고통을 크게 줄여주었습니다.

| 항목 | Hadoop 대비 개선 |
|------|-----------------|
| ** 속도**| 인메모리 처리로 Hive 대비 10~100배 빠릅니다 |
| **API**| DataFrame API가 직관적입니다. SQL도 바로 사용 가능합니다 |
| ** 설치**| `pip install pyspark` 한 줄로 로컬에서 바로 시작할 수 있습니다 |

하지만 직접 운영하면 다른 문제가 생깁니다.

| 문제 | 설명 |
|------|------|
| **OOM (Out of Memory)**| Shuffle 과정에서 메모리 부족이 가장 흔한 에러입니다. `spark.sql.shuffle.partitions`, `spark.executor.memory` 튜닝이 필수입니다 |
| ** 데이터 스큐(Skew)**| 특정 키에 데이터가 몰리면 한 executor만 일하고 나머지는 놀게 됩니다. 현업에서 가장 골치아픈 성능 이슈입니다 |
| ** 클러스터 관리**| EMR이나 직접 구축한 Spark 클러스터는 버전 업그레이드, 라이브러리 관리, 노드 추가/제거를 직접 해야 합니다 |

> 💡 **Databricks와의 차이**: Databricks는 이런 Spark 운영의 고통을 해결하기 위해 만들어졌습니다. 클러스터 자동 확장, Photon 엔진(C++ 기반 고성능 실행 엔진), Adaptive Query Execution(자동 튜닝), 데이터 스큐 자동 감지 등을 플랫폼 수준에서 제공합니다.

### 왜 어떤 도구는 사라지고, 어떤 도구는 살아남았는가?

| 도구 | 상태 | 이유 |
|------|------|------|
| **MapReduce**| 사실상 사망 | Spark가 더 빠르고, 더 쉬운 API를 제공했기 때문입니다 |
| **Apache Pig**| 사실상 사망 | Spark DataFrame/SQL이 Pig Latin보다 더 직관적이었습니다 |
| **Apache Oozie**| 레거시 | Airflow, Databricks Jobs 등 더 나은 워크플로우 도구가 등장했습니다 |
| **Apache Hive**| 축소 | Hive Metastore는 여전히 사용되지만, 쿼리 엔진은 Spark SQL, Trino 등으로 대체되었습니다 |
| **Apache Kafka**| 건재 | 실시간 이벤트 스트리밍이라는 고유한 영역을 확보하고 있습니다 |
| **Apache Spark**| 건재 | 배치 + 스트리밍 + ML을 하나의 엔진으로 처리하는 생태계의 힘입니다 |
| **Apache Flink**| 성장 | True Streaming 분야에서 Spark와 차별화됩니다 |
| **dbt**| 급성장 | SQL 기반 변환에 집중하여 데이터 분석가들에게 큰 호응을 얻었습니다 |

> 💡 ** 핵심 교훈**: 살아남은 도구들의 공통점은 **(1) 명확한 차별점이 있고, (2) 활발한 커뮤니티가 있으며, (3) 클라우드 환경에 적응했다** 는 것입니다. 반면 사라진 도구들은 "더 나은 대안이 등장했는데, 전환 비용이 높지 않았던" 도구들입니다.

---

## 현재 생태계에서 Databricks의 위치

Databricks는 "데이터 플랫폼의 스위스 아미 나이프"라고 할 수 있습니다. 아래와 같이 다른 도구들을 하나의 플랫폼으로 대체하거나 통합합니다.

| 기존 도구 조합 | Databricks에서는 |
|---------------|-----------------|
| EMR + Spark (직접 관리) | Databricks Runtime (관리형 Spark) |
| Airflow + Cron | Lakeflow Jobs (워크플로우 오케스트레이션) |
| Fivetran + Airbyte | Lakeflow Connect (관리형 커넥터) |
| Redshift / Snowflake (SQL 분석) | Databricks SQL + Photon |
| SageMaker / Vertex AI (ML) | MLflow + Model Serving |
| Apache Atlas + Ranger (거버넌스) | Unity Catalog |
| Tableau / Looker (BI) | AI/BI Dashboard + Genie |

물론 Databricks가 모든 도구를 100% 대체하는 것은 아닙니다. 예를 들어, 복잡한 시각화가 필요하면 Tableau가 여전히 강력하고, 초저지연 스트리밍이 필요하면 Flink가 적합합니다. 하지만 "80%의 워크로드를 하나의 플랫폼에서 처리할 수 있다"는 것은 운영 복잡성을 크게 줄여줍니다.

> 💡 ** 현업에서는 이렇게 합니다**: 대부분의 기업에서 Databricks를 도입할 때 "모든 기존 도구를 Databricks로 교체"하지는 않습니다. 가장 흔한 패턴은 **(1) 기존 Kafka는 그대로 유지**, **(2) ETL/ML은 Databricks로 통합**, **(3) BI는 기존 Tableau를 Databricks SQL에 연결** 하는 방식입니다. 점진적 마이그레이션이 현실적이며, 한 번에 모든 것을 바꾸려 하면 리스크가 커집니다.

---

## 기술 선택 가이드

| 요구 사항 | 권장 솔루션 |
|-----------|-----------|
| 통합 데이터 플랫폼 (ETL + SQL + ML + AI) | **Databricks**|
| SQL 분석 중심, 사용 편의성 우선 | **Snowflake**|
| AWS 생태계에 올인 | **Redshift + Glue + SageMaker**|
| Google Cloud 생태계 | **BigQuery + Vertex AI**|
| Microsoft 생태계 | **Microsoft Fabric** 또는 **Azure Databricks**|
| 오픈소스 중심, 벤더 독립 | **Spark + Iceberg + Trino + MLflow**|
| 소규모 데이터, 빠른 시작 | **PostgreSQL + dbt + Metabase**|

> 💡 ** 현업에서는 이렇게 합니다**: 기술 선택은 "현재 팀의 역량"과 "데이터 규모"를 기준으로 해야 합니다. 데이터가 수 GB이고 팀이 2~3명이면 PostgreSQL + dbt로 충분합니다. 데이터가 TB~PB 규모이고 팀이 10명 이상이면 Databricks나 Snowflake 같은 클라우드 데이터 플랫폼이 필수적입니다. "나중에 커질 것을 대비해서 처음부터 큰 도구를 도입하자"는 접근은 대부분 과도한 복잡성과 비용만 초래합니다.

---

## 도구 선택 시 피해야 할 함정

현업에서 기술 선택 시 흔히 빠지는 함정들입니다.

### 함정 1: "유행하니까" 도입

| 잘못된 판단 | 현실 |
|------------|------|
| "Netflix가 Iceberg를 쓰니까 우리도" | Netflix는 수만 개의 테이블과 PB급 데이터를 관리합니다. 우리 회사의 데이터 규모와 요구사항이 같은지 먼저 확인해야 합니다 |
| "모두 Kafka를 쓰니까 우리도 도입" | 일일 이벤트가 수만 건이면 Kafka 대신 SQS/Cloud Functions로 충분합니다 |
| "dbt가 대세니까 도입" | 팀에 SQL을 잘 하는 분석가가 있어야 효과가 있습니다 |

### 함정 2: "오픈소스가 무료"라는 착각

| 항목 | 오픈소스 직접 운영 | 관리형 서비스 (Databricks 등) |
|------|------------------|---------------------------|
| 라이선스 비용 | 무료 | 유료 |
| 인프라 비용 | 직접 부담 | 포함 또는 별도 |
| 운영 인력 | 전담 2~3명 필요 | 최소화 |
| 장애 대응 | 직접 해결 | 벤더 지원 |
| 총 소유 비용 (TCO) | 대부분 ** 더 비쌈** | 예측 가능 |

### 함정 3: "한 도구로 모든 것을"

만능 도구는 없습니다. Databricks가 아무리 통합 플랫폼이라 해도, 초저지연 OLTP(온라인 트랜잭션)나 복잡한 시각화에는 전문 도구가 더 적합합니다. 각 도구의 강점과 약점을 이해하고, 적재적소에 배치하는 것이 핵심입니다.

---

## 정리

| 영역 | 주요 솔루션 | Databricks의 대응 |
|------|-----------|------------------|
| 데이터 수집 | Kafka, Fivetran, Airbyte | Lakeflow Connect, Auto Loader |
| 데이터 저장 | Delta Lake, Iceberg, Hudi | Delta Lake + UniForm |
| 데이터 처리 | Spark, Flink, dbt | Apache Spark (내장) |
| SQL 분석 | Snowflake, BigQuery, Redshift | Databricks SQL |
| BI | Tableau, Power BI, Looker | AI/BI Dashboard + Genie |
| ML/AI | SageMaker, Vertex AI, MLflow | MLflow + Model Serving + Agent Framework |
| 거버넌스 | Collibra, Alation, Glue Catalog | Unity Catalog |

---

## 데이터 오케스트레이션 도구

데이터 파이프라인을 스케줄링하고 관리하는 도구도 생태계의 중요한 부분입니다.

| 도구 | 유형 | 설명 |
|------|------|------|
| **Apache Airflow**| 오픈소스 | 가장 널리 사용되는 워크플로우 오케스트레이션 도구입니다. Python으로 DAG(Directed Acyclic Graph)를 정의합니다 |
| **Lakeflow Jobs**| Databricks | Databricks 내장 스케줄러입니다. 노트북, SQL, Python, JAR 등을 태스크로 조합할 수 있습니다 |
| **Prefect**| 오픈소스/상용 | Airflow의 현대적 대안입니다. Python 네이티브 경험을 제공합니다 |
| **Dagster**| 오픈소스/상용 | 데이터 자산 중심의 오케스트레이션 도구입니다 |
| **AWS Step Functions**| AWS | AWS 서버리스 워크플로우 서비스입니다 |

> 💡 ** 현업에서의 선택**: Databricks를 주력으로 사용한다면 **Lakeflow Jobs** 가 가장 자연스럽습니다. 클러스터 관리, 재시도, 알림이 통합되어 있습니다. 만약 Databricks 외의 다른 시스템(AWS Lambda, API 호출 등)도 오케스트레이션해야 한다면 **Airflow** 가 적합합니다. 둘을 함께 사용하는 경우(Airflow가 전체 워크플로우를 관리하고, 개별 Databricks 작업은 Lakeflow Jobs로 실행)도 많습니다.

---

## 데이터 품질 도구

데이터의 정확성과 신뢰성을 보장하기 위한 도구도 점점 중요해지고 있습니다.

| 도구 | 유형 | 설명 |
|------|------|------|
| **Great Expectations**| 오픈소스 | Python 기반 데이터 품질 검증 프레임워크입니다 |
| **Soda**| 오픈소스/상용 | SQL 기반 데이터 품질 체크 도구입니다 |
| **SDP Expectations**| Databricks | SDP 파이프라인에서 선언적으로 데이터 품질 규칙을 정의합니다 |
| **Monte Carlo** | 상용 | 데이터 관측성(Data Observability) 플랫폼입니다 |

---

## 참고 링크

- [Databricks Blog: Data + AI Platform](https://www.databricks.com/blog)
- [Gartner: Cloud Database Management Systems](https://www.gartner.com/reviews/market/cloud-database-management-systems)
- [DB-Engines Ranking](https://db-engines.com/en/ranking)
