# Delta Lake 타임 트래블 (Time Travel)

## 왜 타임 트래블이 필요한가요?

데이터를 다루다 보면 실수는 피할 수 없습니다. 잘못된 UPDATE 하나로 수백만 행의 데이터가 변경될 수 있고, 의도치 않은 DELETE로 중요한 데이터가 사라질 수 있습니다.

> 💡 **타임 트래블(Time Travel)**은 Delta Lake 테이블의 ** 과거 버전 데이터를 조회하거나 복원**할 수 있는 기능입니다. 모든 변경 사항이 트랜잭션 로그에 기록되어 있기 때문에, 마치 시간을 되돌리듯 이전 상태의 데이터에 접근할 수 있습니다.

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

과거 버전의 데이터를 단순히 **조회**하는 것이 아니라, 테이블 자체를 **이전 상태로 되돌리고**싶을 때 사용합니다.

```sql
-- 버전 번호로 복원
RESTORE TABLE catalog.schema.customers TO VERSION AS OF 1;

-- 타임스탬프로 복원
RESTORE TABLE catalog.schema.customers
TO TIMESTAMP AS OF '2025-03-20 10:00:00';
```

> ⚠️ ** 주의사항**: RESTORE는 새로운 버전을 생성하여 과거 상태를 복원합니다. 기존 이력이 삭제되는 것이 아니라, "버전 4: RESTORE to version 1"과 같은 새로운 이력이 추가됩니다. 따라서 RESTORE 이후에도 다시 되돌릴 수 있습니다.

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

## 현장에서 배운 것들: 타임 트래블이 구해준 순간들

### 실수로 DELETE FROM을 WHERE 없이 실행한 사고 복구 실화

이 이야기는 제가 금융 고객 현장에서 직접 목격한 사건입니다. 금요일 오후 4시, 한 시니어 개발자가 테스트 환경에서 작업한다고 생각하고 다음 쿼리를 실행했습니다.

```sql
-- 의도: 테스트 환경의 더미 데이터 삭제
-- 실제: 프로덕션 환경에서 실행됨
DELETE FROM prod.finance.customer_accounts;
-- WHERE 절 없음... 250만 행 전체 삭제
```

데이터베이스 커넥션 문자열을 확인하지 않은 단순한 실수였습니다. 전통적인 데이터베이스였다면, 이 시점에서 DBA가 WAL(Write-Ahead Log) 백업을 뒤져서 복구하는 데 **6~12시간**이 걸렸을 것입니다. 주말 내내 작업해야 했을 겁니다.

Delta Lake에서의 복구는 **3분** 이 걸렸습니다.

```sql
-- Step 1: 방금 무슨 일이 일어났는지 확인 (30초)
DESCRIBE HISTORY prod.finance.customer_accounts LIMIT 5;
-- version 47: DELETE (operationMetrics: numDeletedRows = 2,500,000) ← 이거!
-- version 46: MERGE  (정상적인 일일 적재)

-- Step 2: 삭제 직전 상태로 복원 (2분)
RESTORE TABLE prod.finance.customer_accounts TO VERSION AS OF 46;

-- Step 3: 복원 확인 (30초)
SELECT COUNT(*) FROM prod.finance.customer_accounts;
-- 2,500,000 ✅ 모든 행이 돌아왔습니다
```

> 💡 **이 사고에서 배운 교훈 3가지**:
> 1. **프로덕션 접근은 반드시 Job을 통해서만**: 개인 노트북에서 프로덕션 테이블을 직접 수정하는 것을 정책으로 금지해야 합니다
> 2. **DELETE/UPDATE 전 항상 SELECT로 영향 범위 확인**: `SELECT COUNT(*) FROM ... WHERE ...`를 먼저 실행하는 습관
> 3. **Unity Catalog의 권한 분리**: 개발자에게 프로덕션 테이블 DELETE 권한을 주지 않는 것이 근본 해결

### 타임 트래블이 감사(Audit)에서 활용되는 실제 패턴

금융, 의료, 공공기관에서는 "**특정 시점에 데이터가 어떤 상태였는가?**"를 증명해야 하는 규제 요건이 있습니다. 타임 트래블은 이 요건을 기술적으로 완벽하게 충족합니다.

#### 패턴 1: 분기말 재무 데이터 스냅샷 증빙

```sql
-- 감사인이 요청: "2025년 1분기 말 시점의 고객 잔액 데이터를 보여주세요"
-- TIMESTAMP AS OF로 정확한 시점의 데이터를 재현
SELECT
    account_id,
    balance,
    last_updated
FROM prod.finance.customer_accounts
TIMESTAMP AS OF '2025-03-31 23:59:59';

-- 추가 검증: 그 시점과 현재의 차이 분석
SELECT
    a.account_id,
    a.balance AS q1_end_balance,
    b.balance AS current_balance,
    b.balance - a.balance AS change
FROM (
    SELECT * FROM prod.finance.customer_accounts
    TIMESTAMP AS OF '2025-03-31 23:59:59'
) a
JOIN prod.finance.customer_accounts b
ON a.account_id = b.account_id
WHERE ABS(b.balance - a.balance) > 10000000;  -- 1천만원 이상 변동 계정
```

