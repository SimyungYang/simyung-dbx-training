# 비용 최적화 모범 사례

## 이 문서에서 다루는 내용

Databricks 플랫폼의 비용을 체계적으로 관리하고 최적화하는 방법을 다룹니다. 비용 구조를 이해하는 것부터 시작하여, 컴퓨트/스토리지/SQL Warehouse 각 영역의 최적화 전략, 그리고 비용 모니터링 대시보드 구축까지 실전에서 바로 적용 가능한 가이드를 제공합니다.

---

## 1. 비용 구조 이해

### DBU(Databricks Unit) 단가 체계

Databricks의 과금은 **DBU(Databricks Unit)** 라는 정규화된 처리 단위를 기반으로 합니다. 동일한 작업이라도 어떤 SKU를 사용하느냐에 따라 DBU 소모량과 단가가 달라집니다.

> 💡 **DBU란?** Databricks의 과금 단위로, CPU 코어, 메모리, 컴퓨트 유형을 종합하여 정규화한 값입니다. 1 DBU는 약 1시간의 기본 컴퓨트 처리량에 해당합니다.

### SKU별 가격 비교표 (AWS 기준, 2025 참고가)

| SKU 유형 | 대표 용도 | 상대 단가 | 특징 |
|----------|----------|----------|------|
| **Jobs Compute** | 스케줄된 배치 작업 | ★☆☆☆☆ (최저) | 가장 경제적, 프로덕션 파이프라인 권장 |
| **Jobs Compute (Serverless)** | 서버리스 배치 작업 | ★★☆☆☆ | 인프라 관리 불필요, 빠른 시작 |
| **All-Purpose Compute** | 인터랙티브 개발, 노트북 | ★★★★☆ | 개발 시에만 사용, 프로덕션 금지 |
| **All-Purpose (Serverless)** | 서버리스 인터랙티브 | ★★★☆☆ | 유휴 비용 없음, 개발자 생산성 극대화 |
| **SQL Warehouse (Serverless)** | BI 쿼리, 대시보드 | ★★★☆☆ | 자동 스케일링, 캐싱 내장 |
| **SQL Warehouse (Pro)** | BI 쿼리 (클래식) | ★★★☆☆ | 수동 스케일링, 예측 가능한 비용 |
| **Model Serving** | 실시간 추론 | ★★★★☆ | GPU 사용 시 비용 급증 |
| **Foundation Model API** | DBRX, Llama 호출 | 토큰 기반 | 입력/출력 토큰 수에 비례 |

> ⚠️ **핵심 원칙**: 개발은 All-Purpose, 프로덕션은 Jobs Compute를 사용하면 **동일 작업 대비 2~3배 비용 절감**이 가능합니다.

### 비용 발생 3대 영역

```
┌─────────────────────────────────────────────┐
│              Databricks 비용 구조              │
├──────────────┬──────────────┬───────────────┤
│   컴퓨트 (60-80%)  │  스토리지 (10-20%) │  네트워크 (5-10%) │
│  - 클러스터 DBU   │  - S3/ADLS 저장    │  - 리전 간 전송   │
│  - SQL Warehouse  │  - Delta 로그      │  - 외부 연동      │
│  - Model Serving  │  - VACUUM 미실행   │  - PrivateLink    │
└──────────────┴──────────────┴───────────────┘
```

---

## 2. 컴퓨트 최적화

### 2.1 Serverless vs Classic 선택 기준

| 판단 기준 | Serverless 추천 | Classic 추천 |
|----------|----------------|-------------|
| **워크로드 패턴** | 간헐적, 불규칙 실행 | 24시간 상시 실행 |
| **시작 시간 민감도** | 즉시 시작 필요 (< 10초) | 수 분 대기 가능 |
| **비용 예측 필요** | 사용량 기반 과금 OK | 고정 예산 필수 |
| **특수 라이브러리** | 표준 라이브러리만 사용 | 커스텀 Docker, GPU, R |
| **데이터 규모** | 소~중규모 (TB 미만) | 대규모 (수십 TB 이상) |
| **관리 여력** | 인프라 팀 없음 | 전담 플랫폼 팀 있음 |

