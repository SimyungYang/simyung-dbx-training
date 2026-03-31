# Delta Lake 핵심

## Delta Lake란?

> 💡 **Delta Lake**는 클라우드 오브젝트 스토리지(S3, ADLS, GCS) 위에서 **ACID 트랜잭션**, **스키마 관리**, **타임 트래블** 등의 기능을 제공하는 오픈소스 스토리지 레이어입니다.

쉽게 비유하면, 일반 데이터 레이크가 **정리되지 않은 창고**라면, Delta Lake는 그 창고에 **재고 관리 시스템**, **출입 통제**, **변경 기록 장부**를 설치한 것과 같습니다.

### Delta Lake의 구조

| 구성 요소 | 역할 |
|-----------|------|
| Delta Log (트랜잭션 로그) | 모든 변경 사항을 기록합니다 |
| Parquet 파일 | 실제 데이터를 저장합니다 |

Delta Lake = Parquet 파일 + 트랜잭션 로그 (`_delta_log/`)

Delta Lake 테이블은 실제로 두 가지로 구성됩니다.

| 구성 요소 | 역할 |
|-----------|------|
| **Parquet 파일** | 실제 데이터가 저장되는 파일입니다. 컬럼 기반의 오픈 포맷입니다 |
| **Delta Log (_delta_log/)** | 트랜잭션 로그가 저장되는 디렉토리입니다. 모든 변경 사항이 JSON 파일로 기록됩니다 |

---

## 핵심 기능 1: ACID 트랜잭션

### ACID란?

> 💡 **ACID**는 데이터베이스 트랜잭션(작업 단위)이 갖추어야 할 네 가지 특성의 약자입니다. 전통적인 RDBMS(MySQL, PostgreSQL 등)에서는 당연한 기능이지만, 데이터 레이크에서는 제공되지 않았던 기능입니다.

| 속성 | 영문 | 설명 | 비유 |
|------|------|------|------|
| **원자성** | Atomicity | 작업이 전부 완료되거나 전부 취소되어야 합니다 | 은행 이체: 인출+입금이 반드시 함께 성공하거나 함께 실패해야 합니다 |
| **일관성** | Consistency | 작업 전후로 데이터가 항상 유효한 상태를 유지해야 합니다 | 재고: 마이너스 재고가 생기면 안 됩니다 |
| **격리성** | Isolation | 동시에 실행되는 작업들이 서로 간섭하지 않아야 합니다 | ATM: 두 사람이 동시에 같은 계좌에서 출금해도 잔액이 꼬이면 안 됩니다 |
| **지속성** | Durability | 완료된 작업의 결과는 영구적으로 보존되어야 합니다 | 저장: 컴퓨터가 갑자기 꺼져도 저장된 데이터는 유지되어야 합니다 |

### 왜 데이터 레이크에서 ACID가 중요한가요?

ACID가 없는 일반 데이터 레이크에서는 다음과 같은 문제가 발생할 수 있습니다.

| 시나리오 | ACID 없는 데이터 레이크 | Delta Lake (ACID 보장) |
|---------|---------------------|---------------------|
| 쓰기 중 실패 | 부분적으로 쓰인 불완전한 데이터 → 데이터 오염 | 트랜잭션 롤백 → 이전 상태 유지 |
| 동시 읽기/쓰기 | 읽기 중 데이터 변경 → 불일치 결과 | 스냅샷 격리 → 일관된 결과 |

### 실습 예제

```sql
-- Delta 테이블 생성
CREATE TABLE catalog.schema.customers (
    customer_id BIGINT,
    name STRING,
    email STRING,
    city STRING,
    signup_date DATE
) USING DELTA;

-- 데이터 삽입
INSERT INTO catalog.schema.customers VALUES
(1, '김철수', 'cs.kim@email.com', '서울', '2025-01-15'),
(2, '이영희', 'yh.lee@email.com', '부산', '2025-02-20'),
(3, '박민수', 'ms.park@email.com', '대구', '2025-03-10');
```

