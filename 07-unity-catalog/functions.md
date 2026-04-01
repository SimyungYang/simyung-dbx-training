# Unity Catalog 함수 (Functions)

## UC 함수란?

Unity Catalog에서 **함수(Function)** 는 재사용 가능한 로직을 SQL 또는 Python으로 정의하여, 카탈로그에 등록하고 권한을 관리할 수 있는 객체입니다. 일반 데이터베이스의 UDF(User-Defined Function)와 유사하지만, Unity Catalog의 거버넌스(권한, 리니지, 감사)가 적용된다는 점이 핵심입니다.

> 💡 **UDF(User-Defined Function, 사용자 정의 함수)** 란 사용자가 직접 작성한 함수를 의미합니다. Databricks에서는 SQL과 Python 두 언어로 UDF를 작성할 수 있으며, Unity Catalog에 등록하여 조직 전체에서 공유할 수 있습니다.

---

## 함수 유형

| 유형 | 언어 | 특징 | 사용 사례 |
|------|------|------|----------|
| **SQL UDF** | SQL | 가장 기본적인 함수. SQL 표현식으로 구성됩니다 | 데이터 변환, 포맷팅, 비즈니스 규칙 |
| **Python UDF** | Python | Python 코드를 실행합니다. 외부 라이브러리 사용 가능 | 복잡한 로직, ML 추론, 외부 API 호출 |
| ** 테이블 반환 함수** | SQL | 단일 값이 아닌 테이블을 반환합니다 | 동적 데이터 생성, 파라미터화된 조회 |

---

## SQL UDF 생성

### 기본 문법

```sql
CREATE [OR REPLACE] FUNCTION [IF NOT EXISTS]
  <catalog>.<schema>.<function_name>(<param_name> <data_type>, ...)
  RETURNS <return_type>
  [LANGUAGE SQL]
  [COMMENT '<설명>']
  RETURN <expression>;
```

### 실전 예시

```sql
-- 1. 이메일 도메인 추출 함수
CREATE OR REPLACE FUNCTION production.utils.get_email_domain(email STRING)
  RETURNS STRING
  COMMENT '이메일 주소에서 도메인 부분을 추출합니다'
  RETURN SPLIT(email, '@')[1];

-- 사용
SELECT production.utils.get_email_domain('alice@company.com');
-- 결과: 'company.com'

-- 2. 한국 전화번호 포맷팅 함수
CREATE OR REPLACE FUNCTION production.utils.format_phone_kr(phone STRING)
  RETURNS STRING
  COMMENT '전화번호를 010-XXXX-XXXX 형식으로 변환합니다'
  RETURN CONCAT(
    SUBSTR(REGEXP_REPLACE(phone, '[^0-9]', ''), 1, 3), '-',
    SUBSTR(REGEXP_REPLACE(phone, '[^0-9]', ''), 4, 4), '-',
    SUBSTR(REGEXP_REPLACE(phone, '[^0-9]', ''), 8, 4)
  );

-- 3. 나이 그룹 분류 함수
CREATE OR REPLACE FUNCTION production.utils.age_group(age INT)
  RETURNS STRING
  COMMENT '나이를 기반으로 연령대 그룹을 반환합니다'
  RETURN CASE
    WHEN age < 20 THEN '10대'
    WHEN age < 30 THEN '20대'
    WHEN age < 40 THEN '30대'
    WHEN age < 50 THEN '40대'
    WHEN age < 60 THEN '50대'
    ELSE '60대 이상'
  END;

-- 테이블 쿼리에서 사용
SELECT
  name,
  age,
  production.utils.age_group(age) AS age_group
FROM production.ecommerce.customers;
```

---

## Python UDF 생성

### 기본 문법

```sql
CREATE [OR REPLACE] FUNCTION <catalog>.<schema>.<function_name>(<param_name> <data_type>, ...)
  RETURNS <return_type>
  LANGUAGE PYTHON
  [COMMENT '<설명>']
  AS $$
    # Python 코드
    return result
  $$;
```

### 실전 예시

