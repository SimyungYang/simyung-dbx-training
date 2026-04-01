# 관계형 데이터베이스 기초

## 왜 관계형 데이터베이스를 알아야 하나요?

"Databricks를 배우는데 왜 RDBMS부터 알아야 하나요?"라는 질문을 자주 받습니다. 이유는 간단합니다. **현재 사용되는 거의 모든 데이터 기술은 RDBMS의 개념을 기반으로 하거나, RDBMS의 한계를 극복하기 위해 만들어졌기 때문입니다.**SQL, 테이블, JOIN, 트랜잭션 — 이 모든 개념은 RDBMS에서 시작되었고, Databricks에서도 그대로 사용됩니다.

RDBMS를 이해하지 않고 Databricks를 배우면, "왜 Delta Lake에 ACID가 필요한지", "왜 분산 처리에서 트랜잭션이 어려운지"를 이해할 수 없습니다. 기초가 없으면 도구만 따라 쓸 수는 있어도, 문제가 생겼을 때 해결할 수 없습니다.

---

## 관계형 데이터베이스(RDBMS)란?

> 💡 **RDBMS(Relational Database Management System, 관계형 데이터베이스 관리 시스템)**란 데이터를 ** 테이블(표)**형태로 저장하고, 테이블 간의 ** 관계(Relationship)**를 통해 데이터를 연결하는 데이터베이스 시스템입니다.

1970년 IBM의 연구원 ** 에드거 코드(Edgar F. Codd)**가 발표한 논문에서 처음 제안된 이후, 50년 넘게 전 세계에서 가장 널리 사용되는 데이터 저장 방식입니다. 은행, 병원, 쇼핑몰, 정부 시스템 — 여러분이 일상에서 접하는 거의 모든 서비스 뒤에는 RDBMS가 있습니다.

RDBMS의 핵심 가치는 ** 동시에 수천 명이 접근해도 데이터 일관성을 보장**하는 것입니다.

현업에서 이것이 왜 중요한지 구체적으로 설명하겠습니다. 예를 들어, 은행 이체 시스템에서 A 계좌에서 B 계좌로 100만 원을 이체할 때, "A에서 출금"과 "B에 입금"은 **반드시 둘 다 성공하거나 둘 다 실패**해야 합니다. 출금만 되고 입금이 안 되면 100만 원이 사라지는 것이고, 반대면 100만 원이 생기는 것입니다. 이를 보장하는 것이 **ACID 트랜잭션**이며, RDBMS는 이것을 50년간 안정적으로 제공해 왔습니다.

RDBMS의 구조는 단순합니다: **테이블**(행과 열로 구성된 데이터 구조), **관계**(Foreign Key로 테이블 간 연결), **인덱스**(검색 속도를 높이는 자료 구조), 그리고 **SQL**(이 모든 것을 조작하는 표준 언어)입니다. 이 기본 구조를 이해하면 Databricks의 Delta Lake, Unity Catalog, SQL Warehouse가 왜 그렇게 설계되었는지 자연스럽게 이해할 수 있습니다.

---

## 핵심 개념

### 테이블 (Table)

데이터를 저장하는 기본 단위입니다. 행(Row)과 열(Column)으로 구성됩니다.

**고객 테이블 (customers)**

| customer_id (PK) | name | email | city |
|-------------------|------|-------|------|
| 1 | 김철수 | cs@mail.com | 서울 |
| 2 | 이영희 | yh@mail.com | 부산 |
| 3 | 박민수 | ms@mail.com | 대구 |

**주문 테이블 (orders)**

| order_id (PK) | customer_id (FK) | product | amount | order_date |
|----------------|------------------|---------|--------|------------|
| 101 | 1 | 노트북 | 1,200,000 | 2025-03-01 |
| 102 | 2 | 키보드 | 89,000 | 2025-03-02 |
| 103 | 1 | 마우스 | 65,000 | 2025-03-03 |

