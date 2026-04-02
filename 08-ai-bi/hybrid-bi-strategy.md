# 하이브리드 BI 전략 (Hybrid BI Strategy)

## 1. 왜 하이브리드 BI 전략이 필요한가?

대부분의 엔터프라이즈 조직은 **Tableau**, **Power BI**, **Looker** 등 기존 BI 도구에 이미 수년간 상당한 투자를 해왔습니다. 라이선스, 교육, 커스터마이징, 보고서 자산이 누적되어 있어 단순 교체는 현실적이지 않습니다.

동시에, 단일 BI 도구만으로 모든 사용 요구를 충족하기는 어렵습니다.

| 요구 유형 | 기존 BI 도구의 한계 | Databricks AI/BI가 해결하는 방법 |
|----------|-------------------|-------------------------------|
| **비기술 사용자 셀프서비스** | SQL 작성 필요, IT 의존성 높음 | Genie 자연어 질의로 즉시 분석 |
| **실시간 데이터** | Extract 갱신 지연 (수 시간~1일) | Lakehouse 직접 연결, 항상 최신 |
| **AI/ML 인사이트** | 제한적 (규칙 기반) | 네이티브 LLM 기반 분석 |
| **거버넌스 통합** | 도구별 권한 이원화 | Unity Catalog 단일 권한 체계 |
| **비용 효율** | 사용자당 고정 라이선스 | 쿼리 실행 비용만 (사용한 만큼) |

**결론**: Databricks AI/BI는 기존 BI 도구를 **대체** 하는 것이 아니라 **보완** 합니다. 두 도구의 강점을 조합하는 하이브리드 전략이 현실적이고 효과적입니다.

---

## 2. 하이브리드 BI 아키텍처 패턴

### 2-1. 역할 분담 패턴 (Role-Based Pattern)

| 용도 | 권장 도구 | 이유 |
|------|----------|------|
| **복잡한 정적 보고서** | Tableau / Power BI | 기존 인프라 활용, 풍부한 시각화 옵션 |
| **Ad-hoc 탐색 분석** | Genie | 자연어로 빠르게 탐색, SQL 작성 불필요 |
| **실시간 운영 모니터링** | AI/BI Dashboard | Lakehouse 직접 연결, 지연 없음 |
| **데이터 알림** | Alerts | Slack / 이메일 통합, 자동 모니터링 |
| **데이터 민주화** | Genie | 비기술 사용자 직접 탐색 |
| **임원 KPI 보고** | AI/BI Dashboard + Genie | 핵심 지표 대시보드 + 즉석 질의 |
| **고급 임베디드 분석** | Tableau Embedded / Power BI Embedded | 고객 대면 앱 내장 |

### 2-2. Headless BI (헤드리스 BI) 아키텍처

시각화 도구에 종속되지 않고 **시맨틱 레이어 (Semantic Layer)** 에서 비즈니스 로직을 중앙 관리하는 패턴입니다. 어떤 BI 도구에서 접근하든 동일한 지표 정의를 보장합니다.

```
┌─────────────────────────────────────────────────────────┐
│              소비 채널 (Consumption Layer)                │
│  Genie  │  AI/BI Dashboard  │  Tableau  │  Power BI  │  API  │
└─────────────────────┬───────────────────────────────────┘
                      │ SQL / REST
┌─────────────────────▼───────────────────────────────────┐
│       시맨틱 레이어 (Semantic Layer)                       │
│    Unity Catalog + Metric Views (SSOT)                   │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│         쿼리 엔진 (Query Engine)                           │
│    SQL Warehouse (Serverless / Pro / Classic)            │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              Lakehouse (Delta Lake)                       │
│    Bronze → Silver → Gold (Medallion Architecture)       │
└─────────────────────────────────────────────────────────┘
```

### 2-3. Embedded BI (임베디드 BI) 아키텍처

고객 대면 애플리케이션에 분석 기능을 직접 내장하는 패턴입니다.

