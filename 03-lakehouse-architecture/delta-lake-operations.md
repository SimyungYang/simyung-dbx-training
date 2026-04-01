# Delta Lake 실전 — MERGE, OPTIMIZE, VACUUM

## 이 문서에서 다루는 내용

Delta Lake의 핵심 개념을 이해하셨다면, 이제 실무에서 자주 사용하는 **고급 조작법** 을 살펴보겠습니다. 이 기능들은 데이터 파이프라인을 안정적이고 효율적으로 운영하는 데 필수적입니다.

---

## MERGE (Upsert)

### 개념

> 💡 **MERGE**는 "있으면 업데이트하고(UPDATE), 없으면 삽입한다(INSERT)"를 하나의 명령으로 수행하는 작업입니다. 이를 **Upsert(Update + Insert)** 라고도 부릅니다.

전통적인 데이터 레이크에서는 UPDATE가 불가능하여, 전체 테이블을 다시 써야 했습니다. Delta Lake의 MERGE는 이 문제를 우아하게 해결합니다.

### 문법

```sql
MERGE INTO target_table AS target
USING source_table AS source
ON target.id = source.id
WHEN MATCHED THEN
    UPDATE SET target.name = source.name,
               target.amount = source.amount,
               target.updated_at = current_timestamp()
WHEN NOT MATCHED THEN
    INSERT (id, name, amount, created_at, updated_at)
    VALUES (source.id, source.name, source.amount, current_timestamp(), current_timestamp());
```

### 실습 예제: 고객 정보 동기화

```sql
-- 소스 시스템에서 변경된 고객 데이터가 도착했다고 가정합니다
CREATE OR REPLACE TEMP VIEW updated_customers AS
SELECT * FROM VALUES
    (1, '김철수', 'cs.kim@newmail.com', '서울'),      -- 기존 고객: 이메일 변경
    (4, '한지민', 'jm.han@email.com', '대전')          -- 신규 고객
AS t(customer_id, name, email, city);

-- MERGE 실행
MERGE INTO catalog.schema.customers AS target
USING updated_customers AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN
    UPDATE SET
        target.name = source.name,
        target.email = source.email,
        target.city = source.city
WHEN NOT MATCHED THEN
    INSERT (customer_id, name, email, city, signup_date)
    VALUES (source.customer_id, source.name, source.email, source.city, current_date());
```

### MERGE의 활용 패턴

| 패턴 | 설명 |
|------|------|
| **Upsert**| 있으면 UPDATE, 없으면 INSERT (가장 일반적) |
| **SCD Type 1**| 최신 값으로 덮어쓰기 (이력 보존 안 함) |
| **SCD Type 2**| 변경 이력을 모두 보존 (유효 기간 관리) |
| **Deduplication**| 중복 데이터 제거 |
| **Delete 동기화**| 소스에서 삭제된 레코드를 타겟에서도 삭제 |

> 💡 **SCD(Slowly Changing Dimension, 완만하게 변하는 차원)란?** 데이터 웨어하우스에서 시간에 따라 변하는 데이터(예: 고객 주소, 상품 가격)를 어떻게 관리할지에 대한 전략입니다.
> - **Type 1**: 변경 시 이전 값을 덮어씁니다. 이력을 보존하지 않습니다.
> - **Type 2**: 변경 시 이전 레코드를 만료 처리하고 새 레코드를 추가합니다. 전체 이력을 보존합니다.

---

## OPTIMIZE (데이터 압축)

### 개념

> 💡 **OPTIMIZE**는 Delta 테이블의 작은 파일들을 **큰 파일로 합치는(Compaction)** 작업입니다. 데이터를 지속적으로 추가하다 보면 작은 파일이 매우 많아지는데, 이를 적절한 크기로 합치면 쿼리 성능이 크게 향상됩니다.

### 왜 필요한가요?

