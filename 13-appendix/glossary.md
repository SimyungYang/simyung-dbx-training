# 용어 사전 (Glossary)

> 이 문서는 교육 자료 전체에서 사용되는 주요 용어를 한국어로 정리한 것입니다.
> 영문 알파벳 순으로 정렬되어 있으며, Databricks 고유 용어와 산업 표준 용어를 함께 다룹니다.
> 각 용어는 **한국어 명칭** + 영문 원어 + 한 줄 정의 구조로 작성되었고, 중요 용어에는 추가 설명이 포함되어 있습니다.

---

## A

### ABAC (Attribute-Based Access Control)
**속성 기반 접근 제어** — 사용자 역할뿐 아니라 속성(부서, 직책, 리전 등)에 따라 데이터 접근을 동적으로 제어하는 방식입니다.
Row Filter, Column Mask와 함께 사용하여 동일 테이블에서 사용자별로 다른 데이터를 보이게 할 수 있습니다. Unity Catalog에서 Python UDF 기반으로 구현합니다.

### ACID
**원자성·일관성·격리성·지속성** — 트랜잭션의 안전성을 보장하는 네 가지 속성(Atomicity, Consistency, Isolation, Durability)입니다.
Delta Lake는 클라우드 오브젝트 스토리지 위에서 ACID 트랜잭션을 구현하여, 기존 데이터 레이크의 불안정한 읽기/쓰기 문제를 해결합니다.

### AI Agent (AI 에이전트)
**AI 에이전트** — LLM이 자율적으로 도구(Tool)를 선택하고 실행하여 복잡한 멀티스텝 작업을 완수하는 AI 시스템입니다.
단순 챗봇과 달리, 에이전트는 SQL 실행·파일 읽기·외부 API 호출 등을 스스로 결정하며 반복 루프를 돌 수 있습니다. Databricks에서는 MLflow로 에이전트를 추적·평가하고 Model Serving으로 배포합니다.

### AI Functions (AI 함수)
**AI 함수** — SQL 쿼리 내에서 LLM을 직접 호출하여 텍스트 요약, 분류, 감정 분석, 번역 등을 수행하는 내장 함수 집합입니다.
`ai_summarize()`, `ai_classify()`, `ai_extract()` 등이 있으며 SQL Warehouse에서 실행 가능합니다.

### AI/BI
**AI 기반 비즈니스 인텔리전스** — Lakeview 대시보드와 Genie를 포함하는 Databricks의 차세대 BI 플랫폼입니다.
시각화 도구(Lakeview)와 자연어 질의 인터페이스(Genie)를 결합하여 데이터 분석가뿐 아니라 비기술 사용자도 데이터를 직접 탐색할 수 있도록 설계되었습니다.

### Asset Bundles (DAB)
**애셋 번들** — Databricks Asset Bundles의 약칭. Job, Pipeline, 노트북, 대시보드 등 워크스페이스 리소스를 YAML + 코드로 정의하고 CI/CD 파이프라인으로 배포하는 IaC(Infrastructure as Code) 도구입니다.
개발·스테이징·프로덕션 환경별 설정을 분리하여 안전하게 프로모션할 수 있습니다. `databricks bundle deploy` 명령 하나로 전체 리소스가 배포됩니다.

### Auto Loader (오토 로더)
**자동 로더** — 클라우드 스토리지(S3, ADLS, GCS)에 새로 도착하는 파일을 자동 감지하고 증분 수집하는 Databricks 기능입니다.
`cloudFiles` 소스 포맷을 사용하며, 스키마 추론·진화를 자동으로 처리합니다. 수백만 개 파일이 있는 디렉토리에서도 전체 목록 조회 없이 효율적으로 신규 파일만 처리합니다.

### AutoML (자동 머신러닝)
**자동 머신러닝** — 데이터셋을 입력하면 전처리, 피처 엔지니어링, 모델 선택, 하이퍼파라미터 튜닝을 자동으로 수행하는 기능입니다.
Databricks AutoML은 결과로 Python 노트북 코드를 생성하므로, 자동화 결과를 그대로 재사용하거나 커스터마이즈할 수 있습니다.

---

## B

### Batch Processing (배치 처리)
**배치 처리** — 데이터를 일정 주기로 모아서 한꺼번에 처리하는 방식입니다.
실시간성보다 처리량과 비용 효율이 중요한 경우에 적합합니다. Lakeflow Jobs로 스케줄링하여 실행합니다.

### Bronze / Silver / Gold (메달리온 아키텍처)
**메달리온 아키텍처** — 원시 데이터(Bronze) → 정제 데이터(Silver) → 집계·서비스 데이터(Gold)로 데이터를 3계층으로 단계적으로 정제하는 설계 패턴입니다.
각 계층은 Delta Lake 테이블로 관리되며, Unity Catalog 내 별도 스키마 또는 카탈로그로 구분합니다. SDP(Spark Declarative Pipelines)와 함께 사용하면 계층 간 의존성을 선언적으로 관리할 수 있습니다.

---

## C

### Catalog (카탈로그)
**카탈로그** — Unity Catalog 4계층 네임스페이스(Metastore → Catalog → Schema → Table)에서 두 번째 수준의 컨테이너입니다.
여러 스키마(데이터베이스)를 그룹화하며, 환경(dev/prod)이나 도메인(finance/marketing)별로 분리하는 데 사용합니다. `catalog.schema.table` 형식의 3-level 이름 규칙으로 접근합니다.

### Catalyst (카탈리스트)
**카탈리스트** — Apache Spark SQL의 규칙 기반·비용 기반 쿼리 최적화 엔진입니다.
논리 플랜 → 최적화된 논리 플랜 → 물리 플랜 → 코드 생성 단계를 거칩니다. Photon은 이 물리 플랜의 실행 단계를 C++로 교체하여 성능을 극대화합니다.

