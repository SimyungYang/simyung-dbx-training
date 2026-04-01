# 권한 관리

## Unity Catalog의 권한 모델

Unity Catalog는 **ANSI SQL 표준** 의 GRANT/REVOKE 문을 사용하여 데이터 접근 권한을 관리합니다. 모든 보안 가능한 객체(Securable Object)에 대해 세밀한 권한 제어가 가능합니다.

---

## 보안 가능한 객체 계층

| 계층 | 오브젝트 | 상위 계층 |
|------|---------|-----------|
| **Metastore**| 최상위 컨테이너 | - |
| **Catalog**| 카탈로그 | Metastore |
| **Schema**| 스키마 | Catalog |
| **Table**| 테이블 | Schema |
| **View**| 뷰 | Schema |
| **Volume**| 볼륨 | Schema |
| **Function**| 함수 | Schema |
| **Model**| 모델 | Schema |
| **Connection**| 연결 | Catalog |
| **External Location**| 외부 위치 | Metastore |
| **Storage Credential**| 스토리지 자격 증명 | Metastore |

> 💡 **권한 상속**: 상위 객체에 부여된 권한은 하위 객체에 **자동으로 상속** 됩니다. 예를 들어, Catalog에 `SELECT`를 부여하면 해당 Catalog 아래의 모든 Schema, Table에 SELECT 권한이 적용됩니다.

---

## 권한 유형 (Privilege Types)

### 데이터 객체 권한

| 권한 | 대상 객체 | 설명 |
|------|----------|------|
| **SELECT**| Table, View, MV, Streaming Table | 데이터를 읽을 수 있습니다 |
| **MODIFY**| Table, MV, Streaming Table | 데이터를 INSERT, UPDATE, DELETE할 수 있습니다 |
| **CREATE TABLE**| Schema | 스키마 안에 테이블을 생성할 수 있습니다 |
| **CREATE VIEW**| Schema | 뷰를 생성할 수 있습니다 |
| **CREATE MATERIALIZED VIEW**| Schema | Materialized View를 생성할 수 있습니다 |
| **CREATE VOLUME**| Schema | Volume을 생성할 수 있습니다 |
| **CREATE FUNCTION**| Schema | 함수를 생성할 수 있습니다 |
| **CREATE MODEL**| Schema | ML 모델을 등록할 수 있습니다 |
| **CREATE SCHEMA**| Catalog | 카탈로그 안에 스키마를 생성할 수 있습니다 |
| **CREATE CATALOG**| Metastore | 카탈로그를 생성할 수 있습니다 |
| **USE CATALOG**| Catalog | 카탈로그에 접근할 수 있습니다 (하위 객체 접근의 전제 조건) |
| **USE SCHEMA**| Schema | 스키마에 접근할 수 있습니다 (하위 객체 접근의 전제 조건) |
| **READ VOLUME**| Volume | Volume의 파일을 읽을 수 있습니다 |
| **WRITE VOLUME**| Volume | Volume에 파일을 쓸 수 있습니다 |
| **EXECUTE**| Function | 함수를 실행할 수 있습니다 |
| **APPLY TAG**| Table, Schema, Catalog | 거버넌스 태그를 적용할 수 있습니다 |
| **ALL PRIVILEGES**| 모든 객체 | 해당 객체에 대한 모든 권한을 부여합니다 |

### 관리 권한

| 권한 | 설명 |
|------|------|
| **OWNERSHIP**| 객체의 소유자. 모든 권한을 가지며, 다른 사용자에게 권한을 부여할 수 있습니다 |
| **MANAGE**| External Location, Storage Credential의 관리 권한입니다 |

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
| **사용자**| `` `user@company.com` `` | 개별 사용자 (백틱으로 감쌈) |
| **그룹**| `` `data_analysts` `` | 사용자 그룹 |
| **서비스 프린시팔**| `` `sp-etl-pipeline` `` | 서비스 계정 |

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

> 💡 **행 필터(Row Filter)**를 사용하면, 사용자마다 **보이는 행을 다르게** 설정할 수 있습니다. 예를 들어, 서울 지역 담당자에게는 서울 데이터만 보이도록 할 수 있습니다.

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

