# Genie — 자연어 데이터 분석

## Genie란?

> 💡 **Genie**는 **자연어**로 데이터에 질문하면, AI가 적절한 **SQL 쿼리를 자동으로 생성**하여 결과를 보여주는 Databricks의 AI 기반 데이터 탐색 도구입니다. SQL을 모르는 비즈니스 사용자도 "이번 달 서울 지역 매출이 얼마야?"라고 물으면 답변을 받을 수 있습니다.

---

## Genie의 동작 방식

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | 사용자 질문 | "이번 달 서울 매출이 전월 대비 어떻게 됐어?" |
| 2 | Genie AI | 질문을 이해하고, 테이블을 매핑하고, SQL을 생성합니다 |
| 3 | 생성된 SQL | `SELECT month, SUM(amount)... WHERE city='서울'` |
| 4 | SQL Warehouse | 쿼리를 실행합니다 |
| 5 | 결과 + 차트 | 실행 결과를 차트와 함께 표시합니다 |

1. 사용자가 **자연어**로 질문을 입력합니다
2. Genie AI가 질문을 분석하고, 등록된 테이블의 스키마와 지시사항을 참고하여 **SQL을 생성**합니다
3. 생성된 SQL이 **SQL Warehouse**에서 실행됩니다
4. 결과가 **테이블과 차트**로 표시됩니다
5. 사용자가 생성된 SQL을 확인하고, 필요하면 수정할 수 있습니다

---

## Genie Space 구성

> 💡 **Genie Space**는 Genie가 질문에 답변할 때 참조하는 **전용 분석 공간**입니다. 어떤 테이블을 사용할지, 비즈니스 용어를 어떻게 해석할지, 자주 묻는 질문에 어떻게 답변할지를 사전에 정의합니다.

### Genie Space의 구성 요소

| 구성 요소 | 설명 | 예시 |
|-----------|------|------|
| **테이블 선택** | Genie가 참조할 테이블을 지정합니다 | `gold.daily_revenue`, `gold.customer_360` |
| **지시사항 (Instructions)** | 비즈니스 맥락과 용어를 설명합니다 | "매출"은 `total_amount` 컬럼을 의미합니다. 금액은 원 단위입니다 |
| **예시 질문** | 사용자가 물을 수 있는 질문의 예시를 등록합니다 | "이번 달 매출 Top 5 지역", "VIP 고객 수 추이" |
| **인증된 답변 (Trusted Assets)** | 자주 묻는 질문에 대해 **검증된 SQL**을 미리 등록합니다. AI가 생성한 SQL 대신 검증된 SQL을 사용합니다 | "월별 매출" → 사전 검증된 정확한 SQL |
| **샘플 질문** | Genie Space에 처음 접속한 사용자에게 보여줄 추천 질문입니다 | "가장 많이 팔린 상품은?", "고객 이탈률은?" |

### Genie Space 생성 방법

1. **New** → **Genie Space** 클릭
2. Space 이름 입력 (예: "매출 분석 Genie")
3. **테이블 추가**: Unity Catalog에서 Gold 테이블들을 선택합니다
4. **지시사항 작성**: 비즈니스 맥락을 제공합니다

```
지시사항 예시:
- "매출"이라고 하면 total_revenue 컬럼을 사용하세요
- 금액의 단위는 원(KRW)입니다
- "이번 달"은 현재 월을 의미합니다
- "VIP 고객"은 tier = 'Gold' 또는 tier = 'Platinum'인 고객입니다
- 날짜 관련 질문에는 order_date 컬럼을 기준으로 답변하세요
- 답변은 항상 한국어로 제공하세요
```

5. **인증된 답변 등록** (선택): 핵심 비즈니스 질문에 검증된 SQL을 매핑합니다
6. **공유**: 비즈니스 사용자에게 Genie Space 접근 권한을 부여합니다

---

## 인증된 답변 (Trusted Assets)

