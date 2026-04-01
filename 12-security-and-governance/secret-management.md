# 시크릿 관리 (Secret Management) 상세

## 왜 시크릿 관리가 필요한가요?

데이터 파이프라인에서는 데이터베이스 비밀번호, API 키, 서비스 계정 자격 증명 등 **민감한 정보** 를 자주 사용합니다. 이러한 값을 노트북 코드나 설정 파일에 직접 입력하면 **보안 사고** 로 이어질 수 있습니다.

> 💡 기본 개념은 [보안 개요](./security-overview.md)의 "비밀 관리" 섹션에서 소개했습니다. 이 문서에서는 Secret Scope의 유형, CLI 관리, ACL 설정을 상세히 다룹니다.

```python
# ❌ 나쁜 예: 비밀번호를 코드에 직접 입력
password = "SuperSecretP@ssw0rd!"

# ✅ 좋은 예: Secret Scope에서 가져오기
password = dbutils.secrets.get(scope="my-secrets", key="db-password")
```

---

## Secret Scope 유형

### Databricks-backed Scope

Databricks가 자체적으로 관리하는 암호화된 저장소입니다.

| 항목 | 설명 |
|------|------|
| **저장소**| Databricks 내부 (AES-256 암호화) |
| **설정 복잡도**| 낮음 (별도 인프라 불필요) |
| **적합한 환경**| AWS, GCP, 간편한 설정이 필요한 경우 |

### Azure Key Vault-backed Scope

Azure Key Vault의 Secret을 Databricks에서 직접 참조합니다.

| 항목 | 설명 |
|------|------|
| **저장소**| Azure Key Vault |
| **설정 복잡도**| 중간 (Key Vault 사전 구성 필요) |
| **적합한 환경**| Azure 환경, 중앙 집중 키 관리 필요 시 |

```bash
# Azure Key Vault-backed Scope 생성
databricks secrets create-scope my-azure-secrets \
  --scope-backend-type AZURE_KEYVAULT \
  --resource-id "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.KeyVault/vaults/<vault-name>" \
  --dns-name "https://<vault-name>.vault.azure.net/"
```

---

## Secret Scope 생성 및 관리

### Scope 생성

```bash
# Databricks-backed Scope 생성
databricks secrets create-scope my-secrets

# Scope 목록 조회
databricks secrets list-scopes
```

### Secret 추가/수정

```bash
# 문자열 값으로 Secret 추가
databricks secrets put-secret my-secrets db-password \
  --string-value "SuperSecretP@ssw0rd!"

# API 키 추가
databricks secrets put-secret my-secrets api-key \
  --string-value "sk-abc123def456ghi789"

# JDBC 연결 문자열 추가
databricks secrets put-secret my-secrets jdbc-url \
  --string-value "jdbc:postgresql://prod-db.company.com:5432/analytics"

# 파일에서 Secret 읽기 (바이너리 데이터, 인증서 등)
databricks secrets put-secret my-secrets tls-cert \
  --binary-file /path/to/certificate.pem
```

### Secret 목록 조회

```bash
# Scope 내 Secret 목록 (값은 표시되지 않음)
databricks secrets list-secrets my-secrets

# 결과 예시:
# key             last_updated_timestamp
# db-password     1705123456789
# api-key         1705234567890
# jdbc-url        1705345678901
```

### Secret 삭제

```bash
# 특정 Secret 삭제
databricks secrets delete-secret my-secrets db-password

# Scope 전체 삭제 (내부 Secret 모두 삭제)
databricks secrets delete-scope my-secrets
```

---

## 노트북에서 Secret 사용

### dbutils.secrets.get()

```python
# 기본 사용법
db_password = dbutils.secrets.get(scope="my-secrets", key="db-password")
api_key = dbutils.secrets.get(scope="my-secrets", key="api-key")

# JDBC 연결에서 사용
jdbc_url = dbutils.secrets.get(scope="my-secrets", key="jdbc-url")
db_user = dbutils.secrets.get(scope="my-secrets", key="db-user")
db_pass = dbutils.secrets.get(scope="my-secrets", key="db-password")

df = (spark.read.format("jdbc")
    .option("url", jdbc_url)
    .option("user", db_user)
    .option("password", db_pass)
    .option("dbtable", "public.customers")
    .load())
```