위 INSERT 문이 실행 중에 네트워크 오류가 발생해도, Delta Lake는 **원자성(Atomicity)** 을 보장하여 불완전한 데이터가 테이블에 남지 않습니다.

---

## 핵심 기능 2: 타임 트래블 (Time Travel)

### 개념

> 💡 **타임 트래블(Time Travel)** 이란 Delta 테이블의 **과거 시점 데이터**를 조회하거나 복원할 수 있는 기능입니다. Delta Lake는 모든 변경 사항을 트랜잭션 로그에 기록하기 때문에, 이전 버전의 데이터로 되돌아갈 수 있습니다.

이 기능은 마치 **문서의 버전 관리(Version Control)** 와 같습니다. Google Docs에서 이전 편집 내역으로 되돌릴 수 있는 것처럼, Delta 테이블도 이전 버전으로 되돌릴 수 있습니다.

### 사용 방법

```sql
-- 현재 데이터 조회
SELECT * FROM catalog.schema.customers;

-- 데이터 수정
UPDATE catalog.schema.customers
SET city = '인천'
WHERE customer_id = 1;

-- 수정 전 데이터 조회 (버전 번호로)
SELECT * FROM catalog.schema.customers VERSION AS OF 0;

-- 수정 전 데이터 조회 (타임스탬프로)
SELECT * FROM catalog.schema.customers TIMESTAMP AS OF '2025-03-15 10:00:00';

-- 테이블 변경 이력 확인
DESCRIBE HISTORY catalog.schema.customers;
```

### 활용 사례

| 사례 | 설명 |
|------|------|
| **실수 복구** | 잘못된 UPDATE/DELETE를 실행한 후, 이전 버전으로 즉시 복원할 수 있습니다 |
| **감사(Audit)** | 특정 시점의 데이터 상태를 확인하여 규정 준수를 증명할 수 있습니다 |
| **데이터 디버깅** | "어제까지는 정상이었는데 오늘 이상하다" → 어제 버전과 비교 가능합니다 |
| **ML 재현성** | 특정 시점의 데이터로 모델을 재학습하여 결과를 재현할 수 있습니다 |

---

## 핵심 기능 3: 스키마 관리

### 스키마 강제 (Schema Enforcement)

> 💡 **스키마 강제(Schema Enforcement)** 란 테이블에 정의된 스키마에 맞지 않는 데이터를 삽입하려고 하면 **자동으로 거부**하는 기능입니다.

```sql
-- customers 테이블에 잘못된 데이터 삽입 시도
INSERT INTO catalog.schema.customers VALUES
(4, '최지은', 'je.choi@email.com', '서울', 'not-a-date');
-- ❌ 오류 발생: 'not-a-date'는 DATE 타입이 아닙니다
```

### 스키마 진화 (Schema Evolution)

> 💡 **스키마 진화(Schema Evolution)** 란 기존 테이블의 스키마(구조)를 데이터를 유실하지 않고 **안전하게 변경**할 수 있는 기능입니다. 새로운 컬럼 추가, 컬럼 이름 변경 등이 가능합니다.

```sql
-- 새 컬럼 추가
ALTER TABLE catalog.schema.customers
ADD COLUMN phone_number STRING;

-- 기존 데이터는 새 컬럼이 NULL로 채워짐
SELECT * FROM catalog.schema.customers;
-- +------------+------+------------------+----+------------+--------------+
-- |customer_id | name | email            |city|signup_date | phone_number |
-- +------------+------+------------------+----+------------+--------------+
-- |1           |김철수|cs.kim@email.com  |인천|2025-01-15  | NULL         |
-- |2           |이영희|yh.lee@email.com  |부산|2025-02-20  | NULL         |
-- ...
```

---

## 핵심 기능 4: 트랜잭션 로그 (Delta Log)