> 💡 **인증된 답변**은 특정 질문에 대해 **사전 검증된 정확한 SQL**을 등록하는 기능입니다. Genie는 해당 질문이 들어오면 AI가 생성한 SQL 대신 인증된 SQL을 사용하므로, **정확도가 100% 보장**됩니다.

### 왜 필요한가요?

- AI가 생성한 SQL이 항상 100% 정확하지는 않습니다
- 비즈니스적으로 **정의가 복잡한 지표**(예: "순매출 = 총매출 - 반품 - 할인")는 AI가 틀릴 가능성이 높습니다
- 경영진에게 보고되는 핵심 KPI는 **검증된 쿼리**를 사용해야 합니다

### 등록 방법

```sql
-- 인증된 답변으로 등록할 SQL 쿼리 예시
-- 질문: "이번 달 순매출은?"

SELECT
    DATE_TRUNC('MONTH', order_date) AS 월,
    SUM(CASE WHEN type = 'SALE' THEN amount ELSE 0 END)
    - SUM(CASE WHEN type = 'RETURN' THEN amount ELSE 0 END)
    - SUM(CASE WHEN type = 'DISCOUNT' THEN amount ELSE 0 END) AS 순매출
FROM gold.transactions
WHERE order_date >= DATE_TRUNC('MONTH', CURRENT_DATE())
GROUP BY 1;
```

---

## Genie 활용 시나리오

### 경영진 의사결정 지원

```
👤 CEO: "지난 분기 대비 이번 분기 매출 성장률은?"
🧞 Genie: "2025년 1분기 매출은 45.2억원으로, 전 분기(42.8억원) 대비 5.6% 성장했습니다."
         [차트: 분기별 매출 추이 막대 그래프]

👤 CEO: "가장 성장이 빠른 지역은?"
🧞 Genie: "대전 지역이 전분기 대비 23.4% 성장으로 가장 높은 성장률을 보였습니다."
         [표: 지역별 성장률 순위]
```

### 마케팅 팀 캠페인 분석

```
👤 마케터: "3월 프로모션 기간 동안 신규 고객 수는?"
🧞 Genie: "3월 1일~15일 프로모션 기간 동안 신규 가입 고객은 1,847명입니다."

👤 마케터: "그 중 실제 구매한 고객 비율은?"
🧞 Genie: "1,847명 중 623명(33.7%)이 첫 구매를 완료했습니다."
```

---

## Genie Space의 데이터 보안

Genie는 Unity Catalog의 권한 체계를 그대로 따릅니다.

| 보안 사항 | 설명 |
|-----------|------|
| **테이블 권한** | 사용자가 SELECT 권한이 없는 테이블의 데이터는 Genie도 접근할 수 없습니다 |
| **행 필터** | Row Filter가 설정된 테이블은 Genie에서도 필터가 적용됩니다 |
| **컬럼 마스킹** | Column Mask가 설정된 컬럼은 Genie에서도 마스킹됩니다 |
| **SQL 노출** | 사용자가 생성된 SQL을 볼 수 있으므로, 민감한 테이블 이름이 노출될 수 있습니다 |

---

## Genie Space 품질 향상 팁

| 방법 | 설명 |
|------|------|
| **명확한 컬럼 이름** | `amt` 대신 `total_revenue_krw`처럼 의미가 명확한 컬럼 이름을 사용합니다 |
| **컬럼 설명 추가** | `COMMENT ON COLUMN table.col IS '설명'`으로 각 컬럼에 설명을 달아둡니다 |
| **상세한 지시사항** | 비즈니스 용어, 약어, 관계를 충분히 설명합니다 |
| **인증된 답변 활용** | 핵심 KPI에는 반드시 검증된 SQL을 등록합니다 |
| **피드백 활용** | 사용자가 "좋아요/싫어요"를 표시한 결과를 분석하여 지시사항을 개선합니다 |
| **Gold 테이블 사용** | 복잡한 조인이 필요 없는 Gold 계층 테이블을 사용합니다 |

