# 용어 사전 (Glossary)

> 이 문서는 교육 자료 전체에서 사용되는 주요 용어를 한국어로 정리한 것입니다. 각 용어는 카테고리별로 분류되어 있으며, 산업 표준 용어와 Databricks 고유 용어를 구분하여 설명합니다.

---

## 데이터 기초

| 용어 | 영문 | 설명 |
|------|------|------|
| 데이터 엔지니어링 | Data Engineering | 원시 데이터를 수집, 변환, 적재하여 분석 가능한 형태로 만드는 과정입니다 |
| 데이터 웨어하우스 | Data Warehouse (DW) | 분석에 최적화된 구조화된 데이터 저장소입니다. 주로 OLAP 워크로드를 처리합니다 |
| 데이터 레이크 | Data Lake | 모든 유형의 데이터를 원본 그대로 저장하는 대용량 저장소입니다. Schema-on-Read 방식을 사용합니다 |
| 레이크하우스 | Lakehouse | 데이터 레이크 + 웨어하우스의 장점을 결합한 아키텍처입니다. ACID 트랜잭션과 스키마 관리를 지원합니다 |
| 데이터 메시 | Data Mesh | 도메인별로 데이터를 소유하고 관리하는 분산형 데이터 아키텍처 패러다임입니다 |
| 데이터 패브릭 | Data Fabric | 분산된 데이터 환경을 통합적으로 관리하는 아키텍처입니다 |
| ETL | Extract-Transform-Load | 추출 → 변환 → 적재 순서의 전통적 데이터 처리 패턴입니다 |
| ELT | Extract-Load-Transform | 추출 → 적재 → 변환 순서의 현대적 데이터 처리 패턴입니다. 클라우드 환경에서 주로 사용합니다 |
| CDC | Change Data Capture | 데이터베이스 변경 사항(Insert, Update, Delete)을 실시간으로 감지하여 전달하는 기술입니다 |
| SCD | Slowly Changing Dimension | 시간에 따라 변하는 디멘전 데이터를 관리하는 전략입니다. Type 1(덮어쓰기), Type 2(이력 유지) 등이 있습니다 |
| 배치 처리 | Batch Processing | 데이터를 모아서 한꺼번에 처리하는 방식입니다. 주기적으로 실행됩니다 |
| 스트리밍 처리 | Stream Processing | 데이터가 도착하는 즉시 연속적으로 처리하는 방식입니다 |
| 정형 데이터 | Structured Data | 행과 열로 구성된 테이블 형태의 데이터입니다 (예: CSV, 관계형 DB) |
| 반정형 데이터 | Semi-Structured Data | JSON, XML, Avro 등 유연한 구조의 데이터입니다 |
| 비정형 데이터 | Unstructured Data | 이미지, 동영상, 자연어 텍스트 등 고정된 구조가 없는 데이터입니다 |
| Schema-on-Read | 읽기 시 스키마 적용 | 데이터를 읽을 때 스키마를 적용하는 방식입니다. 데이터 레이크에서 주로 사용합니다 |
| Schema-on-Write | 쓰기 시 스키마 적용 | 데이터를 저장할 때 스키마를 적용하는 방식입니다. 웨어하우스에서 주로 사용합니다 |
| 데이터 파이프라인 | Data Pipeline | 데이터를 소스에서 목적지까지 자동으로 이동, 변환하는 일련의 처리 과정입니다 |
| 데이터 거버넌스 | Data Governance | 데이터의 품질, 보안, 접근성, 규정 준수를 관리하는 체계입니다 |

---

## 데이터베이스

