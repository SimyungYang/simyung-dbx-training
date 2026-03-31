# Liquid Clustering 상세 가이드

## 왜 Liquid Clustering이 필요한가요?

데이터가 쌓이면 쿼리 성능은 자연스럽게 느려집니다. 수억 행의 테이블에서 특정 조건의 데이터를 찾으려면, **관련 없는 파일까지 모두 읽어야** 하기 때문입니다. 이 문제를 해결하기 위해 기존에는 **파티셔닝(Partitioning)**과 **Z-Order**를 사용했지만, 각각 뚜렷한 한계가 있었습니다.

---

## 기존 방식의 한계

### Hive 스타일 파티셔닝의 문제

```sql
-- 기존 파티셔닝 방식
CREATE TABLE catalog.schema.orders (
    order_id BIGINT,
    customer_id BIGINT,
    amount DECIMAL(10, 2),
    order_date DATE
) PARTITIONED BY (order_date);
```

| 문제 | 설명 |
|------|------|
| **Small File Problem** | 파티션 키의 고유값이 많으면(예: 날짜 3년치 = 1,095개 파티션) 파일이 매우 작아집니다 |
| **파티션 키 변경 불가** | 한번 정한 파티션 키를 바꾸려면 테이블을 **처음부터 다시 써야** 합니다 |
| **파티션 키 선택의 어려움** | 다양한 쿼리 패턴에 맞는 최적의 파티션 키를 선택하기 어렵습니다 |
| **High Cardinality 부적합** | 고유값이 너무 많은 컬럼(예: user_id)으로는 파티셔닝할 수 없습니다 |

### Z-Order의 한계

```sql
-- 기존 Z-Order 방식
OPTIMIZE catalog.schema.orders
ZORDER BY (customer_id, order_date);
```

| 문제 | 설명 |
|------|------|
| **증분 처리 불가** | OPTIMIZE 실행 시 매번 **전체 데이터를 재정렬**합니다 |
| **매번 컬럼 지정** | OPTIMIZE를 실행할 때마다 ZORDER BY 컬럼을 명시해야 합니다 |
| **테이블 속성이 아님** | Z-Order 설정이 테이블 메타데이터에 저장되지 않아 관리가 어렵습니다 |

---

## Liquid Clustering이란?

> 💡 **Liquid Clustering**은 Databricks가 도입한 **차세대 데이터 레이아웃 최적화 기술**입니다. 기존의 Hive 파티셔닝과 Z-Order를 대체하며, 데이터를 **증분 방식으로 자동 재배치**하여 쿼리 성능을 최적화합니다.

