# Liquid Clustering 상세 가이드

## 왜 Liquid Clustering이 필요한가요?

데이터가 쌓이면 쿼리 성능은 자연스럽게 느려집니다. 수억 행의 테이블에서 특정 조건의 데이터를 찾으려면, **관련 없는 파일까지 모두 읽어야**하기 때문입니다. 이 문제를 해결하기 위해 기존에는 **파티셔닝(Partitioning)**과 **Z-Order** 를 사용했지만, 각각 뚜렷한 한계가 있었습니다.

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
| **Small File Problem**| 파티션 키의 고유값이 많으면(예: 날짜 3년치 = 1,095개 파티션) 파일이 매우 작아집니다 |
| **파티션 키 변경 불가**| 한번 정한 파티션 키를 바꾸려면 테이블을 **처음부터 다시 써야** 합니다 |
| **파티션 키 선택의 어려움**| 다양한 쿼리 패턴에 맞는 최적의 파티션 키를 선택하기 어렵습니다 |
| **High Cardinality 부적합**| 고유값이 너무 많은 컬럼(예: user_id)으로는 파티셔닝할 수 없습니다 |

### Z-Order의 한계

```sql
-- 기존 Z-Order 방식
OPTIMIZE catalog.schema.orders
ZORDER BY (customer_id, order_date);
```

| 문제 | 설명 |
|------|------|
| **증분 처리 불가**| OPTIMIZE 실행 시 매번 **전체 데이터를 재정렬** 합니다 |
| **매번 컬럼 지정**| OPTIMIZE를 실행할 때마다 ZORDER BY 컬럼을 명시해야 합니다 |
| **테이블 속성이 아님**| Z-Order 설정이 테이블 메타데이터에 저장되지 않아 관리가 어렵습니다 |

---

## Liquid Clustering이란?

> 💡 **Liquid Clustering**은 Databricks가 도입한 **차세대 데이터 레이아웃 최적화 기술**입니다. 기존의 Hive 파티셔닝과 Z-Order를 대체하며, 데이터를 **증분 방식으로 자동 재배치** 하여 쿼리 성능을 최적화합니다.

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

> 💡 Liquid Clustering은 **증분 방식** 으로 동작합니다. OPTIMIZE를 실행하면 아직 클러스터링되지 않은 새 데이터만 처리하므로, 매번 전체 데이터를 재정렬하는 Z-Order보다 훨씬 효율적입니다.

---

## 클러스터링 키 선택 기준

적절한 클러스터링 키를 선택하는 것이 성능 최적화의 핵심입니다.

| 기준 | 설명 |
|------|------|
| **WHERE 절에 자주 사용되는 컬럼**| 필터 조건에 자주 등장하는 컬럼을 선택합니다 |
| **JOIN 조건에 사용되는 컬럼**| 테이블 간 조인에 자주 사용되는 키를 포함합니다 |
| **카디널리티(Cardinality)**| 고유값이 적당히 많은 컬럼이 효과적입니다 (너무 적거나 너무 많으면 효과 감소) |
| **최대 4개 컬럼**| 일반적으로 1~4개의 컬럼을 권장합니다 |

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
| **설정 방법**| `PARTITIONED BY` | `OPTIMIZE ... ZORDER BY` | `CLUSTER BY` |
| **키 변경**| ❌ 테이블 재생성 필요 | 매번 지정 | ✅ `ALTER TABLE ... CLUSTER BY` |
| **동작 방식**| 물리적 디렉토리 분리 | 전체 데이터 재정렬 | 증분 자동 재배치 |
| **High Cardinality**| ❌ (Small File 발생) | ⚠️ (제한적) | ✅ 지원 |
| **데이터 스킵**| 디렉토리 기반 | 파일 내 통계 기반 | 파일 내 통계 기반 |
| **OPTIMIZE 필요**| 아니오 | 예 (매번 명시) | 예 (자동 적용) |
| **Predictive Optimization**| ❌ | ❌ | ✅ 호환 |
| **추천**| 레거시 | 레거시 | ✅ **신규 테이블 권장**|

---

## 성능 이점

