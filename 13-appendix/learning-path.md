# 학습 로드맵 (Learning Path)

## 왜 학습 순서가 중요한가요?

Databricks는 데이터 엔지니어링, 분석, 머신러닝, 거버넌스를 아우르는 **통합 플랫폼**입니다. 각 영역이 서로 밀접하게 연결되어 있기 때문에, 기초 개념 없이 고급 주제를 학습하면 이해에 어려움이 생깁니다.

예를 들어, Model Serving(모델 서빙)을 이해하려면 먼저 Spark 클러스터와 Delta Lake를 알아야 하고, Unity Catalog의 권한 체계를 이해해야 프로덕션 배포가 가능합니다. 이 로드맵은 **학습 의존성**을 고려하여 최적의 순서를 안내합니다.

> 💡 **Tip**: 자신의 역할에 맞는 경로를 먼저 따라가되, 시간이 허용되면 다른 역할의 경로도 살펴보시는 것을 권장합니다. 플랫폼 전체를 이해하면 팀 간 협업이 훨씬 수월해집니다.

---

## 전체 학습 경로 흐름도

**전체 학습 경로**

| 순서 | 섹션 | 내용 |
|------|------|------|
| 00 | 선행 지식 | RDB, 빅데이터 역사 |
| 01 | 데이터 기초 | DW vs DL, ETL/ELT |
| 02 | Databricks 개요 | 플랫폼, Workspace |
| 03 | 레이크하우스 | Delta Lake, Medallion |
| 04 | 컴퓨트 | Spark, Cluster, SQL WH |
| 05 | 데이터 엔지니어링 | Auto Loader, SDP, Jobs |
| 06 | 데이터 웨어하우징 | DBSQL, AI 함수 |
| 07 | Unity Catalog | 거버넌스, 권한, 리니지 |
| 08 | AI/BI | Dashboard, Genie, Alerts |
| 09 | 머신러닝 | MLflow, Feature, Serving |
| 10 | 에이전트 개발 | RAG, Agent Framework |
| 11 | Lakebase | PostgreSQL 호환 OLTP |
| 12 | 보안/거버넌스 | ID 관리, 네트워크 보안 |
| 13 | 부록 | Apps, 학습 경로 |

> 실선 화살표는 **필수 선행 학습**, 점선 화살표는 **권장 선행 학습**을 의미합니다.

---

## 역할별 학습 경로

### 1. 데이터 엔지니어 (Data Engineer)

데이터 파이프라인 설계, 구축, 운영을 담당하는 역할입니다.

**데이터 엔지니어 학습 경로**

| 순서 | 섹션 | 비고 |
|------|------|------|
| 1 | 00 선행 지식 | - |
| 2 | 01 데이터 기초 | - |
| 3 | 02 Databricks 개요 | - |
| 4 | 03 레이크하우스 | - |
| 5 | 04 컴퓨트 | - |
| 6 | 05 데이터 엔지니어링 | 핵심 |
| 7 | 06 웨어하우징 | - |
| 8 | 07 Unity Catalog | - |

| 단계 | 학습 내용 | 예상 소요시간 | 관련 문서 |
|------|----------|-------------|----------|
| **00 선행 지식**| RDB 기초, Star/Snowflake 스키마, 빅데이터 역사 | 3~4시간 | `00-prerequisites/` |
| **01 데이터 기초**| DW vs DL, ETL/ELT, 배치/스트리밍 | 2~3시간 | `01-data-fundamentals/` |
| **02 Databricks 개요**| 플랫폼 아키텍처, Workspace, Notebook | 2시간 | `02-databricks-overview/` |
| **03 레이크하우스**| Delta Lake, ACID, Medallion 아키텍처 | 3~4시간 | `03-lakehouse-architecture/` |
| **04 컴퓨트**| Spark 기초, 클러스터, SQL Warehouse | 3시간 | `04-compute-workspace/` |
| **05 데이터 엔지니어링**| Auto Loader, SDP, Lakeflow Connect/Jobs | **6~8시간**| `05-data-engineering/` |
| **06 웨어하우징**| DBSQL, 쿼리 최적화 | 2~3시간 | `06-data-warehousing/` |
| **07 Unity Catalog**| 네임스페이스, 권한 관리, 리니지 | 3~4시간 | `07-unity-catalog/` |

** 추가 권장**: 12. 보안/거버넌스 (프로덕션 환경 설정)

---

### 2. 데이터 분석가 (Data Analyst)

SQL과 시각화를 활용한 데이터 분석, 리포팅을 담당하는 역할입니다.

**데이터 분석가 학습 경로**

