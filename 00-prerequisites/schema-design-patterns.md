# 데이터 모델링 — 정규화, Star 스키마, Snowflake 스키마

## 왜 데이터 모델링을 알아야 하나요?

데이터베이스에 데이터를 저장할 때, "테이블을 어떻게 설계하느냐"에 따라 저장 효율, 쿼리 성능, 유지보수 편의성이 크게 달라집니다. 이 설계 과정을 **데이터 모델링(Data Modeling)** 이라고 합니다.

현업에서 20년 가까이 데이터를 다루면서 가장 많이 본 문제는, 모델링을 대충 하고 시작해서 6개월 후에 전체 파이프라인을 뜯어고치는 상황입니다. 모델링은 "나중에 하면 되지"가 통하지 않습니다. 데이터가 쌓이기 시작하면 구조를 바꾸는 비용은 기하급수적으로 늘어납니다.

데이터 모델링은 크게 두 가지 방향이 있습니다.

| 방향 | 목적 | 적합한 시스템 |
|------|------|-------------|
| **정규화 (Normalization)** | 중복을 제거하고 무결성을 보장합니다 | OLTP (운영 시스템) |
| **비정규화 (Denormalization)** | 분석 성능을 극대화합니다 | OLAP (분석 시스템, 웨어하우스) |

---

## 정규화 (Normalization)

### 개념

> 💡 **정규화(Normalization)** 란 데이터의 **중복을 최소화**하기 위해 테이블을 분리하고, 관계를 통해 연결하는 설계 방법입니다. 데이터의 일관성과 무결성을 보장하는 것이 목표입니다.

### 비유: 고객 정보가 100군데에 흩어져 있다면?

실무에서 정규화가 왜 중요한지, 실제 장애 사례로 설명하겠습니다. 한 이커머스 회사에서 고객 주소를 주문 테이블에 직접 넣어두었습니다. 어느 날 고객이 주소를 변경했는데, 과거 주문 300건의 주소는 그대로였습니다. 배송팀에서 "이 주소가 맞나요?"라는 CS가 폭주했고, 결국 수동으로 데이터를 수정하는 데 이틀이 걸렸습니다.

**정규화하지 않은 경우 (중복 가득):**

| 주문번호 | 고객명 | 고객전화 | 고객주소 | 상품명 | 가격 |
|----------|--------|----------|----------|--------|------|
| 001 | 김철수 | 010-1234 | 서울시 강남구 | 노트북 | 120만 |
| 002 | 김철수 | 010-1234 | 서울시 강남구 | 키보드 | 8.9만 |
| 003 | 이영희 | 010-5678 | 부산시 해운대구 | 마우스 | 6.5만 |

❌ **문제점**: 김철수의 전화번호가 변경되면, 해당 고객의 모든 주문 행을 수정해야 합니다. 주문이 10만 건이면 10만 건을 업데이트해야 합니다. 하나라도 놓치면 데이터 불일치가 발생하고, 이런 불일치는 보통 3개월 후에야 발견됩니다.

**정규화한 경우 (중복 제거):**

**고객 테이블:**

| customer_id | name | phone | address |
|------------|------|-------|---------|
| C1 | 김철수 | 010-1234 | 서울시 강남구 |
| C2 | 이영희 | 010-5678 | 부산시 해운대구 |

**주문 테이블:**

| order_id | customer_id | product | price |
|----------|------------|---------|-------|
| 001 | C1 | 노트북 | 120만 |
| 002 | C1 | 키보드 | 8.9만 |
| 003 | C2 | 마우스 | 6.5만 |

✅ 김철수의 전화번호를 고객 테이블에서 **한 번만** 수정하면, 모든 주문에서 올바른 번호가 조회됩니다.

### 정규화 단계

