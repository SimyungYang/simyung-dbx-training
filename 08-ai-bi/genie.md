# Genie — 자연어 데이터 분석

## Genie란?

> 💡 **Genie**는 **자연어**로 데이터에 질문하면, AI가 적절한 **SQL 쿼리를 자동으로 생성**하여 결과를 보여주는 Databricks의 AI 기반 데이터 탐색 도구입니다. SQL을 모르는 비즈니스 사용자도 "이번 달 서울 지역 매출이 얼마야?"라고 물으면 답변을 받을 수 있습니다.

---

## Genie의 동작 방식

```mermaid
graph LR
    Q["👤 '이번 달 서울 매출이<br/>전월 대비 어떻게 됐어?'"]
    AI["🧞 Genie AI<br/>1. 질문 이해<br/>2. 테이블 매핑<br/>3. SQL 생성"]
    SQL["📝 생성된 SQL:<br/>SELECT month,<br/>SUM(amount)...<br/>WHERE city='서울'"]
    WH["🖥️ SQL Warehouse<br/>쿼리 실행"]
    R["📊 결과 + 차트"]

    Q --> AI --> SQL --> WH --> R
```

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

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Genie** | 자연어로 데이터에 질문하는 AI 분석 도구입니다 |
| **Genie Space** | Genie가 참조할 테이블, 지시사항, 인증된 답변을 설정한 전용 공간입니다 |
| **인증된 답변** | 핵심 질문에 검증된 SQL을 매핑하여 정확도를 보장합니다 |
| **지시사항** | 비즈니스 용어와 맥락을 Genie에게 제공합니다 |
| **Unity Catalog 보안** | 데이터 권한이 Genie에도 동일하게 적용됩니다 |

---

## 참고 링크

- [Databricks: Genie](https://docs.databricks.com/aws/en/aibi/genie/)
- [Databricks: Genie spaces](https://docs.databricks.com/aws/en/aibi/genie/genie-spaces.html)
- [Azure Databricks: Genie](https://learn.microsoft.com/en-us/azure/databricks/aibi/genie/)