> 💡 **컬럼 마스킹(Column Mask)**을 사용하면, 민감한 컬럼의 값을 사용자 권한에 따라 **마스킹(가림) 처리** 할 수 있습니다. 원본 데이터를 변경하지 않고, 조회 시 동적으로 적용됩니다.

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
  email: ***@***.***      phone: 010-****-5678
```

---

## 태그 기반 접근 제어 (ABAC)

> 🆕 **ABAC(Attribute-Based Access Control)**는 거버넌스 태그를 기반으로 접근 정책을 정의하는 기능입니다 (Public Preview). 개별 테이블/컬럼마다 권한을 설정하는 대신, **태그가 붙은 모든 객체에 일괄 적용** 할 수 있습니다.

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
| **최소 권한 원칙**| 필요한 최소한의 권한만 부여합니다. `ALL PRIVILEGES`는 가급적 사용하지 않습니다 |
| **그룹 기반 관리**| 개별 사용자가 아닌 **그룹** 에 권한을 부여합니다. 사용자는 그룹에 추가/제거합니다 |
| **서비스 프린시팔 사용**| 프로덕션 파이프라인은 개인 계정이 아닌 **서비스 프린시팔** 로 실행합니다 |
| **환경별 카탈로그 분리**| `dev`, `staging`, `production` 카탈로그를 분리하여 권한을 다르게 설정합니다 |
| **USE 권한 필수**| `SELECT` 만으로는 부족합니다. `USE CATALOG` + `USE SCHEMA` + `SELECT` 모두 필요합니다 |
| **정기 감사**| `SHOW GRANTS`와 시스템 테이블(`system.access.audit`)로 정기적으로 권한을 점검합니다 |

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

## 현업 사례: 권한을 개인에게 직접 줬다가 퇴사자 정리에 3일 걸린 경험

> 🔥 **거의 모든 조직에서 한 번쯤 겪는 문제입니다.**

프로젝트 초기에는 팀원이 5명이라 "그냥 개인한테 직접 GRANT 해주면 편하지"라고 생각합니다. 하지만 6개월이 지나면 상황이 완전히 달라집니다.

### 실제 사고 시나리오

```
1월: 팀원 5명, 개인별로 GRANT 직접 부여
  GRANT SELECT ON CATALOG production TO `alice@company.com`;
  GRANT SELECT ON CATALOG production TO `bob@company.com`;
  GRANT MODIFY ON SCHEMA production.bronze TO `alice@company.com`;
  ...

6월: 팀원 25명으로 증가, GRANT 문이 200개 이상으로 누적
  - 누가 어디에 접근할 수 있는지 파악 불가
  - SHOW GRANTS 결과가 화면 3페이지

7월: 핵심 데이터 엔지니어 퇴사
  - "이 사람이 어디에 권한이 있었지?"
  - SHOW GRANTS TO `engineer@company.com` → 45개 권한
  - 각각 확인하며 REVOKE하는 데 반나절
  - "혹시 빼먹은 거 없나?" 확인하는 데 또 반나절

8월: 감사(Audit) 요청
  - "프로덕션 데이터에 접근 가능한 사람 목록을 주세요"
  - 개인별로 GRANT된 권한을 전부 모아서 Excel로 정리
  - 3일 걸림... 😱
```

### 이것을 안 하면 벌어지는 일

| 안티패턴 | 결과 | 영향 |
|---------|------|------|
| 개인에게 직접 GRANT | 퇴사자 정리가 N x 권한 수 작업 | 운영 부담 폭증 |
| ALL PRIVILEGES 남발 | 신입사원이 프로덕션 테이블을 DROP | 데이터 유실 사고 |
| 권한 문서화 안 함 | 감사 시 누가 무엇에 접근하는지 파악 불가 | 규정 위반 |
| dev/prod 카탈로그 미분리 | 개발 중 실수로 프로덕션 데이터 변경 | 서비스 장애 |

---

## 그룹 기반 권한 설계 패턴 (역할별 3~5개 그룹)

현업에서 가장 효과적인 권한 관리 방법은 **역할(Role) 기반의 그룹** 을 설계하는 것입니다. 개인에게는 절대 직접 GRANT하지 않고, 그룹 멤버십만 관리합니다.

### 권장 그룹 설계 (5개 기본 그룹)

```sql
-- ===== 1. 데이터 뷰어 (읽기 전용 — 비즈니스 분석가, PM, 경영진) =====
-- Gold 레이어만 조회 가능, 원시 데이터 접근 불가
GRANT USE CATALOG ON CATALOG production TO `data_viewers`;
GRANT USE SCHEMA ON SCHEMA production.gold TO `data_viewers`;
GRANT SELECT ON SCHEMA production.gold TO `data_viewers`;