| 순서 | 섹션 | 비고 |
|------|------|------|
| 1 | 00 선행 지식 (RDB, 스키마) | - |
| 2 | 01 데이터 기초 | - |
| 3 | 02 Databricks 개요 | - |
| 4 | 06 웨어하우징 | 핵심 |
| 5 | 08 AI/BI | 핵심 |
| 6 | 07 Unity Catalog | - |

| 단계 | 학습 내용 | 예상 소요시간 | 관련 문서 |
|------|----------|-------------|----------|
| **00 선행 지식**| RDB 기초, Star/Snowflake 스키마 | 2~3시간 | `00-prerequisites/` |
| **01 데이터 기초**| DW vs DL 개념 이해 | 1~2시간 | `01-data-fundamentals/` |
| **02 Databricks 개요**| Workspace, Notebook 사용법 | 1~2시간 | `02-databricks-overview/` |
| **06 웨어하우징**| DBSQL, SQL Warehouse, AI 함수, 쿼리 최적화 | **4~5시간**| `06-data-warehousing/` |
| **08 AI/BI**| Lakeview 대시보드, Genie, 알림 | **3~4시간**| `08-ai-bi/` |
| **07 Unity Catalog**| 데이터 검색, 리니지 활용 | 2시간 | `07-unity-catalog/` |

** 추가 권장**: 03. 레이크하우스 (Medallion 아키텍처 이해)

---

### 3. ML 엔지니어 / 데이터 과학자

모델 학습, 실험 관리, 모델 배포를 담당하는 역할입니다.

**ML/AI 엔지니어 학습 경로**

| 순서 | 섹션 | 비고 |
|------|------|------|
| 1 | 01 데이터 기초 | - |
| 2 | 02 Databricks 개요 | - |
| 3 | 04 컴퓨트 | - |
| 4 | 09 머신러닝 | 핵심 |
| 5 | 10 에이전트 개발 | 핵심 |

| 단계 | 학습 내용 | 예상 소요시간 | 관련 문서 |
|------|----------|-------------|----------|
| **01 데이터 기초**| 데이터 파이프라인 이해 | 1~2시간 | `01-data-fundamentals/` |
| **02 Databricks 개요**| Workspace, Notebook | 1~2시간 | `02-databricks-overview/` |
| **04 컴퓨트**| Spark 기초, ML Runtime, GPU 클러스터 | 3시간 | `04-compute-workspace/` |
| **09 머신러닝**| MLflow, Feature Engineering, Model Serving | **8~10시간**| `09-machine-learning/` |
| **10 에이전트 개발**| RAG, Vector Search, Agent Evaluation | **6~8시간**| `10-agent-development/` |

** 추가 권장**: 03. 레이크하우스, 07. Unity Catalog (모델 거버넌스), 11. Lakebase

---

### 4. 플랫폼 관리자 (Admin)

워크스페이스 설정, 보안, 비용 관리를 담당하는 역할입니다.

**플랫폼 관리자 학습 경로**

| 순서 | 섹션 | 비고 |
|------|------|------|
| 1 | 02 Databricks 개요 | - |
| 2 | 04 컴퓨트 | - |
| 3 | 07 Unity Catalog | 핵심 |
| 4 | 12 보안/거버넌스 | 핵심 |

| 단계 | 학습 내용 | 예상 소요시간 | 관련 문서 |
|------|----------|-------------|----------|
| **02 Databricks 개요**| 아키텍처(Control Plane/Data Plane), 배포 모델 | 2시간 | `02-databricks-overview/` |
| **04 컴퓨트**| 클러스터 정책, SQL Warehouse 설정 | 3시간 | `04-compute-workspace/` |
| **07 Unity Catalog**| 메타스토어 설정, 권한 체계, 리니지 | **4~5시간**| `07-unity-catalog/` |
| **12 보안/거버넌스**| 네트워크 보안, 암호화, 감사 로그, 시스템 테이블 | **5~6시간**| `12-security-governance/` |

** 추가 권장**: 03. 레이크하우스, 06. 웨어하우징, 13. 부록 (CLI, Asset Bundles)

---

## 전체 학습 순서 (처음부터 끝까지)

모든 내용을 체계적으로 학습하고 싶다면, 아래 순서대로 진행하시면 됩니다.

