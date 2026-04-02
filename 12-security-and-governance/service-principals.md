# 서비스 프린시펄 (Service Principal) 상세

## 서비스 프린시펄이란?

**서비스 프린시펄(Service Principal, SP)** 은 사람이 아닌 **애플리케이션, 자동화 시스템, CI/CD 파이프라인** 을 위한 전용 계정입니다. 개인 계정 대신 서비스 프린시펄을 사용하면 특정 사람에게 종속되지 않는 안정적인 자동화를 구현할 수 있습니다.

> 💡 기본 개념은 [인증과 접근 제어](./identity-and-access.md)의 "서비스 프린시펄" 섹션에서 소개했습니다. 이 문서에서는 생성, OAuth 인증, 실전 활용을 상세히 다룹니다.

---

## SP vs PAT: 왜 서비스 프린시펄인가?

| 항목 | 개인 계정 + PAT | 서비스 프린시펄 + OAuth |
|------|----------------|----------------------|
| **인증 주체** | 특정 사람 | 서비스/애플리케이션 |
| **퇴사 영향** | 파이프라인 중단 | 영향 없음 |
| **토큰 관리** | 수동 갱신, 유출 위험 | 자동 갱신 (OAuth) |
| **감사 추적** | `alice@corp.com이 실행` | `etl-pipeline-sp가 실행` |
| **권한 범위** | 개인의 전체 권한 | 필요한 최소 권한만 설정 |
| **팀 관리** | 소유자가 명확하지 않음 | 팀이 공동으로 관리 |

---

## SP 생성

### CLI로 생성

```bash
# Account 수준 SP 생성
databricks account service-principals create \
  --display-name "etl-pipeline-sp" \
  --active true

# 결과 예시
# {
#   "id": "7890123456",
#   "application_id": "abcd-1234-efgh-5678",
#   "display_name": "etl-pipeline-sp",
#   "active": true
# }
```

### REST API로 생성

```bash
curl -X POST \
  "https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2/ServicePrincipals" \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "etl-pipeline-sp",
    "active": true
  }'
```

### Terraform으로 생성

```hcl
resource "databricks_service_principal" "etl_pipeline" {
  display_name = "etl-pipeline-sp"
  active       = true
}
```

---

## OAuth Secret 생성

서비스 프린시펄로 인증하려면 **OAuth Client Secret** 이 필요합니다.

```bash
# OAuth Secret 생성
databricks account service-principals create-oauth2-secret \
  --service-principal-id 7890123456

# 결과 예시
# {
#   "id": "secret-abc-123",
#   "secret": "dose_xxxxxxxxxxxxxxxxxxxxxxxx",  ← 이 값을 안전하게 저장하세요!
#   "status": "ACTIVE",
#   "create_time": "2025-01-15T10:30:00Z"
# }
```

> ⚠️ **Secret은 생성 시에만 표시됩니다.** 이후에는 조회할 수 없으므로, 반드시 안전한 곳(Vault, Secret Manager 등)에 저장하세요.

---

## SP로 인증하기

### Python SDK (M2M OAuth)

```python
from databricks.sdk import WorkspaceClient

# Service Principal로 인증
w = WorkspaceClient(
    host="https://dbc-abc123.cloud.databricks.com",
    client_id="abcd-1234-efgh-5678",       # SP의 application_id
    client_secret="dose_xxxxxxxxxxxxxxxx"   # OAuth secret
)

# API 호출 테스트
current_user = w.current_user.me()
print(f"인증된 SP: {current_user.display_name}")
```

### Databricks CLI

```bash
# CLI 프로필에 SP 인증 설정
cat >> ~/.databrickscfg << 'EOF'
[etl-pipeline]
host          = https://dbc-abc123.cloud.databricks.com
client_id     = abcd-1234-efgh-5678
client_secret = dose_xxxxxxxxxxxxxxxx
EOF

# 프로필로 CLI 사용
databricks --profile etl-pipeline clusters list
```