| 최적화 전 | 최적화 후 (OPTIMIZE) |
|-----------|---------------------|
| 1MB, 500KB, 2MB, 100KB, 3MB, 200KB (작은 파일 6개) | 128MB (최적화된 파일 1개) |

작은 파일이 많으면 읽기 성능이 저하됩니다. OPTIMIZE로 적절한 크기(128MB~1GB)로 병합합니다.

> 💡 **Small File Problem(작은 파일 문제)이란?** 스트리밍 수집이나 빈번한 INSERT로 인해 테이블에 수천~수만 개의 작은 파일이 쌓이는 현상입니다. 쿼리 시 각 파일을 열고 닫는 오버헤드가 누적되어 성능이 크게 저하됩니다. OPTIMIZE는 이 문제를 해결합니다.

### 사용 방법

```sql
-- 테이블 전체 최적화
OPTIMIZE catalog.schema.orders;

-- 특정 파티션만 최적화
OPTIMIZE catalog.schema.orders
WHERE order_date >= '2025-03-01';
```

### Liquid Clustering

> 🆕 **Liquid Clustering** 은 Databricks가 최근 도입한 차세대 데이터 배치 최적화 기술입니다. 기존의 Z-Order와 파티셔닝을 대체하며, 데이터를 자동으로 최적의 레이아웃으로 재배치합니다.

```sql
-- Liquid Clustering 활성화하여 테이블 생성
CREATE TABLE catalog.schema.orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    amount DECIMAL(10,2)
) CLUSTER BY (order_date, customer_id);

-- OPTIMIZE 실행 시 자동으로 Liquid Clustering 적용
OPTIMIZE catalog.schema.orders;
```

| 비교 항목 | 파티셔닝 (기존) | Z-Order (기존) | Liquid Clustering (최신) |
|-----------|----------------|----------------|------------------------|
| 설정 방법 | `PARTITIONED BY` | `OPTIMIZE ... ZORDER BY` | `CLUSTER BY` |
| 컬럼 변경 | 테이블 재생성 필요 | 매번 지정 | `ALTER TABLE ... CLUSTER BY` |
| 동작 방식 | 물리적 디렉토리 분리 | 파일 내 정렬 | 증분 자동 재배치 |
| 추천 여부 | 레거시 | 레거시 | ✅ 신규 테이블에 권장 |

> 💡 **파티셔닝(Partitioning)이란?** 데이터를 특정 컬럼 값(예: 날짜)에 따라 물리적으로 다른 디렉토리에 저장하는 방식입니다. `order_date = 2025-03-15` 데이터는 `/order_date=2025-03-15/` 디렉토리에 저장되므로, 특정 날짜만 조회할 때 해당 디렉토리만 읽으면 됩니다. 다만 카디널리티(고유값 수)가 높은 컬럼으로 파티셔닝하면 오히려 작은 파일이 많아지는 문제가 생깁니다.

---

## VACUUM (오래된 파일 정리)

### 개념

> 💡 **VACUUM**은 Delta 테이블에서 **더 이상 사용되지 않는 오래된 데이터 파일** 을 삭제하여 스토리지 비용을 절약하는 작업입니다.

Delta Lake에서 UPDATE나 DELETE를 실행하면, 기존 파일이 즉시 삭제되지 않고 새 파일이 추가됩니다 (타임 트래블을 위해). VACUUM은 일정 기간이 지난 오래된 파일을 정리합니다.

| 파일 | VACUUM 전 | VACUUM 후 (7일 기준) |
|------|-----------|---------------------|
| Version 0 파일 (7일 전) | 존재 | 삭제됨 |
| Version 1 파일 (5일 전) | 존재 | 삭제됨 |
| Version 2 파일 (3일 전) | 존재 | 유지 |
| Version 3 파일 (현재) | 존재 | 유지 |

### 사용 방법