### CDC (Change Data Capture)
**변경 데이터 캡처** — 데이터베이스의 Insert·Update·Delete 변경 이벤트를 실시간으로 감지하여 하위 시스템으로 전달하는 기술입니다.
Lakeflow Connect 또는 Delta Lake의 CDF(Change Data Feed)와 결합하여 소스 DB 변경 사항을 레이크하우스에 지속적으로 동기화할 수 있습니다.

### CDF (Change Data Feed)
**변경 데이터 피드** — Delta Lake 테이블에서 행 수준 변경 이력(Insert·Update·Delete 및 Before/After 값)을 추적하는 기능입니다.
`SHOW HISTORY`, `table_changes()` 함수로 조회하며, 다운스트림 파이프라인이 전체 재처리 없이 변경분만 소비할 수 있게 합니다.

### Clean Room (클린 룸)
**클린 룸** — 두 조직이 원본 데이터를 서로에게 노출하지 않고 공동으로 분석할 수 있는 격리된 협업 환경입니다.
Databricks Clean Room은 Delta Sharing 프로토콜 위에서 구현되며, 각 참여자는 합의된 SQL 쿼리만 실행할 수 있고 raw 데이터는 조회할 수 없습니다.

### Cluster (클러스터)
**클러스터** — Spark 작업을 실행하는 Driver 노드 + Worker 노드 집합으로 구성된 컴퓨팅 리소스입니다.
All-Purpose 클러스터(대화형 작업)와 Job 클러스터(자동화 작업 전용) 두 가지 유형이 있습니다. 자동 시작·종료, 자동 크기 조정(Auto Scaling)을 지원합니다.

### CMK (Customer Managed Key)
**고객 관리형 키** — 고객이 직접 생성하고 관리하는 암호화 키로, 저장 데이터(Data at Rest)와 관리 플레인 데이터를 암호화하는 데 사용합니다.
AWS KMS, Azure Key Vault, GCP KMS와 연동하며, 키 폐기 시 해당 데이터에 대한 접근이 즉시 차단됩니다. 금융·의료 등 규제 산업에서 필수적으로 요구됩니다.

### Column Mask (열 마스킹)
**열 마스킹** — 사용자의 역할·속성에 따라 특정 컬럼 값을 가림 처리(마스킹)하는 Unity Catalog 보안 기능입니다.
예를 들어 일반 사용자에게는 주민등록번호를 `***-****-****` 형태로 보여주고 관리자에게만 원본 값을 제공합니다. Python UDF로 마스킹 로직을 정의합니다.

### Columnar Storage (컬럼형 저장)
**컬럼형 저장** — 데이터를 행(Row) 단위가 아닌 열(Column) 단위로 저장하는 방식입니다.
특정 컬럼만 읽는 분석 쿼리에서 I/O를 대폭 줄이며, 같은 타입의 값이 연속 저장되어 압축 효율도 높습니다. Parquet이 대표적인 컬럼형 파일 포맷입니다.

### CTE (Common Table Expression)
**공통 테이블 표현식** — SQL의 `WITH` 절로 정의하는 이름 있는 임시 결과 집합입니다.
복잡한 서브쿼리를 분리하여 가독성을 높이고, 동일 결과를 여러 번 참조할 때 유용합니다.

---

## D

### Data Drift (데이터 드리프트)
**데이터 드리프트** — 프로덕션 환경의 입력 데이터 분포가 모델 학습 시 사용한 데이터와 시간이 지남에 따라 달라지는 현상입니다.
모델 성능 저하의 주요 원인이며, Lakehouse Monitoring으로 자동 감지할 수 있습니다.

### Data Governance (데이터 거버넌스)
**데이터 거버넌스** — 데이터의 품질, 보안, 접근성, 규정 준수, 생명주기를 조직 차원에서 관리하는 정책·프로세스·기술의 체계입니다.
Databricks에서는 Unity Catalog가 통합 거버넌스 레이어 역할을 담당합니다.

### Data Lake (데이터 레이크)
**데이터 레이크** — 정형·반정형·비정형 데이터를 원본 형태 그대로 대규모로 저장하는 저장소입니다.
Schema-on-Read 방식을 사용하여 유연성이 높으나, 데이터 품질 관리와 거버넌스가 어렵습니다. 이 한계를 해결한 것이 Lakehouse 패턴입니다.

### Data Lineage (데이터 리니지)
**데이터 리니지** — 데이터가 어느 소스에서 생성되어, 어떤 파이프라인·쿼리를 거쳐, 어떤 테이블·대시보드에 사용되었는지를 추적하는 기능입니다.
Unity Catalog는 테이블·컬럼 수준의 리니지를 자동으로 수집하며, UI에서 시각적 그래프로 확인할 수 있습니다. 규정 준수 감사 및 영향 범위 분석에 활용됩니다.

### Data Mesh (데이터 메시)
**데이터 메시** — 데이터를 중앙 집중식 팀이 아닌, 각 도메인 팀이 데이터를 제품(Product)으로 소유하고 관리하는 분산형 조직·기술 패러다임입니다.
Delta Sharing과 Unity Catalog를 활용하면 도메인 간 안전한 데이터 공유를 구현할 수 있습니다.

### Data Pipeline (데이터 파이프라인)
**데이터 파이프라인** — 데이터를 소스에서 목적지까지 수집·변환·적재하는 일련의 자동화된 처리 과정입니다.

### Data Warehouse (데이터 웨어하우스)
**데이터 웨어하우스** — 분석에 최적화된 구조화된 데이터 저장소입니다. Schema-on-Write 방식을 사용하며 주로 OLAP 워크로드를 처리합니다.