### Spark 설정에서 사용

```python
# 외부 스토리지 접근 키를 Secret에서 로드
storage_key = dbutils.secrets.get(scope="my-secrets", key="storage-account-key")

spark.conf.set(
    "fs.azure.account.key.myaccount.dfs.core.windows.net",
    storage_key
)
```

### SQL에서 사용

```sql
-- SQL에서는 secret() 함수를 사용합니다
CREATE TABLE bronze.external_data
USING jdbc
OPTIONS (
  url = secret('my-secrets', 'jdbc-url'),
  user = secret('my-secrets', 'db-user'),
  password = secret('my-secrets', 'db-password'),
  dbtable = 'public.customers'
);
```

---

## 보안 동작

### REDACTED 처리

```python
# Secret 값을 print하면 REDACTED로 표시됩니다
password = dbutils.secrets.get(scope="my-secrets", key="db-password")
print(password)  # 출력: [REDACTED]

# display()에서도 마찬가지입니다
# 이는 노트북 출력이나 로그에 비밀 값이 노출되지 않도록 하는 보안 조치입니다
```

### Secret 참조 시 주의사항

```python
# ❌ Secret을 문자열 조합으로 우회하면 REDACTED가 해제될 수 있습니다
password = dbutils.secrets.get(scope="my-secrets", key="db-password")
# print("".join(password))  ← 이런 우회를 하지 마세요!

# ✅ Secret은 항상 직접 함수 인자로 전달합니다
df = spark.read.format("jdbc").option("password", password).load()
```

---

## ACL (접근 제어 목록) 설정

Secret Scope에 대한 접근 권한을 세밀하게 관리할 수 있습니다.

### 권한 유형

| 권한 | 설명 |
|------|------|
| **READ**| `dbutils.secrets.get()`으로 Secret 값을 읽을 수 있습니다 |
| **WRITE**| Secret을 추가, 수정, 삭제할 수 있습니다 |
| **MANAGE**| Scope 자체를 관리(삭제, ACL 변경)할 수 있습니다 |

### ACL 설정 명령

```bash
# 그룹에 READ 권한 부여
databricks secrets put-acl my-secrets data-engineers READ

# 특정 사용자에게 WRITE 권한 부여
databricks secrets put-acl my-secrets alice@company.com WRITE

# 관리자 그룹에 MANAGE 권한 부여
databricks secrets put-acl my-secrets platform-admins MANAGE

# ACL 목록 확인
databricks secrets list-acls my-secrets

# 결과 예시:
# principal        permission
# data-engineers   READ
# alice@company.com WRITE
# platform-admins  MANAGE

# ACL 삭제
databricks secrets delete-acl my-secrets data-engineers
```

### 권장 ACL 구성

| 대상 | 권한 | 사유 |
|------|------|------|
| **platform-admins**| MANAGE | Scope 전체 관리 |
| **data-engineers**| READ | 파이프라인에서 Secret 참조 |
| **ml-engineers**| READ | ML 모델에서 API 키 사용 |
| **etl-pipeline-sp**| READ | SP가 파이프라인에서 Secret 참조 |

---

## 실전 활용 패턴

### 패턴 1: 환경별 Scope 분리

```bash
# 환경별로 별도 Scope 생성
databricks secrets create-scope dev-secrets
databricks secrets create-scope staging-secrets
databricks secrets create-scope prod-secrets

# 같은 키 이름으로 환경별 값 관리
databricks secrets put-secret dev-secrets db-host --string-value "dev-db.internal"
databricks secrets put-secret prod-secrets db-host --string-value "prod-db.internal"
```

```python
# 노트북에서 환경에 따라 Scope 선택
env = dbutils.widgets.get("environment")  # "dev" 또는 "prod"
scope = f"{env}-secrets"
db_host = dbutils.secrets.get(scope=scope, key="db-host")
```

### 패턴 2: Lakeflow Connect 연결에서 Secret 사용

```python
# 외부 데이터 소스 연결 시 Secret 활용
connection_config = {
    "host": dbutils.secrets.get("prod-secrets", "salesforce-host"),
    "username": dbutils.secrets.get("prod-secrets", "salesforce-user"),
    "password": dbutils.secrets.get("prod-secrets", "salesforce-pass"),
    "security_token": dbutils.secrets.get("prod-secrets", "salesforce-token")
}
```

