# AI/BI 개요

## 왜 AI/BI가 등장했는가 — 전통 BI의 한계

> 💡 **AI/BI** 는 기존 BI 도구가 해결하지 못한 세 가지 근본 문제에 대한 Databricks의 해답입니다.

### 문제 1: 대시보드 피로 (Dashboard Fatigue)

전통 BI 환경에서는 분석가가 사전에 예측한 질문만 대시보드로 만들 수 있습니다. 조직 내 대시보드 수가 수백~수천 개로 늘어나면서 "어떤 대시보드가 최신인가?", "이 숫자가 맞는가?"라는 불신이 생깁니다. 정작 필요한 질문은 대시보드에 없어 다시 분석가에게 요청하는 악순환이 반복됩니다.

### 문제 2: 셀프서비스의 실패 (Self-Service BI의 현실)

Tableau, Power BI는 "비즈니스 사용자도 직접 분석할 수 있다"고 약속했지만, 실제로는:
- 드래그앤드롭 인터페이스를 배우는 데 수 일이 걸립니다.
- 복잡한 계산식, LOD 표현식, DAX 함수는 결국 IT/분석가가 담당합니다.
- 데이터 모델 구조를 모르면 잘못된 집계를 만들어도 알아차리기 어렵습니다.

결과적으로 "셀프서비스"는 일부 파워유저에게만 해당되었고, 비기술 비즈니스 사용자는 여전히 분석가 대기열에 줄을 섭니다.

### 문제 3: 데이터 문해력 격차 (Data Literacy Gap)

조직의 80~90%는 SQL을 모릅니다. 분석가는 소수입니다. 데이터 기반 의사결정을 하고 싶어도 실제로 데이터를 조회할 수 있는 사람이 병목이 됩니다. AI가 SQL을 대신 작성해 주지 않는 한 이 격차는 해소되지 않습니다.

| 한계 | 증상 | AI/BI의 해법 |
|------|------|------------|
| **대시보드 피로** | 수천 개 대시보드, 불신 | Genie — 질문할 때마다 최신 데이터로 답변 |
| **셀프서비스 실패** | IT 병목, 대기열 | 자연어 → SQL 자동 생성 |
| **데이터 문해력 격차** | 80% 직원이 데이터 접근 불가 | SQL 없이 대화로 분석 |

---

## Databricks AI/BI란?

> 💡 **AI/BI** 는 Databricks의 비즈니스 인텔리전스 솔루션으로, 데이터 분석가뿐만 아니라 **비기술 비즈니스 사용자** 도 데이터에서 인사이트를 얻을 수 있도록 설계된 도구 모음입니다.

Databricks는 2024년 AI/BI를 발표하면서 "**Intelligence is the new interface**"라는 비전을 제시했습니다. 핵심 아이디어는 다음과 같습니다:

- **레이크하우스 위에서 직접 실행**: 데이터를 BI 도구로 복사하지 않고 Delta Lake에서 직접 조회합니다.
- **LLM이 SQL 중간자 역할**: 자연어 질문을 LLM이 SQL로 변환하여 실행합니다.
- **Unity Catalog 거버넌스 통합**: 테이블 권한, COMMENT, 메타데이터가 AI 정확도와 직결됩니다.
- **Lakeview Dashboard + Genie의 조합**: 정형화된 리포트(대시보드)와 탐색적 대화(Genie)를 하나의 플랫폼에서 제공합니다.

---

## 핵심 구성 요소

| 구성 요소 | 역할 | 대상 사용자 |
|-----------|------|-----------|
| **AI/BI Dashboard (Lakeview)** | SQL 쿼리 결과를 차트, 표, KPI로 시각화합니다 | 분석가, 비즈니스 사용자 |
| **Genie** | 자연어로 데이터에 질문하면 SQL을 자동 생성하여 답변합니다 | 비기술 사용자, 경영진 |
| **Alerts** | SQL 쿼리 결과가 조건을 만족하면 자동으로 알림을 보냅니다 | 운영팀, 분석가 |
| **Metric Views** | 비즈니스 메트릭을 중앙에서 정의하고 일관되게 사용합니다 | 전 조직 (UC 거버넌스 적용) |