| 순서 | 섹션 | 예상 소요시간 | 난이도 |
|------|------|-------------|--------|
| 1 | 00. 선행 지식 | 3~4시간 | 입문 |
| 2 | 01. 데이터 기초 | 2~3시간 | 입문 |
| 3 | 02. Databricks 개요 | 2시간 | 입문 |
| 4 | 03. 레이크하우스 | 3~4시간 | 초급 |
| 5 | 04. 컴퓨트 | 3시간 | 초급 |
| 6 | 05. 데이터 엔지니어링 | 6~8시간 | 중급 |
| 7 | 06. 데이터 웨어하우징 | 3~4시간 | 중급 |
| 8 | 07. Unity Catalog | 4~5시간 | 중급 |
| 9 | 08. AI/BI | 2~3시간 | 초급 |
| 10 | 09. 머신러닝 | 8~10시간 | 중급~고급 |
| 11 | 10. 에이전트 개발 | 6~8시간 | 고급 |
| 12 | 11. Lakebase | 2~3시간 | 중급 |
| 13 | 12. 보안/거버넌스 | 4~5시간 | 중급 |
| 14 | 13. 부록 | 2~3시간 | 초급~중급 |
| | **총 예상 소요시간** | **약 50~65시간** | |

---

## Databricks 공식 인증 소개

Databricks는 역할별 공식 인증 시험을 제공합니다. 이 교육 자료를 학습한 후 인증에 도전해 보시기 바랍니다.

### Associate 레벨 (입문~중급)

| 인증명 | 대상 역할 | 관련 섹션 | 시험 개요 |
|--------|----------|----------|----------|
| **Databricks Certified Data Engineer Associate**| 데이터 엔지니어 | 03, 04, 05, 07 | Delta Lake, ELT, 파이프라인, 거버넌스 기초 |
| **Databricks Certified Data Analyst Associate**| 데이터 분석가 | 06, 08 | DBSQL, 대시보드, 쿼리 최적화 |
| **Databricks Certified Machine Learning Associate**| ML 엔지니어 | 09 | MLflow, Feature Store, Model Serving 기초 |
| **Databricks Certified Generative AI Engineer Associate**| AI 엔지니어 | 09, 10 | RAG, Agent, LLM 서빙 기초 |

### Professional 레벨 (중급~고급)

| 인증명 | 대상 역할 | 관련 섹션 | 시험 개요 |
|--------|----------|----------|----------|
| **Databricks Certified Data Engineer Professional**| 시니어 DE | 03, 04, 05, 07, 12 | 고급 파이프라인, 성능 최적화, 보안 |
| **Databricks Certified Machine Learning Professional**| 시니어 ML | 09, 10 | 고급 MLOps, 프로덕션 배포, 모니터링 |

> 💡 ** 인증 준비 팁**: Databricks Academy의 무료 학습 과정을 먼저 수강하고, 이 교육 자료로 개념을 보충한 뒤, 공식 Practice Exam으로 실력을 점검하시는 것을 권장합니다.

---

## 추천 학습 리소스

### 공식 리소스