```sql
-- 1. 주민번호 유효성 검사 함수
CREATE OR REPLACE FUNCTION production.utils.validate_ssn(ssn STRING)
  RETURNS BOOLEAN
  LANGUAGE PYTHON
  COMMENT '한국 주민등록번호의 유효성을 검사합니다'
  AS $$
    import re
    if not re.match(r'^\d{6}-?\d{7}$', ssn):
        return False
    digits = re.sub(r'-', '', ssn)
    weights = [2, 3, 4, 5, 6, 7, 8, 9, 2, 3, 4, 5]
    total = sum(int(d) * w for d, w in zip(digits[:12], weights))
    check = (11 - total % 11) % 10
    return check == int(digits[12])
  $$;

-- 2. JSON 특정 필드 추출 함수
CREATE OR REPLACE FUNCTION production.utils.extract_json_field(
  json_str STRING,
  field_name STRING
)
  RETURNS STRING
  LANGUAGE PYTHON
  COMMENT 'JSON 문자열에서 지정된 필드 값을 추출합니다'
  AS $$
    import json
    try:
        data = json.loads(json_str)
        return str(data.get(field_name, ''))
    except (json.JSONDecodeError, TypeError):
        return None
  $$;
```

---

## 테이블 반환 함수 (Table Function)

테이블 반환 함수는 단일 값이 아닌 **행 집합(테이블)** 을 반환합니다.

```sql
-- 날짜 범위를 생성하는 테이블 반환 함수
CREATE OR REPLACE FUNCTION production.utils.date_range(
  start_date DATE,
  end_date DATE
)
  RETURNS TABLE (dt DATE)
  RETURN SELECT EXPLODE(SEQUENCE(start_date, end_date, INTERVAL 1 DAY)) AS dt;

-- 사용: FROM 절에서 함수 호출
SELECT dt
FROM production.utils.date_range('2025-01-01', '2025-01-07');
```

---

## 에이전트 도구로서의 UC 함수

> 🆕 **UC Functions as Tools**: Unity Catalog 함수는 AI 에이전트의 ** 도구(Tool)** 로 활용할 수 있습니다. 에이전트가 함수를 호출하여 데이터 조회, 계산, 외부 API 연동 등을 수행합니다.

```python
# AI 에이전트에서 UC 함수를 도구로 사용하는 예시
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# UC 함수를 에이전트 도구로 등록
tools = [
    {
        "type": "uc_function",
        "function": {
            "name": "production.utils.get_customer_info",
            "description": "고객 ID로 고객 정보를 조회합니다"
        }
    }
]
```

UC 함수를 에이전트 도구로 사용하면 다음과 같은 이점이 있습니다:

| 이점 | 설명 |
|------|------|
| ** 거버넌스** | EXECUTE 권한이 있는 사용자/에이전트만 함수를 실행할 수 있습니다 |
| ** 감사 추적** | 누가 언제 함수를 호출했는지 `system.access.audit`에 기록됩니다 |
| ** 버전 관리** | `CREATE OR REPLACE`로 함수를 업데이트하면 에이전트가 자동으로 최신 버전을 사용합니다 |
| ** 중앙 관리** | 모든 도구가 Unity Catalog에서 일원화 관리됩니다 |

---

## 권한 관리

### 함수 관련 권한

| 권한 | 대상 | 설명 |
|------|------|------|
| **CREATE FUNCTION** | Schema | 스키마 안에 함수를 생성할 수 있습니다 |
| **EXECUTE** | Function | 함수를 실행할 수 있습니다 |
| **ALL PRIVILEGES** | Function | 함수에 대한 모든 권한을 갖습니다 |

### 권한 부여 예시

```sql
-- 분석가 그룹에게 유틸리티 함수 실행 권한 부여
GRANT EXECUTE ON FUNCTION production.utils.get_email_domain TO `analysts`;

-- 스키마 내 모든 함수 실행 권한 부여
GRANT EXECUTE ON SCHEMA production.utils TO `analysts`;

-- 데이터 엔지니어에게 함수 생성 권한 부여
GRANT CREATE FUNCTION ON SCHEMA production.utils TO `data_engineers`;

-- 권한 확인
SHOW GRANTS ON FUNCTION production.utils.get_email_domain;
```

---

## 함수 관리

```sql
-- 함수 목록 조회
SHOW FUNCTIONS IN production.utils;

-- 함수 상세 정보
DESCRIBE FUNCTION production.utils.get_email_domain;
DESCRIBE FUNCTION EXTENDED production.utils.get_email_domain;

-- 함수 삭제
DROP FUNCTION IF EXISTS production.utils.get_email_domain;
```

---

## 실전 인사이트: 비즈니스 로직을 노트북에 산발적으로 두지 마세요

20년간 데이터 엔지니어링을 하면서, "** 같은 로직이 10개 노트북에 복사-붙여넣기 되어 있는**" 상황을 셀 수 없이 많이 봐왔습니다. 한 팀이 `고객등급분류` 로직을 노트북에 작성하면, 다른 팀이 그것을 복사하고, 시간이 지나면 각 노트북의 로직이 미묘하게 달라지면서 ** 같은 고객이 팀마다 다른 등급으로 분류**되는 사태가 발생합니다.