#### 패턴 2: 데이터 변조 감지

```sql
-- "누군가 과거 데이터를 임의로 수정했는가?" 검증
-- 같은 버전의 데이터가 시간이 지나도 동일한지 확인
SELECT
    v42.account_id,
    v42.balance AS version_42_balance,
    v42_copy.balance AS version_42_recheck
FROM (SELECT * FROM prod.finance.customer_accounts VERSION AS OF 42) v42
FULL OUTER JOIN (SELECT * FROM prod.finance.customer_accounts VERSION AS OF 42) v42_copy
ON v42.account_id = v42_copy.account_id
WHERE v42.balance != v42_copy.balance;
-- 결과가 0행이면 데이터 무결성 확인 (Delta Lake의 불변 파일 특성)
```

#### 패턴 3: 변경 이력 전수 추적 (Change Data Feed 연계)

```sql
-- CDF(Change Data Feed)가 활성화된 테이블에서 모든 변경 기록 추적
SELECT
    _change_type,  -- insert, update_preimage, update_postimage, delete
    _commit_version,
    _commit_timestamp,
    *
FROM table_changes('prod.finance.customer_accounts', 40, 47)
WHERE account_id = 'ACC-12345'
ORDER BY _commit_version;
```

### 7일 VACUUM 보존과 비용의 트레이드오프: 실전 가이드

타임 트래블 보존 기간 설정은 **비용 vs 안전성**의 트레이드오프입니다. 20년간의 경험에서 나온 가이드라인을 공유합니다.

#### 보존 기간별 실전 시나리오

| 보존 기간 | 적합한 상황 | 리스크 | 추가 스토리지 비용 (1TB 테이블, 일 10% 변경 기준) |
|----------|-----------|-------|----------------------------------------------|
| **7일 (기본)**| 일반적인 ETL 테이블, 사고 시 1주일 내 발견 가능 | 7일 전 사고는 복구 불가 | 기준 (추가 ~700GB) |
| **30일**| 월말 결산 데이터, 감사 대비 | 30일 이상 지난 변경은 추적 불가 | ~3TB 추가 |
| **90일**| 금융 규제 대상, 분기 감사 필요 | 스토리지 비용 증가 | ~9TB 추가 |
| **365일**| 연 1회 감사, 법적 보존 의무 | 비용이 매우 높아짐 | ~36TB 추가 |

```sql
-- 실전 설정 예시: 테이블 용도에 따라 차등 적용
-- Gold 테이블 (비즈니스 보고용): 90일 보존
ALTER TABLE prod.gold.financial_summary
SET TBLPROPERTIES (
    'delta.logRetentionDuration' = 'interval 90 days',
    'delta.deletedFileRetentionDuration' = 'interval 90 days'
);

-- Bronze 테이블 (원본 적재용): 30일 보존
ALTER TABLE prod.bronze.raw_transactions
SET TBLPROPERTIES (
    'delta.logRetentionDuration' = 'interval 30 days',
    'delta.deletedFileRetentionDuration' = 'interval 30 days'
);

-- Silver 테이블 (가공 중간): 14일 보존 (재생성 가능하므로)
ALTER TABLE prod.silver.cleaned_transactions
SET TBLPROPERTIES (
    'delta.logRetentionDuration' = 'interval 14 days',
    'delta.deletedFileRetentionDuration' = 'interval 14 days'
);
```

> ⚠️ ** 이것을 안 하면 이런 일이 벌어집니다**: 한 고객이 모든 테이블의 보존 기간을 365일로 설정했습니다. 1년 뒤 스토리지 비용이 **원래 데이터 크기의 15배**가 되어 있었습니다. 매일 UPDATE가 많은 테이블이었기 때문에, 365일치의 이전 파일이 모두 보존되고 있었던 것입니다. 보존 기간은 **테이블의 변경 빈도와 비즈니스 요건을 고려하여 차등 적용**해야 합니다.

### 타임 트래블이 안 되는 경우: 알아두어야 할 함정들

| 상황 | 원인 | 예방법 |
|------|------|--------|
| "Version X not found" 에러 | VACUUM이 해당 버전의 데이터 파일을 삭제함 | `deletedFileRetentionDuration`을 필요한 만큼 설정 |
| 복원했는데 스키마가 달라짐 | 과거 버전과 현재 버전의 스키마가 다름 (컬럼 추가/삭제) | RESTORE 전 `DESCRIBE`로 스키마 확인 |
| RESTORE 후 다운스트림 테이블 불일치 | 상위 테이블만 복원하고 의존 테이블은 그대로 | 리니지를 확인하고 의존 테이블도 함께 재처리 |
| 외부 도구에서 직접 파일 삭제 | S3/ADLS에서 직접 Parquet 파일을 삭제하면 Delta Log와 불일치 | 반드시 Delta Lake API(SQL/Spark)를 통해서만 데이터 조작 |

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