| 구성 요소 | 역할 | 구현 방법 |
|----------|------|----------|
| **Databricks SQL** | 쿼리 엔진 | SQL Warehouse API로 쿼리 실행 |
| **Tableau Embedded** | 시각화 렌더링 | Tableau Server + REST API로 iFrame 내장 |
| **Power BI Embedded** | 시각화 렌더링 | Power BI Embedded Capacity (A/EM SKU) |
| **Databricks Apps** | 커스텀 UI | Streamlit / Dash 앱을 Databricks에서 호스팅 |

> **Databricks Apps 활용**: Tableau / Power BI Embedded 라이선스 비용이 부담된다면, Databricks Apps로 Streamlit 기반 대시보드를 구축하면 추가 BI 라이선스 없이 임베디드 분석을 구현할 수 있습니다.

---

## 3. BI 도구 연결 방법 (Connectivity)

### 3-1. Partner Connect

**Partner Connect** 는 Databricks Workspace UI에서 클릭 몇 번으로 외부 BI 도구와 연결을 설정하는 가장 간단한 방법입니다.

- Workspace → **Partner Connect** 메뉴 진입
- Tableau, Power BI, Looker 등 파트너 목록에서 선택
- SQL Warehouse HTTP Path 및 인증 정보 자동 생성
- 일부 파트너는 계정 자동 프로비저닝 지원

### 3-2. Tableau 연결 (JDBC / ODBC)

```text
[연결 단계]
1. Tableau Desktop → Connect → Databricks 선택
2. Server Hostname: <workspace-url>.cloud.databricks.com
3. HTTP Path: /sql/1.0/warehouses/<warehouse-id>
4. Authentication: Personal Access Token (PAT) 또는 OAuth (M2M)
5. Catalog / Schema 선택 → 테이블 드래그 앤 드롭
```

| 설정 항목 | 권장값 | 이유 |
|----------|-------|------|
| **Initial SQL** | `SET use_cached_result = true` | 반복 쿼리 캐시 활용 |
| **Connection Pool** | 최대 8~16 | Tableau Server 동시 연결 제한 |
| **Extract 갱신 주기** | Gold 테이블: 1일 1회, Silver: 불필요 | Gold는 집계 완료, Silver는 Live 연결 권장 |
| **Live vs Extract** | 대시보드: Extract, 탐색: Live | Extract는 빠르지만 데이터 지연 발생 |

> **Tableau Extract 주의사항**: Extract 갱신 시 전체 테이블을 재읽기하므로, 대용량 테이블은 Incremental Extract를 설정하거나 Materialized View를 소스로 사용하세요. Extract 갱신 전용 SQL Warehouse를 분리하는 것이 좋습니다.

### 3-3. Power BI 연결 (ODBC / DirectQuery)

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
- On-premises Gateway 불필요 (Databricks에 직접 연결)
- 스케줄 갱신: 최대 8회/일 (Pro), 48회/일 (Premium)
```

### 3-4. Databricks SQL Connector (Python / Node.js)

커스텀 애플리케이션에서 Databricks SQL Warehouse에 직접 연결할 때 사용합니다.

```python
from databricks import sql

with sql.connect(
    server_hostname="<workspace-url>.cloud.databricks.com",
    http_path="/sql/1.0/warehouses/<warehouse-id>",
    access_token="<personal-access-token>"
) as connection:
    with connection.cursor() as cursor:
        cursor.execute("SELECT * FROM sales.gold.revenue_metrics LIMIT 100")
        result = cursor.fetchall()
