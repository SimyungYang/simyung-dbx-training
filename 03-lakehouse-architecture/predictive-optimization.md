# Predictive Optimization (예측 최적화)

## 왜 Predictive Optimization이 필요한가요?

Delta Lake 테이블을 건강하게 유지하려면 OPTIMIZE(소형 파일 병합), VACUUM(불필요한 파일 삭제), ANALYZE TABLE(통계 수집)을 정기적으로 실행해야 합니다. 하지만 수십, 수백 개의 테이블을 관리할 때 이 작업들을 수동으로 스케줄링하고 모니터링하는 것은 상당한 운영 부담입니다.

> 💡 **Predictive Optimization(예측 최적화)** 은 Databricks가 테이블의 상태를 ** 자동으로 모니터링**하고, 최적의 시점에 **OPTIMIZE, VACUUM, ANALYZE를 자동 실행** 하는 기능입니다. 사용자가 별도의 유지 보수 작업을 스케줄링할 필요가 없어집니다.

비유하자면, 자동차의 **자동 정비 시스템** 과 같습니다. 운전자가 일일이 엔진 오일 교환 주기를 기억하지 않아도, 시스템이 상태를 감지하여 최적의 시점에 정비를 수행합니다.

---

## 자동 수행 작업

Predictive Optimization이 활성화되면 다음 세 가지 작업이 자동으로 수행됩니다.

| 작업 | 설명 | 효과 |
|------|------|------|
| **자동 OPTIMIZE** | 소형 파일이 감지되면 자동으로 병합합니다 | 쿼리 성능 유지 |
| ** 자동 VACUUM** | 보존 기간이 지난 불필요한 파일을 자동 삭제합니다 | 스토리지 비용 절약 |
| ** 자동 ANALYZE** | 테이블 통계를 자동으로 갱신합니다 | 쿼리 최적화기 성능 향상 |

### 동작 방식

1. Databricks가 테이블의 ** 메타데이터와 쓰기 패턴**을 지속적으로 모니터링합니다
2. 소형 파일 비율, 마지막 OPTIMIZE 시점, 데이터 변경량 등을 분석합니다
3. 최적화가 필요하다고 판단되면 ** 자동으로 유지 보수 작업을 실행**합니다
4. 워크로드가 적은 시간대에 실행하여 성능 영향을 최소화합니다

---

## 활성화 방법

Predictive Optimization은 ** 카탈로그, 스키마, 테이블** 세 가지 수준에서 활성화할 수 있습니다.

### 카탈로그 수준 (가장 넓은 범위)

```sql
-- 카탈로그의 모든 Managed Table에 일괄 적용
ALTER CATALOG my_catalog
ENABLE PREDICTIVE OPTIMIZATION;
```

### 스키마 수준

```sql
-- 특정 스키마의 모든 Managed Table에 적용
ALTER SCHEMA my_catalog.my_schema
ENABLE PREDICTIVE OPTIMIZATION;
```

### 테이블 수준

```sql
-- 특정 테이블에만 적용
ALTER TABLE my_catalog.my_schema.orders
ENABLE PREDICTIVE OPTIMIZATION;
```

### 비활성화

```sql
-- 특정 수준에서 비활성화
ALTER TABLE my_catalog.my_schema.orders
DISABLE PREDICTIVE OPTIMIZATION;

-- 상위에서 활성화, 특정 테이블만 제외하는 것도 가능
ALTER CATALOG my_catalog ENABLE PREDICTIVE OPTIMIZATION;
ALTER TABLE my_catalog.my_schema.sensitive_table DISABLE PREDICTIVE OPTIMIZATION;
```

### 상속 규칙

| 오브젝트 | 설정 | 결과 |
|---------|------|------|
| 카탈로그 | ENABLE | - |
| └ 스키마 A | 상속: ENABLE | - |
|   └ 테이블 1 | 상속: ENABLE | 자동 최적화 적용 |
|   └ 테이블 2 | DISABLE | 제외 |
| └ 스키마 B | DISABLE | - |
|   └ 테이블 3 | 상속: DISABLE | 미적용 |
|   └ 테이블 4 | ENABLE | 명시적 활성화 |

