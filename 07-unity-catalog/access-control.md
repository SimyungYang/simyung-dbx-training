# 권한 관리

## Unity Catalog의 권한 모델

Unity Catalog는 **ANSI SQL 표준**의 GRANT/REVOKE 문을 사용하여 데이터 접근 권한을 관리합니다. 모든 보안 가능한 객체(Securable Object)에 대해 세밀한 권한 제어가 가능합니다.

---

## 보안 가능한 객체 계층

| 계층 | 오브젝트 | 상위 계층 |
|------|---------|-----------|
| **Metastore** | 최상위 컨테이너 | - |
| **Catalog** | 카탈로그 | Metastore |
| **Schema** | 스키마 | Catalog |
| **Table** | 테이블 | Schema |
| **View** | 뷰 | Schema |
| **Volume** | 볼륨 | Schema |
| **Function** | 함수 | Schema |
| **Model** | 모델 | Schema |
| **Connection** | 연결 | Catalog |
| **External Location** | 외부 위치 | Metastore |
| **Storage Credential** | 스토리지 자격 증명 | Metastore |

> 💡 **권한 상속**: 상위 객체에 부여된 권한은 하위 객체에 **자동으로 상속**됩니다. 예를 들어, Catalog에 `SELECT`를 부여하면 해당 Catalog 아래의 모든 Schema, Table에 SELECT 권한이 적용됩니다.

---

## 권한 유형 (Privilege Types)

### 데이터 객체 권한

| 권한 | 대상 객체 | 설명 |
|------|----------|------|
| **SELECT** | Table, View, MV, Streaming Table | 데이터를 읽을 수 있습니다 |
| **MODIFY** | Table, MV, Streaming Table | 데이터를 INSERT, UPDATE, DELETE할 수 있습니다 |
| **CREATE TABLE** | Schema | 스키마 안에 테이블을 생성할 수 있습니다 |
| **CREATE VIEW** | Schema | 뷰를 생성할 수 있습니다 |
| **CREATE MATERIALIZED VIEW** | Schema | Materialized View를 생성할 수 있습니다 |
| **CREATE VOLUME** | Schema | Volume을 생성할 수 있습니다 |
| **CREATE FUNCTION** | Schema | 함수를 생성할 수 있습니다 |
| **CREATE MODEL** | Schema | ML 모델을 등록할 수 있습니다 |
| **CREATE SCHEMA** | Catalog | 카탈로그 안에 스키마를 생성할 수 있습니다 |
| **CREATE CATALOG** | Metastore | 카탈로그를 생성할 수 있습니다 |
| **USE CATALOG** | Catalog | 카탈로그에 접근할 수 있습니다 (하위 객체 접근의 전제 조건) |
| **USE SCHEMA** | Schema | 스키마에 접근할 수 있습니다 (하위 객체 접근의 전제 조건) |
| **READ VOLUME** | Volume | Volume의 파일을 읽을 수 있습니다 |
| **WRITE VOLUME** | Volume | Volume에 파일을 쓸 수 있습니다 |
| **EXECUTE** | Function | 함수를 실행할 수 있습니다 |
| **APPLY TAG** | Table, Schema, Catalog | 거버넌스 태그를 적용할 수 있습니다 |
| **ALL PRIVILEGES** | 모든 객체 | 해당 객체에 대한 모든 권한을 부여합니다 |

### 관리 권한

| 권한 | 설명 |
|------|------|
| **OWNERSHIP** | 객체의 소유자. 모든 권한을 가지며, 다른 사용자에게 권한을 부여할 수 있습니다 |
| **MANAGE** | External Location, Storage Credential의 관리 권한입니다 |

---

## GRANT / REVOKE 문법

### 기본 문법

```sql
-- 권한 부여
GRANT <privilege> ON <object_type> <object_name> TO <principal>;

-- 권한 회수
REVOKE <privilege> ON <object_type> <object_name> FROM <principal>;

-- 현재 권한 확인
SHOW GRANTS ON <object_type> <object_name>;
SHOW GRANTS TO <principal>;
```

### principal(권한 대상)

| 유형 | 문법 | 설명 |
|------|------|------|
| **사용자** | `` `user@company.com` `` | 개별 사용자 (백틱으로 감쌈) |
| **그룹** | `` `data_analysts` `` | 사용자 그룹 |
| **서비스 프린시팔** | `` `sp-etl-pipeline` `` | 서비스 계정 |