### Databricks Apps (데이터브릭스 앱)
**데이터브릭스 앱** — Streamlit, Gradio, Dash 등 Python 기반 웹 프레임워크로 만든 애플리케이션을 Databricks 플랫폼 내에서 직접 호스팅하는 서비스입니다.
별도 서버 없이 Unity Catalog 데이터와 Model Serving 엔드포인트에 안전하게 접근하는 데이터 앱을 빠르게 배포할 수 있습니다.

### Databricks CLI (데이터브릭스 CLI)
**데이터브릭스 CLI** — 터미널에서 Databricks 리소스(클러스터, Job, 파일, 번들 등)를 관리하는 공식 명령줄 인터페이스입니다.
`databricks bundle`, `databricks jobs run-now` 등 자동화·CI/CD 작업에 활수 불가결한 도구입니다.

### DBU (Databricks Unit)
**데이터브릭스 단위** — Databricks 컴퓨팅 사용량을 측정하는 표준 과금 단위입니다.
워크로드 유형(Jobs Compute, SQL Compute, Serverless, ML 등)과 인스턴스 사양에 따라 시간당 DBU 소비량이 달라집니다. 비용 최적화를 위해 System Tables의 `billing.usage` 테이블로 DBU 소비 패턴을 분석합니다.

### Delta Lake (델타 레이크)
**델타 레이크** — 클라우드 오브젝트 스토리지(S3, ADLS, GCS) 위에 ACID 트랜잭션, 스키마 관리, Time Travel을 제공하는 오픈소스 스토리지 레이어입니다.
데이터는 Parquet 파일로 저장되고, `_delta_log` 디렉토리의 JSON 트랜잭션 로그로 메타데이터를 관리합니다. Delta Lake는 Linux Foundation에 기증된 오픈소스 프로젝트이며, Databricks의 레이크하우스 아키텍처의 핵심입니다.

### Delta Log (델타 로그)
**델타 로그** — `_delta_log` 디렉토리에 저장되는 Delta Lake의 트랜잭션 로그입니다.
모든 쓰기 작업은 원자적으로 JSON 커밋 파일에 기록되며, 10개 커밋마다 Parquet 체크포인트 파일로 압축됩니다. 이 로그가 Time Travel, ACID, CDF 기능의 기반입니다.

### Delta Sharing (델타 셰어링)
**델타 셰어링** — 클라우드·플랫폼에 무관하게 조직 간 데이터를 안전하게 공유하는 오픈 프로토콜입니다.
수신자는 Databricks가 없어도 Python, Spark, pandas, Power BI 등 다양한 클라이언트로 공유 데이터에 접근할 수 있습니다. 데이터 복사 없이 공유하므로 항상 최신 데이터를 제공합니다.

### Deletion Vector (삭제 벡터)
**삭제 벡터** — Delta Lake에서 Delete·Update 작업 시 기존 Parquet 파일을 재작성하지 않고, 삭제·변경된 행의 위치를 비트맵으로 별도 파일에 기록하는 최적화 기법입니다.
소량 변경 시 전체 파일 재작성을 피하여 쓰기 성능을 크게 향상시킵니다. 읽기 시 삭제 벡터를 적용하여 삭제된 행을 필터링합니다.

### Driver (드라이버)
**드라이버** — Spark 애플리케이션의 메인 프로세스입니다. 실행 계획(DAG)을 생성하고 Executor에게 태스크를 분배합니다.
OOM(Out of Memory) 오류가 잦은 경우 드라이버 메모리 증설을 검토해야 합니다.

---

## E

### Embedding (임베딩)
**임베딩** — 텍스트, 이미지, 코드 등 비정형 데이터를 고차원 숫자 벡터로 변환하는 표현 방식입니다.
의미적으로 유사한 데이터는 벡터 공간에서 가깝게 위치합니다. RAG 시스템에서 문서 검색의 핵심 기술이며, Databricks Vector Search에 저장하여 빠른 유사도 검색을 수행합니다.

### ETL / ELT
**추출-변환-적재 / 추출-적재-변환** — ETL은 전통적인 데이터 처리 순서(Extract → Transform → Load)이며, ELT는 클라우드 환경에서 선호하는 순서(Extract → Load → Transform)입니다.
ELT는 먼저 원본 데이터를 레이크에 적재한 뒤 컴퓨팅 파워로 변환하므로 유연성이 높습니다.

### Executor (실행자)
**실행자** — Spark에서 실제 데이터를 병렬로 처리하는 JVM 프로세스입니다. 각 Worker 노드에서 하나 이상 실행됩니다.

### Expectations (기대값)
**기대값** — SDP(Spark Declarative Pipelines)에서 데이터 품질 규칙을 선언적으로 정의하는 기능입니다.
규칙 위반 시 `WARN`(경고만), `DROP`(해당 행 제거), `FAIL`(파이프라인 중단) 세 가지 처리 방식을 지정할 수 있습니다.

---

## F

### Feature Store (피처 스토어)
**피처 스토어** — ML 모델에 사용되는 피처를 중앙에서 정의·저장·재사용·공유하는 관리 시스템입니다.
Databricks Feature Store는 Unity Catalog와 통합되어 피처 테이블을 관리하며, 학습 시와 서빙 시 동일한 피처 변환 로직을 보장하는 일관성(Training-Serving Skew 방지)이 핵심 가치입니다.

### Feature Table (피처 테이블)
**피처 테이블** — ML 피처를 체계적으로 저장하고 재사용 가능하도록 관리하는 Delta Lake 테이블입니다.
기본 키(Primary Key)와 타임스탬프 키를 가지며, Feature Store가 학습·서빙 시 피처를 자동으로 조인합니다.

