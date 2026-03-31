# Delta Lake 타임 트래블 (Time Travel)

## 왜 타임 트래블이 필요한가요?

데이터를 다루다 보면 실수는 피할 수 없습니다. 잘못된 UPDATE 하나로 수백만 행의 데이터가 변경될 수 있고, 의도치 않은 DELETE로 중요한 데이터가 사라질 수 있습니다.

> 💡 **타임 트래블(Time Travel)** 은 Delta Lake 테이블의 **과거 버전 데이터를 조회하거나 복원**할 수 있는 기능입니다. 모든 변경 사항이 트랜잭션 로그에 기록되어 있기 때문에, 마치 시간을 되돌리듯 이전 상태의 데이터에 접근할 수 있습니다.

Google Docs의 "버전 기록"이나 Git의 "커밋 이력"과 비슷한 개념입니다. 각 변경 작업(INSERT, UPDATE, DELETE, MERGE)이 하나의 "버전"으로 기록됩니다.

---

## 동작 원리

Delta Lake는 테이블에 변경이 발생할 때마다 `_delta_log/` 디렉토리에 새로운 JSON 로그 파일을 생성합니다. 각 로그 파일이 하나의 **버전(version)** 을 나타냅니다.

| 버전 | 로그 파일 | 작업 내용 |
|------|----------|----------|
| 0 | `00000000000000000000.json` | CREATE TABLE |
| 1 | `00000000000000000001.json` | INSERT 3건 |
| 2 | `00000000000000000002.json` | UPDATE 1건 |
| 3 | `00000000000000000003.json` | DELETE 2건 |

타임 트래블은 특정 버전의 로그를 기준으로, **해당 시점에 유효했던 Parquet 파일만 읽어오는 방식**으로 동작합니다. 과거 데이터를 별도로 저장하는 것이 아니라, 로그를 통해 "그 시점의 테이블 상태"를 재구성하는 것입니다.

---

## 변경 이력 확인: DESCRIBE HISTORY

타임 트래블을 사용하기 전에, 먼저 테이블의 변경 이력을 확인하는 것이 좋습니다.

```sql
-- 테이블의 전체 변경 이력 조회
DESCRIBE HISTORY catalog.schema.customers;
```

결과 예시:

| version | timestamp | operation | operationParameters |
|---------|-----------|-----------|-------------------|
| 3 | 2025-03-20 15:30:00 | DELETE | `{"predicate":"id < 100"}` |
| 2 | 2025-03-20 14:00:00 | UPDATE | `{"predicate":"city = '서울'"}` |
| 1 | 2025-03-20 10:00:00 | WRITE | `{"mode":"Append"}` |
| 0 | 2025-03-19 09:00:00 | CREATE TABLE | `{}` |

```sql
-- 최근 N개 이력만 조회
DESCRIBE HISTORY catalog.schema.customers LIMIT 10;
```

---

## 버전 번호로 조회: VERSION AS OF

```sql
-- 버전 1 시점의 데이터 조회
SELECT * FROM catalog.schema.customers VERSION AS OF 1;

-- 특정 버전의 행 수 확인
SELECT COUNT(*) FROM catalog.schema.customers VERSION AS OF 0;

-- 버전 간 데이터 비교
SELECT 'current' AS version, COUNT(*) AS cnt FROM catalog.schema.customers
UNION ALL
SELECT 'version_1', COUNT(*) FROM catalog.schema.customers VERSION AS OF 1;
```

---

## 타임스탬프로 조회: TIMESTAMP AS OF

정확한 버전 번호를 모를 때는 타임스탬프로도 조회할 수 있습니다.

```sql
-- 특정 시각 기준으로 조회
SELECT * FROM catalog.schema.customers
TIMESTAMP AS OF '2025-03-20 10:00:00';

-- 상대적 시간도 사용 가능
SELECT * FROM catalog.schema.customers
TIMESTAMP AS OF current_timestamp() - INTERVAL 2 HOURS;

-- 어제 시점의 데이터
SELECT * FROM catalog.schema.customers
TIMESTAMP AS OF current_date() - INTERVAL 1 DAY;
```

> 💡 `TIMESTAMP AS OF`는 지정한 시각 **이전의 가장 최신 버전**을 반환합니다. 예를 들어, 10:00에 INSERT가 있었고 14:00에 UPDATE가 있었다면, `TIMESTAMP AS OF '12:00'`은 10:00 시점(버전 1)의 데이터를 반환합니다.

---

## 테이블 복원: RESTORE TABLE

과거 버전의 데이터를 단순히 **조회**하는 것이 아니라, 테이블 자체를 **이전 상태로 되돌리고** 싶을 때 사용합니다.

```sql
-- 버전 번호로 복원
RESTORE TABLE catalog.schema.customers TO VERSION AS OF 1;

-- 타임스탬프로 복원
RESTORE TABLE catalog.schema.customers
TO TIMESTAMP AS OF '2025-03-20 10:00:00';
```

