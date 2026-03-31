# SCIM 프로비저닝 상세

## SCIM이란?

**SCIM(System for Cross-domain Identity Management)** 은 기업의 ID 공급자(IdP)에서 Databricks로 사용자와 그룹 정보를 **자동 동기화**하는 표준 프로토콜입니다.

> 💡 기본 개념은 [인증과 접근 제어](./identity-and-access.md)의 "SCIM 프로비저닝" 섹션에서 소개했습니다. 이 문서에서는 실제 설정 절차와 고급 구성을 다룹니다.

---

## SCIM이 해결하는 문제

| 상황 | SCIM 없이 (수동) | SCIM으로 (자동) |
|------|----------------|----------------|
| **신규 입사** | 관리자가 Databricks에 수동 추가 | IdP에 추가하면 자동 동기화 |
| **퇴사** | 관리자가 수동 비활성화 (누락 위험) | IdP에서 비활성화하면 자동 반영 |
| **그룹 변경** | 수동 그룹 멤버십 변경 | IdP 그룹 변경이 자동 반영 |
| **부서 이동** | 여러 시스템에서 개별 변경 | IdP 한 곳에서 변경하면 전파 |

---

## Account 수준 vs Workspace 수준 SCIM

Databricks에서는 **두 수준**의 SCIM을 제공합니다.

| 항목 | Account 수준 SCIM | Workspace 수준 SCIM |
|------|-------------------|---------------------|
| **관리 범위** | 전체 Account의 사용자/그룹 | 개별 Workspace의 사용자/그룹 |
| **SCIM 엔드포인트** | `accounts.cloud.databricks.com/api/2.0/accounts/<id>/scim/v2` | `<workspace-url>/api/2.0/preview/scim/v2` |
| **Unity Catalog 호환** | 완전 호환 | 제한적 |
| **권장 여부** | **강력 권장** | 레거시 (비권장) |

> 💡 **항상 Account 수준 SCIM을 사용하세요.** Account 수준에서 동기화된 사용자/그룹은 ID Federation을 통해 모든 워크스페이스에서 사용할 수 있습니다. Unity Catalog의 GRANT 문에도 Account 그룹만 사용할 수 있습니다.

---

## 동기화 대상

SCIM을 통해 동기화되는 정보는 다음과 같습니다.

| 동기화 항목 | 설명 | IdP → Databricks 방향 |
|------------|------|---------------------|
| **사용자 생성** | IdP에서 Databricks 앱에 사용자를 할당하면 계정이 생성됩니다 | 단방향 |
| **사용자 비활성화** | IdP에서 사용자를 비활성화하면 Databricks에서도 비활성화됩니다 | 단방향 |
| **사용자 속성** | 이름, 이메일, 부서 등 프로필 정보가 동기화됩니다 | 단방향 |
| **그룹 생성** | IdP의 그룹이 Databricks Account 그룹으로 생성됩니다 | 단방향 |
| **그룹 멤버십** | 그룹의 멤버 추가/제거가 자동 반영됩니다 | 단방향 |

> ⚠️ SCIM은 **IdP → Databricks 단방향** 동기화입니다. Databricks에서 직접 변경한 내용은 다음 동기화 시 IdP의 정보로 덮어씌워질 수 있습니다.

---

## Okta SCIM 설정

### 사전 준비

1. Databricks Account Admin 권한 필요
2. Okta Admin 권한 필요

### 단계 1: Databricks에서 SCIM 토큰 생성

1. **Databricks Account Console** > **Settings** > **User provisioning**
2. **Enable user provisioning** 클릭
3. **Generate token** 클릭하여 SCIM 토큰을 복사합니다

```
SCIM Base URL: https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2
Bearer Token:  dapi_xxxxxxxxxxxxxxxxxxxxxxxxx
```

### 단계 2: Okta에서 SCIM 앱 설정

1. **Okta Admin Console** > **Applications** > Databricks 앱 선택
2. **Provisioning** 탭 > **Configure API Integration** 클릭
3. **Enable API Integration** 체크

| 항목 | 값 |
|------|---|
| **SCIM connector base URL** | `https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2` |
| **API Token** | 위에서 생성한 SCIM 토큰 |

4. **Test API Credentials** 클릭하여 연결 확인

### 단계 3: 프로비저닝 설정

**To App** 설정에서 다음을 활성화합니다:

| 기능 | 설명 | 권장 |
|------|------|------|
| **Create Users** | 새 사용자 자동 생성 | ✅ 활성화 |
| **Update User Attributes** | 프로필 변경 자동 반영 | ✅ 활성화 |
| **Deactivate Users** | 사용자 비활성화 자동 반영 | ✅ 활성화 |