| 단계 | 이름 | 규칙 | 실무에서의 의미 |
|------|------|------|----------------|
| **1NF** | 제1정규형 | 원자값만 저장 | "서울, 부산" 같은 쉼표 구분 값을 넣지 마세요. 나중에 `WHERE city = '서울'` 쿼리가 제대로 안 됩니다 |
| **2NF** | 제2정규형 | 부분 종속 제거 | 복합 키를 쓸 때, 키의 일부에만 의존하는 컬럼을 분리합니다 |
| **3NF** | 제3정규형 | 이행 종속 제거 | "우편번호 → 도시" 같은 간접 종속을 별도 테이블로 뺍니다 |

> 💡 실무에서는 보통 **3NF(제3정규형)** 까지 적용하면 충분합니다. BCNF, 4NF, 5NF까지 가는 프로젝트는 20년 동안 손에 꼽을 정도로 드뭅니다. 과도한 정규화는 오히려 개발 생산성을 떨어뜨리고, JOIN이 너무 많아져서 쿼리 성능 문제를 일으킵니다.

### 정규화의 장단점

| 장점 | 단점 |
|------|------|
| 데이터 중복이 없어 저장 공간 절약 | JOIN이 많아져 분석 쿼리가 느려질 수 있습니다 |
| 데이터 수정 시 한 곳만 변경하면 됩니다 | 테이블이 많아져 신규 개발자가 구조 파악에 시간이 걸립니다 |
| 데이터 일관성이 보장됩니다 | 대규모 분석 쿼리에서 7~8개 테이블을 JOIN하면 성능이 급격히 떨어집니다 |

---

## 비정규화 — 분석을 위한 설계

### OLTP vs OLAP 설계의 차이

> 💡 **OLTP(Online Transaction Processing)** 와 **OLAP(Online Analytical Processing)** 은 서로 다른 목적을 가진 시스템이며, 데이터 모델링 방식도 완전히 달라야 합니다. 가장 흔한 실수는 OLTP용으로 설계한 정규화 테이블을 그대로 분석에 쓰는 것입니다. 단순한 "이번 달 매출 합계" 쿼리도 5개 테이블을 JOIN해야 하고, 30초 이상 걸리게 됩니다.

| 비교 | OLTP (운영) | OLAP (분석) |
|------|------------|------------|
| **목적** | 주문 처리, 결제, 회원가입 등 개별 트랜잭션 | 매출 집계, 추이 분석, 리포트 등 대량 조회 |
| **쿼리 패턴** | 소량의 행을 빠르게 읽기/쓰기 (한 번에 1~10건) | 수백만~수억 건을 집계 (SUM, AVG, COUNT) |
| **설계 원칙** | 정규화 (중복 제거) | 비정규화 (조인 최소화) |
| **대표 도구** | MySQL, PostgreSQL, Oracle | Databricks SQL, Snowflake, BigQuery |

### 차원 모델링 (Dimensional Modeling)

분석 시스템(OLAP)을 위한 대표적인 설계 방법론이 **차원 모델링**입니다. 1996년 Ralph Kimball이 체계화한 이 방법론은 30년이 지난 지금도 분석 모델링의 사실상 표준(de facto standard)입니다. 데이터를 **팩트(Fact)** 와 **디멘전(Dimension)** 으로 나누어 설계합니다.

> 💡 **팩트 테이블(Fact Table)** 이란 비즈니스에서 발생하는 **이벤트(사건)** 와 그 **측정값(숫자)** 을 저장하는 테이블입니다. 주문, 클릭, 결제, 재고 이동 등의 이벤트가 해당합니다. 보통 테이블 중 가장 행 수가 많으며 — 대기업이라면 수십억 건이 기본입니다.

> 💡 **디멘전 테이블(Dimension Table)** 이란 팩트 이벤트의 **맥락(누가, 언제, 어디서, 무엇을)** 을 설명하는 참조 테이블입니다. 고객, 상품, 날짜, 지역 등이 해당합니다. 팩트 대비 행 수가 적지만(수천~수백만 건), 컬럼 수가 많고 텍스트 데이터가 풍부합니다.