### Foundation Model API (파운데이션 모델 API)
**파운데이션 모델 API** — Meta Llama, DBRX, Mistral 등 대형 언어 모델(LLM)을 API 엔드포인트로 제공하는 Databricks 관리형 서비스입니다.
Pay-per-token 과금 방식의 Serverless 모드와 전용 컴퓨팅(Provisioned Throughput) 모드를 지원합니다. AI Agent 및 RAG 시스템 구축의 기반이 됩니다.

---

## G

### Genie (지니)
**지니** — 비즈니스 사용자가 자연어로 질문하면 SQL을 자동 생성하여 데이터를 조회·시각화하는 AI 기반 셀프 서비스 분석 인터페이스입니다.
Genie Space에 비즈니스 맥락(지표 정의, 샘플 질문, 신뢰할 SQL 예시)을 등록하면 도메인 특화 응답 품질이 높아집니다. AI/BI 제품군의 핵심 구성 요소입니다.

### Git Integration (Git 연동)
**Git 연동** — GitHub, GitLab, Bitbucket, Azure DevOps 등의 원격 저장소를 Databricks 워크스페이스와 연결하는 기능입니다.
노트북, 파일, 폴더를 Git으로 버전 관리하며, Asset Bundles와 결합하면 완전한 GitOps 기반 배포가 가능합니다.

### Guardrails (가드레일)
**가드레일** — LLM이 생성하는 출력이 안전하고 적절한 범위를 벗어나지 않도록 입력·출력 단계에서 검사·필터링하는 안전 장치입니다.
Databricks AI Security (DAIS) 기능을 통해 Model Serving 엔드포인트에 가드레일을 적용할 수 있습니다.

---

## H

### Hallucination (환각)
**환각** — LLM이 사실에 근거 없는 내용을 그럴듯하게 생성하는 현상입니다.
RAG(Retrieval-Augmented Generation) 패턴으로 신뢰할 수 있는 외부 문서를 컨텍스트로 제공하면 환각을 크게 줄일 수 있습니다.

### Hyperparameter (하이퍼파라미터)
**하이퍼파라미터** — 모델 학습 과정을 제어하는 설정값(학습률, 배치 크기, 트리 깊이 등)입니다.
MLflow를 통해 하이퍼파라미터와 결과 메트릭을 함께 기록하고, Hyperopt·Optuna로 자동 탐색할 수 있습니다.

---

## I

### Iceberg (아이스버그)
**Apache Iceberg** — Netflix가 개발한 오픈 테이블 포맷으로, 대규모 분석 테이블을 위한 ACID 트랜잭션·스키마 진화·Time Travel을 제공합니다.
Delta Lake와 유사한 기능을 제공하며, Delta UniForm을 통해 Delta 테이블을 Iceberg 호환 형태로 읽을 수 있습니다.

### Inference Table (추론 테이블)
**추론 테이블** — Model Serving 엔드포인트의 요청(Input)과 응답(Output)을 자동으로 Delta 테이블에 기록하는 기능입니다.
모델 성능 모니터링, 데이터 드리프트 감지, 파인튜닝용 데이터 수집에 활용합니다.

### Instance Pool (인스턴스 풀)
**인스턴스 풀** — 클러스터 시작 시간을 단축하기 위해 미리 할당되어 대기 중인 VM 인스턴스 집합입니다.
새 클러스터가 풀에서 인스턴스를 빌려 사용하므로, 일반 클러스터 시작(5~10분) 대비 수십 초로 단축됩니다. 비용 절감을 위해 자동 축소(Idle Instance Auto-Termination) 설정이 중요합니다.

### IP Access List (IP 접근 제어 목록)
**IP 접근 제어 목록** — 허용된 IP 주소 또는 CIDR 범위에서만 워크스페이스에 접속하도록 제한하는 보안 기능입니다.

---

## L

### Lakehouse (레이크하우스)
**레이크하우스** — 데이터 레이크의 유연성·경제성과 데이터 웨어하우스의 ACID 트랜잭션·성능·거버넌스를 하나의 플랫폼에서 제공하는 아키텍처입니다.
Databricks가 제안한 개념으로, Delta Lake가 기술적 기반을 이룹니다. BI·ML·스트리밍 워크로드를 동일 데이터 위에서 실행합니다.

### Lakebase (레이크베이스)
**레이크베이스** — Databricks가 제공하는 관리형 PostgreSQL 호환 OLTP 데이터베이스 서비스입니다.
Git 브랜치처럼 데이터베이스를 분기(Branch)하여 개발·테스트 환경을 격리할 수 있고, Delta Sync로 OLTP 데이터를 Delta Lake에 자동 복제하여 분석 워크로드와 연계합니다.

### Lakeflow Connect (레이크플로우 커넥트)
**레이크플로우 커넥트** — Salesforce, ServiceNow, Google Analytics 등 SaaS 애플리케이션과 관계형 데이터베이스에서 데이터를 자동·증분 수집하는 Databricks 관리형 커넥터 서비스입니다.
커넥터 설정만으로 CDC 기반 증분 수집이 구성되며, 수집 결과는 Unity Catalog Delta 테이블로 저장됩니다.

### Lakeflow Jobs (레이크플로우 잡스)
**레이크플로우 잡스** — 작업(Task) 스케줄링, 의존성(DAG) 관리, 재시도·알림·파라미터화를 제공하는 Databricks 워크플로우 오케스트레이션 서비스입니다.
Notebook, Python Script, SDP Pipeline, dbt, Spark JAR, SQL 등 다양한 Task 유형을 지원합니다. Asset Bundles로 코드화하여 CI/CD로 배포합니다.

### Lakeview Dashboard (레이크뷰 대시보드)
**레이크뷰 대시보드** — 드래그 앤 드롭 인터페이스로 차트·표·필터를 구성하는 Databricks의 차세대 BI 대시보드 도구입니다.
기존 Legacy 대시보드를 대체하며, 협업·게시·예약 새로고침·구독 기능을 제공합니다.

