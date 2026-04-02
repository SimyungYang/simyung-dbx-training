# 하이브리드 BI 전략

## 왜 하이브리드 전략인가?

대부분의 엔터프라이즈에서는 Tableau, Power BI 등 기존 BI 도구에 이미 상당한 투자가 있습니다. Databricks AI/BI는 이를 **대체** 하는 것이 아니라 **보완** 합니다.

---

## 역할 분담 패턴

| 용도 | 권장 도구 | 이유 |
|------|----------|------|
| **복잡한 정적 보고서**| Tableau/Power BI | 기존 인프라 활용, 풍부한 시각화 옵션 |
| **Ad-hoc 분석**| Genie | 자연어로 빠르게 탐색, SQL 작성 불필요 |
| **실시간 모니터링**| AI/BI Dashboard | 레이크하우스 직접 연결, 지연 없음 |
| **데이터 알림**| Alerts | Slack/이메일 통합, 자동 모니터링 |
| **데이터 민주화**| Genie | 비기술 사용자가 직접 데이터 탐색 |
| **임원 보고**| AI/BI Dashboard + Genie | 핵심 KPI 대시보드 + 즉석 질의 |
| **고급 임베디드 분석**| Tableau/Power BI Embedded | 고객 대면 앱에 내장하는 경우 |

---

## 기존 BI 도구와의 기술적 차이

| 아키텍처 측면 | Tableau/Power BI | Databricks AI/BI |
|-------------|-----------------|-----------------|
| **데이터 모델링**| 도구 내부에 데이터 모델(Extract, Import) 생성 | 레이크하우스에서 직접 쿼리 (복사 없음) |
| **시맨틱 레이어**| 도구별 독자적 | Unity Catalog + Metric View (플랫폼 통합) |
| **캐시/추출**| Tableau Extract (.hyper), Power BI Import | SQL Warehouse 인텔리전트 캐싱 |
| **거버넌스**| 도구별 별도 권한 체계 | Unity Catalog 통합 (단일 권한) |
| **AI 기능**| 제한적 (Einstein, Copilot) | 네이티브 AI (Genie, AI 차트 추천) |
| **비용 구조**| 사용자당 라이선스 ($15~$75/user/month) | 쿼리 실행 비용만 |

---

## Genie vs Tableau Ask Data / Power BI Q&A

| 기능 | Genie | Tableau Ask Data | Power BI Q&A |
|------|-------|-----------------|--------------|
| **LLM 기반**| ✅ (최신 LLM) | ❌ (규칙 기반 NLP) | ✅ (Copilot) |
| **복잡한 질문**| 상~중 | 중~하 | 상~중 |
| **인증된 답변**| ✅ (사전 검증 SQL) | ❌ | ❌ |
| **Metric View 연동**| ✅ (네이티브) | ❌ | ❌ |
| **거버넌스**| Unity Catalog 통합 | Tableau 자체 | Power BI 자체 |

---

## Partner Connect를 활용한 BI 도구 연동

| BI 도구 | 연결 방식 | 특이사항 |
|---------|----------|---------|
| **Tableau**| Databricks JDBC/ODBC | Connector 기본 제공, UC 메타데이터 자동 인식 |
| **Power BI**| Databricks ODBC | DirectQuery + Import 지원, Azure AAD 통합 |
| **Looker**| 네이티브 커넥터 | LookML ↔ Delta Lake 직접 연결 |

> 💡 **SQL Warehouse 분리**: AI/BI 전용과 외부 BI 전용 Warehouse를 분리하면, Tableau Extract 새로고침 같은 무거운 작업이 Genie 실시간 쿼리에 영향을 주지 않습니다.

---

## 실전 아키텍처 패턴

### Embedded BI (임베디드 BI)

고객 대면 애플리케이션에 분석 기능을 직접 내장하는 패턴입니다. SaaS 제품에 대시보드를 포함하거나, 파트너 포털에 데이터 분석 화면을 제공할 때 사용합니다.