### 키 (Key)

> 💡 **기본 키(Primary Key, PK)** 는 테이블에서 각 행을 **고유하게 식별**하는 컬럼입니다. 중복될 수 없고, NULL이 될 수 없습니다. 실무에서 가장 흔한 실수는 "이름"이나 "이메일"을 기본 키로 쓰는 것입니다. 동명이인이 있을 수 있고, 이메일은 변경될 수 있으므로 반드시 **의미 없는 순번(Surrogate Key)** 을 사용하세요.

> 💡 **외래 키(Foreign Key, FK)** 는 다른 테이블의 기본 키를 **참조**하여 두 테이블을 연결하는 컬럼입니다. 위 예시에서 주문 테이블의 `customer_id`는 고객 테이블의 `customer_id`를 참조합니다. 이를 통해 "주문 101은 김철수(customer_id=1)의 주문이다"라는 관계가 형성됩니다. 존재하지 않는 customer_id(예: 999)를 넣으려고 하면 RDBMS가 거부합니다 — 이것이 **참조 무결성(Referential Integrity)** 입니다.

### 조인 (JOIN)

> 💡 **조인(JOIN)** 은 두 개 이상의 테이블을 **공통 컬럼을 기준으로 합치는** 연산입니다. 관계형 데이터베이스의 핵심 기능이며, 이것이 "관계형"이라는 이름의 이유입니다.

```sql
-- 고객 이름과 주문 정보를 함께 조회
SELECT
    c.name AS 고객명,
    o.product AS 상품,
    o.amount AS 금액,
    o.order_date AS 주문일
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;
```

결과:

| 고객명 | 상품 | 금액 | 주문일 |
|--------|------|------|--------|
| 김철수 | 노트북 | 1,200,000 | 2025-03-01 |
| 이영희 | 키보드 | 89,000 | 2025-03-02 |
| 김철수 | 마우스 | 65,000 | 2025-03-03 |

### 주요 JOIN 유형

| JOIN 유형 | 설명 | 실무 사용 빈도 | 예시 |
|-----------|------|---------------|------|
| **INNER JOIN**| 양쪽 모두 일치하는 것만 반환합니다 | ★★★★★ | 주문한 고객 목록 |
| **LEFT JOIN**| 왼쪽 테이블 전체 + 오른쪽 일치하는 것 (없으면 NULL) | ★★★★☆ | 모든 고객 + 주문 여부 |
| **RIGHT JOIN**| 오른쪽 테이블 전체 + 왼쪽 일치하는 것 | ★☆☆☆☆ | LEFT JOIN으로 대체 가능 |
| **FULL OUTER JOIN**| 양쪽 모두 전체 데이터를 반환합니다 | ★★☆☆☆ | 데이터 정합성 검증 |
| **CROSS JOIN**| 양쪽의 모든 조합을 생성합니다 (카테시안 곱) | ★☆☆☆☆ | 모든 상품 × 모든 지역 |

> ⚠️ ** 현업에서 가장 흔한 JOIN 실수**: CROSS JOIN을 의도치 않게 만드는 것입니다. JOIN 조건을 빼먹으면 100만 × 100만 = **1조 건**의 결과가 나옵니다. 쿼리가 영원히 안 끝나고, 서버 메모리가 터지는 장애의 원인 1위입니다. JOIN에는 반드시 ON 조건을 명시하세요.

---

## SQL 기본 구문

> 💡 **SQL(Structured Query Language)** 은 관계형 데이터베이스에서 데이터를 조회, 삽입, 수정, 삭제하기 위한 표준 언어입니다. 1970년대 IBM에서 개발되었으며, 현재 거의 모든 데이터 플랫폼(Databricks 포함)에서 사용됩니다. SQL을 한 번 배우면 MySQL, PostgreSQL, Databricks, Snowflake 어디서든 95%는 그대로 쓸 수 있습니다.

### 데이터 조회 (SELECT)