### Lakeview Dashboard — 코드 기반 대시보드

기존 Tableau/Power BI의 바이너리 파일 방식과 달리, Lakeview 대시보드는 **JSON 직렬화 포맷** 으로 저장됩니다. 이는 다음을 의미합니다:

- **Git 버전 관리 가능**: 대시보드 변경 이력을 코드처럼 추적할 수 있습니다.
- **프로그래밍 방식 배포**: Databricks Asset Bundle (DAB)로 CI/CD 파이프라인에 통합됩니다.
- **API 기반 생성**: REST API로 대시보드를 동적으로 생성·수정할 수 있습니다.

### Genie — 자연어 질의 엔진

Genie는 단순한 "Text-to-SQL" 도구가 아닙니다. 아래 세 가지를 조합하여 정확도를 높입니다:

1. **테이블/컬럼 COMMENT**: Unity Catalog에 등록된 메타데이터를 컨텍스트로 활용합니다.
2. **Trusted Answers (인증된 답변)**: 관리자가 사전 검증한 질문-SQL 쌍을 우선 매칭합니다.
3. **Instructions (지침)**: Space 수준의 비즈니스 규칙과 용어 정의를 LLM에 주입합니다.

### Metric View — 통일된 메트릭 정의

**Metric View** 는 Unity Catalog 객체로, 비즈니스 메트릭(KPI)을 SQL로 한 번만 정의하면 대시보드·Genie·외부 도구에서 일관되게 참조할 수 있습니다.

```sql
-- Metric View 생성 예시
CREATE METRIC VIEW sales.gold.revenue_metrics
  COMMENT '영업팀 핵심 KPI 정의'
AS
  SELECT
    sale_date,
    region,
    SUM(amount) AS total_revenue,
    COUNT(DISTINCT customer_id) AS active_customers,
    SUM(amount) / COUNT(DISTINCT customer_id) AS revenue_per_customer
  FROM sales.gold.daily_orders
  GROUP BY sale_date, region;
```

---

## 전통 BI vs AI/BI — 상세 비교

| 비교 항목 | 전통 BI (Tableau, Power BI) | Databricks AI/BI |
|-----------|----------------------------|-----------------|
| **데이터 위치** | BI 도구로 데이터를 추출/복사해야 함 | 레이크하우스에서 **직접 조회** |
| **AI 기능** | 제한적 | **네이티브 AI** — Genie, AI 차트 추천 |
| **거버넌스** | 별도 관리, 이중 권한 체계 | **Unity Catalog 통합** — 단일 권한 소스 |
| **실시간성** | 추출 주기에 의존 (일 1회, 시간 단위) | Delta Lake **최신 데이터** 즉시 조회 |
| **비용 모델** | 사용자당 월 라이선스 (Tableau: $70~$840/user) | 플랫폼 **내장** — SQL Warehouse 쿼리 비용만 |
| **대시보드 생성** | GUI 드래그앤드롭, 바이너리 파일 저장 | SQL + JSON, **Git/CI-CD 관리 가능** |
| **질문-답변** | 사전 정의된 필터/드릴다운만 | **자연어 대화** — 사전 미정의 질문도 답변 |
| **메타데이터 활용** | BI 도구 내 별도 정의 | Unity Catalog COMMENT가 **AI 정확도에 직결** |
| **학습 곡선** | 비기술 사용자: 수 일~수 주 | Genie: **즉시 사용 가능** (자연어) |
| **복잡 계산** | DAX, LOD 표현식 필요 | Metric View로 **중앙 정의 후 재사용** |

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

## 실전 사용 시나리오

### 시나리오 1: 경영진 KPI 대시보드

**상황**: CMO가 매주 임원 보고용 매출·고객·캠페인 대시보드를 원합니다. 중간에 "이번 분기 해지 고객 중 ARPU 상위 10%는 누구인가?"라는 즉석 질문이 나옵니다.

**AI/BI 활용 방법**:
1. 분석가가 Lakeview Dashboard로 정기 KPI 대시보드를 만들어 공유합니다.
2. 회의 중 즉석 질문은 CMO가 직접 Genie에 자연어로 입력합니다.
3. Genie가 SQL을 생성·실행하여 30초 내 결과를 반환합니다.