모든 Delta Lake의 기능은 **트랜잭션 로그(Delta Log)** 를 기반으로 동작합니다.

### 동작 원리

| 구성 요소 | 설명 |
|-----------|------|
| `_delta_log/` | 트랜잭션 로그 디렉토리 |
| `00000.json` (버전 0) | 최초 테이블 생성 |
| `00001.json` (버전 1) | INSERT 작업 |
| `00002.json` (버전 2) | UPDATE 작업 |
| `00010.checkpoint.parquet` | 10번째 체크포인트 (최적화) |
| `part-00000-*.parquet` | 실제 데이터 파일들 |

- 각 트랜잭션(INSERT, UPDATE, DELETE 등)이 실행될 때마다 새로운 로그 파일(JSON)이 생성됩니다
- 로그 파일에는 "어떤 Parquet 파일이 추가되었고, 어떤 파일이 제거되었는지"가 기록됩니다
- 테이블을 읽을 때는 최신 로그를 참조하여 유효한 Parquet 파일만 읽습니다
- 이 구조 덕분에 타임 트래블, 롤백, 동시 쓰기 제어가 가능합니다

> 💡 **MVCC(Multi-Version Concurrency Control, 다중 버전 동시성 제어)란?** Delta Lake가 동시 읽기/쓰기를 관리하는 방식입니다. 데이터를 수정할 때 기존 파일을 직접 덮어쓰지 않고, 새로운 파일을 추가한 후 로그를 업데이트하는 방식으로 동작합니다. 따라서 누군가 테이블을 읽고 있는 중에 다른 사람이 쓰기를 해도, 읽는 쪽은 일관된 데이터를 볼 수 있습니다.

---

## Delta Lake vs 일반 Parquet 비교

| 비교 항목 | 일반 Parquet 파일 | Delta Lake |
|-----------|-------------------|------------|
| **트랜잭션** | ❌ (부분 쓰기 위험) | ✅ ACID 보장 |
| **스키마 관리** | ❌ (수동 관리) | ✅ 강제 + 진화 |
| **타임 트래블** | ❌ | ✅ 버전별 조회 |
| **UPDATE/DELETE** | ❌ (파일 전체 재작성) | ✅ 효율적 처리 |
| **동시 쓰기** | ❌ (충돌 위험) | ✅ 자동 조율 |
| **데이터 압축** | 수동 | ✅ 자동 (OPTIMIZE) |
| **메타데이터** | 파일 자체에만 | ✅ 트랜잭션 로그로 관리 |

---

## 실습: Delta 테이블 생성과 기본 조작

```sql
-- 1. Delta 테이블 생성
CREATE TABLE catalog.schema.products (
    product_id BIGINT,
    name STRING,
    category STRING,
    price DECIMAL(10, 2),
    stock INT
) USING DELTA;

-- 2. 데이터 삽입
INSERT INTO catalog.schema.products VALUES
(101, '무선 키보드', '전자제품', 89000.00, 150),
(102, '블루투스 이어폰', '전자제품', 65000.00, 300),
(103, '텀블러 500ml', '생활용품', 25000.00, 500);

-- 3. 데이터 수정 (UPDATE)
UPDATE catalog.schema.products
SET price = 79000.00, stock = stock - 10
WHERE product_id = 101;

-- 4. 변경 이력 확인
DESCRIBE HISTORY catalog.schema.products;

-- 5. 수정 전 데이터 확인 (타임 트래블)
SELECT * FROM catalog.schema.products VERSION AS OF 1;

-- 6. 테이블 상세 정보 확인
DESCRIBE DETAIL catalog.schema.products;
```

---

## 심화: Delta Lake 내부 구조와 동시성 제어

이 섹션에서는 Principal SA 수준에서 알아야 할 Delta Lake의 내부 동작 원리, 동시성 제어 메커니즘, 대규모 테이블에서의 성능 이슈와 대응 방안을 다룹니다.

