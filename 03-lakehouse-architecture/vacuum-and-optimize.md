# OPTIMIZE와 VACUUM 상세

## 왜 테이블 유지 보수가 필요한가요?

Delta Lake 테이블은 시간이 지나면서 두 가지 문제가 자연스럽게 발생합니다.

1. **소형 파일 누적**: 잦은 INSERT, 스트리밍 수집으로 인해 작은 파일이 수천~수만 개 쌓입니다
2. **불필요한 파일 잔류**: UPDATE/DELETE 후 이전 버전의 파일이 계속 남아 스토리지 비용이 증가합니다

**OPTIMIZE**는 첫 번째 문제를, **VACUUM** 은 두 번째 문제를 해결합니다. 이 두 작업은 Delta Lake 테이블을 건강하게 유지하는 핵심 유지 보수 작업입니다.

---

## 소형 파일 문제 (Small File Problem)

> 💡 **Small File Problem(소형 파일 문제)** 이란 테이블에 매우 작은 크기의 파일이 대량으로 쌓이는 현상입니다. 각 파일을 열고 닫는 I/O 오버헤드가 누적되어 쿼리 성능이 심각하게 저하됩니다.

### 언제 발생하나요?

| 원인 | 설명 |
|------|------|
| **Structured Streaming**| 마이크로 배치마다 소량의 데이터를 쓰면 작은 파일이 생성됩니다 |
| **잦은 INSERT**| 소량 데이터를 빈번하게 INSERT하면 파일이 쪼개집니다 |
| **과도한 파티셔닝**| 파티션 수가 많으면 각 파티션의 파일이 매우 작아집니다 |
| **많은 동시 쓰기**| 여러 작업이 동시에 테이블에 쓰면 파일이 분산됩니다 |

예를 들어, 1분마다 100행씩 쓰는 스트리밍 파이프라인이 있다면, 하루에 **1,440개의 작은 파일** 이 생성됩니다. 한 달이면 약 43,200개입니다.

---

## OPTIMIZE (데이터 압축)

### 동작 원리

OPTIMIZE는 여러 개의 작은 파일을 읽어 **적절한 크기의 큰 파일로 병합(bin-packing)**합니다. 기본 목표 파일 크기는 약 **1GB** 입니다.

| 상태 | 파일 구성 | 설명 |
|------|---------|------|
| **OPTIMIZE 전**| 1MB, 2MB, 500KB, 3MB, 100KB, 4MB | 6개의 작은 파일 |
| **OPTIMIZE 후**| ~1GB | 1개의 최적화된 파일로 병합 |

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

OPTIMIZE를 수동으로 실행하는 대신, 데이터를 쓸 때 **자동으로 적절한 크기의 파일을 생성** 하는 기능도 있습니다.

```sql
-- 테이블 속성으로 활성화
ALTER TABLE catalog.schema.orders
SET TBLPROPERTIES ('delta.autoOptimize.optimizeWrite' = 'true');
```

> 💡 **Optimized Write** 는 쓰기 시점에 파일 크기를 최적화하지만, 이미 쌓인 소형 파일은 처리하지 못합니다. 기존 파일의 압축에는 OPTIMIZE를 별도로 실행해야 합니다.

---

## VACUUM (불필요한 파일 정리)

### 동작 원리

Delta Lake에서 UPDATE나 DELETE를 실행하면, 기존 파일은 즉시 삭제되지 않습니다. 타임 트래블과 진행 중인 쿼리를 위해 **이전 버전의 파일이 보존**됩니다. VACUUM은 지정된 보존 기간보다 오래된 불필요한 파일을 **영구적으로 삭제** 합니다.

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
> - VACUUM 후에는 보존 기간 이전의 **타임 트래블이 불가능** 해집니다
> - 기본 보존 기간 7일보다 짧게 설정하는 것은 **강력히 권장하지 않습니다**(진행 중인 쿼리가 실패할 수 있음)
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
| **행 수(numRows)**| 테이블의 전체 행 수 | JOIN 순서 최적화 |
| **최소/최대값(min/max)**| 각 컬럼의 최솟값과 최댓값 | Data Skipping |
| **NULL 수(numNulls)**| 각 컬럼의 NULL 개수 | 필터 최적화 |
| **고유값 수(distinctCount)**| 컬럼의 고유값 수 | JOIN 전략 선택 |