```sql
-- 기본 조회
SELECT name, city FROM customers;

-- 조건 필터링
SELECT * FROM orders WHERE amount > 100000;

-- 집계 함수 — 실무에서 가장 많이 쓰는 패턴
SELECT
    city,
    COUNT(*) AS 고객수,
    SUM(amount) AS 총매출,
    AVG(amount) AS 평균주문금액,
    MAX(amount) AS 최대주문금액
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY city
HAVING COUNT(*) >= 2
ORDER BY 총매출 DESC;
```

### SELECT 구문 실행 순서

SQL은 작성 순서와 실행 순서가 다릅니다. 이 차이를 모르면 "왜 WHERE에서 별칭(alias)을 못 쓰는 거지?" 같은 혼란에 빠집니다.

| 실행 순서 | 구문 | 역할 | 흔한 실수 |
|-----------|------|------|----------|
| 1 | `FROM` / `JOIN` | 어떤 테이블에서 가져올지 결정 | JOIN 조건 누락 → 카테시안 곱 |
| 2 | `WHERE` | 행 단위로 조건 필터링 | 집계 함수를 WHERE에 쓰면 에러 |
| 3 | `GROUP BY` | 그룹별로 묶음 | SELECT에 있는 비집계 컬럼을 안 넣으면 에러 |
| 4 | `HAVING` | 그룹에 대한 조건 필터링 | WHERE와 혼동 |
| 5 | `SELECT` | 어떤 컬럼을 표시할지 결정 | 여기서 정의한 별칭을 WHERE에서 못 씀 |
| 6 | `ORDER BY` | 정렬 | 대용량 데이터에서 불필요한 정렬은 성능 킬러 |
| 7 | `LIMIT` | 결과 건수를 제한 | |

### 데이터 조작 (DML)

```sql
-- 삽입 (INSERT)
INSERT INTO customers (customer_id, name, email, city)
VALUES (4, '최지은', 'je@mail.com', '인천');

-- 수정 (UPDATE) — 반드시 WHERE를 확인하세요!
UPDATE customers SET city = '서울' WHERE customer_id = 3;
-- ⚠️ WHERE 없이 UPDATE하면 전체 행이 변경됩니다. 실무에서 이 실수로 장애가 정말 많이 납니다.

-- 삭제 (DELETE) — UPDATE보다 더 위험합니다
DELETE FROM orders WHERE order_date < '2024-01-01';
-- ⚠️ 프로덕션에서는 DELETE 대신 soft delete(is_deleted 플래그)를 권장합니다
```

> ⚠️ **현업 필수 습관**: UPDATE나 DELETE를 실행하기 전에, 반드시 **같은 WHERE 조건으로 SELECT를 먼저 실행**하여 영향받는 행의 수를 확인하세요. "1건만 바꾸려고 했는데 10만 건이 바뀌었습니다"는 실제로 매우 흔한 사고입니다.

### 데이터 정의 (DDL)

```sql
-- 테이블 생성 (CREATE)
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    category VARCHAR(50),
    price DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 테이블 구조 변경 (ALTER)
ALTER TABLE products ADD COLUMN stock INT DEFAULT 0;

-- 테이블 삭제 (DROP) — 되돌릴 수 없습니다!
DROP TABLE products;
```

> 💡 **DML과 DDL의 차이**: **DML(Data Manipulation Language)**은 데이터 자체를 다루는 명령어(SELECT, INSERT, UPDATE, DELETE)이고, **DDL(Data Definition Language)**은 테이블의 구조(스키마)를 정의하는 명령어(CREATE, ALTER, DROP)입니다. DDL은 대부분의 RDBMS에서 자동 COMMIT되므로, ROLLBACK이 불가능합니다. DROP TABLE을 하면 끝입니다.

---

## ACID 트랜잭션 — 데이터 안전의 핵심