### 트랜잭션 로그 내부 구조 — 액션(Action) 상세

Delta Log의 각 JSON 커밋 파일은 하나 이상의 **액션(Action)** 으로 구성됩니다. 이 액션들이 Delta Lake의 모든 기능을 가능하게 하는 핵심 메커니즘입니다.

| 액션 유형 | 설명 | 예시 |
|-----------|------|------|
| **Add** | 새로운 Parquet 파일이 테이블에 추가되었음을 기록합니다 | INSERT, UPDATE의 신규 파일 |
| **Remove** | 기존 Parquet 파일이 논리적으로 제거되었음을 기록합니다 | DELETE, UPDATE의 기존 파일 제거 |
| **CommitInfo** | 커밋 메타데이터(작업 유형, 사용자, 타임스탬프, 노트북 경로 등)를 기록합니다 | `{"operation": "MERGE", "operationMetrics": {...}}` |
| **Metadata** | 테이블 스키마, 파티션 컬럼, 설정 변경을 기록합니다 | `ALTER TABLE SET TBLPROPERTIES` |
| **Protocol** | 리더/라이터 프로토콜 버전을 기록합니다 | Reader v1, Writer v7 (Liquid Clustering 등) |
| **SetTransaction** | 멱등 쓰기를 위한 트랜잭션 ID를 기록합니다 | Structured Streaming의 exactly-once 보장 |

```json
// _delta_log/00005.json 예시 (단순화)
{
  "add": {
    "path": "part-00003-abc123.parquet",
    "size": 52428800,
    "modificationTime": 1711900800000,
    "dataChange": true,
    "stats": "{\"numRecords\":125000,\"minValues\":{\"order_date\":\"2025-01-01\"},\"maxValues\":{\"order_date\":\"2025-03-31\"},\"nullCount\":{\"order_date\":0}}"
  }
}
{
  "remove": {
    "path": "part-00001-def456.parquet",
    "deletionTimestamp": 1711900800000,
    "dataChange": true
  }
}
{
  "commitInfo": {
    "operation": "MERGE",
    "operationParameters": {"predicate": "t.id = s.id"},
    "readVersion": 4,
    "operationMetrics": {"numTargetRowsUpdated": "3200", "numOutputRows": "128000"}
  }
}
```

#### 파일 통계(Stats)와 Data Skipping

Add 액션에 포함된 `stats` 필드는 각 Parquet 파일의 **min/max 값**, **null 개수**, **레코드 수**를 담고 있습니다. 이 통계를 활용하여 쿼리 시 불필요한 파일을 건너뛰는 것이 **Data Skipping**입니다.

```sql
-- 이 쿼리는 order_date 범위에 해당하는 파일만 읽습니다
SELECT * FROM catalog.schema.orders
WHERE order_date BETWEEN '2025-03-01' AND '2025-03-31';
-- Stats에서 max(order_date) < '2025-03-01'인 파일은 자동으로 건너뜀
```

> ⚠️ **Gotcha**: 기본적으로 통계는 **처음 32개 컬럼**에 대해서만 수집됩니다. 와이드 테이블(수백 개 컬럼)에서 33번째 이후 컬럼으로 필터링하면 Data Skipping이 동작하지 않습니다. `delta.dataSkippingNumIndexedCols` 속성으로 조정할 수 있지만, 너무 크게 설정하면 커밋 시 오버헤드가 증가합니다.

> ⚠️ **Gotcha**: Stats는 **STRING 컬럼의 처음 32자**만 인덱싱합니다. 긴 문자열(URL, JSON 등)에 대한 Data Skipping은 효과가 제한적입니다. 이런 경우 Z-ORDER나 Liquid Clustering을 활용하세요.

---

### 쓰기 충돌 해결 (Write Conflict Resolution)

Delta Lake는 **낙관적 동시성 제어(Optimistic Concurrency Control, OCC)** 를 사용합니다. 이는 "충돌이 드물 것"이라고 낙관적으로 가정하고, 충돌이 실제로 발생했을 때만 처리하는 방식입니다.