### 환경 변수 방식

```bash
export DATABRICKS_HOST="https://dbc-abc123.cloud.databricks.com"
export DATABRICKS_CLIENT_ID="abcd-1234-efgh-5678"
export DATABRICKS_CLIENT_SECRET="dose_xxxxxxxxxxxxxxxx"

# SDK가 자동으로 환경 변수를 인식합니다
databricks clusters list
```

---

## SP에 권한 부여

### 그룹에 추가

```bash
# SP를 Account 그룹에 추가
databricks account groups patch <group-id> \
  --json '{
    "Operations": [{
      "op": "add",
      "path": "members",
      "value": [{"value": "7890123456"}]
    }]
  }'
```

### Unity Catalog 권한

```sql
-- SP에 직접 권한 부여
GRANT USE CATALOG ON CATALOG production TO `etl-pipeline-sp`;
GRANT USE SCHEMA ON SCHEMA production.ecommerce TO `etl-pipeline-sp`;
GRANT SELECT, MODIFY ON SCHEMA production.ecommerce TO `etl-pipeline-sp`;

-- 또는 그룹을 통해 간접 부여 (권장)
-- SP를 data_engineers 그룹에 추가한 후, 그룹에 권한 부여
GRANT ALL PRIVILEGES ON SCHEMA production.bronze TO `data_engineers`;
```

### 워크스페이스 할당

```bash
# SP를 특정 워크스페이스에 할당
databricks account workspace-assignment update \
  --workspace-id 12345 \
  --principal-id 7890123456 \
  --permissions USER
```

---

## SP로 Job 실행

### run_as 설정

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Job 생성 시 SP를 실행 주체로 지정
job = w.jobs.create(
    name="daily-etl-pipeline",
    run_as={"service_principal_name": "etl-pipeline-sp"},
    tasks=[{
        "task_key": "extract",
        "notebook_task": {
            "notebook_path": "/Repos/production/etl/extract"
        },
        "new_cluster": {
            "spark_version": "15.4.x-scala2.12",
            "num_workers": 4,
            "node_type_id": "i3.xlarge"
        }
    }]
)

print(f"Job 생성 완료: {job.job_id}")
```

### SQL로 Job 생성 (Workflows)

기존 Job을 SP로 전환할 수 있습니다:

1. **Workflows**> Job 선택 > **Job details**
2. **Run as** 드롭다운에서 서비스 프린시펄 선택
3. SP에 필요한 클러스터 정책, 데이터 접근 권한이 있는지 확인

---

## SP 관리

### 목록 조회

```bash
# 모든 SP 목록 조회
databricks account service-principals list

# 특정 SP 정보 조회
databricks account service-principals get --id 7890123456
```

### 비활성화

```bash
# SP 비활성화 (삭제하지 않고 비활성화)
databricks account service-principals update \
  --id 7890123456 \
  --active false
```

### Secret 로테이션

```bash
# 새 Secret 생성
databricks account service-principals create-oauth2-secret \
  --service-principal-id 7890123456

# 기존 Secret 삭제
databricks account service-principals delete-oauth2-secret \
  --service-principal-id 7890123456 \
  --secret-id old-secret-id
