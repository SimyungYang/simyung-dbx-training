# 외부 로케이션 (External Location)

## 외부 로케이션이란?

Unity Catalog에서 **외부 로케이션(External Location)** 은 클라우드 스토리지의 특정 경로를 Unity Catalog에 등록하여 **거버넌스 하에 관리** 할 수 있게 해주는 객체입니다. 외부 테이블(External Table)이나 외부 볼륨(External Volume)을 생성하려면, 해당 데이터가 저장된 클라우드 경로가 외부 로케이션으로 등록되어 있어야 합니다.

> 💡 **왜 필요한가요?** 기업은 이미 S3, ADLS, GCS 등 클라우드 스토리지에 대량의 데이터를 보유하고 있습니다. 이 데이터를 Unity Catalog로 이동하지 않고도, 경로를 등록하여 접근 권한과 감사를 적용할 수 있습니다.

---

## 외부 로케이션과 Storage Credential의 관계

외부 로케이션은 단독으로 동작하지 않습니다. 클라우드 스토리지에 접근하려면 **인증 정보(Storage Credential)** 가 필요합니다.

| Storage Credential | External Location | 오브젝트 |
|-------------------|------------------|---------|
| 인증 정보 | A (`s3://my-bucket/data/`) | External Table 1, External Volume 1 |
| | B (`s3://my-bucket/logs/`) | External Table 2 |

| 구성 요소 | 역할 | 비유 |
|-----------|------|------|
| **Storage Credential** | 클라우드 스토리지에 접근할 수 있는 "열쇠" | 건물 마스터키 |
| **External Location** | 특정 경로를 Unity Catalog에 등록한 "주소" | 건물 내 특정 사무실 |
| **External Table/Volume** | 외부 로케이션 안에 존재하는 데이터 객체 | 사무실 안의 문서 |

> 💡 **Storage Credential** 에 대한 자세한 내용은 [스토리지 자격 증명](./storage-credentials.md) 문서를 참고하세요.

---

## 외부 로케이션 생성

### CREATE EXTERNAL LOCATION 문법

```sql
CREATE EXTERNAL LOCATION [IF NOT EXISTS] <location_name>
  URL '<cloud_storage_url>'
  WITH (STORAGE CREDENTIAL <credential_name>)
  [COMMENT '<설명>'];
```

### 클라우드별 URL 형식

| 클라우드 | URL 형식 | 예시 |
|---------|---------|------|
| **AWS S3** | `s3://<bucket>/<path>` | `s3://my-company-data/production/` |
| **Azure ADLS** | `abfss://<container>@<account>.dfs.core.windows.net/<path>` | `abfss://data@myaccount.dfs.core.windows.net/prod/` |
| **GCP GCS** | `gs://<bucket>/<path>` | `gs://my-company-data/production/` |

### 실전 예시

```sql
-- AWS S3 외부 로케이션 생성
CREATE EXTERNAL LOCATION production_data
  URL 's3://my-company-datalake/production/'
  WITH (STORAGE CREDENTIAL aws_s3_credential)
  COMMENT '프로덕션 데이터 레이크 경로';

-- Azure ADLS 외부 로케이션 생성
CREATE EXTERNAL LOCATION production_data
  URL 'abfss://datalake@mycompany.dfs.core.windows.net/production/'
  WITH (STORAGE CREDENTIAL azure_managed_identity_cred)
  COMMENT '프로덕션 데이터 레이크 경로';
```

---

## 외부 로케이션에서 외부 테이블 생성

외부 로케이션이 등록되면, 해당 경로 아래에 외부 테이블을 생성할 수 있습니다.

```sql
-- 외부 테이블 생성 (외부 로케이션 경로 안에 데이터 저장)
CREATE TABLE production.ecommerce.orders_external
  (order_id BIGINT, customer_id BIGINT, amount DECIMAL(10,2), order_date DATE)
  LOCATION 's3://my-company-datalake/production/ecommerce/orders/';

-- 외부 볼륨 생성
CREATE EXTERNAL VOLUME production.ecommerce.raw_files
  LOCATION 's3://my-company-datalake/production/ecommerce/raw/';
```

> 주의: `LOCATION`에 지정하는 경로는 반드시 등록된 외부 로케이션 하위 경로여야 합니다. 등록되지 않은 경로를 지정하면 오류가 발생합니다.

---

## 권한 관리

### 외부 로케이션에 대한 권한

