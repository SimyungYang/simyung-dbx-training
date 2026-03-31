# OPTIMIZE와 VACUUM 상세

## 왜 테이블 유지 보수가 필요한가요?

Delta Lake 테이블은 시간이 지나면서 두 가지 문제가 자연스럽게 발생합니다.

1. **소형 파일 누적**: 잦은 INSERT, 스트리밍 수집으로 인해 작은 파일이 수천~수만 개 쌓입니다
2. **불필요한 파일 잔류**: UPDATE/DELETE 후 이전 버전의 파일이 계속 남아 스토리지 비용이 증가합니다

**OPTIMIZE**는 첫 번째 문제를, **VACUUM**은 두 번째 문제를 해결합니다. 이 두 작업은 Delta Lake 테이블을 건강하게 유지하는 핵심 유지 보수 작업입니다.

---

## 소형 파일 문제 (Small File Problem)

> 💡 **Small File Problem(소형 파일 문제)** 이란 테이블에 매우 작은 크기의 파일이 대량으로 쌓이는 현상입니다. 각 파일을 열고 닫는 I/O 오버헤드가 누적되어 쿼리 성능이 심각하게 저하됩니다.

### 언제 발생하나요?

| 원인 | 설명 |
|------|------|
| **Structured Streaming** | 마이크로 배치마다 소량의 데이터를 쓰면 작은 파일이 생성됩니다 |
| **잦은 INSERT** | 소량 데이터를 빈번하게 INSERT하면 파일이 쪼개집니다 |
| **과도한 파티셔닝** | 파티션 수가 많으면 각 파티션의 파일이 매우 작아집니다 |
| **많은 동시 쓰기** | 여러 작업이 동시에 테이블에 쓰면 파일이 분산됩니다 |

예를 들어, 1분마다 100행씩 쓰는 스트리밍 파이프라인이 있다면, 하루에 **1,440개의 작은 파일**이 생성됩니다. 한 달이면 약 43,200개입니다.

---

## OPTIMIZE (데이터 압축)

### 동작 원리

OPTIMIZE는 여러 개의 작은 파일을 읽어 **적절한 크기의 큰 파일로 병합(bin-packing)**합니다. 기본 목표 파일 크기는 약 **1GB**입니다.

```
OPTIMIZE 전:                          OPTIMIZE 후:
┌──────┐ ┌──────┐ ┌──────┐           ┌─────────────────┐
│ 1MB  │ │ 2MB  │ │ 500KB│           │                 │
└──────┘ └──────┘ └──────┘           │    ~1GB         │
┌──────┐ ┌──────┐ ┌──────┐    →     │                 │
│ 3MB  │ │ 100KB│ │ 4MB  │           │                 │
└──────┘ └──────┘ └──────┘           └─────────────────┘
  (6개의 작은 파일)                    (1개의 최적화된 파일)
```

### 기본 사용법

```sql
-- 테이블 전체 최적화
OPTIMIZE catalog.schema.orders;

-- 결과 확인
-- +-------------------------------------------+
-- | path | metrics                             |
-- +-------------------------------------------+
-- | ...  | numFilesAdded: 5, numFilesRemoved: 150, ... |
-- +-------------------------------------------+
```

### 조건부 최적화

```sql
-- 특정 조건의 데이터만 최적화 (파티션 프루닝)
OPTIMIZE catalog.schema.orders
WHERE order_date >= '2025-03-01';

-- 최근 데이터만 최적화 (자주 사용하는 패턴)
OPTIMIZE catalog.schema.orders
WHERE order_date >= current_date() - INTERVAL 7 DAYS;
```

### Optimized Write (자동 파일 크기 최적화)

OPTIMIZE를 수동으로 실행하는 대신, 데이터를 쓸 때 **자동으로 적절한 크기의 파일을 생성**하는 기능도 있습니다.

```sql
-- 테이블 속성으로 활성화
ALTER TABLE catalog.schema.orders
SET TBLPROPERTIES ('delta.autoOptimize.optimizeWrite' = 'true');
```

> 💡 **Optimized Write**는 쓰기 시점에 파일 크기를 최적화하지만, 이미 쌓인 소형 파일은 처리하지 못합니다. 기존 파일의 압축에는 OPTIMIZE를 별도로 실행해야 합니다.

---

## VACUUM (불필요한 파일 정리)

### 동작 원리

Delta Lake에서 UPDATE나 DELETE를 실행하면, 기존 파일은 즉시 삭제되지 않습니다. 타임 트래블과 진행 중인 쿼리를 위해 **이전 버전의 파일이 보존**됩니다. VACUUM은 지정된 보존 기간보다 오래된 불필요한 파일을 **영구적으로 삭제**합니다.

### 기본 사용법

```sql
-- 기본 실행: 7일(168시간)보다 오래된 파일 삭제
VACUUM catalog.schema.orders;

-- 보존 기간 지정: 30일
VACUUM catalog.schema.orders RETAIN 720 HOURS;

-- 보존 기간 지정: 90일
VACUUM catalog.schema.orders RETAIN 2160 HOURS;
```

### DRY RUN (삭제 미리 확인)

실제로 삭제하기 전에, 어떤 파일이 삭제될지 미리 확인할 수 있습니다.