-- ===== 2. 데이터 분석가 (Gold + Silver 읽기) =====
-- Gold + Silver 레이어 조회, 직접 뷰/함수 생성 가능
GRANT USE CATALOG ON CATALOG production TO `data_analysts`;
GRANT USE SCHEMA ON SCHEMA production.gold TO `data_analysts`;
GRANT USE SCHEMA ON SCHEMA production.silver TO `data_analysts`;
GRANT SELECT ON SCHEMA production.gold TO `data_analysts`;
GRANT SELECT ON SCHEMA production.silver TO `data_analysts`;
GRANT CREATE VIEW ON SCHEMA production.gold TO `data_analysts`;

-- ===== 3. 데이터 엔지니어 (Bronze~Gold 쓰기) =====
-- 모든 레이어에 읽기/쓰기, 테이블 생성 가능
GRANT USE CATALOG ON CATALOG production TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA production.bronze TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA production.silver TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA production.gold TO `data_engineers`;

-- ===== 4. ML 엔지니어 (모델 + 피처 관리) =====
-- Gold 읽기 + ML 전용 스키마 관리
GRANT USE CATALOG ON CATALOG production TO `ml_engineers`;
GRANT USE SCHEMA ON SCHEMA production.gold TO `ml_engineers`;
GRANT SELECT ON SCHEMA production.gold TO `ml_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA production.ml_models TO `ml_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA production.features TO `ml_engineers`;

-- ===== 5. 플랫폼 관리자 (전체 관리) =====
-- 카탈로그 생성, External Location 관리, 권한 부여
GRANT ALL PRIVILEGES ON CATALOG production TO `platform_admins`;
GRANT CREATE CATALOG ON METASTORE TO `platform_admins`;
```

### 사용자 관리는 그룹 멤버십만으로

```
새 분석가 입사 시:
  → data_analysts 그룹에 추가 (1분)
  → 끝. 필요한 모든 권한이 자동으로 적용됩니다.

분석가 퇴사 시:
  → data_analysts 그룹에서 제거 (1분)
  → 끝. 모든 권한이 자동으로 회수됩니다.

분석가 → 엔지니어 전환 시:
  → data_analysts에서 제거 + data_engineers에 추가 (2분)
```

> 💡 **현업 팁**: 그룹은 **IdP(Azure AD, Okta 등)에서 관리하고 SCIM으로 동기화** 하는 것이 가장 좋습니다. 사람이 퇴사하면 HR 시스템 → IdP → Databricks로 자동 연쇄 삭제됩니다. 수동으로 Databricks 그룹을 관리하면 반드시 빠뜨리는 사람이 생깁니다.

---

## 최소 권한 원칙의 실전 적용

"최소 권한 원칙(Principle of Least Privilege)"은 보안 교과서에 항상 나오지만, 현업에서 적용하려면 구체적인 패턴이 필요합니다.

### 흔한 실수: ALL PRIVILEGES의 유혹

```sql
-- ❌ 위험: 편하지만 너무 많은 권한
GRANT ALL PRIVILEGES ON CATALOG production TO `data_engineers`;
-- → data_engineers 그룹의 모든 사람이 production 카탈로그의
--   모든 스키마, 테이블을 생성/삭제/수정할 수 있게 됩니다
-- → 신입 엔지니어가 실수로 DROP TABLE을 실행하면?