> 🆕 **Genie Code (GA)**: 기존 Databricks Assistant가 확장되어, 단순 질문 답변을 넘어 **멀티 스텝 데이터 작업**(자산 검색, 코드 생성, 실행, 오류 수정, 결과 시각화)을 자율적으로 수행합니다.

---

## 심화: Genie 엔터프라이즈 운영

이 섹션에서는 Genie를 전사적으로 도입할 때 필요한 품질 관리, 성능 특성, 보안 패턴, 지속적 개선 프로세스를 다룹니다.

### 품질 관리 — 할루시네이션과 정확도

Genie는 LLM 기반이므로, 잘못된 SQL을 생성하는 **할루시네이션(Hallucination)** 이 발생할 수 있습니다. 프로덕션 환경에서는 체계적인 품질 관리가 필수입니다.

#### 할루시네이션 유형

| 유형 | 예시 | 영향 |
|------|------|------|
| **존재하지 않는 컬럼/테이블 참조** | `SELECT revenue FROM sales` (실제 컬럼명: `total_amount`) | SQL 오류 → 사용자에게 오류 메시지 |
| **잘못된 비즈니스 로직** | "순매출"을 `SUM(amount)`으로 계산 (정확: 매출-반품-할인) | **위험!** 잘못된 숫자가 표시됨 |
| **잘못된 조인** | 관련 없는 테이블을 조인하여 카디널리티 폭발 | 부정확한 결과 + 높은 비용 |
| **집계 함수 오용** | COUNT 대신 SUM, AVG 대신 MEDIAN | 미묘하게 다른 결과 |

#### Trusted Assets로 품질 보장

인증된 답변(Trusted Assets)은 Genie의 품질 문제에 대한 가장 강력한 방어 수단입니다.

```
권장 운영 전략:

1. 핵심 KPI (매출, DAU, 이탈률 등)
   → 반드시 Trusted Assets 등록 (정확도 100% 보장)

2. 자주 묻는 질문 Top 20
   → Trusted Assets 등록 (80/20 법칙 적용)

3. 탐색적 질문
   → Genie AI가 생성 (지시사항으로 가이드)
   → 사용자가 SQL 검증 후 사용
```

#### 테스트 패턴

```python
# Genie Space 품질 테스트 자동화 예시
test_cases = [
    {
        "question": "이번 달 총 매출은?",
        "expected_columns": ["total_revenue"],
        "expected_range": (1_000_000, 1_000_000_000),  # 합리적 범위
        "must_contain_table": "gold.daily_revenue"
    },
    {
        "question": "VIP 고객 수는?",
        "expected_columns": ["customer_count"],
        "expected_sql_contains": "tier IN ('Gold', 'Platinum')",  # 비즈니스 로직 확인
    },
]

# Genie API로 자동 테스트 (SDK 활용)
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()

for tc in test_cases:
    # Genie Space에 질문 전송
    response = w.genie.start_conversation(
        space_id="<genie-space-id>",
        content=tc["question"]
    )
    # 생성된 SQL 검증
    generated_sql = response.attachments[0].query.query
    assert tc["must_contain_table"] in generated_sql, f"잘못된 테이블 참조: {generated_sql}"
```

> ⚠️ **Gotcha — Genie의 SQL은 매번 달라질 수 있습니다**: 동일한 질문에도 LLM의 특성상 미묘하게 다른 SQL이 생성될 수 있습니다. 테스트 시 정확한 SQL 문자열 매칭보다는 **결과 값의 범위 검증**, **필수 테이블/컬럼 포함 여부** 등으로 검증하세요.

---

### 성능 특성

