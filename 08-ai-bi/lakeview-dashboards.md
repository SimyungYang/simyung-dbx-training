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

1. 좌측 메뉴 **Dashboards**→ **Create Dashboard**
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

> 💡 `:date_start`, `:date_end`는 **매개변수** 입니다. 필터 위젯과 연동하여 사용자가 날짜 범위를 동적으로 변경할 수 있습니다.

### Step 3: 위젯 배치

**Canvas** 탭에서 위젯을 추가하고 배치합니다.

- **+ Add**→ **Visualization**→ 차트 유형 선택
- 데이터셋 연결 → X축, Y축, 색상 등 설정
- 드래그로 크기 조절 및 위치 배치

### Step 4: 필터 추가

- **+ Add**→ **Filter**→ 필터 유형 선택 (날짜 범위, 드롭다운 등)
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

## 심화: 엔터프라이즈 대시보드 운영

이 섹션에서는 수백 명의 동시 사용자를 지원하는 대규모 환경에서의 대시보드 성능 최적화, 권한 관리, 프로그래밍 방식의 관리 방법을 다룹니다.

### 쿼리 성능 최적화

대시보드의 체감 성능은 결국 **SQL 쿼리가 얼마나 빨리 실행되느냐** 에 달려 있습니다.

#### Materialized View 활용

복잡한 집계 쿼리를 매번 실행하는 대신, **Materialized View(구체화된 뷰)** 로 사전 계산된 결과를 활용하면 대시보드 로딩 시간을 획기적으로 줄일 수 있습니다.

```sql
-- Materialized View 생성 (SDP/DLT에서)
CREATE OR REFRESH MATERIALIZED VIEW gold.daily_revenue_summary AS
SELECT
    order_date,
    region,
    product_category,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM silver.orders
GROUP BY order_date, region, product_category;

-- 대시보드 데이터셋에서는 단순 SELECT만 실행
SELECT * FROM gold.daily_revenue_summary
WHERE order_date BETWEEN :date_start AND :date_end;
```

| 쿼리 방식 | 예상 실행 시간 | 비용 |
|-----------|-------------|------|
| 원본 테이블 직접 집계 (10억 건) | 30초~2분 | 높음 (풀 스캔) |
| Materialized View 조회 | **1~5초** | 낮음 (사전 계산) |
| Materialized View + 파티션 프루닝 | **< 1초** | 매우 낮음 |

#### 파티션 프루닝과 Data Skipping

```sql
-- ✅ 좋은 쿼리: 파티션 컬럼(order_date)으로 필터링
SELECT region, SUM(amount)
FROM gold.daily_orders
WHERE order_date >= '2025-03-01'  -- 파티션 프루닝 발생
GROUP BY region;

-- ❌ 나쁜 쿼리: 함수로 감싸면 파티션 프루닝이 안 됨
SELECT region, SUM(amount)
FROM gold.daily_orders
WHERE YEAR(order_date) = 2025 AND MONTH(order_date) = 3  -- 풀 스캔!
GROUP BY region;
```

#### 쿼리 캐싱 전략

Databricks SQL Warehouse는 **결과 캐시(Result Cache)** 와 **디스크 캐시(Disk Cache)** 를 제공합니다.

| 캐시 유형 | 동작 | 유효 조건 |
|-----------|------|----------|
| **Result Cache** | 동일 쿼리의 결과를 메모리에 캐싱 | 기저 테이블이 변경되지 않은 경우 (최대 24시간) |
| **Disk Cache (SSD)** | 원격 스토리지 데이터를 로컬 SSD에 캐싱 | 같은 데이터를 반복 접근하는 경우 |

> ⚠️ **Gotcha**: 대시보드에 매개변수(`:date_start` 등)를 사용하면, 매개변수 값이 다를 때마다 **다른 쿼리로 인식** 되어 Result Cache가 적중하지 않습니다. 자주 사용되는 날짜 범위(이번 달, 지난 7일 등)를 기본값으로 설정하면 캐시 적중률을 높일 수 있습니다.

---

### 새로고침(Refresh) 전략

대시보드 데이터의 신선도(Freshness)와 비용 사이의 균형을 잡는 것이 중요합니다.

| 전략 | 설정 | 신선도 | 비용 |
|------|------|--------|------|
| **실시간 (기본)** | 대시보드 열 때마다 쿼리 실행 | 항상 최신 | 높음 (매 조회 시 쿼리 실행) |
| **스케줄 실행** | 정해진 주기로 대시보드 새로고침 | 주기적 | 중간 (정해진 시간에만 실행) |
| **Materialized View + SDP** | SDP 파이프라인이 MV를 갱신 | 파이프라인 주기 | 낮음 (증분 처리) |

