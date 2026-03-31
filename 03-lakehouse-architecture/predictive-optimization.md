# Predictive Optimization (예측 최적화)

## 왜 Predictive Optimization이 필요한가요?

Delta Lake 테이블을 건강하게 유지하려면 OPTIMIZE(소형 파일 병합), VACUUM(불필요한 파일 삭제), ANALYZE TABLE(통계 수집)을 정기적으로 실행해야 합니다. 하지만 수십, 수백 개의 테이블을 관리할 때 이 작업들을 수동으로 스케줄링하고 모니터링하는 것은 상당한 운영 부담입니다.

> 💡 **Predictive Optimization(예측 최적화)** 은 Databricks가 테이블의 상태를 **자동으로 모니터링**하고, 최적의 시점에 **OPTIMIZE, VACUUM, ANALYZE를 자동 실행**하는 기능입니다. 사용자가 별도의 유지 보수 작업을 스케줄링할 필요가 없어집니다.

비유하자면, 자동차의 **자동 정비 시스템**과 같습니다. 운전자가 일일이 엔진 오일 교환 주기를 기억하지 않아도, 시스템이 상태를 감지하여 최적의 시점에 정비를 수행합니다.

---

## 자동 수행 작업

Predictive Optimization이 활성화되면 다음 세 가지 작업이 자동으로 수행됩니다.

| 작업 | 설명 | 효과 |
|------|------|------|
| **자동 OPTIMIZE** | 소형 파일이 감지되면 자동으로 병합합니다 | 쿼리 성능 유지 |
| **자동 VACUUM** | 보존 기간이 지난 불필요한 파일을 자동 삭제합니다 | 스토리지 비용 절약 |
| **자동 ANALYZE** | 테이블 통계를 자동으로 갱신합니다 | 쿼리 최적화기 성능 향상 |

### 동작 방식

1. Databricks가 테이블의 **메타데이터와 쓰기 패턴**을 지속적으로 모니터링합니다
2. 소형 파일 비율, 마지막 OPTIMIZE 시점, 데이터 변경량 등을 분석합니다
3. 최적화가 필요하다고 판단되면 **자동으로 유지 보수 작업을 실행**합니다
4. 워크로드가 적은 시간대에 실행하여 성능 영향을 최소화합니다

---

## 활성화 방법

Predictive Optimization은 **카탈로그, 스키마, 테이블** 세 가지 수준에서 활성화할 수 있습니다.

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

```
카탈로그 (ENABLE)
  └── 스키마 A (상속: ENABLE)
  │     └── 테이블 1 (상속: ENABLE) ← 자동 최적화 적용
  │     └── 테이블 2 (DISABLE)     ← 제외
  └── 스키마 B (DISABLE)
        └── 테이블 3 (상속: DISABLE) ← 미적용
        └── 테이블 4 (ENABLE)       ← 명시적 활성화
```

상위 수준에서 활성화하면 하위 객체에 **상속**됩니다. 하위 수준에서 명시적으로 ENABLE/DISABLE을 설정하면 상속을 **오버라이드**합니다.

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

Predictive Optimization의 실행 이력은 **시스템 테이블(System Tables)** 에서 확인할 수 있습니다.

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
| **최근 OPTIMIZE 실행** | 시스템 테이블에서 `operation_type = 'OPTIMIZE'` 필터 |
| **최근 VACUUM 실행** | 시스템 테이블에서 `operation_type = 'VACUUM'` 필터 |
| **절약된 스토리지** | VACUUM 작업의 `operation_metrics`에서 삭제된 파일 크기 확인 |
| **병합된 파일 수** | OPTIMIZE 작업의 `operation_metrics`에서 파일 수 변화 확인 |

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
| **과금 방식** | 자동 최적화 작업에 사용된 Serverless 컴퓨트 시간에 대해 과금됩니다 |
| **비용 절감 효과** | 수동 OPTIMIZE/VACUUM 대비 최적의 시점에만 실행하므로 불필요한 작업이 줄어듭니다 |
| **스토리지 절감** | 자동 VACUUM으로 불필요한 파일이 꾸준히 정리됩니다 |

> 💡 Predictive Optimization의 비용은 일반적으로 수동 스케줄링 대비 **더 효율적**입니다. 불필요한 시점에 OPTIMIZE를 실행하는 낭비가 줄어들고, 필요한 시점에만 정확히 실행되기 때문입니다.

---

## 모범 사례

| 항목 | 권장사항 |
|------|---------|
| **새 카탈로그** | 카탈로그 수준에서 ENABLE하여 모든 테이블에 일괄 적용 |
| **민감한 테이블** | 특정 보존 정책이 필요한 테이블은 개별적으로 DISABLE 후 수동 관리 |
| **External Table** | 지원되지 않으므로, 가능하면 Managed Table로 전환을 고려 |
| **모니터링** | 시스템 테이블을 활용하여 실행 이력과 비용을 주기적으로 검토 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Predictive Optimization** | OPTIMIZE, VACUUM, ANALYZE를 Databricks가 자동으로 실행하는 기능입니다 |
| **활성화 범위** | 카탈로그, 스키마, 테이블 수준에서 설정 가능하며 상속됩니다 |
| **요구사항** | Unity Catalog Managed Table + Delta 포맷이어야 합니다 |
| **모니터링** | 시스템 테이블에서 실행 이력과 효과를 확인할 수 있습니다 |

---

## 참고 링크

- [Databricks: Predictive Optimization for Unity Catalog managed tables](https://docs.databricks.com/aws/en/delta/predictive-optimization.html)
- [Azure Databricks: Predictive Optimization](https://learn.microsoft.com/en-us/azure/databricks/optimizations/predictive-optimization)
- [Databricks Blog: Predictive Optimization](https://www.databricks.com/blog/announcing-predictive-optimization)
- [Databricks: System tables reference](https://docs.databricks.com/aws/en/admin/system-tables/index.html)