```

> 참고: [Databricks SQL Connector for Python](https://docs.databricks.com/aws/en/dev-tools/python-sql-connector.html)

---

## 4. 각 도구별 최적 사용 사례

### 4-1. Databricks AI/BI Dashboard — 셀프서비스 탐색

**최적 시나리오**:
- 데이터 엔지니어 / 분석가가 Lakehouse 데이터를 직접 시각화
- 실시간 운영 지표 모니터링 (파이프라인 상태, 데이터 품질)
- SQL을 아는 사람이 빠르게 인사이트를 공유할 때

**강점**:
- Lakehouse에 직접 연결 → 항상 최신 데이터
- Unity Catalog 권한 자동 상속
- AI 차트 추천 (AI-suggested visuals)
- Genie Space와 긴밀하게 통합

### 4-2. Genie — 자연어 분석

**최적 시나리오**:
- 비기술 사용자(영업, 마케팅, 재무)의 데이터 자율 탐색
- "지난 분기 대비 이번 분기 매출이 얼마나 증가했나요?" 형태의 즉흥 질의
- 반복적인 IT 요청 없이 데이터 민주화 실현

**강점**:
- 최신 LLM 기반 자연어 이해
- 인증된 답변(Certified Answers)으로 신뢰성 보장
- Metric View 네이티브 연동

### 4-3. Tableau — 복잡한 시각화

**최적 시나리오**:
- 정교한 인터랙티브 시각화 (복잡한 드릴다운, 커스텀 차트)
- 인쇄 최적화된 정기 보고서 (재무, 규제, 감사)
- 기존 Tableau 자산(워크북, 대시보드) 재활용
- Tableau Server 기반 임베디드 분석

**강점**:
- 업계 최고 수준의 시각화 표현력
- 오랜 사용자 기반, 교육 자료 풍부
- Tableau Prep으로 데이터 준비 자동화

### 4-4. Power BI — Microsoft 생태계

**최적 시나리오**:
- Microsoft 365 / Azure AD 환경에서 SSO 필요
- Excel / Teams 연동 보고서
- DirectQuery로 실시간 데이터 + Import로 정적 데이터 혼합 (Composite Model)
- Power Apps / Power Automate 워크플로와 통합

**강점**:
- Microsoft 생태계 완벽 통합
- Copilot (AI 기반 자연어 분석) 내장
- E5 라이선스 보유 조직에서 비용 효율적

### 4-5. Genie vs Tableau Ask Data vs Power BI Q&A 비교

| 기능 | Genie | Tableau Ask Data | Power BI Q&A |
|------|-------|-----------------|--------------|
| **LLM 기반** | ✅ (최신 LLM) | ❌ (규칙 기반 NLP) | ✅ (Copilot) |
| **복잡한 질문** | 상~중 | 중~하 | 상~중 |
| **인증된 답변** | ✅ (사전 검증 SQL) | ❌ | ❌ |
| **Metric View 연동** | ✅ (네이티브) | ❌ | ❌ |
| **거버넌스** | Unity Catalog 통합 | Tableau 자체 | Power BI 자체 |

---

## 5. Semantic Layer 전략 — 단일 진실 소스 (SSOT)

### 5-1. Metric View 활용

**Metric View** 는 Unity Catalog에 비즈니스 지표 정의를 저장하는 객체입니다. Tableau, Power BI, Genie 어디서든 동일한 지표 정의를 참조하여 "매출"이 부서마다 다르게 계산되는 문제를 방지합니다.

```sql
-- Metric View 정의 (시맨틱 레이어의 핵심)
CREATE METRIC VIEW sales.gold.revenue_metrics AS
SELECT
  date_trunc('month', order_date) AS period,
  region,
  SUM(amount)                          AS total_revenue,
  COUNT(DISTINCT customer_id)           AS unique_customers,
  SUM(amount) / COUNT(DISTINCT customer_id) AS revenue_per_customer
FROM sales.gold.orders
GROUP BY 1, 2;