### 실전 예시

```sql
-- 1. 분석가 그룹에게 프로덕션 카탈로그 접근 허용
GRANT USE CATALOG ON CATALOG production TO `analysts`;
GRANT USE SCHEMA ON SCHEMA production.ecommerce TO `analysts`;
GRANT SELECT ON SCHEMA production.ecommerce TO `analysts`;
-- → production.ecommerce 스키마의 모든 테이블을 조회할 수 있습니다

-- 2. 데이터 엔지니어 그룹에게 테이블 생성 + 수정 권한
GRANT USE CATALOG ON CATALOG production TO `data_engineers`;
GRANT USE SCHEMA ON SCHEMA production.ecommerce TO `data_engineers`;
GRANT CREATE TABLE ON SCHEMA production.ecommerce TO `data_engineers`;
GRANT MODIFY ON SCHEMA production.ecommerce TO `data_engineers`;

-- 3. ML 팀에게 모델 등록 권한
GRANT CREATE MODEL ON SCHEMA production.ml_models TO `ml_team`;

-- 4. 특정 테이블만 읽기 허용 (스키마 전체가 아닌 개별 테이블)
GRANT SELECT ON TABLE production.ecommerce.gold_daily_revenue TO `ceo`;

-- 5. 소유권 이전
ALTER TABLE production.ecommerce.orders SET OWNER TO `data_engineers`;

-- 6. Volume 접근 권한
GRANT READ VOLUME ON VOLUME production.ecommerce.raw_files TO `analysts`;
GRANT WRITE VOLUME ON VOLUME production.ecommerce.raw_files TO `data_engineers`;

-- 7. 현재 권한 확인
SHOW GRANTS ON SCHEMA production.ecommerce;
SHOW GRANTS TO `analysts`;
```

---

## 행 수준 보안 (Row-Level Security)

> 💡 **행 필터(Row Filter)** 를 사용하면, 사용자마다 **보이는 행을 다르게** 설정할 수 있습니다. 예를 들어, 서울 지역 담당자에게는 서울 데이터만 보이도록 할 수 있습니다.

### 설정 방법

```sql
-- 1. 필터 함수 생성
CREATE FUNCTION production.ecommerce.region_filter(region_col STRING)
RETURN
    IS_ACCOUNT_GROUP_MEMBER('admin')        -- 관리자는 모든 행 조회 가능
    OR region_col = SESSION_USER_ATTRIBUTE('region');  -- 본인 담당 지역만

-- 2. 테이블에 행 필터 적용
ALTER TABLE production.ecommerce.orders
SET ROW FILTER production.ecommerce.region_filter ON (region);

-- 3. 행 필터 제거
ALTER TABLE production.ecommerce.orders DROP ROW FILTER;
```

### 동작 방식

```
관리자 (admin 그룹):
  SELECT * FROM orders → 모든 행 반환

서울 담당자 (region='서울'):
  SELECT * FROM orders → region='서울'인 행만 반환

부산 담당자 (region='부산'):
  SELECT * FROM orders → region='부산'인 행만 반환
```

---

## 열 수준 보안 (Column Masking)

> 💡 **컬럼 마스킹(Column Mask)** 을 사용하면, 민감한 컬럼의 값을 사용자 권한에 따라 **마스킹(가림) 처리**할 수 있습니다. 원본 데이터를 변경하지 않고, 조회 시 동적으로 적용됩니다.

### 설정 방법

```sql
-- 1. 마스킹 함수 생성: 이메일 마스킹
CREATE FUNCTION production.ecommerce.mask_email(email_col STRING)
RETURN
    CASE
        WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN email_col          -- 관리자: 원본
        WHEN IS_ACCOUNT_GROUP_MEMBER('analysts') THEN
            CONCAT(LEFT(email_col, 2), '***@', SPLIT(email_col, '@')[1])  -- 분석가: 부분 마스킹
        ELSE '***@***.***'                                             -- 기타: 완전 마스킹
    END;

-- 2. 마스킹 함수 생성: 전화번호 마스킹
CREATE FUNCTION production.ecommerce.mask_phone(phone_col STRING)
RETURN
    CASE
        WHEN IS_ACCOUNT_GROUP_MEMBER('admin') THEN phone_col
        ELSE CONCAT(LEFT(phone_col, 3), '-****-', RIGHT(phone_col, 4))
    END;

-- 3. 테이블에 마스킹 적용
ALTER TABLE production.ecommerce.customers
ALTER COLUMN email SET MASK production.ecommerce.mask_email;

ALTER TABLE production.ecommerce.customers
ALTER COLUMN phone SET MASK production.ecommerce.mask_phone;

-- 4. 마스킹 제거
ALTER TABLE production.ecommerce.customers
ALTER COLUMN email DROP MASK;
```

