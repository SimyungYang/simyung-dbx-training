# Deletion Vectors 상세

## 왜 Deletion Vectors가 필요한가요?

Delta Lake에서 데이터를 삭제(DELETE)하거나 수정(UPDATE)할 때, 내부적으로는 어떤 일이 벌어질까요? 기존 방식에서는 변경 대상이 포함된 파일 전체를 **다시 작성(rewrite)**해야 했습니다. 1GB 파일에서 단 1행만 삭제해도, 나머지 999,999행을 포함한 새 파일을 써야 했던 것입니다.

> 💡 **Deletion Vectors(삭제 벡터)** 는 이러한 비효율을 해결하기 위해 도입된 기술입니다. 파일을 다시 쓰지 않고, **어떤 행이 삭제되었는지를 별도의 파일에 기록**하는 방식으로 DELETE, UPDATE, MERGE의 성능을 크게 향상시킵니다.

---

## Copy-on-Write vs Deletion Vectors

### Copy-on-Write (기존 방식)

기존 Delta Lake는 **Copy-on-Write(CoW)** 방식을 사용했습니다.

```
DELETE FROM orders WHERE order_id = 42;

Copy-on-Write 동작:
1. order_id=42가 포함된 파일(1GB)을 찾음
2. 해당 파일의 모든 행을 읽음
3. order_id=42를 제외한 나머지 행으로 새 파일(~1GB) 작성
4. 기존 파일을 "제거됨"으로 로그에 기록
5. 새 파일을 "추가됨"으로 로그에 기록

→ 1행 삭제에 ~1GB 읽기 + ~1GB 쓰기 발생
```

### Deletion Vectors (새 방식)

```
DELETE FROM orders WHERE order_id = 42;

Deletion Vectors 동작:
1. order_id=42가 포함된 파일을 찾음
2. 해당 파일에서 order_id=42의 행 번호(row position)를 확인
3. 삭제된 행 번호만 기록한 작은 DV 파일 생성 (~수 KB)

→ 1행 삭제에 메타데이터만 쓰기 발생 (데이터 파일 재작성 없음)
```

### 성능 비교

| 비교 항목 | Copy-on-Write | Deletion Vectors |
|-----------|--------------|-----------------|
| **DELETE 1행** | 전체 파일 재작성 (~GB 단위) | DV 파일만 생성 (~KB 단위) |
| **쓰기 비용** | 높음 | 매우 낮음 |
| **읽기 비용** | 변화 없음 | 약간 증가 (DV 확인 필요) |
| **MERGE 성능** | 대량 파일 재작성 | 변경 행만 처리 |
| **UPDATE 성능** | 파일 전체 재작성 | DV + 새 행만 작성 |

---

## 동작 원리 상세

### Deletion Vector 파일

Deletion Vector는 **비트맵(bitmap)** 형태로 저장됩니다. 각 비트가 파일 내 한 행에 대응하며, 삭제된 행은 1, 유효한 행은 0으로 표시됩니다.

```
Parquet 파일: part-00001.parquet (100만 행)
DV 파일:      part-00001.deletion_vector.bin

비트맵 예시: 0000...0001...0000...0010...0000
             ↑              ↑
          행 #42 삭제     행 #9999 삭제
```

### 읽기 시 동작

쿼리를 실행할 때, Delta Lake는 Parquet 파일과 함께 해당 파일의 Deletion Vector를 읽어 **삭제된 행을 자동으로 필터링**합니다.

```sql
-- 사용자 입장에서는 완전히 투명합니다
SELECT * FROM catalog.schema.orders;
-- Deletion Vector에 의해 삭제 표시된 행은 자동으로 제외됩니다
```

### UPDATE에서의 동작

UPDATE는 내부적으로 "기존 행 삭제 + 새 행 추가"로 동작합니다.

```
UPDATE orders SET amount = 100 WHERE order_id = 42;

Deletion Vectors 활성화 시:
1. order_id=42가 있는 파일의 DV에 해당 행을 삭제 표시
2. 수정된 새 행(order_id=42, amount=100)을 새 파일에 작성

→ 기존 파일은 그대로 유지, DV + 작은 새 파일만 생성
```

---

## 설정 방법

### 활성화

```sql
-- 테이블 생성 시 활성화
CREATE TABLE catalog.schema.orders (
    order_id BIGINT,
    customer_id BIGINT,
    amount DECIMAL(10, 2)
) TBLPROPERTIES ('delta.enableDeletionVectors' = true);

-- 기존 테이블에 활성화
ALTER TABLE catalog.schema.orders
SET TBLPROPERTIES ('delta.enableDeletionVectors' = true);
```