```sql
-- 기본: 7일(168시간)보다 오래된 파일 삭제
VACUUM catalog.schema.orders;

-- 보존 기간 지정: 30일보다 오래된 파일만 삭제
VACUUM catalog.schema.orders RETAIN 720 HOURS;

-- 삭제될 파일 미리 확인 (DRY RUN)
VACUUM catalog.schema.orders DRY RUN;
```

> ⚠️ **주의사항**: VACUUM을 실행하면 해당 기간 이전의 타임 트래블이 불가능해집니다. 예를 들어, `RETAIN 168 HOURS`로 VACUUM을 실행하면 7일 이전의 데이터는 더 이상 타임 트래블로 조회할 수 없습니다. 규정 준수 요건에 따라 보존 기간을 적절히 설정하시기 바랍니다.

---

## DELETE와 UPDATE

### DELETE

```sql
-- 조건에 맞는 행 삭제
DELETE FROM catalog.schema.orders
WHERE status = 'CANCELLED'
  AND order_date < '2024-01-01';
```

### UPDATE

```sql
-- 조건에 맞는 행 수정
UPDATE catalog.schema.orders
SET status = 'REFUNDED',
    updated_at = current_timestamp()
WHERE order_id = 1001;
```

> 💡 일반 데이터 레이크(Parquet 파일)에서는 DELETE/UPDATE가 불가능합니다. Delta Lake이기에 가능한 기능이며, 내부적으로는 해당 데이터를 포함하는 파일을 새로 쓰는 방식(Copy-on-Write)으로 동작합니다.

---

## 테이블 정보 확인 명령어

```sql
-- 테이블의 물리적 정보 (파일 수, 크기, 파티션 등)
DESCRIBE DETAIL catalog.schema.orders;

-- 테이블 변경 이력
DESCRIBE HISTORY catalog.schema.orders;

-- 테이블의 컬럼 정보
DESCRIBE TABLE catalog.schema.orders;

-- 테이블 생성 DDL 확인
SHOW CREATE TABLE catalog.schema.orders;
```

---

## 운영 모범 사례

| 작업 | 권장 빈도 | 설명 |
|------|-----------|------|
| **OPTIMIZE**| 일 1회 또는 데이터 변경 후 | 작은 파일을 합쳐 쿼리 성능을 유지합니다 |
| **VACUUM**| 주 1회 | 오래된 파일을 삭제하여 스토리지 비용을 절약합니다 |
| **ANALYZE TABLE**| 주 1회 | 통계 정보를 갱신하여 쿼리 최적화기의 성능을 높입니다 |

```sql
-- 추천 운영 스크립트
OPTIMIZE catalog.schema.orders;
VACUUM catalog.schema.orders RETAIN 720 HOURS;
ANALYZE TABLE catalog.schema.orders COMPUTE STATISTICS;
```

> 🆕 **최신 기능**: Databricks는 **Predictive Optimization** 을 통해 OPTIMIZE와 VACUUM을 자동으로 실행하는 기능을 제공하고 있습니다. Unity Catalog가 활성화된 관리형(Managed) 테이블에서는 Databricks가 테이블의 상태를 모니터링하고, 최적의 시점에 자동으로 최적화를 수행합니다.

```sql
-- Predictive Optimization 활성화
ALTER TABLE catalog.schema.orders
SET TBLPROPERTIES ('delta.enableOptimizeWrite' = 'true');
```

---

## 현업 사례: MERGE 성능이 갑자기 10배 느려진 원인 분석

현업에서 MERGE는 가장 많이 사용하는 Delta Lake 명령어이면서, 동시에 **가장 많은 성능 이슈를 일으키는** 명령어입니다. 실제 사례를 통해 MERGE의 내부 동작을 깊이 이해하겠습니다.

### 사례: 전자상거래 회사의 주문 테이블

주문 상태를 업데이트하는 MERGE가 평소 5분 걸리다가, 어느 날 갑자기 50분으로 늘어났습니다.

```
[상황]
- 타겟 테이블: orders (10억 건, 500GB, 파티션 없음)
- 소스 데이터: 일일 변경분 50만 건
- MERGE 조건: ON target.order_id = source.order_id
- 평소 실행 시간: 5분
- 문제 발생 후: 50분 (10배 느려짐)
```