-- Tableau, Power BI, Genie 모두 이 Metric View를 소스로 사용
-- → 지표 정의가 BI 도구마다 달라지는 문제 해소
```

> 참고: [Databricks Metric Views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-metric-view.html)

### 5-2. Semantic Layer 옵션 비교

| 시맨틱 레이어 | 장점 | 단점 | Databricks 통합 수준 |
|-------------|------|------|---------------------|
| **Unity Catalog Metric Views** | 플랫폼 네이티브, Genie 자동 연동 | BI 도구별 추가 설정 필요 | ★★★★★ |
| **dbt Semantic Layer (MetricFlow)** | dbt 생태계 활용 | Databricks 외부 서비스 | ★★★★☆ |
| **Tableau Data Model** | Tableau 내부 최적화 | Tableau 전용, 락인 | ★★★☆☆ |
| **Power BI Composite Model** | DirectQuery + Import 혼합 | Power BI 전용 | ★★★☆☆ |
| **LookML (Looker)** | 코드 기반, Git 관리 | Looker 전용 | ★★★☆☆ |

> **권장**: 가능하면 **Unity Catalog Metric Views** 를 SSOT로 삼고, 외부 BI 도구는 이를 소스로 참조하도록 구성합니다.

---

## 6. 성능 최적화 (Performance Optimization)

### 6-1. SQL Warehouse 사이징 전략

BI 워크로드 유형에 따라 SQL Warehouse 종류와 크기를 분리하는 것이 핵심입니다.

| Warehouse 용도 | 권장 타입 | 크기 | 자동 중지 |
|--------------|----------|------|---------|
| **Genie 실시간 질의** | Serverless | Small | 10분 |
| **AI/BI Dashboard 조회** | Serverless | Small~Medium | 10분 |
| **Tableau Live 연결** | Pro | Medium | 30분 |
| **Tableau / Power BI Extract 갱신** | Classic | Large | 갱신 후 즉시 |
| **대규모 배치 집계** | Classic | X-Large | 작업 완료 후 |

> **Warehouse 분리의 이점**: Tableau Extract 갱신처럼 무거운 배치 작업이 Genie 실시간 쿼리에 영향을 주지 않습니다.

### 6-2. 캐싱 전략 (Caching)

Databricks SQL은 세 가지 캐싱 레이어를 제공합니다.

| 캐시 종류 | 설명 | 설정 방법 |
|----------|------|---------|
| **Query Result Cache** | 동일 쿼리의 이전 결과 재사용 | `SET use_cached_result = true` (기본값) |
| **Disk Cache (Local SSD)** | Delta 파일을 워커 노드 로컬 SSD에 캐시 | Pro/Classic Warehouse에서 자동 활성화 |
| **Materialized View** | 집계 결과를 Delta 테이블로 사전 계산 | `CREATE MATERIALIZED VIEW ...` |

```sql
-- Materialized View 예시: 일별 매출 집계를 사전 계산
CREATE MATERIALIZED VIEW sales.gold.daily_revenue_mv AS
SELECT
  DATE(order_date)    AS order_day,
  region,
  SUM(amount)         AS daily_revenue