---

## Star 스키마 (Star Schema)

### 개념

> 💡 **Star 스키마(별 모양 스키마)** 는 중앙에 팩트 테이블을 두고, 주위에 디멘전 테이블을 **직접 연결**하는 가장 기본적인 차원 모델링 패턴입니다. 다이어그램이 별 모양처럼 보여서 이런 이름이 붙었습니다.

결론부터 말씀드리면, **99%의 경우 Star 스키마가 정답**입니다. Snowflake 스키마를 선택해야 하는 경우는 뒤에서 설명하겠지만, 매우 드뭅니다.

### 핵심 디멘전 설계 — dim_date는 반드시 만드세요

Star 스키마를 설계할 때 가장 흔한 실수는 **날짜 디멘전을 DATE 타입 컬럼 하나로 처리**하는 것입니다. "그냥 `WHERE YEAR(order_date) = 2025` 쓰면 되지 않나요?"라고 생각하실 수 있습니다. 하지만 실무에서는 이게 6개월 뒤 재앙이 됩니다.

```sql
-- 실무에서 필요한 dim_date 테이블 예시
CREATE TABLE dim_date (
    date_key      INT PRIMARY KEY,        -- 20250301 형식
    full_date     DATE NOT NULL,
    year          INT,
    quarter       STRING,                  -- 'Q1', 'Q2', ...
    month         INT,
    month_name    STRING,                  -- '3월', '4월', ...
    week_of_year  INT,
    day_of_week   STRING,                  -- '월요일', '화요일', ...
    day_of_week_num INT,                   -- 1(월) ~ 7(일)
    is_weekend    BOOLEAN,
    is_holiday    BOOLEAN,                 -- 공휴일 여부
    holiday_name  STRING,                  -- '설날', '추석', ...
    is_business_day BOOLEAN,              -- 영업일 여부
    fiscal_year   INT,                     -- 회계연도 (3월 결산이면 다를 수 있음)
    fiscal_quarter STRING,
    year_month    STRING                   -- '2025-03' 형식, 리포트용
);
```

**왜 이게 중요한지 구체적으로 설명하겠습니다:**

1. **공휴일/영업일 계산**: "영업일 기준 5일 이내 배송" 같은 비즈니스 룰을 쿼리할 때, `WHERE d.is_business_day = true`로 바로 필터링할 수 있습니다. 이걸 매번 함수로 계산하면 쿼리당 3~5초가 추가됩니다.
2. **회계연도**: 한국 기업 중 12월 결산이 아닌 곳이 많습니다 (삼성 등 대기업은 12월이지만, 많은 기업이 3월 결산). `fiscal_year`와 `fiscal_quarter`를 미리 계산해 두면 재무 리포트가 훨씬 간단해집니다.
3. **대시보드 성능**: 수백 개의 대시보드에서 `WHERE YEAR(order_date) = 2025 AND MONTH(order_date) = 3`을 반복하면, 매번 함수 연산이 발생합니다. `WHERE d.year = 2025 AND d.month = 3`으로 조인하면 사전 계산된 값을 읽기만 하므로 훨씬 빠릅니다.

> ⚠️ dim_date 없이 시작한 프로젝트가 6개월 후 모든 대시보드를 리팩토링하는 경우를 수없이 봤습니다. dim_date 테이블은 보통 10~30년 치를 미리 생성해 두며 (약 1만~1.1만 행), 한 번 만들면 거의 변경할 일이 없습니다. 만드는 데 30분이면 충분하고, 안 만들면 나중에 30일을 고생합니다.

### Degenerate Dimension (퇴화 디멘전)

모든 속성을 디멘전 테이블로 빼야 하는 것은 아닙니다. **주문번호, 송장번호, 영수증 번호** 같은 값은 팩트 테이블의 각 행을 식별하는 데 사용되지만, 별도의 디멘전 테이블로 뺄 의미가 없습니다. 이런 값을 **Degenerate Dimension(퇴화 디멘전)** 이라고 합니다.

