# 고객 관리 키 (Customer-Managed Keys) 암호화 상세

## CMK란?

**CMK(Customer-Managed Keys, 고객 관리 키)** 는 Databricks가 데이터를 암호화할 때 **고객이 소유하고 관리하는 암호화 키**를 사용하는 기능입니다. 기본적으로 Databricks는 플랫폼이 관리하는 키로 데이터를 암호화하지만, CMK를 사용하면 고객이 키의 생성, 로테이션, 폐기를 직접 통제할 수 있습니다.

> 💡 기본 개념은 [보안 개요](./security-overview.md)의 "데이터 암호화" 섹션에서 소개했습니다. 이 문서에서는 CMK의 적용 대상, 클라우드별 설정, 키 관리를 상세히 다룹니다.

---

## 왜 CMK가 필요한가요?

| 상황 | 기본 암호화 | CMK |
|------|----------|-----|
| **키 소유권** | Databricks가 키를 관리 | 고객이 키를 소유/관리 |
| **키 폐기** | 불가 | 키를 폐기하면 Databricks도 데이터에 접근 불가 |
| **규제 준수** | 일반적인 규제 충족 | HIPAA, PCI-DSS, 금융 규제 요건 충족 |
| **감사** | 제한적 | 키 사용 이력을 고객이 직접 감사 |

> 💡 **핵심 가치**: CMK의 가장 큰 가치는 **"키를 폐기하면 데이터가 접근 불가능해진다"**는 점입니다. 이를 통해 고객은 자신의 데이터에 대한 궁극적인 통제권을 확보합니다.

---

## CMK 적용 대상

Databricks에서 CMK를 적용할 수 있는 세 가지 영역이 있습니다.

| 적용 대상 | 설명 | 보호 범위 |
|-----------|------|----------|
| **관리형 서비스 데이터** | Control Plane에 저장되는 노트북, 쿼리 결과, 시크릿 등 | 노트북 소스, 쿼리 히스토리, DBFS 경로 |
| **워크스페이스 스토리지** | 워크스페이스의 루트 스토리지 (DBFS 루트) | DBFS에 저장된 데이터 |
| **Managed Table 스토리지** | Unity Catalog가 관리하는 테이블 데이터 | Delta 테이블 파일 |

### 클라우드별 CMK 지원

| 적용 대상 | AWS (KMS) | Azure (Key Vault) | GCP (Cloud KMS) |
|-----------|-----------|-------------------|-----------------|
| 관리형 서비스 | ✅ | ✅ | ✅ |
| 워크스페이스 스토리지 | ✅ (S3 SSE-KMS) | ✅ (ADLS 암호화) | ✅ |
| EBS 볼륨 (컴퓨트) | ✅ | N/A (Managed Disk) | N/A |

---

## AWS KMS 설정

### 단계 1: KMS 키 생성

```bash
# AWS CLI로 KMS 키 생성
aws kms create-key \
  --description "Databricks CMK - Managed Services" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS

# 결과에서 KeyId를 메모합니다
# "KeyId": "arn:aws:kms:us-east-1:123456789012:key/abcd-1234-efgh-5678"
```

### 단계 2: 키 정책 설정

Databricks가 이 키를 사용할 수 있도록 키 정책을 설정합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow Databricks to use the key",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::414351767826:root"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow Databricks to manage grants",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::414351767826:root"
      },
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    }
  ]
}
```

### 단계 3: Databricks에 CMK 등록

```bash
# Databricks Account API로 CMK 등록
curl -X POST \
  "https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/customer-managed-keys" \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "aws_key_info": {
      "key_arn": "arn:aws:kms:us-east-1:123456789012:key/abcd-1234-efgh-5678",
      "key_region": "us-east-1"
    },
    "use_cases": ["MANAGED_SERVICES"]
  }'
```

---

## Azure Key Vault 설정

### 단계 1: Key Vault 및 키 생성

```bash
# Key Vault 생성
az keyvault create \
  --name "databricks-cmk-vault" \
  --resource-group "databricks-rg" \
  --location "eastus" \
  --sku premium

# 암호화 키 생성
az keyvault key create \
  --vault-name "databricks-cmk-vault" \
  --name "databricks-managed-services-key" \
  --kty RSA \
  --size 2048
```

### 단계 2: 접근 정책 설정

Databricks의 엔터프라이즈 앱에 Key Vault 접근 권한을 부여합니다.

```bash
# Databricks 리소스 프로바이더에 접근 권한 부여
az keyvault set-policy \
  --name "databricks-cmk-vault" \
  --object-id "<databricks-enterprise-app-object-id>" \
  --key-permissions get wrapKey unwrapKey
