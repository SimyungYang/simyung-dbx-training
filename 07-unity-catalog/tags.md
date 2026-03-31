# 태그와 ABAC (Attribute-Based Access Control)

## 거버넌스 태그란?

Unity Catalog의 **태그(Tag)** 는 데이터 자산(테이블, 컬럼, 스키마 등)에 **메타데이터 라벨**을 부착하여 데이터를 분류하고 관리하는 기능입니다. 태그를 활용하면 "이 컬럼은 개인정보다", "이 테이블은 기밀 등급이다"와 같은 **데이터 분류 체계**를 구축할 수 있습니다.

> 💡 **왜 필요한가요?** 조직에 수천 개의 테이블과 수만 개의 컬럼이 있을 때, 어떤 컬럼이 PII(개인 식별 정보)인지 일일이 기억하기 어렵습니다. 태그를 부착하면 자동으로 분류되고, 나아가 **태그를 기반으로 접근 정책을 일괄 적용**할 수 있습니다.

---

## 태그의 구조

태그는 **키-값(Key-Value) 쌍**으로 구성됩니다.

| 구성 요소 | 설명 | 예시 |
|-----------|------|------|
| **키(Key)** | 태그의 이름 (분류 기준) | `pii`, `sensitivity`, `domain` |
| **값(Value)** | 태그의 값 (분류 값) | `true`, `high`, `finance` |

### 태그 적용 가능 객체

| 객체 유형 | 태그 대상 | 예시 |
|-----------|----------|------|
| **Catalog** | 카탈로그 전체 분류 | `environment = production` |
| **Schema** | 스키마 분류 | `domain = ecommerce` |
| **Table** | 테이블 분류 | `sensitivity = confidential` |
| **Column** | 컬럼별 분류 | `pii = true`, `pii_type = email` |
| **Volume** | 볼륨 분류 | `data_type = raw_files` |

---

## 태그 설정 문법

### 태그 추가

```sql
-- 테이블에 태그 추가
ALTER TABLE production.ecommerce.customers
  SET TAGS ('sensitivity' = 'high', 'domain' = 'ecommerce');

-- 컬럼에 태그 추가
ALTER TABLE production.ecommerce.customers
  ALTER COLUMN email SET TAGS ('pii' = 'true', 'pii_type' = 'email');

ALTER TABLE production.ecommerce.customers
  ALTER COLUMN phone SET TAGS ('pii' = 'true', 'pii_type' = 'phone');

ALTER TABLE production.ecommerce.customers
  ALTER COLUMN ssn SET TAGS ('pii' = 'true', 'pii_type' = 'ssn');

-- 스키마에 태그 추가
ALTER SCHEMA production.ecommerce
  SET TAGS ('domain' = 'ecommerce', 'owner_team' = 'data-platform');

-- 카탈로그에 태그 추가
ALTER CATALOG production
  SET TAGS ('environment' = 'production');
```

### 태그 제거

```sql
-- 테이블 태그 제거
ALTER TABLE production.ecommerce.customers
  UNSET TAGS ('sensitivity');

-- 컬럼 태그 제거
ALTER TABLE production.ecommerce.customers
  ALTER COLUMN email UNSET TAGS ('pii_type');
```

---

## 태그 조회

### SQL로 조회

```sql
-- 테이블의 태그 확인
SELECT tag_name, tag_value
FROM system.information_schema.table_tags
WHERE catalog_name = 'production'
  AND schema_name = 'ecommerce'
  AND table_name = 'customers';

-- 컬럼의 태그 확인
SELECT column_name, tag_name, tag_value
FROM system.information_schema.column_tags
WHERE catalog_name = 'production'
  AND schema_name = 'ecommerce'
  AND table_name = 'customers';
```

### 태그 기반 데이터 자산 검색

```sql
-- PII 태그가 있는 모든 컬럼 찾기
SELECT
  catalog_name,
  schema_name,
  table_name,
  column_name,
  tag_value AS pii_type
FROM system.information_schema.column_tags
WHERE tag_name = 'pii'
  AND tag_value = 'true';

-- sensitivity가 high인 모든 테이블 찾기
SELECT
  catalog_name,
  schema_name,
  table_name
FROM system.information_schema.table_tags
WHERE tag_name = 'sensitivity'
  AND tag_value = 'high';
```

---

## 태그 기반 데이터 분류 체계

### 권장 태그 분류 체계

조직에서 일관된 태그를 사용하려면 **표준 태그 체계**를 정의하는 것이 중요합니다.