> 💡 **의사결정 공식**: 일일 평균 가동률이 **40% 미만**이면 Serverless가 경제적입니다. 40% 이상 상시 가동한다면 Classic + Reserved Capacity가 유리합니다.

### 2.2 Auto Stop 설정 전략

Auto Stop은 유휴 클러스터를 자동으로 종료하여 불필요한 비용을 방지합니다.

| 환경 | 권장 Auto Stop | 이유 |
|------|--------------|------|
| **개발 (Interactive)** | 10분 | 개발자가 잊고 퇴근해도 자동 종료 |
| **Ad-hoc 분석** | 15분 | 쿼리 간 대기 시간 고려 |
| **공유 클러스터** | 30분 | 여러 사용자의 간헐적 사용 |
| **프로덕션 (Jobs)** | 설정 불필요 | Jobs는 작업 완료 시 자동 종료 |
| **SQL Warehouse (개발)** | 10분 | 빠른 종료로 유휴 비용 제거 |
| **SQL Warehouse (프로덕션)** | 5~10분 | 자동 재시작 기능이 있으므로 공격적 설정 가능 |

```python
# Compute Policy로 Auto Stop 강제 적용 (관리자용)
{
  "autotermination_minutes": {
    "type": "range",
    "maxValue": 30,
    "defaultValue": 10
  },
  "custom_tags.CostCenter": {
    "type": "fixed",
    "value": "engineering"
  }
}
```

### 2.3 Spot 인스턴스 활용

Spot 인스턴스(AWS) / Spot VM(Azure)을 활용하면 **60~90% 비용 절감**이 가능합니다.

| 구성 요소 | Spot 사용 가능 | 권장 비율 | 주의사항 |
|----------|--------------|----------|---------|
| **Driver** | ❌ On-Demand 필수 | 0% Spot | 드라이버 중단 시 전체 작업 실패 |
| **Worker (배치)** | ✅ 적극 활용 | 80-100% Spot | 재시도 로직 필수 |
| **Worker (스트리밍)** | ⚠️ 주의 필요 | 50% 이하 Spot | 인터럽트 시 재처리 지연 |
| **Worker (ML 학습)** | ⚠️ 체크포인트 필수 | 50-80% Spot | 체크포인트 주기 설정 |

```python
# Spot 인스턴스 + Fallback 설정 예시
{
  "aws_attributes": {
    "first_on_demand": 1,          # 드라이버는 On-Demand
    "availability": "SPOT_WITH_FALLBACK",  # Spot 우선, 불가 시 On-Demand
    "spot_bid_price_percent": 100   # On-Demand 가격까지 입찰
  }
}
```

### 2.4 클러스터 사이징 가이드

워크로드 유형에 따른 인스턴스 선택 기준입니다.

| 워크로드 유형 | 병목 지점 | 권장 인스턴스 | 시작 구성 |
|-------------|----------|-------------|----------|
| **기본 ETL** | I/O | Storage Optimized (i3, i4i) | 4 workers, Medium |
| **복잡한 ETL (대량 조인)** | Memory + Shuffle | Memory Optimized (r5, r6i) | 8 workers, Large |
| **ML 학습** | CPU/GPU | Compute Optimized (c5) 또는 GPU (p3, g5) | 1 node (단일), Large |
| **Ad-hoc 분석** | 다양 | General Purpose (m5, m6i) | 2-4 workers, Medium |
| **스트리밍** | CPU + I/O | Compute Optimized (c5, c6i) | 4 workers, Medium |

> 💡 **사이징 공식**: `필요 코어 수 = (데이터 크기 GB / 목표 처리 시간 분) × 2`. 예를 들어 100GB 데이터를 10분 안에 처리하려면 약 20코어가 필요합니다. 이를 기준으로 워커 수를 결정하세요.