```

### 단계 3: 워크스페이스에 CMK 적용

Azure Portal에서:
1. **Databricks 워크스페이스** 리소스 이동
2. **Encryption** 탭 선택
3. **Customer-managed key** 선택
4. Key Vault와 키 이름 입력
5. 저장

---

## 키 로테이션

### 자동 로테이션 (권장)

```bash
# AWS KMS: 자동 로테이션 활성화 (1년 주기)
aws kms enable-key-rotation \
  --key-id "arn:aws:kms:us-east-1:123456789012:key/abcd-1234-efgh-5678"

# Azure Key Vault: 자동 로테이션 정책 설정
az keyvault key rotation-policy update \
  --vault-name "databricks-cmk-vault" \
  --name "databricks-managed-services-key" \
  --value '{"lifetimeActions": [{"trigger": {"timeAfterCreate": "P365D"}, "action": {"type": "Rotate"}}]}'
```

### 수동 로테이션

```bash
# AWS: 새 키를 생성하고 Databricks에 업데이트
# (기존 데이터는 이전 키로 복호화, 새 데이터는 새 키로 암호화)

# Azure: Key Vault에서 새 버전 생성
az keyvault key create \
  --vault-name "databricks-cmk-vault" \
  --name "databricks-managed-services-key" \
  --kty RSA \
  --size 2048
# → Key Vault는 자동으로 최신 버전을 사용합니다
```

> ⚠️ **키 로테이션 시 주의**: 이전 키 버전을 즉시 삭제하면 이전 키로 암호화된 데이터를 복호화할 수 없습니다. 이전 버전은 충분한 기간 동안 유지하세요.

---

## 키 폐기와 데이터 접근 차단

CMK를 비활성화하면 Databricks가 해당 키로 암호화된 데이터에 접근할 수 없습니다.

```bash
# AWS: 키 비활성화
aws kms disable-key \
  --key-id "arn:aws:kms:us-east-1:123456789012:key/abcd-1234-efgh-5678"

# Azure: 키 비활성화
az keyvault key set-attributes \
  --vault-name "databricks-cmk-vault" \
  --name "databricks-managed-services-key" \
  --enabled false
```

> ⚠️ **키 비활성화는 워크스페이스 중단을 초래합니다.** 키를 비활성화하면 해당 워크스페이스의 모든 기능이 중단됩니다. 테스트 환경에서 먼저 영향을 확인하세요.

---

## 모니터링

### AWS KMS 키 사용 감사

```bash
# CloudTrail에서 KMS 키 사용 이벤트 조회
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::KMS::Key \
  --max-results 10
```

### Azure Key Vault 감사

```bash
# Key Vault 진단 로그 활성화
az monitor diagnostic-settings create \
  --resource "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.KeyVault/vaults/databricks-cmk-vault" \
  --name "cmk-audit" \
  --logs '[{"category": "AuditEvent", "enabled": true}]' \
  --workspace "<log-analytics-workspace-id>"
```

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **자동 로테이션** | 키 로테이션을 자동화하여 수동 관리 부담을 줄입니다 |
| **이전 버전 유지** | 로테이션 후 이전 키 버전을 최소 90일간 유지합니다 |
| **감사 로그** | KMS/Key Vault의 사용 로그를 모니터링합니다 |
| **재해 복구** | 키 백업 및 복구 절차를 수립합니다 |
| **환경 분리** | dev/prod 워크스페이스에 별도 키를 사용합니다 |
| **알림 설정** | 키 만료, 비활성화 이벤트에 대한 알림을 설정합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **CMK** | 고객이 소유/관리하는 키로 데이터를 암호화합니다 |
| **적용 대상** | 관리형 서비스, 워크스페이스 스토리지, EBS 볼륨 |
| **AWS KMS** | AWS 환경에서 CMK를 제공하는 키 관리 서비스입니다 |
| **Azure Key Vault** | Azure 환경에서 CMK를 제공하는 키 관리 서비스입니다 |
| **키 로테이션** | 정기적으로 키를 교체하여 보안을 강화합니다 |
| **키 폐기** | 키를 비활성화하면 데이터 접근이 완전히 차단됩니다 |

---

## 참고 링크

- [Databricks: Customer-managed keys](https://docs.databricks.com/aws/en/security/keys/)
- [Databricks: CMK for managed services](https://docs.databricks.com/aws/en/security/keys/customer-managed-keys-managed-services-aws.html)
- [Databricks: CMK for workspace storage](https://docs.databricks.com/aws/en/security/keys/customer-managed-keys-storage-aws.html)
- [Azure Databricks: Customer-managed keys](https://learn.microsoft.com/en-us/azure/databricks/security/keys/)