FROM sales.silver.orders
GROUP BY 1, 2;
-- BI 도구는 원본 대신 이 MV를 조회 → 쿼리 속도 대폭 향상
```

### 6-3. Photon 활용

**Photon** 은 Databricks의 네이티브 벡터화 쿼리 엔진으로, SQL 워크로드를 CPU/메모리 수준에서 최적화합니다.

- Pro / Classic SQL Warehouse에서 기본 활성화
- 집계, 조인, 필터 쿼리에서 최대 8~10배 성능 향상
- 별도 설정 불필요 — SQL Warehouse 생성 시 자동 적용

### 6-4. 아키텍처 측면 비교

| 아키텍처 측면 | Tableau / Power BI | Databricks AI/BI |
|-------------|-----------------|-----------------|
| **데이터 모델링** | 도구 내부에 Extract / Import 생성 | Lakehouse에서 직접 쿼리 (복사 없음) |
| **시맨틱 레이어** | 도구별 독자적 | Unity Catalog + Metric View (플랫폼 통합) |
| **캐시 / 추출** | Tableau Extract (.hyper), Power BI Import | SQL Warehouse 인텔리전트 캐싱 |
| **거버넌스** | 도구별 별도 권한 체계 | Unity Catalog 통합 (단일 권한) |
| **AI 기능** | 제한적 (Einstein, Copilot) | 네이티브 AI (Genie, AI 차트 추천) |
| **비용 구조** | 사용자당 라이선스 ($15~$75/user/월) | 쿼리 실행 비용만 |

---

## 7. 장단점과 트레이드오프 (Tradeoffs)

### 7-1. 하이브리드 전략의 장단점

| 항목 | 장점 | 단점 |
|------|------|------|
| **도구 통합** | 각 도구의 강점 최대 활용 | 도구 수 증가 → 운영 복잡성 |
| **라이선스 비용** | 기존 투자 보호, 중복 도입 최소화 | 두 플랫폼 모두 유지 비용 발생 |
| **학습 곡선** | 사용자가 익숙한 도구 유지 가능 | 팀별로 다른 도구 → 협업 마찰 |
| **데이터 일관성** | SSOT 구성 시 지표 통일 | 도구별 시맨틱 레이어 이중 관리 위험 |
| **거버넌스** | Unity Catalog 중앙 관리 | 외부 BI 도구의 권한 별도 관리 필요 |

### 7-2. 도구 통합 vs 분산

| 전략 | 적합한 조직 | 리스크 |
|------|-----------|--------|
| **단일 도구 (All-in Databricks)** | 신규 조직, 레거시 없음 | Databricks 락인 |
| **단일 도구 (All-in Tableau/Power BI)** | Databricks 미도입 조직 | AI 기능 제한, 데이터 이동 비용 |
| **하이브리드 (권장)** | 기존 BI 투자 있는 엔터프라이즈 | 복잡성 관리 필요 |

### 7-3. 비용 모델 비교

| 비용 항목 | Tableau Server | Power BI Premium | Databricks AI/BI |
|----------|---------------|-----------------|-----------------|
| **라이선스 (Viewer)** | $15/user/월 | $10/user/월 (E5 포함) | 무료 (플랫폼 포함) |
| **라이선스 (Creator)** | $70/user/월 | $20/user/월 (Pro) | 무료 |
| **인프라** | Tableau Server 운영비 | Premium Capacity ($5,000+/월) | SQL Warehouse 비용만 |
| **데이터 이동** | Extract 갱신 비용 | Import 갱신 비용 | 없음 (직접 조회) |
| **100명 사용자 연간 예상** | $30,000~$60,000 | $15,000~$40,000 | $5,000~$15,000 (쿼리 비용) |

> **총 소유 비용(TCO) 고려사항**: 라이선스 비용만 비교하면 안 됩니다. 데이터 이동 / 복제 비용, Extract 갱신 인프라, 시맨틱 레이어 이중 관리 비용까지 포함해야 합니다. Databricks AI/BI는 데이터 복제가 없으므로 숨겨진 비용이 적습니다.

---

## 8. 마이그레이션 가이드 (Migration Guide)

### 8-1. 기본 원칙 — 단계적 전환 (Phased Migration)

한 번에 전환하는 Big Bang 방식은 위험합니다. 아래 4단계 점진적 전환을 권장합니다.

| 단계 | 이름 | 핵심 작업 | 기간 |
|------|------|---------|------|
| **Phase 1** | 연결 (Connect) | Tableau / Power BI를 Databricks에 연결, 데이터 소스 전환 | 1~2개월 |
| **Phase 2** | 탐색 (Explore) | Genie Space 구성, 파워 사용자 대상 AI/BI 파일럿 | 1~2개월 |
| **Phase 3** | 이전 (Migrate) | 고사용 보고서부터 AI/BI Dashboard로 재구축 | 2~4개월 |
| **Phase 4** | 최적화 (Optimize) | Metric View SSOT 구성, 저사용 보고서 폐기 | 지속 |

### 8-2. Tableau → Databricks AI/BI 전환

| Tableau 구성 요소 | Databricks 대체 | 주의사항 |
|----------------|---------------|---------|
| **Tableau Prep** | Lakeflow Jobs / dbt | 파이프라인 재구축 필요 |
| **Tableau Data Source (.tds)** | Unity Catalog 테이블 + Metric View | 시맨틱 레이어 재정의 |
| **Tableau Workbook** | AI/BI Dashboard | 차트 유형 재매핑 필요 |
| **Tableau Server (배포)** | Databricks Workspace 공유 | 권한 체계 전환 |
| **Tableau Extract (.hyper)** | Materialized View | 갱신 주기 재설계 |

> **팁**: 사용 빈도 분석(Usage Analytics)으로 하위 50% 보고서는 마이그레이션하지 않고 폐기하는 것이 효율적입니다. Genie의 자연어 질의가 임시 보고서 대부분을 대체할 수 있습니다.

### 8-3. Power BI → Databricks AI/BI 전환

| Power BI 구성 요소 | Databricks 대체 | 주의사항 |
|-----------------|---------------|---------|
| **Power Query (M 언어)** | Lakeflow Jobs / dbt | ETL 로직 재구현 |
| **Power BI Dataset** | Unity Catalog 테이블 + Metric View | DirectQuery 연결 유지 가능 |
| **Power BI Report** | AI/BI Dashboard | 단계적 병행 운영 권장 |
| **Power BI Service** | Databricks Workspace | SSO / 권한 전환 |
| **DAX 지표** | SQL + Metric View | DAX → SQL 변환 작업 |

### 8-4. Cognos / SAP BO → Databricks 마이그레이션

| 구성 요소 | Cognos 대체 | SAP BO 대체 |
|----------|-----------|-----------|
| **시맨틱 레이어** | Framework Manager → Metric Views | Universe → Metric Views |
| **보고서** | Report Studio → AI/BI Dashboard | Web Intelligence → AI/BI Dashboard |
| **탐색 분석** | Query Studio → Genie | Lumira → Genie |
| **데이터 소스** | DB2 / Oracle → Databricks Lakehouse | SAP HANA / BW → Lakehouse Federation |

```sql
-- SAP 데이터 수집 패턴 예시
-- 방법 1: Lakehouse Federation (소량 실시간)
CREATE CONNECTION sap_hana_conn
TYPE JDBC
OPTIONS (
  host '<sap-hana-host>',
  port '30015',
  user '<user>',
  password secret('<scope>', '<key>')
);