| 카테고리 | 태그 키 | 가능한 값 | 설명 |
|---------|---------|----------|------|
| **개인정보** | `pii` | `true`, `false` | 개인 식별 정보 여부 |
| **PII 유형** | `pii_type` | `email`, `phone`, `ssn`, `name`, `address` | PII의 세부 유형 |
| **민감도** | `sensitivity` | `public`, `internal`, `confidential`, `restricted` | 데이터 민감도 등급 |
| **도메인** | `domain` | `ecommerce`, `hr`, `finance`, `marketing` | 비즈니스 도메인 |
| **환경** | `environment` | `production`, `staging`, `dev` | 배포 환경 |
| **규제** | `compliance` | `gdpr`, `hipaa`, `pci-dss` | 적용되는 규제 프레임워크 |
| **보존 기간** | `retention` | `30d`, `1y`, `7y`, `permanent` | 데이터 보존 기간 |

### 실전 태그 적용 예시

```sql
-- 고객 테이블: 기밀 등급, GDPR 대상
ALTER TABLE production.ecommerce.customers
  SET TAGS (
    'sensitivity' = 'confidential',
    'compliance' = 'gdpr',
    'domain' = 'ecommerce',
    'retention' = '7y'
  );

-- 이메일 컬럼: PII
ALTER TABLE production.ecommerce.customers
  ALTER COLUMN email SET TAGS ('pii' = 'true', 'pii_type' = 'email');

-- 주문 테이블: 내부용
ALTER TABLE production.ecommerce.orders
  SET TAGS (
    'sensitivity' = 'internal',
    'domain' = 'ecommerce'
  );
```

---

## ABAC: 태그 기반 접근 제어

> 🆕 **ABAC(Attribute-Based Access Control)** 는 태그를 기반으로 접근 정책을 **자동 적용**하는 기능입니다 (Public Preview). 개별 테이블/컬럼마다 권한을 설정하는 대신, "PII 태그가 있는 모든 컬럼에 마스킹을 적용"하는 식의 **정책 기반 제어**가 가능합니다.

### ABAC vs 기존 RBAC 비교

| 항목 | RBAC (기존) | ABAC (태그 기반) |
|------|-----------|-----------------|
| **정책 대상** | 개별 테이블/컬럼 | 태그가 붙은 모든 객체 |
| **확장성** | 새 테이블마다 수동 설정 | 태그만 부착하면 자동 적용 |
| **관리 부담** | 테이블 수에 비례 | 정책 수에 비례 (훨씬 적음) |
| **일관성** | 누락 위험 있음 | 태그 기반 자동 적용으로 일관 |

### ABAC 정책 개념

```sql
-- 개념적 예시 (ABAC 정책)
-- "pii=true 태그가 있는 모든 컬럼에 마스킹 정책을 자동 적용"

-- 1. 태그 부여 (데이터 엔지니어)
ALTER TABLE customers ALTER COLUMN email SET TAGS ('pii' = 'true');
ALTER TABLE employees ALTER COLUMN email SET TAGS ('pii' = 'true');
ALTER TABLE partners ALTER COLUMN contact_email SET TAGS ('pii' = 'true');

-- 2. ABAC 정책 설정 (보안 관리자)
-- → 'pii=true' 태그가 있는 모든 컬럼에
--   자동으로 마스킹 함수가 적용됩니다
-- → 새로 추가되는 테이블의 컬럼에 'pii=true' 태그를 부착하면
--   별도 마스킹 설정 없이 자동으로 정책이 적용됩니다
```

### ABAC 활용 시나리오

| 시나리오 | 태그 조건 | 자동 적용 정책 |
|---------|---------|--------------|
| **PII 마스킹** | `pii = true` | 비관리자에게 마스킹 함수 적용 |
| **기밀 데이터 접근** | `sensitivity = restricted` | 특정 그룹만 SELECT 허용 |
| **규제 대상 필터링** | `compliance = gdpr` | EU 지역 사용자만 접근 허용 |
| **도메인 분리** | `domain = finance` | finance 그룹만 접근 허용 |

---

## 태그 관련 권한

| 권한 | 설명 |
|------|------|
| **APPLY TAG** | 객체에 태그를 설정/해제할 수 있습니다 |

```sql
-- 데이터 엔지니어에게 태그 적용 권한 부여
GRANT APPLY TAG ON TABLE production.ecommerce.customers TO `data_engineers`;

-- 스키마 수준에서 태그 적용 권한 부여
GRANT APPLY TAG ON SCHEMA production.ecommerce TO `data_engineers`;
```

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **표준 태그 체계** | 조직 전체에서 사용할 태그 키와 허용 값을 사전에 정의합니다 |
| **자동 태그 부착** | 데이터 수집 파이프라인에서 PII 컬럼에 자동으로 태그를 부착합니다 |
| **정기 감사** | `system.information_schema.column_tags`로 태그 누락을 주기적으로 점검합니다 |
| **ABAC 활용** | 태그 기반 정책으로 수동 권한 설정을 최소화합니다 |
| **태그 거버넌스** | 태그 생성/수정 권한을 데이터 스튜어드에게만 부여합니다 |

