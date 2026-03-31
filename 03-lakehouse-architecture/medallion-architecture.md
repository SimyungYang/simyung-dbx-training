# Medallion 아키텍처

## Medallion 아키텍처란?

> 💡 **Medallion(메달리온) 아키텍처**는 데이터를 품질과 가공 수준에 따라 **Bronze(원본)**, **Silver(정제)**, **Gold(집계)** 세 계층으로 나누어 관리하는 데이터 설계 패턴입니다.

이름의 유래는 올림픽 메달에서 왔습니다. 동메달(Bronze)부터 금메달(Gold)로 갈수록 데이터의 품질과 비즈니스 가치가 높아집니다.

![Medallion 아키텍처 — Bronze, Silver, Gold 계층](https://docs.databricks.com/en/_images/medallion-architecture.png)

> 출처: [Databricks 공식 문서 — Medallion Architecture](https://docs.databricks.com/aws/en/lakehouse/medallion.html)

### 비유로 이해하기

식재료가 요리가 되는 과정을 떠올려 보겠습니다.

| 단계 | 비유 | 데이터 |
|------|------|--------|
| **Bronze** | 시장에서 사온 식재료 그대로 (흙, 겉껍질 포함) | 원본 데이터 그대로 (오류, 중복 포함) |
| **Silver** | 씻고, 껍질 벗기고, 손질한 재료 | 정제, 중복 제거, 스키마 표준화된 데이터 |
| **Gold** | 완성된 요리 (스테이크, 샐러드) | 비즈니스 목적에 맞게 집계·요약된 데이터 |

---

## 각 계층의 역할

### 🥉 Bronze Layer (원본 데이터 계층)

| 항목 | 설명 |
|------|------|
| **목적** | 소스 데이터를 **있는 그대로** 보존합니다 |
| **데이터 상태** | 원본 그대로 (오류, 중복, 불완전한 데이터 포함 가능) |
| **저장 방식** | 소스의 스키마를 최대한 유지하되, 메타데이터(수집 시간, 소스 파일명 등)를 추가합니다 |
| **주요 도구** | Auto Loader, Lakeflow Connect |
| **핵심 원칙** | "아무것도 변경하지 않는다. 원본을 그대로 보존한다" |

```sql
-- Bronze 테이블 예시: 원본 JSON 데이터를 그대로 수집
CREATE OR REFRESH STREAMING TABLE bronze_orders
AS SELECT
    *,
    _metadata.file_path AS _source_file,
    _metadata.file_modification_time AS _ingested_at,
    current_timestamp() AS _processing_time
FROM STREAM read_files(
    's3://raw-data/orders/',
    format => 'json',
    inferColumnTypes => true
);
```

> 💡 **왜 원본을 보존해야 하나요?** 나중에 변환 로직에 오류가 발견되더라도, 원본이 있으면 처음부터 다시 정제할 수 있습니다. 원본 없이 변환된 데이터만 있으면, 소스 시스템에 다시 요청해야 하는데 이미 삭제되었거나 변경되었을 수 있습니다.

### 🥈 Silver Layer (정제된 데이터 계층)

| 항목 | 설명 |
|------|------|
| **목적** | Bronze 데이터를 **정제하고 표준화**하여 분석 가능한 형태로 만듭니다 |
| **데이터 상태** | 중복 제거, 타입 변환, 유효성 검증이 완료된 깨끗한 데이터입니다 |
| **저장 방식** | 정규화된 스키마, 적절한 데이터 타입, 일관된 명명 규칙을 적용합니다 |
| **주요 작업** | 데이터 정제, 중복 제거, 조인, 스키마 표준화 |
| **핵심 원칙** | "기업 전체에서 공통으로 사용할 수 있는 깨끗한 데이터를 만든다" |

```sql
-- Silver 테이블 예시: 정제 및 표준화
CREATE OR REFRESH STREAMING TABLE silver_orders (
    -- 데이터 품질 규칙 정의
    CONSTRAINT valid_order_id EXPECT (order_id IS NOT NULL) ON VIOLATION DROP ROW,
    CONSTRAINT valid_amount EXPECT (total_amount > 0) ON VIOLATION DROP ROW,
    CONSTRAINT valid_email EXPECT (email RLIKE '^[^@]+@[^@]+\\.[^@]+$') ON VIOLATION DROP ROW
)
AS SELECT
    CAST(order_id AS BIGINT) AS order_id,
    CAST(customer_id AS BIGINT) AS customer_id,
    TRIM(LOWER(email)) AS email,
    CAST(order_date AS TIMESTAMP) AS order_date,
    CAST(total_amount AS DECIMAL(12, 2)) AS total_amount,
    UPPER(TRIM(status)) AS status,
    TRIM(shipping_city) AS shipping_city
FROM STREAM(bronze_orders)
WHERE order_id IS NOT NULL;
```

> 💡 **정규화(Normalization)란?** 데이터를 중복 없이 효율적으로 저장하기 위해 테이블을 분리하고 관계를 정의하는 과정입니다. 예를 들어, 주문 테이블에 고객 이름을 직접 넣는 대신, 고객 ID만 저장하고 고객 테이블을 별도로 만드는 것입니다. 이렇게 하면 고객 이름이 변경되어도 한 곳만 수정하면 됩니다.

### 🥇 Gold Layer (비즈니스 집계 계층)

| 항목 | 설명 |
|------|------|
| **목적** | 특정 비즈니스 요구에 맞게 **집계, 요약, 결합**된 최종 데이터를 제공합니다 |
| **데이터 상태** | 비즈니스 메트릭, KPI, 리포트용 데이터입니다 |
| **저장 방식** | 비정규화(Denormalized)된 넓은 테이블, 사전 집계된 요약 테이블입니다 |
| **주요 소비자** | BI 대시보드, 경영진 리포트, ML 피처 테이블 |
| **핵심 원칙** | "특정 비즈니스 질문에 바로 답할 수 있는 데이터를 만든다" |

```sql
-- Gold 테이블 예시 1: 일별 매출 요약
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_revenue
AS SELECT
    DATE(order_date) AS order_date,
    shipping_city AS city,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM silver_orders
WHERE status = 'COMPLETED'
GROUP BY DATE(order_date), shipping_city;

-- Gold 테이블 예시 2: 고객 360 뷰
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_360
AS SELECT
    c.customer_id,
    c.name,
    c.email,
    c.signup_date,
    COUNT(o.order_id) AS lifetime_orders,
    SUM(o.total_amount) AS lifetime_revenue,
    AVG(o.total_amount) AS avg_order_value,
    MAX(o.order_date) AS last_order_date,
    DATEDIFF(CURRENT_DATE(), MAX(o.order_date)) AS days_since_last_order
FROM silver_customers c
LEFT JOIN silver_orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.email, c.signup_date;
```

> 💡 **비정규화(Denormalization)란?** 정규화의 반대 개념입니다. 분석 성능을 위해 의도적으로 데이터를 중복 저장합니다. 예를 들어, 고객 360 뷰에서는 고객 정보와 주문 집계를 하나의 넓은 테이블에 미리 합쳐 놓아, 조인 없이 빠르게 조회할 수 있습니다.

---

## 전체 흐름 다이어그램

```mermaid
graph TB
    subgraph Sources["📦 데이터 소스"]
        S1["주문 DB"]
        S2["고객 DB"]
        S3["웹 로그 (JSON)"]
    end

    subgraph Bronze["🥉 Bronze Layer"]
        B1["bronze_orders<br/>(원본 주문 데이터)"]
        B2["bronze_customers<br/>(원본 고객 데이터)"]
        B3["bronze_clickstream<br/>(원본 클릭 로그)"]
    end

    subgraph Silver["🥈 Silver Layer"]
        Si1["silver_orders<br/>(정제된 주문)"]
        Si2["silver_customers<br/>(정제된 고객)"]
        Si3["silver_page_views<br/>(정제된 페이지뷰)"]
    end

    subgraph Gold["🥇 Gold Layer"]
        G1["gold_daily_revenue<br/>(일별 매출)"]
        G2["gold_customer_360<br/>(고객 전체 뷰)"]
        G3["gold_product_performance<br/>(상품 성과)"]
    end

    subgraph Consume["📊 소비"]
        C1["경영진 대시보드"]
        C2["마케팅 분석"]
        C3["ML 추천 모델"]
    end

    S1 --> B1
    S2 --> B2
    S3 --> B3

    B1 --> Si1
    B2 --> Si2
    B3 --> Si3

    Si1 --> G1
    Si1 --> G2
    Si2 --> G2
    Si1 --> G3
    Si3 --> G3

    G1 --> C1
    G2 --> C2
    G3 --> C3
```

---

## 왜 Medallion 아키텍처를 사용해야 하나요?

"데이터를 세 벌이나 저장하면 비용만 올라가는 거 아닌가요?" — 이것은 Medallion 아키텍처를 처음 접하는 분들이 가장 많이 하시는 질문입니다. 결론부터 말씀드리면, **세 벌의 데이터가 오히려 비용을 절감하고 생산성을 높입니다.**

### 질문 1: "데이터가 중복되는 거 아닌가요?"

엄밀히 말하면 **물리적으로 여러 벌의 데이터가 존재하는 것은 맞습니다.** 하지만 이것은 "중복"이 아니라 **"가공 단계가 다른 데이터"** 입니다.

| | 단일 테이블 방식 | Medallion 방식 |
|---|---|---|
| **비유** | 식재료를 사서 바로 손님 접시에 올리는 것 | 식재료 → 손질 → 요리 → 플레이팅 |
| **문제 발생 시** | 원본이 없어서 처음부터 다시 소스 시스템에 요청 | Bronze(원본)가 있으므로 Silver/Gold만 다시 처리 |
| **변환 로직 수정 시** | 전체 파이프라인을 다시 설계해야 함 | Silver 로직만 수정하면 Gold에 자동 반영 |

> 💡 **핵심 포인트**: Bronze는 "보험"입니다. 원본 데이터가 없으면 실수를 되돌릴 수 없습니다. 소스 시스템은 데이터를 영구 보관하지 않는 경우가 많고(CDC 로그 만료, API 변경 등), 한번 놓치면 다시 가져올 수 없습니다.

### 질문 2: "스토리지 비용이 올라가지 않나요?"

**스토리지 비용은 컴퓨트 비용의 1/10~1/100 수준**입니다. 데이터를 여러 벌 저장하는 비용보다, 매번 원본에서 복잡한 변환을 재실행하는 컴퓨트 비용이 훨씬 큽니다.

| 비용 항목 | Medallion 없이 | Medallion 사용 |
|----------|---------------|--------------|
| **스토리지** | 1벌 × 100GB = $2.3/월 | 3벌 × 100GB = $6.9/월 (+$4.6) |
| **컴퓨트 (매 쿼리)** | 원본에서 매번 정제+집계 = 매 쿼리 $0.50 | Gold에서 바로 읽기 = 매 쿼리 $0.02 |
| **일 100회 쿼리 시** | $50/일 = **$1,500/월** | $2/일 = **$60/월** |
| **월 총 비용** | **$1,502.3/월** | **$66.9/월** (96% 절감) |

> 💡 **"데이터를 미리 가공해두면 쿼리할 때 빠르고 싸다"** — 이것이 Medallion의 경제적 핵심입니다. 스토리지는 싸지만 컴퓨트는 비쌉니다. Gold 계층의 사전 집계 덕분에 대시보드 쿼리가 수십 배 빨라지고, 컴퓨트 비용도 획기적으로 줄어듭니다.

### 질문 3: "왜 한 번에 최종 테이블로 만들면 안 되나요?"

한 번에 원본 → 최종 테이블로 변환하는 것은 가능하지만, **실무에서는 반드시 문제가 발생합니다:**

**시나리오: 비즈니스 로직 변경**
> 마케팅 팀: "이번 달부터 '활성 고객'의 기준을 '30일 내 구매'에서 '60일 내 구매'로 변경해 주세요."

| 단일 파이프라인 | Medallion 파이프라인 |
|---|---|
| ETL 전체를 수정하고 소스 DB에서 처음부터 다시 추출 | Silver는 그대로, Gold의 집계 쿼리만 `30 DAYS` → `60 DAYS`로 수정 |
| 소요시간: 수 시간 ~ 수 일 | 소요시간: 수 분 |
| 소스 DB에 부하를 다시 발생시킴 | 소스 DB에 영향 없음 |

**시나리오: 데이터 품질 이슈 발견**
> 분석가: "지난주 매출이 갑자기 2배로 뛴 건 주문 데이터가 중복 수집된 거 같아요."

| 단일 파이프라인 | Medallion 파이프라인 |
|---|---|
| 원본이 없어서 소스 시스템에 재확인 요청 | Bronze에서 원본 확인 → Silver의 중복 제거 로직 수정 → Gold 자동 갱신 |
| 원인 파악: 수 일 | 원인 파악: 수 분 (Bronze 원본 대조) |

**시나리오: 새로운 분석 요구**
> 경영진: "고객 이탈 예측 모델을 만들고 싶은데, 과거 2년간 고객 행동 데이터가 필요합니다."

| 단일 파이프라인 | Medallion 파이프라인 |
|---|---|
| 새로운 ETL을 처음부터 설계하고, 소스 DB에서 2년치 데이터를 다시 추출해야 함 | Silver에 이미 정제된 2년치 데이터가 있으므로, 새로운 Gold 테이블만 추가하면 끝 |

### 질문 4: "그래서 실제로 어떤 이점이 있나요?" (정리)

| 이점 | 설명 |
|------|------|
| **🔄 재처리 가능성** | 원본(Bronze)이 있으므로 언제든 변환 로직을 수정하고 다시 처리할 수 있습니다 |
| **💰 비용 절감** | Gold의 사전 집계로 쿼리 비용이 10~100배 감소합니다. 스토리지 추가 비용은 미미합니다 |
| **⚡ 성능 향상** | 대시보드/리포트가 Gold에서 읽으므로 응답 시간이 초 단위로 줄어듭니다 |
| **🛡️ 데이터 품질 보장** | Silver의 Expectations가 품질을 자동 검증합니다. 문제가 있는 데이터는 Gold까지 올라가지 않습니다 |
| **👥 역할 분리** | 데이터 엔지니어는 Bronze/Silver를, 분석가는 Gold를 사용합니다. 서로의 작업에 영향을 주지 않습니다 |
| **📊 다목적 활용** | 하나의 Silver에서 여러 개의 Gold를 만들 수 있습니다. 같은 데이터를 BI, ML, 리포트에서 각각 활용합니다 |
| **🔍 디버깅 용이** | 문제 발생 시 각 계층을 비교하여 어디에서 문제가 생겼는지 빠르게 파악할 수 있습니다 |
| **📜 감사 추적** | Bronze의 원본 보존으로 규제 감사 요건(데이터 보존, 변경 이력)을 충족합니다 |

### Medallion을 사용하지 않아도 되는 경우

모든 상황에서 3계층이 필요한 것은 아닙니다.

| 상황 | 권장 |
|------|------|
| 단순 PoC/프로토타입 | Bronze → Gold (2계층)만으로 충분합니다 |
| 데이터 소스가 이미 깨끗한 경우 | Silver를 간소화하거나 생략할 수 있습니다 |
| 일회성 분석 | 임시 뷰나 노트북으로 처리해도 됩니다 |
| 실시간 스트리밍 (저지연 필수) | 2계층 또는 단일 파이프라인이 적합할 수 있습니다 |

> 💡 **핵심 원칙**: "프로덕션 데이터 파이프라인에는 Medallion을, 탐색적 분석에는 간소화된 구조를" — 두 접근을 상황에 따라 유연하게 사용하세요.

---

## 계층 간 비교

| 비교 항목 | Bronze | Silver | Gold |
|-----------|--------|--------|------|
| **데이터 품질** | 낮음 (원본 그대로) | 높음 (정제됨) | 최고 (비즈니스 검증) |
| **스키마** | 소스와 동일 | 표준화됨 | 비즈니스 도메인 기준 |
| **갱신 빈도** | 실시간 ~ 분 단위 | 분 ~ 시간 단위 | 시간 ~ 일 단위 |
| **주요 소비자** | 데이터 엔지니어 | 데이터 분석가, DS | BI, 경영진, 앱 |
| **보존 기간** | 장기 (원본 보존) | 중기 | 단기 ~ 중기 |
| **테이블 수** | 소스 수와 비례 | Bronze와 유사 | 비즈니스 요구에 따라 |

---

## 모범 사례

### 네이밍 컨벤션

일관된 이름 규칙을 사용하면 테이블의 계층을 쉽게 식별할 수 있습니다.

| 방식 | 예시 |
|------|------|
| **접두어 방식** | `bronze_orders`, `silver_orders`, `gold_daily_revenue` |
| **스키마 분리 방식** | `raw.orders`, `cleaned.orders`, `analytics.daily_revenue` |
| **카탈로그 분리 방식** | `bronze.ecommerce.orders`, `silver.ecommerce.orders`, `gold.ecommerce.daily_revenue` |

Databricks에서는 **Unity Catalog의 3-Level 네임스페이스(Catalog → Schema → Table)** 를 활용하여 계층을 체계적으로 관리하는 것을 권장합니다.

### 권한 관리

| 계층 | 접근 권한 |
|------|-----------|
| **Bronze** | 데이터 엔지니어만 쓰기/읽기 가능합니다 |
| **Silver** | 데이터 엔지니어가 쓰기, 분석가/과학자가 읽기 가능합니다 |
| **Gold** | 넓은 읽기 권한 (BI 사용자, 경영진 포함) |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Bronze** | 원본 데이터를 그대로 보존하는 첫 번째 계층입니다. "있는 그대로 저장"합니다 |
| **Silver** | 정제, 표준화, 중복 제거가 완료된 두 번째 계층입니다. "깨끗하게 정리"합니다 |
| **Gold** | 비즈니스 목적에 맞게 집계된 세 번째 계층입니다. "바로 분석 가능"합니다 |
| **Medallion** | Bronze → Silver → Gold 3계층 구조의 데이터 설계 패턴입니다 |

다음 문서에서는 Delta Lake의 **실전 조작법**(MERGE, OPTIMIZE, VACUUM 등)을 자세히 살펴보겠습니다.

---

## 참고 링크

- [Databricks: Medallion Architecture](https://docs.databricks.com/aws/en/lakehouse/medallion.html)
- [Azure Databricks: Medallion lakehouse architecture](https://learn.microsoft.com/en-us/azure/databricks/lakehouse/medallion)
- [Databricks Blog: Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