### 동작 방식

```
관리자:
  email: cs.kim@company.com    phone: 010-1234-5678

분석가:
  email: cs***@company.com     phone: 010-****-5678

일반 사용자:
  email: ***@***.***            phone: 010-****-5678
```

---

## 태그 기반 접근 제어 (ABAC)

> 🆕 **ABAC(Attribute-Based Access Control)** 는 거버넌스 태그를 기반으로 접근 정책을 정의하는 기능입니다 (Public Preview). 개별 테이블/컬럼마다 권한을 설정하는 대신, **태그가 붙은 모든 객체에 일괄 적용**할 수 있습니다.

### 태그 기반 정책 예시

```sql
-- 1. 태그 부여
ALTER TABLE customers ALTER COLUMN email SET TAGS ('pii' = 'true');
ALTER TABLE customers ALTER COLUMN ssn SET TAGS ('pii' = 'true');
ALTER TABLE employees ALTER COLUMN salary SET TAGS ('confidential' = 'true');

-- 2. 태그 기반 마스킹 정책 (모든 'pii' 태그 컬럼에 자동 적용)
-- → ABAC 정책을 통해 'pii=true' 태그가 있는 모든 컬럼에
--   자동으로 마스킹 함수를 적용할 수 있습니다
```

---

## 권한 관리 모범 사례

| 원칙 | 설명 |
|------|------|
| **최소 권한 원칙** | 필요한 최소한의 권한만 부여합니다. `ALL PRIVILEGES`는 가급적 사용하지 않습니다 |
| **그룹 기반 관리** | 개별 사용자가 아닌 **그룹**에 권한을 부여합니다. 사용자는 그룹에 추가/제거합니다 |
| **서비스 프린시팔 사용** | 프로덕션 파이프라인은 개인 계정이 아닌 **서비스 프린시팔**로 실행합니다 |
| **환경별 카탈로그 분리** | `dev`, `staging`, `production` 카탈로그를 분리하여 권한을 다르게 설정합니다 |
| **USE 권한 필수** | `SELECT` 만으로는 부족합니다. `USE CATALOG` + `USE SCHEMA` + `SELECT` 모두 필요합니다 |
| **정기 감사** | `SHOW GRANTS`와 시스템 테이블(`system.access.audit`)로 정기적으로 권한을 점검합니다 |

### 일반적인 역할별 권한 설정

```sql
-- === 분석가 (읽기 전용) ===
GRANT USE CATALOG ON CATALOG production TO `analysts`;
GRANT USE SCHEMA ON SCHEMA production.gold TO `analysts`;
GRANT SELECT ON SCHEMA production.gold TO `analysts`;

-- === 데이터 엔지니어 (읽기 + 쓰기) ===
GRANT USE CATALOG ON CATALOG production TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA production.bronze TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA production.silver TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA production.gold TO `data_engineers`;

-- === 플랫폼 관리자 ===
GRANT ALL PRIVILEGES ON CATALOG production TO `platform_admins`;
GRANT CREATE CATALOG ON METASTORE TO `platform_admins`;
```

---

## 정리

| 기능 | 설명 |
|------|------|
| **GRANT/REVOKE** | SQL 표준 문법으로 권한을 부여/회수합니다 |
| **권한 상속** | 상위 객체의 권한이 하위 객체에 자동 적용됩니다 |
| **Row Filter** | 사용자별로 보이는 행을 제한합니다 |
| **Column Mask** | 민감한 컬럼의 값을 동적으로 마스킹합니다 |
| **ABAC** | 태그 기반으로 접근 정책을 일괄 적용합니다 (Preview) |
| **USE 권한** | 카탈로그/스키마 접근의 전제 조건입니다 |

---

## 참고 링크

- [Databricks: Manage privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/)
- [Databricks: Row filters and column masks](https://docs.databricks.com/aws/en/data-governance/unity-catalog/row-filters-column-masks.html)
- [Databricks: Privilege types](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/privileges.html)
- [Azure Databricks: Access control](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/manage-privileges/)