| 메트릭 | 일반적인 범위 | 설명 |
|--------|-------------|------|
| **응답 지연시간 (p50)** | 3~8초 | 질문 해석 + SQL 생성 + 쿼리 실행 |
| **응답 지연시간 (p99)** | 15~30초 | 복잡한 질문, 대규모 테이블 |
| **SQL 생성 시간** | 2~5초 | LLM이 SQL을 생성하는 시간 |
| **쿼리 실행 시간** | 1~25초 | SQL Warehouse에서 실행 (데이터 규모에 따라 다름) |
| **동시 사용자** | Space당 권장 20~30명 | 초과 시 SQL Warehouse 큐잉 발생 |

#### 성능 최적화 팁

| 방법 | 효과 | 설명 |
|------|------|------|
| **Gold 테이블 사용** | 쿼리 시간 50~90% 감소 | 사전 집계된 테이블은 조인/집계가 불필요 |
| **테이블 수 제한** | SQL 생성 정확도 향상 | Space당 5~10개 테이블 권장. 너무 많으면 LLM 혼란 |
| **컬럼 설명 추가** | SQL 정확도 향상 | LLM이 컬럼 의미를 정확히 파악 |
| **Serverless Warehouse** | 콜드스타트 감소 | 첫 쿼리 실행까지 대기 시간 최소화 |

> ⚠️ **Gotcha — 테이블 수와 정확도의 반비례**: Genie Space에 20개 이상의 테이블을 등록하면, LLM이 적절한 테이블을 선택하는 정확도가 떨어집니다. **도메인별로 별도 Genie Space**를 만들고, 각 Space에는 관련 테이블만 등록하세요.

---

### 엔터프라이즈 패턴

#### 부서별 Genie Space 분리

```
전사 Genie 아키텍처:

[경영진 Genie Space]
  - 테이블: gold.executive_summary, gold.kpi_monthly
  - 지시사항: 경영 지표 중심, 한국어 답변
  - 사용자: C-Level, VP

[영업팀 Genie Space]
  - 테이블: gold.sales_pipeline, gold.customer_360
  - 지시사항: 영업 용어 정의, 파이프라인 단계 설명
  - 사용자: 영업 담당자

[마케팅팀 Genie Space]
  - 테이블: gold.campaign_performance, gold.customer_segments
  - 지시사항: 캠페인 관련 용어, 전환율 계산 방법
  - 사용자: 마케터

[재무팀 Genie Space]
  - 테이블: gold.financial_ledger, gold.budget_vs_actual
  - 지시사항: 회계 용어, 기간 구분 규칙
  - 사용자: 재무팀
```

#### 데이터 보안 — UC Row Filter/Column Mask 연동

Genie는 Unity Catalog의 보안 정책을 그대로 상속합니다. Row Filter와 Column Mask를 설정하면 Genie에서도 동일하게 적용됩니다.

```sql
-- 행 필터: 사용자가 담당하는 지역의 데이터만 조회 가능
CREATE FUNCTION gold.region_filter(region_col STRING)
RETURN IF(
    IS_ACCOUNT_GROUP_MEMBER('all-regions-access'),
    TRUE,
    region_col = CURRENT_USER_ATTRIBUTE('region')
);

ALTER TABLE gold.sales_data
SET ROW FILTER gold.region_filter ON (region);

-- 컬럼 마스크: 급여 컬럼을 HR 팀만 볼 수 있음
CREATE FUNCTION gold.salary_mask(salary_col DECIMAL)
RETURN IF(
    IS_ACCOUNT_GROUP_MEMBER('hr-team'),
    salary_col,
    NULL
);

ALTER TABLE gold.employee_data
SET COLUMN MASK gold.salary_mask ON salary;
```

