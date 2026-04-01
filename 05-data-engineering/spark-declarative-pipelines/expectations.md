# 데이터 품질 관리 — Expectations

## 왜 데이터 품질 관리가 필요한가?

데이터 파이프라인에서 가장 흔한 문제는 "**Garbage In, Garbage Out**"입니다. 소스 데이터에 결측값, 잘못된 형식, 비즈니스 규칙 위반 등이 포함되면, 이를 기반으로 만들어진 리포트와 ML 모델의 결과도 신뢰할 수 없게 됩니다.

> 💡 전통적으로 데이터 품질 검증은 파이프라인 코드 곳곳에 `if` 문이나 `filter()` 로직을 수동으로 삽입해야 했습니다. 이 방식은 유지보수가 어렵고, 품질 규칙의 전체 현황을 한눈에 파악하기 어렵습니다. SDP의 **Expectations**는 이 문제를 **선언적(Declarative)** 으로 해결합니다.

| 검증 결과 | 동작 | 설명 |
|-----------|------|------|
| 통과 | 정상 데이터 → 다음 단계 | 데이터가 품질 기준을 충족합니다 |
| WARN | 경고 기록 + 데이터 포함 | 위반 사항을 기록하되 데이터는 통과시킵니다 |
| DROP | 위반 행 제거 (격리 가능) | 위반된 행을 제거합니다 |
| FAIL | 파이프라인 중지 | 파이프라인 실행을 즉시 중지합니다 |

---

## Expectations란?

**Expectations**는 SDP(Spark Declarative Pipelines)에서 데이터 품질 규칙을 **선언적으로** 정의하는 기능입니다. SQL의 `CONSTRAINT`처럼 테이블 정의에 품질 규칙을 직접 명시하며, 각 규칙에 대해 부적합 데이터를 어떻게 처리할지(경고, 삭제, 파이프라인 중지)를 지정할 수 있습니다.

### 핵심 특징

| 특징 | 설명 |
|------|------|
| **선언적 정의** | 테이블 DDL 안에 `CONSTRAINT ... EXPECT (조건)` 형태로 작성합니다 |
| **자동 모니터링** | 매 업데이트마다 통과/위반 건수가 자동으로 기록됩니다 |
| **유연한 처리** | 규칙별로 WARN, DROP ROW, FAIL UPDATE 중 선택할 수 있습니다 |
| **파이프라인 통합** | 별도 도구 없이 SDP 파이프라인 안에서 바로 사용할 수 있습니다 |

---

## 위반 처리 방식 3가지

Expectations는 데이터 품질 위반 시 **세 가지 행동**을 지원합니다. 비즈니스 요구사항에 따라 규칙별로 다르게 설정할 수 있습니다.

### 상세 비교

| 방식 | SQL 키워드 | Python 데코레이터 | 위반 시 동작 | 사용 시나리오 |
|------|-----------|-------------------|-------------|-------------|
| **경고만** | `ON VIOLATION WARN` (기본) | `@dlt.expect("name", "condition")` | 위반 레코드를 **포함**하되, 위반 건수를 기록합니다 | 모니터링 목적, 신규 규칙 테스트 |
| **행 삭제** | `ON VIOLATION DROP ROW` | `@dlt.expect_or_drop("name", "condition")` | 위반 레코드를 결과에서 **제외**합니다 | 데이터 품질 보장이 필요한 Silver/Gold 계층 |
| **파이프라인 중지** | `ON VIOLATION FAIL UPDATE` | `@dlt.expect_or_fail("name", "condition")` | 위반 발생 시 파이프라인을 **즉시 중지**합니다 | 치명적 오류(스키마 위반, 필수값 누락 등) |

> ⚠️ **FAIL UPDATE 사용 시 주의**: 파이프라인이 중지되면 해당 업데이트의 **모든** 테이블 갱신이 중단됩니다. 복구하려면 문제 데이터를 수정한 후 파이프라인을 다시 실행해야 합니다.

