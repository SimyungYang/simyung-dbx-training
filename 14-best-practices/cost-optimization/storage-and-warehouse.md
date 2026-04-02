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

> 💡 **Serverless 전략**: Serverless SQL Warehouse를 사용한다면 **큰 단일 Warehouse에서 시작** 하는 것이 권장됩니다. Serverless의 Intelligent Workload Management가 자동으로 동시성을 관리합니다.

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

## 5. SQL Warehouse T-shirt 사이즈별 상세 가이드

### 5.1 사이즈별 스펙과 용도

| 사이즈 | 클러스터 노드 수 | 대략적 코어/메모리 | 적합한 용도 | 시간당 DBU |
|-------|-------------|---------------|-----------|----------|
| **2X-Small** | 1 | 4 코어, 16GB | 단일 사용자 테스트 | ~2 DBU |
| **X-Small** | 1 | 8 코어, 32GB | 소규모 대시보드 (1-3명) | ~4 DBU |
| **Small** | 2 | 16 코어, 64GB | 소규모 팀 (5명 이하) | ~8 DBU |
| **Medium** | 4 | 32 코어, 128GB | 중규모 팀 (5-15명) | ~16 DBU |
| **Large** | 8 | 64 코어, 256GB | 대규모 팀 + 복잡 쿼리 | ~32 DBU |
| **X-Large** | 16 | 128 코어, 512GB | 대규모 동시 접속 | ~64 DBU |
| **2X-Large** | 32 | 256 코어, 1TB | 엔터프라이즈 대시보드 | ~128 DBU |

{% hint style="info" %}
**사이즈 선택 팁**: Small에서 시작하여 Query History의 대기 시간(queued time)을 모니터링하세요. 대기 시간이 빈번하게 발생하면 사이즈를 올리거나 클러스터 수를 늘리세요.
{% endhint %}

### 5.2 사이즈 결정 의사결정 트리

```text
Q1: 동시 사용자가 10명 이상인가?
  → No: Small 또는 Medium
  → Yes: Q2로

Q2: 대부분의 쿼리가 1TB 이상 테이블을 스캔하는가?
  → No: Medium (클러스터 수 2-3)
  → Yes: Large 이상

Q3: Serverless를 사용하는가?
  → Yes: Medium-Large 단일 Warehouse (IWM이 자동 관리)
  → No: 여러 Warehouse 분리 전략 필요
```

---

## 6. 오토스케일링 전략

### 6.1 Classic SQL Warehouse 오토스케일링

| 설정 | 설명 | 권장값 |
|------|------|-------|
| **Min clusters** | 항상 유지하는 최소 클러스터 수 | 업무 시간: 1, 비업무: 0 |
| **Max clusters** | 최대 확장 가능 클러스터 수 | 동시 사용자 / 5 (경험값) |
| **Auto Stop** | 유휴 시 자동 종료 | 5-10분 |
| **Scaling Policy** | 확장 속도 | Queue 기반 (기본값) |

```text
[오토스케일링 시나리오]

시나리오: 대시보드 50명 동시 접속 + Ad-hoc 분석 10명

권장 설정:
  Size: Medium
  Min clusters: 1 (항상 1개 유지)
  Max clusters: 10 (50명 기준)
  Auto Stop: 10분

동작:
  09:00 - 업무 시작, 1개 클러스터로 시작
  09:30 - 대시보드 접속 증가, 3개로 스케일 아웃
  10:00 - 피크 시간, 7개 클러스터
  12:00 - 점심시간, 2개로 스케일 인
  18:00 - 퇴근, 1개로 축소
  18:10 - Auto Stop, 0개로 종료
```

### 6.2 Serverless SQL Warehouse IWM (Intelligent Workload Management)

Serverless SQL Warehouse는 **IWM** 이 자동으로 워크로드를 관리합니다.

| IWM 기능 | 설명 | 효과 |
|---------|------|------|
| **자동 스케일링** | 쿼리 큐 길이에 따라 자동 확장/축소 | 설정 불필요 |
| **쿼리 우선순위** | 인터랙티브 쿼리 > 배치 쿼리 | 대시보드 응답성 보장 |
| **리소스 격리** | 무거운 쿼리가 가벼운 쿼리를 방해하지 않음 | 동시성 보장 |
| **쿼리 타임아웃** | 장시간 실행 쿼리 자동 종료 | 리소스 낭비 방지 |

{% hint style="info" %}
**Serverless 권장 전략**: Serverless SQL Warehouse에서는 여러 작은 Warehouse로 분리하지 말고, **하나의 큰 Warehouse** 를 사용하세요. IWM이 자동으로 워크로드를 분배하므로, 분리하면 오히려 리소스 활용률이 낮아집니다.
{% endhint %}

---

## 7. Query Caching 최적화

### 7.1 Result Cache 히트율 극대화

