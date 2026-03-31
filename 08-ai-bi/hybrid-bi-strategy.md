# 하이브리드 BI 전략

## 왜 하이브리드 전략인가?

대부분의 엔터프라이즈에서는 Tableau, Power BI 등 기존 BI 도구에 이미 상당한 투자가 있습니다. Databricks AI/BI는 이를 **대체**하는 것이 아니라 **보완**합니다.

---

## 역할 분담 패턴

| 용도 | 권장 도구 | 이유 |
|------|----------|------|
| **복잡한 정적 보고서** | Tableau/Power BI | 기존 인프라 활용, 풍부한 시각화 옵션 |
| **Ad-hoc 분석** | Genie | 자연어로 빠르게 탐색, SQL 작성 불필요 |
| **실시간 모니터링** | AI/BI Dashboard | 레이크하우스 직접 연결, 지연 없음 |
| **데이터 알림** | Alerts | Slack/이메일 통합, 자동 모니터링 |
| **데이터 민주화** | Genie | 비기술 사용자가 직접 데이터 탐색 |
| **임원 보고** | AI/BI Dashboard + Genie | 핵심 KPI 대시보드 + 즉석 질의 |
| **고급 임베디드 분석** | Tableau/Power BI Embedded | 고객 대면 앱에 내장하는 경우 |

---

## 기존 BI 도구와의 기술적 차이

| 아키텍처 측면 | Tableau/Power BI | Databricks AI/BI |
|-------------|-----------------|-----------------|
| **데이터 모델링** | 도구 내부에 데이터 모델(Extract, Import) 생성 | 레이크하우스에서 직접 쿼리 (복사 없음) |
| **시맨틱 레이어** | 도구별 독자적 | Unity Catalog + Metric View (플랫폼 통합) |
| **캐시/추출** | Tableau Extract (.hyper), Power BI Import | SQL Warehouse 인텔리전트 캐싱 |
| **거버넌스** | 도구별 별도 권한 체계 | Unity Catalog 통합 (단일 권한) |
| **AI 기능** | 제한적 (Einstein, Copilot) | 네이티브 AI (Genie, AI 차트 추천) |
| **비용 구조** | 사용자당 라이선스 ($15~$75/user/month) | 쿼리 실행 비용만 |

---

## Genie vs Tableau Ask Data / Power BI Q&A

| 기능 | Genie | Tableau Ask Data | Power BI Q&A |
|------|-------|-----------------|--------------|
| **LLM 기반** | ✅ (최신 LLM) | ❌ (규칙 기반 NLP) | ✅ (Copilot) |
| **복잡한 질문** | 상~중 | 중~하 | 상~중 |
| **인증된 답변** | ✅ (사전 검증 SQL) | ❌ | ❌ |
| **Metric View 연동** | ✅ (네이티브) | ❌ | ❌ |
| **거버넌스** | Unity Catalog 통합 | Tableau 자체 | Power BI 자체 |

---

## Partner Connect를 활용한 BI 도구 연동

| BI 도구 | 연결 방식 | 특이사항 |
|---------|----------|---------|
| **Tableau** | Databricks JDBC/ODBC | Connector 기본 제공, UC 메타데이터 자동 인식 |
| **Power BI** | Databricks ODBC | DirectQuery + Import 지원, Azure AAD 통합 |
| **Looker** | 네이티브 커넥터 | LookML ↔ Delta Lake 직접 연결 |

> 💡 **SQL Warehouse 분리**: AI/BI 전용과 외부 BI 전용 Warehouse를 분리하면, Tableau Extract 새로고침 같은 무거운 작업이 Genie 실시간 쿼리에 영향을 주지 않습니다.

---

## 참고 링크

- [Databricks: Partner Connect](https://docs.databricks.com/aws/en/partners/)
- [Databricks: BI Tool Connectivity](https://docs.databricks.com/aws/en/partners/bi/)