### 동작 흐름 비교

**위반 처리 모드별 동작 비교**

| 모드 | 통과 시 | 위반 시 |
|------|--------|---------|
| **WARN** | 테이블에 저장 | 테이블에 저장 + 위반 메트릭 기록 |
| **DROP ROW** | 테이블에 저장 | 레코드 제거 + 위반 메트릭 기록 |
| **FAIL UPDATE** | 테이블에 저장 | 파이프라인 즉시 중지 + 업데이트 실패 |

---

## Expectation 정의 문법

### SQL 문법

SQL에서는 `CREATE OR REFRESH STREAMING TABLE` 또는 `CREATE OR REFRESH MATERIALIZED VIEW` 정의 안에 `CONSTRAINT` 절로 Expectation을 선언합니다.

```sql
CREATE OR REFRESH STREAMING TABLE silver_orders (
    -- 경고만 (모니터링용)
    CONSTRAINT valid_email
        EXPECT (email RLIKE '^[^@]+@[^@]+\\.[^@]+$')
        ON VIOLATION WARN,

    -- 위반 행 삭제 (품질 보장)
    CONSTRAINT valid_order_id
        EXPECT (order_id IS NOT NULL)
        ON VIOLATION DROP ROW,

    CONSTRAINT positive_amount
        EXPECT (amount > 0)
        ON VIOLATION DROP ROW,

    -- 파이프라인 중지 (치명적 오류)
    CONSTRAINT valid_date
        EXPECT (order_date >= '2020-01-01')
        ON VIOLATION FAIL UPDATE
)
AS SELECT * FROM STREAM(bronze_orders);
```

> 💡 `ON VIOLATION WARN`은 기본값이므로, 생략하면 자동으로 WARN이 적용됩니다. 명시적으로 작성하는 것이 가독성에 좋습니다.

### Python 문법

Python에서는 `@dlt.expect`, `@dlt.expect_or_drop`, `@dlt.expect_or_fail` 데코레이터를 사용합니다.

```python
import dlt
from pyspark.sql.functions import col, expr

# 경고만 (WARN)
@dlt.table(comment="정제된 주문 데이터")
@dlt.expect("valid_order_id", "order_id IS NOT NULL")
@dlt.expect("valid_email", "email RLIKE '^[^@]+@[^@]+\\\\.[^@]+$'")
def silver_orders():
    return (
        dlt.read_stream("bronze_orders")
           .select(
               col("order_id").cast("bigint"),
               col("amount").cast("decimal(12,2)"),
               col("order_date").cast("timestamp"),
               col("email")
           )
    )

# 행 삭제 (DROP)
@dlt.table()
@dlt.expect_or_drop("positive_amount", "amount > 0")
@dlt.expect_or_drop("valid_status", "status IN ('pending', 'completed', 'cancelled')")
def silver_orders_clean():
    return dlt.read_stream("silver_orders")

# 파이프라인 중지 (FAIL)
@dlt.table()
@dlt.expect_or_fail("valid_date", "order_date >= '2020-01-01'")
def silver_orders_strict():
    return dlt.read_stream("silver_orders")
```

---

## 복합 조건 Expectation

하나의 Expectation 안에서 여러 조건을 조합할 수 있습니다. SQL 표준 논리 연산자(`AND`, `OR`, `NOT`)를 사용합니다.

### SQL 복합 조건 예제

```sql
CREATE OR REFRESH STREAMING TABLE silver_transactions (
    -- 여러 조건을 AND로 결합
    CONSTRAINT valid_transaction
        EXPECT (
            transaction_id IS NOT NULL
            AND amount > 0
            AND amount < 1000000
            AND currency IN ('USD', 'EUR', 'KRW', 'JPY')
        )
        ON VIOLATION DROP ROW,

    -- OR 조건으로 유연한 검증
    CONSTRAINT valid_contact
        EXPECT (
            email IS NOT NULL
            OR phone IS NOT NULL
        )
        ON VIOLATION WARN,

    -- CASE 표현식을 활용한 조건부 검증
    CONSTRAINT valid_discount
        EXPECT (
            CASE
                WHEN customer_type = 'premium' THEN discount <= 0.30
                WHEN customer_type = 'regular' THEN discount <= 0.15
                ELSE discount <= 0.05
            END
        )
        ON VIOLATION DROP ROW
)
AS SELECT * FROM STREAM(bronze_transactions);
```

