# AI/BI 개요

## Databricks AI/BI란?

> 💡 **AI/BI**는 Databricks의 비즈니스 인텔리전스 솔루션으로, 데이터 분석가뿐만 아니라 **비기술 비즈니스 사용자**도 데이터에서 인사이트를 얻을 수 있도록 설계된 도구 모음입니다. AI를 활용하여 대시보드를 자동으로 구성하고, 자연어로 데이터에 질문할 수 있습니다.

---

## AI/BI를 구성하는 핵심 요소

| 구성 요소 | 역할 | 출력 |
|-----------|------|------|
| **Delta Lake** | 레이크하우스 데이터 소스입니다 | SQL Warehouse로 전달 |
| **SQL Warehouse** | 쿼리를 실행합니다 | AI/BI 도구에 데이터 제공 |
| **AI/BI Dashboard (Lakeview)** | 시각화를 제공합니다 | 차트, 표, KPI 카운터 |
| **Genie** | 자연어를 SQL로 변환합니다 | 질문하면 바로 답변 |
| **Alerts** | 조건을 모니터링합니다 | 이메일/Slack 알림 발송 |

| 구성 요소 | 역할 | 대상 사용자 |
|-----------|------|-----------|
| **AI/BI Dashboard** | SQL 쿼리 결과를 **차트, 표, KPI 카운터**로 시각화합니다. 필터, 매개변수, 페이지 기능을 지원합니다 | 분석가, 비즈니스 사용자 |
| **Genie** | **자연어**로 데이터에 질문하면 SQL 쿼리를 자동 생성하여 답변합니다. 비기술 사용자도 데이터를 탐색할 수 있습니다 | 비즈니스 사용자, 경영진 |
| **Alerts** | SQL 쿼리 결과가 특정 **조건을 만족하면 자동으로 알림**을 보냅니다. 이상 감지, 임계치 모니터링에 사용합니다 | 운영팀, 분석가 |

---

## 기존 BI 도구와의 차이

Databricks AI/BI는 전통적인 BI 도구(Tableau, Power BI 등)와 다음과 같은 차이가 있습니다.

| 비교 항목 | 전통적 BI (Tableau, Power BI) | Databricks AI/BI |
|-----------|----------------------------|-----------------|
| **데이터 위치** | BI 도구로 데이터를 추출/복사해야 함 | 레이크하우스에서 **직접 조회** (복사 불필요) |
| **AI 기능** | 제한적 (최근 추가 중) | **네이티브 AI** — Genie 자연어 분석, AI 기반 대시보드 자동 구성 |
| **데이터 거버넌스** | 별도 관리 필요 | **Unity Catalog 통합** — 동일한 권한 체계가 대시보드에도 적용 |
| **실시간성** | 데이터 추출 주기에 의존 | Delta Lake의 **최신 데이터를 바로 조회** |
| **비용** | BI 도구 라이선스 + 데이터 추출 비용 | Databricks 플랫폼에 **내장** (추가 라이선스 불필요) |
| **협업** | 도구별 공유 체계 | Workspace 내 **통합 공유** |

> 💡 Databricks AI/BI는 Tableau, Power BI를 **대체**하는 것이 아니라, **보완**하는 위치입니다. 복잡한 대시보드나 기존 투자가 있는 경우 외부 BI 도구를 계속 사용하면서, 간단한 분석이나 실시간 모니터링에는 AI/BI를 활용하는 하이브리드 전략이 일반적입니다. 외부 BI 도구는 SQL Warehouse에 JDBC/ODBC로 연결하여 사용합니다.

---

## AI/BI의 데이터 흐름

**데이터 레이어**

| 단계 | 설명 |
|------|------|
| Bronze | 원본 데이터 |
| Silver | 정제된 데이터 |
| Gold (분석용 테이블) | 분석에 최적화된 데이터 |

**BI 레이어**

| 구성 요소 | 연결 | 설명 |
|-----------|------|------|
| **SQL Warehouse** | Gold 테이블에서 쿼리 | 쿼리 엔진 역할을 합니다 |
| **AI/BI Dashboard** | SQL Warehouse | 시각화 대시보드입니다 |
| **Genie Space** | SQL Warehouse | 자연어 데이터 분석입니다 |
| **Alerts** | SQL Warehouse | 조건 기반 알림입니다 |
| **외부 BI 도구** | SQL Warehouse (JDBC/ODBC) | Tableau, Power BI, Looker 등과 연동합니다 |

