# SQL 주요 기능

## Databricks SQL의 고급 기능

Databricks SQL은 표준 SQL(ANSI SQL)을 지원하며, 추가로 분석에 유용한 고급 기능들을 제공합니다. 이 문서에서는 Databricks SQL만의 차별화된 기능과 고급 SQL 패턴을 살펴보겠습니다.

> 💡 **Databricks SQL은 Apache Spark SQL을 기반**으로 하며, 여기에 파이프 구문, QUALIFY, SQL 스크립팅 등 생산성을 높이는 확장 기능을 더했습니다.

---

## 파이프 구문 (Pipe Syntax)

> 🆕 **파이프 구문(`|>`)**은 SQL 쿼리를 ** 위에서 아래로 순서대로**읽을 수 있게 해 주는 Databricks의 SQL 구문입니다. 전통 SQL에서는 실행 순서(FROM → WHERE → GROUP BY → SELECT)와 작성 순서(SELECT → FROM → WHERE)가 다르지만, 파이프 구문은 ** 작성 순서 = 실행 순서**입니다.

### 전통 SQL vs 파이프 구문 비교

```sql
-- 전통적 SQL: 실행 순서와 작성 순서가 다릅니다
SELECT city, COUNT(*) AS cnt
FROM customers
WHERE signup_date >= '2025-01-01'
GROUP BY city
HAVING cnt >= 10
ORDER BY cnt DESC
LIMIT 5;

-- 파이프 구문: 위에서 아래로 자연스럽게 읽힙니다
FROM customers
|> WHERE signup_date >= '2025-01-01'
|> AGGREGATE COUNT(*) AS cnt GROUP BY city
|> WHERE cnt >= 10
|> ORDER BY cnt DESC
|> LIMIT 5;
```

### 파이프 구문의 핵심 규칙

| 규칙 | 설명 |
|------|------|
| **시작**| 반드시 `FROM` 절로 시작합니다 |
| ** 파이프 연산자**| 각 단계를 `\|>`로 연결합니다 |
| ** 집계**| `GROUP BY` 대신 `AGGREGATE ... GROUP BY`를 사용합니다 |
| **HAVING 대체**| 집계 후 필터는 `\|> WHERE`로 작성합니다 (HAVING 불필요) |
| **SELECT**| `\|> SELECT`로 컬럼을 선택하거나 변환합니다 |
| ** 중간 결과 활용**| 각 단계의 결과가 다음 단계의 입력이 됩니다 |

> 💡 ** 파이프 구문의 장점**: 쿼리가 길어질수록 전통 SQL은 읽기 어려워지지만, 파이프 구문은 **데이터가 흐르는 순서대로** 읽을 수 있어 가독성이 유지됩니다. 특히 여러 단계의 집계와 필터가 필요한 분석에서 효과적입니다.

---

## CTE (Common Table Expression)

CTE는 `WITH` 절을 사용하여 **쿼리 내에서 이름 붙은 임시 결과 집합**을 정의하는 기능입니다. 복잡한 쿼리를 논리적 단계로 나누어 **가독성과 재사용성**을 높입니다.

```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('MONTH', order_date) AS month,
        SUM(amount) AS revenue
    FROM silver_orders
    GROUP BY 1
),
growth AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
        ROUND((revenue - LAG(revenue) OVER (ORDER BY month))
            / LAG(revenue) OVER (ORDER BY month) * 100, 1) AS growth_pct
    FROM monthly_sales
)
SELECT * FROM growth ORDER BY month;
```

> ⚠️ **재귀 CTE 주의사항**: 무한 루프를 방지하기 위해 Databricks는 기본적으로 최대 재귀 깊이를 제한합니다. `spark.sql.cte.maxRecursionDepth` 설정으로 조정할 수 있습니다.

---

## QUALIFY 절

> 💡 **QUALIFY** 는 윈도우 함수의 결과를 기준으로 행을 필터링하는 절입니다. 서브쿼리 없이 윈도우 함수 결과를 바로 필터링할 수 있어 쿼리가 훨씬 간결해집니다.

