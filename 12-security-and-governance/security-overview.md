# 보안 개요

## Databricks의 보안 모델

Databricks는 **공유 책임 모델(Shared Responsibility Model)**을 따릅니다. 플랫폼의 보안은 Databricks와 고객이 역할을 나누어 담당합니다. 이 문서에서는 Databricks 보안의 전체 그림을 살펴보고, 각 보안 계층이 어떻게 동작하는지 이해하겠습니다.

> 💡 ** 공유 책임 모델(Shared Responsibility Model)**이란, 클라우드 서비스 제공자(Databricks)와 고객이 각각의 보안 영역을 분담하는 모델입니다. AWS, Azure, GCP 같은 클라우드 제공자들도 동일한 개념을 사용합니다.

---

## 공유 책임 모델

| 책임 영역 | 항목 | 설명 |
|-----------|------|------|
| **Databricks 책임**| Control Plane 인프라 보안 | 플랫폼 인프라를 보호합니다 |
|  | 플랫폼 업데이트 / 보안 패치 | 정기적으로 업데이트합니다 |
|  | 기본 암호화 (TLS + AES-256) | 전송 중 및 저장 시 암호화합니다 |
|  | 인증 유지 (SOC 2, ISO 27001) | 컴플라이언스 인증을 유지합니다 |
| ** 고객 책임**| 네트워크 설정 (VPC, Private Link) | 네트워크 보안을 구성합니다 |
|  | 사용자/그룹 관리 | ID 및 접근을 관리합니다 |
|  | 데이터 접근 정책 (GRANT/REVOKE) | 데이터 권한을 설정합니다 |
|  | CMK, IP Access List 설정 | 추가 보안을 구성합니다 |

| 책임 영역 | Databricks 담당 | 고객 담당 |
|-----------|----------------|----------|
| Control Plane 인프라 보안 | ✅ | |
| 플랫폼 업데이트/보안 패치 | ✅ | |
| 전송/저장 데이터 암호화 | ✅ (기본 활성화) | |
| SOC 2, ISO 27001 등 인증 유지 | ✅ | |
| Data Plane 네트워크 설정 | | ✅ |
| 사용자 계정/그룹 관리 | | ✅ |
| 데이터 접근 정책 (GRANT/REVOKE) | | ✅ |
| IP Access List 설정 | | ✅ |
| Private Link 구성 | | ✅ (선택) |
| CMK(고객 관리 키) 설정 | | ✅ (선택) |
| 규정 준수 (GDPR 등) 이행 | 공유 | 공유 |

---

## 보안 계층 구조 (Defense in Depth)

Databricks의 보안은 ** 다층 방어(Defense in Depth)**전략을 따릅니다. 한 계층이 뚫리더라도 다음 계층이 데이터를 보호합니다.

| 계층 | 보안 영역 | 주요 기술 |
|------|-----------|----------|
| 1 | 네트워크 보안 | VPC, Private Link, IP 제한 |
| 2 | 인증 (Authentication) | SSO, MFA, PAT, OAuth, Service Principal |
| 3 | 인가 (Authorization) | Unity Catalog GRANT/REVOKE, RBAC, ABAC |
| 4 | 데이터 보안 | 암호화(TLS+AES-256), CMK, Row Filter, Column Mask |
| 5 | 감사 (Audit) | system.access.audit, 빌링, 리니지 |

> 네트워크 보안 → 인증 → 인가 → 데이터 보안 → 감사 순으로 계층적으로 보호합니다.

| 계층 | 핵심 질문 | Databricks 도구 | 상세 문서 |
|------|----------|----------------|----------|
| ** 네트워크**| "허용된 네트워크에서만 접근하는가?" | VPC, Private Link, IP Access List | [네트워크 보안](./network-security.md) |
| ** 인증**| "이 사용자가 누구인가?" | SSO, MFA, Service Principal, PAT, OAuth | [ID 및 접근](./identity-and-access.md) |
| ** 인가**| "이 사용자가 이 데이터를 볼 수 있는가?" | Unity Catalog GRANT/REVOKE, Row Filter, Column Mask | [접근 제어](../07-unity-catalog/access-control.md) |
| ** 데이터 보안**| "데이터가 안전하게 저장/전송되는가?" | TLS 1.2+ (전송), AES-256 (저장), CMK (선택) | 아래 섹션 |
| ** 감사**| "누가 언제 무엇을 했는가?" | 시스템 테이블 (audit, billing, lineage) | [시스템 테이블](./system-tables.md) |