| 리소스 | URL | 설명 |
|--------|-----|------|
| **Databricks Academy**| [academy.databricks.com](https://academy.databricks.com) | 공식 온라인 교육 과정 (무료/유료). 인증 준비에 가장 적합합니다 |
| **Databricks Documentation**| [docs.databricks.com](https://docs.databricks.com) | 공식 기술 문서. 최신 기능과 API 레퍼런스를 확인할 수 있습니다 |
| **Databricks Blog**| [databricks.com/blog](https://www.databricks.com/blog) | 신기능 소개, 아키텍처 딥다이브, 고객 사례를 확인할 수 있습니다 |
| **Release Notes**| [docs.databricks.com/release-notes](https://docs.databricks.com/aws/en/release-notes/) | 최신 기능 업데이트와 변경 사항을 확인합니다 |

### 커뮤니티 & 오픈소스

| 리소스 | URL | 설명 |
|--------|-----|------|
| **Databricks Community**| [community.databricks.com](https://community.databricks.com) | 커뮤니티 포럼. 질문과 답변, 모범 사례를 공유합니다 |
| **Delta Lake Docs**| [docs.delta.io](https://docs.delta.io) | Delta Lake 오픈소스 문서입니다 |
| **MLflow Docs**| [mlflow.org/docs](https://mlflow.org/docs/latest/) | MLflow 오픈소스 문서입니다 |
| **Apache Spark Docs**| [spark.apache.org](https://spark.apache.org/docs/latest/) | Apache Spark 공식 문서입니다 |
| **GitHub - databricks**| [github.com/databricks](https://github.com/databricks) | 공식 예제 노트북, SDK, 도구의 소스 코드입니다 |

### 컨퍼런스 & 영상

| 리소스 | 설명 |
|--------|------|
| **Data+AI Summit**| Databricks 연례 컨퍼런스. 최신 기술 발표와 고객 사례를 확인할 수 있습니다. 영상은 YouTube에서 무료 시청 가능합니다 |
| **Databricks YouTube**| 튜토리얼, 데모, 웨비나 영상이 풍부합니다 |

---

## 실전 프로젝트 아이디어

학습 내용을 실무에 적용해 보는 것이 가장 효과적인 학습 방법입니다. 단계별로 도전해 보시기 바랍니다.

### 입문 프로젝트 (관련 섹션: 01~04)

| 프로젝트 | 설명 | 핵심 기술 |
|---------|------|----------|
| **CSV 데이터 Delta 변환**| 공개 CSV 데이터를 Delta 테이블로 변환하고 Time Travel을 체험합니다 | Delta Lake, Notebook |
| **SQL 분석 대시보드**| 공개 데이터셋으로 DBSQL 대시보드를 만들어 봅니다 | DBSQL, Lakeview |

### 중급 프로젝트 (관련 섹션: 05~08)

| 프로젝트 | 설명 | 핵심 기술 |
|---------|------|----------|
| **Medallion 파이프라인**| Bronze → Silver → Gold 3계층 데이터 파이프라인을 구축합니다 | Auto Loader, SDP, Jobs |
| ** 거버넌스 설정**| Unity Catalog로 카탈로그/스키마를 설계하고 권한을 설정합니다 | UC, RBAC, 리니지 |
| ** 실시간 대시보드**| 스트리밍 데이터를 수집하여 실시간 모니터링 대시보드를 만듭니다 | Streaming Table, Lakeview |

### 고급 프로젝트 (관련 섹션: 09~12)

| 프로젝트 | 설명 | 핵심 기술 |
|---------|------|----------|
| **ML 모델 서빙**| 분류 모델을 학습하고 실시간 서빙 엔드포인트로 배포합니다 | MLflow, Model Serving, Feature Store |
| **RAG 챗봇**| 사내 문서를 기반으로 Q&A 챗봇을 구축합니다 | Vector Search, Agent, RAG |
| **E2E MLOps**| 데이터 수집 → 피처 엔지니어링 → 학습 → 서빙 → 모니터링 전체 파이프라인을 자동화합니다 | Jobs, MLflow, Inference Table, Lakehouse Monitoring |

---

## FAQ (자주 묻는 질문)

### Q1. 프로그래밍 경험이 없어도 학습할 수 있나요?

** 네, 가능합니다.**데이터 분석가 경로는 SQL 중심으로 구성되어 있어 프로그래밍 경험이 적어도 학습할 수 있습니다. 다만, 데이터 엔지니어나 ML 엔지니어 경로는 Python 기초 지식이 필요합니다.

### Q2. 어떤 클라우드에서 실습하는 것이 좋나요?

이 교육 자료는 ** 클라우드에 관계없이**적용 가능합니다. AWS, Azure, GCP 모두 Databricks를 지원합니다. 실습 환경은 [Databricks Community Edition](https://community.cloud.databricks.com/)(무료)이나 회사에서 제공하는 워크스페이스를 활용하시면 됩니다.

### Q3. 학습 완료까지 얼마나 걸리나요?

역할별로 다릅니다.

| 역할 | 핵심 경로 소요시간 | 전체 학습 소요시간 |
|------|-----------------|-----------------|
| 데이터 분석가 | 약 15~20시간 | 약 25~30시간 |
| 데이터 엔지니어 | 약 25~30시간 | 약 40~50시간 |
| ML 엔지니어 | 약 20~25시간 | 약 40~50시간 |
| 플랫폼 관리자 | 약 15~20시간 | 약 30~40시간 |

### Q4. 인증 시험은 어디서 응시하나요?

Databricks 인증 시험은 ** 온라인(Kryterion 플랫폼)**으로 응시할 수 있습니다. 자택이나 사무실에서 웹캠과 화면 공유를 통해 감독 하에 시험을 치릅니다. 시험 비용은 인증별로 $200 USD 내외입니다.

### Q5. 이 교육 자료와 Databricks Academy의 차이는 무엇인가요?

| 비교 항목 | 이 교육 자료 | Databricks Academy |
|----------|------------|-------------------|
| ** 언어**| 한국어 | 영어 |
| ** 대상**| 데이터 초보자 ~ 중급자 | 초급 ~ 고급 |
| ** 형태**| 읽기 자료 (Markdown) | 대화형 강의 + 실습 노트북 |
| ** 비용**| 무료 | 일부 무료 / 유료 |
| ** 업데이트**| 수시 업데이트 | 정기 업데이트 |

두 자료를 ** 병행하여 학습**하시면 가장 효과적입니다. 이 자료로 개념을 한국어로 이해하고, Academy에서 실습으로 체득하시기 바랍니다.

---

## 참고 링크

- [Databricks Academy](https://academy.databricks.com)
- [Databricks Certifications](https://www.databricks.com/learn/certification)
- [Databricks Community Edition](https://community.cloud.databricks.com/)
- [Databricks Blog](https://www.databricks.com/blog)
- [Data+AI Summit 영상](https://www.youtube.com/@Databricks)