### 2.5 Instance Pool 활용

Instance Pool은 유휴 인스턴스를 미리 확보하여 클러스터 시작 시간을 단축합니다.

```python
# Instance Pool 설정 예시
{
  "instance_pool_name": "data-engineering-pool",
  "node_type_id": "i3.xlarge",
  "min_idle_instances": 2,        # 항상 2대 대기 (비용 발생)
  "max_capacity": 20,              # 최대 20대까지 확장
  "idle_instance_autotermination_minutes": 15,  # 유휴 인스턴스 15분 후 반환
  "preloaded_spark_versions": ["15.4.x-scala2.12"]  # 런타임 사전 로드
}
```

| 상황 | Pool 사용 권장 | 이유 |
|------|--------------|------|
| 하루 10회 이상 클러스터 시작 | ✅ | 시작 시간 5분 → 30초 |
| 하루 1-2회 실행 | ❌ | 유휴 비용 > 절감 효과 |
| 다수 팀 공유 환경 | ✅ | 시작 시간 SLA 보장 |

---

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

Predictive Optimization은 Unity Catalog 관리형 테이블에 대해 OPTIMIZE, VACUUM, ANALYZE를 **자동으로 실행**합니다.

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

> 💡 **Serverless 전략**: Serverless SQL Warehouse를 사용한다면 **큰 단일 Warehouse에서 시작**하는 것이 권장됩니다. Serverless의 Intelligent Workload Management가 자동으로 동시성을 관리합니다.

### 4.2 Auto Stop vs Always On 비용 비교

```
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

## 5. 비용 모니터링

### 5.1 system.billing.usage 활용 쿼리

`system.billing.usage` 시스템 테이블은 모든 Databricks 사용량을 중앙에서 추적할 수 있는 핵심 데이터소스입니다.

**쿼리 1: 일별 총 DBU 소비량**

```sql
SELECT
  usage_date,
  SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE usage_date >= current_date() - INTERVAL 30 DAYS
GROUP BY usage_date
ORDER BY usage_date;
```

**쿼리 2: SKU별 비용 분포**

```sql
SELECT
  sku_name,
  SUM(usage_quantity) AS total_dbus,
  ROUND(SUM(usage_quantity) / (SELECT SUM(usage_quantity) FROM system.billing.usage
    WHERE usage_date >= current_date() - INTERVAL 30 DAYS) * 100, 1) AS pct
FROM system.billing.usage
WHERE usage_date >= current_date() - INTERVAL 30 DAYS
GROUP BY sku_name
ORDER BY total_dbus DESC;
```

**쿼리 3: 팀별 비용 (태그 기반)**

```sql
SELECT
  usage_metadata.custom_tags['Team'] AS team,
  sku_name,
  SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE usage_date >= current_date() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY total_dbus DESC;
```

**쿼리 4: Top 10 고비용 클러스터**

```sql
SELECT
  usage_metadata.cluster_id,
  SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE
  usage_date >= current_date() - INTERVAL 30 DAYS
  AND usage_metadata.cluster_id IS NOT NULL
GROUP BY 1
ORDER BY total_dbus DESC
LIMIT 10;
```

**쿼리 5: 주말/야간 불필요 사용 탐지**

```sql
SELECT
  usage_date,
  HOUR(usage_start_time) AS hour_of_day,
  DAYOFWEEK(usage_date) AS day_of_week,
  SUM(usage_quantity) AS dbus
FROM system.billing.usage
WHERE
  usage_date >= current_date() - INTERVAL 30 DAYS
  AND (DAYOFWEEK(usage_date) IN (1, 7)  -- 주말
       OR HOUR(usage_start_time) NOT BETWEEN 8 AND 20)  -- 야간