```sql
-- 주문번호(order_number)는 팩트 테이블에 그대로 둡니다
-- 별도의 dim_order를 만들 필요가 없습니다
CREATE TABLE fact_orders (
    order_key       BIGINT,
    order_number    STRING,     -- ← Degenerate Dimension
    invoice_number  STRING,     -- ← Degenerate Dimension
    customer_key    INT,
    product_key     INT,
    date_key        INT,
    quantity        INT,
    amount          DECIMAL(15,2)
);
```

처음 모델링하는 분들이 "모든 걸 디멘전으로 빼야 한다"고 오해해서, `dim_order` 같은 불필요한 테이블을 만드는 경우가 있습니다. 주문번호에 대한 추가 속성(상태, 유형 등)이 없다면, 팩트에 놔두는 것이 맞습니다.

### Junk Dimension (정크 디멘전)

팩트 테이블에 `is_returned`, `is_gift_wrapped`, `is_priority`, `payment_type` 같은 **플래그/코드 컬럼**이 5~6개 이상 쌓이기 시작하면, 이것들을 하나의 **Junk Dimension** 으로 묶는 것이 좋습니다.

```sql
-- Before: 팩트 테이블이 플래그로 비대해짐
-- fact_orders에 is_returned, is_gift, is_priority, payment_type, channel ...

-- After: Junk Dimension으로 조합을 미리 만들어 둡니다
CREATE TABLE dim_order_flags (
    flag_key        INT PRIMARY KEY,
    is_returned     BOOLEAN,
    is_gift         BOOLEAN,
    is_priority     BOOLEAN,
    payment_type    STRING,     -- 'card', 'cash', 'transfer'
    channel         STRING      -- 'web', 'app', 'store'
);

-- 팩트에는 flag_key 하나만 남깁니다
-- fact_orders.flag_key → dim_order_flags.flag_key
```

이렇게 하면 팩트 테이블의 컬럼 수가 줄어들고, "신용카드 + 웹 + 일반배송" 같은 조합별 분석도 깔끔하게 할 수 있습니다.

### Star 스키마 — 전체 예시

| 유형 | 테이블 | 주요 컬럼 |
|------|--------|----------|
| **팩트 테이블** | fact_orders | order_key, order_number, customer_key, product_key, date_key, store_key, flag_key, quantity, amount |
| **디멘전 테이블** | dim_customer | customer_key, name, email, city, tier |
|  | dim_product | product_key, name, category, brand, price |
|  | dim_date | date_key, full_date, month, quarter, year, is_business_day, fiscal_year |
|  | dim_store | store_key, name, address, region |
|  | dim_order_flags | flag_key, is_returned, is_gift, payment_type, channel |

### 분석 쿼리 예시

```sql
-- "2025년 Q1, 영업일 기준, 서울 지역의 카테고리별 매출"
-- dim_date가 있으니 이렇게 깔끔하게 쿼리할 수 있습니다
SELECT
    p.category,
    d.fiscal_quarter,
    SUM(f.amount) AS total_revenue,
    COUNT(f.order_key) AS order_count,
    AVG(f.amount) AS avg_order_value
FROM fact_orders f
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_store s ON f.store_key = s.store_key
WHERE d.fiscal_year = 2025
  AND d.fiscal_quarter = 'Q1'
  AND d.is_business_day = true    -- 영업일만!
  AND s.region = '서울'
GROUP BY p.category, d.fiscal_quarter
ORDER BY total_revenue DESC;
```

---

## Slowly Changing Dimension (SCD) — 디멘전 변경 처리

현업에서 "고객이 서울에서 부산으로 이사했을 때, 과거 주문은 서울로 봐야 하나, 부산으로 봐야 하나?"라는 질문이 반드시 나옵니다. 이것을 **Slowly Changing Dimension(SCD)** 이라고 합니다.

