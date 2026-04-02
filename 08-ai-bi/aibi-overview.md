# AI/BI 개요

## Databricks AI/BI란?

> 💡 **AI/BI** 는 Databricks의 비즈니스 인텔리전스 솔루션으로, 데이터 분석가뿐만 아니라 **비기술 비즈니스 사용자** 도 데이터에서 인사이트를 얻을 수 있도록 설계된 도구 모음입니다.

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
| **AI 기능** | 제한적 | **네이티브 AI**— Genie, AI 차트 추천 |
| **거버넌스** | 별도 관리 | **Unity Catalog 통합** |
| **실시간성** | 추출 주기에 의존 | Delta Lake **최신 데이터** 즉시 조회 |
| **비용** | 사용자당 라이선스 | 플랫폼 **내장**(쿼리 비용만) |

> 💡 Databricks AI/BI는 Tableau, Power BI를 **대체** 하는 것이 아니라 **보완** 합니다. 하이브리드 전략에 대한 자세한 내용은 [하이브리드 BI 전략](./hybrid-bi-strategy.md) 문서를 참고하세요.

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

## AI/BI 아키텍처 상세

### 내부 동작 흐름

```text
[AI/BI Dashboard 실행 흐름]

사용자 → Dashboard 열기
         ↓
  SQL 쿼리 실행 요청
         ↓
  SQL Warehouse (Serverless)
  ├── Result Cache 확인 → 히트 시 즉시 반환
  ├── Photon 엔진으로 쿼리 실행
  └── Delta Lake에서 데이터 읽기 (Data Skipping 적용)
         ↓
  시각화 렌더링 (차트/표/KPI)
         ↓
  사용자 브라우저에 표시

[Genie 실행 흐름]

사용자 → 자연어 질문 입력
         ↓
  LLM이 질문 분석
  ├── 테이블 메타데이터 참조 (COMMENT, 컬럼 설명)
  ├── Metric View 정의 참조
  ├── 인증된 답변(Trusted Answer) 매칭
  └── SQL 생성
         ↓
  SQL Warehouse에서 실행
         ↓
  결과를 자연어 + 차트로 반환
```

### 구성 요소 간 관계

| 구성 요소 | 의존 관계 | 비고 |
|----------|----------|------|
| **AI/BI Dashboard** | SQL Warehouse + Delta 테이블 | 쿼리 결과를 시각화 |
| **Genie Space** | SQL Warehouse + 테이블 메타데이터 | LLM이 SQL 생성 |
| **Genie Code** | Serverless Compute + 노트북 환경 | Python/SQL 코드 생성 + 실행 |
| **Alerts** | SQL Warehouse + 스케줄러 | 주기적 쿼리 → 조건 평가 → 알림 |
| **Metric Views** | Unity Catalog | 비즈니스 지표 정의, Genie에서 참조 |

---

## Genie Space vs Genie Code 비교

| 비교 항목 | Genie Space | Genie Code |
|----------|------------|------------|
| **접근 방식** | 사전 구성된 Space에 질문 | 자유로운 탐색적 분석 |
| **대상 사용자** | 비기술 비즈니스 사용자 | 분석가, 데이터 사이언티스트 |
| **쿼리 범위** | 지정된 테이블만 (최대 20개 권장) | Workspace 내 모든 테이블 |
| **생성 코드** | SQL만 | Python + SQL |
| **시각화** | 테이블/차트 자동 생성 | matplotlib, plotly 등 자유 |
| **인증된 답변** | ✅ 지원 (관리자가 사전 검증) | ❌ 미지원 |
| **컴퓨트** | SQL Warehouse | Serverless Compute (노트북) |
| **거버넌스** | Space별 테이블 접근 제어 | Unity Catalog 권한 기반 |
| **MCP 연동** | Genie Agent (Agent Bricks) | AI Dev Kit MCP 서버 |

{% hint style="info" %}
**선택 가이드**: 반복적인 비즈니스 질문에는 **Genie Space** (인증된 답변으로 정확도 보장), 탐색적이고 복잡한 데이터 분석에는 **Genie Code** (Python + 시각화 자유도)를 사용하세요.
{% endhint %}

---

## AI/BI Dashboard 구성 요소 상세

### 위젯 유형

| 위젯 유형 | 용도 | 설정 항목 |
|----------|------|----------|
| **카운터 (Counter)** | 단일 KPI 표시 (매출, 고객 수 등) | 값, 목표값, 비교 기간 |
| **테이블 (Table)** | 상세 데이터 조회 | 정렬, 필터, 컬럼 서식 |
| **막대 차트 (Bar)** | 범주별 비교 | X축, Y축, 색상, 스택 |
| **선 차트 (Line)** | 시계열 트렌드 | X축(시간), Y축, 그룹 |
| **파이 차트 (Pie)** | 비율/구성 | 카테고리, 값 |
| **산점도 (Scatter)** | 변수 간 관계 | X축, Y축, 크기, 색상 |
| **히트맵 (Heatmap)** | 2차원 밀도 표현 | X축, Y축, 값 |
| **텍스트 (Markdown)** | 설명, 제목, 주석 | Markdown 문법 지원 |
| **필터 (Filter)** | 대시보드 전체 필터링 | 드롭다운, 날짜 범위, 텍스트 |

### 대시보드 설계 모범 사례

