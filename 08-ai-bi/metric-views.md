# Metric Views — 비즈니스 시맨틱 레이어

## 왜 비즈니스 시맨틱 레이어가 필요한가요?

데이터 플랫폼이 성숙해질수록 가장 큰 문제는 **"같은 지표인데 팀마다 숫자가 다르다"** 는 것입니다. 이것은 기술적 문제가 아니라 **비즈니스 신뢰의 문제**입니다.

> 💡 **비즈니스 시맨틱 레이어(Business Semantic Layer)** 란, "매출", "활성 사용자", "이탈률" 같은 비즈니스 지표의 **정의(Definition), 계산 로직(Logic), 차원(Dimension)을 중앙에서 한 번 정의**하고, 모든 소비 도구(대시보드, SQL, Genie, 외부 BI)에서 동일하게 사용하는 계층입니다. dbt의 Semantic Layer, Looker의 LookML, AtScale 등이 비슷한 개념이지만, Databricks의 Metric View는 **Unity Catalog에 네이티브로 통합**되어 거버넌스(권한, 리니지, 감사)가 자동 적용되는 것이 차별점입니다.

### 시맨틱 레이어 없이 발생하는 문제

```
CFO: "지난 분기 매출이 얼마야?"

재무팀 대시보드: "₩45.2억" (환불 제외, 세금 포함)
마케팅팀 대시보드: "₩52.8억" (환불 포함, 할인 전 금액)
영업팀 리포트:    "₩48.1억" (확정 매출만, 보류 제외)

→ "도대체 우리 매출이 얼마인 거야?" (경영진의 신뢰 상실)
```

이 문제의 근본 원인은 **각 팀이 독립적으로 SQL을 작성하면서 "매출"의 정의가 분산**되기 때문입니다. 10개 대시보드에 10개의 다른 `SUM(amount)` 로직이 존재합니다.

### 시맨틱 레이어의 해결 방식

| 관점 | 시맨틱 레이어 없이 | 시맨틱 레이어 사용 |
|------|-----------------|-----------------|
| **지표 정의** | 각 대시보드/쿼리에 하드코딩 | 중앙에서 한 번 정의 |
| **변경 관리** | 10개 대시보드를 일일이 수정 | 정의 1곳 수정 → 전체 자동 반영 |
| **거버넌스** | 누가 정의했는지 추적 불가 | UC의 소유자/권한/감사 적용 |
| **일관성** | 팀마다 숫자가 다름 | **전 조직 동일한 숫자** |
| **셀프서비스** | SQL을 알아야 분석 가능 | Genie에 "매출 보여줘"만 하면 됨 |

---

## Metric View란?

> 💡 **Metric View(메트릭 뷰)** 는 Databricks의 비즈니스 시맨틱 레이어 구현체입니다. Unity Catalog의 **1급 객체(First-class Object)** 로서, 테이블이나 뷰처럼 `catalog.schema.metric_view` 네임스페이스로 관리됩니다. 비즈니스 메트릭을 중앙에서 한 번 정의하면, AI/BI 대시보드, Genie, SQL 쿼리 어디서든 일관되게 사용할 수 있습니다.

### 왜 Metric View가 필요한가요?

조직이 커지면 **같은 비즈니스 지표가 팀마다 다르게 정의**되는 문제가 흔하게 발생합니다.

| 문제 상황 | 예시 |
|-----------|------|
| **지표 정의 불일치** | 마케팅팀의 "매출"에는 환불이 포함되고, 재무팀에는 포함되지 않음 |
| **로직 중복** | 같은 계산 로직이 10개 대시보드에 각각 하드코딩되어 있음 |
| **변경 관리 어려움** | 계산 방식이 바뀌면 모든 대시보드와 쿼리를 일일이 수정해야 함 |
| **거버넌스 부재** | 누가 어떤 지표를 어떻게 정의했는지 추적할 수 없음 |

Metric View는 이 문제를 다음과 같이 해결합니다.

| Metric View 장점 | 설명 |
|------------------|------|
| **Single Source of Truth** | 메트릭을 한 곳에서 정의하고 모든 도구에서 동일하게 사용합니다 |
| **Unity Catalog 거버넌스** | 메트릭에도 접근 권한, 태그, 리니지가 적용됩니다 |
| **자동 집계** | 차원(Dimension)과 측정값(Measure)을 정의하면 자동으로 집계합니다 |
| **AI/BI 통합** | Dashboard, Genie에서 메트릭을 직접 활용할 수 있습니다 |

---

## 핵심 개념

### 차원 (Dimension)과 측정값 (Measure)

