# 스토리지 자격 증명 (Storage Credential)

## 스토리지 자격 증명이란?

**스토리지 자격 증명(Storage Credential)** 은 Unity Catalog가 클라우드 스토리지(S3, ADLS, GCS)에 접근할 때 사용하는 **인증 정보 객체** 입니다. 외부 로케이션(External Location)을 생성하려면 반드시 스토리지 자격 증명이 필요합니다.

> 💡 **비유**: 스토리지 자격 증명은 "건물 마스터키"와 같습니다. 이 키가 있어야 외부 로케이션(건물 내 특정 사무실)에 접근할 수 있습니다. Databricks는 사용자의 클라우드 자격 증명 대신, Unity Catalog가 관리하는 자격 증명으로 스토리지에 접근합니다.

---

## 클라우드별 인증 방식

각 클라우드 환경마다 지원하는 인증 방식이 다릅니다.

### AWS: IAM Role

AWS 환경에서는 **IAM Role** 을 사용하여 S3 버킷에 접근합니다. Databricks가 이 역할을 **Assume(위임)** 하여 스토리지에 접근합니다.

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

Azure 환경에서는 **Managed Identity** 또는 **Service Principal** 을 사용합니다.

| 방식 | 설명 | 권장 여부 |
|------|------|----------|
| **Managed Identity**| Azure가 자동으로 관리하는 ID. 자격 증명 교체 불필요 | 권장 |
| **Service Principal**| Client ID + Secret 기반 인증 | 레거시 환경에서 사용 |

### GCP: Service Account

GCP 환경에서는 **Service Account** 를 사용하여 GCS 버킷에 접근합니다.

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
| **CREATE EXTERNAL LOCATION**| 이 Credential을 사용하여 외부 로케이션을 생성할 수 있습니다 |
| **MANAGE**| Credential을 수정/삭제할 수 있습니다 |
| **READ FILES**| Credential을 통해 파일을 읽을 수 있습니다 |
| **WRITE FILES**| Credential을 통해 파일을 쓸 수 있습니다 |

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
| **최소 권한 IAM 정책**| S3 버킷 전체가 아닌, 필요한 경로에 대해서만 접근을 허용합니다 |
| **Managed Identity 우선**| Azure에서는 Secret 기반 SP 대신 Managed Identity를 사용합니다 |
| **Credential 재사용**| 동일 버킷의 여러 경로에 하나의 Credential을 공유할 수 있습니다 |
| **네이밍 규칙**| `{클라우드}_{용도}_{환경}` 형식으로 이름을 지정합니다 |
| **정기 검증**| `VALIDATE` 명령으로 자격 증명의 유효성을 주기적으로 확인합니다 |
| **소유권 관리**| 플랫폼 관리자 그룹에 소유권을 할당합니다 |

---

## 실전 인사이트: IAM Role을 하나만 만들어서 모든 버킷에 접근하게 한 실수

초기 Databricks 도입 시 가장 흔한 실수가 "** 편의를 위해 하나의 IAM Role에 모든 S3 버킷 접근 권한을 몰아주는 것**"입니다.

### 사고 사례

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

한 고객이 위와 같은 IAM 정책으로 Storage Credential을 생성했습니다. 처음에는 "어차피 Unity Catalog가 External Location 단위로 접근을 제어하니까 IAM은 넓게 열어도 괜찮다"고 생각했습니다. 그런데 문제가 발생한 상황은 다음과 같습니다:

| 문제 | 설명 |
|------|------|
| **보안 감사 실패**| AWS Well-Architected Review에서 "최소 권한 원칙 위반" 지적 |
| **사고 시 피해 범위 확대**| Credential이 노출되면 모든 버킷의 데이터가 위험 |
| **규제 위반** | PCI-DSS 환경에서 결제 데이터 버킷과 일반 버킷의 접근 경로가 분리되지 않음 |

### 올바른 설계: 용도별 Credential 분리

```
[권장 구조]

Storage Credential 1: prod-analytics-cred
  → IAM Role: s3://company-analytics-prod/* (읽기 전용)
  → External Location: /analytics/reports/, /analytics/dashboards/

Storage Credential 2: prod-ml-cred
  → IAM Role: s3://company-ml-prod/* (읽기/쓰기)
  → External Location: /ml/features/, /ml/models/

Storage Credential 3: prod-raw-cred
  → IAM Role: s3://company-raw-data/* (쓰기 전용, 읽기는 별도)
  → External Location: /raw/ingestion/
```