| 용어 | 영문 | 설명 |
|------|------|------|
| RDBMS | Relational Database Management System | 관계형 데이터베이스 관리 시스템입니다. Oracle, MySQL, PostgreSQL 등이 있습니다 |
| ACID | Atomicity, Consistency, Isolation, Durability | 트랜잭션의 안전성을 보장하는 네 가지 속성입니다. 원자성, 일관성, 격리성, 지속성을 의미합니다 |
| 정규화 | Normalization | 중복을 제거하여 데이터 일관성을 보장하는 설계 기법입니다. 주로 OLTP에서 사용합니다 |
| 비정규화 | Denormalization | 분석 성능을 위해 의도적으로 중복을 허용하는 설계입니다. OLAP에서 자주 사용합니다 |
| Star 스키마 | Star Schema | 중앙의 팩트 테이블 + 주변 디멘전 테이블로 구성된 별 모양 설계입니다 |
| Snowflake 스키마 | Snowflake Schema | 디멘전을 추가 정규화한 눈꽃 모양 설계입니다 |
| OLTP | Online Transaction Processing | 트랜잭션(삽입, 수정, 삭제) 처리에 최적화된 시스템입니다. 은행, 쇼핑몰 등에서 사용합니다 |
| OLAP | Online Analytical Processing | 대량 데이터 분석(집계, 조인)에 최적화된 시스템입니다 |
| MPP | Massively Parallel Processing | 여러 노드가 병렬로 쿼리를 처리하는 아키텍처입니다. Snowflake, Redshift 등이 대표적입니다 |
| 컬럼형 저장 | Columnar Storage | 데이터를 열 단위로 저장하여 분석 쿼리 성능을 높이는 방식입니다. Parquet이 대표 포맷입니다 |
| 인덱스 | Index | 데이터 검색 속도를 높이는 색인 구조입니다 |
| 파티셔닝 | Partitioning | 테이블을 물리적으로 분할하여 쿼리 성능을 향상시키는 기법입니다 |

---

## Databricks / Apache Spark

| 용어 | 영문 | 설명 |
|------|------|------|
| 워크스페이스 | Workspace | Databricks에서 작업하는 독립적인 환경입니다. 노트북, 클러스터, 작업 등을 포함합니다 |
| 클러스터 | Cluster | Spark 작업을 실행하는 컴퓨팅 리소스의 집합입니다. Driver + Worker 노드로 구성됩니다 |
| 드라이버 | Driver | Spark 작업을 계획하고 지휘하는 메인 프로세스입니다 |
| 실행자 | Executor | 실제 데이터를 병렬로 처리하는 작업자 프로세스입니다 |
| 파티션 | Partition | DataFrame을 물리적으로 나눈 단위입니다. 병렬 처리의 기본 단위가 됩니다 |
| 셔플 | Shuffle | 조인이나 집계 시 Executor 간 데이터가 재분배되는 과정입니다. 비용이 큰 연산입니다 |
| 지연 평가 | Lazy Evaluation | Spark가 변환(Transformation)을 즉시 실행하지 않고, 액션(Action) 호출 시 한꺼번에 실행하는 방식입니다 |
| 카탈리스트 | Catalyst | Spark SQL의 쿼리 최적화 엔진입니다 |
| DBU | Databricks Unit | Databricks의 컴퓨팅 과금 단위입니다. 워크로드 유형에 따라 DBU/시간 단가가 다릅니다 |
| 포톤 | Photon | Databricks의 C++ 기반 고성능 쿼리 엔진입니다. 기존 Spark 대비 2~8배 빠릅니다 |
| 서버리스 | Serverless | 인프라 관리 없이 자동으로 리소스가 할당되는 방식입니다. SQL Warehouse, Jobs, Notebook에서 지원합니다 |
| SQL Warehouse | SQL 웨어하우스 | SQL 쿼리 실행을 위한 전용 컴퓨팅 리소스입니다 |
| 노트북 | Notebook | Python, SQL, Scala, R 코드를 셀 단위로 실행하는 대화형 개발 환경입니다 |

---

## Delta Lake