GROUP BY 1, 2, 3
HAVING SUM(usage_quantity) > 10
ORDER BY dbus DESC;
```

**쿼리 6: 서버리스 vs 클래식 비용 비교**

```sql
SELECT
  CASE
    WHEN sku_name LIKE '%SERVERLESS%' THEN 'Serverless'
    ELSE 'Classic'
  END AS compute_type,
  SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE
  usage_date >= current_date() - INTERVAL 30 DAYS
  AND billing_origin_product IN ('JOBS', 'SQL', 'ALL_PURPOSE')
GROUP BY 1;
```

**쿼리 7: 일별 비용 추이 (전주 대비)**

```sql
WITH daily AS (
  SELECT
    usage_date,
    SUM(usage_quantity) AS dbus
  FROM system.billing.usage
  WHERE usage_date >= current_date() - INTERVAL 60 DAYS
  GROUP BY usage_date
)
SELECT
  d1.usage_date,
  d1.dbus AS today_dbus,
  d2.dbus AS last_week_dbus,
  ROUND((d1.dbus - d2.dbus) / d2.dbus * 100, 1) AS change_pct
FROM daily d1
LEFT JOIN daily d2 ON d1.usage_date = d2.usage_date + INTERVAL 7 DAYS
WHERE d1.usage_date >= current_date() - INTERVAL 30 DAYS
ORDER BY d1.usage_date DESC;
```

**쿼리 8: 프로덕트별 월간 비용 트렌드**

```sql
SELECT
  date_trunc('month', usage_date) AS month,
  billing_origin_product,
  SUM(usage_quantity) AS monthly_dbus
FROM system.billing.usage
WHERE usage_date >= current_date() - INTERVAL 180 DAYS
GROUP BY 1, 2
ORDER BY month, monthly_dbus DESC;
```

**쿼리 9: 사용자별 비용 (Chargeback)**

```sql
SELECT
  identity_metadata.run_as AS user_email,
  billing_origin_product,
  SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE
  usage_date >= current_date() - INTERVAL 30 DAYS
  AND identity_metadata.run_as IS NOT NULL
GROUP BY 1, 2
ORDER BY total_dbus DESC
LIMIT 20;
```

**쿼리 10: 비용 이상 감지 (일평균 대비 200% 초과)**

```sql
WITH stats AS (
  SELECT
    AVG(daily_dbus) AS avg_dbus,
    STDDEV(daily_dbus) AS stddev_dbus
  FROM (
    SELECT usage_date, SUM(usage_quantity) AS daily_dbus
    FROM system.billing.usage
    WHERE usage_date BETWEEN current_date() - INTERVAL 60 DAYS
                         AND current_date() - INTERVAL 7 DAYS
    GROUP BY usage_date
  )
)
SELECT
  b.usage_date,
  SUM(b.usage_quantity) AS daily_dbus,
  s.avg_dbus,
  ROUND((SUM(b.usage_quantity) - s.avg_dbus) / s.avg_dbus * 100, 1) AS deviation_pct
FROM system.billing.usage b
CROSS JOIN stats s
WHERE b.usage_date >= current_date() - INTERVAL 7 DAYS
GROUP BY b.usage_date, s.avg_dbus, s.stddev_dbus
HAVING SUM(b.usage_quantity) > s.avg_dbus + 2 * s.stddev_dbus
ORDER BY b.usage_date;
```

### 5.2 팀별/프로젝트별 비용 할당 (태그 기반)

비용을 조직 단위로 추적하려면 **Custom Tag** 체계를 설계해야 합니다.

```python
# 태그 표준 체계 예시
required_tags = {
  "Team": "data-engineering",       # 팀명
  "Project": "customer-360",        # 프로젝트명
  "Environment": "production",      # 환경 (dev/staging/prod)
  "CostCenter": "CC-1234",          # 비용 센터 코드
  "Owner": "user@company.com"       # 책임자
}
```

```json
// Compute Policy로 태그 강제 적용
{
  "custom_tags.Team": {
    "type": "fixed",
    "value": "data-engineering"
  },
  "custom_tags.CostCenter": {
    "type": "regex",
    "pattern": "CC-\\d{4}"
  },
  "custom_tags.Environment": {
    "type": "allowlist",
    "values": ["dev", "staging", "prod"]
  }
}
```

### 5.3 비용 알림 설정

```sql
-- 일일 비용이 임계값을 초과하면 알림 트리거
-- Databricks SQL Alert로 설정
SELECT
  usage_date,
  SUM(usage_quantity) AS daily_dbus,
  1000 AS threshold_dbus  -- 임계값: 1,000 DBU/일