| 개념 | 설명 | 예시 |
|------|------|------|
| **차원 (Dimension)** | 데이터를 분류하는 기준 (GROUP BY 대상) | 날짜, 지역, 제품 카테고리 |
| **측정값 (Measure)** | 집계되는 수치 (SUM, AVG 등의 대상) | 매출액, 주문 수, 평균 단가 |
| **필터 (Filter)** | 메트릭에 적용되는 기본 조건 | status = 'completed' |
| **Time Dimension** | 시간 기반 분석의 기준 시간 컬럼 | order_date |

> 💡 **비유**: 엑셀의 피벗 테이블에서 "행"에 넣는 것이 차원, "값"에 넣는 것이 측정값이라고 생각하면 됩니다.

---

## Metric View 생성

### SQL로 생성

```sql
-- 매출 메트릭 뷰 생성
CREATE OR REPLACE METRIC VIEW catalog.schema.revenue_metrics
COMMENT '매출 관련 핵심 비즈니스 메트릭'
AS SELECT
  -- 차원 (Dimensions)
  order_date AS time          DIMENSION COMMENT '주문 날짜',
  region                      DIMENSION COMMENT '판매 지역',
  product_category            DIMENSION COMMENT '제품 카테고리',
  customer_segment            DIMENSION COMMENT '고객 등급',

  -- 측정값 (Measures)
  SUM(order_amount)           MEASURE total_revenue
                              COMMENT '총 매출 (환불 제외)',
  COUNT(DISTINCT order_id)    MEASURE order_count
                              COMMENT '주문 건수',
  SUM(order_amount) / COUNT(DISTINCT order_id)
                              MEASURE avg_order_value
                              COMMENT '평균 주문 금액',
  COUNT(DISTINCT customer_id) MEASURE unique_customers
                              COMMENT '고유 고객 수'
FROM catalog.schema.gold_orders
WHERE order_status = 'completed';  -- 완료된 주문만 포함
```

### 주요 구문 요소

| 키워드 | 역할 | 설명 |
|--------|------|------|
| `DIMENSION` | 차원 정의 | GROUP BY에 사용되는 분류 기준입니다 |
| `MEASURE` | 측정값 정의 | 집계 함수(SUM, COUNT 등)가 적용되는 값입니다 |
| `COMMENT` | 설명 추가 | 메트릭의 비즈니스 의미를 문서화합니다 |
| `WHERE` | 기본 필터 | 메트릭에 항상 적용되는 조건입니다 |

---

## Metric View 쿼리

### 기본 쿼리

```sql
-- 일별 총 매출 조회
SELECT time, total_revenue
FROM catalog.schema.revenue_metrics
ORDER BY time DESC
LIMIT 30;
```

### 차원별 집계

```sql
-- 지역별, 카테고리별 매출 (자동으로 GROUP BY 처리)
SELECT region, product_category, total_revenue, order_count
FROM catalog.schema.revenue_metrics;
```

### 필터 적용

```sql
-- 서울 지역의 월별 매출 추이
SELECT time, total_revenue, avg_order_value
FROM catalog.schema.revenue_metrics
WHERE region = '서울'
ORDER BY time;
```

> 💡 **자동 집계**: Metric View를 쿼리하면 SELECT에 포함된 차원을 기준으로 **자동으로 GROUP BY**가 적용됩니다. 사용자가 직접 GROUP BY를 작성할 필요가 없습니다.

---

## AI/BI 대시보드 연동

Metric View는 **Lakeview 대시보드에서 직접 데이터 소스로 사용**할 수 있습니다.

### 대시보드에서 활용하는 방법

1. **대시보드 편집 화면**에서 새 데이터셋을 추가합니다
2. 데이터 소스로 **Metric View**를 선택합니다
3. 차원과 측정값이 자동으로 인식되어 **시각화가 쉬워집니다**
4. 필터 위젯은 차원 컬럼에 자동으로 연결됩니다

| 연동 장점 | 설명 |
|-----------|------|
| **일관된 지표** | 모든 대시보드가 동일한 매출 정의를 사용합니다 |
| **자동 집계** | 차트에 차원을 끌어다 놓으면 자동으로 올바른 집계가 적용됩니다 |
| **권한 통합** | Metric View의 접근 권한이 대시보드에도 적용됩니다 |
| **변경 전파** | 메트릭 정의가 바뀌면 모든 대시보드에 자동 반영됩니다 |

---

## Genie에서 메트릭 활용

**Genie**에 Metric View를 연결하면, 비즈니스 사용자가 자연어로 정확한 지표를 질문할 수 있습니다.

### Genie Space 설정