Liquid Clustering을 적절히 설정하면 다음과 같은 성능 향상을 기대할 수 있습니다.

| 개선 영역 | 설명 |
|-----------|------|
| **Data Skipping**| 관련 없는 파일을 건너뛰어 읽기 데이터 양을 대폭 줄입니다 |
| **OPTIMIZE 비용 감소**| 증분 처리로 매번 전체 데이터를 재정렬할 필요가 없습니다 |
| **쓰기 성능 유지**| 파티셔닝과 달리 Small File 문제가 발생하지 않습니다 |
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
| **신규 테이블**| Hive 파티셔닝 대신 Liquid Clustering을 기본으로 사용 |
| **기존 테이블**| `ALTER TABLE ... CLUSTER BY`로 전환 가능 (기존 파티션 제거 불필요) |
| **Predictive Optimization**| 함께 활성화하면 OPTIMIZE를 자동으로 실행하여 운영 부담 감소 |
| **키 선택**| 쿼리 로그를 분석하여 가장 자주 필터링되는 컬럼을 선택 |

---

## 현장에서 배운 것들: 파티셔닝과 클러스터링의 현실

### 파티셔닝을 잘못 설계해서 소형 파일 100만 개가 생긴 프로젝트

이건 제가 실제로 겪은 사례입니다. 한 이커머스 고객이 주문 테이블을 `PARTITIONED BY (order_date, store_id, category)` 세 컬럼으로 파티셔닝했습니다. 얼핏 보면 합리적으로 보이지만, 결과는 참담했습니다.

```
계산해 봅시다:
- order_date: 365일/년 × 3년 = 1,095 파티션
- store_id: 200개 매장
- category: 50개 카테고리

총 파티션 수: 1,095 × 200 × 50 = 10,950,000 (약 1,100만 개!)
```

실제로는 모든 조합이 존재하지 않아서 약 **120만 개 파티션**이 생성되었지만, 각 파티션의 평균 파일 크기는 **800KB** 였습니다. Spark가 파일 목록을 읽는 것만으로 15분이 걸렸고, 실제 쿼리는 그 뒤에 시작되었습니다.

| 지표 | 파티셔닝 적용 전 | 과도한 파티셔닝 후 | Liquid Clustering 전환 후 |
|------|---------------|------------------|------------------------|
| **파일 수**| 5,000개 | 1,200,000개 | 3,500개 |
| **평균 파일 크기**| 500MB | 800KB | 800MB |
| **파일 목록 조회**| 2초 | 15분 | 1초 |
| **일반 쿼리 성능**| 30초 | 5분+ | 3초 |
| **OPTIMIZE 소요 시간**| 5분 | 2시간 (OOM 발생) | 3분 (증분) |

> ⚠️ **많은 팀이 이 실수를 합니다**: "파티셔닝은 많이 할수록 좋다"는 잘못된 믿음. 파티션 키의 카디널리티 곱이 10,000을 넘으면 거의 확실히 Small File Problem이 발생합니다. **파티션 수가 적을수록 좋고, 아예 안 하는 것이 나을 때가 많습니다.**

### 클러스터링 키 선택: 실전에서 배운 원칙들

교과서에는 "자주 필터링하는 컬럼을 선택하세요"라고만 나옵니다. 하지만 현업에서는 그것만으로 부족합니다.

#### 원칙 1: 단일 키 vs 복합 키 — 언제 무엇을?

```sql
-- 상황 A: 대부분의 쿼리가 날짜 범위 필터만 사용
-- → 단일 키가 최적
CLUSTER BY (event_date)

-- 상황 B: 날짜 + 고객 ID로 필터링하는 쿼리가 70% 이상
-- → 복합 키, 카디널리티 낮은 것을 앞에 배치
CLUSTER BY (event_date, customer_id)

-- 상황 C: 다양한 컬럼으로 필터링하는 ad-hoc 쿼리가 많음
-- → 가장 빈번한 2~3개만 선택, 나머지는 포기
CLUSTER BY (region, event_date)
```

> 💡 **현업 팁**: 4개 이상의 클러스터링 키는 효과가 급감합니다. 키가 많아질수록 각 키의 데이터 스킵 효과가 희석되기 때문입니다. 저는 실전에서 2개를 기본으로, 매우 특별한 경우에만 3개를 사용합니다.

