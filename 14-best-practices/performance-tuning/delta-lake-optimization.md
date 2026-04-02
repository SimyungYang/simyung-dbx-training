# Delta Lake 성능 최적화

## 이 문서에서 다루는 내용

Databricks 플랫폼에서 쿼리, 파이프라인, ML 워크로드의 성능을 극대화하는 방법을 다룹니다. Delta Lake 데이터 레이아웃부터 Spark 쿼리 튜닝, SQL Warehouse 최적화, 파이프라인 처리량 개선, ML/AI 지연시간 최적화까지 실전 트러블슈팅 가이드를 제공합니다.

---

## 1. Delta Lake 성능 최적화

### 1.1 Liquid Clustering 키 선택 전략

Liquid Clustering은 기존의 파티셔닝과 Z-ORDER를 대체하는 현대적인 데이터 레이아웃 기술입니다. 올바른 키 선택이 성능을 좌우합니다.

**키 선택 의사결정 매트릭스**

| 고려 요소 | 좋은 키 | 나쁜 키 |
|----------|--------|--------|
| **쿼리 필터 빈도** | WHERE절에 자주 등장 | 거의 필터링되지 않음 |
| **카디널리티** | 중~고 (수천~수백만) | 극저 (boolean, 2-3값) 또는 극고 (UUID) |
| **상관관계** | 다른 키와 독립적 | 다른 키와 높은 상관 (둘 다 선택 불필요) |
| **데이터 타입** | Date, String, Integer | Array, Map, Struct |
| **키 개수** | 1~4개 | 5개 이상 (최대 4개 제한) |

```sql
-- 사례 1: 이커머스 트랜잭션 테이블
-- 쿼리 패턴: 날짜 범위 + 지역 필터가 대부분
ALTER TABLE sales.silver.transactions
CLUSTER BY (transaction_date, region);

-- 사례 2: IoT 센서 데이터
-- 쿼리 패턴: 디바이스별 + 시간 범위 조회
ALTER TABLE iot.silver.sensor_readings
CLUSTER BY (device_id, event_timestamp);

-- 사례 3: 쿼리 패턴이 자주 변경되는 테이블
-- Databricks가 자동으로 최적 키를 분석하여 결정
ALTER TABLE analytics.gold.user_behavior
CLUSTER BY AUTO;
```

**기존 테이블 마이그레이션 가이드**

| 현재 방식 | 마이그레이션 전략 |
|----------|----------------|
| `PARTITIONED BY (date)` | `CLUSTER BY (date)` — 파티션 키를 클러스터링 키로 |
| `ZORDER BY (col_a, col_b)` | `CLUSTER BY (col_a, col_b)` — Z-ORDER 키를 그대로 |
| 파티션 + Z-ORDER | `CLUSTER BY (partition_col, zorder_col)` — 둘 다 포함 |
| 없음 (최적화 안 함) | `CLUSTER BY AUTO` — 자동 분석 후 적용 |

> 💡 **핵심 장점**: Liquid Clustering은 키를 변경해도 **기존 데이터를 즉시 다시 쓸 필요가 없습니다**. 이후 OPTIMIZE 실행 시 점진적으로 새 키에 맞게 재배치됩니다.

### 1.2 파일 사이즈 최적화

Delta Lake의 최적 파일 크기는 **128MB ~ 1GB** 입니다. 이 범위를 벗어나면 성능이 저하됩니다.

| 문제 | 원인 | 증상 | 해결 방법 |
|------|------|------|----------|
| **Small File 문제** | 잦은 스트리밍 쓰기, 작은 배치 | 파일 수 과다, 메타데이터 오버헤드 | `OPTIMIZE` 실행 |
| **Large File 문제** | 과도한 OPTIMIZE, 큰 배치 | 프루닝 효과 감소, 메모리 압박 | 파일 크기 상한 설정 |

```sql
-- Small File 진단: 파일 수와 평균 크기 확인
DESCRIBE DETAIL catalog.schema.table_name;
-- numFiles, sizeInBytes를 확인하여 평균 파일 크기 계산
-- 평균 < 32MB면 OPTIMIZE 필요

-- OPTIMIZE로 파일 병합
OPTIMIZE catalog.schema.table_name;

-- 특정 파티션만 OPTIMIZE (대용량 테이블)
OPTIMIZE catalog.schema.table_name
WHERE event_date >= current_date() - INTERVAL 7 DAYS;
```

```python
# 스트리밍 쓰기 시 파일 크기 제어
(df.writeStream
  .option("maxBytesPerTrigger", "100m")  # 트리거당 최대 100MB 읽기
  .option("maxFilesPerTrigger", "1000")   # 트리거당 최대 파일 수
  .trigger(processingTime="30 seconds")   # 30초 간격 → 적당한 파일 크기
  .table("catalog.schema.streaming_table")
)
```