---

## 운영 주기 권장사항

| 작업 | 권장 주기 | 설명 |
|------|-----------|------|
| **OPTIMIZE**| 일 1회 또는 데이터 변경 후 | 소형 파일을 합쳐 쿼리 성능 유지 |
| **VACUUM**| 주 1회 | 불필요한 파일 삭제로 스토리지 비용 절약 |
| **ANALYZE TABLE**| 주 1회 또는 대량 변경 후 | 쿼리 최적화기에 최신 통계 제공 |

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

> 💡 **Predictive Optimization** 이 활성화된 Managed Table이라면, OPTIMIZE와 VACUUM을 Databricks가 자동으로 실행합니다. 수동 유지 보수가 필요 없어지므로, 가능하다면 Predictive Optimization 사용을 권장합니다. 자세한 내용은 [Predictive Optimization](./predictive-optimization.md) 문서를 참고하세요.

---

## 현장에서 배운 것들: OPTIMIZE와 VACUUM의 현실

### VACUUM을 안 해서 스토리지 비용이 3배로 뛴 사례

이건 제가 2022년에 실제로 마주한 상황입니다. 한 미디어 고객이 Databricks를 도입한 지 8개월째였는데, 월간 S3 비용이 계속 올라가서 원인 분석을 요청해왔습니다.

원인은 단순했습니다. **VACUUM을 한 번도 실행하지 않았습니다.**

```
상황:
- 20개 주요 테이블, 합계 원본 데이터 약 5TB
- 매일 MERGE(upsert) 작업으로 평균 30%의 행이 업데이트
- MERGE는 기존 파일을 삭제 마킹하고 새 파일을 생성
- 8개월간 이전 파일이 모두 보존

결과:
- S3에 실제 저장된 데이터: 약 18TB (원본의 3.6배)
- 그중 현재 유효한 파일: 5TB
- 삭제 대상(VACUUM으로 정리 가능): 13TB
- 낭비되는 월간 스토리지 비용: 약 $300/월 (연간 $3,600)
```

문제를 더 심각하게 만든 것은, 이전 파일이 많아질수록 **파일 목록 조회(listing)** 가 느려져서 쿼리 성능까지 저하되었다는 점입니다.

```sql
-- VACUUM 한 번으로 13TB 정리
VACUUM prod.media.user_events RETAIN 168 HOURS;  -- 7일 보존
-- 삭제된 파일 수: 약 450,000개
-- 해방된 스토리지: 13TB
-- 실행 시간: 약 45분
```

> ⚠️ **교훈**: VACUUM은 "언젠가 해야지"가 아니라 **Day 1부터 스케줄링** 해야 합니다. Predictive Optimization이 있으면 자동이지만, 그렇지 않다면 주 1회 VACUUM을 반드시 스케줄에 넣으세요.

### 프로덕션에서 VACUUM DRY RUN을 먼저 돌려야 하는 이유

VACUUM은 **되돌릴 수 없는 작업** 입니다. 한번 삭제된 파일은 복구할 수 없습니다. 그래서 프로덕션에서는 반드시 DRY RUN을 먼저 실행해야 합니다.

실제로 한 고객이 DRY RUN 없이 VACUUM을 실행했다가 문제가 생긴 사례가 있습니다.

```sql
-- 실수: 보존 기간을 0시간으로 설정 (절대 하면 안 됩니다!)
SET spark.databricks.delta.retentionDurationCheck.enabled = false;
VACUUM prod.crm.customer_master RETAIN 0 HOURS;
-- → 현재 진행 중인 장시간 쿼리가 참조하던 파일까지 삭제
-- → 해당 쿼리가 FileNotFoundException으로 실패
-- → 타임 트래블 완전 불가능
```

**올바른 VACUUM 실행 절차:**