### Python 복합 조건 예제 (expect_all)

여러 Expectation을 딕셔너리로 한 번에 정의할 수 있습니다.

```python
import dlt

# 여러 Expectation을 딕셔너리로 정의
order_expectations = {
    "valid_order_id": "order_id IS NOT NULL",
    "positive_amount": "amount > 0 AND amount < 1000000",
    "valid_status": "status IN ('pending', 'completed', 'cancelled')",
    "valid_date": "order_date >= '2020-01-01'"
}

# expect_all: 모든 규칙에 WARN 적용
@dlt.table()
@dlt.expect_all(order_expectations)
def silver_orders_monitored():
    return dlt.read_stream("bronze_orders")

# expect_all_or_drop: 모든 규칙에 DROP 적용
@dlt.table()
@dlt.expect_all_or_drop(order_expectations)
def silver_orders_clean():
    return dlt.read_stream("bronze_orders")

# expect_all_or_fail: 모든 규칙에 FAIL 적용
@dlt.table()
@dlt.expect_all_or_fail(order_expectations)
def silver_orders_strict():
    return dlt.read_stream("bronze_orders")
```

---

## 고급 패턴

### 1. 데이터 격리(Quarantine) 패턴

위반 데이터를 단순히 삭제하는 대신, **별도 테이블에 격리**하여 나중에 분석하거나 수정할 수 있습니다. 이 패턴은 "어떤 데이터가 왜 탈락했는지"를 추적해야 할 때 매우 유용합니다.

```sql
-- 1단계: 모든 데이터를 일단 받되, 품질 규칙은 WARN으로 설정
CREATE OR REFRESH STREAMING TABLE bronze_orders_validated (
    CONSTRAINT valid_order_id
        EXPECT (order_id IS NOT NULL)
        ON VIOLATION WARN,
    CONSTRAINT positive_amount
        EXPECT (amount > 0)
        ON VIOLATION WARN,
    CONSTRAINT valid_email
        EXPECT (email RLIKE '^[^@]+@[^@]+\\.[^@]+$')
        ON VIOLATION WARN
)
AS SELECT * FROM STREAM(raw_orders);

-- 2단계: 정상 데이터만 Silver로 이동
CREATE OR REFRESH STREAMING TABLE silver_orders_clean
AS SELECT *
FROM STREAM(bronze_orders_validated)
WHERE order_id IS NOT NULL
  AND amount > 0
  AND email RLIKE '^[^@]+@[^@]+\\.[^@]+$';

-- 3단계: 위반 데이터를 격리 테이블로 이동
CREATE OR REFRESH STREAMING TABLE quarantine_orders
AS SELECT
    *,
    CASE
        WHEN order_id IS NULL THEN 'missing_order_id'
        WHEN amount <= 0 THEN 'invalid_amount'
        WHEN NOT email RLIKE '^[^@]+@[^@]+\\.[^@]+$' THEN 'invalid_email'
    END AS violation_reason,
    current_timestamp() AS quarantined_at
FROM STREAM(bronze_orders_validated)
WHERE order_id IS NULL
   OR amount <= 0
   OR NOT email RLIKE '^[^@]+@[^@]+\\.[^@]+$';
```

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | Bronze (원본 데이터) | 원본 데이터 소스입니다 |
| 2 | Expectations (WARN) | 품질 검증을 수행합니다 |
| 3a | Silver (정상 데이터) | 검증을 통과한 데이터가 저장됩니다 |
| 3b | Quarantine (위반 데이터) | 위반 데이터를 격리합니다 |
| 4 | 수동 검토 / 자동 수정 | 격리된 데이터를 검토하고 수정하여 Silver에 재투입합니다 |