### 1.3 통계 수집 및 Data Skipping

Data Skipping은 쿼리 실행 시 불필요한 파일을 자동으로 건너뛰는 기능입니다. 정확한 통계 정보가 필수적입니다.

```sql
-- Delta 통계 재수집 (Runtime 14.3 LTS+)
ANALYZE TABLE catalog.schema.table_name COMPUTE DELTA STATISTICS;

-- 특정 컬럼에 대한 통계 수집
ALTER TABLE catalog.schema.table_name
SET TBLPROPERTIES('delta.dataSkippingStatsColumns' = 'event_date, user_id, region');

-- 통계 수집 대상 컬럼 수 확인 (기본: 처음 32개 컬럼)
-- Unity Catalog 관리형 테이블은 Predictive Optimization이 자동으로 관리
```

| 기능 | 동작 방식 | 성능 효과 |
|------|----------|----------|
| **Min/Max 통계** | 파일별 최소/최대값 기록 | WHERE절 조건으로 파일 단위 스킵 |
| **Null Count** | 파일별 NULL 수 기록 | IS NOT NULL 필터 최적화 |
| **Liquid Clustering** | 관련 데이터를 같은 파일에 배치 | Data Skipping 효과 극대화 |
| **ANALYZE TABLE** | 통계 강제 재수집 | 테이블 속성 변경 후 반드시 실행 |

---

## 2. Liquid Clustering 심층 가이드

### 2.1 Liquid Clustering vs Z-ORDER vs 파티셔닝

| 비교 항목 | 파티셔닝 | Z-ORDER | Liquid Clustering |
|----------|---------|---------|------------------|
| **적용 시점** | 테이블 생성 시 고정 | OPTIMIZE 시 수동 | 언제든 변경 가능 |
| **키 변경** | 불가 (재생성 필요) | 변경 가능 (전체 재정렬) | 변경 가능 (점진적) |
| **증분 적용** | N/A | 불가 (전체 테이블) | ✅ 새 데이터만 정렬 |
| **자동 최적화** | ❌ | ❌ | ✅ Predictive Optimization |
| **Small File 문제** | 고카디널리티 시 심각 | 없음 | 없음 |
| **쓰기 성능 영향** | 없음 | OPTIMIZE 시 높은 비용 | 최소 (증분) |

{% hint style="info" %}
**마이그레이션 권장**: 신규 테이블은 Liquid Clustering을 기본으로 사용하세요. 기존 파티셔닝/Z-ORDER 테이블은 점진적으로 전환하되, `CLUSTER BY AUTO` 를 먼저 적용하여 Databricks가 최적 키를 자동 분석하게 하는 것이 안전합니다.
{% endhint %}

### 2.2 Liquid Clustering 모니터링

```sql
-- 클러스터링 상태 확인
DESCRIBE DETAIL catalog.schema.table_name;
-- clusteringColumns: 현재 클러스터링 키
-- numFiles: 파일 수
-- sizeInBytes: 총 크기

-- 클러스터링 효과 측정: 동일 쿼리의 스캔량 비교
-- 클러스터링 적용 전
EXPLAIN FORMATTED
SELECT * FROM catalog.schema.table_name
WHERE event_date = '2026-03-01' AND region = 'KR';
-- "number of files read" 값을 기록

-- OPTIMIZE 실행 (클러스터링 적용)
OPTIMIZE catalog.schema.table_name;

-- 클러스터링 적용 후 동일 쿼리 실행
-- "number of files read" 값이 줄어들었는지 확인
```

### 2.3 클러스터링 키 튜닝 사례

```sql
-- 사례: 로그 테이블에서 클러스터링 키 최적화
-- 초기 설정: 날짜만
ALTER TABLE logs.silver.app_events CLUSTER BY (event_date);

-- 문제: user_id로 필터링하는 쿼리가 많지만 여전히 풀 스캔
-- 개선: 날짜 + user_id 조합
ALTER TABLE logs.silver.app_events CLUSTER BY (event_date, user_id);

-- OPTIMIZE 실행 (새 키에 맞게 재배치 시작)
OPTIMIZE logs.silver.app_events;

-- 결과: user_id 필터 쿼리의 스캔량 80% 감소
```

---

## 3. VACUUM 전략

### 3.1 VACUUM 기본 동작

VACUUM은 더 이상 Delta 트랜잭션 로그에서 참조되지 않는 오래된 파일을 삭제합니다.