### Lazy Evaluation (지연 평가)
**지연 평가** — Spark가 변환(Transformation: `filter`, `select`, `join` 등)을 즉시 실행하지 않고 실행 계획만 구성하다가, 액션(`count`, `collect`, `write` 등) 호출 시 한꺼번에 최적화하여 실행하는 방식입니다.
전체 실행 계획을 보고 불필요한 단계를 제거하거나 순서를 조정하는 최적화가 가능해집니다.

### Liquid Clustering (리퀴드 클러스터링)
**리퀴드 클러스터링** — 지정한 컬럼 기준으로 데이터 파일의 물리적 배치를 자동으로 최적화하는 차세대 데이터 스킵핑 기술입니다.
기존 파티셔닝과 Z-Ordering을 대체합니다. 클러스터링 키는 테이블 생성 이후에도 언제든 변경 가능하며, 증분 방식으로 점진적으로 최적화되어 전체 재작성 없이 적용됩니다.

---

## M

### Medallion Architecture (메달리온 아키텍처)
→ **Bronze / Silver / Gold** 항목 참조.

### Metastore (메타스토어)
**메타스토어** — Unity Catalog의 최상위 메타데이터 컨테이너입니다.
하나의 클라우드 리전에 하나의 메타스토어가 존재하며, 테이블 정의·권한·리니지 정보를 중앙에서 관리합니다. 여러 워크스페이스가 하나의 메타스토어를 공유하여 데이터를 일관되게 접근합니다.

### Metric View (메트릭 뷰)
**메트릭 뷰** — 비즈니스 지표(KPI)를 SQL로 정의하고 재사용 가능한 단위로 게시하는 Unity Catalog 내의 시맨틱 레이어 객체입니다.
Genie와 AI/BI 대시보드에서 메트릭 뷰를 참조하여 조직 전체에서 동일한 지표 정의를 사용할 수 있습니다.

### MLflow (엠엘플로우)
**MLflow** — ML 실험 추적, 모델 패키징(MLmodel), 모델 레지스트리, 배포 자동화를 제공하는 오픈소스 ML 라이프사이클 플랫폼입니다.
Databricks에 완전 통합되어 있으며, Unity Catalog 기반 Model Registry로 모델 버전 관리와 거버넌스를 지원합니다. MLflow Tracing으로 AI 에이전트의 실행 경로까지 추적합니다.

### MLflow Tracing (MLflow 트레이싱)
**MLflow 트레이싱** — AI 에이전트·LLM 체인의 실행 경로(LLM 호출 → Tool 실행 → 검색 → 응답)를 계층적 Span으로 추적·기록하는 기능입니다.
디버깅, 지연 시간 분석, 에이전트 평가에 활용됩니다. OpenTelemetry 표준과 호환됩니다.

### Model Registry (모델 레지스트리)
**모델 레지스트리** — 학습된 ML 모델의 버전, 스테이지(Staging/Production), 태그, 설명을 중앙에서 관리하는 저장소입니다.
Unity Catalog 기반 Model Registry를 사용하면 카탈로그 수준의 접근 제어와 리니지 추적이 가능합니다.

### Model Serving (모델 서빙)
**모델 서빙** — 학습된 ML 모델 또는 LLM을 REST API 엔드포인트로 실시간 배포하는 Databricks 관리형 서비스입니다.
자동 스케일링(0 → N)을 지원하며, A/B 테스트를 위한 트래픽 분할, GPU 지원, Inference Table 자동 로깅을 제공합니다.

### MPP (Massively Parallel Processing)
**대규모 병렬 처리** — 여러 노드가 각자의 메모리와 CPU로 쿼리를 분산 처리하는 아키텍처입니다.
Spark, Redshift, Snowflake 등이 MPP 아키텍처를 기반으로 합니다.

---

## N

### Normalization / Denormalization (정규화 / 비정규화)
**정규화 / 비정규화** — 정규화는 중복을 제거하여 데이터 일관성을 보장하는 설계(OLTP에 적합), 비정규화는 분석 성능을 위해 의도적으로 중복을 허용하는 설계(OLAP에 적합)입니다.

### Notebook (노트북)
**노트북** — Python, SQL, Scala, R 코드를 셀 단위로 작성하고 실행 결과(텍스트, 차트, 테이블)를 인라인으로 확인하는 대화형 개발 환경입니다.
Databricks 노트북은 복수 언어 혼용, 실시간 공동 편집, 버전 관리를 지원합니다.

---

## O

### OAuth (오어스)
**OAuth** — 토큰 기반 인증·인가 표준(OAuth 2.0)입니다.
Databricks는 사용자 인증(U2M)과 서비스 간 인증(M2M) 모두 OAuth를 지원합니다. PAT 대비 만료 시간이 짧아 보안이 강화됩니다.

### OLAP / OLTP
**분석 처리 / 트랜잭션 처리** — OLAP(Online Analytical Processing)은 대량 데이터 집계·조인에 최적화, OLTP(Online Transaction Processing)은 빠른 단건 삽입·수정·삭제에 최적화된 시스템입니다.

### Online Table (온라인 테이블)
**온라인 테이블** — Feature Table의 피처를 밀리초 수준의 저지연으로 조회할 수 있도록 DynamoDB·Cosmos DB 기반 온라인 스토어로 동기화한 테이블입니다.
Model Serving 추론 시 실시간 피처 조회에 사용됩니다.

### OPTIMIZE (최적화 명령)
**OPTIMIZE 명령** — Delta 테이블의 작은 파일을 하나의 큰 파일로 병합하여 읽기 성능을 향상시키는 SQL 명령입니다.
`OPTIMIZE table ZORDER BY (col)` 형태로 Z-Ordering과 함께 실행하거나, Liquid Clustering 테이블에서는 `OPTIMIZE table` 만으로 클러스터링 최적화를 수행합니다.

