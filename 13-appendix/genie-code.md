# Genie Code (Databricks Assistant)

## Genie Code란?

Genie Code는 Databricks 노트북과 SQL 에디터에 내장된 **AI 코딩 어시스턴트**입니다. 자연어로 코드를 생성하고, 기존 코드를 설명하거나 수정하며, 오류를 디버깅할 수 있습니다. 별도의 설치 없이 Databricks 워크스페이스 내에서 바로 사용할 수 있습니다.

> 💡 **Genie Code의 핵심 가치**
> 코딩 경험이 적은 분석가도 자연어로 데이터 분석 코드를 생성할 수 있고, 숙련된 개발자도 반복적인 코드 작성 시간을 크게 줄일 수 있습니다.

---

## Genie vs Genie Code: 차이점

Databricks에는 "Genie"라는 이름이 붙은 두 가지 기능이 있습니다. 혼동하기 쉬우므로 차이점을 명확히 이해하는 것이 중요합니다.

| 구분 | Genie (AI/BI Genie) | Genie Code (Databricks Assistant) |
|---|---|---|
| **주요 대상** | 비즈니스 사용자, 분석가 | 데이터 엔지니어, 데이터 사이언티스트, 분석가 |
| **사용 위치** | Genie Space (전용 인터페이스) | 노트북, SQL 에디터 |
| **입력** | 자연어 질문 | 자연어 프롬프트 |
| **출력** | SQL 쿼리 + 결과 테이블/차트 | Python, SQL, Scala, R 코드 셀 |
| **데이터 범위** | Genie Space에 등록된 테이블만 | 워크스페이스 내 모든 접근 가능한 데이터 |
| **커스터마이징** | 샘플 질문, 지침 설정 가능 | 노트북 컨텍스트를 자동으로 인식 |
| **목적** | 셀프서비스 BI (질문→답변) | 코드 생성·수정·디버깅 |

쉽게 말해, **Genie**는 "질문하면 답을 알려주는 BI 도구"이고, **Genie Code**는 "노트북에서 코드를 대신 써주는 코딩 도우미"입니다.

---

## 주요 기능

### 1. 코드 생성

자연어 설명을 입력하면 해당하는 코드를 자동으로 생성합니다.

**예시 프롬프트:**
```
고객 테이블에서 가입일 기준 월별 신규 가입자 수를 집계하고,
최근 12개월 트렌드를 라인 차트로 시각화해줘
```

**생성되는 코드 (Python):**
```python
import matplotlib.pyplot as plt
import matplotlib.dates as mdates

df = spark.sql("""
    SELECT
        date_trunc('month', signup_date) AS signup_month,
        COUNT(*) AS new_customers
    FROM catalog.schema.customers
    WHERE signup_date >= add_months(current_date(), -12)
    GROUP BY 1
    ORDER BY 1
""").toPandas()

fig, ax = plt.subplots(figsize=(12, 6))
ax.plot(df['signup_month'], df['new_customers'], marker='o')
ax.set_title('월별 신규 가입자 추이')
ax.set_xlabel('월')
ax.set_ylabel('신규 가입자 수')
ax.xaxis.set_major_formatter(mdates.DateFormatter('%Y-%m'))
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

### 2. 코드 설명

기존 코드를 선택하고 "이 코드를 설명해줘"라고 요청하면 한국어로 상세한 설명을 제공합니다.

### 3. 코드 수정 및 리팩토링

기존 코드에 대해 수정 요청을 할 수 있습니다.

```
이 코드에서 pandas 대신 PySpark DataFrame API를 사용하도록 변환해줘
```

```
이 SQL에 윈도우 함수를 추가해서 전월 대비 증감률도 계산해줘
```

### 4. 오류 디버깅

셀 실행 시 오류가 발생하면 Genie Code가 자동으로 오류를 분석하고 수정 방안을 제안합니다.

```
오류 메시지: AnalysisException: Column 'signup_date' does not exist