#### 동작 원리

```
Writer A: 버전 5 읽기 → 변환 처리 (수 분) → 버전 6으로 커밋 시도 → ✅ 성공
Writer B: 버전 5 읽기 → 변환 처리 (수 분) → 버전 6으로 커밋 시도 → ❌ 충돌!
         → 버전 6 로그를 읽고 충돌 여부 판단 → 재시도 or ConflictException
```

1. 트랜잭션 시작 시 현재 테이블 버전(예: v5)을 읽습니다
2. 데이터 변환 처리를 수행합니다
3. 커밋 시, `_delta_log/00000000000000000006.json` 파일을 **원자적으로(atomically)** 생성합니다
4. 다른 Writer가 이미 v6을 커밋했다면, 충돌이 발생합니다
5. 충돌 발생 시, 새로 커밋된 로그를 읽어 **논리적 충돌**이 있는지 확인합니다

#### ConflictException 발생 조건

모든 동시 쓰기가 충돌하는 것은 아닙니다. Delta Lake는 **실제로 같은 파일/파티션에 영향을 주는 경우**에만 충돌로 판단합니다.

| 작업 조합 | 충돌 여부 | 설명 |
|-----------|-----------|------|
| INSERT + INSERT (append-only) | ✅ 충돌 없음 | 서로 다른 파일을 추가하므로 안전합니다 |
| INSERT + DELETE (다른 파티션) | ✅ 충돌 없음 | 서로 다른 파티션에 영향을 주므로 안전합니다 |
| DELETE + DELETE (같은 파일) | ❌ **충돌** | 같은 파일을 제거하려고 합니다 |
| UPDATE + UPDATE (같은 파일) | ❌ **충돌** | 같은 파일을 수정하려고 합니다 |
| MERGE + MERGE (겹치는 조건) | ❌ **충돌 가능** | 조건에 따라 같은 파일에 영향을 줄 수 있습니다 |
| OPTIMIZE + 모든 쓰기 | ❌ **충돌** | OPTIMIZE는 파일을 재배열하므로 대부분 충돌합니다 |

#### 재시도 패턴

```python
from delta.exceptions import ConcurrentAppendException, ConcurrentDeleteReadException
import time

MAX_RETRIES = 3
for attempt in range(MAX_RETRIES):
    try:
        # MERGE 작업 실행
        deltaTable.alias("t").merge(
            source.alias("s"),
            "t.id = s.id"
        ).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()
        break  # 성공 시 루프 종료
    except ConcurrentAppendException:
        if attempt < MAX_RETRIES - 1:
            time.sleep(2 ** attempt)  # 지수 백오프
            continue
        raise  # 최대 재시도 초과 시 예외 전파
```

> ⚠️ **Gotcha — OPTIMIZE와 동시 쓰기**: OPTIMIZE(또는 Auto Compaction)가 실행 중일 때 다른 Writer가 커밋하면 `ConcurrentAppendException`이 발생할 수 있습니다. 프로덕션에서는 OPTIMIZE를 **쓰기 워크로드가 적은 시간대**에 스케줄링하거나, 파티션 단위로 나누어 실행하세요.

> ⚠️ **Gotcha — Structured Streaming + Batch 쓰기**: 같은 테이블에 Streaming과 Batch 작업이 동시에 실행되면 충돌이 빈번합니다. Streaming은 `foreachBatch` + `merge` 패턴을 사용하고, Batch 작업은 다른 파티션에 쓰도록 분리하는 것이 좋습니다.

---

### 격리 수준 (Isolation Levels)

Delta Lake는 두 가지 격리 수준을 제공합니다. 기본값은 `WriteSerializable`이며, 대부분의 워크로드에 적합합니다.

