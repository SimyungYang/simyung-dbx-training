# 스토리지 자격 증명 (Storage Credential)

## 스토리지 자격 증명이란?

**스토리지 자격 증명(Storage Credential)** 은 Unity Catalog가 클라우드 스토리지(S3, ADLS, GCS)에 접근할 때 사용하는 **인증 정보 객체**입니다. 외부 로케이션(External Location)을 생성하려면 반드시 스토리지 자격 증명이 필요합니다.

> 💡 **비유**: 스토리지 자격 증명은 "건물 마스터키"와 같습니다. 이 키가 있어야 외부 로케이션(건물 내 특정 사무실)에 접근할 수 있습니다. Databricks는 사용자의 클라우드 자격 증명 대신, Unity Catalog가 관리하는 자격 증명으로 스토리지에 접근합니다.

---

## 클라우드별 인증 방식

각 클라우드 환경마다 지원하는 인증 방식이 다릅니다.

### AWS: IAM Role

AWS 환경에서는 **IAM Role**을 사용하여 S3 버킷에 접근합니다. Databricks가 이 역할을 **Assume(위임)**하여 스토리지에 접근합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-datalake",
        "arn:aws:s3:::my-company-datalake/*"
      ]
    }
  ]
}
```

### Azure: Managed Identity / Service Principal

Azure 환경에서는 **Managed Identity** 또는 **Service Principal**을 사용합니다.

| 방식 | 설명 | 권장 여부 |
|------|------|----------|
| **Managed Identity** | Azure가 자동으로 관리하는 ID. 자격 증명 교체 불필요 | 권장 |
| **Service Principal** | Client ID + Secret 기반 인증 | 레거시 환경에서 사용 |

### GCP: Service Account

GCP 환경에서는 **Service Account**를 사용하여 GCS 버킷에 접근합니다.

---

## Storage Credential 생성

### SQL 문법

```sql
-- AWS: IAM Role 기반 Storage Credential 생성
CREATE STORAGE CREDENTIAL [IF NOT EXISTS] <credential_name>
  WITH (
    AWS_IAM_ROLE.ROLE_ARN = 'arn:aws:iam::123456789012:role/databricks-s3-access'
  )
  [COMMENT '<설명>'];

-- Azure: Managed Identity 기반 Storage Credential 생성
CREATE STORAGE CREDENTIAL [IF NOT EXISTS] <credential_name>
  WITH (
    AZURE_MANAGED_IDENTITY.ACCESS_CONNECTOR_ID =
      '/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Databricks/accessConnectors/<connector>'
  )
  [COMMENT '<설명>'];
```

### 실전 예시

```sql
-- AWS 환경
CREATE STORAGE CREDENTIAL aws_s3_credential
  WITH (
    AWS_IAM_ROLE.ROLE_ARN = 'arn:aws:iam::123456789012:role/uc-external-access'
  )
  COMMENT 'S3 데이터 레이크 접근용 IAM Role';

-- Azure 환경
CREATE STORAGE CREDENTIAL azure_adls_credential
  WITH (
    AZURE_MANAGED_IDENTITY.ACCESS_CONNECTOR_ID =
      '/subscriptions/abc-123/resourceGroups/data-rg/providers/Microsoft.Databricks/accessConnectors/uc-connector'
  )
  COMMENT 'ADLS Gen2 접근용 Managed Identity';
```

---

## Storage Credential과 External Location 연결

Storage Credential을 생성한 후, 이를 사용하여 외부 로케이션을 등록합니다.

```sql
-- 1. Storage Credential 생성
CREATE STORAGE CREDENTIAL prod_s3_cred
  WITH (AWS_IAM_ROLE.ROLE_ARN = 'arn:aws:iam::123456789012:role/prod-access');

-- 2. 외부 로케이션 등록 (Storage Credential 참조)
CREATE EXTERNAL LOCATION prod_data
  URL 's3://my-company-datalake/production/'
  WITH (STORAGE CREDENTIAL prod_s3_cred);