관계형 데이터베이스의 가장 중요한 특성이 **ACID 트랜잭션**입니다. 이것은 "있으면 좋은 기능"이 아니라, **금융, 의료, 전자상거래에서 절대적으로 필요한 안전장치**입니다.

### 실생활 예시: 은행 이체에서 실제로 벌어지는 일

A 계좌에서 B 계좌로 100만 원을 이체하는 상황을 생각해 보겠습니다.

```sql
BEGIN TRANSACTION;

-- 1. A 계좌에서 100만 원 차감
UPDATE accounts SET balance = balance - 1000000 WHERE account_id = 'A';

-- 2. B 계좌에 100만 원 입금
UPDATE accounts SET balance = balance + 1000000 WHERE account_id = 'B';

-- 두 작업 모두 성공하면 확정
COMMIT;
```

만약 1번은 성공했는데, 2번 실행 중 서버가 다운되면 어떻게 될까요? ACID가 없다면 A 계좌에서 100만 원이 사라지고 B 계좌에는 입금되지 않는 — 돈이 증발하는 상황이 발생합니다. **ACID 트랜잭션이 있으면 서버 복구 시 자동으로 ROLLBACK되어, 차감도 취소됩니다.**

| ACID 속성 | 의미 | 이것이 없으면 벌어지는 일 |
|-----------|------|-------------------------|
| **Atomicity (원자성)**| 전부 성공하거나 전부 실패 | 이체 중 장애 시 돈 증발 |
| **Consistency (일관성)**| 트랜잭션 전후 데이터 규칙 유지 | 잔액이 마이너스가 되는 등 비즈니스 규칙 위반 |
| **Isolation (격리성)**| 동시 실행 트랜잭션이 서로 간섭하지 않음 | 같은 좌석을 두 명이 동시 예약 |
| **Durability (지속성)**| COMMIT된 데이터는 서버 장애 후에도 보존 | 전원 꺼지면 오늘 매출 데이터 소실 |

### 격리 수준(Isolation Level)과 실제 발생하는 문제

격리성은 "완벽한 격리"와 "성능" 사이의 트레이드오프입니다. 격리 수준이 높을수록 안전하지만 느려집니다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read | 성능 |
|-----------|-----------|-------------------|-------------|------|
| READ UNCOMMITTED | 발생 | 발생 | 발생 | 가장 빠름 |
| READ COMMITTED | 방지 | 발생 | 발생 | 빠름 |
| REPEATABLE READ | 방지 | 방지 | 발생 | 보통 |
| SERIALIZABLE | 방지 | 방지 | 방지 | 가장 느림 |

** 실제로 이런 문제가 발생합니다:**

- **Dirty Read (더티 리드)**: A가 계좌 잔액을 100만 원에서 0원으로 UPDATE했지만 아직 COMMIT 안 한 상태에서, B가 잔액을 조회하면 0원이 보입니다. A가 ROLLBACK하면 실제 잔액은 100만 원인데, B는 0원이라고 잘못된 정보를 본 것입니다. 대부분의 운영 DB는 READ COMMITTED 이상을 사용하므로 이 문제는 드뭅니다.

- **Phantom Read (팬텀 리드)**: "VIP 고객 수를 세는 쿼리"를 같은 트랜잭션에서 두 번 실행했는데, 첫 번째는 100명, 두 번째는 101명이 나옵니다. 그 사이에 다른 트랜잭션이 VIP 고객을 한 명 추가했기 때문입니다. 리포트 정합성이 중요한 경우 문제가 됩니다.

> 💡 **실무 팁**: 대부분의 웹 애플리케이션은 **READ COMMITTED**로 충분합니다. 금융 시스템에서도 SERIALIZABLE까지 가는 경우는 드물고, 보통 REPEATABLE READ + 낙관적 잠금(Optimistic Locking) 조합을 사용합니다.

### Deadlock — 실제로 발생하는 패턴

