# AI/BI 대시보드 (Lakeview)

## 대시보드란?

> 💡 **AI/BI Dashboard (Lakeview)** 는 SQL 쿼리 결과를 **차트, 표, KPI 카운터** 등으로 시각화하는 Databricks의 내장 대시보드 도구입니다. 데이터셋(SQL 쿼리)을 정의하면 AI가 적절한 차트를 추천하기도 합니다.

---

## 대시보드 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **Dataset (데이터셋)** | 대시보드에 데이터를 공급하는 SQL 쿼리입니다. 하나의 대시보드에 여러 데이터셋을 정의할 수 있습니다 |
| **Widget (위젯)** | 데이터를 시각화하는 개별 구성 요소입니다 (차트, 표, 텍스트 등) |
| **Filter (필터)** | 사용자가 대화형으로 데이터 범위를 조절할 수 있는 컨트롤입니다 |
| **Parameter (매개변수)** | 여러 위젯에서 공통으로 사용하는 동적 변수입니다 |
| **Page (페이지)** | 대시보드를 여러 탭으로 나누어 구성할 수 있습니다 |
| **Canvas (캔버스)** | 위젯을 자유롭게 배치하는 레이아웃 영역입니다 |

---

## 위젯 유형

### 시각화 위젯

| 위젯 | 적합한 데이터 | 예시 |
|------|-------------|------|
| **Bar Chart (막대)** | 카테고리별 비교 | 지역별 매출, 상품별 판매량 |
| **Line Chart (선)** | 시계열 추이 | 일별 매출 추이, 월별 사용자 수 |
| **Area Chart (영역)** | 시계열 + 구성 비율 | 채널별 매출 추이 (누적) |
| **Pie / Donut (원)** | 비율 비교 | 카테고리별 매출 비중 |
| **Scatter (산점도)** | 두 변수 관계 | 가격 vs 판매량 상관관계 |
| **Counter (카운터)** | 단일 KPI 수치 | 총 매출, 오늘 주문 수, 전월 대비 성장률 |
| **Table (테이블)** | 상세 데이터 | 최근 주문 목록, 고객 리스트 |
| **Pivot Table (피벗)** | 크로스탭 분석 | 지역 × 월별 매출 매트릭스 |
| **Heatmap (히트맵)** | 2차원 밀도 | 시간대 × 요일별 트래픽 |
| **Map (지도)** | 지리 데이터 | 지역별 매출 지도 |

### 비시각화 위젯

| 위젯 | 용도 |
|------|------|
| **Text (텍스트)** | Markdown으로 제목, 설명, 주석을 추가합니다 |
| **Image (이미지)** | 로고, 범례 등을 삽입합니다 |
| **Filter Control** | 날짜 범위, 드롭다운 등 필터 UI를 배치합니다 |

---

## 대시보드 생성 방법

### Step 1: 대시보드 생성

1. 좌측 메뉴 **Dashboards** → **Create Dashboard**
2. 대시보드 이름 입력 (예: "월간 매출 리포트")

### Step 2: 데이터셋 정의

**Data** 탭에서 SQL 쿼리로 데이터셋을 정의합니다.

```sql
-- 데이터셋 1: 일별 매출
SELECT
    order_date,
    region,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value
FROM gold.daily_orders
WHERE order_date >= :date_start AND order_date <= :date_end
GROUP BY order_date, region
ORDER BY order_date;

-- 데이터셋 2: KPI 요약
SELECT
    SUM(amount) AS total_revenue,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(*) AS total_orders,
    ROUND(SUM(amount) / COUNT(DISTINCT customer_id), 0) AS revenue_per_customer
FROM gold.daily_orders
WHERE order_date >= :date_start AND order_date <= :date_end;
```

> 💡 `:date_start`, `:date_end`는 **매개변수**입니다. 필터 위젯과 연동하여 사용자가 날짜 범위를 동적으로 변경할 수 있습니다.

### Step 3: 위젯 배치

**Canvas** 탭에서 위젯을 추가하고 배치합니다.

- **+ Add** → **Visualization** → 차트 유형 선택
- 데이터셋 연결 → X축, Y축, 색상 등 설정
- 드래그로 크기 조절 및 위치 배치

### Step 4: 필터 추가

- **+ Add** → **Filter** → 필터 유형 선택 (날짜 범위, 드롭다운 등)
- 필터를 데이터셋의 매개변수와 연결

### Step 5: 공유 및 게시

- **Share** 버튼으로 사용자/그룹에게 공유
- **Publish** 버튼으로 게시 (게시 후에도 수정 가능)

---

## 대시보드 공유 방법

| 방법 | 설명 | 적합한 사용 |
|------|------|-----------|
| **Workspace 내 공유** | 사용자/그룹에게 View/Edit 권한 부여 | 팀 내 공유 |
| **스케줄 이메일** | 정해진 시간에 대시보드 스냅샷을 이메일로 전송 | 정기 리포트 |
| **PDF 다운로드** | 현재 상태를 PDF로 저장 | 오프라인 공유, 인쇄 |
| **임베딩 (Embed)** | iframe으로 외부 사이트/포털에 삽입 | 사내 포털, 고객 포탈 |
| **구독** | 사용자가 직접 이메일 알림을 구독 | 자율 구독 |

### 스케줄 이메일 설정

```
갱신 빈도: 매일 오전 9시 (Asia/Seoul)
수신자: team-analytics@company.com
형식: 대시보드 링크 + PDF 첨부
```

---

## 대시보드 디자인 모범 사례

| 원칙 | 설명 |
|------|------|
| **핵심 KPI를 상단에** | 가장 중요한 Counter 위젯을 대시보드 최상단에 배치합니다 |
| **왼쪽→오른쪽, 위→아래** | 사람이 자연스럽게 읽는 방향(Z패턴)으로 정보를 배치합니다 |
| **필터는 상단에** | 날짜 범위 등 필터 컨트롤은 상단에 배치하여 쉽게 접근합니다 |
| **색상 일관성** | 동일한 카테고리에는 동일한 색상을 사용합니다 |
| **페이지 분리** | 내용이 많으면 "요약", "상세", "지역별" 등으로 페이지를 나눕니다 |
| **Gold 테이블 사용** | 복잡한 쿼리는 Gold 테이블에서 사전 집계하여 대시보드 성능을 높입니다 |

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **데이터셋** | SQL 쿼리로 데이터를 정의합니다. 매개변수로 동적 필터링이 가능합니다 |
| **위젯** | 10+ 차트 유형으로 데이터를 시각화합니다 |
| **필터** | 날짜 범위, 드롭다운 등으로 대화형 탐색이 가능합니다 |
| **멀티 페이지** | 하나의 대시보드에 여러 페이지를 구성합니다 |
| **스케줄 공유** | 정기적으로 이메일/PDF로 자동 공유합니다 |

---

## 참고 링크

- [Databricks: AI/BI Dashboards](https://docs.databricks.com/aws/en/aibi/dashboards/)
- [Databricks: Create a dashboard](https://docs.databricks.com/aws/en/aibi/dashboards/create.html)
- [Azure Databricks: Dashboards](https://learn.microsoft.com/en-us/azure/databricks/aibi/dashboards/)
