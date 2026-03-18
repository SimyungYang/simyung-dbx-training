# 데이터 품질 관리 — Expectations

## Expectations란?

> 💡 **Expectations**는 SDP에서 데이터 품질 규칙을 **선언적으로** 정의하는 기능입니다. 각 규칙에 대해 부적합 데이터를 어떻게 처리할지(경고, 삭제, 파이프라인 중지)를 지정할 수 있습니다.

---

## 위반 처리 방식

| 방식 | 키워드 | 동작 |
|------|--------|------|
| **경고만** | `ON VIOLATION WARN` (기본) | 위반 레코드를 포함하되, 위반 건수를 기록합니다 |
| **행 삭제** | `ON VIOLATION DROP ROW` | 위반 레코드를 결과에서 제외합니다 |
| **파이프라인 중지** | `ON VIOLATION FAIL UPDATE` | 위반 발생 시 파이프라인을 즉시 중지합니다 |

---

## 사용 예시

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

---

## 품질 모니터링

Expectations의 결과는 Pipeline UI에서 시각적으로 확인할 수 있습니다. 각 규칙별로 통과/위반 건수, 위반율이 표시되어, 데이터 품질을 지속적으로 모니터링할 수 있습니다.

| 메트릭 | 설명 |
|--------|------|
| **통과 건수** | 규칙을 만족한 레코드 수 |
| **위반 건수** | 규칙을 위반한 레코드 수 |
| **위반율** | 위반 건수 / 전체 건수 |

---

## 참고 링크

- [Databricks: Manage data quality with expectations](https://docs.databricks.com/aws/en/sdp/expectations.html)