```sql
-- Result Cache가 무효화되는 경우
-- 1. 테이블 데이터가 변경됨 (INSERT, UPDATE, DELETE, MERGE)
-- 2. 비결정적 함수 사용 (CURRENT_TIMESTAMP(), RAND(), UUID())
-- 3. 쿼리 텍스트가 다름 (공백, 대소문자 차이도 미스)

-- ✅ Cache 친화적 쿼리 패턴
SELECT region, SUM(revenue) AS total
FROM sales.gold.daily_revenue
WHERE sale_date BETWEEN '2026-01-01' AND '2026-03-31'
GROUP BY region;

-- ❌ Cache 비친화적 쿼리 패턴 (매번 다른 결과)
SELECT region, SUM(revenue) AS total
FROM sales.gold.daily_revenue
WHERE sale_date >= current_date() - INTERVAL 90 DAYS  -- 매일 다른 결과
GROUP BY region;
```

| Cache 최적화 전략 | 설명 | 효과 |
|----------------|------|------|
| **고정 날짜 범위 사용** | `WHERE date BETWEEN '2026-01-01' AND '2026-03-31'` | 동일 쿼리 = Cache 히트 |
| **파라미터화된 쿼리** | 대시보드 필터는 동일 구조 유지 | 필터값이 같으면 히트 |
| **MV 활용** | Materialized View로 사전 계산 | Cache와 무관하게 빠름 |
| **비결정적 함수 제거** | `current_date()` 대신 고정값 | Cache 무효화 방지 |

### 7.2 Disk Cache 활용

```sql
-- Disk Cache는 자주 접근하는 데이터를 로컬 SSD에 저장
-- Classic SQL Warehouse에서 자동 동작
-- 확인 방법: Query Profile에서 "bytes read from cache" 지표

-- Disk Cache를 극대화하려면:
-- 1. Storage-optimized 인스턴스 사용 (로컬 SSD 보유)
-- 2. Warehouse 재시작 최소화 (재시작 시 Cache 초기화)
-- 3. 자주 조인하는 디멘션 테이블은 캐시에 자동 로드
```

---

## 8. 멀티 Warehouse 설계 패턴

### 8.1 용도별 분리

| Warehouse 이름 | 용도 | 사이즈 | Auto Stop | 사용자 |
|--------------|------|-------|----------|-------|
| **wh-dashboard** | 대시보드 서빙 | Medium | 10분 | 전체 사용자 |
| **wh-adhoc** | Ad-hoc 분석 | Small | 10분 | 분석가 |
| **wh-etl** | ETL/데이터 변환 | Large | 즉시 종료 | Service Principal |
| **wh-bi-tool** | Tableau/Power BI | Medium | 10분 | BI 도구 |
| **wh-genie** | Genie Space | Small | 10분 | 비기술 사용자 |

{% hint style="warning" %}
**Serverless에서는 멀티 Warehouse 불필요**: Serverless SQL Warehouse의 IWM이 자동으로 워크로드를 격리하므로, 용도별 분리가 불필요합니다. Classic SQL Warehouse에서만 멀티 Warehouse 전략이 효과적입니다.
{% endhint %}

### 8.2 비용 최적화를 위한 Warehouse 통합

```text
[통합 전]
wh-team-a (Small, 10명)  → 월 $500
wh-team-b (Small, 10명)  → 월 $500
wh-team-c (Small, 5명)   → 월 $500
합계: 월 $1,500

[통합 후]
wh-analytics (Medium, 25명) → 월 $800
합계: 월 $800 (47% 절감)

이유: 각 팀의 피크 타임이 다르므로, 통합하면 리소스 활용률이 높아짐
```

---

## 9. Serverless SQL Warehouse 비용 절감 팁

| 전략 | 설명 | 절감 효과 |
|------|------|----------|
| **Auto Stop 5분** | Serverless는 즉시 시작하므로 공격적 설정 | 유휴 비용 80%+ 절감 |
| **Materialized View** | 반복 집계를 사전 계산 | 쿼리당 비용 90% 절감 |
| **Liquid Clustering** | 스캔량 감소 | 쿼리당 비용 50-90% 절감 |
| **Result Cache 활용** | 동일 쿼리 재사용 | 반복 쿼리 비용 0 |
| **쿼리 최적화** | SELECT *, 불필요한 JOIN 제거 | 쿼리당 20-50% 절감 |
| **Warehouse 통합** | 여러 개 → 하나로 | 관리 비용 + 유휴 비용 절감 |

```sql
-- Serverless 비용 모니터링 (시스템 테이블)
SELECT
  date_trunc('day', usage_date) AS day,
  warehouse_id,
  SUM(usage_quantity) AS total_dbus,
  SUM(usage_quantity * list_price) AS estimated_cost
FROM system.billing.usage
WHERE usage_type = 'SQL_WAREHOUSE'
AND usage_date >= current_date() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY estimated_cost DESC;
```

---

## 참고 링크

- [Databricks: SQL Warehouse Sizing](https://docs.databricks.com/aws/en/compute/sql-warehouse/warehouse-behavior)
- [Databricks: Serverless SQL Warehouse](https://docs.databricks.com/aws/en/compute/sql-warehouse/serverless)
- [Databricks: Materialized Views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-materialized-views)
- [Databricks: Result Caching](https://docs.databricks.com/aws/en/optimizations/result-caching)
- [Databricks: Predictive Optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
- [Databricks: System Tables - Billing](https://docs.databricks.com/aws/en/admin/system-tables/billing)

---