상위 수준에서 활성화하면 하위 객체에 ** 상속**됩니다. 하위 수준에서 명시적으로 ENABLE/DISABLE을 설정하면 상속을 ** 오버라이드**합니다.

---

## 요구사항

Predictive Optimization을 사용하려면 다음 조건을 충족해야 합니다.

| 요구사항 | 설명 |
|----------|------|
| **Unity Catalog** | 테이블이 Unity Catalog에 등록되어 있어야 합니다 |
| **Managed Table** | Managed Table에서만 지원됩니다. External Table은 미지원 |
| **Delta 포맷** | Delta Lake 포맷 테이블이어야 합니다 |
| **Serverless 컴퓨트** | 자동 최적화 작업은 Serverless 컴퓨트에서 실행됩니다 |

> ⚠️ **External Table에서는 Predictive Optimization이 지원되지 않습니다.** External Table의 유지 보수는 수동으로 OPTIMIZE, VACUUM을 실행해야 합니다. 이것이 가능하다면 Managed Table을 사용하는 것을 권장하는 이유 중 하나입니다.

---

## 모니터링

### 현재 설정 상태 확인

```sql
-- 테이블의 Predictive Optimization 상태 확인
DESCRIBE TABLE EXTENDED my_catalog.my_schema.orders;
-- 결과에서 'delta.enablePredictiveOptimization' 속성 확인
```

### 시스템 테이블을 통한 모니터링

Predictive Optimization의 실행 이력은 ** 시스템 테이블(System Tables)** 에서 확인할 수 있습니다.

```sql
-- Predictive Optimization 실행 이력 조회
SELECT
    table_name,
    operation_type,
    operation_status,
    operation_metrics,
    start_time,
    end_time
FROM system.storage.predictive_optimization_operations_history
WHERE catalog_name = 'my_catalog'
  AND start_time >= current_date() - INTERVAL 7 DAYS
ORDER BY start_time DESC;
```

### 주요 모니터링 항목

| 확인 항목 | 쿼리/방법 |
|-----------|----------|
| ** 최근 OPTIMIZE 실행** | 시스템 테이블에서 `operation_type = 'OPTIMIZE'` 필터 |
| ** 최근 VACUUM 실행** | 시스템 테이블에서 `operation_type = 'VACUUM'` 필터 |
| ** 절약된 스토리지** | VACUUM 작업의 `operation_metrics`에서 삭제된 파일 크기 확인 |
| ** 병합된 파일 수** | OPTIMIZE 작업의 `operation_metrics`에서 파일 수 변화 확인 |

```sql
-- 최근 7일간 Predictive Optimization으로 절약된 효과 요약
SELECT
    operation_type,
    COUNT(*) AS execution_count,
    SUM(operation_metrics.bytes_removed) AS total_bytes_saved
FROM system.storage.predictive_optimization_operations_history
WHERE catalog_name = 'my_catalog'
  AND start_time >= current_date() - INTERVAL 7 DAYS
GROUP BY operation_type;
```

---

## 비용 관련 참고사항

| 항목 | 설명 |
|------|------|
| **컴퓨트 비용** | Predictive Optimization 작업은 Serverless 컴퓨트에서 실행되므로 별도의 클러스터가 필요 없습니다 |
| ** 과금 방식** | 자동 최적화 작업에 사용된 Serverless 컴퓨트 시간에 대해 과금됩니다 |
| ** 비용 절감 효과** | 수동 OPTIMIZE/VACUUM 대비 최적의 시점에만 실행하므로 불필요한 작업이 줄어듭니다 |
| ** 스토리지 절감** | 자동 VACUUM으로 불필요한 파일이 꾸준히 정리됩니다 |

> 💡 Predictive Optimization의 비용은 일반적으로 수동 스케줄링 대비 ** 더 효율적**입니다. 불필요한 시점에 OPTIMIZE를 실행하는 낭비가 줄어들고, 필요한 시점에만 정확히 실행되기 때문입니다.