**Deadlock(데드락)**은 두 트랜잭션이 서로가 잡고 있는 자원을 기다리면서 영원히 멈추는 상황입니다. 이론으로는 알지만, 실제로 만나면 당황하는 분이 많습니다.

```
-- 트랜잭션 A                    -- 트랜잭션 B
UPDATE accounts                  UPDATE accounts
SET balance = balance - 100      SET balance = balance - 200
WHERE id = 1;  -- A가 id=1 잠금   WHERE id = 2;  -- B가 id=2 잠금

UPDATE accounts                  UPDATE accounts
SET balance = balance + 100      SET balance = balance + 200
WHERE id = 2;  -- id=2 대기 🔒    WHERE id = 1;  -- id=1 대기 🔒
-- A는 B가 id=2를 놓기를 기다림   -- B는 A가 id=1을 놓기를 기다림
-- → 영원히 대기 = DEADLOCK!
```

** 실무에서의 해결법:**
1. **같은 순서로 접근**: 항상 id가 작은 것부터 UPDATE하면 데드락이 원천 차단됩니다
2. **트랜잭션을 짧게**: 잠금을 오래 들고 있을수록 데드락 확률이 올라갑니다
3. **RDBMS의 데드락 감지**: MySQL, PostgreSQL 등은 자동으로 데드락을 감지하고, 한쪽 트랜잭션을 강제 ROLLBACK합니다. 애플리케이션에서는 이때 재시도 로직을 구현해야 합니다

---

## 인덱스 (Index) — 성능의 핵심

> 💡 **인덱스(Index)** 란 테이블에서 특정 데이터를 빠르게 찾기 위한 **색인**입니다. 책 뒤의 "찾아보기" 페이지와 동일한 원리입니다.

### B-Tree 인덱스가 실제로 동작하는 방식

대부분의 RDBMS에서 기본 인덱스는 **B-Tree(Balanced Tree)**구조입니다. 이것이 왜 빠른지 구체적인 수치로 설명하겠습니다.

```
데이터 100만 건에서 1건을 찾을 때:

인덱스 없음 (Full Table Scan):
→ 최악의 경우 100만 번 비교
→ 디스크 I/O: 수천 회
→ 소요 시간: 수 초 ~ 수십 초

B-Tree 인덱스 있음:
→ log₂(1,000,000) ≈ 20번 비교
→ 디스크 I/O: 3~4회 (트리 깊이만큼)
→ 소요 시간: 수 밀리초
```

### B-Tree vs Hash 인덱스

| 특성 | B-Tree | Hash |
|------|--------|------|
| ** 등호 검색**(`WHERE id = 5`) | 빠름 | 매우 빠름 (O(1)) |
| ** 범위 검색**(`WHERE id > 5`) | 가능 | ** 불가능**|
| ** 정렬**(`ORDER BY id`) | 가능 | 불가능 |
| ** 실무 사용 비율** | 95% | 5% (특수 경우만) |

> 💡 실무에서는 거의 항상 B-Tree를 씁니다. Hash 인덱스는 "정확히 일치하는 값을 찾는 것만 필요하고, 범위 검색은 절대 안 하는" 경우에만 유리합니다. 확신이 없으면 B-Tree를 쓰세요.

### Composite Index (복합 인덱스) — 순서가 매우 중요합니다

```sql
-- 이 인덱스를 만들었다고 합시다
CREATE INDEX idx_orders_date_customer
ON orders(order_date, customer_id);

-- ✅ 이 쿼리는 인덱스를 활용합니다 (왼쪽부터 매칭)
SELECT * FROM orders WHERE order_date = '2025-03-01';
SELECT * FROM orders WHERE order_date = '2025-03-01' AND customer_id = 1;

-- ❌ 이 쿼리는 인덱스를 활용하지 못합니다!
SELECT * FROM orders WHERE customer_id = 1;
-- order_date 없이 customer_id만 쓰면, 인덱스의 왼쪽 컬럼을 건너뛰므로 Full Scan
```