```sql
-- 기본 VACUUM (7일 보존)
VACUUM catalog.schema.table_name RETAIN 168 HOURS;

-- 보존 기간 확인
DESCRIBE DETAIL catalog.schema.table_name;
-- 'delta.deletedFileRetentionDuration' 속성 확인

-- DRY RUN: 삭제 대상 파일 목록만 확인 (실제 삭제 안 함)
VACUUM catalog.schema.table_name RETAIN 168 HOURS DRY RUN;
```

### 3.2 보존 기간 전략

| 테이블 유형 | 권장 보존 기간 | 이유 |
|-----------|-------------|------|
| **실시간 스트리밍** | 24~48시간 | 파일 생성 빈번, 스토리지 비용 억제 |
| **일일 배치** | 7일 (기본값) | Time Travel 활용 여유 |
| **규제 데이터 (금융)** | 30일 | 감사/롤백 요구사항 |
| **ML Feature 테이블** | 7일 | 재학습 시 이전 버전 필요 가능 |
| **임시/실험 테이블** | 24시간 | 스토리지 절약 우선 |

```sql
-- 테이블별 보존 기간 설정
ALTER TABLE catalog.schema.streaming_events
SET TBLPROPERTIES ('delta.deletedFileRetentionDuration' = 'interval 48 hours');

ALTER TABLE catalog.schema.financial_transactions
SET TBLPROPERTIES ('delta.deletedFileRetentionDuration' = 'interval 30 days');

-- 로그 보존 기간 (트랜잭션 로그 자체)
ALTER TABLE catalog.schema.table_name
SET TBLPROPERTIES ('delta.logRetentionDuration' = 'interval 60 days');
```

{% hint style="warning" %}
**VACUUM 후 Time Travel 불가**: VACUUM으로 삭제된 파일에 해당하는 버전은 Time Travel로 접근할 수 없습니다. 규제 요건이 있는 테이블은 보존 기간을 충분히 길게 설정하세요.
{% endhint %}

### 3.3 VACUUM 자동화

```sql
-- 방법 1: Predictive Optimization (권장)
-- Unity Catalog 관리형 테이블에 대해 자동 VACUUM
ALTER CATALOG my_catalog ENABLE PREDICTIVE OPTIMIZATION;

-- 방법 2: 스케줄 Job으로 수동 자동화
-- 매일 실행하는 VACUUM Job (노트북에서)
```

```python
# VACUUM 자동화 노트북 예시
tables_to_vacuum = [
    ("catalog.schema.table1", 168),   # 7일 보존
    ("catalog.schema.table2", 48),    # 2일 보존
    ("catalog.schema.table3", 720),   # 30일 보존
]

for table_name, retain_hours in tables_to_vacuum:
    try:
        spark.sql(f"VACUUM {table_name} RETAIN {retain_hours} HOURS")
        print(f"✅ VACUUM 완료: {table_name}")
    except Exception as e:
        print(f"❌ VACUUM 실패: {table_name} - {e}")
```

---

## 4. Deletion Vectors 성능 영향

### 4.1 Deletion Vectors란?

Deletion Vectors는 DELETE/UPDATE/MERGE 시 파일을 즉시 다시 쓰지 않고, 삭제된 행의 비트맵을 별도로 기록하는 기술입니다.

| 항목 | 기존 방식 (Copy-on-Write) | Deletion Vectors |
|------|------------------------|-----------------|
| **DELETE 속도** | 전체 파일 재작성 | 비트맵만 기록 (10~100x 빠름) |
| **스토리지 사용** | 즉시 정리 | 삭제 행이 파일에 남음 (OPTIMIZE 시 정리) |
| **읽기 성능** | 최적 | 약간의 오버헤드 (비트맵 확인) |
| **활성화** | 기본 비활성화 (레거시) | Runtime 14.0+에서 기본 활성화 |

```sql
-- Deletion Vectors 상태 확인
DESCRIBE DETAIL catalog.schema.table_name;
-- 'delta.enableDeletionVectors' = 'true' 확인

-- Deletion Vectors 활성화 (레거시 테이블)
ALTER TABLE catalog.schema.table_name
SET TBLPROPERTIES ('delta.enableDeletionVectors' = true);

-- Deletion Vectors 누적 시 성능 저하 해소
-- OPTIMIZE가 Deletion Vectors를 물리적으로 정리
OPTIMIZE catalog.schema.table_name;
```

### 4.2 Deletion Vectors 모니터링

```sql
-- Deletion Vectors가 누적된 파일 비율 확인
-- DESCRIBE DETAIL의 numFiles와 실제 유효 행 수 비교

-- 대량 DELETE 후 읽기 성능이 저하되면 OPTIMIZE 실행
-- 일반적으로 삭제 비율이 10% 이상이면 OPTIMIZE 권장
```

