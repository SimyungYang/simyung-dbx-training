# Unity Catalog란?

## 왜 Unity Catalog가 필요한가?

데이터 플랫폼이 성장하면서 조직은 다음과 같은 **거버넌스 문제**에 직면하게 됩니다.

| 문제 | 구체적 상황 |
|------|-------------|
| **데이터 사일로** | 각 팀이 자체 데이터 저장소를 운영하며, 같은 데이터가 여러 곳에 중복됩니다 |
| **권한 관리 혼란** | 누가 어떤 데이터에 접근할 수 있는지 추적이 어렵습니다 |
| **감사 불가** | 규정 준수(GDPR, HIPAA 등)를 위해 데이터 접근 이력이 필요하지만, 추적할 방법이 없습니다 |
| **리니지 부재** | 데이터가 어디서 왔고, 어디로 흘러가는지 알 수 없어 장애 영향도 파악이 어렵습니다 |
| **ML 모델 관리** | 테이블은 관리되지만 ML 모델, 함수 등은 별도 시스템에서 관리해야 합니다 |

> 💡 **데이터 거버넌스(Data Governance)란?** 조직의 데이터가 안전하게 관리되고, 적절한 사람만 접근할 수 있으며, 규정을 준수하도록 하는 정책과 프로세스의 총체입니다. "누가, 어떤 데이터를, 언제, 왜 사용했는지"를 추적하고 통제하는 것이 핵심입니다.

---

## Hive Metastore의 한계

Unity Catalog 이전에 Databricks는 **Hive Metastore(HMS)** 를 사용하여 테이블 메타데이터를 관리했습니다. 하지만 HMS에는 근본적인 한계가 있었습니다.

| 항목 | Hive Metastore (HMS) | Unity Catalog (UC) |
|------|---------------------|-------------------|
| **관리 범위** | 테이블, 뷰만 관리 | 테이블, 뷰, 볼륨, 함수, ML 모델, 커넥션 등 |
| **네임스페이스** | 2-Level (`schema.table`) | **3-Level** (`catalog.schema.table`) |
| **권한 모델** | Workspace 단위로 격리 | **Account 단위**로 통합 관리 |
| **세밀한 접근 제어** | 테이블 수준까지만 | **행(Row), 열(Column)** 수준까지 |
| **크로스 워크스페이스** | 불가 (각 Workspace 별도 HMS) | **여러 Workspace에서 동일 데이터 공유** |
| **감사(Audit)** | 제한적 | **통합 감사 로그** (시스템 테이블) |
| **데이터 리니지** | 없음 | **자동 리니지 추적** |
| **Identity 관리** | Workspace별 별도 관리 | **Account 수준 Identity Federation** |

> ⚠️ **마이그레이션 안내**: Databricks는 Hive Metastore에서 Unity Catalog로의 마이그레이션을 적극 권장하고 있습니다. 신규 Workspace는 기본적으로 Unity Catalog가 활성화됩니다. 기존 HMS 테이블은 `SYNC` 명령으로 UC로 마이그레이션할 수 있습니다.

---

## Unity Catalog란?

