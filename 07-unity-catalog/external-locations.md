# 외부 로케이션 (External Location)

## 외부 로케이션이란?

Unity Catalog에서 **외부 로케이션(External Location)** 은 클라우드 스토리지의 특정 경로를 Unity Catalog에 등록하여 **거버넌스 하에 관리**할 수 있게 해주는 객체입니다. 외부 테이블(External Table)이나 외부 볼륨(External Volume)을 생성하려면, 해당 데이터가 저장된 클라우드 경로가 외부 로케이션으로 등록되어 있어야 합니다.

> 💡 **왜 필요한가요?** 기업은 이미 S3, ADLS, GCS 등 클라우드 스토리지에 대량의 데이터를 보유하고 있습니다. 이 데이터를 Unity Catalog로 이동하지 않고도, 경로를 등록하여 접근 권한과 감사를 적용할 수 있습니다.

---

## 외부 로케이션과 Storage Credential의 관계

외부 로케이션은 단독으로 동작하지 않습니다. 클라우드 스토리지에 접근하려면 **인증 정보(Storage Credential)** 가 필요합니다.

```
Storage Credential (인증 정보)
  └── External Location A (s3://my-bucket/data/)
        ├── External Table 1
        └── External Volume 1
  └── External Location B (s3://my-bucket/logs/)
        └── External Table 2
```

| 구성 요소 | 역할 | 비유 |
|-----------|------|------|
| **Storage Credential** | 클라우드 스토리지에 접근할 수 있는 "열쇠" | 건물 마스터키 |
| **External Location** | 특정 경로를 Unity Catalog에 등록한 "주소" | 건물 내 특정 사무실 |
| **External Table/Volume** | 외부 로케이션 안에 존재하는 데이터 객체 | 사무실 안의 문서 |

> 💡 **Storage Credential**에 대한 자세한 내용은 [스토리지 자격 증명](./storage-credentials.md) 문서를 참고하세요.

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

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **External Location** | 클라우드 스토리지 경로를 UC에 등록하여 거버넌스를 적용합니다 |
| **Storage Credential 연결** | 외부 로케이션은 반드시 하나의 Storage Credential과 연결됩니다 |
| **경로 기반 접근 제어** | 로케이션 단위로 외부 테이블/볼륨 생성 권한을 관리합니다 |
| **VALIDATE** | 생성 후 연결 상태를 반드시 검증합니다 |

---

## 참고 링크

- [Databricks: External locations](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-external-locations.html)
- [Databricks: CREATE EXTERNAL LOCATION](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-location.html)
- [Databricks: Manage external locations](https://docs.databricks.com/aws/en/connect/unity-catalog/external-locations.html)
- [Azure Databricks: External locations](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/external-locations)
