# SQL 스크립팅 상세

## SQL 스크립팅이란?

**SQL 스크립팅(SQL Scripting)** 은 Databricks SQL에서 **변수, 조건문, 반복문, 예외 처리** 를 포함한 절차적 프로그래밍을 가능하게 하는 기능입니다. 기존에는 Python이나 Scala로 작성해야 했던 복잡한 데이터 처리 로직을 **SQL만으로 구현** 할 수 있습니다.

> 💡 **비유**: 일반 SQL이 "단일 명령"이라면, SQL 스크립팅은 "여러 명령을 순서대로 실행하는 프로그램"입니다. 마치 저장 프로시저(Stored Procedure)를 작성하는 것과 비슷합니다.

---

## DECLARE 변수 선언

### 기본 변수 선언

```sql
-- 타입과 기본값을 지정한 변수 선언
DECLARE target_date DATE DEFAULT CURRENT_DATE() - INTERVAL 1 DAY;
DECLARE batch_size INT DEFAULT 1000;
DECLARE status_msg STRING DEFAULT '';
DECLARE total_amount DECIMAL(12,2);

-- 변수에 값 할당
SET total_amount = 0.00;
```

### 쿼리 결과로 변수 할당

```sql
DECLARE row_count INT;
DECLARE max_date DATE;

-- 단일 값 쿼리 결과를 변수에 할당
SET row_count = (SELECT COUNT(*) FROM silver.orders WHERE order_date = target_date);
SET max_date = (SELECT MAX(order_date) FROM silver.orders);
```

### 변수 타입

| 타입 | 예시 | 설명 |
|------|------|------|
| `INT` / `BIGINT` | `DECLARE count INT DEFAULT 0` | 정수형 |
| `DECIMAL(p,s)` | `DECLARE amount DECIMAL(12,2)` | 고정 소수점 |
| `DOUBLE` | `DECLARE ratio DOUBLE` | 부동 소수점 |
| `STRING` | `DECLARE name STRING DEFAULT ''` | 문자열 |
| `DATE` | `DECLARE d DATE DEFAULT CURRENT_DATE()` | 날짜 |
| `TIMESTAMP` | `DECLARE ts TIMESTAMP` | 타임스탬프 |
| `BOOLEAN` | `DECLARE flag BOOLEAN DEFAULT TRUE` | 불리언 |

---

## SET 문

변수에 값을 할당하거나 Spark 설정을 변경합니다.

```sql
-- 변수 값 할당
SET row_count = 100;
SET status_msg = '처리 완료';

-- 표현식으로 할당
SET batch_id = CONCAT('batch_', DATE_FORMAT(CURRENT_TIMESTAMP(), 'yyyyMMdd_HHmmss'));

-- 쿼리 결과로 할당
SET total_revenue = (
    SELECT SUM(amount) FROM gold.daily_revenue
    WHERE sale_date = target_date
);
```

---

## IF / ELSEIF / ELSE

### 기본 조건문

```sql
DECLARE row_count INT;
DECLARE status STRING;

SET row_count = (SELECT COUNT(*) FROM silver.orders WHERE order_date = CURRENT_DATE() - INTERVAL 1 DAY);

IF row_count = 0 THEN
    SET status = 'NO_DATA';
    INSERT INTO audit.alerts (message, severity, created_at)
    VALUES ('어제 주문 데이터가 없습니다', 'CRITICAL', CURRENT_TIMESTAMP());

ELSEIF row_count < 100 THEN
    SET status = 'LOW_VOLUME';
    INSERT INTO audit.alerts (message, severity, created_at)
    VALUES ('어제 주문이 예상보다 적습니다: ' || CAST(row_count AS STRING), 'WARNING', CURRENT_TIMESTAMP());

ELSE
    SET status = 'NORMAL';
    -- 정상: Gold 테이블 갱신
    INSERT OVERWRITE gold.daily_summary
    SELECT
        CURRENT_DATE() - INTERVAL 1 DAY AS summary_date,
        COUNT(*) AS order_count,
        SUM(amount) AS total_revenue
    FROM silver.orders
    WHERE order_date = CURRENT_DATE() - INTERVAL 1 DAY;
END IF;

SELECT status AS processing_result, row_count AS processed_rows;
```

