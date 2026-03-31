# 데이터 리니지

## 리니지란?

> 💡 **데이터 리니지(Data Lineage)** 란 데이터가 **어디서 왔고(upstream), 어디로 가는지(downstream)** 를 추적하는 기능입니다. "이 테이블의 데이터는 어떤 소스에서 왔지?", "이 컬럼을 변경하면 어디에 영향이 가지?"라는 질문에 답할 수 있습니다.

---

## 왜 리니지가 중요한가요?

| 사용 사례 | 설명 |
|-----------|------|
| **영향도 분석** | 소스 테이블의 스키마를 변경할 때, 어떤 downstream 테이블과 대시보드에 영향이 있는지 확인합니다 |
| **데이터 품질 추적** | Gold 테이블에 이상이 발견되면, 어느 단계(Bronze? Silver?)에서 문제가 생겼는지 역추적합니다 |
| **규정 준수** | 개인정보(PII)가 어떤 경로로 흘러가는지 파악하여 GDPR, 개인정보보호법 등을 준수합니다 |
| **문서화 자동화** | 수동으로 데이터 흐름을 문서화하는 대신, 자동으로 추적됩니다 |
| **마이그레이션 계획** | 테이블을 변경/삭제할 때, 영향 받는 모든 객체를 사전에 파악합니다 |

---

## 리니지의 범위

Unity Catalog는 **테이블 수준**과 **컬럼 수준** 리니지를 모두 자동으로 추적합니다.

### 테이블 수준 리니지

| 단계 | 소스/대상 | 설명 |
|------|-----------|------|
| 소스 | MySQL (외부 소스) | 원본 데이터입니다 |
| Bronze | bronze_orders | 원본 데이터를 수집합니다 |
| Silver | silver_orders | 정제된 데이터입니다 |
| Gold | gold_daily_revenue | 일별 매출 집계 → 매출 대시보드에 활용 |
| Gold | gold_customer_360 | 고객 통합 뷰 → 고객 분석 대시보드 및 이탈 예측 모델에 활용 |

### 컬럼 수준 리니지

```
silver_orders.total_amount
  ← bronze_orders.raw_data:amount (JSON에서 추출)

gold_daily_revenue.total_revenue
  ← silver_orders.total_amount (SUM 집계)

gold_customer_360.lifetime_revenue
  ← silver_orders.total_amount (고객별 SUM)
```

---

## 리니지 확인 방법

### Catalog Explorer UI

1. **Catalog** → 테이블 선택
2. **Lineage** 탭 클릭
3. Upstream(이 테이블의 소스)과 Downstream(이 테이블을 참조하는 객체)을 시각적으로 확인합니다

### 시스템 테이블로 SQL 조회

```sql
-- 특정 테이블의 downstream (이 테이블을 참조하는 모든 객체)
SELECT
    target_table_full_name AS downstream_table,
    target_type,
    target_column_name
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
ORDER BY target_table_full_name;

-- 특정 테이블의 upstream (이 테이블이 참조하는 소스)
SELECT
    source_table_full_name AS upstream_table,
    source_type,
    source_column_name
FROM system.lineage.table_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue'
ORDER BY source_table_full_name;

-- 컬럼 수준 리니지: total_revenue가 어디서 왔는지
SELECT
    source_table_full_name,
    source_column_name,
    target_column_name
FROM system.lineage.column_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue'
    AND target_column_name = 'total_revenue';

-- PII 컬럼의 전파 경로 추적
SELECT
    source_table_full_name,
    source_column_name,
    target_table_full_name,
    target_column_name
FROM system.lineage.column_lineage
WHERE source_column_name IN ('email', 'phone', 'ssn', 'name')
ORDER BY source_table_full_name, target_table_full_name;
```

---

## 리니지가 추적되는 작업

| 작업 유형 | 추적 여부 | 설명 |
|-----------|----------|------|
| **SQL 쿼리** (CTAS, INSERT, MERGE) | ✅ | SQL Warehouse, Notebook에서 실행한 쿼리 |
| **SDP Pipeline** | ✅ | Streaming Table, Materialized View의 소스 추적 |
| **Spark DataFrame** | ✅ | `df.write.saveAsTable()` 등 |
| **MLflow 모델** | ✅ | 모델이 어떤 Feature Table에서 학습되었는지 |
| **AI/BI Dashboard** | ✅ | 대시보드가 어떤 테이블을 조회하는지 |
| **외부 도구 (JDBC)** | ⚠️ 부분적 | SQL Warehouse를 통한 외부 BI 도구 접근 |

---

## 리니지 활용 시나리오

### 시나리오 1: 스키마 변경 영향도 분석

`silver_orders` 테이블의 `amount` 컬럼을 `total_amount`로 이름을 변경하려 합니다.

```sql
-- 이 컬럼을 참조하는 모든 downstream 확인
SELECT DISTINCT
    target_table_full_name,
    target_column_name
FROM system.lineage.column_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
    AND source_column_name = 'amount';

-- 결과:
-- gold_daily_revenue.total_revenue
-- gold_customer_360.lifetime_revenue
-- 매출 대시보드 (dataset query)
-- → 이 3곳도 함께 변경해야 합니다
```

### 시나리오 2: 데이터 품질 이슈 역추적

`gold_daily_revenue` 테이블에서 특정 날짜의 매출이 비정상적으로 낮습니다.

```sql
-- upstream 추적: 이 테이블의 소스 확인
SELECT source_table_full_name
FROM system.lineage.table_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue';
-- → silver_orders

-- silver_orders의 upstream 확인
SELECT source_table_full_name
FROM system.lineage.table_lineage
WHERE target_table_full_name = 'production.ecommerce.silver_orders';
-- → bronze_orders

-- → bronze_orders에서 해당 날짜의 원본 데이터를 확인하여 원인 파악
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **리니지** | 데이터의 출처(upstream)와 행선지(downstream)를 추적합니다 |
| **테이블 리니지** | 테이블 간의 데이터 흐름을 추적합니다 |
| **컬럼 리니지** | 컬럼 간의 데이터 흐름을 추적합니다 |
| **자동 추적** | SQL, Spark, SDP 등의 작업에서 자동으로 추적됩니다 |
| **시스템 테이블** | `system.lineage.table_lineage`, `column_lineage`로 SQL 조회가 가능합니다 |

---

## 참고 링크

- [Databricks: Data lineage](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-lineage.html)
- [Azure Databricks: Data lineage](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/data-lineage)