### 노트북 산재 vs UC 함수 중앙화

| 항목 | 노트북에 산재 | UC 함수로 중앙화 |
|------|-------------|-----------------|
| ** 로직 일관성** | 복사본마다 다를 수 있음 | 단일 정의, 모든 곳에서 동일 |
| ** 변경 관리** | 10개 노트북을 모두 찾아서 수정 | 함수 하나만 `CREATE OR REPLACE` |
| ** 권한 관리** | 노트북 접근 권한으로 간접 통제 | `EXECUTE` 권한으로 명확한 통제 |
| ** 감사 추적** | 노트북 실행 로그에서 간접 추적 | `system.access.audit`에 함수별 호출 기록 |
| ** 재사용** | 다른 팀이 복사해야 함 | SQL/Python 어디서든 함수 호출 가능 |
| ** 테스트** | 노트북별로 별도 테스트 | 함수 단위 테스트 가능 |

### 중앙화 대상이 되어야 할 비즈니스 로직 유형

```sql
-- 1. 고객 등급 분류 (모든 팀이 동일 기준 사용해야 함)
CREATE OR REPLACE FUNCTION production.rules.customer_tier(
  total_spend DECIMAL(12,2),
  order_count INT,
  registration_days INT
)
RETURNS STRING
COMMENT '고객 등급을 산정합니다. 마케팅, CS, 분석 팀 모두 이 함수를 사용하세요.'
RETURN CASE
  WHEN total_spend >= 1000000 AND order_count >= 50 THEN 'VIP'
  WHEN total_spend >= 300000 AND order_count >= 20 THEN 'Gold'
  WHEN total_spend >= 100000 AND order_count >= 5 THEN 'Silver'
  ELSE 'Standard'
END;

-- 2. 매출 계산 (할인, 세금, 포인트 차감 등 복잡한 규칙)
CREATE OR REPLACE FUNCTION production.rules.net_revenue(
  gross_amount DECIMAL(12,2),
  discount_pct DECIMAL(5,2),
  tax_rate DECIMAL(5,2),
  point_used DECIMAL(12,2)
)
RETURNS DECIMAL(12,2)
COMMENT '순매출을 계산합니다. 할인, 세금, 포인트를 모두 반영합니다.'
RETURN ROUND(
  (gross_amount * (1 - discount_pct/100) - point_used) * (1 + tax_rate/100),
  2
);
```

> 💡 **규칙**: 만약 **2개 이상의 팀이나 파이프라인에서 동일한 로직을 사용** 한다면, 그것은 UC 함수로 중앙화해야 할 후보입니다. `production.rules` 스키마에 비즈니스 규칙을, `production.utils`에 유틸리티 함수를 분리하여 관리하면 깔끔합니다.

---

## 실전 인사이트: SQL UDF vs Python UDF 성능 차이

UC 함수를 작성할 때 SQL과 Python 중 어떤 언어를 선택해야 할지 고민될 수 있습니다. 성능 차이가 상당하므로, 용도에 따라 신중하게 선택해야 합니다.

### 성능 벤치마크 (1000만 행 테이블, SELECT에서 UDF 호출)

| 함수 유형 | 단순 문자열 처리 | 숫자 계산 | 외부 라이브러리 사용 |
|-----------|----------------|----------|-------------------|
| **SQL UDF** | 12초 | 10초 | N/A (불가) |
| **Python UDF** | 85초 | 45초 | 120초+ |
| ** 오버헤드 배수** | ** 약 7배** | ** 약 4.5배** | - |

Python UDF가 느린 이유는 다음과 같습니다:
1. ** 행 단위 직렬화/역직렬화**: 각 행의 데이터를 JVM에서 Python 프로세스로 전달하고 결과를 돌려받는 과정에서 오버헤드가 발생합니다
2. **Python 프로세스 기동**: 첫 호출 시 Python 인터프리터를 시작하는 콜드 스타트가 있습니다
3. **GIL 제약**: Python의 GIL로 인해 병렬 처리가 제한됩니다

### 언어 선택 가이드