이것을 "**Leftmost Prefix Rule (최좌측 접두사 규칙)**" 이라고 합니다. 복합 인덱스 `(A, B, C)`는 `A`, `A+B`, `A+B+C` 조건에서만 효과가 있습니다. `B만`, `C만`, `B+C`로는 인덱스가 동작하지 않습니다.

> ⚠️ **많은 팀이 이 실수를 합니다**: 쿼리 성능이 느려서 인덱스를 추가했는데, 컬럼 순서가 잘못되어 효과가 전혀 없는 경우. `EXPLAIN` 명령어로 실행 계획을 반드시 확인하세요.

### 인덱스의 비용

인덱스는 공짜가 아닙니다.

| 항목 | 인덱스 있음의 비용 |
|------|------------------|
| **INSERT**성능 | 10~30% 느려짐 (인덱스도 함께 갱신해야 함) |
| **UPDATE**성능 | 인덱스 컬럼 변경 시 느려짐 |
| ** 저장 공간**| 테이블 크기의 10~30% 추가 |
| ** 유지보수**| 인덱스 단편화 → 주기적 REBUILD 필요 |

> 💡 ** 실무 원칙**: "조회가 많고 변경이 적은 테이블"에는 인덱스를 적극적으로 만들고, "변경이 빈번한 테이블"에는 꼭 필요한 인덱스만 만드세요. 하나의 테이블에 인덱스가 10개 이상이면 INSERT 성능이 눈에 띄게 나빠집니다.

---

## Connection Pooling — 왜 중요한가

초보 개발자가 만든 애플리케이션이 운영 환경에서 가장 먼저 문제를 일으키는 부분이 **데이터베이스 커넥션 관리**입니다.

### 커넥션이 왜 비싼가

데이터베이스에 연결할 때마다 다음이 발생합니다:
1. TCP 3-way handshake (네트워크 왕복)
2. 인증 및 권한 확인
3. 서버 측 메모리 할당 (MySQL 기준 커넥션당 약 10MB)
4. 세션 초기화

하나의 커넥션을 만드는 데 **50~100ms**가 소요됩니다.

```
사용자 1,000명이 동시에 요청하는 상황:

Connection Pooling 없이:
→ 1,000개의 커넥션 동시 생성 시도
→ 서버 메모리: 10MB × 1,000 = 10GB 소진
→ CPU: 커넥션 생성/해제에 대부분 소비
→ 결과: 서버 다운 또는 극심한 지연

Connection Pooling 사용 시:
→ 미리 20~50개 커넥션을 만들어 재사용
→ 서버 메모리: 10MB × 50 = 500MB
→ 요청이 밀리면 큐에서 대기 (서버는 안전)
→ 결과: 안정적으로 처리
```

> ⚠️ **실무 경험**: 서비스가 갑자기 느려져서 확인해 보면, DB 커넥션이 max에 도달해서 새 요청을 처리할 수 없는 경우가 정말 많습니다. Connection Pool 설정은 반드시 애플리케이션 배포 전에 확인하세요. Java는 HikariCP, Python은 SQLAlchemy의 connection pool이 대표적입니다.

---

## 대표적인 RDBMS 제품

| 제품 | 특징 | 주 사용 환경 | 현업 한마디 |
|------|------|-------------|-----------|
| **MySQL**| 가장 널리 사용되는 오픈소스 RDBMS | 웹 애플리케이션, SaaS | 시작은 MySQL, 하지만 복잡한 쿼리에서 한계가 옵니다 |
| **PostgreSQL**| 가장 기능이 풍부한 오픈소스 RDBMS | 복잡한 쿼리, GIS, JSON | "MySQL에서 안 되는 건 PostgreSQL에서 됩니다" |
| **Oracle Database**| 엔터프라이즈 급 상용 RDBMS | 대기업, 금융, 통신 | 비싸지만 안정적. 한국 대기업의 70%가 사용 |
| **SQL Server**| Microsoft의 상용 RDBMS | Windows 기반 엔터프라이즈 | .NET 환경에서는 사실상 표준 |
| **SQLite**| 파일 기반 경량 RDBMS | 모바일 앱, 임베디드 | 여러분 스마트폰에 이미 들어있습니다 |

