# 테이블과 뷰

## 테이블 유형

| 유형 | 설명 | 데이터 관리 |
|------|------|-----------|
| **Managed Table** | Unity Catalog가 데이터와 메타데이터를 모두 관리합니다 | 테이블 삭제 시 데이터도 삭제 |
| **External Table** | 메타데이터만 관리하고, 데이터는 외부 경로에 위치합니다 | 테이블 삭제 시 데이터는 유지 |

```sql
-- Managed Table (권장)
CREATE TABLE catalog.schema.orders (
    order_id BIGINT,
    amount DECIMAL(10,2)
) USING DELTA;

-- External Table
CREATE TABLE catalog.schema.external_orders (
    order_id BIGINT,
    amount DECIMAL(10,2)
) USING DELTA
LOCATION 's3://my-bucket/external/orders/';
```

---

## 뷰 유형

| 유형 | 설명 | 데이터 저장 |
|------|------|-----------|
| **View** | SQL 쿼리의 별칭. 매번 실행 시 쿼리가 다시 실행됩니다 | ❌ 저장하지 않음 |
| **Materialized View** | 쿼리 결과를 물리적으로 저장합니다. 주기적으로 갱신됩니다 | ✅ 물리적 저장 |
| **Temporary View** | 세션 동안만 유효한 임시 뷰입니다 | ❌ 세션 종료 시 삭제 |
| **Streaming Table** | 증분 처리 기반의 추가 전용 테이블입니다 | ✅ 물리적 저장 |

```sql
-- 일반 뷰
CREATE VIEW catalog.schema.v_active_customers AS
SELECT * FROM customers WHERE status = 'ACTIVE';

-- Materialized View (결과를 물리적으로 저장)
CREATE MATERIALIZED VIEW catalog.schema.mv_daily_revenue AS
SELECT DATE(order_date) AS day, SUM(amount) AS revenue
FROM orders GROUP BY 1;
```

---

## 참고 링크

- [Databricks: Tables](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-tables.html)