![Unity Catalog 오브젝트 모델 계층 구조](https://docs.databricks.com/aws/en/assets/images/object-model-40d730065eefed283b936a8664f1b247.png)

*출처: [Databricks 공식 문서](https://docs.databricks.com/aws/en/data-governance/unity-catalog/index.html)*


> 💡 **Unity Catalog**는 Databricks의 **통합 데이터 거버넌스 솔루션**입니다. 테이블, 파일, ML 모델, AI 에이전트 등 모든 데이터 자산을 하나의 카탈로그에서 관리하며, 접근 권한, 감사, 리니지를 통합적으로 제공합니다.

Unity Catalog는 Databricks 플랫폼의 **모든 데이터와 AI 자산을 하나의 통합 거버넌스 레이어**에서 관리하는 솔루션입니다. 데이터 엔지니어링, 데이터 웨어하우징, 머신러닝, AI 에이전트 개발 등 모든 워크로드에서 일관된 보안과 거버넌스를 적용할 수 있습니다.

---

## Unity Catalog의 핵심 가치

| 가치 | 설명 |
|------|------|
| **통합 관리** | 테이블, 파일, ML 모델, 함수를 하나의 시스템에서 관리합니다 |
| **세밀한 접근 제어** | 테이블, 행, 열 수준까지 권한을 관리합니다 |
| **감사 (Audit)** | 누가, 언제, 어떤 데이터에 접근했는지 추적합니다 |
| **리니지 (Lineage)** | 데이터가 어디서 왔고, 어디로 가는지 시각적으로 추적합니다 |
| **데이터 디스커버리** | 태그, 설명, 검색을 통해 필요한 데이터를 빠르게 찾을 수 있습니다 |
| **크로스 워크스페이스** | 여러 Workspace에서 동일한 카탈로그를 공유합니다 |
| **Delta Sharing** | 오픈 프로토콜로 외부 조직과 안전하게 데이터를 공유합니다 |

> 🆕 **ABAC (Attribute-Based Access Control)**: 태그 기반으로 접근 정책을 정의할 수 있는 기능이 Preview로 출시되었습니다. 예를 들어, "PII" 태그가 붙은 모든 컬럼에 마스킹을 자동 적용할 수 있습니다.

---

## 아키텍처 개요

Unity Catalog의 전체 아키텍처는 **Account → Metastore → Catalog → Schema → Object** 계층으로 구성됩니다.

![Unity Catalog 객체 모델 — 계층 구조](https://docs.databricks.com/en/_images/unity-catalog-object-hierarchy.png)

> 출처: [Databricks 공식 문서 — Unity Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)

### 계층 구조 상세

| 계층 | 설명 | 개수 제한 |
|------|------|-----------|
| **Account** | Databricks 최상위 계정입니다. 모든 리소스의 루트입니다 | 1개 |
| **Metastore** | 클라우드 리전별로 생성되며, 메타데이터를 저장합니다 | 리전별 1개 |
| **Catalog** | 최상위 네임스페이스 컨테이너입니다. 환경별/팀별로 분리합니다 | Metastore당 다수 |
| **Schema (Database)** | 관련 객체를 논리적으로 그룹화합니다 | Catalog당 다수 |
| **Object** | 실제 데이터 자산 (테이블, 뷰, 볼륨, 함수, 모델) | Schema당 다수 |

---

## 3-Level 네임스페이스

Unity Catalog는 모든 데이터 자산을 **`catalog.schema.object`** 3-Level 네임스페이스로 참조합니다.

```
Catalog (카탈로그)
  └── Schema (스키마)
       ├── Table (테이블)
       ├── View (뷰)
       ├── Volume (볼륨 - 파일)
       ├── Function (함수)
       └── Model (ML 모델)
```

```sql
-- 완전한 3-Level 네임스페이스로 테이블 참조
SELECT * FROM production.ecommerce.orders;

-- 기본 카탈로그와 스키마 설정
USE CATALOG production;
USE SCHEMA ecommerce;

-- 설정 후 간단하게 참조 가능
SELECT * FROM orders;
```

| 레벨 | 역할 | 예시 |
|------|------|------|
| **Catalog** | 최상위 컨테이너. 환경별, 팀별로 분리합니다 | `production`, `development`, `staging` |
| **Schema** | 관련 객체를 논리적으로 그룹화합니다 | `ecommerce`, `hr`, `finance` |
| **Object** | 실제 데이터 자산 (테이블, 뷰, 볼륨 등) | `orders`, `customers` |

### 카탈로그 설계 패턴

| 패턴 | 예시 카탈로그 |
|------|-------------|
| **환경별 분리** | `production`, `development`, `staging` |
| **팀별 분리** | `data_engineering`, `data_science`, `analytics` |
| **혼합 패턴 (권장)** | `prod_ecommerce`, `dev_ecommerce`, `prod_analytics` |

*출처: [Databricks Docs](https://docs.databricks.com)*

> 💡 **카탈로그 설계 모범 사례**: 환경(prod/dev/staging)과 도메인(ecommerce/finance)을 결합하는 혼합 패턴이 가장 유연합니다. 예를 들어, `prod_ecommerce`, `dev_ecommerce`처럼 명명하면 환경과 도메인이 한눈에 파악됩니다.

---

## Unity Catalog가 관리하는 자산 유형

Unity Catalog는 테이블뿐 아니라 다양한 유형의 데이터 자산을 통합 관리합니다.

| 자산 유형 | 설명 | 예시 |
|-----------|------|------|
| **Managed Table** | UC가 데이터와 메타데이터를 모두 관리합니다 | `CREATE TABLE orders (...)` |
| **External Table** | 메타데이터만 UC가 관리하고, 데이터는 외부 스토리지에 있습니다 | `CREATE TABLE ... LOCATION 's3://...'` |
| **View** | SQL 쿼리 기반의 가상 테이블입니다 | `CREATE VIEW daily_summary AS ...` |
| **Materialized View** | 미리 계산되어 저장된 뷰입니다 | `CREATE MATERIALIZED VIEW ...` |
| **Volume** | 비정형 파일(CSV, JSON, 이미지 등)을 관리합니다 | `CREATE VOLUME raw_files` |
| **Function** | SQL 또는 Python UDF입니다 | `CREATE FUNCTION calc_tax(...)` |
| **Model** | MLflow에 등록된 ML 모델입니다 | `models:/fraud_detector/1` |
| **Connection** | 외부 데이터 소스 연결 정보입니다 | MySQL, PostgreSQL, Salesforce 등 |
| **Share** | Delta Sharing으로 공유하는 데이터 묶음입니다 | `CREATE SHARE customer_share` |

### Managed Table vs External Table

```sql
-- Managed Table: UC가 데이터 수명주기를 완전히 관리
-- DROP TABLE 시 데이터도 함께 삭제됩니다
CREATE TABLE production.ecommerce.orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    total_amount DECIMAL(10, 2)
);

-- External Table: UC가 메타데이터만 관리
-- DROP TABLE 시 메타데이터만 삭제되고, 실제 데이터는 스토리지에 남습니다
CREATE TABLE production.ecommerce.legacy_orders
LOCATION 's3://my-bucket/legacy/orders'
AS SELECT * FROM hive_metastore.default.old_orders;
```

---

## 외부 위치와 스토리지 자격 증명

Unity Catalog에서 외부 데이터를 관리하려면 **Storage Credential**과 **External Location**이 필요합니다.

| 구성 요소 | 역할 | 연결 |
|-----------|------|------|
| Storage Credential (클라우드 인증 정보) | AWS: IAM Role / Azure: Storage Account / GCP: Service Account | → 클라우드 스토리지 |
| External Location (스토리지 경로) | Storage Credential을 사용하여 접근할 수 있는 경로 | Storage Credential → External Location |
| External Table | 메타데이터만 UC가 관리 | External Location → External Table |
| External Volume | 비정형 파일 관리 | External Location → External Volume |

*출처: [Databricks Docs](https://docs.databricks.com)*

| 개념 | 설명 | 예시 |
|------|------|------|
| **Storage Credential** | 클라우드 스토리지에 접근하기 위한 인증 정보입니다 | AWS IAM Role, Azure Managed Identity |
| **External Location** | Storage Credential을 사용하여 접근할 수 있는 스토리지 경로입니다 | `s3://my-bucket/data/` |

```sql
-- Storage Credential 생성 (관리자)
CREATE STORAGE CREDENTIAL my_aws_credential
WITH (
    AWS_IAM_ROLE.ROLE_ARN = 'arn:aws:iam::123456789012:role/databricks-role'
);

-- External Location 생성
CREATE EXTERNAL LOCATION my_data_location
URL 's3://my-bucket/data/'
WITH (STORAGE CREDENTIAL my_aws_credential);

-- External Location을 사용하는 External Table 생성
CREATE TABLE production.ecommerce.external_orders
LOCATION 's3://my-bucket/data/orders/';
```

---

## Identity Federation (계정 수준 사용자 관리)

Unity Catalog는 **Account 수준에서 사용자와 그룹을 관리**합니다. 이를 통해 여러 Workspace에서 일관된 권한 관리가 가능합니다.

| 개념 | 설명 |
|------|------|
| **Account-level User** | Databricks Account에 등록된 사용자입니다. 여러 Workspace에 할당할 수 있습니다 |
| **Account-level Group** | Account에서 생성한 그룹입니다. 권한 부여의 기본 단위입니다 |
| **SCIM Provisioning** | IdP(Okta, Azure AD 등)에서 사용자/그룹을 자동 동기화합니다 |
| **Workspace Assignment** | Account 사용자를 특정 Workspace에 할당합니다 |

```sql
-- 그룹에 카탈로그 접근 권한 부여
GRANT USE CATALOG ON CATALOG production TO `data_analysts`;
GRANT USE SCHEMA ON SCHEMA production.ecommerce TO `data_analysts`;
GRANT SELECT ON TABLE production.ecommerce.orders TO `data_analysts`;

-- 모든 테이블에 대한 읽기 권한 부여
GRANT SELECT ON SCHEMA production.ecommerce TO `data_analysts`;
```

---

## Metastore 설정 및 Workspace 연결

### Metastore 개념

**Metastore**는 Unity Catalog의 메타데이터를 저장하는 최상위 컨테이너입니다. 클라우드 리전별로 하나의 Metastore를 생성하고, 해당 리전의 Workspace들을 연결합니다.

| 구성 요소 | 설명 |
|-----------|------|
| **Metastore** | 메타데이터 저장소. 리전별 1개 생성을 권장합니다 |
| **Metastore Admin** | Metastore의 최고 관리자입니다 |
| **Root Storage** | Managed Table의 기본 저장 위치입니다 |
| **Workspace 연결** | 하나의 Workspace는 하나의 Metastore에만 연결됩니다 |

> 💡 **자동 Metastore 생성**: 최신 Databricks Account에서는 첫 번째 Workspace 생성 시 Metastore가 자동으로 생성되고 연결됩니다. 별도의 수동 설정이 필요하지 않습니다.

---

## Unity Catalog와 데이터 플랫폼의 관계

Unity Catalog는 Databricks 플랫폼의 모든 워크로드에서 핵심 역할을 합니다.

| 워크로드 | UC의 역할 |
|----------|-----------|
| **데이터 엔지니어링** | ETL 파이프라인의 소스/타겟 테이블 관리, 리니지 자동 추적 |
| **데이터 웨어하우징** | SQL Warehouse에서 UC 테이블 조회, 행/열 수준 보안 적용 |
| **머신러닝** | Feature Table, ML 모델을 UC에서 통합 관리 |
| **AI 에이전트** | Vector Search 인덱스, 에이전트 도구(함수)를 UC에서 관리 |
| **데이터 공유** | Delta Sharing으로 외부 조직과 안전하게 데이터 공유 |
| **BI/Analytics** | 대시보드에서 UC 테이블 직접 조회, 권한 자동 적용 |

---

## 실습: 카탈로그, 스키마, 테이블 생성

### 1단계: 카탈로그와 스키마 생성

```sql
-- 개발용 카탈로그 생성
CREATE CATALOG IF NOT EXISTS dev_ecommerce
COMMENT '이커머스 개발 환경 카탈로그';

-- 스키마 생성
CREATE SCHEMA IF NOT EXISTS dev_ecommerce.orders
COMMENT '주문 관련 데이터';

CREATE SCHEMA IF NOT EXISTS dev_ecommerce.customers
COMMENT '고객 관련 데이터';
```

### 2단계: 테이블 및 뷰 생성

```sql
-- 테이블 생성
CREATE TABLE dev_ecommerce.orders.transactions (
    transaction_id BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT,
    total_amount DECIMAL(10, 2),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    status STRING DEFAULT 'PENDING'
)
COMMENT '주문 트랜잭션 테이블'
TBLPROPERTIES ('quality' = 'gold');

-- 뷰 생성
CREATE VIEW dev_ecommerce.orders.daily_summary AS
SELECT
    DATE(order_date) AS order_day,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS total_revenue
FROM dev_ecommerce.orders.transactions
GROUP BY DATE(order_date);
```

### 3단계: Volume 생성 및 파일 관리

```sql
-- Managed Volume 생성
CREATE VOLUME dev_ecommerce.orders.raw_files
COMMENT '원본 CSV, JSON 파일 저장';

-- Volume에 파일 업로드 후 테이블로 로드
-- PUT 명령 또는 UI로 파일 업로드 후:
CREATE TABLE dev_ecommerce.orders.imported_data
AS SELECT * FROM read_files(
    '/Volumes/dev_ecommerce/orders/raw_files/sales.csv',
    format => 'csv',
    header => true
);
```

### 4단계: 권한 설정

```sql
-- 분석팀 그룹에 읽기 권한 부여
GRANT USE CATALOG ON CATALOG dev_ecommerce TO `analysts`;
GRANT USE SCHEMA ON SCHEMA dev_ecommerce.orders TO `analysts`;
GRANT SELECT ON SCHEMA dev_ecommerce.orders TO `analysts`;

-- 엔지니어 그룹에 쓰기 권한 부여
GRANT USE CATALOG ON CATALOG dev_ecommerce TO `engineers`;
GRANT ALL PRIVILEGES ON SCHEMA dev_ecommerce.orders TO `engineers`;

-- 권한 확인
SHOW GRANTS ON SCHEMA dev_ecommerce.orders;
```

---

## 심화: 멀티 리전 UC 통합 전략

### 왜 멀티 리전이 필요한가?

엔터프라이즈 환경에서는 **여러 리전에 Workspace가 분산**되어 있는 경우가 많습니다:
- **데이터 레지던시**: GDPR 등 규제로 EU 데이터는 EU 리전에 저장해야 합니다
- **지연시간**: 사용자가 여러 국가에 분산되어 있어 각 리전에 Workspace가 필요합니다
- **DR(재해 복구)**: 리전 장애에 대비하여 보조 리전을 운영합니다

### 현재 UC Metastore 구조

Unity Catalog의 Metastore는 **리전 단위**로 생성됩니다. 각 리전의 Workspace는 해당 리전의 Metastore에 연결됩니다.

```
Databricks Account
├── Metastore (us-east-1)
│   ├── Workspace A (프로덕션)
│   └── Workspace B (개발)
├── Metastore (eu-west-1)
│   └── Workspace C (EU 데이터)
└── Metastore (ap-northeast-2)
    └── Workspace D (한국)
```

### 멀티 리전 통합 전략

#### 전략 1: Delta Sharing으로 크로스 리전 데이터 공유

가장 **권장되는 방식**입니다. 각 리전의 Metastore는 독립적으로 유지하되, **Delta Sharing으로 리전 간 데이터를 공유**합니다.

```sql
-- us-east-1 Metastore에서 Share 생성
CREATE SHARE global_customer_data;
ALTER SHARE global_customer_data ADD TABLE prod.customers.gold_customer_360;

-- eu-west-1 Metastore에서 수신자로 등록하여 데이터 접근
CREATE CATALOG IF NOT EXISTS shared_us_data;
-- 관리자가 Account Console에서 Sharing 연결 설정
```

| 장점 | 단점 |
|------|------|
| 데이터 레지던시 유지 (데이터가 이동하지 않음) | 리전 간 쿼리 시 네트워크 지연 |
| 각 리전의 거버넌스 독립 운영 | Share 관리 오버헤드 |
| 네트워크 비용 예측 가능 | 실시간 동기화 아님 |

#### 전략 2: 단일 리전 Metastore + 멀티 리전 Storage

**하나의 Metastore**에 여러 리전의 Workspace를 연결하되, **External Location으로 각 리전의 스토리지를 참조**합니다.

```sql
-- 단일 Metastore (ap-northeast-2)에 여러 리전의 스토리지 연결
CREATE STORAGE CREDENTIAL us_east_cred
  WITH (AWS_IAM_ROLE 'arn:aws:iam::role/us-east-access');

CREATE EXTERNAL LOCATION us_east_data
  URL 's3://us-east-bucket/data/'
  WITH (STORAGE CREDENTIAL us_east_cred);

-- 한국 Workspace에서 미국 데이터 접근 가능
SELECT * FROM catalog.us_data.transactions LIMIT 10;
```

| 장점 | 단점 |
|------|------|
| 거버넌스 통합 (권한, 리니지 한 곳에서) | 크로스 리전 쿼리 시 데이터 전송 비용 |
| 관리 단순화 | 데이터 레지던시 위반 가능성 (메타데이터가 한 리전) |
| 통합 검색, 통합 리니지 | Metastore 리전 장애 시 전체 영향 |

#### 전략 3: 하이브리드 (권장)

```
[운영 원칙]
1. 데이터 레지던시가 필수인 리전 → 별도 Metastore
2. 같은 규제 권역 내 → 단일 Metastore 공유
3. 리전 간 데이터 교환 → Delta Sharing
4. 글로벌 거버넌스 정책 → Account 수준에서 관리
```

| 구성 요소 | 전략 |
|----------|------|
| **한국 + 일본** (아시아) | 하나의 Metastore (ap-northeast-2) 공유 |
| **EU** (GDPR) | 별도 Metastore (eu-west-1) |
| **미국** | 별도 Metastore (us-east-1) |
| **리전 간 공유** | Delta Sharing으로 필요한 테이블만 공유 |
| **글로벌 사용자** | Account 수준 SCIM으로 통합 관리 |

### Account-level 거버넌스

리전별 Metastore가 분산되어 있어도, **Account 수준**에서 다음을 통합 관리할 수 있습니다:

| 관리 영역 | Account 수준 | Metastore 수준 |
|----------|-------------|---------------|
| **사용자/그룹** | ✅ SCIM으로 통합 관리 | Metastore 멤버십으로 접근 제어 |
| **Service Principal** | ✅ 통합 관리 | Metastore별 권한 부여 |
| **보안 정책** | ✅ IP Access List, SSO | Metastore별 접근 제어 |
| **빌링** | ✅ 통합 비용 추적 | 리전별 사용량 분석 |
| **감사** | ✅ Account 감사 로그 | Metastore별 상세 로그 |

> 💡 **핵심**: 멀티 리전 환경에서는 **"데이터는 분산, 거버넌스는 통합"** 이 원칙입니다. 데이터가 물리적으로 어디에 있든, Account 수준에서 사용자 관리와 보안 정책을 일관되게 적용하고, Delta Sharing으로 필요한 데이터만 크로스 리전 공유합니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Unity Catalog** | Databricks의 통합 데이터 거버넌스 솔루션입니다 |
| **3-Level 네임스페이스** | `catalog.schema.object` 형태로 모든 자산을 참조합니다 |
| **Metastore** | 리전별 메타데이터 저장소이며, 여러 Workspace가 공유합니다 |
| **Storage Credential** | 클라우드 스토리지 접근을 위한 인증 정보입니다 |
| **External Location** | 외부 스토리지 경로를 UC에서 관리하는 방법입니다 |
| **Identity Federation** | Account 수준에서 사용자/그룹을 통합 관리합니다 |
| **Managed vs External** | Managed는 UC가 데이터까지, External은 메타데이터만 관리합니다 |

---

## 참고 링크

- [Databricks: Unity Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)
- [Databricks: Unity Catalog Best Practices](https://docs.databricks.com/aws/en/data-governance/unity-catalog/best-practices.html)
- [Databricks: Manage External Locations](https://docs.databricks.com/aws/en/connect/unity-catalog/external-locations.html)
- [Databricks: Storage Credentials](https://docs.databricks.com/aws/en/connect/unity-catalog/storage-credentials.html)
- [Azure Databricks: Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/)
- [Databricks Blog: Unity Catalog Best Practices](https://www.databricks.com/blog/unity-catalog-best-practices)