---

## 모범 사례

| 항목 | 권장사항 |
|------|---------|
| ** 새 카탈로그** | 카탈로그 수준에서 ENABLE하여 모든 테이블에 일괄 적용 |
| ** 민감한 테이블** | 특정 보존 정책이 필요한 테이블은 개별적으로 DISABLE 후 수동 관리 |
| **External Table** | 지원되지 않으므로, 가능하면 Managed Table로 전환을 고려 |
| ** 모니터링** | 시스템 테이블을 활용하여 실행 이력과 비용을 주기적으로 검토 |

---

## 내부 동작 원리 심층 분석

Predictive Optimization은 단순한 스케줄 실행이 아닌, ** 머신러닝 기반 워크로드 분석**을 통해 최적의 시점과 방법을 결정합니다.

### 3단계 동작 프로세스

| Phase | 작업 | 상세 내용 |
|-------|------|---------|
| **Phase 1: 워크로드 분석** (지속적) | 패턴 분석 | 테이블별 쓰기 패턴(빈도, 크기, 시간대), 쿼리 패턴(자주 필터링되는 컬럼), 소형 파일 비율, 데이터 스큐 감지, 마지막 OPTIMIZE/VACUUM 이후 변경량 |
| **Phase 2: 최적화 계획 수립** | 의사결정 | 비용-편익 분석, 우선순위 결정(효과 큰 테이블 우선), 실행 시간 선택(워크로드 낮은 시간대), 리소스 할당(서버리스 컴퓨트) |
| **Phase 3: 자동 실행** | 최적화 수행 | OPTIMIZE(소형 파일 병합, DV 물리 반영), VACUUM(보존 기간 지난 파일 삭제), ANALYZE(컬럼 통계 갱신), 결과 기록(시스템 테이블에 실행 이력) |

### 최적화 트리거 조건 (추정)

Databricks가 정확한 알고리즘을 공개하지 않지만, 관찰된 동작 패턴에 기반한 추정 트리거 조건입니다.

| 작업 | 추정 트리거 조건 | 설명 |
|------|----------------|------|
| **OPTIMIZE** | 소형 파일(< 128MB) 비율 > 20~30% | 쿼리 성능에 의미 있는 영향을 미치는 시점 |
| **OPTIMIZE** | Deletion Vector가 파일 행의 > 10% 차지 | 읽기 오버헤드가 증가하는 시점 |
| **VACUUM** | 마지막 VACUUM 이후 7일 이상 경과 | 기본 보존 기간(7일) 이후 정리 |
| **ANALYZE** | 테이블 데이터 > 10% 변경 이후 | 통계 정확도가 의미 있게 저하되는 시점 |

> ⚠️ ** 주의**: 위 조건은 공식 문서에 명시되지 않은 관찰 기반 추정입니다. 실제 동작은 다를 수 있으며, Databricks가 알고리즘을 지속적으로 개선합니다.

---

## 비용 분석 — 자동 최적화 DBU vs 수동 OPTIMIZE DBU

### Predictive Optimization의 과금 구조

| 비용 항목 | 설명 |
|-----------|------|
| ** 컴퓨트 DBU** | 서버리스 컴퓨트에서 실행되며, **JOBS_SERVERLESS** SKU로 과금됩니다 |
| ** 스토리지** | OPTIMIZE 과정에서 새 파일 생성 → 일시적으로 스토리지 증가 (VACUUM 후 감소) |
| ** 네트워크** | 같은 리전 내 트래픽이므로 무시 가능 |

### 수동 vs 자동 비용 비교

```
시나리오: 100개 테이블, 각각 일 평균 10GB 신규 데이터

수동 OPTIMIZE (매일 전체 테이블):
  100테이블 × 1분/테이블 × 10 DBU/시간 = ~17 DBU/일 = 510 DBU/월
  (실제 최적화가 필요한 테이블: ~30개/일)
  불필요 비용: 약 70% (357 DBU/월 낭비)

Predictive Optimization (필요한 테이블만):
  30테이블/일 × 1분/테이블 × 10 DBU/시간 = ~5 DBU/일 = 150 DBU/월
  서버리스 DBU 단가 프리미엄 (1.5x): 150 × 1.5 = 225 DBU 환산

비교: 수동 510 DBU vs 자동 225 DBU 환산 → 약 56% 절감
```

