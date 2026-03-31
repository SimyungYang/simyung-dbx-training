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

## AI/BI 기술 아키텍처 심화

### Compound AI System 아키텍처

Databricks AI/BI는 단일 LLM이 아니라, 여러 AI 컴포넌트가 협력하는 **Compound AI System(복합 AI 시스템)** 으로 구축되어 있습니다.

| 컴포넌트 | 역할 | 상세 설명 |
|----------|------|-----------|
| **NLU (Natural Language Understanding)** | 질문 의도 파악 | 사용자의 자연어 질문에서 의도(intent), 엔터티(entity), 필터 조건을 추출합니다 |
| **Schema Understanding** | 테이블 구조 이해 | Unity Catalog의 메타데이터(테이블명, 컬럼명, COMMENT, 데이터 타입)를 활용하여 적절한 테이블/컬럼을 매칭합니다 |
| **SQL Generator** | SQL 생성 | 파악된 의도를 실행 가능한 SQL로 변환합니다. Databricks SQL 방언에 최적화되어 있습니다 |
| **SQL Validator** | SQL 검증 | 생성된 SQL의 문법 오류, 권한 문제, 성능 이슈를 사전 검증합니다 |
| **Result Formatter** | 결과 포맷팅 | 쿼리 결과를 자연어 답변, 표, 차트로 포맷팅합니다 |
| **Feedback Loop** | 학습 개선 | 사용자 피드백(좋아요/싫어요), 인증된 답변을 반영하여 응답 품질을 개선합니다 |

### 자연어 → SQL 변환 파이프라인 상세

Genie가 사용자의 질문을 SQL로 변환하는 과정을 단계별로 살펴보겠습니다.

```
사용자: "지난 달 서울 매출 상위 5개 제품을 알려줘"

1단계: NLU (의도 분석)
  ├─ 시간 필터: "지난 달" → DATEADD(month, -1, CURRENT_DATE()) ~ CURRENT_DATE()
  ├─ 공간 필터: "서울" → region = '서울'
  ├─ 측정값: "매출" → SUM(order_amount) 또는 Metric View의 total_revenue
  ├─ 차원: "제품" → product_name 또는 product_category
  └─ 수량 제한: "상위 5개" → ORDER BY ... DESC LIMIT 5

2단계: Schema Matching
  ├─ 테이블: gold_orders (COMMENT: "주문 집계 테이블")
  ├─ 컬럼 매핑: 매출 → order_amount, 제품 → product_name
  └─ Metric View 존재 시: revenue_metrics의 정의를 우선 사용

3단계: SQL Generation
  SELECT product_name, SUM(order_amount) AS total_revenue
  FROM catalog.schema.gold_orders
  WHERE region = '서울'
    AND order_date >= DATE_TRUNC('month', DATEADD(month, -1, CURRENT_DATE()))
    AND order_date < DATE_TRUNC('month', CURRENT_DATE())
  GROUP BY product_name
  ORDER BY total_revenue DESC
  LIMIT 5

4단계: Validation
  ├─ 문법 검증: ✅ 유효한 SQL
  ├─ 권한 검증: ✅ 사용자가 gold_orders에 SELECT 권한 있음
  └─ 성능 검증: ✅ 예상 스캔 행 수 적절

5단계: 실행 & 포맷팅
  └─ SQL Warehouse에서 실행 → 결과를 표 + 자연어로 반환
```

### Photon 엔진의 역할

AI/BI의 쿼리는 **Serverless SQL Warehouse**에서 실행되며, 내부적으로 **Photon 엔진**이 핵심 역할을 합니다.

> 💡 **Photon**은 Databricks가 C++로 작성한 **네이티브 벡터화 쿼리 엔진**입니다. JVM 기반 Spark SQL 엔진을 대체하여 BI 쿼리에서 2~8배의 성능 향상을 제공합니다.

| Photon 최적화 | 설명 | AI/BI 연관성 |
|-------------|------|-------------|
| **벡터화 실행** | SIMD 명령어로 한 번에 여러 행을 처리합니다 | 대시보드의 집계 쿼리가 빠르게 실행됩니다 |
| **코드 생성 (Codegen)** | 쿼리를 네이티브 코드로 컴파일하여 실행합니다 | 반복 실행되는 대시보드 쿼리에 효과적입니다 |
| **메모리 관리** | Off-heap 메모리로 GC 오버헤드를 제거합니다 | 동시 다수 쿼리(대시보드 로딩)에 안정적입니다 |
| **인텔리전트 캐싱** | Disk I/O 결과를 메모리에 캐시합니다 | 대시보드 새로고침 시 빠른 응답을 제공합니다 |
| **Adaptive Query Execution** | 실행 중 최적 실행 계획을 동적으로 조정합니다 | Genie의 다양한 쿼리 패턴에 유연하게 대응합니다 |