```sql
-- Step 1: DRY RUN으로 삭제 대상 파일 수와 크기 확인
VACUUM prod.crm.customer_master DRY RUN;
-- "Found 12,345 files (8.2 GB) that are eligible for deletion"

-- Step 2: 예상 결과가 합리적인지 판단
-- (전체 테이블 크기 대비 삭제 비율이 너무 높으면 주의)

-- Step 3: 트래픽이 적은 시간에 실행
VACUUM prod.crm.customer_master RETAIN 168 HOURS;

-- Step 4: 실행 후 테이블 정상 여부 확인
SELECT COUNT(*) FROM prod.crm.customer_master;
DESCRIBE HISTORY prod.crm.customer_master LIMIT 3;
```

### OPTIMIZE 주기: 실전 가이드

"얼마나 자주 OPTIMIZE를 해야 하나요?"는 가장 많이 받는 질문입니다. 정답은 "**테이블의 쓰기 패턴에 따라 다르다**"입니다.

| 쓰기 패턴 | OPTIMIZE 주기 | 이유 |
|-----------|-------------|------|
| **Structured Streaming (1분 마이크로배치)**| 1~4시간마다 | 매 배치마다 소형 파일 생성, 빠르게 누적 |
| **시간별 배치 적재**| 일 1회 (적재 후) | 적재 직후가 소형 파일이 가장 많은 시점 |
| **일 1회 배치 적재**| 적재 직후 1회 | 하루치 데이터가 이미 적절한 크기일 수 있으므로 확인 후 |
| **주 1회 대량 적재**| 적재 직후 1회 | 대량 적재 자체가 큰 파일을 생성할 수 있음 |
| **빈번한 MERGE (upsert)** | 일 1~2회 | MERGE는 기존 파일을 재작성하므로 파편화 발생 |

#### OPTIMIZE가 정말 필요한지 확인하는 방법

무작정 OPTIMIZE를 실행하지 말고, 먼저 테이블 상태를 진단하세요.

```sql
-- 테이블의 파일 크기 분포 확인
DESCRIBE DETAIL prod.schema.my_table;
-- numFiles, sizeInBytes를 확인
-- 평균 파일 크기 = sizeInBytes / numFiles
-- 평균이 100MB 미만이면 OPTIMIZE 필요

-- 구체적인 파일 크기 분포 (PySpark)
```

```python
# 파일 크기 분포를 확인하는 진단 스크립트
from delta.tables import DeltaTable

dt = DeltaTable.forName(spark, "prod.schema.my_table")
files_df = dt.detail().select("numFiles", "sizeInBytes").collect()[0]

num_files = files_df["numFiles"]
size_bytes = files_df["sizeInBytes"]
avg_size_mb = (size_bytes / num_files) / (1024 * 1024) if num_files > 0 else 0

print(f"총 파일 수: {num_files:,}")
print(f"총 크기: {size_bytes / (1024**3):.1f} GB")
print(f"평균 파일 크기: {avg_size_mb:.1f} MB")

if avg_size_mb < 100:
    print("⚠️ OPTIMIZE 권장: 평균 파일 크기가 100MB 미만입니다")
elif avg_size_mb < 500:
    print("ℹ️ 양호: 하지만 OPTIMIZE로 추가 개선 가능")
else:
    print("✅ 최적: OPTIMIZE 불필요")
```

### OPTIMIZE + VACUUM 자동화: 프로덕션 운영 패턴

수동으로 OPTIMIZE와 VACUUM을 실행하는 것은 초기에만 허용됩니다. 프로덕션에서는 반드시 자동화해야 합니다.

#### 패턴 1: Predictive Optimization (권장)

```sql
-- Managed Table에서 자동으로 OPTIMIZE와 VACUUM을 실행
-- Unity Catalog에서 카탈로그 레벨로 활성화
ALTER CATALOG my_catalog
SET DBPROPERTIES ('predictive_optimization' = 'ENABLE');

-- 또는 스키마 레벨
ALTER SCHEMA my_catalog.my_schema
SET DBPROPERTIES ('predictive_optimization' = 'ENABLE');
```