Genie Code 제안:
"signup_date 컬럼이 존재하지 않습니다.
테이블의 실제 컬럼명은 'created_at'입니다.
수정된 코드를 생성할까요?"
```

### 5. 자동 완성 (Autocomplete)

코드를 입력하는 도중 실시간으로 코드 완성 제안을 표시합니다. `Tab` 키를 눌러 제안을 수락할 수 있습니다.

---

## 사용 방법

### 노트북에서 사용

1. **새 셀에서 프롬프트 입력**: 빈 코드 셀에 자연어로 원하는 작업을 설명합니다.
2. **Generate 버튼 클릭**: 또는 `Cmd+Shift+G` (Mac) / `Ctrl+Shift+G` (Windows) 단축키를 사용합니다.
3. **코드 검토 및 수정**: 생성된 코드를 검토하고 필요하면 추가 프롬프트로 수정합니다.
4. **셀 실행**: 검토가 완료되면 셀을 실행합니다.

### SQL 에디터에서 사용

SQL 에디터에서도 동일하게 자연어 프롬프트로 SQL 쿼리를 생성할 수 있습니다.

```
사용자: "지역별 매출 Top 10을 피벗 테이블로 만들어줘"
```

```sql
-- 생성된 SQL
SELECT *
FROM (
    SELECT
        region,
        product_category,
        SUM(revenue) AS total_revenue
    FROM catalog.schema.sales
    GROUP BY region, product_category
)
PIVOT (
    SUM(total_revenue)
    FOR product_category IN ('Electronics', 'Clothing', 'Food', 'Books', 'Sports')
)
ORDER BY Electronics DESC
LIMIT 10;
```

### 지원 언어

| 언어 | 노트북 | SQL 에디터 | 주요 용도 |
|---|---|---|---|
| **Python**| O | - | 데이터 처리, ML, 시각화 |
| **SQL**| O | O | 데이터 조회, 분석 |
| **Scala**| O | - | Spark 네이티브 처리 |
| **R** | O | - | 통계 분석, 시각화 |

---

## 활용 시나리오

### 시나리오 1: 데이터 탐색 (EDA)

```
프롬프트: "sales 테이블의 기본 통계를 보여주고,
NULL 비율이 높은 컬럼을 찾아줘"
```

Genie Code가 `describe()`, 컬럼별 NULL 카운트, 데이터 타입 요약 등의 코드를 자동으로 생성합니다.

### 시나리오 2: ETL 코드 생성

```
프롬프트: "raw_events 테이블에서 event_type이 'purchase'인 레코드만 필터링하고,
user_id별 총 구매 금액을 계산해서 user_purchases 테이블로 저장해줘"
```

```python
# 생성된 코드
from pyspark.sql import functions as F

purchases = (
    spark.table("catalog.schema.raw_events")
    .filter(F.col("event_type") == "purchase")
    .groupBy("user_id")
    .agg(
        F.sum("amount").alias("total_purchase_amount"),
        F.count("*").alias("purchase_count"),
        F.max("event_timestamp").alias("last_purchase_at")
    )
)