---

## WHILE 루프

### 기본 반복문

```sql
-- 최근 12개월에 대해 월별 집계 생성
DECLARE month_offset INT DEFAULT 0;

WHILE month_offset < 12 DO
    DECLARE target_month DATE;
    SET target_month = DATE_TRUNC('MONTH', CURRENT_DATE()) - MAKE_INTERVAL(0, month_offset);

    MERGE INTO gold.monthly_summary AS target
    USING (
        SELECT
            target_month AS month,
            COUNT(*) AS order_count,
            SUM(amount) AS total_revenue,
            AVG(amount) AS avg_order_value
        FROM silver.orders
        WHERE DATE_TRUNC('MONTH', order_date) = target_month
    ) AS source
    ON target.month = source.month
    WHEN MATCHED THEN UPDATE SET *
    WHEN NOT MATCHED THEN INSERT *;

    SET month_offset = month_offset + 1;
END WHILE;

SELECT '12개월 집계 완료' AS result;
```

### 동적 테이블 처리

```sql
-- 여러 테이블에 대해 동일한 처리 반복
DECLARE table_index INT DEFAULT 0;
DECLARE table_count INT;
DECLARE current_table STRING;

-- 처리할 테이블 목록을 임시 뷰로 생성
CREATE TEMPORARY VIEW tables_to_process AS
SELECT table_name FROM catalog.information_schema.tables
WHERE table_schema = 'bronze' AND table_name LIKE '%_raw';

SET table_count = (SELECT COUNT(*) FROM tables_to_process);

WHILE table_index < table_count DO
    SET current_table = (
        SELECT table_name FROM (
            SELECT table_name, ROW_NUMBER() OVER (ORDER BY table_name) AS rn
            FROM tables_to_process
        ) WHERE rn = table_index + 1
    );

    -- 각 테이블에 OPTIMIZE 실행
    OPTIMIZE IDENTIFIER('catalog.bronze.' || current_table);

    SET table_index = table_index + 1;
END WHILE;
```

---

## 예외 처리 (SIGNAL)

### SIGNAL로 사용자 정의 오류 발생

```sql
DECLARE data_count INT;
SET data_count = (SELECT COUNT(*) FROM silver.orders WHERE order_date = CURRENT_DATE());

IF data_count = 0 THEN
    -- 사용자 정의 오류 발생
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = '오늘 날짜의 주문 데이터가 없습니다. 데이터 수집 파이프라인을 확인하세요.';
END IF;

-- 데이터가 있으면 이후 로직 계속
SELECT '데이터 검증 통과. 행 수: ' || CAST(data_count AS STRING) AS result;
```

### SQLSTATE 코드

| SQLSTATE | 의미 | 사용 시나리오 |
|----------|------|-------------|
| `'45000'` | 사용자 정의 일반 오류 | 비즈니스 규칙 위반 |
| `'45001'` | 데이터 품질 오류 | 데이터 검증 실패 |
| `'45002'` | 환경 설정 오류 | 필수 설정 누락 |

---

## BEGIN ATOMIC 트랜잭션

여러 테이블에 걸친 **원자적 트랜잭션** 을 수행합니다. 모든 작업이 성공하거나, 하나라도 실패하면 전체가 롤백됩니다.

```sql
-- 계좌 이체: 원자적 트랜잭션
BEGIN ATOMIC
    -- 출금
    UPDATE accounts
    SET balance = balance - 50000
    WHERE account_id = 'A001';

    -- 입금
    UPDATE accounts
    SET balance = balance + 50000
    WHERE account_id = 'A002';

    -- 이체 기록
    INSERT INTO transactions (from_acc, to_acc, amount, txn_time)
    VALUES ('A001', 'A002', 50000, CURRENT_TIMESTAMP());
END;
```