### 단계 4: 사용자/그룹 할당

1. **Assignments** 탭에서 동기화할 사용자/그룹을 선택합니다
2. **Push Groups** 탭에서 Databricks로 동기화할 그룹을 설정합니다

---

## Azure AD (Entra ID) SCIM 설정

### 단계 1: Databricks SCIM 토큰 생성

Okta 설정과 동일하게 Account Console에서 SCIM 토큰을 생성합니다.

### 단계 2: Azure Portal에서 프로비저닝 설정

1. **Azure Portal** > **Microsoft Entra ID** > **Enterprise applications**
2. Databricks 앱 선택 > **Provisioning** > **Get started**
3. **Provisioning Mode**: `Automatic` 선택
4. **Admin Credentials** 설정:

| 항목 | 값 |
|------|---|
| **Tenant URL** | `https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2` |
| **Secret Token** | Databricks SCIM 토큰 |

5. **Test Connection** 클릭하여 연결 확인

### 단계 3: 매핑 설정

기본 매핑을 확인하고 필요시 조정합니다:

| Azure AD 속성 | Databricks 속성 |
|--------------|----------------|
| `userPrincipalName` | `userName` |
| `mail` | `emails[type eq "work"].value` |
| `givenName` | `name.givenName` |
| `surname` | `name.familyName` |
| `Switch([IsSoftDeleted], ...)` | `active` |

### 단계 4: 프로비저닝 시작

1. **Provisioning Status**: `On`으로 변경
2. 초기 동기화는 최대 40분이 소요될 수 있습니다
3. 이후 약 40분 간격으로 증분 동기화가 실행됩니다

---

## 동기화 모니터링

### 감사 로그로 확인

```sql
-- SCIM 프로비저닝 이벤트 조회
SELECT
  event_time,
  action_name,
  request_params,
  response.status_code
FROM system.access.audit
WHERE service_name = 'accounts'
  AND action_name LIKE '%scim%'
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 24 HOURS
ORDER BY event_time DESC;
```

### CLI로 사용자/그룹 확인

```bash
# 동기화된 사용자 목록 확인
databricks account users list

# 동기화된 그룹 목록 확인
databricks account groups list

# 특정 그룹의 멤버 확인
databricks account groups get <group-id>
```

---

## 주의사항

| 항목 | 설명 |
|------|------|
| **토큰 갱신** | SCIM 토큰은 만료 기간이 있습니다. 만료 전 갱신해야 동기화가 중단되지 않습니다 |
| **삭제 vs 비활성화** | SCIM은 사용자를 **삭제하지 않고 비활성화**합니다. 완전 삭제는 수동으로 처리합니다 |
| **그룹 이름 변경** | IdP에서 그룹 이름을 변경하면, Databricks에서 새 그룹으로 인식될 수 있습니다 |
| **동기화 지연** | IdP에서 변경 후 Databricks에 반영되기까지 최대 40분이 소요될 수 있습니다 |
| **중복 방지** | Account SCIM과 Workspace SCIM을 동시에 사용하지 마세요 |

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **Account 수준 SCIM** | 반드시 Account 수준에서 SCIM을 설정합니다 |
| **그룹 기반 할당** | 개별 사용자가 아닌 그룹 단위로 동기화합니다 |
| **네이밍 규칙** | IdP 그룹 이름을 Databricks 역할과 일치시킵니다 (예: `dbx-data-engineers`) |
| **토큰 모니터링** | SCIM 토큰 만료 알림을 설정합니다 |
| **정기 감사** | 동기화 상태를 주기적으로 확인합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SCIM** | 사용자/그룹을 IdP에서 Databricks로 자동 동기화하는 표준 프로토콜입니다 |
| **Account 수준** | Account 수준 SCIM을 사용하여 모든 워크스페이스에 적용합니다 |
| **단방향** | IdP → Databricks 방향으로만 동기화됩니다 |
| **비활성화** | 퇴사자를 IdP에서 비활성화하면 Databricks에서도 자동 비활성화됩니다 |
| **그룹 동기화** | IdP의 그룹 멤버십이 Databricks Account 그룹에 반영됩니다 |

---

## 참고 링크

- [Databricks: SCIM provisioning](https://docs.databricks.com/aws/en/admin/users-groups/scim/)
- [Databricks: Account-level SCIM](https://docs.databricks.com/aws/en/admin/users-groups/scim/aad.html)
- [Databricks: Okta SCIM](https://docs.databricks.com/aws/en/admin/users-groups/scim/okta.html)
- [Azure Databricks: Provision users with SCIM](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/scim/)
