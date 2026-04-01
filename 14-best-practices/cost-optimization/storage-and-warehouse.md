# 스토리지와 SQL Warehouse 최적화

## 3. 스토리지 최적화

### 3.1 Liquid Clustering으로 스캔량 감소

Liquid Clustering은 테이블 데이터를 쿼리 패턴에 맞게 자동으로 재배치하여, 불필요한 데이터 스캔을 대폭 줄입니다.

```sql
-- Liquid Clustering 적용 (새 테이블)
CREATE TABLE sales.gold.daily_revenue (
  sale_date DATE,
  region STRING,
  product_category STRING,
  revenue DECIMAL(18,2)
)
CLUSTER BY (sale_date, region);

-- 기존 테이블에 Liquid Clustering 추가
ALTER TABLE sales.gold.daily_revenue
CLUSTER BY (sale_date, region);

-- 자동 키 선택 (Databricks가 쿼리 패턴 분석 후 최적 키 결정)
ALTER TABLE sales.gold.daily_revenue
CLUSTER BY AUTO;
```

| 방식 | 스캔 데이터량 (1TB 테이블, 특정 날짜 조회) | 상대 비용 |
|------|----------------------------------------|----------|
| 클러스터링 없음 | ~1TB (풀 스캔) | ★★★★★ |
| 파티션 (날짜) | ~3GB (1일치) | ★☆☆☆☆ |
| Liquid Clustering (날짜) | ~3GB (1일치) + 점진적 최적화 | ★☆☆☆☆ |

### 3.2 VACUUM / OPTIMIZE 주기 최적화

```sql
-- OPTIMIZE: 작은 파일들을 병합하여 읽기 성능 향상
OPTIMIZE catalog.schema.table_name;

-- VACUUM: 더 이상 참조되지 않는 오래된 파일 삭제 (스토리지 비용 절감)
VACUUM catalog.schema.table_name RETAIN 168 HOURS;  -- 7일 보존

-- 테이블 통계 재수집 (쿼리 옵티마이저 성능 향상)
ANALYZE TABLE catalog.schema.table_name COMPUTE DELTA STATISTICS;
```

| 테이블 유형 | OPTIMIZE 주기 | VACUUM 주기 | 비고 |
|-----------|-------------|------------|------|
| 실시간 스트리밍 (분 단위) | 매 1-4시간 | 매일 | Small File 문제 빈번 |
| 일일 배치 | 매일 (배치 후) | 주 1회 | 배치 완료 직후 실행 |
| 주간/월간 배치 | 배치 후 | 월 1회 | 빈도 낮으면 수동 OK |
| 읽기 전용 (분석용) | 월 1회 | 분기 1회 | 변경 없으면 최소화 |

### 3.3 Predictive Optimization 활용

Predictive Optimization은 Unity Catalog 관리형 테이블에 대해 OPTIMIZE, VACUUM, ANALYZE를 **자동으로 실행** 합니다.

```sql
-- 카탈로그 수준에서 Predictive Optimization 활성화
ALTER CATALOG my_catalog ENABLE PREDICTIVE OPTIMIZATION;

-- 스키마 수준에서 활성화
ALTER SCHEMA my_catalog.my_schema ENABLE PREDICTIVE OPTIMIZATION;

-- 특정 스키마만 비활성화 (비용 민감한 경우)
ALTER SCHEMA my_catalog.temp_schema DISABLE PREDICTIVE OPTIMIZATION;
```

> 💡 **비용 효과**: Predictive Optimization을 활성화하면 수동 OPTIMIZE/VACUUM 작업을 관리할 필요가 없어지고, 서버리스 Jobs SKU로 과금되어 효율적입니다. 2024년 11월 이후 생성된 계정에는 기본 활성화되어 있습니다.

---

## 4. SQL Warehouse 최적화

### 4.1 Warehouse 사이징 가이드

| 동시 사용자 | 쿼리 복잡도 | 권장 크기 | 클러스터 수 |
|-----------|-----------|----------|-----------|
| 1-5명 | 간단한 조회 | Small | 1 |
| 5-15명 | 중간 복잡도 | Medium | 1-2 |
| 15-50명 | 복잡한 조인/집계 | Large | 2-5 |
| 50-200명 | 다양한 혼합 | X-Large | 5-10 |
| 200명+ | 대시보드 + Ad-hoc | 2X-Large | 10+ |

> 💡 **Serverless 전략**: Serverless SQL Warehouse를 사용한다면 ** 큰 단일 Warehouse에서 시작**하는 것이 권장됩니다. Serverless의 Intelligent Workload Management가 자동으로 동시성을 관리합니다.

### 4.2 Auto Stop vs Always On 비용 비교

```text
시나리오: Medium Warehouse, 하루 8시간 업무 시간 중 실제 쿼리 4시간

[Always On]
  24시간 × 365일 = 8,760시간/년 과금

[Auto Stop 10분]
  ~4.5시간/일 × 250 업무일 = 1,125시간/년 과금
  → 약 87% 비용 절감!

[Serverless + Auto Stop 5분]
  ~4.2시간/일 × 250 업무일 = 1,050시간/년
  → 즉시 시작으로 사용자 경험 유지 + 비용 최적화
```

### 4.3 Result Cache와 Materialized View 활용

```sql
-- Materialized View로 반복 쿼리 비용 제거
CREATE MATERIALIZED VIEW sales.gold.monthly_summary AS
SELECT
  date_trunc('month', sale_date) AS month,
  region,
  SUM(revenue) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers
FROM sales.silver.transactions
GROUP BY 1, 2;

-- Materialized View는 자동으로 증분 갱신됩니다
-- 동일한 쿼리를 100명이 실행해도 사전 계산된 결과를 반환
```

| 캐싱 전략 | 적용 대상 | 효과 | 비용 영향 |
|----------|----------|------|----------|
| **Result Cache** | 동일 쿼리 반복 | 재계산 없이 즉시 반환 | 자동 적용, 무료 |
| **Disk Cache** | 자주 접근하는 데이터 | 리모트 스토리지 읽기 제거 | 로컬 SSD 필요 |
| **Materialized View** | 복잡한 집계 쿼리 | 사전 계산 + 증분 갱신 | 저장 비용 발생, 컴퓨트 비용 대폭 절감 |

---