| 구성 요소 | 역할 | 구현 방법 |
|----------|------|----------|
| **Databricks SQL** | 쿼리 엔진 | SQL Warehouse API로 쿼리 실행 |
| **Tableau Embedded** | 시각화 렌더링 | Tableau Server + REST API로 iFrame 내장 |
| **Power BI Embedded** | 시각화 렌더링 | Power BI Embedded Capacity (A/EM SKU) |
| **Databricks Apps** | 커스텀 UI | Streamlit/Dash 앱을 Databricks에서 호스팅 |

{% hint style="info" %}
**Databricks Apps 활용**: Tableau/Power BI Embedded 라이선스 비용이 부담된다면, Databricks Apps로 Streamlit 기반 대시보드를 구축하면 추가 BI 라이선스 없이 임베디드 분석을 구현할 수 있습니다.
{% endhint %}

### Headless BI (헤드리스 BI)

시각화 도구에 종속되지 않고, **시맨틱 레이어** 에서 비즈니스 로직을 중앙 관리하는 패턴입니다. 어떤 BI 도구에서 접근하든 동일한 지표 정의를 보장합니다.

| 계층 | 구성 요소 | 설명 |
|------|----------|------|
| **시맨틱 레이어** | Unity Catalog + Metric Views | 비즈니스 지표의 단일 진실 소스(SSOT) |
| **쿼리 엔진** | SQL Warehouse (Serverless) | 모든 BI 도구의 공통 실행 엔진 |
| **소비 채널** | Genie, Tableau, Power BI, API | 용도에 맞는 도구로 동일 지표 조회 |

```sql
-- Metric View 정의 (시맨틱 레이어의 핵심)
CREATE METRIC VIEW sales.gold.revenue_metrics AS
SELECT
  date_trunc('month', order_date) AS period,
  region,
  SUM(amount) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers,
  SUM(amount) / COUNT(DISTINCT customer_id) AS revenue_per_customer
FROM sales.gold.orders
GROUP BY 1, 2;

-- Tableau, Power BI, Genie 모두 이 Metric View를 소스로 사용
-- → 지표 정의가 BI 도구마다 달라지는 문제 해소
```

### Semantic Layer 전략 비교

| 시맨틱 레이어 | 장점 | 단점 | Databricks 통합 |
|-------------|------|------|----------------|
| **Unity Catalog Metric Views** | 플랫폼 네이티브, Genie 자동 연동 | BI 도구별 추가 설정 필요 | ★★★★★ |
| **dbt Semantic Layer** | dbt 생태계 활용, MetricFlow | Databricks 외부 서비스 | ★★★★☆ |
| **Tableau Data Model** | Tableau 내부 최적화 | Tableau 전용, 락인 | ★★★☆☆ |
| **Power BI Composite Model** | DirectQuery + Import 혼합 | Power BI 전용 | ★★★☆☆ |
| **LookML (Looker)** | 코드 기반, Git 관리 | Looker 전용 | ★★★☆☆ |

---

## BI 도구별 상세 연결 가이드

### Tableau 연결 상세

```text
[연결 단계]
1. Tableau Desktop → Connect → Databricks 선택
2. Server Hostname: <workspace-url>.cloud.databricks.com
3. HTTP Path: /sql/1.0/warehouses/<warehouse-id>
4. Authentication: Personal Access Token 또는 OAuth (M2M)
5. Catalog / Schema 선택 → 테이블 드래그
```

| 설정 항목 | 권장값 | 이유 |
|----------|-------|------|
| **Initial SQL** | `SET use_cached_result = true` | 반복 쿼리 캐시 활용 |
| **Connection Pool** | 최대 8~16 | Tableau Server 동시 연결 제한 |
| **Extract 갱신 주기** | Gold 테이블: 1일 1회, Silver: 불필요 | Gold는 집계 완료, Silver는 Live 연결 |
| **Live vs Extract** | 대시보드: Extract, 탐색: Live | Extract는 빠르지만 데이터 지연 |

{% hint style="warning" %}
**Tableau Extract 주의사항**: Extract 갱신 시 전체 테이블을 다시 읽으므로, 대용량 테이블은 Incremental Extract를 설정하거나 Materialized View를 소스로 사용하세요. Extract 갱신 전용 SQL Warehouse를 분리하는 것이 좋습니다.
{% endhint %}