```
권장 패턴:

[Source Tables] → [SDP Pipeline (15분 주기)]
    → [Materialized View (Gold)]
        → [대시보드 데이터셋 (단순 SELECT)]
            → [스케줄 새로고침 (매시간)]
                → [이메일 전송 (매일 오전 9시)]
```

> ⚠️ **Gotcha — 새로고침 중 대시보드 접근**: 대시보드가 스케줄 새로고침 중일 때 사용자가 접근하면, **마지막 새로고침 결과** 가 표시됩니다. 실시간 데이터가 필요한 경우, 사용자가 수동으로 "Run" 버튼을 클릭하여 최신 데이터를 조회할 수 있습니다.

---

### 대규모 운영: 동시 사용자 대응

수백 명의 사용자가 동시에 대시보드를 조회하는 환경에서의 설계 전략입니다.

#### SQL Warehouse 사이징

| 동시 사용자 | 권장 Warehouse 크기 | Scaling 설정 | 예상 비용 |
|------------|-------------------|-----------| ---------|
| 1~10명 | Small (1 클러스터) | Min 1, Max 1 | ~$5/시간 |
| 10~50명 | Medium | Min 1, Max 3 | ~$10~30/시간 |
| 50~200명 | Large | Min 2, Max 5 | ~$30~75/시간 |
| 200~500명 | Large | Min 3, Max 10 | ~$50~150/시간 |
| 500명+ | X-Large + Auto Scaling | Min 5, Max 20 | $100+/시간 |

> 💡 **Serverless SQL Warehouse** 를 사용하면 Auto Scaling이 더 빠르게 반응합니다 (스케일아웃 시간: 기존 수 분 → Serverless **10~20초**). 대규모 동시 사용자 환경에서는 Serverless를 강력히 권장합니다.

#### 쿼리 큐잉과 동시성

```
SQL Warehouse의 동시 쿼리 처리:
- 각 클러스터: 최대 10~50개 동시 쿼리 (쿼리 복잡도에 따라 다름)
- 큐잉: 동시 실행 한도 초과 시 큐에 대기 (기본 타임아웃: 8시간)
- 사용자 체감: 큐잉 시 대시보드 로딩 대기 시간 증가
```

> ⚠️ **Gotcha — "월요일 오전 9시" 문제**: 전사 대시보드를 월요일 아침에 이메일로 배포하면, 수백 명이 동시에 대시보드를 열어 SQL Warehouse에 부하가 집중됩니다. 이메일 스케줄을 **시차 배포**(팀별로 9시, 9시 15분, 9시 30분)하거나, 스케줄 새로고침으로 사전 캐싱하세요.

---

### 권한 모델

#### 대시보드 접근 권한

| 역할 | 권한 | 설명 |
|------|------|------|
| **Owner (소유자)** | 전체 관리 | 대시보드 수정, 삭제, 공유, 권한 관리 |
| **CAN EDIT** | 편집 | 데이터셋, 위젯 수정 가능. 권한 관리는 불가 |
| **CAN RUN** | 실행 | 대시보드 조회, 필터 변경, 새로고침 가능. 수정 불가 |
| **CAN VIEW** | 조회 | 게시된 대시보드만 조회 가능. 편집 모드 접근 불가 |

#### Unity Catalog 연동 — 데이터 접근 제어

대시보드의 데이터 접근은 두 가지 모드로 동작합니다.

| 모드 | 설명 | 적합한 상황 |
|------|------|-----------|
| **Viewer Credentials (뷰어 인증)** | 조회자 본인의 UC 권한으로 쿼리 실행 | Row Filter/Column Mask 적용 필요 시 |
| **Owner Credentials (소유자 인증)** | 대시보드 소유자의 권한으로 쿼리 실행 | 조회자에게 테이블 직접 접근 권한 없이 대시보드만 공유 |

```
시나리오: 영업팀 대시보드
- 데이터: 전체 고객 매출 데이터
- 요구사항: 각 영업 담당자는 자기 지역 데이터만 볼 수 있어야 함

해결:
1. UC Row Filter로 사용자별 지역 필터 설정
2. 대시보드를 "Viewer Credentials" 모드로 설정
3. 각 사용자가 대시보드를 열면 자기 지역 데이터만 표시됨
```