#### 원칙 2: 카디널리티 스위트 스팟 찾기

| 카디널리티 | 예시 | Liquid Clustering 효과 |
|-----------|------|----------------------|
| **매우 낮음**(2~10) | `gender`, `is_active` | 효과 미미 — 데이터 스킵할 파일이 거의 없음 |
| **낮음**(10~1,000) | `region`, `category` | 좋음 — 적절한 파일 분포 |
| **중간**(1,000~100만) | `customer_id`, `zip_code` | 최적 — Liquid Clustering이 가장 빛나는 영역 |
| **매우 높음**(100만+) | `uuid`, `transaction_id` | 제한적 — 범위 필터가 아닌 점 조회에는 효과 적음 |

#### 원칙 3: 쿼리 로그 분석으로 키 결정하기

감으로 키를 정하지 마세요. **시스템 테이블의 쿼리 히스토리** 를 분석하세요.

```sql
-- 가장 자주 필터링에 사용되는 컬럼 파악
-- (Unity Catalog 시스템 테이블 활용)
SELECT
    filter_column,
    COUNT(*) AS query_count,
    AVG(duration_ms) AS avg_duration_ms
FROM (
    -- 쿼리 히스토리에서 WHERE 절 파싱
    -- 실제로는 query_text를 분석
    SELECT * FROM system.query.history
    WHERE statement_type = 'SELECT'
      AND start_time > current_date() - INTERVAL 30 DAYS
)
GROUP BY filter_column
ORDER BY query_count DESC
LIMIT 10;
```

### Z-Order에서 Liquid Clustering으로 마이그레이션: 실전 가이드

한 금융 고객이 2TB 거래 테이블을 Z-Order에서 Liquid Clustering으로 전환한 사례를 공유합니다.

#### 전환 전 상황

```sql
-- 매일 밤 3시에 수동으로 OPTIMIZE + Z-Order 실행
-- 이 작업만 45분 소요, 전체 데이터를 재정렬하므로 매일 비용 발생
OPTIMIZE prod.finance.transactions
ZORDER BY (account_id, transaction_date);
```

문제점:
- 매일 45분 × 365일 = 연간 **274시간** 의 컴퓨트 비용
- 누군가 ZORDER BY 컬럼을 잘못 지정하면 성능이 즉시 저하
- Z-Order 설정이 테이블 메타데이터에 없어서 "현재 어떤 컬럼으로 정렬되어 있는지" 확인 불가

#### 전환 과정

```sql
-- Step 1: Liquid Clustering 활성화 (기존 데이터는 그대로 유지)
ALTER TABLE prod.finance.transactions
CLUSTER BY (account_id, transaction_date);

-- Step 2: 첫 OPTIMIZE로 기존 데이터 클러스터링 (1회성, 시간 소요)
OPTIMIZE prod.finance.transactions;
-- → 첫 실행은 전체 데이터를 클러스터링하므로 2시간 소요

-- Step 3: 이후부터는 증분 OPTIMIZE (새 데이터만 처리)
-- Predictive Optimization 활성화하면 자동 실행
ALTER TABLE prod.finance.transactions
SET TBLPROPERTIES ('delta.enableDeletionVectors' = true);
```

#### 전환 결과

| 지표 | Z-Order | Liquid Clustering | 개선 |
|------|---------|-------------------|------|
| **일일 OPTIMIZE 시간**| 45분 | 3분 (증분) | **93% 감소**|
| **월간 OPTIMIZE 비용**| ~$2,400 | ~$160 | **93% 절감**|
| **주요 쿼리 성능**| 12초 | 4초 | **67% 개선**|
| **키 변경 시 다운타임**| 전체 재정렬 (2시간+) | ALTER TABLE 즉시 적용 | **무중단**|
| **운영 부담**| 수동 스케줄 관리 | Predictive Optimization 자동 | **자동화**|

