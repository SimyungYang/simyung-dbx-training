# 비용 모니터링과 체크리스트

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

비용을 조직 단위로 추적하려면 **Custom Tag**체계를 설계해야 합니다.

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

> 💡 ** 알림 설정**: 위 쿼리를 Databricks SQL Alert로 등록하고, Slack 또는 이메일 알림을 연동하면 비용 이상을 즉시 감지할 수 있습니다.

---

## 6. 실전 절감 사례

### 사례 1: 개발 환경 최적화

| 항목 | Before | After | 절감률 |
|------|--------|-------|-------|
| 클러스터 유형 | All-Purpose, 항시 가동 | Serverless + Auto Stop 10분 | - |
| 일일 가동 시간 | 24시간 | ~4시간 | 83% |
| 월간 DBU | 7,200 DBU | 1,200 DBU | **83%**|

### 사례 2: ETL 파이프라인 최적화

| 항목 | Before | After | 절감률 |
|------|--------|-------|-------|
| 컴퓨트 | All-Purpose Compute | Jobs Compute + Spot 80% | - |
| 인스턴스 | r5.2xlarge × 10 | i3.xlarge × 8 (스토리지 최적화) | - |
| 실행 시간 | 2시간 | 1.5시간 (Photon 적용) | 25% |
| 월간 DBU | 3,000 DBU | 600 DBU | **80%**|

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