-- 3. 연결 검증
VALIDATE EXTERNAL LOCATION prod_data;
```

---

## 권한 모델

### Storage Credential 권한

| 권한 | 설명 |
|------|------|
| **CREATE EXTERNAL LOCATION** | 이 Credential을 사용하여 외부 로케이션을 생성할 수 있습니다 |
| **MANAGE** | Credential을 수정/삭제할 수 있습니다 |
| **READ FILES** | Credential을 통해 파일을 읽을 수 있습니다 |
| **WRITE FILES** | Credential을 통해 파일을 쓸 수 있습니다 |

### 권한 부여 예시

```sql
-- 데이터 엔지니어에게 외부 로케이션 생성 권한 부여
GRANT CREATE EXTERNAL LOCATION ON STORAGE CREDENTIAL aws_s3_credential
  TO `data_engineers`;

-- 현재 권한 확인
SHOW GRANTS ON STORAGE CREDENTIAL aws_s3_credential;
```

---

## Storage Credential 관리

### 조회

```sql
-- 모든 Storage Credential 목록
SHOW STORAGE CREDENTIALS;

-- 특정 Credential 상세 정보
DESCRIBE STORAGE CREDENTIAL aws_s3_credential;
```

### 수정 및 삭제

```sql
-- IAM Role ARN 변경
ALTER STORAGE CREDENTIAL aws_s3_credential
  SET AWS_IAM_ROLE.ROLE_ARN = 'arn:aws:iam::123456789012:role/new-role';

-- 코멘트 수정
ALTER STORAGE CREDENTIAL aws_s3_credential
  SET COMMENT '프로덕션 S3 접근용 (2025년 갱신)';

-- 소유권 이전
ALTER STORAGE CREDENTIAL aws_s3_credential
  SET OWNER TO `platform_admins`;

-- 삭제 (참조하는 External Location이 없어야 합니다)
DROP STORAGE CREDENTIAL aws_s3_credential;
```

---

## AWS IAM Role 신뢰 정책 설정

AWS에서 Storage Credential을 사용하려면, IAM Role의 **신뢰 정책(Trust Policy)** 에 Databricks를 추가해야 합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::414351767826:role/unity-catalog-prod-UCMasterRole-abc123"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<external-id-from-databricks>"
        }
      }
    }
  ]
}
```

> `<external-id-from-databricks>`는 Storage Credential 생성 시 Databricks가 제공하는 고유 식별자입니다. 이 값을 IAM Role의 신뢰 정책에 추가해야 안전한 위임이 가능합니다.

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **최소 권한 IAM 정책** | S3 버킷 전체가 아닌, 필요한 경로에 대해서만 접근을 허용합니다 |
| **Managed Identity 우선** | Azure에서는 Secret 기반 SP 대신 Managed Identity를 사용합니다 |
| **Credential 재사용** | 동일 버킷의 여러 경로에 하나의 Credential을 공유할 수 있습니다 |
| **네이밍 규칙** | `{클라우드}_{용도}_{환경}` 형식으로 이름을 지정합니다 |
| **정기 검증** | `VALIDATE` 명령으로 자격 증명의 유효성을 주기적으로 확인합니다 |
| **소유권 관리** | 플랫폼 관리자 그룹에 소유권을 할당합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Storage Credential** | 클라우드 스토리지 접근을 위한 인증 정보 객체입니다 |
| **AWS IAM Role** | STS AssumeRole을 통해 S3에 접근합니다 |
| **Azure Managed Identity** | 자동 관리되는 ID로 ADLS에 접근합니다 |
| **External Location 연결** | Credential + 경로 = 외부 로케이션이 됩니다 |
| **권한 분리** | Credential 관리자와 로케이션 사용자를 분리합니다 |

---

## 참고 링크

- [Databricks: Storage credentials](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-storage-credentials.html)
- [Databricks: CREATE STORAGE CREDENTIAL](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-storage-credential.html)
- [Databricks: Manage storage credentials](https://docs.databricks.com/aws/en/connect/unity-catalog/storage-credentials.html)
- [Azure Databricks: Storage credentials](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/storage-credentials)