| 권한 | 설명 |
|------|------|
| **CREATE EXTERNAL TABLE** | 해당 로케이션에 외부 테이블을 생성할 수 있습니다 |
| **CREATE EXTERNAL VOLUME** | 해당 로케이션에 외부 볼륨을 생성할 수 있습니다 |
| **BROWSE** | 로케이션의 파일 목록을 조회할 수 있습니다 |
| **READ FILES** | 로케이션의 파일을 읽을 수 있습니다 |
| **WRITE FILES** | 로케이션에 파일을 쓸 수 있습니다 |
| **MANAGE** | 로케이션을 수정/삭제할 수 있습니다 |

### 권한 부여 예시

```sql
-- 데이터 엔지니어 그룹에 외부 테이블 생성 권한 부여
GRANT CREATE EXTERNAL TABLE ON EXTERNAL LOCATION production_data
  TO `data_engineers`;

-- 분석가 그룹에 파일 읽기 권한 부여
GRANT READ FILES ON EXTERNAL LOCATION production_data
  TO `analysts`;

-- 현재 권한 확인
SHOW GRANTS ON EXTERNAL LOCATION production_data;
```

---

## 외부 로케이션 관리

### 정보 조회

```sql
-- 모든 외부 로케이션 목록 조회
SHOW EXTERNAL LOCATIONS;

-- 특정 외부 로케이션 상세 정보
DESCRIBE EXTERNAL LOCATION production_data;
```

### 수정 및 삭제

```sql
-- URL 변경
ALTER EXTERNAL LOCATION production_data
  SET URL 's3://my-company-datalake/production-v2/';

-- Storage Credential 변경
ALTER EXTERNAL LOCATION production_data
  SET STORAGE CREDENTIAL new_credential;

-- 코멘트 수정
ALTER EXTERNAL LOCATION production_data
  SET COMMENT '프로덕션 데이터 레이크 (2025년 마이그레이션 완료)';

-- 삭제 (해당 로케이션을 참조하는 외부 테이블이 없어야 합니다)
DROP EXTERNAL LOCATION production_data;
```

---

## 외부 로케이션 설계 모범 사례

| 원칙 | 설명 |
|------|------|
| **경로 계층화** | 버킷 루트가 아닌 의미 있는 하위 경로 단위로 등록합니다 (예: `s3://bucket/production/`) |
| **환경 분리** | `production/`, `staging/`, `dev/` 경로를 별도 외부 로케이션으로 등록합니다 |
| **최소 권한** | 외부 테이블 생성 권한을 데이터 엔지니어에게만 부여합니다 |
| **네이밍 규칙** | `{환경}_{도메인}` 형식으로 이름을 지정합니다 (예: `prod_ecommerce`) |
| **겹침 방지** | 하나의 클라우드 경로가 여러 외부 로케이션에 겹치지 않도록 관리합니다 |

---

## 연결 확인 (Validate)

외부 로케이션이 올바르게 설정되었는지 검증할 수 있습니다.

```sql
-- 외부 로케이션 연결 테스트
VALIDATE EXTERNAL LOCATION production_data;
```

이 명령은 Storage Credential이 해당 경로에 대해 읽기/쓰기 권한을 갖고 있는지 확인합니다. 설정 직후 반드시 실행하여 연결 상태를 점검하는 것을 권장합니다.

---

## External Location 설계 패턴 — 세분화 전략

엔터프라이즈 환경에서 External Location을 어떻게 설계하느냐에 따라 거버넌스 수준, 관리 복잡도, 성능이 달라집니다.

### 세분화 수준별 비교

| 전략 | 예시 | 장점 | 단점 |
|------|------|------|------|
| **버킷 레벨** | `s3://company-datalake/` | 관리 간편 | 세분화된 권한 제어 불가 |
| **환경 레벨** | `s3://company-datalake/prod/`, `.../dev/` | 환경 격리 | 도메인 간 권한 분리 부족 |
| **도메인 레벨** | `.../prod/finance/`, `.../prod/marketing/` | 도메인별 권한 관리 | Location 수 증가 |
| **테이블 레벨** | `.../prod/finance/orders/` | 최대 세분화 | Location 수 폭발, 관리 복잡 |

### 권장 설계 패턴

| Storage Credential | External Location | URL | GRANT |
|-------------------|------------------|-----|-------|
| **aws_prod_credential**(IAM Role) | prod_finance | `s3://datalake/production/finance/` | finance_team (CREATE EXTERNAL TABLE, READ FILES) |
| | prod_marketing | `s3://datalake/production/marketing/` | marketing_team (CREATE EXTERNAL TABLE, READ FILES) |
| | prod_shared | `s3://datalake/production/shared/` | all_data_engineers (READ FILES) |
| | staging_all
        URL: s3://datalake/staging/
        → GRANT: data_engineers (ALL PRIVILEGES)