```sql
-- ❌ 전통적 방식: 서브쿼리 필요
SELECT * FROM (
    SELECT
        customer_id,
        order_date,
        amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
WHERE rn = 1;

-- ✅ QUALIFY 사용: 서브쿼리 불필요
SELECT
    customer_id,
    order_date,
    amount
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;
```

---

## LATERAL VIEW / EXPLODE (반구조화 데이터)

배열(Array)이나 맵(Map) 같은 반구조화 데이터를 행으로 펼칠 때 사용합니다.

```sql
-- 배열 데이터를 행으로 펼치기
SELECT
    order_id,
    customer_id,
    EXPLODE(items) AS item
FROM orders;
```

---

## 고급 집계: GROUPING SETS, CUBE, ROLLUP

| 함수 | 생성되는 집계 조합 | 사용 사례 |
|------|-----------------|----------|
| `GROUPING SETS` | 명시적으로 지정한 조합만 | 특정 조합의 소계가 필요할 때 |
| `CUBE` | 모든 가능한 조합 (2^n) | 다차원 분석, 다양한 각도의 집계 |
| `ROLLUP` | 계층적 조합 (n+1) | 시간 계층(연/분기/월), 지역 계층 |

---

## JSON / 반구조화 데이터 처리

Databricks SQL은 JSON 데이터를 **네이티브로 지원**합니다. 콜론(`:`) 구문으로 JSON 필드에 직접 접근할 수 있습니다.

```sql
SELECT
    event_id,
    raw_data:user_id::INT          AS user_id,
    raw_data:event_type::STRING    AS event_type,
    raw_data:metadata.browser::STRING AS browser,
    raw_data:tags[0]::STRING       AS first_tag
FROM raw_events;
```

> 💡 **콜론(`:`) 구문의 장점**: JSON 파싱 함수를 호출할 필요 없이, SQL 컬럼처럼 자연스럽게 JSON 필드에 접근할 수 있습니다. 중첩 필드는 점(`.`)으로, 배열 요소는 대괄호(`[]`)로 접근합니다.

---

## 상세 문서

각 기능별 심화 내용은 아래 문서를 참고하시기 바랍니다.

| 주제 | 문서 |
|------|------|
| 윈도우 함수 심화 | [윈도우 함수](./window-functions.md) |
| SQL 스크립팅 | [SQL 스크립팅](./sql-scripting.md) |
| PIVOT과 UNPIVOT | [PIVOT과 UNPIVOT](./pivot-unpivot.md) |

---

## 정리

| 기능 | 핵심 포인트 |
|------|------------|
| **파이프 구문** | `\|>`로 쿼리를 위에서 아래로 읽을 수 있게 합니다 |
| **CTE / Recursive CTE**| 복잡한 쿼리를 단계별로 분리하고, 계층 데이터를 처리합니다 |
| ** 윈도우 함수**| GROUP BY 없이 순위, 누적, 이동 평균을 계산합니다 |
| **SQL 스크립팅**| 변수, 조건문, 반복문으로 절차적 로직을 SQL로 작성합니다 |
| **PIVOT / UNPIVOT**| 행/열을 변환하여 크로스탭 분석을 수행합니다 |
| **QUALIFY**| 윈도우 함수 결과를 서브쿼리 없이 바로 필터링합니다 |
| **EXPLODE**| 배열, 맵 데이터를 행으로 펼칩니다 |
| **GROUPING SETS / CUBE / ROLLUP**| 여러 집계 조합을 한 번에 계산합니다 |
| **JSON 처리** | 콜론(`:`) 구문으로 JSON 필드에 직접 접근합니다 |

---

## 참고 링크

- [Databricks: SQL language reference](https://docs.databricks.com/aws/en/sql/language-manual/)
- [Databricks: Pipe syntax](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-pipe-syntax.html)
- [Databricks: Window functions](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-window-functions.html)
- [Databricks: SQL scripting](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-scripting.html)
- [Databricks: QUALIFY clause](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify.html)
- [Databricks: Semi-structured data](https://docs.databricks.com/aws/en/query/semi-structured.html)
- [Azure Databricks: SQL reference](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/)