> 💡 **마이그레이션 주의사항**: Liquid Clustering을 활성화한 직후에는 기존 데이터가 아직 클러스터링되지 않은 상태입니다. 첫 OPTIMIZE는 전체 데이터를 처리하므로 시간이 오래 걸립니다. **트래픽이 적은 시간(주말 새벽)에 첫 OPTIMIZE를 실행** 하는 것을 권장합니다.

### Liquid Clustering과 Predictive Optimization의 시너지

Liquid Clustering의 최대 운영 이점은 **Predictive Optimization과 결합했을 때** 나타납니다.

```sql
-- Predictive Optimization 활성화 (카탈로그 레벨)
ALTER CATALOG prod SET DBPROPERTIES ('predictive_optimization' = 'ENABLE');

-- 이후 Liquid Clustering 테이블에 대해:
-- 1. 새 데이터가 충분히 쌓이면 자동으로 OPTIMIZE 실행
-- 2. 쿼리 패턴을 분석하여 최적의 시점에 클러스터링
-- 3. VACUUM도 자동 실행
-- → 데이터 엔지니어가 유지보수에 쓰는 시간: 0
```

| 수동 운영 | Predictive Optimization | 차이 |
|----------|----------------------|------|
| 매일 OPTIMIZE 스케줄 관리 | 자동 판단 + 실행 | 운영 공수 제로 |
| 어떤 테이블을 먼저 할지 우선순위 고민 | 비용 대비 효과가 큰 테이블부터 자동 처리 | 최적의 순서 |
| 새벽 배치 윈도우에 맞춰 실행 | 쿼리 부하가 낮은 시점에 자동 실행 | 워크로드 영향 최소화 |

### 실전에서 자주 받는 질문들

| 질문 | 답변 |
|------|------|
| "기존 파티셔닝된 테이블에 Liquid Clustering을 적용하면?" | 가능합니다. `ALTER TABLE ... CLUSTER BY`를 실행하면 됩니다. 기존 파티션 구조는 유지되지만, 새로운 데이터부터 클러스터링이 적용됩니다. 전체 데이터를 재클러스터링하려면 OPTIMIZE를 실행하세요. |
| "Liquid Clustering이 적용된 테이블의 파일 크기는?" | 기본 목표 파일 크기는 1GB입니다. 클러스터링 과정에서 파일이 합쳐지므로, Small File Problem이 자연스럽게 해결됩니다. |
| "Delta Sharing으로 공유된 테이블에도 적용 가능한가요?" | 네, 공유하는 쪽(Provider)에서 클러스터링을 적용하면, 수신하는 쪽(Recipient)도 자동으로 Data Skipping 혜택을 받습니다. |
| "Liquid Clustering과 파티셔닝을 동시에 사용할 수 있나요?" | 아닙니다. Liquid Clustering은 파티셔닝을 대체합니다. CLUSTER BY를 지정하면 PARTITIONED BY는 사용할 수 없습니다. |
| "10GB 이하의 작은 테이블에도 의미가 있나요?" | 작은 테이블에서는 효과가 미미합니다. 파일 수가 적어서 스킵할 파일 자체가 별로 없기 때문입니다. **수십 GB 이상** 부터 체감 효과가 나타납니다. |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Liquid Clustering**| 기존 파티셔닝/Z-Order를 대체하는 차세대 데이터 레이아웃 최적화 기술입니다 |
| **CLUSTER BY**| 테이블 생성 또는 ALTER TABLE로 클러스터링 키를 지정합니다 |
| **증분 처리**| OPTIMIZE 시 새 데이터만 처리하여 비용을 절감합니다 |
| **키 변경 자유** | 쿼리 패턴이 바뀌면 언제든지 클러스터링 키를 변경할 수 있습니다 |

---

## 참고 링크

- [Databricks: Use Liquid Clustering for Delta tables](https://docs.databricks.com/aws/en/delta/clustering.html)
- [Azure Databricks: Liquid Clustering](https://learn.microsoft.com/en-us/azure/databricks/delta/clustering)
- [Databricks Blog: Liquid Clustering](https://www.databricks.com/blog/2023/06/12/introducing-liquid-clustering-delta-lake.html)
- [Databricks: Predictive Optimization](https://docs.databricks.com/aws/en/delta/predictive-optimization.html)