Storage Credential: aws_dev_credential (별도 IAM Role)
| | dev_sandbox |
        URL: s3://datalake-dev/
        → GRANT: all_developers (ALL PRIVILEGES)
```

> 💡 **SA 권장**: 대부분의 고객에게는 **환경 + 도메인 레벨** 조합을 권장합니다. 너무 세분화하면 관리 부담이 크고, 너무 넓으면 거버넌스가 약해집니다. 초기에는 넓게 시작하고, 보안 요구사항이 구체화되면 점진적으로 세분화하는 것이 좋습니다.

---

## 중첩 Location과 우선순위 규칙

하나의 클라우드 경로가 여러 External Location에 해당할 수 있는 "중첩(Overlap)" 상황이 발생할 수 있습니다. Unity Catalog는 이를 **가장 구체적인(가장 긴) 경로 우선** 규칙으로 처리합니다.

### 중첩 규칙 예시

```sql
-- Location A: s3://bucket/data/
-- Location B: s3://bucket/data/finance/
-- Location C: s3://bucket/data/finance/reports/

-- 외부 테이블 생성 시:
CREATE TABLE t1 LOCATION 's3://bucket/data/finance/reports/q1/';
-- → Location C가 적용됨 (가장 구체적인 경로)

CREATE TABLE t2 LOCATION 's3://bucket/data/finance/transactions/';
-- → Location B가 적용됨

CREATE TABLE t3 LOCATION 's3://bucket/data/marketing/';
-- → Location A가 적용됨
```

### 중첩 시 주의사항

| 상황 | 동작 | 주의 |
|------|------|------|
| 하위 Location에 더 좁은 권한 | 하위 Location 권한이 우선 | 상위 Location에 권한이 있어도 하위에서 차단 가능 |
| 동일 경로에 두 Location | **생성 불가**(오류 발생) | 정확히 같은 URL은 중복 등록 불가 |
| 하위 Location 삭제 | 상위 Location 권한으로 폴백 | 의도하지 않은 권한 확대에 주의 |

> ⚠️ **Gotcha**: Location B(`s3://bucket/data/finance/`)를 삭제하면, 해당 경로의 테이블들은 Location A(`s3://bucket/data/`)의 권한을 상속받게 됩니다. Location A에 더 넓은 권한이 부여되어 있다면, 의도하지 않은 접근이 가능해질 수 있습니다. Location 삭제 전에 반드시 영향 범위를 확인하세요.

---

## 크로스 계정 접근 — AWS IAM Role Chaining

대규모 기업에서는 여러 AWS 계정에 데이터가 분산되어 있는 경우가 많습니다. Unity Catalog에서 크로스 계정 데이터에 접근하려면 **IAM Role Chaining** 을 설정해야 합니다.

### 아키텍처

| 계정 | 구성 요소 | 역할/정책 |
|------|----------|---------|
| **Account A**(Databricks) | Unity Catalog Metastore | - |
| | Storage Credential: cross_account_role | IAM Role: `arn:aws:iam::ACCOUNT_A:role/uc-access-role` |
| | Trust Policy | Databricks Control Plane이 AssumeRole 가능 |
| | Role Policy | `sts:AssumeRole → arn:aws:iam::ACCOUNT_B:role/data-reader-role` |
| **Account B**(데이터) | S3 버킷: `s3://account-b-datalake/` | Bucket Policy: data-reader-role만 접근 |
| | IAM Role: data-reader-role | Trust Policy: ACCOUNT_A의 uc-access-role이 AssumeRole 가능 |

### 설정 절차

```sql
-- 1. Storage Credential 생성 (Databricks 계정의 IAM Role)
CREATE STORAGE CREDENTIAL cross_account_cred
  WITH (AWS_IAM_ROLE = 'arn:aws:iam::111111111111:role/uc-access-role');

-- 2. External Location 생성 (다른 계정의 S3 경로)
CREATE EXTERNAL LOCATION account_b_data
  URL 's3://account-b-datalake/shared-data/'
  WITH (STORAGE CREDENTIAL cross_account_cred);

-- 3. 연결 검증
VALIDATE EXTERNAL LOCATION account_b_data;
```

> ⚠️ **보안 주의**: Role Chaining에서 AssumeRole의 세션 기간은 최대 1시간입니다. 대용량 데이터 처리 시 세션 만료로 실패할 수 있으므로, 장시간 작업에는 직접 IAM Role을 사용하거나 세션 갱신 메커니즘을 고려하세요.

---

## 성능 고려사항 — S3 Prefix 최적화

External Location의 경로 설계는 S3의 성능 특성과도 관련이 있습니다.

