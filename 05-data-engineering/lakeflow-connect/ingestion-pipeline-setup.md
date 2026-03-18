# 수집 파이프라인 구성

## Unity Catalog Connection 설정

Lakeflow Connect를 사용하려면 먼저 Unity Catalog에 **Connection** 객체를 생성해야 합니다.

```sql
-- MySQL 연결 생성
CREATE CONNECTION mysql_prod
TYPE MYSQL
OPTIONS (
    host = 'mysql-prod.example.com',
    port = '3306',
    user = 'databricks_reader',
    password = secret('scope', 'mysql-password')
);
```

---

## 수집 파이프라인 생성 (SQL)

```sql
-- Lakeflow Connect로 수집 파이프라인 정의
CREATE OR REFRESH STREAMING TABLE bronze_customers
AS SELECT * FROM read_changefeed(
    connection => 'mysql_prod',
    source_schema => 'ecommerce',
    source_table => 'customers'
);

CREATE OR REFRESH STREAMING TABLE bronze_orders
AS SELECT * FROM read_changefeed(
    connection => 'mysql_prod',
    source_schema => 'ecommerce',
    source_table => 'orders'
);
```

---

## 모니터링

| 확인 항목 | 위치 |
|-----------|------|
| 수집 지연 시간 (Latency) | Pipeline UI → 각 테이블의 메트릭 |
| 처리 건수 | Pipeline UI → 업데이트 이력 |
| 에러/경고 | Pipeline UI → 이벤트 로그 |
| 소스-타겟 건수 비교 | 수동 쿼리 또는 시스템 테이블 |

---

## 참고 링크

- [Databricks: Configure Lakeflow Connect](https://docs.databricks.com/aws/en/lakeflow-connect/)