| SCD 유형 | 동작 방식 | 언제 쓰나요 |
|----------|----------|------------|
| **Type 1** | 새 값으로 **덮어쓰기** (이력 없음) | 오타 수정, 이력이 중요하지 않을 때 |
| **Type 2** | 새 행 추가, 기존 행에 종료일 기록 | 고객 등급 변경, 조직 개편 등 이력 추적이 필수일 때 |
| **Type 3** | 이전 값과 현재 값을 별도 컬럼으로 저장 | 직전 값만 알면 될 때 (거의 안 씀) |

### Type 2가 정말 필요한 경우 vs 오버엔지니어링

솔직히 말씀드리면, **Type 2를 구현해야 하는 경우는 생각보다 적습니다.** 많은 팀이 "혹시 필요할까 봐" Type 2를 모든 디멘전에 적용하는데, 이는 명백한 오버엔지니어링입니다.

**Type 2가 진짜 필요한 경우:**
- 고객 등급(VIP → Gold)이 바뀌었을 때, "VIP일 때의 매출"과 "Gold일 때의 매출"을 구분해서 봐야 하는 경우
- 영업 조직이 개편되었을 때, "개편 전 A팀 소속일 때 실적"을 따로 봐야 하는 경우
- 규정/감사 요건으로 "당시 시점의 정보"를 보존해야 하는 경우

**Type 1로 충분한 경우 (=Type 2가 오버엔지니어링):**
- 고객 전화번호, 이메일 변경 — 이전 전화번호로 분석할 일이 없습니다
- 상품명 오타 수정
- 주소 변경인데 지역별 매출 분석이 불필요한 경우

```sql
-- Type 2 SCD 예시: 고객 등급 변경 이력
-- customer_key는 Surrogate Key (자동 생성), customer_id는 Natural Key (비즈니스 키)
| customer_key | customer_id | name   | tier   | effective_date | end_date   | is_current |
|-------------|-------------|--------|--------|----------------|------------|------------|
| 1001        | C1          | 김철수 | Silver | 2024-01-01     | 2025-02-28 | false      |
| 1002        | C1          | 김철수 | Gold   | 2025-03-01     | 9999-12-31 | true       |
```

> ⚠️ Type 2를 적용하면 디멘전 테이블의 행 수가 빠르게 증가하고, JOIN 조건에 날짜 범위가 추가되어 쿼리 복잡도가 올라갑니다. "이 디멘전에 Type 2가 정말 필요한가?"를 반드시 비즈니스 담당자와 확인하세요.

---

## Snowflake 스키마 (Snowflake Schema)

### 개념

> 💡 **Snowflake 스키마(눈꽃 모양 스키마)** 는 Star 스키마의 디멘전 테이블을 더 세분화(정규화)하여, 디멘전이 **하위 디멘전을 참조**하는 구조입니다.

### 언제 Snowflake를 쓰나요? (거의 안 씁니다)

솔직히 말씀드리면, **현대 데이터 플랫폼에서 Snowflake 스키마를 선택하는 경우는 매우 드뭅니다.** Star 스키마의 디멘전 중복으로 인한 저장 비용 증가가 문제가 되던 시절(스토리지가 TB당 수천 달러이던 2000년대)에는 의미가 있었습니다. 하지만 지금은 S3 스토리지가 TB당 월 23달러입니다.

Snowflake 스키마를 고려해야 하는 유일한 상황:
- 디멘전 테이블 하나가 **수천만 건 이상**이고, 그 안에 반복되는 서브 엔티티(예: 상품 → 카테고리 → 부서)의 텍스트 데이터가 매우 크면서, 그 서브 엔티티를 독립적으로 관리해야 할 때

### Star vs Snowflake 비교