---

## P

### Parquet (파케이)
**Apache Parquet** — 컬럼형 바이너리 파일 포맷으로, Delta Lake의 기본 데이터 파일 형식입니다.
Snappy, ZSTD 등의 압축과 함께 열 단위 인코딩을 사용하여 높은 압축률과 빠른 분석 성능을 제공합니다.

### Partition (파티션)
**파티션** — DataFrame을 병렬 처리를 위해 물리적으로 나눈 단위입니다(Spark 파티션). 또한 테이블을 특정 컬럼 값으로 디렉토리 구조로 나누는 테이블 파티셔닝도 있습니다.
너무 많은 작은 파티션은 오히려 스케줄링 오버헤드를 증가시키므로, Liquid Clustering이 파티셔닝의 현대적 대안으로 권장됩니다.

### PAT (Personal Access Token)
**개인 액세스 토큰** — 사용자가 Databricks REST API 또는 CLI 인증을 위해 생성하는 토큰입니다.
만료 기간 설정을 강제하고, 가능하면 OAuth M2M 또는 서비스 프린시펄로 대체하는 것이 보안상 권장됩니다.

### Photon (포톤)
**포톤** — Databricks가 자체 개발한 C++ 기반 벡터화 쿼리 실행 엔진입니다.
Spark의 JVM 기반 실행 엔진을 대체하여 SQL·DataFrame 워크로드를 2~8배 빠르게 처리합니다. SQL Warehouse와 Photon 활성화 클러스터에서 자동으로 사용됩니다.

### PrivateLink (프라이빗 링크)
**프라이빗 링크** — 공용 인터넷을 거치지 않고 클라우드 프로바이더의 내부 네트워크(AWS PrivateLink, Azure Private Link)를 통해 Databricks Control Plane·Data Plane 간 통신을 격리하는 기능입니다.
금융·공공 등 보안 규정이 엄격한 환경에서 데이터 유출 위험을 줄이기 위해 필수적으로 구성합니다. Frontend PrivateLink(사용자 트래픽)와 Backend PrivateLink(데이터 플레인 트래픽)로 구분됩니다.

### Predictive I/O (예측 I/O)
**예측 I/O** — Delta Lake의 쿼리 필터를 분석하여 읽어야 할 파일·행 그룹을 사전에 예측하고 불필요한 I/O를 최소화하는 Databricks 최적화 기능입니다.
Delete·Update가 많은 테이블에서 Deletion Vector와 함께 사용하면 성능 향상 효과가 큽니다.

### Prompt Engineering (프롬프트 엔지니어링)
**프롬프트 엔지니어링** — LLM에 전달하는 지시문(시스템 프롬프트, 사용자 메시지, 예시 등)을 최적화하여 원하는 응답을 얻는 기법입니다.
Few-shot, Chain-of-Thought, ReAct 등의 기법이 있으며, AI Agent 설계에서 핵심 스킬입니다.

---

## R

### RAG (Retrieval-Augmented Generation)
**검색 증강 생성** — 사용자 질문에 관련된 문서를 Vector Search로 검색한 뒤, 검색 결과를 컨텍스트로 포함하여 LLM이 답변을 생성하는 AI 패턴입니다.
LLM의 학습 데이터에 없는 최신·사내 정보를 활용할 수 있고, 환각(Hallucination)을 줄이는 데 효과적입니다. Databricks에서는 Vector Search + Foundation Model API + Model Serving 조합으로 구현합니다.

### RBAC (Role-Based Access Control)
**역할 기반 접근 제어** — 사용자 또는 그룹에 역할(Role)을 부여하고, 역할에 따라 리소스 접근 권한을 관리하는 방식입니다.
Unity Catalog의 `GRANT` 문으로 카탈로그·스키마·테이블·볼륨 수준의 권한을 세밀하게 제어합니다.

### Row Filter (행 필터)
**행 필터** — 사용자의 역할·속성에 따라 테이블의 특정 행만 보이도록 동적으로 필터링하는 Unity Catalog 보안 기능입니다.
예를 들어 각 지역 관리자는 자신의 지역 데이터만 조회되도록 설정합니다. Python UDF로 필터 조건을 정의하며 테이블에 연결합니다.

### Run (MLflow 런)
**MLflow 런** — MLflow Experiment 내에서 하나의 모델 학습 또는 평가 실행을 나타내는 단위입니다.
파라미터, 메트릭, 태그, 아티팩트(모델 파일, 그래프 등)를 자동 또는 수동으로 기록합니다.

---

## S

### Schema (스키마)
**스키마** — Unity Catalog 4계층 네임스페이스에서 세 번째 수준의 컨테이너입니다(기존 Hive의 Database에 해당).
테이블, 뷰, 함수, 볼륨 등을 그룹화합니다. `catalog.schema.table` 형식으로 접근합니다.

### Schema Evolution (스키마 진화)
**스키마 진화** — 기존 Delta 테이블에 새 컬럼을 추가하거나 타입을 변경하는 등 스키마를 안전하게 변경하는 기능입니다.
`mergeSchema` 옵션 또는 `ALTER TABLE` 명령으로 활성화합니다. 기존 데이터는 유지되며 새 컬럼에는 `null`이 채워집니다.

### Schema Enforcement (스키마 강제)
**스키마 강제** — Delta Lake에서 테이블 스키마와 맞지 않는 데이터가 쓰여지는 것을 자동으로 거부하는 기능입니다.
데이터 품질의 1차 방어선 역할을 하며, 기본적으로 활성화되어 있습니다.

### SCIM (시킴)
**SCIM** — System for Cross-domain Identity Management. Okta, Azure AD, OneLogin 등 외부 ID 공급자(IdP)와 Databricks 사용자·그룹 정보를 자동으로 동기화하는 표준 프로토콜(RFC 7644)입니다.
신규 직원 추가·퇴사 처리가 IdP에서만 이루어져도 Databricks에 자동 반영됩니다.