**기대 효과**: 분석가 요청 → 대기 → 결과 사이클(평균 1~2일)이 즉시 응답으로 단축됩니다.

### 시나리오 2: 현업 셀프서비스 분석

**상황**: 영업팀 매니저가 특정 지역의 신규 고객 현황을 매일 확인하고 싶습니다. SQL은 모릅니다.

**AI/BI 활용 방법**:
1. 데이터 팀이 `영업 분석` Genie Space를 만들고 관련 Gold 테이블 3~5개를 연결합니다.
2. Instructions에 지역 코드, 고객 등급 정의 등 비즈니스 용어를 상세히 작성합니다.
3. 매니저는 "지난 주 서울 지역 신규 고객 수와 평균 계약 금액을 보여줘"라고 입력합니다.

**기대 효과**: 분석가 없이 매일 스스로 데이터를 조회할 수 있어 의사결정 속도가 향상됩니다.

### 시나리오 3: 데이터팀 파이프라인 모니터링

**상황**: 데이터 엔지니어가 야간 배치 파이프라인의 이상 여부를 실시간으로 모니터링하고 싶습니다.

**AI/BI 활용 방법**:
1. Lakeview Dashboard에 파이프라인 성공률, 처리 건수, 지연 시간 위젯을 구성합니다.
2. Alerts로 처리 건수가 전일 대비 30% 이하이면 Slack 채널에 즉시 알림을 보냅니다.
3. 이상 감지 후 Genie에 "오늘 오전 2시 배치에서 실패한 레코드의 공통점은?"을 질문합니다.

**기대 효과**: 이상 감지부터 원인 분석까지 하나의 플랫폼에서 해결됩니다.

---

## 도입 시 고려사항

### 데이터 품질 요구사항

AI/BI, 특히 Genie의 정확도는 **메타데이터 품질** 에 직접 비례합니다. 도입 전 아래 체크리스트를 확인하세요.

| 체크 항목 | 권장 사항 | 이유 |
|----------|----------|------|
| **테이블 COMMENT** | 모든 Gold 테이블에 한글/영문 설명 필수 | Genie가 테이블 용도를 파악하는 데 사용 |
| **컬럼 COMMENT** | 약어·코드 컬럼에 반드시 기재 (`region: KR=한국`) | LLM이 WHERE 조건 생성 시 참조 |
| **명명 규칙** | 스네이크케이스, 의미 있는 이름 (`rev` → `total_revenue`) | 모호한 컬럼명은 Genie 오류의 주요 원인 |
| **Gold 테이블 준비** | Genie Space 대상 테이블은 집계·정제 완료 상태 | Raw 테이블 노출 시 복잡한 JOIN이 필요 |

```sql
-- 메타데이터 보강 예시
ALTER TABLE sales.gold.daily_orders
  SET TBLPROPERTIES ('comment' = '일별 주문 집계 테이블. 취소 건 제외, 배송 완료 기준으로 집계');

ALTER TABLE sales.gold.daily_orders
  ALTER COLUMN region COMMENT 'KR=한국, JP=일본, US=미국, SG=싱가포르';

ALTER TABLE sales.gold.daily_orders
  ALTER COLUMN total_revenue COMMENT '배송비 포함, 할인 적용 후 최종 결제 금액 (KRW 기준)';
```

### SQL Warehouse 비용 관리

AI/BI는 모든 쿼리를 SQL Warehouse에서 실행합니다. 예상치 못한 비용 발생을 방지하기 위해:

- **Serverless SQL Warehouse**: 사용한 쿼리 시간만 청구, Auto Stop 10분 권장
- **대시보드 새로고침 주기**: 실시간이 필요 없다면 1시간 이상 캐시 활용
- **Genie Space 접근 제어**: 전체 공개 대신 필요한 팀에게만 권한 부여
- **쿼리 최적화**: Gold 테이블에 Liquid Clustering 적용, 대형 JOIN 제거

### 기존 BI 도구와의 공존 (하이브리드 전략)