| 비교 항목 | Star 스키마 | Snowflake 스키마 |
|-----------|-----------|-----------------|
| 디멘전 구조 | 하나의 넓은 테이블 | 여러 정규화된 테이블 |
| JOIN 수 | 적음 (팩트 ↔ 디멘전만) | 많음 (디멘전 ↔ 하위 디멘전도) |
| 쿼리 성능 | 빠름 | 상대적으로 느림 (JOIN 증가) |
| 쿼리 작성 난이도 | 쉬움 | 복잡 (BI 분석가가 싫어합니다) |
| 저장 공간 | 디멘전 중복 있음 | 중복 최소화 |
| **실무 권장** | ✅ 거의 모든 경우 | 매우 특수한 경우에만 |

---

## One Big Table (OBT) 패턴 — 레이크하우스 시대의 비정규화

### 왜 OBT가 떠오르고 있는가

최근 Databricks, Snowflake 같은 현대 플랫폼에서는 모든 디멘전을 팩트에 미리 합쳐 놓은 **하나의 큰 테이블(One Big Table, OBT)** 패턴이 급격히 늘고 있습니다. 이유는 간단합니다:

1. **스토리지가 압도적으로 저렴해졌습니다.** S3에 Parquet/Delta로 저장하면 TB당 월 23달러. 중복 데이터 걱정할 비용이 아닙니다.
2. **컬럼 기반 압축이 뛰어납니다.** 같은 값이 반복되는 디멘전 컬럼(예: city = '서울'이 100만 번)은 압축 시 거의 공간을 차지하지 않습니다.
3. **JOIN이 없으니 쿼리가 빠릅니다.** Star 스키마에서 4개 테이블을 JOIN하면 30초 걸리던 쿼리가, OBT에서는 3초에 끝납니다.
4. **BI 사용자가 SQL을 몰라도 됩니다.** 테이블 하나만 알면 모든 분석이 가능하므로, 셀프 서비스 분석이 훨씬 쉬워집니다.

```sql
-- Gold 계층: 비정규화된 OBT 생성
CREATE OR REPLACE TABLE gold.wide_orders AS
SELECT
    f.order_key,
    f.order_number,
    f.quantity,
    f.amount,
    -- 고객 디멘전
    c.name AS customer_name,
    c.city AS customer_city,
    c.tier AS customer_tier,
    -- 상품 디멘전
    p.name AS product_name,
    p.category AS product_category,
    p.brand AS product_brand,
    -- 날짜 디멘전
    d.full_date AS order_date,
    d.month,
    d.quarter,
    d.year,
    d.is_business_day,
    d.fiscal_year,
    -- 플래그 디멘전
    fl.is_returned,
    fl.payment_type,
    fl.channel
FROM fact_orders f
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_order_flags fl ON f.flag_key = fl.flag_key;
```

### Star 스키마 vs OBT — 언제 뭘 쓰나요

| 상황 | 추천 패턴 | 이유 |
|------|----------|------|
| 데이터 엔지니어가 관리하는 Silver/Gold 계층 | Star 스키마 | 유지보수성, 데이터 일관성이 중요 |
| BI 대시보드의 소스 테이블 | OBT | 분석가가 JOIN 없이 바로 사용 |
| ML Feature Store | OBT | 모델 학습에 필요한 피처를 한 테이블에 |
| Ad-hoc 분석이 많은 팀 | OBT | 셀프 서비스 분석 효율성 |

> 💡 **현업에서는 Star와 OBT를 함께 씁니다.** Silver 계층에서 정규화된 엔티티 테이블을 관리하고, Gold 계층에서 용도별 OBT를 생성하는 패턴이 가장 일반적입니다. "Star냐 OBT냐"의 양자택일이 아니라, **계층별로 다른 패턴을 적용**하는 것이 정답입니다.

---

## Databricks 레이크하우스에서의 모델링 실전 팁

### Medallion 아키텍처와 모델링의 결합