### 2. 계층별 차등 적용 패턴

Medallion 아키텍처의 각 계층에 서로 다른 엄격도를 적용하는 것이 좋습니다.

| 계층 | 권장 방식 | 이유 |
|------|----------|------|
| **Bronze** | `WARN` 또는 Expectation 없음 | 원본 데이터를 최대한 보존합니다 |
| **Silver** | `DROP ROW` | 비즈니스 로직에 필요한 최소 품질을 보장합니다 |
| **Gold** | `FAIL UPDATE` 또는 `DROP ROW` | 리포트/ML에 사용되므로 엄격한 품질이 필요합니다 |

### 3. 동적 Expectation (테이블 기반 규칙)

품질 규칙을 코드에 하드코딩하지 않고 외부 테이블에서 관리할 수 있습니다.

```python
import dlt
from pyspark.sql import SparkSession

def get_rules_for_table(table_name):
    """외부 규칙 테이블에서 해당 테이블의 품질 규칙을 조회합니다."""
    spark = SparkSession.builder.getOrCreate()
    rules = spark.sql(f"""
        SELECT rule_name, rule_expression
        FROM catalog.schema.data_quality_rules
        WHERE target_table = '{table_name}'
          AND is_active = true
    """).collect()
    return {row.rule_name: row.rule_expression for row in rules}

@dlt.table()
@dlt.expect_all_or_drop(get_rules_for_table("silver_orders"))
def silver_orders():
    return dlt.read_stream("bronze_orders")
```

---

## Expectation 결과 모니터링

### Pipeline UI에서 확인

Expectations의 결과는 SDP Pipeline UI에서 시각적으로 확인할 수 있습니다. 각 규칙별로 통과/위반 건수, 위반율이 실시간으로 표시됩니다.

| 메트릭 | 설명 |
|--------|------|
| **통과 건수** | 규칙을 만족한 레코드 수 |
| **위반 건수** | 규칙을 위반한 레코드 수 |
| **위반율** | 위반 건수 / 전체 건수 (%) |
| **처리 방식** | WARN / DROP / FAIL 중 어떤 방식이 적용되었는지 |

### 이벤트 로그로 프로그래밍 방식 조회

이벤트 로그를 SQL로 직접 조회하여 품질 메트릭을 분석하거나 외부 모니터링 도구에 연동할 수 있습니다.

```sql
-- 파이프라인 이벤트 로그에서 Expectation 결과 조회
SELECT
    timestamp,
    details:flow_definition.output_dataset AS dataset_name,
    details:flow_progress.data_quality.expectations AS expectations
FROM event_log(TABLE(my_catalog.my_schema.my_pipeline))
WHERE event_type = 'flow_progress'
  AND details:flow_progress.data_quality IS NOT NULL
ORDER BY timestamp DESC;
```

결과 예시:

```json
{
  "expectations": [
    {
      "name": "valid_order_id",
      "dataset": "silver_orders",
      "passed_records": 9850,
      "failed_records": 150
    },
    {
      "name": "positive_amount",
      "dataset": "silver_orders",
      "passed_records": 9990,
      "failed_records": 10
    }
  ]
}
```

### 품질 대시보드 구축

이벤트 로그 데이터를 활용하여 데이터 품질 추이를 시각화할 수 있습니다.