Databricks AI/BI로 전면 교체가 어려운 경우, 역할 분담 전략을 권장합니다:

| 용도 | 권장 도구 | 이유 |
|------|----------|------|
| **픽셀 퍼펙트 리포트** (인쇄, PDF 제출) | Tableau, Power BI | 정교한 레이아웃 제어 |
| **임시 탐색 질문** | Genie | 자연어 대화로 즉시 답변 |
| **실시간 운영 대시보드** | Lakeview Dashboard | Delta Lake 직접 조회 |
| **복잡한 통계 분석** | Databricks Notebook | Python/R 자유도 |
| **경영진 셀프서비스** | Genie | SQL 없이 즉시 사용 |

---

## 장단점과 한계

### 현재 할 수 있는 것

| 기능 | 상세 |
|------|------|
| **자연어 → SQL 자동 생성** | 한국어 질문도 지원 (2024년 이후) |
| **대시보드 Git 관리** | DAB (Databricks Asset Bundle)로 CI/CD |
| **Metric View 중앙 정의** | 전사 KPI를 Unity Catalog에서 일원 관리 |
| **Row-Level Security 연동** | Unity Catalog RLS가 Genie/Dashboard에도 적용 |
| **MCP 연동** | Genie Agent를 외부 AI 에이전트에 노출 가능 |
| **embedded 대시보드** | iFrame으로 외부 앱에 대시보드 삽입 가능 |

### 현재 할 수 없는 것 (한계)

| 한계 | 설명 | 우회 방법 |
|------|------|----------|
| **픽셀 퍼펙트 레이아웃** | 인쇄용 정밀 레이아웃 불가 | Tableau/Power BI 사용 |
| **오프라인/로컬 접근** | 항상 인터넷 + SQL Warehouse 필요 | 없음 |
| **Genie 다중 테이블 복잡 JOIN** | 20개 이상 테이블 정확도 급감 | Space를 도메인별로 분리 |
| **Genie 쓰기 작업** | SELECT만 가능, DML 불가 | Workflow/Job 사용 |
| **모바일 앱** | 전용 모바일 앱 없음 (브라우저만) | PWA 방식으로 접근 |

### 경쟁사 BI 대비 포지셔닝

| 도구 | 강점 | Databricks AI/BI와의 차별점 |
|------|------|---------------------------|
| **Tableau** | 시각화 표현력, 픽셀 퍼펙트 | AI/BI: 레이크하우스 직접 연결, 자연어 질의 |
| **Power BI** | Microsoft 생태계, 모바일 | AI/BI: Unity Catalog 거버넌스, Delta 직접 조회 |
| **Looker** | Git 기반 LookML, 시맨틱 레이어 | AI/BI: Metric View + Genie로 유사 기능 제공 |
| **ThoughtSpot** | 자연어 검색 특화 | AI/BI: 레이크하우스 통합, 비용 내장 |

---

## 시작하기

### 권장 학습 순서

```text
1단계: 기반 준비 (1~2시간)
  └── SQL Warehouse (Serverless) 생성
  └── Unity Catalog에 Gold 테이블 준비
  └── 테이블/컬럼 COMMENT 작성

2단계: 첫 대시보드 (30분)
  └── Workspace → Dashboards → Create Dashboard
  └── SQL 쿼리 추가 → 차트 선택 → 저장
  └── 팀원과 공유

3단계: Genie Space 설정 (1시간)
  └── Genie Space 생성 → 테이블 추가 (10개 이내)
  └── Instructions 작성 (비즈니스 용어 정의)
  └── Trusted Answers 등록 (핵심 질문 5~10개)

4단계: 운영 고도화
  └── Metric View 생성 (KPI 표준화)
  └── Alerts 설정 (이상 감지 자동화)
  └── DAB로 대시보드 CI/CD 구성
```

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
| **인증된 답변** | 지원 (관리자가 사전 검증) | 미지원 |
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
- [Databricks: SQL Warehouses](https://docs.databricks.com/aws/en/compute/sql-warehouse/index.html)
- [Databricks: Unity Catalog — Table Comments](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/ownership.html)