> 💡 ** 핵심 인사이트**: Predictive Optimization의 가장 큰 비용 절감 효과는 "불필요한 OPTIMIZE 실행을 줄이는 것"입니다. 수동 스케줄은 데이터 변경이 없는 테이블에도 OPTIMIZE를 실행하지만, Predictive Optimization은 실제로 필요한 테이블에만 실행합니다.

### 비용 모니터링 쿼리

```sql
-- Predictive Optimization의 일별 DBU 소비량
SELECT
    DATE(start_time) AS day,
    operation_type,
    COUNT(*) AS operations,
    SUM(operation_metrics.num_files_optimized) AS total_files,
    ROUND(SUM(TIMESTAMPDIFF(SECOND, start_time, end_time)) / 3600.0 * 10, 2)
        AS estimated_dbu  -- 대략적인 DBU 추정 (10 DBU/시간 가정)
FROM system.storage.predictive_optimization_operations_history
WHERE start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY DATE(start_time), operation_type
ORDER BY day DESC, operation_type;

-- billing 시스템 테이블에서 정확한 비용 확인
SELECT
    DATE(usage_date) AS day,
    SUM(usage_quantity) AS total_dbu,
    SUM(usage_quantity * list_price) AS estimated_cost_usd
FROM system.billing.usage
WHERE sku_name LIKE '%PREDICTIVE_OPTIMIZATION%'
    AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY DATE(usage_date)
ORDER BY day DESC;
```

---

## 비활성화가 필요한 경우

대부분의 테이블에서 Predictive Optimization을 활성화하는 것이 좋지만, 특정 상황에서는 비활성화하는 것이 적절합니다.

| 상황 | 이유 | 대안 |
|------|------|------|
| **규정 준수 테이블** | VACUUM이 특정 보존 기간을 준수해야 하는 경우 (예: 7년 보존) | 수동 VACUUM (보존 기간 명시) |
| ** 쓰기 집중 테이블** | 대량 쓰기 직후 자동 OPTIMIZE가 실행되면 쓰기 성능에 영향 | 수동 OPTIMIZE (쓰기 완료 후) |
| ** 일시적 테이블** | 임시 분석용 테이블에 최적화는 낭비 | 사용 후 즉시 삭제 |
| ** 매우 작은 테이블** | 수 MB 이하 테이블은 최적화 효과가 미미 | 불필요 |
| ** 비용 예측 필요** | 자동 실행으로 비용 예측이 어려운 경우 | 수동 스케줄 (고정 비용) |

```sql
-- 규정 준수 테이블: 자동 최적화 제외, 수동 관리
ALTER TABLE catalog.schema.audit_logs DISABLE PREDICTIVE OPTIMIZATION;

-- 수동 VACUUM (7년 보존)
VACUUM catalog.schema.audit_logs RETAIN 2555 HOURS;  -- 약 7년
```

---

## 대규모 환경에서의 동작 특성

### 스케일 관련 관찰 사항

| 환경 규모 | 테이블 수 | 관찰 특성 |
|-----------|----------|----------|
| ** 소규모** | < 100 | 대부분 테이블이 빠르게 최적화됨. 특별한 이슈 없음 |
| ** 중규모** | 100~1,000 | 우선순위에 따라 순차 처리. 쿼리 빈도 높은 테이블 우선 |
| ** 대규모** | > 1,000 | 모든 테이블이 즉시 최적화되지 않을 수 있음. 가장 효과가 큰 테이블부터 처리 |

### 대규모 환경 모범 사례