### 트랜잭션 규칙

| 규칙 | 설명 |
|------|------|
| **원자성**| 모든 문이 성공하거나, 전부 롤백됩니다 |
| **지원 대상**| Unity Catalog 관리형 테이블만 지원합니다 |
| **중첩 불가**| 트랜잭션 내에 트랜잭션을 중첩할 수 없습니다 |
| **DDL 제한** | CREATE/ALTER/DROP 등 DDL 문은 트랜잭션 내에서 사용할 수 없습니다 |

---

## 저장 프로시저 패턴

SQL 스크립팅의 모든 기능을 조합하여 재사용 가능한 프로시저 패턴을 구현할 수 있습니다.

### 데이터 품질 검증 프로시저

```sql
-- 범용 데이터 품질 검증 스크립트
DECLARE target_table STRING DEFAULT 'catalog.silver.orders';
DECLARE target_date DATE DEFAULT CURRENT_DATE() - INTERVAL 1 DAY;
DECLARE null_count INT;
DECLARE dup_count INT;
DECLARE negative_count INT;
DECLARE total_issues INT DEFAULT 0;

-- 1. NULL 체크
SET null_count = (
    SELECT COUNT(*)
    FROM IDENTIFIER(target_table)
    WHERE order_id IS NULL AND order_date = target_date
);
SET total_issues = total_issues + null_count;

-- 2. 중복 체크
SET dup_count = (
    SELECT COUNT(*) - COUNT(DISTINCT order_id)
    FROM IDENTIFIER(target_table)
    WHERE order_date = target_date
);
SET total_issues = total_issues + dup_count;

-- 3. 음수 금액 체크
SET negative_count = (
    SELECT COUNT(*)
    FROM IDENTIFIER(target_table)
    WHERE amount < 0 AND order_date = target_date
);
SET total_issues = total_issues + negative_count;

-- 4. 결과 기록
INSERT INTO audit.quality_checks (table_name, check_date, null_count, dup_count, negative_count, total_issues, checked_at)
VALUES (target_table, target_date, null_count, dup_count, negative_count, total_issues, CURRENT_TIMESTAMP());

-- 5. 임계치 초과 시 오류 발생
IF total_issues > 0 THEN
    SIGNAL SQLSTATE '45001'
    SET MESSAGE_TEXT = CONCAT(
        '데이터 품질 이슈 발견: NULL=', null_count,
        ', 중복=', dup_count,
        ', 음수금액=', negative_count
    );
END IF;

SELECT 'Quality check passed' AS result;
```

---

## 정리

| 기능 | 구문 | 핵심 포인트 |
|------|------|-----------|
| **변수 선언**| `DECLARE ... DEFAULT ...` | 타입과 기본값을 지정하여 변수를 선언합니다 |
| **값 할당**| `SET var = expr` | 리터럴, 표현식, 쿼리 결과로 할당합니다 |
| **조건문**| `IF / ELSEIF / ELSE / END IF` | 조건에 따라 다른 SQL을 실행합니다 |
| **반복문**| `WHILE ... DO ... END WHILE` | 조건이 만족하는 동안 반복 실행합니다 |
| **예외 처리**| `SIGNAL SQLSTATE` | 사용자 정의 오류를 발생시킵니다 |
| **트랜잭션** | `BEGIN ATOMIC ... END` | 여러 DML을 원자적으로 실행합니다 |

---

## 참고 링크

- [Databricks: SQL scripting](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-sql-scripting.html)
- [Databricks: DECLARE statement](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-sql-scripting-declare.html)
- [Databricks: Multi-table transactions](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-aux-sql-scripting-begin-end.html)
- [Azure Databricks: SQL scripting](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-syntax-aux-sql-scripting)