| 용어 | 영문 | 설명 |
|------|------|------|
| Delta Lake | 델타 레이크 | 클라우드 스토리지 위에 ACID 트랜잭션을 제공하는 오픈소스 스토리지 레이어입니다 |
| Delta 로그 | Delta Log (_delta_log) | Delta Lake의 트랜잭션 로그입니다. 모든 변경 사항이 JSON 파일로 기록됩니다 |
| 타임 트래블 | Time Travel | 과거 시점의 데이터를 버전이나 타임스탬프로 조회, 복원하는 기능입니다 |
| 스키마 진화 | Schema Evolution | 기존 데이터를 유지하면서 테이블 구조(컬럼 추가, 타입 변경 등)를 안전하게 변경하는 기능입니다 |
| 스키마 강제 | Schema Enforcement | 데이터 쓰기 시 테이블 스키마와 일치하지 않는 데이터를 거부하는 기능입니다 |
| 리퀴드 클러스터링 | Liquid Clustering | 데이터 레이아웃을 자동으로 최적화하는 차세대 클러스터링 기술입니다. Z-Ordering을 대체합니다 |
| OPTIMIZE | 최적화 | 작은 파일을 합쳐 쿼리 성능을 향상시키는 명령입니다 |
| VACUUM | 정리 | 더 이상 참조되지 않는 오래된 파일을 삭제하여 스토리지를 절약하는 명령입니다 |
| Z-Ordering | Z 오더링 | 자주 필터링하는 컬럼 기준으로 데이터를 물리적으로 정렬하는 기법입니다 |
| CDF | Change Data Feed | Delta 테이블의 행 수준 변경 이력(Insert, Update, Delete)을 추적하는 기능입니다 |
| UniForm | 유니폼 | Delta 테이블을 Apache Iceberg, Apache Hudi 포맷으로도 읽을 수 있게 하는 호환성 기능입니다 |
| 메달리온 | Medallion Architecture | Bronze(원시) → Silver(정제) → Gold(집계) 3계층으로 데이터를 단계적으로 정제하는 설계 패턴입니다 |
| Parquet | 파케이 | Apache 컬럼형 파일 포맷입니다. Delta Lake의 기본 데이터 파일 형식입니다 |

---

## 데이터 엔지니어링

| 용어 | 영문 | 설명 |
|------|------|------|
| Auto Loader | 오토 로더 | 클라우드 스토리지의 새 파일을 자동 감지, 수집하는 증분 수집 기능입니다. `cloudFiles` 포맷을 사용합니다 |
| SDP | Spark Declarative Pipelines | 선언적(Declarative) 데이터 변환 파이프라인 프레임워크입니다. 기존 DLT(Delta Live Tables)의 새 이름입니다 |
| 스트리밍 테이블 | Streaming Table | 새로 도착한 데이터만 증분 처리하는 테이블입니다. Append-only 워크로드에 최적화되어 있습니다 |
| 구체화된 뷰 | Materialized View (MV) | 쿼리 결과를 물리적으로 저장하고, 소스 변경 시 자동으로 갱신하는 뷰입니다 |
| 기대값 | Expectations | SDP에서 데이터 품질 규칙을 선언적으로 정의하는 기능입니다. 위반 시 경고, 삭제, 실패 중 선택할 수 있습니다 |
| Lakeflow Connect | 레이크플로우 커넥트 | SaaS, 데이터베이스 등 외부 소스에서 데이터를 자동 수집하는 관리형 커넥터입니다 |
| Lakeflow Jobs | 레이크플로우 잡스 | 작업(Task) 스케줄링, 의존성 관리, 오케스트레이션을 제공하는 워크플로우 서비스입니다 |
| 체크포인트 | Checkpoint | 스트리밍 작업의 진행 상태를 저장하여 재시작 시 이어서 처리할 수 있게 하는 기능입니다 |
| 워터마크 | Watermark | 스트리밍에서 늦게 도착하는 데이터(Late Data)를 처리하기 위한 시간 기준입니다 |

---

## 데이터 웨어하우징 / SQL