{% hint style="info" %}
**Deletion Vectors + MERGE 패턴**: SCD Type 2 MERGE에서 Deletion Vectors가 큰 효과를 발휘합니다. 기존에 수십 분 걸리던 MERGE 작업이 수 분으로 단축됩니다. Predictive Optimization이 활성화되어 있으면 누적된 Deletion Vectors는 자동으로 정리됩니다.
{% endhint %}

---

## 5. Predictive Optimization 설정 및 모니터링

### 5.1 활성화 범위

```sql
-- 카탈로그 수준 활성화 (하위 모든 스키마/테이블에 적용)
ALTER CATALOG prod_sales ENABLE PREDICTIVE OPTIMIZATION;

-- 스키마 수준 활성화
ALTER SCHEMA prod_sales.gold ENABLE PREDICTIVE OPTIMIZATION;

-- 특정 스키마 비활성화 (예: 임시 데이터)
ALTER SCHEMA prod_sales.sandbox DISABLE PREDICTIVE OPTIMIZATION;

-- 비활성화한 객체의 하위 객체도 비활성화됨
-- 상위가 활성화되어도 하위에서 명시적으로 비활성화 가능
```

### 5.2 모니터링

```sql
-- Predictive Optimization 작업 이력 확인
SELECT
  table_name,
  operation_type,    -- OPTIMIZE, VACUUM, ANALYZE
  operation_status,  -- SUCCESSFUL, FAILED, SKIPPED
  operation_metrics, -- 처리한 파일 수, 크기 등
  timestamp
FROM system.storage.predictive_optimization_operations_history
WHERE timestamp >= current_date() - INTERVAL 7 DAYS
ORDER BY timestamp DESC;

-- 비용 확인 (Serverless Jobs SKU로 과금)
SELECT
  date_trunc('day', timestamp) AS day,
  operation_type,
  COUNT(*) AS num_operations,
  SUM(operation_metrics.dbus_consumed) AS total_dbus
FROM system.storage.predictive_optimization_operations_history
WHERE timestamp >= current_date() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY 1 DESC;
```

| Predictive Optimization 작업 | 실행 조건 | 효과 |
|---------------------------|---------|------|
| **Auto-OPTIMIZE** | Small File 감지 시 | 파일 병합, 읽기 성능 향상 |
| **Auto-VACUUM** | 참조 없는 파일 감지 시 | 스토리지 비용 절감 |
| **Auto-ANALYZE** | 통계 불일치 감지 시 | 쿼리 옵티마이저 정확도 향상 |
| **Auto-Compaction** | 스트리밍 테이블 Small File | 실시간 파일 병합 |

{% hint style="info" %}
**비용 대비 효과**: Predictive Optimization의 비용은 일반적으로 수동 OPTIMIZE/VACUUM 실행 비용의 30~50%입니다. 자동 실행이므로 관리 인력 비용도 절감됩니다. 2024년 11월 이후 생성된 계정에는 기본 활성화되어 있습니다.
{% endhint %}

---

## 6. 파일 크기 최적화 상세

### 6.1 워크로드별 최적 파일 크기

| 워크로드 | 권장 파일 크기 | 설정 방법 |
|---------|-------------|----------|
| **대규모 ETL (TB 단위)** | 512MB ~ 1GB | `spark.databricks.delta.optimizeWrite.fileSize = 536870912` |
| **대시보드/BI 쿼리** | 128MB ~ 256MB | 기본값 유지 |
| **스트리밍 (초 단위)** | 64MB ~ 128MB | Auto-compaction + OPTIMIZE |
| **ML Feature 테이블** | 32MB ~ 64MB | 소규모 테이블, 빈번한 읽기 |

```python
# 쓰기 시 파일 크기 제어
# Optimize Write: 자동으로 적절한 크기의 파일 생성
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")

# 파일 크기 목표 설정 (바이트 단위)
spark.conf.set("spark.databricks.delta.optimizeWrite.fileSize", "134217728")  # 128MB

# Auto Compaction: 커밋 후 자동으로 Small File 병합
spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
spark.conf.set("spark.databricks.delta.autoCompact.minNumFiles", "20")  # 20개 이상이면 병합
```

---

## 참고 링크

- [Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering)
- [Data Skipping](https://docs.databricks.com/aws/en/delta/data-skipping)
- [VACUUM](https://docs.databricks.com/aws/en/sql/language-manual/delta-vacuum)
- [Deletion Vectors](https://docs.databricks.com/aws/en/delta/deletion-vectors)
- [Predictive Optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
- [Optimize Write](https://docs.databricks.com/aws/en/delta/optimize-write)

---