> ⚠️ **Gotcha — Genie에서 보안 정책 우회 가능성**: Genie가 생성한 SQL에서 필터 조건을 교묘하게 구성하면, Row Filter를 논리적으로 우회하려는 시도가 가능합니다(예: "급여가 1억 이상인 직원이 있어?"라는 질문). UC의 Row Filter/Column Mask는 **SQL 레벨에서 강제 적용**되므로 기술적으로 우회는 불가능하지만, **SQL이 노출**되어 테이블/컬럼 이름 같은 메타데이터가 보일 수 있습니다. 민감한 테이블 이름을 사용하지 않도록 주의하세요.

---

### 피드백 루프 — 지속적 품질 개선

Genie를 도입한 후에는 **사용자 피드백을 수집하고 지속적으로 개선**하는 프로세스가 필요합니다.

#### 개선 사이클

```
[1] 사용자가 Genie에 질문
    ↓
[2] Genie가 SQL 생성 + 결과 반환
    ↓
[3] 사용자가 결과에 👍/👎 피드백
    ↓
[4] 관리자가 👎 피드백 분석
    ↓
[5-a] 반복적인 오류 → Trusted Asset 등록
[5-b] 용어 문제 → 지시사항 보완
[5-c] 데이터 문제 → 테이블/컬럼 설명 보완
    ↓
[6] 다음 주기에 동일 질문으로 검증
```

#### 피드백 분석 체크리스트

| 피드백 유형 | 분석 방법 | 대응 |
|------------|----------|------|
| SQL 오류 | 생성된 SQL에 존재하지 않는 컬럼/테이블 | 지시사항에 정확한 컬럼명 명시 |
| 잘못된 숫자 | 결과를 수동 쿼리와 비교 | Trusted Asset으로 검증된 SQL 등록 |
| "모르겠습니다" 응답 | 질문과 등록된 테이블 매핑 확인 | 테이블 추가 또는 지시사항에 매핑 규칙 추가 |
| 느린 응답 | 생성된 SQL의 실행 계획 분석 | Gold 테이블 추가, 인덱싱 개선 |

#### 운영 지표 모니터링

| 지표 | 목표 | 측정 방법 |
|------|------|----------|
| **질문 성공률** | > 85% | (응답 반환 횟수 / 전체 질문 수) |
| **사용자 만족도** | > 70% 👍 | (👍 수 / 전체 피드백 수) |
| **Trusted Asset 적중률** | > 40% | (Trusted Asset 사용 횟수 / 전체 질문 수) |
| **평균 응답 시간** | < 10초 | SQL 생성 시간 + 쿼리 실행 시간 |

> ⚠️ **Gotcha — 초기 도입 시 기대 관리**: Genie 도입 초기에는 지시사항과 Trusted Asset이 부족하여 정확도가 낮을 수 있습니다(50~60%). **최소 2~4주의 튜닝 기간**을 계획하고, 초기에는 **파워 유저 그룹**(분석가, 데이터 엔지니어)과 함께 피드백을 수집하여 품질을 높인 후 전사 배포하세요.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Genie** | 자연어로 데이터에 질문하는 AI 분석 도구입니다 |
| **Genie Space** | Genie가 참조할 테이블, 지시사항, 인증된 답변을 설정한 전용 공간입니다 |
| **인증된 답변** | 핵심 질문에 검증된 SQL을 매핑하여 정확도를 보장합니다 |
| **지시사항** | 비즈니스 용어와 맥락을 Genie에게 제공합니다 |
| **Unity Catalog 보안** | 데이터 권한이 Genie에도 동일하게 적용됩니다 |
| **피드백 루프** | 사용자 피드백을 수집하여 지속적으로 품질을 개선합니다 |
| **부서별 분리** | 도메인별 Genie Space로 정확도와 보안을 동시에 확보합니다 |

---

## 참고 링크

- [Databricks: Genie](https://docs.databricks.com/aws/en/aibi/genie/)
- [Databricks: Genie spaces](https://docs.databricks.com/aws/en/aibi/genie/genie-spaces.html)
- [Azure Databricks: Genie](https://learn.microsoft.com/en-us/azure/databricks/aibi/genie/)