> 💡 **Predictive Optimization이 하는 일**: Databricks가 테이블의 쓰기 패턴, 쿼리 패턴, 파일 크기 분포를 자동으로 분석하여, 최적의 시점에 OPTIMIZE와 VACUUM을 실행합니다. 사람이 판단하는 것보다 정확하고, 잊지 않습니다. **가능하다면 이것을 1순위로 사용하세요.**

#### 패턴 2: Lakeflow Jobs로 스케줄링

```python
# 모든 테이블을 자동으로 유지보수하는 잡
import datetime

# 유지보수 대상 테이블 목록 (메타데이터에서 동적으로 조회도 가능)
tables = spark.sql("""
    SELECT table_catalog, table_schema, table_name
    FROM system.information_schema.tables
    WHERE table_catalog = 'prod'
      AND table_type = 'MANAGED'
""").collect()

results = []
for t in tables:
    full_name = f"{t.table_catalog}.{t.table_schema}.{t.table_name}"
    try:
        # OPTIMIZE
        opt_result = spark.sql(f"OPTIMIZE {full_name}")
        metrics = opt_result.collect()[0]

        # VACUUM (30일 보존)
        spark.sql(f"VACUUM {full_name} RETAIN 720 HOURS")

        results.append({
            "table": full_name,
            "status": "SUCCESS",
            "files_added": metrics.metrics.numFilesAdded,
            "files_removed": metrics.metrics.numFilesRemoved,
            "timestamp": datetime.datetime.now()
        })
    except Exception as e:
        results.append({
            "table": full_name,
            "status": f"FAILED: {str(e)}",
            "timestamp": datetime.datetime.now()
        })

# 결과를 감사 테이블에 기록
results_df = spark.createDataFrame(results)
results_df.write.mode("append").saveAsTable("ops.maintenance.optimize_vacuum_log")
```

### 많은 팀이 하는 실수들

| 실수 | 결과 | 올바른 방법 |
|------|------|-----------|
| OPTIMIZE 후 VACUUM을 안 함 | OPTIMIZE가 새 파일을 만들지만, 이전 소형 파일이 그대로 남아 스토리지 낭비 | OPTIMIZE → VACUUM 순서로 함께 실행 |
| VACUUM 보존 기간을 0으로 설정 | 진행 중인 쿼리 실패, 타임 트래블 완전 불가 | 최소 7일(168시간) 유지 |
| 피크 시간에 OPTIMIZE 실행 | 대량의 I/O로 다른 쿼리 성능 저하 | 트래픽이 적은 시간(새벽)에 실행 |
| 모든 테이블에 같은 주기 적용 | 쓰기가 적은 테이블에 불필요한 OPTIMIZE, 쓰기가 많은 테이블에 부족한 OPTIMIZE | 테이블별 쓰기 패턴에 따라 차등 적용 |
| WHERE 조건 없이 대형 테이블 OPTIMIZE | 전체 데이터 재작성으로 수 시간 소요 + 비용 폭증 | `WHERE order_date >= current_date() - INTERVAL 7 DAYS` 등 범위 지정 |

---

## 정리

| 명령어 | 목적 | 효과 |
|--------|------|------|
| **OPTIMIZE**| 소형 파일 병합 | 쿼리 성능 향상 |
| **VACUUM**| 불필요한 파일 삭제 | 스토리지 비용 절약 |
| **ANALYZE TABLE** | 통계 정보 수집 | 쿼리 최적화기 성능 향상 |

---

## 참고 링크

- [Databricks: OPTIMIZE](https://docs.databricks.com/aws/en/delta/optimize.html)
- [Databricks: VACUUM](https://docs.databricks.com/aws/en/sql/language-manual/delta-vacuum.html)
- [Databricks: ANALYZE TABLE](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-analyze-table.html)
- [Azure Databricks: Optimize data file layout](https://learn.microsoft.com/en-us/azure/databricks/delta/optimize)
- [Databricks: Predictive Optimization](https://docs.databricks.com/aws/en/delta/predictive-optimization.html)