### MERGE의 내부 동작 이해

MERGE는 내부적으로 3단계로 동작합니다. 각 단계를 이해하면 성능 문제의 원인을 찾을 수 있습니다.

| 단계 | 동작 | 최적화 팁 |
|------|------|----------|
| **1단계: Scan**| 타겟 테이블의 모든 파일을 스캔하여 매칭할 행을 찾습니다. 비용 = 타겟 테이블 크기에 비례 | Liquid Clustering이나 파티션으로 스캔 범위를 줄입니다 |
| **2단계: Match**| ON 조건으로 소스와 타겟의 매칭 행을 식별합니다 (Inner/Outer Join 발생) | 매칭 키에 인덱스/클러스터링이 있으면 빠릅니다 |
| **3단계: Write**| 매칭된 파일만 새로 씁니다 (Copy-on-Write). 변경 1건이라도 해당 파일 전체 재작성 | 파일이 적절히 클러스터링되면 재작성할 파일 수가 줄어듭니다 |

### 원인 분석: 왜 10배 느려졌는가

```
[원인 1: Small File 문제]
- OPTIMIZE를 2주간 안 돌렸음
- 1MB 미만 파일이 50,000개 → 파일 열기/닫기 오버헤드 폭증

[원인 2: 클러스터링 없음]
- order_id 기준 MERGE인데, 데이터가 order_id 순서로 정렬되어 있지 않음
- 50만 건 변경이 500GB 전체 파일에 골고루 분포 → 거의 모든 파일을 재작성

[원인 3: 소스 데이터 급증]
- 블랙프라이데이 이벤트로 일일 변경분이 50만 → 500만 건으로 10배 증가
```

### 대규모 MERGE 최적화 전략

| 전략 | 효과 | 적용 방법 |
|------|------|----------|
| **Liquid Clustering**| 스캔 범위 축소 (10~100배) | `CLUSTER BY (merge_key)` — MERGE ON 조건의 키를 클러스터링 키로 설정 |
| **정기 OPTIMIZE**| Small File 제거 | 매일 OPTIMIZE 실행 (또는 Predictive Optimization 활성화) |
| **소스 필터링**| 매칭 대상 축소 | MERGE 전에 소스에서 불필요한 행을 미리 필터 |
| **조건부 MERGE**| 스캔 범위 한정 | `MERGE INTO ... WHERE target.order_date >= '2025-03-01'` |
| **배치 분할** | 메모리 부족 방지 | 500만 건을 50만 건씩 10회로 분할 실행 |

```sql
-- 최적화된 MERGE 패턴
-- 1. 타겟 테이블에 Liquid Clustering 적용
ALTER TABLE catalog.schema.orders CLUSTER BY (order_date, order_id);

-- 2. MERGE에 조건 추가로 스캔 범위 한정
MERGE INTO catalog.schema.orders AS target
USING daily_changes AS source
ON target.order_id = source.order_id
   AND target.order_date >= current_date() - INTERVAL 30 DAYS  -- 30일 이내만 스캔
WHEN MATCHED THEN
    UPDATE SET target.status = source.status,
               target.updated_at = current_timestamp()
WHEN NOT MATCHED THEN
    INSERT *;
```

### UPSERT 패턴의 현실적 고려사항

