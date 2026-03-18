# CDC 처리 — APPLY CHANGES

## CDC란?

> 💡 **CDC(Change Data Capture)**는 소스 데이터베이스에서 발생하는 변경(INSERT, UPDATE, DELETE)을 실시간으로 감지하여 타겟 시스템에 반영하는 기술입니다.

SDP에서는 **APPLY CHANGES** 구문을 사용하여 CDC 데이터를 처리합니다.

---

## SCD Type 1: 최신 값으로 덮어쓰기

변경이 발생하면 기존 값을 **최신 값으로 교체**합니다. 이력을 보존하지 않습니다.

```sql
-- 타겟 Streaming Table 생성
CREATE OR REFRESH STREAMING TABLE silver_customers;

-- CDC 적용 (SCD Type 1)
APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS * EXCEPT (_rescued_data)
STORED AS SCD TYPE 1;
```

---

## SCD Type 2: 변경 이력 보존

변경이 발생하면 기존 레코드를 만료 처리하고 **새 레코드를 추가**합니다. 전체 변경 이력이 보존됩니다.

```sql
CREATE OR REFRESH STREAMING TABLE silver_customers_history;

APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
COLUMNS * EXCEPT (_rescued_data)
STORED AS SCD TYPE 2;
```

SCD Type 2로 저장되면 자동으로 다음 컬럼이 추가됩니다.

| 컬럼 | 설명 |
|------|------|
| `__START_AT` | 해당 버전의 유효 시작 시간 |
| `__END_AT` | 해당 버전의 유효 종료 시간 (NULL이면 현재 유효) |

---

## DELETE 처리

소스에서 삭제된 레코드를 반영하려면 `APPLY AS DELETE WHEN` 조건을 추가합니다.

```sql
APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customer_cdc)
KEYS (customer_id)
SEQUENCE BY updated_at
APPLY AS DELETE WHEN operation = 'DELETE'
STORED AS SCD TYPE 1;
```

---

## 참고 링크

- [Databricks: APPLY CHANGES](https://docs.databricks.com/aws/en/sdp/cdc.html)