![Liquid Clustering 개념](https://docs.databricks.com/aws/en/_images/liquid-clustering.png)

> 이미지 출처: [Databricks: Use Liquid Clustering for Delta tables](https://docs.databricks.com/aws/en/delta/clustering.html)

핵심 차이는 "유동적(Liquid)"이라는 이름에 담겨 있습니다. 기존 파티셔닝이 데이터를 **고정된 칸**에 넣는 방식이라면, Liquid Clustering은 데이터가 **자유롭게 흘러** 최적의 위치에 배치되도록 합니다.

---

## 사용 방법

### 테이블 생성 시 활성화

```sql
-- CLUSTER BY로 클러스터링 키를 지정합니다
CREATE TABLE catalog.schema.orders (
    order_id BIGINT,
    customer_id BIGINT,
    product STRING,
    amount DECIMAL(10, 2),
    order_date DATE,
    region STRING
) CLUSTER BY (region, order_date);
```

### 기존 테이블에 적용

```sql
-- 기존 테이블에 Liquid Clustering 활성화
ALTER TABLE catalog.schema.orders
CLUSTER BY (region, order_date);
```

### 클러스터링 키 변경

```sql
-- 쿼리 패턴이 바뀌면 키를 자유롭게 변경할 수 있습니다
ALTER TABLE catalog.schema.orders
CLUSTER BY (customer_id, order_date);
```

### 클러스터링 제거

```sql
-- Liquid Clustering 비활성화
ALTER TABLE catalog.schema.orders
CLUSTER BY NONE;
```

### 클러스터링 트리거

```sql
-- OPTIMIZE 실행 시 자동으로 Liquid Clustering이 적용됩니다
OPTIMIZE catalog.schema.orders;
```

> 💡 Liquid Clustering은 **증분 방식**으로 동작합니다. OPTIMIZE를 실행하면 아직 클러스터링되지 않은 새 데이터만 처리하므로, 매번 전체 데이터를 재정렬하는 Z-Order보다 훨씬 효율적입니다.

---

## 클러스터링 키 선택 기준

적절한 클러스터링 키를 선택하는 것이 성능 최적화의 핵심입니다.

| 기준 | 설명 |
|------|------|
| **WHERE 절에 자주 사용되는 컬럼** | 필터 조건에 자주 등장하는 컬럼을 선택합니다 |
| **JOIN 조건에 사용되는 컬럼** | 테이블 간 조인에 자주 사용되는 키를 포함합니다 |
| **카디널리티(Cardinality)** | 고유값이 적당히 많은 컬럼이 효과적입니다 (너무 적거나 너무 많으면 효과 감소) |
| **최대 4개 컬럼** | 일반적으로 1~4개의 컬럼을 권장합니다 |

### 좋은 예시와 나쁜 예시

```sql
-- ✅ 좋은 선택: 날짜와 지역으로 자주 필터링하는 경우
CLUSTER BY (order_date, region)

-- ✅ 좋은 선택: 고객 ID와 날짜로 자주 조회하는 경우
CLUSTER BY (customer_id, event_date)

-- ❌ 나쁜 선택: Boolean 컬럼 (카디널리티가 2밖에 안 됨)
CLUSTER BY (is_active)

-- ❌ 나쁜 선택: 고유 ID만 사용 (범위 필터에 부적합)
CLUSTER BY (uuid)
```

---

## Hive 파티셔닝 vs Z-Order vs Liquid Clustering 비교

| 비교 항목 | Hive 파티셔닝 | Z-Order | Liquid Clustering |
|-----------|--------------|---------|-------------------|
| **설정 방법** | `PARTITIONED BY` | `OPTIMIZE ... ZORDER BY` | `CLUSTER BY` |
| **키 변경** | ❌ 테이블 재생성 필요 | 매번 지정 | ✅ `ALTER TABLE ... CLUSTER BY` |
| **동작 방식** | 물리적 디렉토리 분리 | 전체 데이터 재정렬 | 증분 자동 재배치 |
| **High Cardinality** | ❌ (Small File 발생) | ⚠️ (제한적) | ✅ 지원 |
| **데이터 스킵** | 디렉토리 기반 | 파일 내 통계 기반 | 파일 내 통계 기반 |
| **OPTIMIZE 필요** | 아니오 | 예 (매번 명시) | 예 (자동 적용) |
| **Predictive Optimization** | ❌ | ❌ | ✅ 호환 |
| **추천** | 레거시 | 레거시 | ✅ **신규 테이블 권장** |

---

## 성능 이점

Liquid Clustering을 적절히 설정하면 다음과 같은 성능 향상을 기대할 수 있습니다.

| 개선 영역 | 설명 |
|-----------|------|
| **Data Skipping** | 관련 없는 파일을 건너뛰어 읽기 데이터 양을 대폭 줄입니다 |
| **OPTIMIZE 비용 감소** | 증분 처리로 매번 전체 데이터를 재정렬할 필요가 없습니다 |
| **쓰기 성능 유지** | 파티셔닝과 달리 Small File 문제가 발생하지 않습니다 |
| **운영 편의성** | 키 변경이 자유로워 쿼리 패턴 변화에 유연하게 대응합니다 |

### Data Skipping 동작 예시

```sql
-- region과 order_date로 클러스터링된 테이블
SELECT * FROM catalog.schema.orders
WHERE region = 'APAC' AND order_date BETWEEN '2025-03-01' AND '2025-03-31';

-- Liquid Clustering이 적용되어 있으면:
-- 전체 1,000개 파일 중 관련된 50개 파일만 읽습니다
-- → 95%의 파일을 건너뛰어 쿼리 시간이 대폭 단축됩니다
```

---

## 모범 사례

| 항목 | 권장사항 |
|------|---------|
| **신규 테이블** | Hive 파티셔닝 대신 Liquid Clustering을 기본으로 사용 |
| **기존 테이블** | `ALTER TABLE ... CLUSTER BY`로 전환 가능 (기존 파티션 제거 불필요) |
| **Predictive Optimization** | 함께 활성화하면 OPTIMIZE를 자동으로 실행하여 운영 부담 감소 |
| **키 선택** | 쿼리 로그를 분석하여 가장 자주 필터링되는 컬럼을 선택 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Liquid Clustering** | 기존 파티셔닝/Z-Order를 대체하는 차세대 데이터 레이아웃 최적화 기술입니다 |
| **CLUSTER BY** | 테이블 생성 또는 ALTER TABLE로 클러스터링 키를 지정합니다 |
| **증분 처리** | OPTIMIZE 시 새 데이터만 처리하여 비용을 절감합니다 |
| **키 변경 자유** | 쿼리 패턴이 바뀌면 언제든지 클러스터링 키를 변경할 수 있습니다 |

---

## 참고 링크

- [Databricks: Use Liquid Clustering for Delta tables](https://docs.databricks.com/aws/en/delta/clustering.html)
- [Azure Databricks: Liquid Clustering](https://learn.microsoft.com/en-us/azure/databricks/delta/clustering)
- [Databricks Blog: Liquid Clustering](https://www.databricks.com/blog/2023/06/12/introducing-liquid-clustering-delta-lake.html)
- [Databricks: Predictive Optimization](https://docs.databricks.com/aws/en/delta/predictive-optimization.html)