> ⚠️ **Gotcha — Owner Credentials의 보안 위험**: Owner Credentials 모드에서는 소유자의 권한으로 쿼리가 실행되므로, 소유자가 접근 가능한 **모든 데이터가 조회자에게 노출** 될 수 있습니다. 민감 데이터가 포함된 대시보드는 반드시 Viewer Credentials를 사용하세요.

---

### 프로그래밍 방식 관리

#### REST API로 대시보드 관리

Lakeview 대시보드는 **JSON 직렬화 형식** 으로 정의되며, REST API를 통해 프로그래밍 방식으로 생성, 수정, 삭제할 수 있습니다.

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 대시보드 목록 조회
dashboards = w.lakeview.list()
for d in dashboards:
    print(f"{d.dashboard_id}: {d.display_name}")

# 대시보드 정의(JSON) 내보내기
dashboard = w.lakeview.get(dashboard_id="<dashboard-id>")
serialized = dashboard.serialized_dashboard  # JSON 문자열

# 대시보드 생성
new_dashboard = w.lakeview.create(
    display_name="매출 대시보드 (복제)",
    serialized_dashboard=serialized,  # 기존 대시보드의 JSON 재사용
    warehouse_id="<warehouse-id>"
)

# 대시보드 게시
w.lakeview.publish(
    dashboard_id=new_dashboard.dashboard_id,
    embed_credentials=True,  # Owner Credentials
    warehouse_id="<warehouse-id>"
)
```

#### Asset Bundles 연동 (IaC)

Databricks Asset Bundles(DABs)를 사용하면 대시보드를 **코드로 관리(Infrastructure as Code)** 할 수 있습니다.

```yaml
# databricks.yml
bundle:
  name: analytics-dashboards

resources:
  dashboards:
    monthly_revenue:
      display_name: "월간 매출 리포트"
      file_path: ./dashboards/monthly_revenue.lvdash.json
      warehouse_id: ${var.warehouse_id}
      permissions:
        - group_name: analytics-team
          permission_level: CAN_VIEW
        - group_name: analytics-admins
          permission_level: CAN_EDIT
```

```bash
# 배포
databricks bundle deploy --target production

# 여러 환경에 동일 대시보드 배포
databricks bundle deploy --target staging
databricks bundle deploy --target production
```

#### CI/CD 파이프라인 예시

```
[Git Push] → [CI: 대시보드 JSON 유효성 검사]
    → [CD: staging 환경 배포]
        → [수동 검토]
            → [CD: production 환경 배포]
                → [Slack 알림]
```

| 단계 | 도구 | 설명 |
|------|------|------|
| 버전 관리 | Git | `.lvdash.json` 파일을 Git으로 관리 |
| 유효성 검사 | `databricks bundle validate` | JSON 형식, 참조 테이블 존재 여부 확인 |
| 배포 | `databricks bundle deploy` | 환경별(dev/staging/prod) 배포 |
| 테스트 | REST API | 배포 후 대시보드 쿼리 실행 + 결과 검증 |

> ⚠️ **Gotcha — 대시보드 JSON 충돌**: 두 명이 동시에 같은 대시보드를 UI에서 수정하면, 나중에 저장한 사람의 변경만 반영됩니다. Asset Bundles + Git을 사용하면 **변경 이력 추적과 충돌 해결** 이 가능합니다.

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **데이터셋** | SQL 쿼리로 데이터를 정의합니다. 매개변수로 동적 필터링이 가능합니다 |
| **위젯** | 10+ 차트 유형으로 데이터를 시각화합니다 |
| **필터** | 날짜 범위, 드롭다운 등으로 대화형 탐색이 가능합니다 |
| **멀티 페이지** | 하나의 대시보드에 여러 페이지를 구성합니다 |
| **스케줄 공유** | 정기적으로 이메일/PDF로 자동 공유합니다 |
| **Materialized View** | 사전 계산된 결과로 대시보드 성능을 최적화합니다 |
| **Asset Bundles** | 대시보드를 코드로 관리하여 CI/CD 파이프라인에 통합합니다 |
| **권한 모델** | UC 연동으로 행/컬럼 수준의 데이터 접근 제어가 가능합니다 |

---

## 참고 링크

- [Databricks: AI/BI Dashboards](https://docs.databricks.com/aws/en/aibi/dashboards/)
- [Databricks: Create a dashboard](https://docs.databricks.com/aws/en/aibi/dashboards/create.html)
- [Azure Databricks: Dashboards](https://learn.microsoft.com/en-us/azure/databricks/aibi/dashboards/)
