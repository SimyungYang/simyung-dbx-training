# Databricks SQL 개요

## Databricks SQL이란?

> 💡 **Databricks SQL (DBSQL)**은 Delta Lake 위의 데이터를 **SQL로 직접 조회하고 분석**할 수 있는 Databricks의 SQL 분석 환경입니다. 별도의 데이터 복사 없이, 레이크하우스 데이터에 대해 **웨어하우스급 SQL 성능**을 제공합니다.

---

## DBSQL의 핵심 가치

| 가치 | 설명 |
|------|------|
| **데이터 복사 불필요** | 레이크하우스의 Delta 테이블을 직접 조회합니다. ETL로 별도 웨어하우스에 복사할 필요가 없습니다 |
| **웨어하우스급 성능** | Photon 엔진이 기본 활성화되어 기존 웨어하우스(Redshift, Snowflake) 수준의 SQL 성능을 제공합니다 |
| **통합 거버넌스** | Unity Catalog의 권한이 SQL 쿼리에도 동일하게 적용됩니다 |
| **AI 함수 내장** | SQL 안에서 직접 LLM을 호출하여 분류, 추출, 요약을 수행할 수 있습니다 |
| **Serverless** | SQL Warehouse가 수 초 만에 시작되고, 유휴 시 자동 중지됩니다 |

---

## 주요 구성 요소

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **SQL Editor** | SQL 작성·실행 | 자동완성, 구문 강조, 실행 계획 분석을 지원하는 웹 기반 편집기입니다 |
| **SQL Warehouse** | 쿼리 실행 엔진 | SQL 쿼리를 실행하는 전용 서버리스 컴퓨팅 리소스입니다 |
| **AI/BI Dashboard** | 시각화 | SQL 쿼리 결과를 차트, 표, KPI 카운터로 시각화합니다 |
| **Query History** | 이력 관리 | 실행한 쿼리의 이력, 성능, 비용을 확인합니다 |
| **Alerts** | 자동 알림 | 쿼리 결과가 조건을 만족하면 알림을 발송합니다 |

---

## 기존 데이터 웨어하우스와의 비교

| 비교 항목 | Databricks SQL | Snowflake | Amazon Redshift |
|-----------|---------------|-----------|-----------------|
| **아키텍처** | 레이크하우스 (오픈 포맷) | 독자 포맷 | 독자 포맷 |
| **데이터 저장** | 고객 클라우드 (S3/ADLS) | Snowflake 관리 | Redshift 관리 |
| **파일 포맷** | Delta Lake (Parquet 기반, 오픈) | 독자 포맷 | 독자 포맷 |
| **ML/AI 통합** | ✅ 네이티브 (MLflow, Agent) | 제한적 | 제한적 (SageMaker 연동) |
| **AI 함수** | ✅ ai_query, ai_classify 등 | Cortex (유사) | 없음 |
| **스트리밍** | ✅ Structured Streaming 통합 | Snowpipe | Kinesis 연동 |
| **거버넌스** | Unity Catalog (통합) | 내장 거버넌스 | IAM + Lake Formation |
| **Serverless** | ✅ 수 초 시작 | ✅ | Serverless 옵션 있음 |
| **Photon 엔진** | ✅ C++ 네이티브 | Snowflake 엔진 | - |

---

## 언제 DBSQL을 사용하나요?

| 사용 사례 | 설명 | 대안 |
|-----------|------|------|
| **대화형 SQL 분석** | 데이터를 탐색하고, 집계하고, 패턴을 찾을 때 | 없음 (DBSQL이 최적) |
| **대시보드 생성** | AI/BI Dashboard에서 시각화할 데이터를 쿼리할 때 | Tableau, Power BI |
| **BI 도구 연결** | Tableau, Power BI 등 외부 BI 도구의 백엔드로 사용할 때 | 없음 (JDBC/ODBC 연결) |
| **데이터 검증** | ETL 파이프라인의 결과를 SQL로 검증할 때 | Notebook |
| **Ad-hoc 분석** | 비정기적인 일회성 분석을 수행할 때 | Notebook |
| **AI 분석** | SQL 안에서 LLM으로 텍스트 분류, 요약을 수행할 때 | Python + LLM API |
| **알림 모니터링** | 조건 기반 알림으로 이상 감지를 할 때 | 외부 모니터링 도구 |

---

## SQL Editor 주요 기능

| 기능 | 설명 |
|------|------|
| **자동완성** | 테이블명, 컬럼명, SQL 키워드를 자동 완성합니다 |
| **구문 강조** | SQL 구문을 색상으로 구분하여 가독성을 높입니다 |
| **실행 계획** | `EXPLAIN` 없이도 쿼리 실행 계획을 시각적으로 분석합니다 |
| **매개변수** | `{{parameter_name}}` 형태로 동적 매개변수를 사용합니다 |
| **스니펫** | 자주 사용하는 SQL 패턴을 저장하고 재사용합니다 |
| **버전 이력** | 쿼리의 변경 이력이 자동으로 저장됩니다 |
| **공유** | 쿼리를 다른 사용자와 공유할 수 있습니다 |

> 🆕 **Query Tags**: SQL Warehouse에서 실행되는 쿼리에 태그를 달아 비즈니스 맥락별로 비용을 추적할 수 있습니다.

> 🆕 **Warehouse Activity Details**: SQL Warehouse의 쿼리 활동, 세션, 유휴 상태를 색상 코딩된 시각화로 확인할 수 있습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Databricks SQL** | 레이크하우스 데이터에 대한 SQL 분석 환경입니다 |
| **SQL Warehouse** | SQL 쿼리를 실행하는 서버리스 컴퓨팅입니다. Photon 엔진이 기본 활성화됩니다 |
| **SQL Editor** | 웹 기반 SQL 작성·실행 도구입니다 |
| **AI 함수** | SQL 안에서 LLM을 호출할 수 있습니다 |
| **오픈 포맷** | Delta Lake(Parquet 기반)로 저장되어 벤더 종속이 없습니다 |

---

## 참고 링크

- [Databricks: Databricks SQL](https://docs.databricks.com/aws/en/sql/)
- [Databricks: SQL Editor](https://docs.databricks.com/aws/en/sql/user/queries/)
- [Azure Databricks: Data Warehousing](https://learn.microsoft.com/en-us/azure/databricks/sql/)