| 상황 | 권장 언어 | 이유 |
|------|----------|------|
| 문자열 변환, 포맷팅 | **SQL** | 기본 SQL 함수가 충분히 빠릅니다 |
| 수학적 계산, CASE 분기 | **SQL** | SQL 표현식이 JVM 내에서 직접 실행됩니다 |
| 정규식 기반 유효성 검사 | **SQL** (간단) / **Python** (복잡) | 간단한 정규식은 SQL `REGEXP`로 충분합니다 |
| 외부 API 호출 | **Python** | SQL에서는 HTTP 호출이 불가능합니다 |
| 복잡한 알고리즘 (ML 추론 등) | **Python** | Python 라이브러리 활용이 필수입니다 |
| 대량 데이터 변환 (수백만 행+) | **SQL** | 성능 차이가 분 단위로 벌어집니다 |

> ⚠️ ** 실전 팁**: Python UDF는 ** 소량 호출(에이전트 도구, 단건 API 호출)** 에는 적합하지만, ** 대량 배치 처리(수백만 행 변환)** 에는 부적합합니다. 대량 처리가 필요한 경우, Python UDF 대신 **Spark DataFrame 변환** 이나 **SQL UDF** 를 사용하세요.

---

## 실전 인사이트: 에이전트 도구로서의 UC 함수 설계 패턴

UC 함수를 AI 에이전트 도구로 설계할 때는, 일반적인 UDF 설계와는 다른 고려사항이 있습니다.

### 에이전트 도구용 함수 설계 원칙

| 원칙 | 일반 UDF | 에이전트 도구 UDF |
|------|---------|-----------------|
| **COMMENT** | 개발자 참고용 (간략) | LLM이 읽는 "사용 설명서" (상세하게) |
| ** 파라미터** | 정확한 타입 (ID는 BIGINT) | LLM이 추론 가능한 유연한 입력 |
| ** 반환값** | 효율적인 데이터 타입 | LLM이 해석 가능한 자연어 포함 |
| ** 에러 처리** | 예외 발생 | 에러 메시지를 문자열로 반환 |

```sql
-- 에이전트 도구용 함수: LLM 친화적 설계
CREATE OR REPLACE FUNCTION production.agent_tools.lookup_product(
  search_term STRING COMMENT '제품명 또는 제품코드. 예: "갤럭시 S24", "PRD-001"'
)
RETURNS TABLE(
  product_name STRING,
  price DECIMAL(10,2),
  stock_status STRING,
  description STRING
)
COMMENT '제품 정보를 검색합니다. 고객이 특정 제품에 대해 문의할 때 사용하세요.
검색어는 제품명의 일부 또는 제품코드를 입력할 수 있습니다.
정확한 제품코드를 아는 경우 "PRD-001" 형태로 입력하면 정확한 결과를 얻을 수 있습니다.
결과가 없으면 빈 테이블이 반환됩니다.'
RETURN
  SELECT
    name AS product_name,
    price,
    CASE
      WHEN stock > 0 THEN CONCAT('재고 있음 (', CAST(stock AS STRING), '개)')
      ELSE '품절'
    END AS stock_status,
    description
  FROM production.ecommerce.products
  WHERE name LIKE CONCAT('%', search_term, '%')
     OR product_code = search_term
  LIMIT 5;
```

> 💡 ** 핵심**: 에이전트 도구용 함수는 **LLM이 자연어를 입력으로 넣어도 작동** 하도록 설계해야 합니다. BIGINT ID를 요구하면 LLM이 ID를 모를 때 도구를 사용할 수 없습니다. `LIKE` 검색이나 유연한 매칭을 지원하고, 반환값에도 "재고 있음 (15개)"처럼 LLM이 바로 사용자에게 전달할 수 있는 자연어를 포함하세요.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SQL UDF** | SQL 표현식으로 작성하는 가장 기본적인 함수입니다 |
| **Python UDF** | Python 코드로 복잡한 로직을 구현합니다 |
| ** 테이블 반환 함수** | FROM 절에서 사용할 수 있는 행 집합을 반환합니다 |
| ** 에이전트 도구** | UC 함수를 AI 에이전트의 도구로 활용할 수 있습니다 |
| **EXECUTE 권한** | 함수 실행을 위해 반드시 필요한 권한입니다 |

---

## 참고 링크

- [Databricks: User-defined functions (UDFs)](https://docs.databricks.com/aws/en/udf/)
- [Databricks: CREATE FUNCTION](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-sql-function.html)
- [Databricks: UC functions as agent tools](https://docs.databricks.com/aws/en/generative-ai/agent-framework/uc-functions-tools.html)
- [Azure Databricks: UDFs](https://learn.microsoft.com/en-us/azure/databricks/udf/)