### SCD (Slowly Changing Dimension)
**완만 변경 차원** — 시간에 따라 서서히 변하는 차원 데이터를 관리하는 전략입니다.
Type 1(기존 값 덮어쓰기), Type 2(변경 이력 행 추가·유효 기간 관리), Type 3(이전 값 컬럼 추가) 등이 있습니다. Delta Lake의 `MERGE INTO` 구문으로 Type 2 SCD를 구현하는 경우가 많습니다.

### SDP (Spark Declarative Pipelines)
**스파크 선언형 파이프라인** — Python/SQL 코드로 데이터 변환을 선언적으로 정의하면 Databricks가 실행 순서·의존성·재시도·증분 처리를 자동으로 관리하는 파이프라인 프레임워크입니다.
이전 이름은 Delta Live Tables(DLT)이며 2025년에 SDP로 변경되었습니다. Streaming Table, Materialized View, Expectations(데이터 품질)를 지원합니다.

### Serverless (서버리스)
**서버리스** — 사용자가 VM 유형·크기를 선택하거나 클러스터 관리를 하지 않아도 Databricks가 자동으로 인프라를 프로비저닝·스케일링·종료하는 컴퓨팅 방식입니다.
Serverless SQL Warehouse, Serverless Jobs, Serverless Notebooks을 지원합니다. 미사용 시 비용이 발생하지 않으며 콜드 스타트가 빠릅니다.

### Service Principal (서비스 프린시펄)
**서비스 프린시펄** — CI/CD 파이프라인, 자동화 스크립트 등 비인간 주체가 Databricks에 인증하기 위한 가상 ID입니다.
개인 계정 대신 서비스 프린시펄을 사용하면 권한 범위를 최소화하고 개인 퇴사 시 서비스 중단을 방지합니다.

### Shuffle (셔플)
**셔플** — Spark에서 `join`, `groupBy`, `repartition` 등 연산 시 Executor 간에 데이터가 재분배되는 과정입니다.
네트워크 I/O와 디스크 I/O가 대규모로 발생하므로 성능 병목의 주요 원인입니다. 셔플 횟수를 줄이는 것이 Spark 최적화의 핵심입니다.

### Spark (Apache Spark)
**아파치 스파크** — 대규모 분산 데이터 처리를 위한 오픈소스 병렬 컴퓨팅 프레임워크입니다.
배치·스트리밍·SQL·ML·Graph 처리를 통합 엔진으로 지원합니다. Databricks는 Spark의 창시자들이 설립한 회사로, Databricks Runtime에는 최적화된 Spark가 포함됩니다.

### Spot Instance (스팟 인스턴스)
**스팟 인스턴스** — 클라우드 프로바이더(AWS Spot, Azure Spot, GCP Preemptible)가 남는 용량을 저렴한 가격(최대 70~90% 할인)에 제공하는 인스턴스입니다.
가용성이 보장되지 않아 중단될 수 있으므로, 중단 허용 가능한 배치 잡의 Worker 노드에 사용하고 Driver 노드는 On-Demand로 설정하는 것이 일반적입니다.

### SQL Warehouse (SQL 웨어하우스)
**SQL 웨어하우스** — SQL 쿼리 실행에 특화된 Databricks 관리형 컴퓨팅 리소스입니다.
Classic(고객 VPC 내 VM), Pro(고급 기능 + 서버리스 옵션), Serverless(완전 관리형) 세 가지 타입이 있습니다. Photon 엔진을 기본으로 사용하며 쿼리 결과 캐싱, 자동 스케일링을 제공합니다.

### SSO (Single Sign-On)
**단일 로그인** — 하나의 인증으로 여러 서비스에 로그인하는 방식입니다.
Databricks는 SAML 2.0 기반 SSO를 지원하며, Okta·Azure AD·Google Workspace 등과 연동됩니다. SCIM과 함께 구성하면 사용자 프로비저닝까지 자동화됩니다.

### Star Schema (스타 스키마)
**스타 스키마** — 중앙의 팩트 테이블(Fact Table)과 주변의 디멘전 테이블(Dimension Table)로 구성된 별 모양의 데이터 웨어하우스 설계 패턴입니다.
조인 수가 최소화되어 집계 쿼리 성능이 우수합니다.

### Streaming Table (스트리밍 테이블)
**스트리밍 테이블** — SDP에서 새로 도착한 데이터만 증분으로 처리하는 Append-only 테이블입니다.
Auto Loader, Kafka, Kinesis 등 스트리밍 소스에서 지속적으로 데이터를 수집하는 데 최적화되어 있습니다.

### System Tables (시스템 테이블)
**시스템 테이블** — Databricks 플랫폼의 운영 데이터(감사 로그, 비용·DBU 사용량, 리니지, 클러스터 이벤트 등)를 Unity Catalog 내 `system.*` 스키마에 자동으로 저장하는 내장 테이블입니다.
`system.billing.usage`, `system.access.audit`, `system.lineage.table_lineage` 등의 테이블로 비용 최적화·보안 감사·거버넌스 대시보드를 구축합니다.

---

## T

### Tag (태그)
**태그** — Unity Catalog 데이터 자산(테이블, 컬럼, 스키마 등)에 부착하는 키-값 메타데이터 레이블입니다.
PII(개인정보) 컬럼 식별, 비용 센터 분류, 데이터 분류(Confidential, Public 등)에 활용합니다.

### Terraform Provider (테라폼 프로바이더)
**테라폼 프로바이더** — HashiCorp Terraform으로 Databricks 워크스페이스·클러스터·Job·Unity Catalog 등 리소스를 코드(HCL)로 선언하고 관리하는 IaC 도구입니다.

