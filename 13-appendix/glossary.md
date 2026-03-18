# 용어 사전 (Glossary)

> 이 문서는 교육 자료 전체에서 사용되는 주요 용어를 한국어로 정리한 것입니다.

---

## 데이터 기초

| 용어 | 영문 | 설명 |
|------|------|------|
| 데이터 엔지니어링 | Data Engineering | 원시 데이터를 수집·변환·적재하여 분석 가능한 형태로 만드는 과정입니다 |
| 데이터 웨어하우스 | Data Warehouse | 분석에 최적화된 구조화된 데이터 저장소입니다 |
| 데이터 레이크 | Data Lake | 모든 유형의 데이터를 원본 그대로 저장하는 대용량 저장소입니다 |
| 레이크하우스 | Lakehouse | 데이터 레이크 + 웨어하우스의 장점을 결합한 아키텍처입니다 |
| ETL | Extract-Transform-Load | 추출 → 변환 → 적재 순서의 데이터 처리 패턴입니다 |
| ELT | Extract-Load-Transform | 추출 → 적재 → 변환 순서의 현대적 데이터 처리 패턴입니다 |
| CDC | Change Data Capture | 데이터베이스 변경 사항을 실시간으로 감지·전달하는 기술입니다 |
| 배치 처리 | Batch Processing | 데이터를 모아서 한꺼번에 처리하는 방식입니다 |
| 스트리밍 처리 | Stream Processing | 데이터가 도착하는 즉시 연속적으로 처리하는 방식입니다 |
| 정형 데이터 | Structured Data | 행·열 테이블 형태의 데이터입니다 |
| 반정형 데이터 | Semi-Structured Data | JSON, XML 등 유연한 구조의 데이터입니다 |
| 비정형 데이터 | Unstructured Data | 이미지, 동영상, 텍스트 등 구조가 없는 데이터입니다 |

---

## 데이터베이스

| 용어 | 영문 | 설명 |
|------|------|------|
| RDBMS | Relational Database Management System | 관계형 데이터베이스 관리 시스템입니다 |
| ACID | Atomicity, Consistency, Isolation, Durability | 트랜잭션의 안전성을 보장하는 네 가지 속성입니다 |
| 정규화 | Normalization | 중복을 제거하여 데이터 일관성을 보장하는 설계입니다 |
| 비정규화 | Denormalization | 분석 성능을 위해 의도적으로 중복을 허용하는 설계입니다 |
| Star 스키마 | Star Schema | 팩트 테이블 + 디멘전 테이블의 별 모양 설계입니다 |
| Snowflake 스키마 | Snowflake Schema | 디멘전을 추가 정규화한 눈꽃 모양 설계입니다 |
| OLTP | Online Transaction Processing | 트랜잭션 처리에 최적화된 시스템입니다 |
| OLAP | Online Analytical Processing | 분석 처리에 최적화된 시스템입니다 |
| 인덱스 | Index | 데이터 검색 속도를 높이는 색인 구조입니다 |
| SCD | Slowly Changing Dimension | 시간에 따라 변하는 데이터를 관리하는 전략입니다 |

---

## Databricks / Spark

| 용어 | 영문 | 설명 |
|------|------|------|
| Workspace | 워크스페이스 | Databricks에서 작업하는 독립적인 환경입니다 |
| Cluster | 클러스터 | Spark 작업을 실행하는 컴퓨팅 리소스의 집합입니다 |
| Driver | 드라이버 | Spark 작업을 계획하고 지휘하는 프로세스입니다 |
| Executor | 실행자 | 실제 데이터를 처리하는 작업자 프로세스입니다 |
| Partition | 파티션 | DataFrame을 물리적으로 나눈 단위입니다 |
| Shuffle | 셔플 | Executor 간 데이터가 재분배되는 과정입니다 |
| DBU | Databricks Unit | Databricks의 컴퓨팅 과금 단위입니다 |
| Photon | 포톤 | Databricks의 C++ 기반 고성능 쿼리 엔진입니다 |
| Serverless | 서버리스 | 인프라 관리 없이 자동으로 리소스가 할당되는 방식입니다 |