-- 방법 2: 배치 수집 대량 권장 (Fivetran / Informatica → Bronze)
-- Bronze → Silver → Gold (Medallion) 파이프라인 구성 후
-- Genie / AI/BI Dashboard를 Gold 테이블에 연결
```

---

## 9. 권장 하이브리드 아키텍처 요약

| 사용 사례 | 도구 | 이유 |
|----------|------|------|
| **C-레벨 KPI 대시보드** | AI/BI Dashboard | 실시간 데이터, 간결한 시각화 |
| **자연어 ad-hoc 분석** | Genie | 비기술 사용자 자율 분석 |
| **재무 / 규제 정기 보고서** | Tableau / Power BI | 기존 양식 유지, 인쇄 최적화 |
| **고객 대면 포털** | Databricks Apps | 추가 BI 라이선스 불필요 |
| **데이터 알림** | Alerts | Slack / 이메일 자동 알림 |
| **데이터 품질 모니터링** | AI/BI Dashboard + Alerts | 파이프라인 상태 실시간 감시 |

---

## 참고 링크

- [Databricks: Partner Connect](https://docs.databricks.com/aws/en/partners/)
- [Databricks: BI Tool Connectivity](https://docs.databricks.com/aws/en/partners/bi/)
- [Databricks: Metric Views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-metric-view.html)
- [Databricks: AI/BI Dashboards](https://docs.databricks.com/aws/en/aibi/dashboards/)
- [Databricks: SQL Warehouses — Sizing Guide](https://docs.databricks.com/aws/en/compute/sql-warehouse/warehouse-behavior.html)
- [Databricks: SQL Connector for Python](https://docs.databricks.com/aws/en/dev-tools/python-sql-connector.html)
- [Databricks: Materialized Views](https://docs.databricks.com/aws/en/views/materialized.html)
- [Tableau: Databricks Connector](https://help.tableau.com/current/pro/desktop/en-us/examples_databricks.htm)
- [Power BI: Azure Databricks Connector](https://learn.microsoft.com/en-us/power-bi/connect-data/service-azure-databricks)
- [dbt Semantic Layer (MetricFlow)](https://docs.getdbt.com/docs/build/about-metricflow)