FROM system.billing.usage
WHERE usage_date = current_date() - INTERVAL 1 DAY
GROUP BY usage_date
HAVING SUM(usage_quantity) > 1000;
```

> 💡 **알림 설정**: 위 쿼리를 Databricks SQL Alert로 등록하고, Slack 또는 이메일 알림을 연동하면 비용 이상을 즉시 감지할 수 있습니다.

---

## 6. 실전 절감 사례

### 사례 1: 개발 환경 최적화

| 항목 | Before | After | 절감률 |
|------|--------|-------|-------|
| 클러스터 유형 | All-Purpose, 항시 가동 | Serverless + Auto Stop 10분 | - |
| 일일 가동 시간 | 24시간 | ~4시간 | 83% |
| 월간 DBU | 7,200 DBU | 1,200 DBU | **83%** |

### 사례 2: ETL 파이프라인 최적화

| 항목 | Before | After | 절감률 |
|------|--------|-------|-------|
| 컴퓨트 | All-Purpose Compute | Jobs Compute + Spot 80% | - |
| 인스턴스 | r5.2xlarge × 10 | i3.xlarge × 8 (스토리지 최적화) | - |
| 실행 시간 | 2시간 | 1.5시간 (Photon 적용) | 25% |
| 월간 DBU | 3,000 DBU | 600 DBU | **80%** |

### 사례 3: BI 대시보드 환경 최적화

| 항목 | Before | After | 절감률 |
|------|--------|-------|-------|
| Warehouse | Pro, Large, Always On | Serverless, Medium, Auto Stop 5분 | - |
| 반복 쿼리 | 매번 재계산 | Materialized View 활용 | - |
| 월간 가동 | 720시간 | ~150시간 | 79% |
| 월간 DBU | 5,400 DBU | 900 DBU | **83%** |

---

## 7. 비용 최적화 체크리스트

프로젝트 시작 시 아래 항목을 점검하세요.

- [ ] 모든 클러스터에 Auto Stop이 설정되어 있는가?
- [ ] 프로덕션 워크로드가 Jobs Compute를 사용하는가?
- [ ] Spot 인스턴스를 최대한 활용하고 있는가?
- [ ] Custom Tag 체계가 수립되어 있는가?
- [ ] Predictive Optimization이 활성화되어 있는가?
- [ ] 불필요한 All-Purpose 클러스터가 방치되어 있지 않은가?
- [ ] SQL Warehouse에 적절한 Auto Stop이 설정되어 있는가?
- [ ] 반복 쿼리에 Materialized View를 적용했는가?
- [ ] 비용 모니터링 대시보드와 알림이 설정되어 있는가?
- [ ] 월간 비용 리뷰 프로세스가 존재하는가?

---

## 참고 링크

- [Databricks 시스템 테이블 - Billing Usage](https://docs.databricks.com/aws/en/admin/system-tables/billing)
- [클러스터 구성 모범 사례](https://docs.databricks.com/aws/en/compute/cluster-config-best-practices)
- [SQL Warehouse 사이징 가이드](https://docs.databricks.com/aws/en/compute/sql-warehouse/warehouse-behavior)
- [Predictive Optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
- [Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering)
- [Databricks 가격표 (AWS)](https://www.databricks.com/product/pricing/product-pricing)