---

## RDBMS의 한계 — 왜 Lakehouse가 필요한가

RDBMS는 50년간 검증된 훌륭한 기술이지만, 현대 데이터 환경에서 명확한 한계가 있습니다. ** 이 한계를 이해하면 Databricks 같은 레이크하우스 플랫폼이 왜 필요한지 자연스럽게 이해됩니다.**

| 한계 | RDBMS에서의 문제 | Lakehouse에서의 해결 |
|------|-----------------|-------------------|
| **수직 확장 한계** | 서버 한 대의 물리적 한계 (CPU, 메모리). Oracle Exadata 같은 장비는 수십억 원 | Spark가 수백 대의 서버에 자동 분산. 서버 추가만으로 성능 향상 |
| **비정형 데이터** | 이미지, 동영상, 로그, JSON을 효율적으로 다루기 어려움 | 오브젝트 스토리지에 모든 형식 저장. Delta Lake로 구조화 |
| **비용** | 1PB를 Oracle에 저장하면 라이선스만 연간 수십억 원 | S3/ADLS에 저장하면 월 수백만 원. 1/100 이하 비용 |
| **분석 성능** | 수십억 건 집계 시 수 분~수 시간. OLTP 시스템에 분석 쿼리를 돌리면 운영에도 영향 | Photon 엔진으로 수십억 건을 초 단위로 집계. 컴퓨트와 스토리지 분리 |
| **실시간 + 배치** | 실시간 스트리밍 처리가 근본적으로 어려움 | Structured Streaming으로 실시간/배치 통합 |
| **ML/AI**| 모델 학습을 위한 대규모 데이터 처리가 불가능 | Spark MLlib, GPU 클러스터 지원 |

> 💡 ** 오해하지 마세요**: RDBMS가 나쁜 기술이라는 것이 아닙니다. 운영 시스템(주문 처리, 결제, 회원 관리)에서는 여전히 RDBMS가 최적입니다. 문제는 **분석과 AI** 워크로드를 RDBMS로 처리하려 할 때 발생합니다. "OLTP는 RDBMS, 분석은 Lakehouse" — 이것이 현대 데이터 아키텍처의 기본 원칙입니다.

---

## 정리

| 핵심 개념 | 설명 | 왜 중요한가 |
|-----------|------|-----------|
| **RDBMS**| 데이터를 테이블(행/열) 형태로 저장하고, 관계로 연결하는 시스템 | 모든 데이터 기술의 기초 |
| ** 기본 키 (PK)**| 각 행을 고유하게 식별하는 컬럼 | 의미 없는 순번(Surrogate Key)을 쓰세요 |
| ** 외래 키 (FK)**| 다른 테이블의 PK를 참조하여 관계 형성 | 참조 무결성 보장 |
| **JOIN**| 두 테이블을 공통 컬럼으로 합치는 연산 | ON 조건 누락 주의 |
| **ACID**| 트랜잭션 안전성 보장 4대 속성 | 금융/의료에서 절대적으로 필요 |
| ** 인덱스**| 검색 속도를 높이는 색인 (B-Tree 기반) | 복합 인덱스 순서가 핵심 |
| **Connection Pooling** | 커넥션을 미리 만들어 재사용 | 없으면 운영 장애 확실 |

---

## 참고 링크

- [Databricks: SQL language reference](https://docs.databricks.com/aws/en/sql/language-manual/)
- [PostgreSQL Tutorial](https://www.postgresql.org/docs/current/tutorial.html)
- [MySQL Documentation](https://dev.mysql.com/doc/)
- [Use The Index, Luke (인덱스 심화)](https://use-the-index-luke.com/)