> 💡 **원칙**: Credential은 **용도(analytics, ML, raw data)와 환경(dev, staging, prod)** 별로 분리하세요. 하나의 Credential이 접근할 수 있는 범위가 좁을수록, 사고 시 피해 범위도 좁아집니다.

---

## 실전 인사이트: 크로스 계정 접근의 실전 구성

대규모 조직에서는 AWS 계정이 여러 개로 분리되어 있는 경우가 대부분입니다. Databricks가 있는 계정(Account A)에서 데이터가 있는 계정(Account B)의 S3에 접근해야 하는 시나리오입니다.

### 크로스 계정 구성 단계

| 단계 | 역할 | 대상 |
|------|------|------|
| 1 | Unity Catalog Master Role | Account A: Databricks 워크스페이스 |
| 2 | AssumeRole → Storage Credential IAM Role | Account A |
| 3 | AssumeRole → Data Access Role | Account B |
| 4 | S3 버킷 접근 | 최종 데이터 접근 |

> **역할 체인**: UC Master Role → Credential Role → Data Role

```json
// Account B의 Data Access Role 신뢰 정책
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/databricks-storage-credential"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "databricks-external-id-xxx"
        }
      }
    }
  ]
}
```

```json
// Account A의 Storage Credential IAM Role 정책
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::222222222222:role/data-access-role"
    }
  ]
}
```

> ⚠️ **크로스 계정 접근 시 주의사항**:
> - **STS 토큰 만료**: AssumeRole 체인이 길어지면 토큰 갱신이 빈번해집니다. 장시간 배치 잡에서 토큰 만료 에러가 날 수 있으므로, 세션 기간(Duration)을 충분히 설정하세요 (최대 12시간)
> - **리전 일치**: S3 버킷과 IAM Role이 같은 리전에 있어야 지연이 최소화됩니다
> - **CloudTrail 양쪽 활성화**: Account A와 B 모두에서 AssumeRole 이벤트를 추적할 수 있도록 CloudTrail을 활성화하세요

---

## 실전 인사이트: Credential 라이프사이클 관리

Storage Credential은 생성 후 "방치"되기 쉬운 객체입니다. 시간이 지나면 IAM Role이 삭제되거나, 정책이 변경되어 접근이 실패하는 경우가 발생합니다.

### 월간 점검 체크리스트

```sql
-- 1. 모든 Credential의 유효성 검증
-- (자동화 스크립트로 월 1회 실행)
VALIDATE EXTERNAL LOCATION prod_data;
VALIDATE EXTERNAL LOCATION staging_data;
VALIDATE EXTERNAL LOCATION raw_ingestion;

-- 2. 사용되지 않는 Credential 식별
-- (External Location이 연결되지 않은 Credential)
SELECT *
FROM system.information_schema.storage_credentials sc
WHERE NOT EXISTS (
    SELECT 1 FROM system.information_schema.external_locations el
    WHERE el.credential_name = sc.credential_name
);
```

### Credential 변경 시 영향 분석

IAM Role 정책을 변경하기 전에, 해당 Credential을 사용하는 모든 External Location과 테이블을 파악해야 합니다.

```sql
-- 특정 Credential을 사용하는 External Location 조회
SELECT location_name, url
FROM system.information_schema.external_locations
WHERE credential_name = 'aws_s3_credential';

-- 해당 External Location을 사용하는 테이블 파악
-- (Catalog Explorer에서 리니지 그래프로 확인하는 것이 더 효율적)
```

> 💡 **실전 팁**: Credential이나 IAM Role을 변경할 때는 반드시 **dev 환경에서 먼저 테스트** 하세요. 프로덕션에서 `VALIDATE`가 실패하면 모든 외부 테이블 쿼리가 즉시 실패합니다. Terraform이나 Databricks Asset Bundles로 Credential을 코드로 관리하면, 변경 이력 추적과 롤백이 용이합니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Storage Credential**| 클라우드 스토리지 접근을 위한 인증 정보 객체입니다 |
| **AWS IAM Role**| STS AssumeRole을 통해 S3에 접근합니다 |
| **Azure Managed Identity**| 자동 관리되는 ID로 ADLS에 접근합니다 |
| **External Location 연결**| Credential + 경로 = 외부 로케이션이 됩니다 |
| **권한 분리** | Credential 관리자와 로케이션 사용자를 분리합니다 |

---

## 참고 링크

- [Databricks: Storage credentials](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-storage-credentials.html)
- [Databricks: CREATE STORAGE CREDENTIAL](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-storage-credential.html)
- [Databricks: Manage storage credentials](https://docs.databricks.com/aws/en/connect/unity-catalog/storage-credentials.html)
- [Azure Databricks: Storage credentials](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/storage-credentials)