> 🆕 **Predictive I/O**: Photon의 최신 기능으로, 쿼리 패턴을 학습하여 필요한 데이터를 사전에 메모리에 로드합니다. 반복적인 대시보드 쿼리에서 특히 효과적입니다.

---

## 기존 BI 도구(Tableau/Power BI)와의 기술적 차이

### 아키텍처 비교

| 아키텍처 측면 | Tableau/Power BI | Databricks AI/BI |
|-------------|-----------------|-----------------|
| **데이터 모델링** | 도구 내부에 데이터 모델(Extract, Import)을 생성 | 레이크하우스에서 직접 쿼리 (데이터 복사 없음) |
| **시맨틱 레이어** | 도구별 독자적 (Tableau Data Model, Power BI Semantic Model) | Unity Catalog + Metric View (플랫폼 통합) |
| **캐시/추출** | Tableau Extract (.hyper), Power BI Import 모드 | SQL Warehouse의 인텔리전트 캐싱 |
| **거버넌스** | 도구별 별도 권한 체계 | Unity Catalog 통합 (단일 권한 체계) |
| **AI 기능** | 제한적 (Einstein, Copilot) | 네이티브 AI (Genie, AI 기반 차트 추천) |
| **확장성** | 독자적 확장 (Server/Online) | SQL Warehouse Auto Scaling |
| **비용 구조** | 사용자당 라이선스 ($15~$75/user/month) | Databricks 플랫폼에 포함 (쿼리 실행 비용만) |

### Genie vs Tableau Ask Data / Power BI Q&A

| 기능 | Genie | Tableau Ask Data | Power BI Q&A |
|------|-------|-----------------|--------------|
| **LLM 기반** | ✅ (최신 LLM) | ❌ (규칙 기반 NLP) | ✅ (Copilot) |
| **복잡한 질문 처리** | 상, 중 | 중, 하 | 상, 중 |
| **데이터 소스 제약** | SQL Warehouse 연결 테이블 | Tableau 데이터 소스 | Power BI Semantic Model |
| **인증된 답변** | ✅ (사전 검증 SQL 등록) | ❌ | ❌ |
| **Metric View 연동** | ✅ (네이티브) | ❌ | ❌ |
| **거버넌스** | Unity Catalog 통합 | Tableau 자체 | Power BI 자체 |

---

## 하이브리드 BI 전략 (Databricks + 기존 BI)

대부분의 엔터프라이즈에서는 기존 BI 도구 투자가 있으므로, **점진적 하이브리드 전략**이 현실적입니다.

### 역할 분담 패턴

| 용도 | 권장 도구 | 이유 |
|------|----------|------|
| **복잡한 정적 보고서** | Tableau/Power BI | 기존 인프라 활용, 풍부한 시각화 옵션 |
| **Ad-hoc 분석** | Genie | 자연어로 빠르게 탐색, SQL 작성 불필요 |
| **실시간 모니터링** | AI/BI Dashboard | 레이크하우스 직접 연결, 지연 없음 |
| **데이터 알림** | Alerts | Slack/이메일 통합, 자동 모니터링 |
| **데이터 민주화** | Genie | 비기술 사용자가 직접 데이터 탐색 |
| **임원 보고** | AI/BI Dashboard + Genie | 핵심 KPI 대시보드 + 즉석 질의 |
| **고급 임베디드 분석** | Tableau Embedded / Power BI Embedded | 고객 대면 앱에 내장하는 경우 |

### 연동 아키텍처

```
                    ┌──────────────────────────┐
                    │      Unity Catalog        │
                    │   (Single Source of Truth) │
                    └──────────┬───────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐ ┌──────▼───────┐ ┌──────▼───────┐
    │ SQL Warehouse  │ │ SQL Warehouse│ │ SQL Warehouse│
    │ (AI/BI 전용)    │ │ (BI 도구 전용) │ │ (ML/ETL 전용) │
    └─────┬──────────┘ └──────┬───────┘ └──────────────┘
          │                   │
    ┌─────▼──────┐     ┌─────▼──────┐
    │ AI/BI      │     │ Tableau    │
    │ Dashboard  │     │ Power BI   │
    │ Genie      │     │ Looker     │
    │ Alerts     │     │ (JDBC/ODBC)│
    └────────────┘     └────────────┘
```