> 💡 Databricks Runtime 14.0 이상에서 생성된 Delta 테이블은 **기본적으로 Deletion Vectors가 활성화**되어 있습니다. 별도의 설정 없이 자동으로 적용됩니다.

### 비활성화

```sql
-- 호환성 문제 등으로 비활성화가 필요한 경우
ALTER TABLE catalog.schema.orders
SET TBLPROPERTIES ('delta.enableDeletionVectors' = false);
```

---

## OPTIMIZE와의 관계

Deletion Vectors가 쌓이면 읽기 시 DV를 확인하는 오버헤드가 증가할 수 있습니다. **OPTIMIZE**를 실행하면 DV가 적용된 파일들이 새로 작성되면서, Deletion Vectors가 **물리적으로 반영(materialized)**됩니다.

```sql
-- OPTIMIZE 실행 시 DV가 적용된 파일이 정리됩니다
OPTIMIZE catalog.schema.orders;

-- OPTIMIZE 전: Parquet 파일 + DV 파일 (논리적 삭제)
-- OPTIMIZE 후: 삭제된 행이 제거된 새 Parquet 파일 (물리적 삭제)
```

> 💡 Deletion Vectors는 **쓰기 성능을 즉시 개선**하지만, DV가 많이 쌓이면 읽기 성능이 약간 저하될 수 있습니다. 정기적인 OPTIMIZE로 DV를 정리하는 것이 좋습니다. **Predictive Optimization**이 활성화되어 있다면 이 작업도 자동으로 처리됩니다.

---

## 호환성 고려사항

| 항목 | 설명 |
|------|------|
| **Databricks Runtime** | 12.1 이상에서 지원, 14.0 이상에서 기본 활성화 |
| **Delta Lake OSS** | Delta Lake 2.3 이상에서 지원 |
| **외부 리더** | DV를 지원하지 않는 리더는 삭제된 행이 보일 수 있음 |
| **UniForm** | Deletion Vectors와 UniForm은 함께 사용 가능 |
| **Liquid Clustering** | Deletion Vectors와 함께 사용 시 최적의 성능 |

> ⚠️ **외부 엔진 호환성**: Databricks 외부에서 Delta 테이블을 직접 읽는 경우, 해당 리더가 Deletion Vectors를 지원하는지 확인해야 합니다. 지원하지 않으면 삭제된 행이 결과에 포함될 수 있습니다. UniForm을 통해 Iceberg로 읽는 경우에는 이 문제가 발생하지 않습니다.

---

## DV 내부 저장 구조 — RoaringBitmap

Deletion Vectors는 내부적으로 **RoaringBitmap** 데이터 구조를 사용하여 삭제된 행의 위치를 매우 효율적으로 저장합니다.

### RoaringBitmap이란?

> 💡 **RoaringBitmap**은 정수 집합을 메모리 효율적으로 저장하는 압축 비트맵 데이터 구조입니다. 일반 비트맵보다 훨씬 적은 공간을 사용하면서도 빠른 집합 연산(합집합, 교집합)을 지원합니다.

```
일반 비트맵 (100만 행, 3행 삭제):
  000000...0001...000000...0010...000000...0001...
  → 100만 비트 = 125KB (대부분이 0으로 낭비)

RoaringBitmap (동일 상황):
  Container[42] → {42}
  Container[9999] → {9999}
  Container[500000] → {500000}
  → 수십 바이트 (삭제된 행 수에 비례)
```

RoaringBitmap은 32비트 정수를 상위 16비트(Container 키)와 하위 16비트(Container 내 값)로 분리하여 저장합니다. 삭제 비율이 낮을수록 압축률이 높아지므로, 일반적인 DELETE/UPDATE 패턴(전체 대비 소수 행 변경)에 매우 적합합니다.

### DV 파일의 물리적 저장

DV는 두 가지 방식으로 저장될 수 있습니다.

| 저장 방식 | 설명 | 사용 시점 |
|-----------|------|----------|
| **인라인(Inline)** | Delta 트랜잭션 로그에 직접 포함 | 삭제된 행이 소수일 때 (수십 바이트) |
| **독립 파일(On-disk)** | 별도 `.bin` 파일로 저장 | 삭제된 행이 많을 때 (수 KB~수 MB) |

```
_delta_log/
  00000000000000000042.json  ← 트랜잭션 로그
    {
      "remove": {"path": "part-00001.parquet", "deletionVector": {
        "storageType": "u",       // "u" = UUID 기반 파일 참조
        "pathOrInlineDv": "ab12cd34...",  // DV 파일 UUID
        "offset": 0,
        "sizeInBytes": 48,
        "cardinality": 3           // 삭제된 행 수
      }}
    }
```