---

## 현업 사례: 코드에 비밀번호를 하드코딩해서 Git에 올라간 사고

> 🔥 **실전 경험담**
>
> 한 스타트업의 데이터 엔지니어가 급하게 외부 API 연동 노트북을 작성하면서, **API 키를 코드에 직접 입력** 했습니다.
>
> ```python
> # 실제 사고 코드 (일부 마스킹)
> api_key = "sk-proj-abc123xxxxxxxxxxxxxxxx"  # ← 이 줄이 문제
> response = requests.get(url, headers={"Authorization": f"Bearer {api_key}"})
> ```
>
> 이 노트북이 Git Repo에 커밋되었고, 해당 Repo가 **퍼블릭 리포지토리** 로 설정되어 있었습니다. 24시간도 안 되어 외부에서 API 키를 탈취하여 **약 $2,000(약 270만원)의 부정 API 호출** 이 발생했습니다. GitHub에는 Secret 탐지 봇이 돌고 있어서 퍼블릭 리포의 Secret은 수분 내에 발견됩니다.
>
> **사고 수습 과정:**
> 1. API 키 즉시 무효화 (API 제공사에 연락)
> 2. Git 이력에서 Secret 제거 (`git filter-branch` 사용, 하지만 이미 캐시된 사본은 제거 불가)
> 3. 리포지토리를 Private으로 전환
> 4. 모든 노트북 대상 Secret 하드코딩 감사 수행
> 5. **Secret Scope 도입** 및 개발 가이드라인 배포
>
> **교훈**: 한 번 Git에 올라간 Secret은 **이력을 삭제해도 완전히 제거할 수 없습니다.**Fork, Cache, 로컬 Clone 등에 남아 있을 수 있기 때문입니다. **처음부터 코드에 Secret을 넣지 않는 것이 유일한 해결책** 입니다.

---

## Secret Scope 설계 전략 (환경별 분리)

현업에서 가장 효과적인 Secret Scope 설계 패턴을 소개합니다.

### 권장 Scope 구조

| Scope | Key | 설명 |
|-------|-----|------|
| **dev-db-secrets**| mysql-host, mysql-user, mysql-password | 개발 환경 DB 접속 정보 |
| **prod-db-secrets**| mysql-host, mysql-user, mysql-password | 프로덕션 환경 DB 접속 정보 |
| **api-keys**| openai-key, slack-webhook | 외부 API 키 |
| **cloud-credentials**| aws-access-key, aws-secret-key | 클라우드 접근 정보 |

### 설계 원칙

| 원칙 | 설명 | 잘못된 예 | 올바른 예 |
|------|------|----------|----------|
| **환경별 분리**| dev/prod는 반드시 다른 Scope | `my-secrets` 하나에 전부 | `dev-db-secrets`, `prod-db-secrets` |
| **용도별 분리**| DB, API, SP 등 용도별로 분리 | 모든 Secret을 한 Scope에 | DB/API/SP 별도 Scope |
| **키 이름 통일**| 환경 간 같은 키 이름 사용 | `dev_password`, `prod_pw` | 양쪽 모두 `mysql-password` |
| **최소 Scope 수**| 너무 많이 쪼개지 않음 | 팀원마다 별도 Scope | 환경+용도 조합 (5~8개) |

> 💡 **현업 팁**: **키 이름을 환경 간에 통일** 하면, 노트북 코드에서 Scope 이름만 변경하여 환경을 전환할 수 있습니다. 이는 Asset Bundles와 결합하면 더욱 강력합니다.

```python
# 환경에 따라 Scope만 전환하는 패턴
env = spark.conf.get("environment")  # "dev" 또는 "prod"
db_scope = f"{env}-db-secrets"

# 같은 코드로 dev와 prod 모두 동작
host = dbutils.secrets.get(scope=db_scope, key="mysql-host")
user = dbutils.secrets.get(scope=db_scope, key="mysql-user")
password = dbutils.secrets.get(scope=db_scope, key="mysql-password")
```

---

## Azure Key Vault 연동 시 주의사항