| 계층 | 모델링 방식 | 설명 |
|------|-----------|------|
| **Bronze** | 원본 그대로 | 모델링 없이 원본을 보존합니다. 스키마가 바뀌어도 괜찮습니다 |
| **Silver** | 3NF에 가까운 정규화 | 정제된 엔티티별 테이블 (고객, 주문, 상품 등). 여기가 "Single Source of Truth"입니다 |
| **Gold** | Star 스키마 또는 OBT | 비즈니스 분석용 팩트+디멘전 또는 용도별 비정규화 테이블 |

### 파티셔닝? Liquid Clustering을 쓰세요

전통적인 데이터 웨어하우스에서는 날짜나 지역 기준으로 **파티셔닝(Partitioning)** 을 해야 쿼리 성능이 나왔습니다. Databricks에서도 예전에는 `PARTITIONED BY (date)` 같은 설정이 필수였습니다.

하지만 이제는 **Liquid Clustering**을 사용하세요. 파티셔닝의 모든 장점을 가지면서, 파티션 키를 나중에 바꿀 수도 있고, 데이터 분포가 변해도 자동으로 최적화됩니다.

```sql
-- 전통적 파티셔닝 (이제 권장하지 않음)
-- CREATE TABLE fact_orders (...) PARTITIONED BY (order_date);

-- Liquid Clustering (현재 권장)
CREATE TABLE fact_orders (
    order_key BIGINT,
    order_number STRING,
    customer_key INT,
    product_key INT,
    date_key INT,
    quantity INT,
    amount DECIMAL(15,2)
)
CLUSTER BY (date_key, customer_key);

-- 클러스터링 키는 나중에 변경 가능!
ALTER TABLE fact_orders CLUSTER BY (date_key, product_key);
```

> ⚠️ 파티셔닝으로 시작했다가 "파티션 키를 바꿔야 하는데 테이블을 다시 만들어야 합니다"라는 상황에 처한 팀을 많이 봤습니다. Liquid Clustering은 이 문제를 근본적으로 해결합니다. 신규 테이블은 반드시 Liquid Clustering을 사용하세요.

### Delta Lake의 모델링 영향

Databricks의 Delta Lake를 쓰면, 전통적 RDBMS와 다른 몇 가지 설계 고려사항이 있습니다:

| 전통 RDBMS | Delta Lake on Databricks |
|-----------|------------------------|
| 인덱스 생성 필수 | 인덱스 불필요 (Liquid Clustering + Data Skipping) |
| 파티셔닝 필수 | Liquid Clustering 권장 |
| 정규화로 저장 비용 절약 | 비정규화해도 컬럼 압축으로 비용 미미 |
| SCD Type 2 직접 구현 | Delta Lake의 MERGE + Time Travel로 간소화 |
| 물리적 삭제 | DELETE 후 VACUUM으로 정리 |

---

## 정리

| 핵심 개념 | 실무 핵심 |
|-----------|----------|
| **정규화** | OLTP, Silver 계층에 적합. 3NF까지만 적용하세요 |
| **Star 스키마** | 분석 모델링의 기본. 99%는 Star가 정답입니다 |
| **dim_date** | 반드시 만드세요. 공휴일, 영업일, 회계연도를 포함해야 합니다 |
| **Degenerate Dimension** | 주문번호 같은 식별자는 팩트에 남기세요 |
| **Junk Dimension** | 플래그/코드 컬럼이 5개 이상이면 묶으세요 |
| **SCD Type 2** | 비즈니스가 정말 요구할 때만 적용하세요. 오버엔지니어링 주의 |
| **OBT** | Gold 계층, 대시보드 소스로 적극 활용하세요 |
| **Liquid Clustering** | Databricks에서는 파티셔닝 대신 이것을 쓰세요 |

---

## 참고 링크

- [Databricks: Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering)
- [Databricks: Data modeling best practices](https://docs.databricks.com/aws/en/sql/language-manual/)
- [Kimball Group: Dimensional Modeling Techniques](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/)
- [Databricks Blog: Star Schema vs OBT](https://www.databricks.com/blog)