---

## 실전 인사이트: PII 태그를 수동으로 관리하다가 누락이 발생한 사례

한 이커머스 기업에서 데이터 스튜어드가 새 테이블이 생성될 때마다 수동으로 PII 컬럼에 태그를 부착하고 있었습니다. 처음에는 테이블이 20~30개라 관리가 가능했지만, 1년이 지나면서 테이블이 300개 이상으로 늘어났고, 다음과 같은 문제가 발생했습니다.

| 문제 | 구체적 상황 |
|------|-----------|
| **신규 테이블 누락** | 데이터 엔지니어가 새 테이블을 생성하고 스튜어드에게 알리지 않아, PII 컬럼에 태그가 없는 상태로 3개월간 운영 |
| **컬럼명 불일치** | 어떤 테이블은 `email`, 다른 테이블은 `user_email`, `contact_email`로 컬럼명이 달라서 PII 식별이 어려움 |
| **ABAC 정책 공백** | 태그가 없는 PII 컬럼에는 마스킹 정책이 적용되지 않아, 비인가자가 원본 데이터를 조회 |
| **감사 지적** | 개인정보보호 감사에서 "PII 컬럼 중 23%에 태그가 누락되어 있다"는 지적 |

> ⚠️ **교훈**: PII 태그를 수동으로 관리하면, 테이블이 100개를 넘는 시점부터 **반드시 누락이 발생**합니다. 자동화가 필수입니다.

---

## 실전 인사이트: 자동 태깅 파이프라인 구축 패턴

### 패턴 1: 컬럼명 기반 자동 태깅

가장 간단하면서도 효과적인 방법입니다. 컬럼명에 특정 패턴이 포함되어 있으면 자동으로 PII 태그를 부착합니다.

```sql
-- PII 컬럼명 패턴 정의 테이블
CREATE OR REPLACE TABLE production.governance.pii_patterns (
  pattern STRING,
  pii_type STRING,
  description STRING
);

INSERT INTO production.governance.pii_patterns VALUES
  ('%email%', 'email', '이메일 주소'),
  ('%phone%', 'phone', '전화번호'),
  ('%mobile%', 'phone', '휴대폰 번호'),
  ('%ssn%', 'ssn', '주민등록번호'),
  ('%jumin%', 'ssn', '주민번호'),
  ('%birth%', 'birth_date', '생년월일'),
  ('%address%', 'address', '주소'),
  ('%card_number%', 'card', '카드번호'),
  ('%account_no%', 'account', '계좌번호');

-- 태그 미부착 PII 컬럼 식별 쿼리
SELECT
  c.table_catalog,
  c.table_schema,
  c.table_name,
  c.column_name,
  p.pii_type,
  CASE WHEN ct.tag_name IS NOT NULL THEN '태그 있음' ELSE '⚠️ 태그 누락' END AS status
FROM system.information_schema.columns c
CROSS JOIN production.governance.pii_patterns p
LEFT JOIN system.information_schema.column_tags ct
  ON c.table_catalog = ct.catalog_name
  AND c.table_schema = ct.schema_name
  AND c.table_name = ct.table_name
  AND c.column_name = ct.column_name
  AND ct.tag_name = 'pii'
WHERE LOWER(c.column_name) LIKE p.pattern
  AND c.table_catalog = 'production';
```

### 패턴 2: AI 기반 자동 PII 탐지

컬럼명으로 식별하기 어려운 경우(예: `field_1`, `col_a` 같은 비표준 이름), **데이터 샘플을 분석하여 PII를 탐지**하는 방법입니다.

```python
# AI 함수를 활용한 PII 자동 탐지 (개념)
# 실제 구현 시 Databricks의 ai_classify 또는 외부 NER 모델을 활용합니다

def detect_pii_columns(table_name: str) -> list:
    """테이블의 각 컬럼에서 PII 패턴을 탐지합니다"""

    # 1. 각 STRING 컬럼에서 샘플 100행 추출
    # 2. 정규식 패턴 매칭 (이메일, 전화번호, 주민번호 등)
    # 3. 매칭률이 80% 이상이면 해당 PII 유형으로 분류

    pii_patterns = {
        'email': r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
        'phone': r'^01[016789]-?\d{3,4}-?\d{4}$',
        'ssn': r'^\d{6}-?\d{7}$',
        'card': r'^\d{4}-?\d{4}-?\d{4}-?\d{4}$'
    }
    # ... 탐지 로직
```