Azure 환경에서 Key Vault-backed Scope를 사용할 때, 현업에서 자주 만나는 문제들입니다.

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| **`SecretDoesNotExist` 에러**| Key Vault의 Secret 이름에 밑줄(`_`) 사용 | Key Vault는 영숫자와 하이픈(`-`)만 허용합니다. `db_password` → `db-password`로 변경하세요 |
| **접근 거부 (403)**| Databricks에 Key Vault 접근 권한 미부여 | Key Vault의 Access Policy에 Databricks 앱 등록이 필요합니다 |
| **Secret 업데이트 미반영**| Key Vault의 캐싱 | Key Vault Secret을 업데이트한 후 5~10분 대기하거나, 클러스터를 재시작하세요 |
| **Soft Delete 충돌**| 삭제 후 같은 이름으로 재생성 불가 | Key Vault의 Soft Delete 기능이 활성화되어 있으면, 삭제된 Secret을 먼저 Purge해야 합니다 |
| **네트워크 접근 차단**| Key Vault 방화벽 설정 | Databricks의 Control Plane IP를 Key Vault 방화벽 허용 목록에 추가하세요 |

> 🔥 **이것을 안 하면**: Azure Key Vault-backed Scope는 **읽기 전용** 입니다. `dbutils.secrets.put()`으로 값을 업데이트할 수 없고, Key Vault Portal이나 CLI에서 직접 업데이트해야 합니다. 이 사실을 모르고 자동화 스크립트에서 `put()`을 호출하여 에러가 발생하는 경우가 많습니다.

### Key Vault 연동 시 추가 보안 권장사항

```bash
# 1. Managed Identity 사용 (Secret 자체를 관리하지 않아도 됨)
# Azure Databricks는 Managed Identity로 Key Vault에 접근 가능

# 2. Key Vault 감사 로그 활성화
az monitor diagnostic-settings create \
  --name "kv-audit-logs" \
  --resource "/subscriptions/.../Microsoft.KeyVault/vaults/my-vault" \
  --workspace "/subscriptions/.../Microsoft.OperationalInsights/workspaces/my-workspace" \
  --logs '[{"category": "AuditEvent", "enabled": true}]'

# 3. Secret 만료일 설정 (Key Vault에서)
az keyvault secret set \
  --vault-name "my-vault" \
  --name "api-key" \
  --value "sk-xxx" \
  --expires "2025-06-30T00:00:00Z"
```

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **코드에 비밀 금지**| 비밀번호, API 키를 코드에 절대 하드코딩하지 않습니다 |
| **환경별 분리**| dev, staging, prod Scope을 분리합니다 |
| **최소 권한**| 필요한 사용자/그룹에게만 READ 권한을 부여합니다 |
| **정기 로테이션**| Secret 값을 주기적으로 변경합니다 (90일 권장) |
| **감사**| Secret 접근 이벤트를 모니터링합니다 |
| **SP 사용**| 프로덕션에서는 SP에 READ 권한을 부여합니다 |
| **Git Hook 설정**| 커밋 전 Secret 패턴 검사 (pre-commit hook 활용) |
| **Key Vault 선호**| Azure 환경에서는 Key Vault-backed Scope를 권장합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Secret Scope**| Secret을 그룹화하여 관리하는 컨테이너입니다 |
| **Databricks-backed**| Databricks 내부에 AES-256으로 암호화 저장합니다 |
| **Key Vault-backed**| Azure Key Vault와 연동하여 중앙 관리합니다 |
| **dbutils.secrets.get()**| 노트북에서 Secret 값을 안전하게 읽습니다 |
| **ACL**| READ/WRITE/MANAGE 권한으로 접근을 제어합니다 |
| **REDACTED** | Secret 값은 출력 시 자동으로 가려집니다 |

---

## 참고 링크

- [Databricks: Secret management](https://docs.databricks.com/aws/en/security/secrets/)
- [Databricks: Secret scopes](https://docs.databricks.com/aws/en/security/secrets/secret-scopes.html)
- [Databricks: Secret access control](https://docs.databricks.com/aws/en/security/secrets/secret-acl.html)
- [Azure Databricks: Azure Key Vault-backed scopes](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/secret-scopes)