-- ✅ 안전: 필요한 권한만 명시적으로 부여
GRANT USE CATALOG ON CATALOG production TO `data_engineers`;
GRANT USE SCHEMA ON SCHEMA production.bronze TO `data_engineers`;
GRANT CREATE TABLE ON SCHEMA production.bronze TO `data_engineers`;
GRANT MODIFY ON SCHEMA production.bronze TO `data_engineers`;
GRANT SELECT ON SCHEMA production.bronze TO `data_engineers`;
-- → DROP TABLE 권한이 없으므로 실수로 테이블 삭제 불가
```

### 환경별 권한 차등 적용

```sql
-- DEV 카탈로그: 자유롭게 실험
GRANT ALL PRIVILEGES ON CATALOG dev TO `data_engineers`;
GRANT ALL PRIVILEGES ON CATALOG dev TO `data_analysts`;

-- STAGING 카탈로그: 제한적 쓰기
GRANT USE CATALOG ON CATALOG staging TO `data_engineers`;
GRANT SELECT ON CATALOG staging TO `data_engineers`;
GRANT CREATE TABLE ON SCHEMA staging.bronze TO `data_engineers`;

-- PRODUCTION 카탈로그: 서비스 프린시팔만 쓰기
GRANT ALL PRIVILEGES ON CATALOG production TO `sp-etl-pipeline`;  -- 서비스 프린시팔
GRANT SELECT ON CATALOG production TO `data_engineers`;           -- 엔지니어는 읽기만
GRANT SELECT ON SCHEMA production.gold TO `data_analysts`;        -- 분석가는 Gold만
```

### 서비스 프린시팔 활용

> 💡 **이것은 프로덕션 환경에서 가장 중요한 패턴입니다.**

```
프로덕션 파이프라인은 절대 개인 계정으로 실행하면 안 됩니다.

이유:
1. 개인이 퇴사하면 파이프라인이 멈춤
2. 개인 계정은 비밀번호 변경, MFA 문제 등으로 인증이 끊길 수 있음
3. 감사 시 "누가 이 데이터를 변경했는가"에 대한 추적이 어려움

서비스 프린시팔 장점:
- 퇴사와 무관하게 항상 동작
- 파이프라인별로 별도의 서비스 프린시팔을 만들면 최소 권한이 자연스럽게 적용
- 감사 로그에서 "어떤 파이프라인이 변경했는지" 명확하게 추적 가능
```

### 정기 감사 쿼리

```sql
-- 분기별 권한 감사 쿼리: 프로덕션 카탈로그에 접근 가능한 주체 확인
SHOW GRANTS ON CATALOG production;

-- 특정 테이블에 누가 접근할 수 있는지 확인
SHOW GRANTS ON TABLE production.gold.daily_revenue;

-- 시스템 테이블로 실제 접근 이력 감사
SELECT
    event_date,
    user_identity.email AS user_email,
    action_name,
    request_params.full_name_arg AS target_object
FROM system.access.audit
WHERE action_name IN ('getTable', 'commandSubmit')
    AND request_params.full_name_arg LIKE 'production.%'
    AND event_date >= CURRENT_DATE() - INTERVAL 30 DAY
ORDER BY event_date DESC
LIMIT 100;
```

> 💡 **현업 팁**: 분기에 한 번 "이 사람이 정말 이 권한이 필요한가?"를 검토하는 **권한 리뷰(Access Review)**를 하세요. 프로젝트가 끝났는데 권한이 남아있는 경우가 전체의 30% 이상입니다. 이것을 안 하면 시간이 지날수록 **권한이 비대해지는(Privilege Creep)** 현상이 발생합니다.

---

## 정리

| 기능 | 설명 |
|------|------|
| **GRANT/REVOKE**| SQL 표준 문법으로 권한을 부여/회수합니다 |
| **권한 상속**| 상위 객체의 권한이 하위 객체에 자동 적용됩니다 |
| **Row Filter**| 사용자별로 보이는 행을 제한합니다 |
| **Column Mask**| 민감한 컬럼의 값을 동적으로 마스킹합니다 |
| **ABAC**| 태그 기반으로 접근 정책을 일괄 적용합니다 (Preview) |
| **USE 권한** | 카탈로그/스키마 접근의 전제 조건입니다 |

---

## 참고 링크

- [Databricks: Manage privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/)
- [Databricks: Row filters and column masks](https://docs.databricks.com/aws/en/data-governance/unity-catalog/row-filters-column-masks.html)
- [Databricks: Privilege types](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/privileges.html)
- [Azure Databricks: Access control](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/manage-privileges/)