1. Genie Space를 생성하거나 편집합니다
2. 데이터 소스에 **Metric View를 추가**합니다
3. 사용자가 "이번 달 매출은?"이라고 물으면 Metric View의 정의대로 정확한 결과를 반환합니다

### Genie 질문 예시

| 자연어 질문 | Metric View 활용 |
|------------|-----------------|
| "이번 달 총 매출은?" | `total_revenue` 측정값 + `time` 차원 필터 |
| "지역별 주문 건수를 보여줘" | `order_count` 측정값 + `region` 차원 |
| "VIP 고객의 평균 주문 금액은?" | `avg_order_value` + `customer_segment` 필터 |

> 💡 **Genie + Metric View의 시너지**: Metric View 없이 Genie를 사용하면, Genie가 SQL을 자유롭게 생성하므로 "매출"의 정의가 질문마다 달라질 수 있습니다. Metric View를 연결하면 **항상 동일한 비즈니스 정의**로 답변합니다.

---

## 거버넌스 및 권한 관리

Metric View는 Unity Catalog의 **3-Level Namespace** 아래에서 관리되므로, 테이블과 동일한 거버넌스가 적용됩니다.

```sql
-- Metric View에 접근 권한 부여
GRANT SELECT ON METRIC VIEW catalog.schema.revenue_metrics
TO `analytics_team`;

-- 권한 확인
SHOW GRANTS ON METRIC VIEW catalog.schema.revenue_metrics;
```

| 거버넌스 기능 | 설명 |
|-------------|------|
| **접근 제어** | GRANT/REVOKE로 메트릭 접근 권한을 관리합니다 |
| **리니지 추적** | 메트릭이 어떤 테이블에서 파생되었는지 추적합니다 |
| **태그** | 메트릭에 비즈니스 도메인, 데이터 분류 등의 태그를 부여합니다 |
| **감사 로그** | 누가 언제 메트릭을 조회했는지 기록합니다 |

---

## 모범 사례

| 항목 | 권장 사항 |
|------|----------|
| **명확한 네이밍** | 메트릭 이름에 비즈니스 의미를 담습니다 (예: `total_revenue`, `monthly_active_users`) |
| **COMMENT 필수** | 모든 차원과 측정값에 COMMENT를 추가하여 정의를 문서화합니다 |
| **Gold 테이블 기반** | Metric View는 Medallion 아키텍처의 Gold 테이블 위에 생성합니다 |
| **기본 필터 활용** | WHERE 절로 "완료된 주문만", "활성 고객만" 등 기본 필터를 적용합니다 |
| **팀 간 합의** | 메트릭 정의는 데이터팀과 비즈니스팀이 함께 합의합니다 |
| **변경 관리** | 메트릭 정의 변경 시 영향받는 대시보드와 리포트를 확인합니다 |

---

## Metric View vs 일반 View 비교

| 비교 항목 | 일반 View | Metric View |
|-----------|----------|-------------|
| **용도** | 범용 데이터 변환 | 비즈니스 메트릭 정의 전용 |
| **자동 집계** | 없음 (직접 GROUP BY 작성) | 차원 기준 자동 집계 |
| **차원/측정값 구분** | 없음 | DIMENSION / MEASURE 명시 |
| **AI/BI 연동** | 수동 설정 | 자동 인식 |
| **Genie 최적화** | 일반 테이블 취급 | 메트릭 의미 자동 이해 |
| **거버넌스** | 동일 | 동일 + 메트릭 전용 리니지 |

---

## 정리

| 개념 | 핵심 내용 |
|------|----------|
| Metric View | 비즈니스 메트릭을 중앙에서 정의하고 일관되게 사용하는 Unity Catalog 기능입니다 |
| 차원 / 측정값 | DIMENSION은 분류 기준, MEASURE는 집계 대상입니다 |
| 자동 집계 | SELECT에 포함된 차원 기준으로 자동 GROUP BY가 적용됩니다 |
| AI/BI 연동 | Dashboard와 Genie에서 메트릭을 직접 활용할 수 있습니다 |
| 거버넌스 | Unity Catalog의 권한, 리니지, 태그가 그대로 적용됩니다 |

---

## 참고 링크

- [Metric View 공식 문서](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-metric-views.html)
- [Metric View 생성 (CREATE METRIC VIEW)](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-metric-view.html)
- [AI/BI Dashboard 공식 문서](https://docs.databricks.com/aws/en/dashboards/)
- [Genie 공식 문서](https://docs.databricks.com/aws/en/genie/)
- [Databricks 블로그: Metric Views 소개](https://www.databricks.com/blog/introducing-metric-views)