---

## Copy-on-Write vs Merge-on-Read 성능 특성 심층 분석

DV의 도입으로 Delta Lake는 사실상 **Merge-on-Read(MoR)** 방식을 채택하게 되었습니다. 두 방식의 성능 특성을 워크로드별로 비교합니다.

### 워크로드별 성능 비교

| 워크로드 | Copy-on-Write | Merge-on-Read (DV) | 권장 |
|----------|:------------:|:------------------:|:----:|
| **대량 DELETE (테이블 50% 이상)** | ★★★ | ★★☆ | CoW |
| **소량 DELETE (< 1%)** | ★☆☆ | ★★★ | DV |
| **CDC MERGE (소량 변경)** | ★★☆ | ★★★ | DV |
| **대량 UPDATE (파일 대부분 변경)** | ★★★ | ★★☆ | CoW |
| **포인트 UPDATE (단일 행)** | ★☆☆ | ★★★ | DV |
| **스캔 중심 읽기** | ★★★ | ★★☆ | 상황에 따라 |
| **포인트 조회 (키 기반)** | ★★★ | ★★★ | 동일 |

### 읽기 오버헤드 정량 분석

DV가 활성화된 상태에서 읽기 시 추가되는 오버헤드입니다.

```
읽기 과정:
1. Parquet 파일 로드 (기존과 동일)
2. DV 파일 로드 (추가 I/O, 수 KB)
3. RoaringBitmap 적용하여 삭제 행 필터링 (CPU, 마이크로초 수준)

실측 오버헤드:
  - DV 없는 파일: 0ms 추가
  - DV < 1KB: ~0.1ms 추가
  - DV 1~100KB: ~0.5ms 추가
  - DV > 1MB: 1~5ms 추가 (OPTIMIZE 필요 신호)
```

> 💡 **핵심 교훈**: DV의 읽기 오버헤드는 대부분의 경우 무시할 수 있는 수준이지만, DV가 **매우 크게 누적**되면 의미 있는 성능 저하가 발생합니다. 이때가 OPTIMIZE를 실행해야 하는 시점입니다.

---

## DV + Photon 최적화

Photon 엔진은 DV를 네이티브로 지원하며, DV 기반 연산에 대해 추가적인 최적화를 제공합니다.

| 최적화 | 설명 |
|--------|------|
| **벡터화된 DV 필터링** | RoaringBitmap을 SIMD 연산으로 처리하여 CPU 오버헤드를 최소화합니다 |
| **프리페치** | Parquet 파일과 DV 파일을 동시에 프리페치하여 I/O 대기를 줄입니다 |
| **DV-aware 파일 건너뛰기** | DV로 인해 모든 행이 삭제된 파일은 아예 읽지 않습니다 |
| **최적화된 MERGE** | DV + Photon 조합으로 MERGE 연산 성능이 기존 대비 3~10배 향상됩니다 |

> 💡 **실무 팁**: DV가 활성화된 테이블에서 MERGE/UPDATE를 자주 수행하는 경우, Photon 지원 클러스터(i3.xlarge 등)를 사용하면 DV 처리 성능이 크게 향상됩니다. Serverless 컴퓨트는 Photon이 기본 활성화되어 있습니다.

---

## DV와 타임 트래블 상호작용

Delta Lake의 타임 트래블(Time Travel)은 이전 버전의 데이터를 조회하는 기능인데, DV 환경에서 몇 가지 주의할 점이 있습니다.

### DV 환경에서의 타임 트래블 동작

```sql
-- version 10에서 DELETE 실행 (DV 생성)
DELETE FROM orders WHERE customer_id = 42;

-- version 9 조회 (DELETE 이전)
SELECT * FROM orders VERSION AS OF 9;
-- → customer_id = 42 행이 포함됨 (DV 적용 안 됨)

-- version 10 조회 (DELETE 이후)
SELECT * FROM orders VERSION AS OF 10;
-- → customer_id = 42 행이 DV에 의해 필터링됨
```

### VACUUM과 DV의 상호작용

| 시나리오 | 동작 |
|----------|------|
| **VACUUM 전** | DV 파일 + 원본 Parquet 파일 모두 유지 → 타임 트래블 가능 |
| **VACUUM 후** | 보존 기간이 지난 DV 파일 삭제 → 해당 버전 타임 트래블 불가 |
| **OPTIMIZE 후 VACUUM** | DV가 물리적으로 반영된 새 Parquet 생성, 기존 파일+DV 삭제 |