### 패턴 3: 파이프라인 통합 자동 태깅

데이터 수집 파이프라인(SDP, Lakeflow Jobs)에서 테이블을 생성/변경할 때 **자동으로 태그를 부착**하는 패턴입니다.

```python
# 테이블 생성 후 자동 태깅 (파이프라인의 post-processing 단계)
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

def auto_tag_table(catalog, schema, table):
    """신규 테이블 생성 후 자동으로 PII 태그를 부착합니다"""

    # 1. PII 패턴에 매칭되는 컬럼 식별
    columns = spark.sql(f"DESCRIBE TABLE {catalog}.{schema}.{table}").collect()

    pii_column_patterns = {
        'email': 'email', 'phone': 'phone', 'mobile': 'phone',
        'ssn': 'ssn', 'jumin': 'ssn', 'address': 'address'
    }

    # 2. 매칭된 컬럼에 태그 부착
    for col in columns:
        col_name_lower = col.col_name.lower()
        for pattern, pii_type in pii_column_patterns.items():
            if pattern in col_name_lower:
                spark.sql(f"""
                    ALTER TABLE {catalog}.{schema}.{table}
                    ALTER COLUMN {col.col_name}
                    SET TAGS ('pii' = 'true', 'pii_type' = '{pii_type}')
                """)
                print(f"Tagged {col.col_name} as pii_type={pii_type}")
```

---

## 실전 인사이트: ABAC가 RBAC보다 나은 실전 상황

### 상황: 100개 테이블에 PII 마스킹 적용

| 접근 방식 | RBAC (기존) | ABAC (태그 기반) |
|-----------|-----------|-----------------|
| **초기 설정** | 100개 테이블 x 3~5개 PII 컬럼 = 300~500개 마스킹 설정 | 1개 ABAC 정책 + 태그 부착 |
| **신규 테이블 추가** | 매번 수동으로 마스킹 설정 추가 | 태그만 부착하면 자동 적용 |
| **정책 변경** | 300~500개 마스킹 함수를 모두 수정 | 1개 ABAC 정책만 수정 |
| **감사 준비** | 각 테이블별로 마스킹 현황 확인 | 태그 조회 쿼리 1개로 전체 현황 파악 |

### ABAC가 RBAC보다 확실히 나은 시나리오

| 시나리오 | 이유 |
|---------|------|
| **데이터 자산이 100개 이상** | RBAC은 자산 수에 비례하여 관리 비용 증가. ABAC는 정책 수에만 비례 |
| **팀/프로젝트 간 데이터 공유가 빈번** | 새로운 공유 요청마다 RBAC 설정 vs 태그 기반 자동 적용 |
| **규제 요건이 자주 변경** | GDPR → 개인정보보호법 변경 시 태그 기반 정책 1개만 수정 |
| **멀티 카탈로그 환경** | 여러 카탈로그에 걸친 일관된 정책 적용이 ABAC로 쉬움 |

### RBAC가 여전히 적합한 시나리오

| 시나리오 | 이유 |
|---------|------|
| **소규모 조직 (테이블 50개 이하)** | ABAC 설정 오버헤드가 RBAC보다 큼 |
| **단순한 권한 구조** | "이 팀은 이 스키마만 접근"이면 RBAC으로 충분 |
| **ABAC가 Preview 단계** | 프로덕션 안정성이 중요하면 GA까지 대기 |

> 💡 **실전 판단 기준**: 조직의 데이터 자산이 **100개 이상**이고, **PII 보호 요건**이 있다면 ABAC 도입을 적극 검토하세요. 태그 부착 자동화 파이프라인과 함께 도입하면, RBAC 대비 관리 비용을 **80% 이상 절감**할 수 있습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **태그** | 키-값 쌍으로 데이터 자산을 분류하는 메타데이터 라벨입니다 |
| **데이터 분류** | PII, 민감도, 도메인 등의 기준으로 데이터를 체계적으로 분류합니다 |
| **ABAC** | 태그 기반 접근 제어로, 정책을 한 곳에서 정의하면 모든 관련 객체에 자동 적용됩니다 |
| **APPLY TAG 권한** | 태그 부착/제거를 위해 필요한 권한입니다 |
| **태그 검색** | `information_schema`를 통해 태그 기반으로 데이터 자산을 검색할 수 있습니다 |

---

## 참고 링크

- [Databricks: Tags](https://docs.databricks.com/aws/en/data-governance/unity-catalog/tags.html)
- [Databricks: Attribute-based access control](https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/)
- [Databricks: Govern data with tags](https://docs.databricks.com/aws/en/data-governance/unity-catalog/tags.html)
- [Azure Databricks: Tags](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/tags)