purchases.write.mode("overwrite").saveAsTable("catalog.schema.user_purchases")
```

### 시나리오 3: ML 모델 학습 코드 생성

```
프롬프트: "customer_features 테이블을 사용해서 이탈 예측(churn prediction) 모델을
scikit-learn으로 학습하고 MLflow에 로깅해줘"
```

### 시나리오 4: 복잡한 SQL 작성

```
프롬프트: "주문 테이블에서 고객별 RFM(Recency, Frequency, Monetary) 점수를 계산해줘.
각 지표를 5분위로 나누고 총점도 계산해줘"
```

---

## 모범 사례: 효과적인 프롬프트 작성

### 구체적으로 요청하세요

| 비효율적 프롬프트 | 효과적 프롬프트 |
|---|---|
| "데이터 분석해줘" | "orders 테이블에서 월별 매출 추이를 분석하고 전월 대비 증감률을 계산해줘" |
| "그래프 그려줘" | "카테고리별 매출 비중을 파이 차트로 그려줘, 상위 5개만 표시하고 나머지는 기타로 묶어줘" |
| "오류 고쳐줘" | "이 코드에서 AnalysisException이 발생하는데, 테이블명이 변경된 것 같아. 올바른 테이블명으로 수정해줘" |

### 컨텍스트를 제공하세요

Genie Code는 현재 노트북의 이전 셀 내용을 컨텍스트로 활용합니다. 따라서 아래와 같이 하면 더 정확한 코드를 생성할 수 있습니다.

1. **첫 번째 셀**에서 테이블을 로드합니다.
2. **두 번째 셀**에서 "위 데이터프레임에서 이상치를 탐지해줘"라고 요청합니다.

Genie Code가 이전 셀의 변수명, 스키마 정보를 자동으로 인식합니다.

### 단계별로 요청하세요

한 번에 복잡한 작업을 요청하기보다, 단계별로 나누어 요청하면 더 정확한 결과를 얻을 수 있습니다.

```
1단계: "raw_events 테이블의 스키마를 확인해줘"
2단계: "event_type별 건수를 집계해줘"
3단계: "purchase 이벤트만 필터링해서 일별 매출을 계산해줘"
4단계: "위 결과를 시각화해줘"
```

---

## 보안 고려사항

### Unity Catalog 권한 연동

Genie Code가 생성하는 코드는 실행 시 **사용자의 Unity Catalog 권한**이 그대로 적용됩니다. 접근 권한이 없는 테이블에 대한 코드를 생성해도 실행 단계에서 권한 오류가 발생합니다.

| 보안 항목 | 설명 |
|---|---|
| **데이터 접근**| Unity Catalog GRANT 기반 권한 적용 |
| ** 행/열 수준 보안**| Row Filter, Column Mask 정책이 적용된 상태로 데이터 반환 |
| ** 감사 로그**| Genie Code 사용 내역이 시스템 테이블에 기록 |
| ** 코드 실행**| 생성된 코드는 반드시 사용자가 검토 후 수동 실행 |

### 주의사항

- Genie Code가 생성한 코드는 ** 반드시 검토 후 실행**하세요. AI가 생성한 코드에 논리적 오류가 있을 수 있습니다.
- **프로덕션 테이블에 대한 쓰기 작업**(INSERT, UPDATE, DELETE)은 특히 주의 깊게 검토하세요.
- 민감한 데이터(PII)를 프롬프트에 직접 입력하지 마세요. 테이블명과 컬럼명으로 참조하세요.

---

## 설정 및 관리

### 워크스페이스 관리자 설정

워크스페이스 관리자는 Genie Code 기능을 활성화/비활성화할 수 있습니다.

```
워크스페이스 설정 → Preview → Databricks Assistant 토글
```

### 자동 완성 설정

개인 사용자는 노트북에서 자동 완성을 켜거나 끌 수 있습니다.

```
사용자 설정 → Developer → Autocomplete → Enable/Disable
```

---

## 정리

| 항목 | 내용 |
|---|---|
| **핵심 가치** | 노트북 내에서 자연어로 코드 생성·수정·디버깅 |
| **사용 위치** | Databricks 노트북, SQL 에디터 |
| **지원 언어** | Python, SQL, Scala, R |
| **보안** | Unity Catalog 권한 기반, 행/열 수준 보안 적용 |
| **대상 사용자** | 데이터 엔지니어, 데이터 사이언티스트, 분석가 |

---

## 참고 링크

- [Databricks Assistant (Genie Code) 공식 문서](https://docs.databricks.com/aws/en/notebooks/use-databricks-assistant.html)
- [Genie Code를 활용한 노트북 생산성 향상](https://docs.databricks.com/aws/en/notebooks/notebook-assistant-faq.html)
- [AI/BI Genie Space 공식 문서](https://docs.databricks.com/aws/en/genie/)
- [Unity Catalog 데이터 거버넌스](https://docs.databricks.com/aws/en/data-governance/)
