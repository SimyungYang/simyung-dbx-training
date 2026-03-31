# Deletion Vectors 상세

## 왜 Deletion Vectors가 필요한가요?

Delta Lake에서 데이터를 삭제(DELETE)하거나 수정(UPDATE)할 때, 내부적으로는 어떤 일이 벌어질까요? 기존 방식에서는 변경 대상이 포함된 파일 전체를 **다시 작성(rewrite)**해야 했습니다. 1GB 파일에서 단 1행만 삭제해도, 나머지 999,999행을 포함한 새 파일을 써야 했던 것입니다.

> 💡 **Deletion Vectors(삭제 벡터)**는 이러한 비효율을 해결하기 위해 도입된 기술입니다. 파일을 다시 쓰지 않고, **어떤 행이 삭제되었는지를 별도의 파일에 기록**하는 방식으로 DELETE, UPDATE, MERGE의 성능을 크게 향상시킵니다.

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

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Deletion Vectors** | 파일을 다시 쓰지 않고 삭제된 행을 별도로 기록하는 최적화 기술입니다 |
| **Copy-on-Write 대체** | 기존 전체 파일 재작성 방식 대비 DELETE/UPDATE/MERGE 성능이 크게 향상됩니다 |
| **자동 활성화** | DBR 14.0+ 신규 테이블에서 기본 활성화됩니다 |
| **OPTIMIZE로 정리** | 누적된 DV는 OPTIMIZE 실행 시 물리적으로 반영됩니다 |

---

## 참고 링크

- [Databricks: Deletion Vectors](https://docs.databricks.com/aws/en/delta/deletion-vectors.html)
- [Azure Databricks: What are deletion vectors?](https://learn.microsoft.com/en-us/azure/databricks/delta/deletion-vectors)
- [Databricks Blog: Deletion Vectors](https://www.databricks.com/blog/2023/07/05/deletion-vectors.html)
- [Delta Lake: Deletion Vectors Protocol](https://github.com/delta-io/delta/blob/master/PROTOCOL.md#deletion-vectors)