| 원칙 | 설명 | 예시 |
|------|------|------|
| **3초 규칙** | 핵심 KPI를 3초 안에 파악 가능해야 함 | 상단에 카운터 위젯 배치 |
| **피라미드 구조** | 위→아래로 요약→상세 배치 | KPI → 트렌드 차트 → 상세 테이블 |
| **필터 우선** | 날짜/지역 필터를 최상단에 배치 | 전체 대시보드에 영향 |
| **쿼리 최적화** | Gold 테이블 사용, 불필요한 JOIN 제거 | 응답 시간 2초 이내 목표 |
| **공유 시 권한 확인** | 대시보드와 테이블 SELECT 권한 모두 부여 | 데이터 미표시 방지 |

```sql
-- 대시보드용 Gold 테이블 설계 예시
CREATE TABLE sales.gold.dashboard_daily_summary AS
SELECT
  sale_date,
  region,
  product_category,
  COUNT(*) AS order_count,
  SUM(amount) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers,
  AVG(amount) AS avg_order_value
FROM sales.silver.orders
WHERE sale_date >= current_date() - INTERVAL 365 DAYS
GROUP BY sale_date, region, product_category;

-- Liquid Clustering 적용 (대시보드 필터 패턴에 맞게)
ALTER TABLE sales.gold.dashboard_daily_summary
CLUSTER BY (sale_date, region);
```

---

## Quick Start 가이드

### Step 1: SQL Warehouse 준비

```text
1. Workspace → SQL Warehouses → Create SQL Warehouse
2. Name: "ai-bi-warehouse"
3. Cluster size: Small (시작점)
4. Type: Serverless (권장)
5. Auto Stop: 10분
6. → Create
```

### Step 2: 첫 AI/BI Dashboard 생성

```text
1. Workspace → Dashboards → Create Dashboard
2. + Add Widget → SQL 쿼리 작성
3. 예시 쿼리:
   SELECT region, SUM(revenue) AS total
   FROM sales.gold.daily_revenue
   WHERE sale_date >= '2026-01-01'
   GROUP BY region
4. Visualization → Bar Chart 선택
5. Save → 이름 지정
6. Publish → 공유 대상 지정
```

### Step 3: Genie Space 생성

```text
1. Workspace → Genie → New Genie Space
2. Space 이름: "영업 분석"
3. SQL Warehouse 선택: "ai-bi-warehouse"
4. 테이블 추가 (10개 이내 권장):
   - sales.gold.daily_revenue
   - sales.gold.customer_segments
   - sales.gold.product_performance
5. Instructions 작성:
   "이 Space는 영업팀의 매출 분석용입니다.
    매출은 amount 컬럼, 고객은 customer_id 기준입니다.
    지역은 region 컬럼으로, KR=한국, JP=일본, US=미국입니다."
6. Trusted Answers 등록 (핵심 질문 사전 검증)
7. Save → 사용자 초대
```

{% hint style="warning" %}
**Genie Space Instructions 핵심**: Instructions에 비즈니스 용어 정의, 컬럼 의미, 자주 사용하는 필터 조건을 상세히 작성하세요. Instructions 품질이 Genie 답변 정확도를 80% 이상 좌우합니다.
{% endhint %}

### Step 4: Alert 설정

```sql
-- Alert용 SQL 쿼리 예시: 일일 매출이 목표 대비 80% 미만이면 알림
SELECT
  sale_date,
  SUM(amount) AS daily_revenue,
  1000000 AS target,
  SUM(amount) / 1000000 AS achievement_rate
FROM sales.gold.daily_revenue
WHERE sale_date = current_date() - INTERVAL 1 DAY
GROUP BY sale_date
HAVING SUM(amount) < 1000000 * 0.8;
-- 결과가 있으면 (행이 1개 이상) Alert 트리거
```

```text
Alert 설정 단계:
1. SQL Editor에서 위 쿼리 저장
2. Alerts → New Alert
3. Query 선택 → 트리거 조건: "행이 1개 이상"
4. 알림 채널: Slack 웹훅 또는 이메일
5. 스케줄: 매일 오전 9시
```

---

## 실전 활용 사례

### 사례 1: 제조업 생산 모니터링

| 구성 요소 | 용도 | 상세 |
|----------|------|------|
| **AI/BI Dashboard** | 실시간 생산 현황 | 라인별 생산량, 불량률, 설비 가동률 |
| **Alerts** | 이상 감지 | 불량률 > 5% 시 Slack 알림 |
| **Genie** | 원인 분석 | "어제 3라인 불량률이 높았던 원인은?" |

### 사례 2: 리테일 매출 분석

| 구성 요소 | 용도 | 상세 |
|----------|------|------|
| **AI/BI Dashboard** | 일별/주별 매출 트렌드 | 지역별, 카테고리별 매출 비교 |
| **Genie** | 경영진 즉석 질문 | "이번 분기 상위 10개 매장 매출은?" |
| **Metric Views** | KPI 표준화 | 매출, 객단가, 재방문율 통일 정의 |

### 사례 3: 금융 리스크 모니터링

| 구성 요소 | 용도 | 상세 |
|----------|------|------|
| **AI/BI Dashboard** | 포트폴리오 리스크 현황 | VaR, 신용등급 분포, 연체율 |
| **Alerts** | 규제 임계값 위반 | 자본적정성 비율 < 8% 시 즉시 알림 |
| **Genie** | 규제 보고 지원 | "이번 달 신규 연체 고객 목록은?" |

{% hint style="info" %}
**업종별 Genie Space 분리 전략**: 하나의 거대한 Genie Space보다 업무 도메인별로 작은 Space를 여러 개 만드는 것이 정확도에 유리합니다. 예: "매출 분석" Space, "고객 분석" Space, "재고 분석" Space.
{% endhint %}

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
- [Databricks: Metric Views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-metric-view.html)
- [Databricks: Genie Code](https://docs.databricks.com/aws/en/genie/genie-code.html)