> ⚠️ **주의사항**: RESTORE는 새로운 버전을 생성하여 과거 상태를 복원합니다. 기존 이력이 삭제되는 것이 아니라, "버전 4: RESTORE to version 1"과 같은 새로운 이력이 추가됩니다. 따라서 RESTORE 이후에도 다시 되돌릴 수 있습니다.

### 실수 복구 시나리오

```sql
-- 실수로 모든 고객의 도시를 '서울'로 업데이트해버렸다고 가정
UPDATE catalog.schema.customers SET city = '서울';  -- 실수!

-- 1. 변경 이력 확인
DESCRIBE HISTORY catalog.schema.customers;
-- version 5: UPDATE (방금 실수한 작업)
-- version 4: ... (이전 정상 상태)

-- 2. 실수 직전 버전으로 복원
RESTORE TABLE catalog.schema.customers TO VERSION AS OF 4;

-- 3. 복원 결과 확인
SELECT * FROM catalog.schema.customers;
-- 정상 데이터가 복원되었습니다!
```

---

## 타임 트래블 보존 기간

타임 트래블은 **트랜잭션 로그**와 **데이터 파일**이 모두 존재해야 동작합니다. VACUUM을 실행하면 오래된 데이터 파일이 삭제되므로, 그 이전 버전으로의 타임 트래블이 불가능해집니다.

### 보존 기간 설정

```sql
-- 테이블별 로그 보존 기간 설정 (기본값: 30일)
ALTER TABLE catalog.schema.customers
SET TBLPROPERTIES ('delta.logRetentionDuration' = 'interval 90 days');

-- 삭제된 파일 보존 기간 설정 (기본값: 7일, VACUUM 기준)
ALTER TABLE catalog.schema.customers
SET TBLPROPERTIES ('delta.deletedFileRetentionDuration' = 'interval 30 days');
```

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `delta.logRetentionDuration` | 30일 | 트랜잭션 로그(JSON)의 보존 기간 |
| `delta.deletedFileRetentionDuration` | 7일 | VACUUM이 삭제하지 않는 최소 기간 |

> 💡 `logRetentionDuration`을 늘려도, VACUUM이 데이터 파일을 삭제하면 해당 버전의 타임 트래블이 불가능합니다. 타임 트래블 기간을 늘리려면 **두 설정 모두** 함께 조정해야 합니다.

---

## 실전 활용 사례

### 1. 감사(Audit) 및 규정 준수

```sql
-- 분기말 시점의 재무 데이터 확인 (감사 목적)
SELECT * FROM catalog.schema.financial_records
TIMESTAMP AS OF '2025-03-31 23:59:59';
```

### 2. ML 모델 재현성

```sql
-- 모델 학습에 사용한 시점의 데이터를 정확히 재현
CREATE TABLE catalog.schema.training_data_snapshot AS
SELECT * FROM catalog.schema.features
VERSION AS OF 42;
```

### 3. 데이터 품질 디버깅

```sql
-- 어제와 오늘의 데이터를 비교하여 이상 변동 감지
SELECT
    'yesterday' AS period, COUNT(*) AS row_count, SUM(amount) AS total
FROM catalog.schema.orders
TIMESTAMP AS OF current_date() - INTERVAL 1 DAY
UNION ALL
SELECT
    'today', COUNT(*), SUM(amount)
FROM catalog.schema.orders;
```

### 4. ETL 파이프라인에서 재처리

```sql
-- 파이프라인 실패 시, 소스 테이블을 실패 직전 상태로 읽어서 재처리
CREATE OR REPLACE TABLE catalog.schema.silver_orders AS
SELECT * FROM catalog.schema.bronze_orders
VERSION AS OF 100;  -- 마지막 성공 버전
```

---

## 정리

| 기능 | 명령어 | 설명 |
|------|--------|------|
| **이력 확인** | `DESCRIBE HISTORY` | 테이블의 모든 변경 이력을 조회합니다 |
| **버전으로 조회** | `VERSION AS OF n` | 특정 버전의 데이터를 조회합니다 |
| **시각으로 조회** | `TIMESTAMP AS OF` | 특정 시각 기준의 데이터를 조회합니다 |
| **테이블 복원** | `RESTORE TABLE` | 테이블을 과거 버전으로 되돌립니다 |

---

## 참고 링크

- [Databricks: Delta Lake table history](https://docs.databricks.com/aws/en/delta/history.html)
- [Databricks: Restore a Delta table](https://docs.databricks.com/aws/en/delta/restore.html)
- [Azure Databricks: Work with Delta Lake table history](https://learn.microsoft.com/en-us/azure/databricks/delta/history)
- [Databricks: Delta table properties reference](https://docs.databricks.com/aws/en/delta/table-properties.html)
