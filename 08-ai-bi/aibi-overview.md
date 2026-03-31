# AI/BI 개요

## Databricks AI/BI란?

> 💡 **AI/BI** 는 Databricks의 비즈니스 인텔리전스 솔루션으로, 데이터 분석가뿐만 아니라 **비기술 비즈니스 사용자**도 데이터에서 인사이트를 얻을 수 있도록 설계된 도구 모음입니다.

---

## 핵심 구성 요소

| 구성 요소 | 역할 | 대상 사용자 |
|-----------|------|-----------|
| **AI/BI Dashboard (Lakeview)** | SQL 쿼리 결과를 차트, 표, KPI로 시각화합니다 | 분석가, 비즈니스 사용자 |
| **Genie** | 자연어로 데이터에 질문하면 SQL을 자동 생성하여 답변합니다 | 비기술 사용자, 경영진 |
| **Alerts** | SQL 쿼리 결과가 조건을 만족하면 자동으로 알림을 보냅니다 | 운영팀, 분석가 |
| **Metric Views** | 비즈니스 메트릭을 중앙에서 정의하고 일관되게 사용합니다 | 전 조직 (UC 거버넌스 적용) |

---

## 기존 BI 도구와의 차이

| 비교 항목 | 전통적 BI (Tableau, Power BI) | Databricks AI/BI |
|-----------|----------------------------|-----------------|
| **데이터 위치** | BI 도구로 데이터를 추출/복사해야 함 | 레이크하우스에서 **직접 조회** |
| **AI 기능** | 제한적 | **네이티브 AI** — Genie, AI 차트 추천 |
| **거버넌스** | 별도 관리 | **Unity Catalog 통합** |
| **실시간성** | 추출 주기에 의존 | Delta Lake **최신 데이터** 즉시 조회 |
| **비용** | 사용자당 라이선스 | 플랫폼 **내장** (쿼리 비용만) |

> 💡 Databricks AI/BI는 Tableau, Power BI를 **대체**하는 것이 아니라 **보완**합니다. 하이브리드 전략에 대한 자세한 내용은 [하이브리드 BI 전략](./hybrid-bi-strategy.md) 문서를 참고하세요.

---

## 데이터 흐름

| 단계 | 구성 요소 | 설명 |
|------|----------|------|
| **데이터 저장** | Gold 테이블 (Delta Lake) | 분석에 최적화된 집계 테이블입니다 |
| **쿼리 실행** | SQL Warehouse (Serverless) | Photon 엔진으로 고속 쿼리를 처리합니다 |
| **시각화** | AI/BI Dashboard | 차트, 표, KPI 카운터로 시각화합니다 |
| **자연어 분석** | Genie Space | 자연어 질문 → SQL 자동 생성 → 답변 |
| **모니터링** | Alerts | 조건 충족 시 Slack/이메일 알림 |
| **외부 BI** | Tableau, Power BI (JDBC/ODBC) | SQL Warehouse에 직접 연결 |

> 💡 **Gold 테이블을 BI 소스로 사용**: 대시보드는 Medallion 아키텍처의 **Gold 계층** 테이블을 소스로 사용합니다. Gold는 이미 집계되어 있으므로 쿼리가 간결하고 빠릅니다.

---

## 역할별 활용

| 역할 | 주요 도구 | 시나리오 |
|------|----------|---------|
| **데이터 분석가** | Dashboard, SQL Editor | 정기 리포트, 데이터 탐색 |
| **비즈니스 사용자** | Genie | "이번 달 매출이 목표 대비 어떤가요?" |
| **경영진** | Genie, Dashboard | KPI 모니터링, 즉석 질문 |
| **운영팀** | Alerts | 이상 거래 감지, SLA 위반 알림 |
| **데이터 엔지니어** | Alerts, Dashboard | 파이프라인 모니터링, 데이터 품질 |

---

## 시작하기

| 요구 사항 | 설명 |
|-----------|------|
| **SQL Warehouse** | Serverless 권장. 쿼리 실행에 필요합니다 |
| **Unity Catalog** | 거버넌스와 권한 관리에 필요합니다 |
| **Gold 테이블** | 분석용 집계 테이블이 있어야 합니다 |

---

## 주의사항

| 항목 | 설명 |
|------|------|
| **Genie 정확도** | 테이블/컬럼의 COMMENT가 부실하면 정확도가 떨어집니다. 메타데이터 품질이 핵심입니다 |
| **Genie Space 테이블 수** | 20개+ 테이블을 추가하면 정확도가 저하됩니다. 주제별로 분리하세요 |
| **대시보드 공유 권한** | 공유받은 사용자에게 테이블 SELECT 권한이 없으면 데이터가 표시되지 않습니다 |

---

## 더 알아보기

| 주제 | 문서 |
|------|------|
| 대시보드 상세 | [AI/BI 대시보드](./lakeview-dashboards.md) |
| Genie 상세 | [Genie](./genie.md) |
| 알림 상세 | [알림과 스케줄링](./alerts-and-scheduling.md) |
| 기술 아키텍처 | [AI/BI 아키텍처 심화](./aibi-architecture.md) |
| 외부 BI 연동 | [하이브리드 BI 전략](./hybrid-bi-strategy.md) |

---

## 참고 링크

- [Databricks: AI/BI](https://docs.databricks.com/aws/en/aibi/)
- [Databricks: AI/BI Dashboards](https://docs.databricks.com/aws/en/aibi/dashboards/)
- [Databricks: Genie](https://docs.databricks.com/aws/en/aibi/genie/)
- [Databricks: Alerts](https://docs.databricks.com/aws/en/sql/user/alerts/)