### Power BI 연결 상세

| 연결 모드 | 적합한 시나리오 | 데이터 최신성 | 비용 영향 |
|----------|-------------|-------------|----------|
| **DirectQuery** | 실시간 데이터 필요, 대용량 | 항상 최신 | SQL Warehouse 상시 가동 필요 |
| **Import** | 정적 보고서, 오프라인 조회 | 갱신 주기에 따름 | 갱신 시에만 Warehouse 사용 |
| **Composite Model** | 일부 실시간 + 일부 정적 | 혼합 | 균형 잡힌 비용 |

```text
[Power BI Desktop 연결]
1. 데이터 가져오기 → Azure Databricks 선택
2. Server Hostname 입력
3. HTTP Path 입력 (/sql/1.0/warehouses/<id>)
4. Data Connectivity Mode: DirectQuery 또는 Import 선택
5. Authentication: Azure AD (SSO) 또는 PAT

[Power BI Service 자동 갱신]
- Gateway 불필요 (Databricks에 직접 연결)
- 스케줄 갱신: 최대 8회/일 (Pro), 48회/일 (Premium)
```

### Looker 연결 상세

```yaml
# Looker connection 설정 (LookML 프로젝트)
connection: "databricks_lakehouse"

# looker_database.yml
- connection: databricks_lakehouse
  dialect: databricks
  host: <workspace-url>.cloud.databricks.com
  port: 443
  database: prod_catalog
  schema: gold
  jdbc_additional_params: "transportMode=http;ssl=1;httpPath=/sql/1.0/warehouses/<id>"
  authentication: oauth  # Looker 22.0+에서 OAuth 지원
```

---

## 마이그레이션 시나리오

### Cognos → Databricks 마이그레이션

| 단계 | 작업 | 산출물 |
|------|------|-------|
| **1. 현황 분석** | Cognos 리포트/대시보드 목록화, 사용 빈도 분석 | 마이그레이션 우선순위 매트릭스 |
| **2. 데이터 소스 전환** | Cognos 데이터 소스(DB2, Oracle 등) → Databricks Lakehouse | Unity Catalog 테이블 |
| **3. 시맨틱 모델 전환** | Framework Manager 모델 → Metric Views + Gold 테이블 | SQL 기반 시맨틱 레이어 |
| **4. 리포트 전환** | Cognos Report Studio → AI/BI Dashboard | 대시보드 + Genie Space |
| **5. 자연어 분석 추가** | 기존에 없던 기능 | Genie로 ad-hoc 분석 강화 |

{% hint style="info" %}
**마이그레이션 팁**: Cognos의 고정 리포트 중 사용 빈도 하위 50%는 마이그레이션하지 않고 폐기하는 것이 효율적입니다. 대부분 Genie의 자연어 질의로 대체 가능합니다.
{% endhint %}

### SAP BO → Databricks 마이그레이션

| SAP BO 구성 요소 | Databricks 대체 | 비고 |
|----------------|---------------|------|
| **Universe (시맨틱 레이어)** | Unity Catalog + Metric Views | SQL 기반으로 전환 |
| **Web Intelligence** | AI/BI Dashboard | 정적 리포트 대체 |
| **Crystal Reports** | AI/BI Dashboard + PDF Export | 인쇄용 리포트 |
| **Lumira/Analytics Cloud** | Genie + AI/BI Dashboard | 탐색적 분석 |
| **BEx Analyzer (BW)** | Databricks SQL + Gold 테이블 | SAP BW 데이터를 Lakehouse로 |

```sql
-- SAP 데이터 수집 패턴 (Lakehouse Federation 또는 별도 수집)
-- 방법 1: Lakehouse Federation (실시간, 소량)
CREATE CONNECTION sap_hana_conn
TYPE MYSQL  -- SAP HANA는 JDBC로 연결
OPTIONS (
  host '<sap-hana-host>',
  port '30015',
  user '<user>',
  password '<password>'
);

-- 방법 2: 배치 수집 (대용량, 권장)
-- Fivetran / Informatica로 SAP 테이블을 Bronze에 적재
-- SDP 파이프라인으로 Silver → Gold 변환
```

