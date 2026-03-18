# Lakeflow Connect 실습

## 시나리오

운영 중인 MySQL 데이터베이스의 고객 테이블과 주문 테이블을 Databricks 레이크하우스로 실시간 CDC 수집합니다.

---

## Step 1: Connection 생성

Catalog Explorer에서 **External Data** → **Connections** → **Create Connection**으로 UI에서도 생성할 수 있습니다.

## Step 2: 수집 파이프라인 생성

Pipeline UI에서 **Create Pipeline** → **Ingestion (Lakeflow Connect)** 를 선택하면, 코드 없이 GUI로 소스 테이블을 선택하고 대상을 매핑할 수 있습니다.

## Step 3: 수집 후 변환 연결

수집된 Bronze 테이블을 SDP 파이프라인의 입력으로 사용하여 Silver, Gold 계층을 구축합니다.

```sql
-- 별도의 SDP 파이프라인에서 Silver 변환
CREATE OR REFRESH STREAMING TABLE silver_customers;

APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customers)
KEYS (customer_id)
SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 1;
```

---

## 참고 링크

- [Databricks: Lakeflow Connect tutorial](https://docs.databricks.com/aws/en/lakeflow-connect/)