| 용어 | 영문 | 설명 |
|------|------|------|
| Databricks SQL (DBSQL) | 데이터브릭스 SQL | SQL 기반 분석, 대시보드, 알림을 제공하는 Databricks의 웨어하우징 인터페이스입니다 |
| SQL Warehouse | SQL 웨어하우스 | SQL 쿼리를 실행하는 전용 서버리스 컴퓨팅입니다. Classic과 Pro 두 가지 타입이 있습니다 |
| 쿼리 프로파일 | Query Profile | SQL 쿼리의 실행 계획과 성능 병목을 시각화하는 도구입니다 |
| AI 함수 | AI Functions | SQL 내에서 LLM을 호출하여 텍스트 분류, 요약, 감정 분석 등을 수행하는 내장 함수입니다 |
| CTE | Common Table Expression | WITH절을 사용한 임시 결과 집합입니다. 쿼리 가독성을 높입니다 |
| 윈도우 함수 | Window Function | 행 그룹(윈도우) 단위로 집계, 순위를 계산하는 SQL 함수입니다 (ROW_NUMBER, LAG 등) |

---

## Unity Catalog / 거버넌스

| 용어 | 영문 | 설명 |
|------|------|------|
| Unity Catalog (UC) | 유니티 카탈로그 | Databricks의 통합 데이터 거버넌스 솔루션입니다. 메타스토어 → 카탈로그 → 스키마 → 테이블 4계층 네임스페이스를 제공합니다 |
| 메타스토어 | Metastore | Unity Catalog의 최상위 메타데이터 컨테이너입니다. 리전별로 하나 존재합니다 |
| 카탈로그 | Catalog | 스키마(데이터베이스)를 그룹화하는 최상위 네임스페이스입니다 |
| 스키마 | Schema (Database) | 테이블, 뷰, 함수 등을 그룹화하는 네임스페이스입니다 |
| 리니지 | Lineage | 데이터의 출처, 변환 경로, 의존 관계를 추적하는 기능입니다 |
| Delta Sharing | 델타 셰어링 | 클라우드와 플랫폼에 관계없이 조직 간 안전하게 데이터를 공유하는 오픈 프로토콜입니다 |
| 볼륨 | Volume | Unity Catalog에서 비테이블 형태의 파일(CSV, 이미지, 모델 등)을 관리하는 저장 공간입니다 |
| RBAC | Role-Based Access Control | 역할 기반 접근 제어입니다. 사용자 그룹에 권한을 부여합니다 |
| ABAC | Attribute-Based Access Control | 속성 기반 접근 제어입니다. 행/열 수준 필터링에 활용합니다 (Row Filter, Column Mask) |
| 행 필터 | Row Filter | 사용자 속성에 따라 테이블의 특정 행만 보이게 하는 보안 기능입니다 |
| 열 마스킹 | Column Mask | 사용자 속성에 따라 컬럼 값을 마스킹(가림 처리)하는 보안 기능입니다 |
| 태그 | Tag | 데이터 자산에 메타데이터 레이블을 부착하여 분류, 검색, 거버넌스에 활용하는 기능입니다 |

---

## AI/BI

| 용어 | 영문 | 설명 |
|------|------|------|
| Lakeview | 레이크뷰 | Databricks의 차세대 대시보드 도구입니다. 드래그 앤 드롭으로 시각화를 구성합니다 |
| Genie | 지니 | 자연어로 데이터를 질의하고 시각화할 수 있는 AI 기반 분석 인터페이스입니다 |
| AI/BI Alert | 알림 | SQL 쿼리 결과에 기반하여 조건 충족 시 알림을 전송하는 기능입니다 |

---

## 머신러닝 (ML)