| 격리 수준 | 기본값 | 동시성 | 정합성 | 적합한 워크로드 |
|-----------|--------|--------|--------|---------------|
| **WriteSerializable** | ✅ | 높음 | 쓰기 순서만 보장 | 대부분의 ETL, Streaming, 분석 |
| **Serializable** | | 낮음 | 완전한 직렬화 보장 | 금융 트랜잭션, 재고 관리 등 강한 일관성 필요 |

#### WriteSerializable (기본값)

- 동시 쓰기 작업들이 **어떤 직렬 순서로 실행한 것과 동일한 결과**를 보장합니다
- 단, 읽기(read set)와 쓰기(write set) 사이의 직렬화는 보장하지 않습니다
- 즉, Writer A가 "조건 X에 해당하는 행이 없음"을 확인하고 INSERT를 수행하는 동안, Writer B가 조건 X에 해당하는 행을 INSERT할 수 있습니다 (Phantom Read 가능)

#### Serializable

- 모든 작업이 **완전히 직렬적으로** 실행된 것처럼 보장합니다
- Phantom Read를 방지합니다
- 동시성이 크게 감소하므로, **정말 필요한 경우에만** 사용하세요

```sql
-- 테이블의 격리 수준을 Serializable로 변경
ALTER TABLE catalog.schema.financial_ledger
SET TBLPROPERTIES ('delta.isolationLevel' = 'Serializable');
```

> ⚠️ **실무 가이드**: 99%의 데이터 엔지니어링 워크로드에서는 기본값인 `WriteSerializable`이 적합합니다. `Serializable`은 금융 원장, 재고 관리 등 **비즈니스 로직이 read-then-write 패턴에 의존하는 경우**에만 사용하세요. 불필요하게 Serializable로 설정하면 동시 처리량이 크게 떨어집니다.

---

### 트랜잭션 로그 스케일링 — 대규모 테이블에서의 성능

수십만 건 이상의 커밋이 쌓인 대규모 테이블에서는 트랜잭션 로그 자체가 성능 병목이 될 수 있습니다.

#### 체크포인트(Checkpoint)

Delta Lake는 기본적으로 **10번의 커밋마다** 체크포인트 파일(Parquet 형식)을 생성합니다. 체크포인트는 해당 시점까지의 모든 액션을 하나의 Parquet 파일로 압축한 것으로, 테이블의 현재 상태를 빠르게 복원할 수 있게 합니다.

```
_delta_log/
  00000000000000000000.json     ← 버전 0
  00000000000000000001.json     ← 버전 1
  ...
  00000000000000000010.checkpoint.parquet  ← 체크포인트 (v0~v10 요약)
  00000000000000000011.json     ← 버전 11 (체크포인트 이후 증분)
  ...
  00000000000000000020.checkpoint.parquet  ← 체크포인트 (v0~v20 요약)
```

- 테이블을 읽을 때: **가장 최근 체크포인트 + 이후 JSON 파일들**만 읽으면 됩니다
- 체크포인트 주기는 `delta.checkpointInterval` 속성으로 조정할 수 있습니다 (기본값: 10)

#### 대규모 테이블의 성능 이슈와 대응

| 증상 | 원인 | 대응 방법 |
|------|------|----------|
| 테이블 첫 읽기가 느림 (수 초~수십 초) | 체크포인트 파일이 너무 큼 (수십만 개 파일의 메타데이터) | 파일 수를 줄이세요 — OPTIMIZE를 정기적으로 실행합니다 |
| 커밋 시간이 점점 길어짐 | 로그 디렉토리에 JSON 파일이 수십만 개 | 로그 정리: `VACUUM` 실행 후 `delta.logRetentionDuration` 조정 (기본 30일) |
| `DESCRIBE HISTORY`가 느림 | 커밋 기록이 너무 많음 | `LIMIT` 절을 사용하세요. 전체 히스토리 조회는 피합니다 |
| 스키마 변경이 느림 | 체크포인트 재작성 시 모든 파일 메타데이터 포함 | Liquid Clustering 적용으로 파일 수 자체를 줄입니다 |