> ⚠️ **Gotcha**: OPTIMIZE를 실행하면 DV가 적용된 새 파일이 생성되고, 기존 파일은 "removed" 표시됩니다. 이후 VACUUM이 기존 파일을 삭제하면, OPTIMIZE 이전 버전으로의 타임 트래블이 불가능해집니다. 타임 트래블 보존 기간(`delta.logRetentionDuration`)을 신중히 설정하세요.

---

## 대규모 DELETE/UPDATE 시 DV 누적 문제와 해결

### DV 누적 시나리오

지속적인 CDC(Change Data Capture) 처리나 빈번한 UPDATE가 발생하는 테이블에서는 DV가 빠르게 누적됩니다.

```
시나리오: 일 100만 행 UPDATE (전체 10억 행 테이블)

Day 1: 1,000개 파일 중 500개에 DV 생성 (파일당 평균 2,000행 삭제 표시)
Day 7: 대부분 파일에 DV 존재, 파일당 평균 14,000행 삭제 표시
Day 30: DV 크기 급증, 읽기 오버헤드 5~10% 증가

→ OPTIMIZE 없이 방치하면 읽기 성능이 점진적으로 저하됩니다
```

### DV 누적 정도 모니터링

```sql
-- 테이블의 DV 상태 확인
DESCRIBE DETAIL catalog.schema.orders;
-- numFiles, sizeInBytes 외에 DV 관련 메트릭 확인

-- 파일별 DV 현황 (Delta Log 직접 분석)
SELECT
    path,
    size,
    deletionVector.cardinality AS deleted_rows,
    deletionVector.sizeInBytes AS dv_size_bytes
FROM (
    SELECT explode(add) AS (path, size, deletionVector, ...)
    FROM delta.`s3://bucket/path/_delta_log/`
)
WHERE deletionVector IS NOT NULL
ORDER BY deleted_rows DESC;
```

### OPTIMIZE 실행 전략

| 전략 | 설명 | 적합한 환경 |
|------|------|------------|
| **Predictive Optimization** | Databricks가 자동 판단 | 대부분의 환경 (권장) |
| **정기 스케줄** | `OPTIMIZE` 매일/매주 실행 | Predictive Optimization 미사용 환경 |
| **임계치 기반** | DV 비율이 N% 초과 시 실행 | 세밀한 비용 관리가 필요한 경우 |
| **쓰기 후 즉시** | 대규모 MERGE 직후 실행 | 읽기 지연이 매우 민감한 환경 |

```sql
-- 대규모 MERGE 후 즉시 OPTIMIZE (읽기 성능 복원)
MERGE INTO catalog.schema.orders AS target
USING staging.schema.updates AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- MERGE 직후 OPTIMIZE (DV를 물리적으로 반영)
OPTIMIZE catalog.schema.orders;
```

> 💡 **비용 고려**: OPTIMIZE는 파일을 다시 쓰는 작업이므로 컴퓨트 비용이 발생합니다. DV가 적은 상태에서 불필요하게 OPTIMIZE를 실행하면 낭비입니다. Predictive Optimization이 바로 이 "최적의 시점"을 자동으로 판단해 주는 기능입니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Deletion Vectors** | 파일을 다시 쓰지 않고 삭제된 행을 별도로 기록하는 최적화 기술입니다 |
| **RoaringBitmap** | DV의 내부 저장 구조로, 삭제된 행 번호를 압축 비트맵으로 효율 저장합니다 |
| **Copy-on-Write 대체** | 기존 전체 파일 재작성 방식 대비 DELETE/UPDATE/MERGE 성능이 크게 향상됩니다 |
| **Merge-on-Read** | DV는 읽기 시 삭제 행을 필터링하는 방식으로, 소량 변경에 유리합니다 |
| **Photon 연동** | DV + Photon 조합으로 MERGE 성능이 3~10배 향상됩니다 |
| **자동 활성화** | DBR 14.0+ 신규 테이블에서 기본 활성화됩니다 |
| **OPTIMIZE로 정리** | 누적된 DV는 OPTIMIZE 실행 시 물리적으로 반영됩니다 |
| **DV 누적 주의** | 빈번한 UPDATE/DELETE 시 DV가 누적되어 읽기 성능이 저하될 수 있습니다 |

---

## 참고 링크

- [Databricks: Deletion Vectors](https://docs.databricks.com/aws/en/delta/deletion-vectors.html)
- [Azure Databricks: What are deletion vectors?](https://learn.microsoft.com/en-us/azure/databricks/delta/deletion-vectors)
- [Databricks Blog: Deletion Vectors](https://www.databricks.com/blog/2023/07/05/deletion-vectors.html)
- [Delta Lake: Deletion Vectors Protocol](https://github.com/delta-io/delta/blob/master/PROTOCOL.md#deletion-vectors)