---

## 비용 모델 비교

| 비용 항목 | Tableau Server | Power BI Premium | Databricks AI/BI |
|----------|---------------|-----------------|-----------------|
| **라이선스 (Viewer)** | $15/user/월 | $10/user/월 (E5에 포함) | 무료 (플랫폼 포함) |
| **라이선스 (Creator)** | $70/user/월 | $20/user/월 (Pro) | 무료 |
| **인프라** | Tableau Server 운영 비용 | Premium Capacity ($5,000+/월) | SQL Warehouse 비용만 |
| **데이터 이동** | Extract 갱신 비용 | Import 갱신 비용 | 없음 (직접 조회) |
| **100명 사용자 연간 예상** | $30,000~$60,000 | $15,000~$40,000 | $5,000~$15,000 (쿼리 비용) |

{% hint style="info" %}
**총 소유 비용(TCO) 고려사항**: 라이선스 비용만 비교하면 안 됩니다. 데이터 이동/복제 비용, Extract 갱신 인프라, 시맨틱 레이어 이중 관리 비용까지 포함해야 합니다. Databricks AI/BI는 데이터 복제가 없으므로 숨겨진 비용이 적습니다.
{% endhint %}

---

## 성능 벤치마크

### 쿼리 응답 시간 비교 (1TB 팩트 테이블 기준)

| 쿼리 유형 | Tableau Extract | Power BI Import | Databricks AI/BI (Serverless) |
|----------|----------------|-----------------|------------------------------|
| **단순 필터 + 집계** | 0.5초 | 0.3초 | 1~2초 |
| **다중 조인 (3+ 테이블)** | 2~5초 | 1~3초 | 2~4초 |
| **기간 비교 (YoY)** | 1~3초 | 1~2초 | 1~3초 |
| **자연어 질의** | N/A | 3~5초 (Copilot) | 2~5초 (Genie) |
| **데이터 최신성** | Extract 갱신 시점 | Import 갱신 시점 | **항상 실시간** |
| **동시 사용자 50명** | 서버 부하 증가 | Capacity 제한 | 자동 스케일링 |

{% hint style="warning" %}
**성능 비교 시 주의**: Tableau Extract/Power BI Import는 데이터를 로컬에 복제하므로 단순 쿼리에서 빠를 수 있습니다. 하지만 **데이터 최신성** 과 **대규모 동시 접속** 에서는 Databricks AI/BI가 유리합니다.
{% endhint %}

### 권장 하이브리드 아키텍처

| 사용 사례 | 도구 | 이유 |
|----------|------|------|
| **C-레벨 KPI 대시보드** | AI/BI Dashboard | 실시간 데이터, 간결한 시각화 |
| **자연어 ad-hoc 분석** | Genie | 비기술 사용자 자율 분석 |
| **재무/규제 정기 보고서** | Tableau/Power BI | 기존 양식 유지, 인쇄 최적화 |
| **고객 대면 포털** | Databricks Apps | 라이선스 비용 없음 |
| **데이터 알림** | Alerts | Slack/이메일 자동 알림 |
| **데이터 품질 모니터링** | AI/BI Dashboard + Alerts | 파이프라인 상태 실시간 감시 |

---

## 참고 링크

- [Databricks: Partner Connect](https://docs.databricks.com/aws/en/partners/)
- [Databricks: BI Tool Connectivity](https://docs.databricks.com/aws/en/partners/bi/)
- [Databricks: Metric Views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-metric-view.html)
- [Databricks: AI/BI Dashboards](https://docs.databricks.com/aws/en/aibi/dashboards/)
- [Tableau: Databricks Connector](https://help.tableau.com/current/pro/desktop/en-us/examples_databricks.htm)
- [Power BI: Azure Databricks Connector](https://learn.microsoft.com/en-us/power-bi/connect-data/service-azure-databricks)