```sql
-- 로그 보관 기간 조정 (기본 30일 → 7일)
ALTER TABLE catalog.schema.huge_table
SET TBLPROPERTIES (
  'delta.logRetentionDuration' = 'interval 7 days',
  'delta.checkpointInterval' = '10'  -- 기본값 유지 권장
);

-- 오래된 파일 정리 (7일 이전)
VACUUM catalog.schema.huge_table RETAIN 168 HOURS;
```

> ⚠️ **Gotcha — VACUUM과 타임 트래블**: `VACUUM`은 오래된 Parquet 파일을 물리적으로 삭제합니다. VACUUM 실행 후에는 보관 기간 이전 버전으로의 타임 트래블이 **불가능**합니다. 규정 준수 요건(데이터 감사, 7년 보관 등)이 있다면 VACUUM 보관 기간을 신중하게 설정하세요.

> ⚠️ **Gotcha — 체크포인트 주기 변경**: 체크포인트 주기를 너무 크게 늘리면(예: 100) 테이블 첫 읽기 시 100개의 JSON 파일을 파싱해야 하므로 성능이 저하됩니다. 반대로 너무 작게 줄이면(예: 1) 매 커밋마다 체크포인트를 생성하여 쓰기 오버헤드가 증가합니다. 기본값 10이 대부분의 경우 최적입니다.

#### 프로덕션 규모별 가이드

| 테이블 규모 | 파일 수 | 일 커밋 수 | 권장 설정 |
|------------|---------|-----------|----------|
| 소규모 (< 100GB) | ~1,000개 | < 100 | 기본 설정으로 충분합니다 |
| 중규모 (100GB ~ 1TB) | ~10,000개 | 100~1,000 | OPTIMIZE 주 1회, VACUUM 주 1회 |
| 대규모 (1TB ~ 10TB) | ~100,000개 | 1,000~10,000 | OPTIMIZE 일 1회 + Liquid Clustering, VACUUM 일 1회 |
| 초대규모 (> 10TB) | 100,000개+ | 10,000+ | Liquid Clustering 필수, Predictive Optimization 활성화 |

> 🆕 **Predictive Optimization (GA)**: Databricks가 테이블 사용 패턴을 분석하여 OPTIMIZE, VACUUM, ANALYZE를 자동으로 실행하는 기능입니다. 대규모 테이블 운영의 부담을 크게 줄여줍니다. Unity Catalog Managed Table에서 사용 가능합니다.

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **ACID 트랜잭션** | 데이터 변경이 항상 완전하고 일관되게 처리됩니다 |
| **타임 트래블** | 과거 시점의 데이터를 조회하거나 복원할 수 있습니다 |
| **스키마 강제** | 잘못된 형식의 데이터가 테이블에 들어오는 것을 방지합니다 |
| **스키마 진화** | 테이블 구조를 안전하게 변경할 수 있습니다 |
| **트랜잭션 로그** | 모든 변경 사항을 기록하여 위 기능들을 가능하게 하는 핵심 메커니즘입니다 |
| **낙관적 동시성 제어** | 충돌이 발생한 경우에만 처리하여 높은 동시성을 제공합니다 |
| **Data Skipping** | 파일 통계(min/max)를 활용하여 불필요한 파일 읽기를 건너뜁니다 |

다음 문서에서는 Delta Lake 테이블을 체계적으로 구성하는 설계 패턴인 **Medallion 아키텍처**를 살펴보겠습니다.

---

## 참고 링크

- [Databricks: Delta Lake](https://docs.databricks.com/aws/en/delta/)
- [Azure Databricks: What is Delta Lake?](https://learn.microsoft.com/en-us/azure/databricks/delta/)
- [Delta Lake Official Documentation](https://docs.delta.io/)
- [Databricks Blog: Delta Lake](https://www.databricks.com/blog/category/engineering-blog)