> 💡 **SQL Warehouse 분리**: AI/BI 도구용과 외부 BI 도구용 SQL Warehouse를 분리하면, 워크로드 간 간섭을 방지할 수 있습니다. 특히 Tableau Extract 새로고침 같은 무거운 작업이 Genie의 실시간 쿼리에 영향을 주지 않도록 합니다.

### Partner Connect를 활용한 BI 도구 연동

Databricks는 **Partner Connect**를 통해 주요 BI 도구와 원클릭 연동을 지원합니다.

| BI 도구 | 연결 방식 | 특이사항 |
|---------|----------|---------|
| **Tableau** | Databricks JDBC/ODBC 드라이버 | Tableau Connector 기본 제공, Unity Catalog 메타데이터 자동 인식 |
| **Power BI** | Databricks ODBC 드라이버 | DirectQuery + Import 모드 지원, Azure에서는 AAD 통합 인증 |
| **Looker** | Databricks 네이티브 커넥터 | LookML ↔ Delta Lake 직접 연결 |
| **Qlik** | Databricks ODBC 드라이버 | Associative Engine과 Delta Lake 연동 |
| **ThoughtSpot** | Databricks Connector | 자연어 검색 + Databricks 쿼리 엔진 |

---

## 비용 최적화

| 전략 | 설명 | 예상 효과 |
|------|------|----------|
| **Serverless SQL Warehouse 사용** | Auto Scale + Auto Stop으로 비용 최적화 | 유휴 시간 비용 제거 |
| **결과 캐싱 활용** | 동일 쿼리 반복 시 캐시 결과 반환 (TTL 설정) | 쿼리 비용 30~50% 절감 |
| **Gold 테이블 최적화** | 대시보드에 맞게 사전 집계된 테이블 사용 | 스캔 데이터 80%+ 감소 |
| **Metric View 활용** | 일관된 집계로 불필요한 전체 스캔 방지 | 쿼리 효율성 향상 |
| **Query Profile 분석** | 느린 쿼리를 식별하고 최적화 | 사용자 경험 + 비용 개선 |
| **Warehouse 사이즈 적정화** | 워크로드에 맞는 사이즈 선택 (과도한 사이즈 방지) | 20~40% 비용 절감 가능 |

---

## Edge Case와 주의사항

| 주의사항 | 설명 |
|---------|------|
| **Genie 정확도** | 테이블/컬럼의 COMMENT가 부실하면 Genie의 SQL 생성 정확도가 크게 떨어집니다. 메타데이터 품질이 핵심입니다 |
| **인증된 답변 유지보수** | 테이블 스키마가 변경되면 인증된 답변의 SQL도 함께 업데이트해야 합니다 |
| **권한 기반 필터링** | Unity Catalog 행 수준 보안(Row Filter)이 적용된 테이블은 Genie에서도 동일하게 적용됩니다. 사용자별로 다른 결과가 나올 수 있습니다 |
| **대시보드 공유 시 권한** | 대시보드를 공유받은 사용자가 해당 테이블에 SELECT 권한이 없으면 데이터가 표시되지 않습니다 |
| **Genie Space 테이블 수 제한** | 하나의 Genie Space에 너무 많은 테이블(20개+)을 추가하면 정확도가 떨어질 수 있습니다. 주제별로 분리하는 것이 좋습니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **AI/BI Dashboard** | SQL 기반 데이터 시각화. 차트, 필터, 매개변수, 멀티 페이지를 지원합니다 |
| **Genie** | 자연어로 데이터에 질문. 비기술 사용자도 데이터를 탐색할 수 있습니다 |
| **Alerts** | 조건 기반 자동 알림. 이상 감지와 모니터링에 사용합니다 |
| **Lakeview** | AI/BI Dashboard의 기술적 이름입니다 |
| **Compound AI System** | 여러 AI 컴포넌트(NLU, SQL Generator 등)가 협력하는 아키텍처입니다 |
| **Photon** | C++ 네이티브 벡터화 쿼리 엔진으로 BI 쿼리 성능을 극대화합니다 |
| **하이브리드 BI** | Databricks AI/BI + 기존 BI 도구를 역할별로 병행하는 전략입니다 |
| **Unity Catalog 통합** | 대시보드에도 동일한 데이터 권한이 적용됩니다 |

---

## 참고 링크

- [Databricks: AI/BI](https://docs.databricks.com/aws/en/aibi/)
- [Databricks: AI/BI Dashboards](https://docs.databricks.com/aws/en/aibi/dashboards/)
- [Databricks: Genie](https://docs.databricks.com/aws/en/aibi/genie/)
- [Databricks: Alerts](https://docs.databricks.com/aws/en/sql/user/alerts/)
- [Azure Databricks: Business Intelligence](https://learn.microsoft.com/en-us/azure/databricks/aibi/)