> 💡 **Gold 테이블을 BI 소스로 사용**: AI/BI 대시보드는 주로 Medallion 아키텍처의 **Gold 계층** 테이블을 데이터 소스로 사용합니다. Gold 테이블은 이미 비즈니스 목적에 맞게 집계되어 있으므로, 대시보드 쿼리가 간결하고 빠릅니다.

---

## 역할별 AI/BI 활용

| 역할 | 주로 사용하는 기능 | 사용 시나리오 |
|------|-------------------|-------------|
| **데이터 분석가** | AI/BI Dashboard, SQL Editor | 정기 리포트 작성, 데이터 탐색, KPI 대시보드 |
| **비즈니스 사용자** | Genie | "이번 달 매출이 목표 대비 어떤가요?" 같은 자연어 질문 |
| **경영진** | Genie, Dashboard (공유) | 핵심 KPI 모니터링, 즉석 데이터 질문 |
| **운영팀** | Alerts | 이상 거래 감지, SLA 위반 알림, 시스템 모니터링 |
| **데이터 엔지니어** | Alerts, Dashboard | 파이프라인 모니터링, 데이터 품질 대시보드 |

---

## 주요 기능 요약

### AI/BI Dashboard (Lakeview)

- **데이터셋 정의**: SQL 쿼리로 데이터를 가져옵니다
- **위젯**: Bar, Line, Pie, Scatter, Counter, Table, Pivot 등 다양한 차트 유형을 제공합니다
- **필터**: 날짜 범위, 드롭다운, 텍스트 필터로 대화형 탐색이 가능합니다
- **매개변수**: 위젯 간 연동되는 동적 매개변수를 지원합니다
- **멀티 페이지**: 하나의 대시보드에 여러 페이지를 구성할 수 있습니다
- **AI 자동 구성**: 데이터셋을 추가하면 AI가 적절한 차트를 자동으로 추천합니다
- **공유**: Workspace 내 사용자/그룹에게 공유, 스케줄 이메일, PDF 다운로드를 지원합니다

### Genie

- **자연어 질문**: "지난 달 서울 매출"처럼 일상 언어로 질문합니다
- **SQL 자동 생성**: Genie가 질문을 이해하고 적절한 SQL을 작성합니다
- **Genie Space**: 특정 테이블과 지시사항을 설정한 전용 공간입니다
- **인증된 답변**: 자주 묻는 질문에 대해 검증된 SQL을 미리 등록합니다
- **데이터 보안**: Unity Catalog 권한이 그대로 적용되어, 권한 없는 데이터는 조회할 수 없습니다

### Alerts

- **조건 기반 알림**: "일별 에러 수 > 100" 같은 조건을 설정합니다
- **다양한 알림 채널**: 이메일, Slack Webhook, PagerDuty, Microsoft Teams
- **스케줄 체크**: 1분~1일 간격으로 조건을 주기적으로 확인합니다
- **알림 상태**: Triggered, OK, Unknown 세 가지 상태를 추적합니다

---

## 시작하기

AI/BI를 사용하려면 다음이 필요합니다.

| 요구 사항 | 설명 |
|-----------|------|
| **SQL Warehouse** | Serverless SQL Warehouse를 권장합니다. 쿼리 실행에 필요합니다 |
| **Unity Catalog** | 데이터 거버넌스와 권한 관리에 필요합니다 |
| **Gold 테이블** | 분석용으로 정제된 테이블이 있어야 의미 있는 대시보드를 만들 수 있습니다 |
| **CAN USE 권한** | SQL Warehouse에 대한 사용 권한이 필요합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **AI/BI Dashboard** | SQL 기반 데이터 시각화. 차트, 필터, 매개변수, 멀티 페이지를 지원합니다 |
| **Genie** | 자연어로 데이터에 질문. 비기술 사용자도 데이터를 탐색할 수 있습니다 |
| **Alerts** | 조건 기반 자동 알림. 이상 감지와 모니터링에 사용합니다 |
| **Lakeview** | AI/BI Dashboard의 기술적 이름입니다 |
| **Unity Catalog 통합** | 대시보드에도 동일한 데이터 권한이 적용됩니다 |

---

## 참고 링크

- [Databricks: AI/BI](https://docs.databricks.com/aws/en/aibi/)
- [Databricks: AI/BI Dashboards](https://docs.databricks.com/aws/en/aibi/dashboards/)
- [Databricks: Genie](https://docs.databricks.com/aws/en/aibi/genie/)
- [Databricks: Alerts](https://docs.databricks.com/aws/en/sql/user/alerts/)
- [Azure Databricks: Business Intelligence](https://learn.microsoft.com/en-us/azure/databricks/aibi/)