---

## 데이터 암호화

Databricks는 데이터를 ** 저장 시(At-Rest)**와 ** 전송 중(In-Transit)**모두 암호화합니다.

### 암호화 유형

| 유형 | 방식 | 기본 활성화 | 설명 |
|------|------|-----------|------|
| ** 전송 중 (In-Transit)**| TLS 1.2+ | ✅ 자동 | 모든 네트워크 통신이 TLS로 암호화됩니다 |
| ** 저장 시 (At-Rest)**| AES-256 | ✅ 자동 | 클라우드 제공자의 기본 암호화가 적용됩니다 |
| ** 고객 관리 키 (CMK)**| 고객 KMS 키 | ❌ 선택 | 고객이 소유한 암호화 키를 사용합니다 |

### 고객 관리 키 (Customer-Managed Keys, CMK)

CMK를 사용하면 Databricks가 데이터를 암호화할 때 ** 고객이 소유한 키**를 사용합니다. 키를 철회(revoke)하면 Databricks도 데이터에 접근할 수 없습니다.

| CMK 적용 대상 | AWS | Azure |
|--------------|-----|-------|
| **관리형 서비스 데이터** | KMS | Key Vault |
| **Workspace 스토리지**| S3 SSE-KMS | ADLS 암호화 |
| **DBFS 루트**| KMS | Key Vault |

> 💡 **CMK(Customer-Managed Key)**는 규제 산업(금융, 의료)에서 "우리 데이터는 우리 키로만 암호화해야 한다"는 요건을 충족하기 위해 사용합니다. 키 관리 서비스(AWS KMS, Azure Key Vault)에서 키를 생성하고 Databricks에 연결합니다.

---

## 비밀 관리 (Secrets)

데이터베이스 비밀번호, API 키 같은 민감한 정보를 코드에 직접 넣으면 보안 사고로 이어질 수 있습니다. Databricks의 **Secret**기능을 사용하면 민감한 값을 안전하게 저장하고 참조할 수 있습니다.

### Secret Scope 유형

| 유형 | 저장소 | 특징 |
|------|-------|------|
| **Databricks-backed**| Databricks 내부 (AES-256 암호화) | 별도 설정 불필요, 간편함 |
| **Azure Key Vault-backed**| Azure Key Vault | Azure 환경에서 중앙 집중 키 관리 |

### 실습: Secret Scope 생성 및 사용

```bash
# 1. Secret Scope 생성
databricks secrets create-scope my-secrets

# 2. Secret 저장
databricks secrets put-secret my-secrets db-password --string-value "SuperSecretP@ssw0rd!"

# 3. Secret 목록 조회
databricks secrets list-secrets my-secrets
```

```python
# 노트북에서 Secret 사용
db_password = dbutils.secrets.get(scope="my-secrets", key="db-password")

# JDBC 연결에서 사용
jdbc_url = f"jdbc:postgresql://host:5432/mydb?user=admin&password={db_password}"
df = spark.read.format("jdbc").option("url", jdbc_url).load()
```

> ⚠️ ** 주의**: `dbutils.secrets.get()`의 결과를 `print()`하면 `[REDACTED]`로 표시됩니다. 이는 의도적인 보안 조치입니다. 노트북 출력이나 로그에 비밀 값이 노출되지 않도록 보호합니다.

### Secret에 대한 접근 제어

```bash
# 특정 그룹에게 Secret 읽기 권한 부여
databricks secrets put-acl my-secrets data-engineers READ

# 접근 제어 목록 확인
databricks secrets list-acls my-secrets
```

| 권한 | 설명 |
|------|------|
| **READ**| `dbutils.secrets.get()`으로 값을 읽을 수 있습니다 |
| **WRITE**| Secret을 추가/수정/삭제할 수 있습니다 |
| **MANAGE**| Secret Scope 자체를 관리(삭제, ACL 변경)할 수 있습니다 |

---

## 토큰 관리 (Personal Access Token)

**PAT(Personal Access Token)**는 REST API나 CLI에서 인증할 때 사용하는 토큰입니다.

| 설정 | 설명 | 권장 |
|------|------|------|
| ** 토큰 수명**| 토큰의 만료 기간을 제한합니다 | 90일 이하 |
| ** 최대 토큰 수**| 사용자당 생성 가능한 토큰 수를 제한합니다 | 필요 최소한 |
| ** 토큰 권한 제어**| 특정 그룹만 토큰을 생성할 수 있도록 제한합니다 | 관리자만 생성 허용 |