```

> 💡 **Secret 로테이션 절차**: 새 Secret을 생성하고, 모든 시스템에 새 Secret을 배포한 후, 기존 Secret을 삭제합니다. 이렇게 하면 다운타임 없이 교체할 수 있습니다.

---

## 현업 사례: 개인 PAT로 프로덕션 Job을 돌리다가 퇴사로 장애 발생

> 🔥 **실전 경험담**
>
> 한 고객사에서 데이터 엔지니어 A씨가 모든 프로덕션 Lakeflow Jobs를 **본인의 개인 계정 + PAT(Personal Access Token)** 으로 설정하여 운영하고 있었습니다. A씨가 퇴사하자, IT 팀이 계정을 비활성화했고, **그 순간 15개의 프로덕션 ETL Job이 동시에 실패** 했습니다.
>
> 장애 복구에 걸린 시간: **6시간**. 각 Job의 `run_as`를 새 계정으로 변경하고, UC 권한을 재설정하고, Secret Scope의 ACL을 수정해야 했습니다. 그 사이에 일별 매출 리포트가 지연되어 경영진 회의에 영향을 미쳤습니다.
>
> **이 사고 이후 도입한 규칙:**
> 1. **모든 프로덕션 Job은 반드시 SP로 실행**(개인 계정 사용 금지)
> 2. **SP의 Secret은 Vault에 저장**(퇴사와 무관하게 유지)
> 3. **SP 관리 책임은 팀 단위**(개인이 아닌 그룹이 SP를 소유)

---

## SP 관리 체계 설계

실제 기업에서 SP를 효과적으로 관리하려면 체계적인 네이밍 규칙과 관리 프로세스가 필요합니다.

### 권장 네이밍 규칙

**명명 규칙**: `{환경}-{팀}-{용도}-sp`

| 이름 | 용도 |
|------|------|
| prod-de-etl-sp | 프로덕션 데이터 엔지니어링 ETL |
| prod-de-lakeflow-sp | 프로덕션 Lakeflow Connect 수집 |
| prod-ml-training-sp | 프로덕션 ML 모델 학습 |
| prod-ml-serving-sp | 프로덕션 모델 서빙 |
| prod-bi-reporting-sp | 프로덕션 BI 리포트 |
| dev-ds-experiment-sp | 개발 환경 실험용 |

### SP 인벤토리 관리

현업에서는 SP가 늘어나면 "이 SP가 뭐에 쓰이는 거지?"라는 질문이 반복됩니다. SP 인벤토리를 관리하세요.

```sql
-- SP 인벤토리 테이블 (수동 관리 또는 자동화)
CREATE TABLE IF NOT EXISTS admin.governance.sp_inventory (
    sp_name STRING,
    sp_application_id STRING,
    owner_team STRING,
    purpose STRING,
    created_date DATE,
    last_secret_rotation DATE,
    environments ARRAY<STRING>,
    uc_grants STRING,
    status STRING  -- 'ACTIVE', 'DEPRECATED', 'DEACTIVATED'
);

-- 미사용 SP 탐지 (시스템 테이블 활용)
SELECT
    sp.display_name,
    sp.application_id,
    MAX(al.event_time) AS last_activity
FROM system.access.audit AS al
RIGHT JOIN admin.governance.sp_inventory AS sp
    ON al.user_identity.email = sp.sp_name
WHERE sp.status = 'ACTIVE'
GROUP BY sp.display_name, sp.application_id
HAVING MAX(al.event_time) < CURRENT_DATE() - INTERVAL 90 DAYS
    OR MAX(al.event_time) IS NULL
ORDER BY last_activity;
```

> 💡 **현업 팁**: **90일 이상 사용되지 않은 SP는 비활성화 후보** 입니다. 바로 삭제하지 말고, 먼저 비활성화(`active = false`)하고 2주간 문제가 없으면 삭제합니다. 갑자기 삭제하면 분기별로만 실행되는 Job이 영향받을 수 있습니다.

---

## 시크릿 로테이션 자동화

현업에서 가장 많이 놓치는 부분이 Secret 로테이션입니다. 수동으로 하면 100% 잊어버립니다.

### Lakeflow Jobs로 자동 로테이션

```python
# 매 90일마다 실행되는 Secret 로테이션 Job
from databricks.sdk import AccountClient
import json