```sql
-- 일별 데이터 품질 추이 집계
CREATE OR REFRESH MATERIALIZED VIEW data_quality_daily_summary
AS
SELECT
    DATE(timestamp) AS check_date,
    details:flow_definition.output_dataset AS dataset_name,
    exp.name AS expectation_name,
    SUM(exp.passed_records) AS total_passed,
    SUM(exp.failed_records) AS total_failed,
    ROUND(
        SUM(exp.failed_records) * 100.0 /
        NULLIF(SUM(exp.passed_records) + SUM(exp.failed_records), 0),
        2
    ) AS failure_rate_pct
FROM event_log(TABLE(my_catalog.my_schema.my_pipeline)),
     LATERAL VIEW EXPLODE(
         from_json(
             details:flow_progress.data_quality.expectations,
             'ARRAY<STRUCT<name:STRING, passed_records:BIGINT, failed_records:BIGINT>>'
         )
     ) AS exp
WHERE event_type = 'flow_progress'
GROUP BY 1, 2, 3
ORDER BY check_date DESC, failure_rate_pct DESC;
```

---

## 모범 사례 및 안티패턴

### 모범 사례

| 항목 | 권장 사항 |
|------|----------|
| **규칙 이름** | 의미 있는 이름을 사용합니다 (예: `valid_email`, `positive_amount`) |
| **계층별 차등 적용** | Bronze → WARN, Silver → DROP, Gold → FAIL 순으로 엄격도를 높입니다 |
| **점진적 도입** | 신규 규칙은 먼저 WARN으로 배포하여 영향 범위를 파악한 후 DROP/FAIL로 전환합니다 |
| **복합 규칙 분리** | 하나의 규칙에 너무 많은 조건을 넣지 말고, 위반 원인을 추적하기 쉽게 분리합니다 |
| **격리 패턴 활용** | DROP 대신 Quarantine 패턴을 사용하여 위반 데이터의 원인을 분석할 수 있게 합니다 |
| **모니터링 자동화** | 이벤트 로그를 정기 조회하여 품질 추이를 모니터링합니다 |

### 안티패턴

| 안티패턴 | 문제점 | 개선 방법 |
|---------|--------|----------|
| 모든 규칙에 `FAIL UPDATE` 사용 | 사소한 위반에도 파이프라인이 중단됩니다 | 치명적 오류만 FAIL, 나머지는 DROP 또는 WARN 사용 |
| Bronze에서 `DROP ROW` 사용 | 원본 데이터가 유실되어 복구할 수 없습니다 | Bronze는 WARN만 사용하고, Silver에서 필터링합니다 |
| 규칙 이름을 `rule1`, `rule2`로 지정 | 위반 알림에서 어떤 규칙인지 파악할 수 없습니다 | 규칙의 의미를 설명하는 이름을 사용합니다 |
| 하나의 규칙에 10개 이상 AND 조건 | 어떤 조건이 위반되었는지 특정할 수 없습니다 | 조건별로 별도 규칙으로 분리합니다 |
| Expectation만 의존하고 후속 모니터링 없음 | 품질 저하 추세를 감지할 수 없습니다 | 이벤트 로그 기반 대시보드를 구축합니다 |

---

## 정리

| 핵심 포인트 | 설명 |
|------------|------|
| **Expectations의 목적** | 데이터 품질 규칙을 선언적으로 정의하여 "Garbage In, Garbage Out"을 방지합니다 |
| **3가지 처리 방식** | WARN(모니터링), DROP ROW(필터링), FAIL UPDATE(파이프라인 중지) |
| **계층별 전략** | Bronze는 WARN, Silver는 DROP, Gold는 FAIL로 점진적으로 엄격하게 적용합니다 |
| **격리 패턴** | 위반 데이터를 별도 테이블에 격리하여 원인 분석과 수정을 가능하게 합니다 |
| **모니터링** | Pipeline UI와 이벤트 로그를 활용하여 품질 추이를 지속적으로 추적합니다 |

---

## 참고 링크

- [Databricks: Manage data quality with expectations](https://docs.databricks.com/aws/en/sdp/expectations.html)
- [Databricks: Monitor SDP pipelines](https://docs.databricks.com/aws/en/sdp/observability.html)
- [Databricks: SDP event log](https://docs.databricks.com/aws/en/sdp/observability.html#event-log)
- [Databricks Blog: Data quality best practices](https://www.databricks.com/blog/applying-software-development-devops-best-practices-delta-live-tables)
