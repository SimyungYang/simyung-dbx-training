# Databricks SQL 개요

## Databricks SQL이란?

> 💡 **Databricks SQL (DBSQL)**은 Delta Lake 위의 데이터를 **SQL로 직접 조회하고 분석**할 수 있는 Databricks의 SQL 분석 인터페이스입니다. 별도의 데이터 복사 없이 레이크하우스 데이터에 대해 웨어하우스급 SQL 성능을 제공합니다.

---

## 주요 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| **SQL Editor** | 웹 기반 SQL 쿼리 편집기. 자동완성, 구문 강조를 지원합니다 |
| **SQL Warehouse** | SQL 쿼리를 실행하는 전용 컴퓨팅 리소스입니다 |
| **Query History** | 실행한 쿼리의 이력, 성능, 비용을 확인할 수 있습니다 |
| **Alerts** | 쿼리 결과가 특정 조건을 만족하면 알림을 보냅니다 |

---

## 언제 DBSQL을 사용하나요?

| 사용 사례 | 설명 |
|-----------|------|
| **대화형 SQL 분석** | 데이터를 탐색하고, 집계하고, 패턴을 찾을 때 |
| **대시보드 생성** | AI/BI Dashboard에서 시각화할 데이터를 쿼리할 때 |
| **BI 도구 연결** | Tableau, Power BI 등 외부 BI 도구의 백엔드로 사용할 때 |
| **데이터 검증** | ETL 파이프라인의 결과를 검증할 때 |
| **Ad-hoc 분석** | 비정기적인 일회성 분석을 수행할 때 |

> 🆕 **Query Tags**: SQL Warehouse에 쿼리 태그를 달아 비즈니스 맥락별로 비용을 추적할 수 있는 기능이 추가되었습니다.

---

## 참고 링크

- [Databricks: Databricks SQL](https://docs.databricks.com/aws/en/sql/)