| 고려사항 | 설명 | 권장 |
|---------|------|------|
| **멱등성(Idempotency)**| 같은 MERGE를 2번 실행해도 결과가 같아야 합니다 | `WHEN MATCHED AND source.updated_at > target.updated_at` 조건 추가 |
| **DELETE 동기화**| 소스에서 삭제된 레코드를 어떻게 처리할 것인가 | Soft Delete (`is_deleted` 플래그) 권장. Hard Delete는 리스크가 큼 |
| **Late Arriving Data**| 3일 전 주문의 상태가 오늘에야 도착하는 경우 | MERGE 조건에 날짜 범위를 충분히 넓게 설정 (예: 7일) |
| **중복 소스 데이터**| CDC에서 같은 키의 이벤트가 여러 번 올 수 있음 | MERGE 전에 소스에서 `ROW_NUMBER()` 등으로 중복 제거 |
| **대용량 초기 적재**| 빈 테이블에 10억 건을 MERGE하면 극도로 느림 | 초기 적재는 `INSERT INTO`로, 이후부터 MERGE 사용 |

> ⚠️ **현업에서는 이렇게 합니다**: MERGE의 성능은 "얼마나 많은 파일을 재작성하느냐"에 달려있습니다. Liquid Clustering으로 MERGE 키 기준 데이터를 모아두면, 변경 대상 파일만 재작성하므로 10~100배 빨라집니다. 이것을 안 하면 500GB 테이블에서 1건만 UPDATE해도 수십 GB의 파일을 재작성하게 됩니다.

> 💡 **숫자로 이해하기**: 10억 건 테이블(500GB, 파일 5,000개)에서 50만 건 MERGE 시:
> - 클러스터링 없음: 5,000개 파일 중 4,800개 재작성 (96%) → **50분**
> - Liquid Clustering(order_id): 5,000개 파일 중 250개 재작성 (5%) → **3분**

### MERGE 운영 시 반드시 모니터링해야 할 지표

현업에서 MERGE 파이프라인을 안정적으로 운영하려면 다음 지표를 모니터링해야 합니다.

```sql
-- MERGE 실행 후 확인해야 할 핵심 지표
DESCRIBE HISTORY catalog.schema.orders LIMIT 5;

-- 확인 항목:
-- operationMetrics.numTargetRowsInserted: 삽입된 행 수
-- operationMetrics.numTargetRowsUpdated: 업데이트된 행 수
-- operationMetrics.numTargetFilesAdded: 새로 생성된 파일 수
-- operationMetrics.numTargetFilesRemoved: 재작성된 파일 수 ← 이것이 핵심!
-- operationMetrics.executionTimeMs: 실행 시간 (ms)
```

| 모니터링 지표 | 정상 범위 | 이상 신호 | 조치 |
|-------------|---------|---------|------|
| **재작성 파일 비율**| <20% | >50% | Liquid Clustering 확인, OPTIMIZE 실행 |
| **실행 시간 추이**| 일정 | 주 단위로 증가 | Small File 문제, 데이터 볼륨 변화 확인 |
| **삽입 행 수**| 예상 범위 내 | 0건 또는 급증 | 소스 데이터 문제 확인 |
| **업데이트 행 수**| 예상 범위 내 | 급증 | 중복 소스 데이터 확인 |

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **MERGE**| Upsert(있으면 UPDATE, 없으면 INSERT)를 한 번에 수행합니다 |
| **OPTIMIZE**| 작은 파일들을 합쳐서 쿼리 성능을 향상시킵니다 |
| **Liquid Clustering**| 차세대 데이터 레이아웃 최적화. 기존 파티셔닝/Z-Order를 대체합니다 |
| **VACUUM**| 불필요한 오래된 파일을 삭제하여 스토리지 비용을 절약합니다 |
| **Predictive Optimization**| OPTIMIZE/VACUUM을 Databricks가 자동으로 실행합니다 |

다음 문서에서는 Delta Lake와 **Apache Iceberg** 의 관계 및 상호 운용성을 살펴보겠습니다.

---

## 참고 링크

- [Databricks: MERGE INTO](https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into.html)
- [Databricks: OPTIMIZE](https://docs.databricks.com/aws/en/sql/language-manual/delta-optimize.html)
- [Databricks: VACUUM](https://docs.databricks.com/aws/en/sql/language-manual/delta-vacuum.html)
- [Databricks: Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering.html)
- [Databricks: Predictive Optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization.html)