def rotate_sp_secret(sp_id: str, sp_name: str):
    """SP의 OAuth Secret을 다운타임 없이 교체합니다."""

    account_client = AccountClient(
        host="https://accounts.cloud.databricks.com",
        # 로테이션 Job 자체도 SP로 실행 (관리자 SP)
        client_id=dbutils.secrets.get("admin-secrets", "admin-sp-client-id"),
        client_secret=dbutils.secrets.get("admin-secrets", "admin-sp-secret")
    )

    # Step 1: 새 Secret 생성 (이 시점에서 기존 + 새 Secret 모두 유효)
    new_secret = account_client.service_principals.create_oauth2_secret(
        service_principal_id=sp_id
    )

    # Step 2: Vault/Secret Manager에 새 Secret 저장
    # (실제로는 AWS Secrets Manager, Azure Key Vault 등에 저장)
    dbutils.secrets.put(
        scope="prod-sp-secrets",
        key=f"{sp_name}-client-secret",
        string_value=new_secret.secret
    )

    # Step 3: 새 Secret이 전파될 시간을 대기 (보수적으로 5분)
    import time
    time.sleep(300)

    # Step 4: 기존 Secret 삭제
    old_secrets = account_client.service_principals.list_oauth2_secrets(
        service_principal_id=sp_id
    )
    for old in old_secrets:
        if old.id != new_secret.id:
            account_client.service_principals.delete_oauth2_secret(
                service_principal_id=sp_id,
                secret_id=old.id
            )

    print(f"✅ {sp_name} Secret 로테이션 완료")
```

### 로테이션 스케줄 권장사항

| SP 용도 | 로테이션 주기 | 이유 |
|---------|:-----------:|------|
| **프로덕션 ETL** | 90일 | 보안 기준 준수 |
| **CI/CD 배포** | 60일 | 외부 시스템(GitHub Actions 등)에 노출 |
| **모델 서빙** | 90일 | 장기 실행 엔드포인트 |
| **관리자 SP** | 30일 | 높은 권한을 가지므로 더 자주 로테이션 |

> 🔥 **이것을 안 하면**: Secret을 1년 이상 로테이션하지 않으면, 만약 Secret이 유출되었어도 **유출 사실조차 인지하지 못한 채 장기간 악용** 될 수 있습니다. 또한 많은 보안 감사(SOC2, ISO27001 등)에서 "자격 증명 로테이션 주기"를 필수 점검 항목으로 요구합니다.

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **용도별 SP 분리** | ETL, ML, CI/CD 등 용도별로 별도 SP를 생성합니다 |
| **그룹 기반 권한** | SP를 그룹에 추가하고, 그룹에 권한을 부여합니다 |
| **Secret 로테이션** | 90일 간격으로 OAuth Secret을 로테이션합니다 |
| **Secret 안전 보관** | Secret을 코드에 하드코딩하지 않고, Vault에 저장합니다 |
| **최소 권한** | 각 SP에 필요한 최소 권한만 부여합니다 |
| **네이밍 규칙** | `{환경}-{팀}-{용도}-sp` 형식으로 이름을 지정합니다 |
| **정기 감사** | 미사용 SP를 정기적으로 식별하고 비활성화합니다 |
| **인벤토리 관리** | 모든 SP의 용도, 소유 팀, 권한을 문서화합니다 |
| **퇴사 프로세스 연동** | 직원 퇴사 시 개인 계정 기반 Job을 SP로 전환합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Service Principal** | 자동화/애플리케이션용 비인간 계정입니다 |
| **OAuth M2M** | Client ID + Secret으로 토큰을 발급받는 인증 방식입니다 |
| **run_as** | Job을 SP 권한으로 실행하도록 설정합니다 |
| **Secret 로테이션** | 정기적으로 Secret을 교체하여 보안을 유지합니다 |
| **그룹 기반** | SP를 그룹에 추가하여 권한을 관리합니다 |

---

## 참고 링크

- [Databricks: Service principals](https://docs.databricks.com/aws/en/admin/users-groups/service-principals.html)
- [Databricks: OAuth M2M authentication](https://docs.databricks.com/aws/en/dev-tools/auth/oauth-m2m.html)
- [Databricks: Manage service principals](https://docs.databricks.com/aws/en/admin/users-groups/service-principals.html)
- [Azure Databricks: Service principals](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/service-principals)