1. ** 카탈로그 레벨에서 활성화**: 개별 테이블 관리 대신 카탈로그 레벨에서 ENABLE하고, 예외 테이블만 DISABLE합니다.
2. ** 시스템 테이블 대시보드 구축**: 실행 이력과 효과를 주기적으로 모니터링하는 대시보드를 만듭니다.
3. ** 예외 목록 관리**: 비활성화한 테이블 목록을 문서화하고, 비활성화 사유를 기록합니다.
4. ** 비용 알림 설정**: Predictive Optimization DBU 소비가 임계치를 초과하면 알림을 받도록 설정합니다.

```sql
-- Predictive Optimization 상태 전체 현황 대시보드 쿼리
SELECT
    catalog_name,
    schema_name,
    COUNT(*) AS total_operations,
    COUNT(CASE WHEN operation_type = 'OPTIMIZE' THEN 1 END) AS optimize_count,
    COUNT(CASE WHEN operation_type = 'VACUUM' THEN 1 END) AS vacuum_count,
    COUNT(CASE WHEN operation_type = 'ANALYZE' THEN 1 END) AS analyze_count,
    SUM(operation_metrics.bytes_removed) AS total_bytes_cleaned,
    MIN(start_time) AS earliest_operation,
    MAX(end_time) AS latest_operation
FROM system.storage.predictive_optimization_operations_history
WHERE start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY catalog_name, schema_name
ORDER BY total_operations DESC;
```

---

## Liquid Clustering과의 시너지

Predictive Optimization은 **Liquid Clustering** 이 적용된 테이블에서 특히 효과적입니다. Liquid Clustering은 데이터를 지정된 컬럼 기준으로 자동 재배치하는 기능인데, 이 재배치 작업 자체가 OPTIMIZE의 일부로 실행됩니다.

```sql
-- Liquid Clustering + Predictive Optimization 조합 (최적)
CREATE TABLE catalog.schema.events
CLUSTER BY (event_date, region)
AS SELECT * FROM raw_events;

ALTER CATALOG catalog ENABLE PREDICTIVE OPTIMIZATION;
-- → OPTIMIZE 실행 시 Liquid Clustering도 함께 수행
-- → 쿼리 필터링에 최적화된 데이터 레이아웃이 자동으로 유지됨
```

| Z-Order (기존) + 수동 OPTIMIZE | Liquid Clustering + Predictive Optimization |
|-------------------------------|---------------------------------------------|
| 수동 OPTIMIZE 시에만 Z-Order 적용 | 자동 OPTIMIZE 시 자동 재배치 |
| 최적화 시점 판단이 어려움 | 시스템이 자동 판단 |
| 운영 부담 높음 | 운영 부담 거의 없음 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Predictive Optimization** | OPTIMIZE, VACUUM, ANALYZE를 Databricks가 자동으로 실행하는 기능입니다 |
| ** 내부 동작** | 워크로드 분석 → 비용-편익 판단 → 최적 시점에 자동 실행의 3단계로 동작합니다 |
| ** 활성화 범위** | 카탈로그, 스키마, 테이블 수준에서 설정 가능하며 상속됩니다 |
| ** 요구사항** | Unity Catalog Managed Table + Delta 포맷이어야 합니다 |
| ** 비용 효율** | 수동 대비 불필요한 OPTIMIZE를 줄여 약 50~70% DBU 절감이 가능합니다 |
| ** 비활성화 케이스** | 규정 준수, 쓰기 집중, 임시 테이블 등에서 비활성화를 고려합니다 |
| ** 모니터링** | 시스템 테이블 + billing 테이블에서 실행 이력과 비용을 확인합니다 |
| **Liquid Clustering** | Predictive Optimization과 조합 시 최적의 데이터 레이아웃이 자동 유지됩니다 |

---

## 참고 링크

- [Databricks: Predictive Optimization for Unity Catalog managed tables](https://docs.databricks.com/aws/en/delta/predictive-optimization.html)
- [Azure Databricks: Predictive Optimization](https://learn.microsoft.com/en-us/azure/databricks/optimizations/predictive-optimization)
- [Databricks Blog: Predictive Optimization](https://www.databricks.com/blog/announcing-predictive-optimization)
- [Databricks: System tables reference](https://docs.databricks.com/aws/en/admin/system-tables/index.html)