```bash
# 토큰 생성 (CLI)
databricks tokens create --lifetime-seconds 7776000 --comment "CI/CD pipeline"

# 토큰 목록 조회
databricks tokens list
```

> 💡 ** 프로덕션 환경에서는 PAT 대신 Service Principal + OAuth를 권장합니다.**PAT는 개인에게 종속되어, 해당 직원이 퇴사하면 토큰도 무효화해야 합니다.

---

## 서비스 프린시펄 (Service Principal)

**Service Principal**은 자동화된 파이프라인, CI/CD, 스케줄된 작업에서 **사람 대신 인증하는 서비스 계정**입니다.

### PAT vs Service Principal 비교

| 항목 | PAT | Service Principal + OAuth |
|------|-----|--------------------------|
| **인증 주체**| 개인 사용자 | 서비스 (비인간) |
| ** 수명 관리**| 수동 갱신 필요 | 자동 토큰 교체 (OAuth) |
| ** 퇴사 영향**| 토큰 무효화 필요 | 영향 없음 |
| ** 감사 추적**| 개인 이름으로 기록 | 서비스 이름으로 기록 |
| ** 적합한 용도**| 개발/테스트 | 프로덕션 자동화 |

```bash
# Service Principal 생성 (Account 수준)
databricks account service-principals create \
  --display-name "etl-pipeline-sp" \
  --active true

# Service Principal에 OAuth Secret 생성
databricks service-principals create-oauth2-secret <sp-id>
```

---

## Enhanced Security Monitoring

**Enhanced Security Monitoring (강화된 보안 모니터링)**은 보안 프로필을 활성화하면 사용할 수 있는 추가 보안 기능입니다.

| 기능 | 설명 |
|------|------|
| ** 강화된 모니터링 에이전트**| 클러스터에서 실행되는 프로세스, 네트워크 연결을 모니터링합니다 |
| ** 안티바이러스 모니터링**| 악성 파일 탐지 기능을 제공합니다 |
| ** 파일 무결성 모니터링**| 시스템 파일의 무단 변경을 감지합니다 |
| ** 취약점 스캐닝**| 클러스터 이미지의 보안 취약점을 스캔합니다 |

> 🆕 **Compliance Security Profile**: HIPAA, PCI-DSS, FedRAMP 등 규정 준수가 필요한 환경에서 활성화합니다. 활성화하면 위의 모니터링 기능이 자동으로 적용됩니다.

---

## 감사 로그 (Audit Logs) 활용

Databricks는 모든 사용자 활동을 **시스템 테이블**에 기록합니다. 이를 활용하여 보안 이벤트를 모니터링할 수 있습니다.

### 주요 감사 쿼리 예제

```sql
-- 최근 24시간 내 로그인 실패 이벤트 조회
SELECT
    event_time,
    user_identity.email AS user_email,
    source_ip_address,
    service_name,
    action_name,
    response.status_code
FROM system.access.audit
WHERE action_name = 'login'
  AND response.status_code != 200
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 24 HOURS
ORDER BY event_time DESC;
```

```sql
-- 민감한 테이블에 대한 접근 이력 조회
SELECT
    event_time,
    user_identity.email AS user_email,
    action_name,
    request_params.full_name_arg AS table_name,
    source_ip_address
FROM system.access.audit
WHERE request_params.full_name_arg LIKE '%pii%'
  AND action_name IN ('getTable', 'commandSubmit')
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 7 DAYS
ORDER BY event_time DESC;
```

```sql
-- 권한 변경 이력 추적
SELECT
    event_time,
    user_identity.email AS changed_by,
    action_name,
    request_params
FROM system.access.audit
WHERE action_name IN ('updatePermissions', 'grantPermission', 'revokePermission')
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 30 DAYS
ORDER BY event_time DESC;
```

---

## 규정 준수 (Compliance)

Databricks는 주요 보안 인증 및 규정 준수 프레임워크를 지원합니다.

| 인증/규정 | 설명 | 관련 산업 |
|-----------|------|----------|
| **SOC 2 Type II**| 보안, 가용성, 처리 무결성, 기밀성, 개인정보 | 전 산업 |
| **ISO 27001**| 정보보안 관리 시스템(ISMS) 국제 표준 | 전 산업 |
| **HIPAA**| 미국 의료 정보 보호법 | 의료/헬스케어 |
| **GDPR**| EU 개인정보 보호 규정 | EU 대상 사업 |
| **PCI DSS**| 결제 카드 데이터 보안 표준 | 금융/결제 |
| **FedRAMP**| 미국 연방 정부 클라우드 보안 | 공공/정부 |
| **HITRUST**| 건강 정보 보안 프레임워크 | 의료/보험 |