### S3 Prefix 파티셔닝과 성능

AWS S3는 **prefix 단위로 초당 5,500 GET / 3,500 PUT 요청** 을 처리합니다. 하나의 prefix에 대량의 파일이 집중되면 스로틀링이 발생할 수 있습니다.

| 경로 설계 | 파일 수 | 성능 |
|-----------|--------|------|
| `s3://bucket/all_data/` (단일 prefix) | 수백만 | 스로틀링 위험 |
| `s3://bucket/data/year=2025/month=01/` | 수천~수만 | 양호 |
| `s3://bucket/data/{hash_prefix}/year=2025/` | 수백~수천 | 최적 |

### External Location과 파일 리스팅 성능

Auto Loader의 디렉토리 리스팅 모드를 사용할 때, External Location의 경로가 너무 넓으면 파일 리스팅에 시간이 많이 소요됩니다.

```
비효율적:
  External Location: s3://bucket/         (수백만 파일 포함)
  Auto Loader가 전체 버킷을 스캔 → 수분 소요

효율적:
  External Location: s3://bucket/events/  (수만 파일)
  Auto Loader가 특정 경로만 스캔 → 수초 소요
```

> 💡 **실무 팁**: Auto Loader를 사용하는 경우, External Location을 가능한 한 **데이터 소스 경로와 가깝게** 설정하세요. 그리고 Auto Loader의 파일 알림 모드(Notification Mode)를 사용하면 디렉토리 리스팅 자체를 건너뛸 수 있어 대규모 경로에서도 성능 문제가 없습니다.

---

## 와일드카드 경로와 제한사항

Unity Catalog의 External Location은 **와일드카드를 지원하지 않습니다**. 각 Location은 정확한 경로 prefix여야 합니다.

```sql
-- ❌ 불가: 와일드카드
CREATE EXTERNAL LOCATION test
  URL 's3://bucket/data/*/events/';

-- ✅ 가능: 정확한 prefix
CREATE EXTERNAL LOCATION test
  URL 's3://bucket/data/region1/events/';
```

여러 하위 경로에 같은 권한을 적용하려면, **상위 경로에 하나의 Location을 생성** 하거나, 각 경로에 개별 Location을 생성해야 합니다.

---

## 마이그레이션 패턴 — HMS에서 UC로

기존 Hive Metastore(HMS) 기반 외부 테이블을 Unity Catalog로 마이그레이션할 때, External Location 설정이 핵심입니다.

### 마이그레이션 절차

```
1. 기존 HMS 외부 테이블의 LOCATION 경로 파악
   DESCRIBE FORMATTED hive_metastore.schema.table;

2. 해당 경로를 포함하는 External Location 생성
   CREATE EXTERNAL LOCATION ...;

3. UC에 외부 테이블 재생성 (동일 LOCATION 지정)
   CREATE TABLE catalog.schema.table LOCATION 's3://...';

4. HMS 테이블 삭제 (데이터는 유지)
   DROP TABLE hive_metastore.schema.table;
```

> 💡 **주의**: HMS의 외부 테이블과 UC의 외부 테이블이 **동일한 S3 경로** 를 참조할 수 있습니다. 이 경우 두 테이블 모두에서 동일 데이터를 읽을 수 있지만, 권한 모델이 다르므로 마이그레이션 완료 후 HMS 테이블을 삭제하는 것을 권장합니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **External Location** | 클라우드 스토리지 경로를 UC에 등록하여 거버넌스를 적용합니다 |
| **Storage Credential 연결** | 외부 로케이션은 반드시 하나의 Storage Credential과 연결됩니다 |
| **경로 기반 접근 제어** | 로케이션 단위로 외부 테이블/볼륨 생성 권한을 관리합니다 |
| **중첩 우선순위** | 가장 구체적인(가장 긴) 경로의 Location이 우선 적용됩니다 |
| **크로스 계정** | AWS IAM Role Chaining으로 다른 계정의 S3에 접근할 수 있습니다 |
| **세분화 전략** | 환경 + 도메인 레벨의 Location 설계를 권장합니다 |
| **S3 성능** | Location 경로가 너무 넓으면 파일 리스팅과 S3 API 성능에 영향합니다 |
| **VALIDATE** | 생성 후 연결 상태를 반드시 검증합니다 |

---

## 참고 링크

- [Databricks: External locations](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-external-locations.html)
- [Databricks: CREATE EXTERNAL LOCATION](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-location.html)
- [Databricks: Manage external locations](https://docs.databricks.com/aws/en/connect/unity-catalog/external-locations.html)
- [Azure Databricks: External locations](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/external-locations)