| 용어 | 영문 | 설명 |
|------|------|------|
| MLflow | 엠엘플로우 | ML 실험 추적, 모델 패키징, 배포, 레지스트리를 제공하는 오픈소스 플랫폼입니다 |
| 실험 | Experiment | MLflow에서 모델 학습 시도를 관리하는 단위입니다 |
| 런 | Run | 하나의 모델 학습 실행을 의미합니다. 파라미터, 메트릭, 아티팩트를 기록합니다 |
| 모델 레지스트리 | Model Registry (UC) | Unity Catalog에서 모델의 버전, 스테이지, 메타데이터를 관리하는 저장소입니다 |
| 모델 서빙 | Model Serving | 학습된 모델을 REST API 엔드포인트로 배포하여 실시간 추론을 제공하는 서비스입니다 |
| 피처 | Feature | ML 모델의 입력으로 사용되는 개별 데이터 속성(변수)입니다 |
| 피처 테이블 | Feature Table | 피처를 체계적으로 저장하고 재사용할 수 있도록 관리하는 Delta 테이블입니다 |
| Online Table | 온라인 테이블 | 실시간 피처 조회를 위해 Feature Table을 밀리초 수준 조회 가능한 형태로 동기화한 테이블입니다 |
| 피처 함수 | Feature Function | Python 함수로 정의된 온디맨드(On-Demand) 피처입니다. 서빙 시점에 실시간 계산됩니다 |
| 추론 테이블 | Inference Table | 서빙 엔드포인트의 요청/응답을 자동으로 Delta 테이블에 기록하는 기능입니다 |
| Lakehouse Monitoring | 레이크하우스 모니터링 | 테이블의 데이터 품질, 드리프트, 모델 성능을 자동 모니터링하는 기능입니다 |
| Foundation Model API | 파운데이션 모델 API | 대형 언어 모델(LLM)을 API로 제공하는 Databricks 서비스입니다 |
| ML Runtime | ML 런타임 | TensorFlow, PyTorch, Scikit-learn 등 ML 라이브러리가 사전 설치된 클러스터 런타임입니다 |
| AutoML | 자동 머신러닝 | 데이터셋을 입력하면 자동으로 최적의 모델을 탐색하는 기능입니다 |
| 하이퍼파라미터 | Hyperparameter | 모델 학습 과정을 제어하는 설정값입니다 (예: 학습률, 트리 개수) |
| 데이터 드리프트 | Data Drift | 프로덕션 데이터의 분포가 학습 데이터와 달라지는 현상입니다 |

---

## 에이전트 개발 (Agent)

| 용어 | 영문 | 설명 |
|------|------|------|
| AI 에이전트 | AI Agent | LLM 기반으로 자율적으로 도구(Tool)를 선택, 실행하여 복잡한 작업을 수행하는 AI 시스템입니다 |
| RAG | Retrieval-Augmented Generation | 외부 문서를 검색하여 LLM의 답변을 보강하는 패턴입니다. 환각(Hallucination) 방지에 효과적입니다 |
| 벡터 검색 | Vector Search | 텍스트를 벡터(숫자 배열)로 변환하여 의미적 유사도 기반으로 검색하는 기능입니다 |
| 임베딩 | Embedding | 텍스트, 이미지 등을 고차원 숫자 벡터로 변환하는 것입니다 |
| 벡터 인덱스 | Vector Index | 벡터 검색을 빠르게 수행하기 위한 인덱스 구조입니다. Delta Sync Index와 Direct Access Index가 있습니다 |
| 도구 호출 | Tool Calling (Function Calling) | LLM이 외부 함수(API, SQL, 검색 등)를 호출하여 정보를 얻거나 작업을 수행하는 기능입니다 |
| Agent Evaluation | 에이전트 평가 | RAG/에이전트의 응답 품질(정확성, 관련성, 안전성)을 자동으로 평가하는 프레임워크입니다 |
| MLflow Tracing | 트레이싱 | 에이전트의 실행 경로(검색 → LLM 호출 → 도구 실행)를 단계별로 추적, 기록하는 기능입니다 |
| 프롬프트 엔지니어링 | Prompt Engineering | LLM에 전달하는 지시문(프롬프트)을 최적화하여 원하는 결과를 얻는 기법입니다 |
| 환각 | Hallucination | LLM이 사실이 아닌 내용을 그럴듯하게 생성하는 현상입니다 |
| 가드레일 | Guardrails | LLM의 출력을 제한하여 안전하고 적절한 응답만 생성하도록 하는 안전 장치입니다 |

---

## Lakebase