### Time Travel (타임 트래블)
**타임 트래블** — Delta Lake 테이블의 과거 스냅샷을 버전 번호(`VERSION AS OF`) 또는 타임스탬프(`TIMESTAMP AS OF`)로 조회하거나 복원하는 기능입니다.
트랜잭션 로그가 보존되는 기간(기본 7일, VACUUM 실행 전까지) 동안 사용할 수 있습니다.

### Tool Calling (도구 호출)
**도구 호출 (Function Calling)** — LLM이 사전 정의된 외부 함수(SQL 실행, API 호출, 파일 검색 등)를 자율적으로 선택하여 호출하는 기능입니다.
AI Agent의 핵심 메커니즘이며, 도구 스펙은 JSON Schema 형태로 LLM에 전달됩니다.

---

## U

### UniForm (유니폼)
**UniForm** — Delta Lake 테이블을 Apache Iceberg 또는 Apache Hudi 포맷으로도 읽을 수 있게 하는 호환성 레이어입니다.
데이터 복사 없이 Delta 테이블에 Iceberg 메타데이터를 추가로 생성합니다. Athena, Snowflake 등 Iceberg를 지원하는 외부 엔진에서 동일 데이터를 조회할 때 유용합니다.

### Unity Catalog (유니티 카탈로그)
**유니티 카탈로그** — Databricks의 통합 데이터·AI 거버넌스 솔루션입니다.
Metastore → Catalog → Schema → Table/Volume/Model의 4계층 네임스페이스를 제공하며, 중앙화된 접근 제어(RBAC/ABAC), 데이터 리니지, 감사 로그, Delta Sharing을 단일 플랫폼에서 관리합니다. 여러 워크스페이스가 하나의 Unity Catalog를 공유합니다.

---

## V

### VACUUM (정리 명령)
**VACUUM 명령** — Delta 테이블에서 더 이상 참조되지 않는 오래된 Parquet 파일을 삭제하여 스토리지를 절약하는 명령입니다.
기본 보존 기간은 7일이며, 이 기간 이전 파일을 삭제하면 Time Travel이 해당 버전에서 불가능해집니다.

### Vector Search (벡터 검색)
**벡터 검색** — 텍스트·이미지 임베딩 벡터를 저장하고 코사인 유사도·L2 거리 기반으로 의미적으로 유사한 항목을 빠르게 검색하는 Databricks 관리형 벡터 데이터베이스입니다.
Delta Sync Index(Delta 테이블과 자동 동기화)와 Direct Access Index(직접 업서트) 두 가지 모드를 제공합니다. RAG 파이프라인의 검색 단계에 핵심적으로 사용됩니다.

### Volume (볼륨)
**볼륨** — Unity Catalog에서 테이블 형태가 아닌 파일(CSV, 이미지, 모델 가중치, 로그 등)을 관리하는 스토리지 공간입니다.
External Volume(기존 S3/ADLS 경로 참조)과 Managed Volume(Unity Catalog가 완전 관리) 두 가지 유형이 있습니다. `dbfs:/Volumes/catalog/schema/volume/` 경로로 접근합니다.

### VPC Peering (VPC 피어링)
**VPC 피어링** — 두 VPC(Virtual Private Cloud)를 인터넷을 거치지 않고 직접 연결하는 AWS/Azure 네트워크 기능입니다.
PrivateLink에 비해 구성이 단순하지만, 전이적 피어링을 지원하지 않아 다수 VPC 연결 시 복잡도가 높아집니다.

---

## W

### Watermark (워터마크)
**워터마크** — 스트리밍 처리에서 늦게 도착하는 데이터(Late Data)를 허용하는 최대 지연 시간 기준입니다.
워터마크 이후로 늦게 도착한 데이터는 집계에서 제외됩니다. `withWatermark("event_time", "10 minutes")` 형태로 설정합니다.

### Window Function (윈도우 함수)
**윈도우 함수** — 특정 행 집합(윈도우) 단위로 순위, 누적 합계, 이전/다음 행 값 등을 계산하는 SQL/Spark 함수입니다.
`ROW_NUMBER()`, `RANK()`, `LAG()`, `LEAD()`, `SUM() OVER()` 등이 있습니다.

### Workspace (워크스페이스)
**워크스페이스** — Databricks에서 팀이나 프로젝트 단위로 사용하는 독립적인 작업 환경입니다.
노트북, 클러스터, Job, SQL Warehouse, 파이프라인, 대시보드 등 모든 리소스가 워크스페이스 내에 존재합니다. Account Console에서 여러 워크스페이스를 생성하고 Unity Catalog 메타스토어를 연결합니다.

---

## Z

### Z-Ordering (Z 오더링)
**Z 오더링** — 자주 필터링하는 컬럼 기준으로 관련 데이터를 동일 파일에 물리적으로 모아 저장하여 파일 스킵(Data Skipping) 효율을 높이는 기법입니다.
`OPTIMIZE table ZORDER BY (column)` 으로 실행합니다. Liquid Clustering이 동적·증분 방식으로 이를 대체하고 있습니다.

---

## 참고 링크

- [Databricks 공식 문서 (AWS)](https://docs.databricks.com/aws/en/)
- [Databricks 공식 문서 (Azure)](https://learn.microsoft.com/en-us/azure/databricks/)
- [Delta Lake 공식 문서](https://docs.delta.io/)
- [MLflow 공식 문서](https://mlflow.org/docs/latest/index.html)
- [Apache Spark 공식 문서](https://spark.apache.org/docs/latest/)
- [Unity Catalog 오픈소스](https://github.com/unitycatalog/unitycatalog)
- [Databricks Terraform Provider](https://registry.terraform.io/providers/databricks/databricks/latest/docs)