---

## 보안 구성 권장 단계

| 단계 | 작업 | 우선순위 | 상세 |
|------|------|---------|------|
| 1 | **SSO 설정**| 필수 | 기업 IdP(Okta, Azure AD, OneLogin)와 연동합니다 |
| 2 | **MFA 활성화**| 필수 | 다중 인증을 강제합니다 (IdP에서 설정) |
| 3 | **SCIM 연동**| 필수 | 사용자/그룹을 IdP에서 자동 동기화합니다 |
| 4 | **Unity Catalog 활성화**| 필수 | 데이터 거버넌스의 기반입니다 |
| 5 | **IP Access List 설정**| 권장 | 허용된 IP에서만 접근하도록 제한합니다 |
| 6 | **Service Principal 사용**| 권장 | 프로덕션 자동화에서 PAT 대신 사용합니다 |
| 7 | **Secret Scope 구성**| 권장 | 비밀번호/API 키를 안전하게 관리합니다 |
| 8 | **Private Link 구성**| 규제 산업 | 퍼블릭 인터넷을 거치지 않는 통신을 보장합니다 |
| 9 | **CMK 설정**| 규제 산업 | 고객 관리 키로 데이터를 암호화합니다 |
| 10 | ** 감사 대시보드 구축**| 권장 | 시스템 테이블로 보안 이벤트를 모니터링합니다 |

---

## 보안 모범 사례 체크리스트

| # | 영역 | 항목 | 확인 |
|---|------|------|------|
| 1 | ** 인증**| SSO + MFA가 활성화되어 있는가? | ☐ |
| 2 | ** 인증**| SCIM으로 사용자/그룹이 자동 동기화되는가? | ☐ |
| 3 | ** 인증**| PAT 대신 Service Principal + OAuth를 사용하는가? | ☐ |
| 4 | ** 인증**| PAT 수명이 90일 이하로 제한되어 있는가? | ☐ |
| 5 | ** 네트워크**| IP Access List가 활성화되어 있는가? | ☐ |
| 6 | ** 네트워크**| Customer-Managed VPC를 사용하는가? (프로덕션) | ☐ |
| 7 | ** 권한**| Unity Catalog로 모든 데이터 자산을 관리하는가? | ☐ |
| 8 | ** 권한**| 최소 권한 원칙(Least Privilege)을 따르는가? | ☐ |
| 9 | ** 권한**| PII 데이터에 Row Filter / Column Mask가 적용되어 있는가? | ☐ |
| 10 | ** 암호화**| 민감한 워크로드에 CMK가 적용되어 있는가? | ☐ |
| 11 | ** 비밀 관리**| 모든 비밀번호/API 키가 Secret Scope에 저장되어 있는가? | ☐ |
| 12 | ** 감사**| 시스템 테이블 기반 모니터링 대시보드가 있는가? | ☐ |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| ** 공유 책임**| Databricks가 플랫폼 보안, 고객이 데이터 접근 정책을 담당합니다 |
| **5계층 보안**| 네트워크 → 인증 → 인가 → 데이터 → 감사 순서로 보안을 구성합니다 |
| ** 암호화**| 전송 중(TLS), 저장 시(AES-256) 기본 활성화. CMK로 키를 직접 관리할 수 있습니다 |
| **Secret**| 민감한 정보를 안전하게 저장하고, 코드에서 참조합니다 |
| **Service Principal**| 프로덕션 자동화에서 PAT 대신 사용하는 서비스 계정입니다 |
| ** 감사 로그** | `system.access.audit` 테이블로 모든 활동을 추적합니다 |

---

## 참고 링크

- [Databricks: Security and compliance](https://docs.databricks.com/aws/en/security/)
- [Databricks: Trust Center](https://www.databricks.com/trust)
- [Databricks: Manage secrets](https://docs.databricks.com/aws/en/security/secrets/)
- [Databricks: Service principals](https://docs.databricks.com/aws/en/admin/users-groups/service-principals.html)
- [Databricks: Enhanced security monitoring](https://docs.databricks.com/aws/en/security/privacy/enhanced-security-monitoring.html)
- [Azure Databricks: Security](https://learn.microsoft.com/en-us/azure/databricks/security/)