| 용어 | 영문 | 설명 |
|------|------|------|
| Lakebase | 레이크베이스 | Databricks의 관리형 PostgreSQL 호환 OLTP 데이터베이스입니다 |
| 브랜치 | Branch (Lakebase) | Lakebase 데이터베이스의 독립적인 복사본입니다. Git 브랜치처럼 격리된 환경을 제공합니다 |
| Delta 동기화 | Delta Sync (Lakebase) | Lakebase의 OLTP 데이터를 Delta Lake로 자동 동기화하여 분석에 활용하는 기능입니다 |

---

## 보안 / 네트워크

| 용어 | 영문 | 설명 |
|------|------|------|
| PAT | Personal Access Token | 사용자 인증을 위한 개인 액세스 토큰입니다. API 호출 시 사용합니다 |
| OAuth | Open Authorization | 토큰 기반 인증/인가 표준입니다. M2M(Machine-to-Machine) 인증에도 활용됩니다 |
| 서비스 프린시펄 | Service Principal | 자동화 작업(CI/CD, 스케줄 Job 등)을 위한 비인간 ID입니다 |
| CMK | Customer Managed Key | 고객이 직접 관리하는 암호화 키입니다. 저장 데이터(Data at Rest) 암호화에 사용합니다 |
| IP 접근 제어 | IP Access List | 허용된 IP 주소 범위에서만 워크스페이스에 접속하도록 제한하는 기능입니다 |
| 프라이빗 링크 | Private Link | 공용 인터넷을 거치지 않고 클라우드 내부 네트워크로 통신하는 기능입니다 |
| 시스템 테이블 | System Tables | Databricks 워크스페이스의 운영 데이터(감사 로그, 비용, 리니지 등)를 저장하는 내장 테이블입니다 |
| 감사 로그 | Audit Log | 워크스페이스 내 모든 사용자 활동(로그인, 쿼리, 데이터 접근 등)을 기록하는 로그입니다 |
| SCIM | System for Cross-domain Identity Management | 사용자 및 그룹을 ID 공급자(Okta, Azure AD 등)와 자동 동기화하는 표준입니다 |
| 조건부 접근 | Conditional Access | 장치 상태, 위치, 위험 수준 등 조건에 따라 접근을 동적으로 제어하는 정책입니다 |

---

## 기타 / 도구

| 용어 | 영문 | 설명 |
|------|------|------|
| Databricks CLI | 데이터브릭스 CLI | 명령줄에서 Databricks 리소스를 관리하는 도구입니다 |
| Databricks Asset Bundles (DAB) | 애셋 번들 | 워크스페이스 리소스(Job, Pipeline, 노트북 등)를 코드로 정의하고 CI/CD로 배포하는 IaC 도구입니다 |
| Databricks Apps | 데이터브릭스 앱 | Streamlit, Gradio 등으로 만든 웹 앱을 Databricks 내에서 호스팅하는 서비스입니다 |
| Terraform Provider | 테라폼 프로바이더 | Terraform으로 Databricks 인프라를 코드로 관리하는 도구입니다 |
| Git 연동 | Git Integration (Repos) | GitHub, GitLab 등의 Git 저장소를 Databricks 워크스페이스에 연동하는 기능입니다 |
| REST API | REST API | HTTP 기반으로 Databricks 리소스를 프로그래밍 방식으로 제어하는 인터페이스입니다 |
| SDK | Software Development Kit | Python, Java 등에서 Databricks API를 쉽게 호출하기 위한 라이브러리입니다 |
| 마켓플레이스 | Databricks Marketplace | 데이터셋, 모델, 노트북 등을 공유하고 검색하는 데이터 마켓플레이스입니다 |

---

## 참고 링크

- [Databricks 공식 문서 (AWS)](https://docs.databricks.com/aws/en/)
- [Databricks 공식 문서 (Azure)](https://learn.microsoft.com/en-us/azure/databricks/)
- [Delta Lake 공식 문서](https://docs.delta.io/)
- [MLflow 공식 문서](https://mlflow.org/docs/latest/index.html)
- [Apache Spark 공식 문서](https://spark.apache.org/docs/latest/)