```sql
-- DRY RUN: 삭제될 파일 목록만 반환 (실제 삭제 안 함)
VACUUM catalog.schema.orders DRY RUN;

-- 결과 예시:
-- +--------------------------------------------+
-- | path                                       |
-- +--------------------------------------------+
-- | s3://bucket/table/part-00001-old.parquet   |
-- | s3://bucket/table/part-00002-old.parquet   |
-- +--------------------------------------------+
-- Found 2 files to delete.
```

### 보존 기간 설정

```sql
-- 테이블 속성으로 기본 보존 기간 설정
ALTER TABLE catalog.schema.orders
SET TBLPROPERTIES ('delta.deletedFileRetentionDuration' = 'interval 30 days');
```

> ⚠️ **주의사항**:
> - VACUUM 후에는 보존 기간 이전의 **타임 트래블이 불가능**해집니다
> - 기본 보존 기간 7일보다 짧게 설정하는 것은 **강력히 권장하지 않습니다** (진행 중인 쿼리가 실패할 수 있음)
> - 7일 미만으로 설정하려면 `spark.databricks.delta.retentionDurationCheck.enabled = false`를 먼저 설정해야 합니다

---

## ANALYZE TABLE (통계 수집)

> 💡 **ANALYZE TABLE**은 테이블의 **컬럼 통계(statistics)** 를 수집하는 명령입니다. 쿼리 최적화기(Query Optimizer)가 이 통계를 활용하여 더 효율적인 실행 계획을 수립합니다.

### 사용법

```sql
-- 전체 테이블 통계 수집
ANALYZE TABLE catalog.schema.orders COMPUTE STATISTICS;

-- 특정 컬럼의 통계만 수집
ANALYZE TABLE catalog.schema.orders
COMPUTE STATISTICS FOR COLUMNS order_date, customer_id, amount;

-- Delta Statistics 수집 (데이터 스킵을 위한 통계)
ANALYZE TABLE catalog.schema.orders
COMPUTE DELTA STATISTICS;
```

### 수집되는 통계 정보

| 통계 | 설명 | 활용 |
|------|------|------|
| **행 수(numRows)** | 테이블의 전체 행 수 | JOIN 순서 최적화 |
| **최소/최대값(min/max)** | 각 컬럼의 최솟값과 최댓값 | Data Skipping |
| **NULL 수(numNulls)** | 각 컬럼의 NULL 개수 | 필터 최적화 |
| **고유값 수(distinctCount)** | 컬럼의 고유값 수 | JOIN 전략 선택 |

---

## 운영 주기 권장사항

| 작업 | 권장 주기 | 설명 |
|------|-----------|------|
| **OPTIMIZE** | 일 1회 또는 데이터 변경 후 | 소형 파일을 합쳐 쿼리 성능 유지 |
| **VACUUM** | 주 1회 | 불필요한 파일 삭제로 스토리지 비용 절약 |
| **ANALYZE TABLE** | 주 1회 또는 대량 변경 후 | 쿼리 최적화기에 최신 통계 제공 |

### 운영 스크립트 예시

```sql
-- 일일 유지 보수 스크립트
-- Step 1: 데이터 압축
OPTIMIZE catalog.schema.orders;

-- Step 2: 불필요한 파일 정리 (30일 보존)
VACUUM catalog.schema.orders RETAIN 720 HOURS;

-- Step 3: 통계 갱신
ANALYZE TABLE catalog.schema.orders COMPUTE STATISTICS;
```

```python
# PySpark로 여러 테이블 일괄 유지 보수
tables = [
    "catalog.schema.orders",
    "catalog.schema.customers",
    "catalog.schema.products"
]

for table in tables:
    spark.sql(f"OPTIMIZE {table}")
    spark.sql(f"VACUUM {table} RETAIN 720 HOURS")
    spark.sql(f"ANALYZE TABLE {table} COMPUTE STATISTICS")
    print(f"✅ {table} 유지 보수 완료")
```

> 💡 **Predictive Optimization**이 활성화된 Managed Table이라면, OPTIMIZE와 VACUUM을 Databricks가 자동으로 실행합니다. 수동 유지 보수가 필요 없어지므로, 가능하다면 Predictive Optimization 사용을 권장합니다. 자세한 내용은 [Predictive Optimization](./predictive-optimization.md) 문서를 참고하세요.

---

## 정리

| 명령어 | 목적 | 효과 |
|--------|------|------|
| **OPTIMIZE** | 소형 파일 병합 | 쿼리 성능 향상 |
| **VACUUM** | 불필요한 파일 삭제 | 스토리지 비용 절약 |
| **ANALYZE TABLE** | 통계 정보 수집 | 쿼리 최적화기 성능 향상 |

---

## 참고 링크

- [Databricks: OPTIMIZE](https://docs.databricks.com/aws/en/delta/optimize.html)
- [Databricks: VACUUM](https://docs.databricks.com/aws/en/sql/language-manual/delta-vacuum.html)
- [Databricks: ANALYZE TABLE](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-analyze-table.html)
- [Azure Databricks: Optimize data file layout](https://learn.microsoft.com/en-us/azure/databricks/delta/optimize)
- [Databricks: Predictive Optimization](https://docs.databricks.com/aws/en/delta/predictive-optimization.html)