---

## Delta Lake

| 용어 | 영문 | 설명 |
|------|------|------|
| Delta Lake | 델타 레이크 | 클라우드 스토리지 위에 ACID 트랜잭션을 제공하는 오픈소스 스토리지 레이어입니다 |
| Delta Log | 델타 로그 | Delta Lake의 트랜잭션 로그입니다 (모든 변경 사항 기록) |
| Time Travel | 타임 트래블 | 과거 시점의 데이터를 조회·복원하는 기능입니다 |
| Schema Evolution | 스키마 진화 | 테이블 구조를 안전하게 변경하는 기능입니다 |
| Liquid Clustering | 리퀴드 클러스터링 | 차세대 데이터 레이아웃 최적화 기술입니다 |
| OPTIMIZE | 최적화 | 작은 파일을 합쳐 쿼리 성능을 향상시키는 명령입니다 |
| VACUUM | 정리 | 불필요한 오래된 파일을 삭제하는 명령입니다 |
| UniForm | 유니폼 | Delta 테이블을 Iceberg로도 읽을 수 있게 하는 호환성 기능입니다 |
| Medallion | 메달리온 | Bronze → Silver → Gold 3계층 데이터 설계 패턴입니다 |

---

## 데이터 엔지니어링

| 용어 | 영문 | 설명 |
|------|------|------|
| Auto Loader | 오토 로더 | 클라우드 스토리지의 새 파일을 자동 감지·수집하는 기능입니다 |
| SDP | Spark Declarative Pipelines | 선언적 데이터 변환 파이프라인 프레임워크입니다 (구 DLT) |
| Streaming Table | 스트리밍 테이블 | 새 데이터만 증분 처리하는 테이블입니다 |
| Materialized View | 구체화된 뷰 | 쿼리 결과를 물리적으로 저장하고 자동 갱신하는 뷰입니다 |
| Expectations | 기대값 | SDP에서 데이터 품질 규칙을 정의하는 기능입니다 |
| Lakeflow Connect | 레이크플로우 커넥트 | 외부 소스에서 데이터를 자동 수집하는 관리형 커넥터입니다 |
| Lakeflow Jobs | 레이크플로우 잡스 | 작업 스케줄링·오케스트레이션 서비스입니다 |

---

## AI / ML

| 용어 | 영문 | 설명 |
|------|------|------|
| MLflow | 엠엘플로우 | ML 라이프사이클 관리 오픈소스 플랫폼입니다 |
| Experiment | 실험 | MLflow에서 모델 학습 시도를 관리하는 단위입니다 |
| Run | 런 | 하나의 모델 학습 실행을 의미합니다 |
| Model Registry | 모델 레지스트리 | 모델의 버전을 관리하는 저장소입니다 |
| Model Serving | 모델 서빙 | 모델을 실시간 추론 API로 배포하는 서비스입니다 |
| Feature | 피처 | ML 모델의 입력으로 사용되는 개별 데이터 속성입니다 |
| RAG | Retrieval-Augmented Generation | 문서를 검색하여 LLM의 답변을 보강하는 패턴입니다 |
| Vector Search | 벡터 검색 | 의미적 유사도 기반의 검색 기능입니다 |
| Embedding | 임베딩 | 텍스트를 숫자 벡터로 변환하는 것입니다 |
| Agent | 에이전트 | LLM 기반으로 자율적으로 도구를 사용하여 작업을 수행하는 AI 시스템입니다 |

---

## 거버넌스 / 보안

| 용어 | 영문 | 설명 |
|------|------|------|
| Unity Catalog | 유니티 카탈로그 | Databricks의 통합 데이터 거버넌스 솔루션입니다 |
| Lineage | 리니지 | 데이터의 출처와 흐름을 추적하는 기능입니다 |
| Delta Sharing | 델타 셰어링 | 조직 간 안전한 데이터 공유 프로토콜입니다 |
| Volume | 볼륨 | Unity Catalog에서 파일을 관리하는 저장 공간입니다 |
| Lakebase | 레이크베이스 | Databricks의 관리형 PostgreSQL OLTP 데이터베이스입니다 |
