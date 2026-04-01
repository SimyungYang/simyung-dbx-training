# SCIM 프로비저닝 상세

## SCIM이란?

**SCIM(System for Cross-domain Identity Management)**은 기업의 ID 공급자(IdP)에서 Databricks로 사용자와 그룹 정보를 **자동 동기화** 하는 표준 프로토콜입니다.

> 💡 기본 개념은 [인증과 접근 제어](./identity-and-access.md)의 "SCIM 프로비저닝" 섹션에서 소개했습니다. 이 문서에서는 실제 설정 절차와 고급 구성을 다룹니다.

---

## SCIM이 해결하는 문제

| 상황 | SCIM 없이 (수동) | SCIM으로 (자동) |
|------|----------------|----------------|
| **신규 입사**| 관리자가 Databricks에 수동 추가 | IdP에 추가하면 자동 동기화 |
| **퇴사**| 관리자가 수동 비활성화 (누락 위험) | IdP에서 비활성화하면 자동 반영 |
| **그룹 변경**| 수동 그룹 멤버십 변경 | IdP 그룹 변경이 자동 반영 |
| **부서 이동**| 여러 시스템에서 개별 변경 | IdP 한 곳에서 변경하면 전파 |

---

## Account 수준 vs Workspace 수준 SCIM

Databricks에서는 **두 수준** 의 SCIM을 제공합니다.

| 항목 | Account 수준 SCIM | Workspace 수준 SCIM |
|------|-------------------|---------------------|
| **관리 범위**| 전체 Account의 사용자/그룹 | 개별 Workspace의 사용자/그룹 |
| **SCIM 엔드포인트**| `accounts.cloud.databricks.com/api/2.0/accounts/<id>/scim/v2` | `<workspace-url>/api/2.0/preview/scim/v2` |
| **Unity Catalog 호환**| 완전 호환 | 제한적 |
| **권장 여부**| **강력 권장**| 레거시 (비권장) |

> 💡 **항상 Account 수준 SCIM을 사용하세요.**Account 수준에서 동기화된 사용자/그룹은 ID Federation을 통해 모든 워크스페이스에서 사용할 수 있습니다. Unity Catalog의 GRANT 문에도 Account 그룹만 사용할 수 있습니다.

---

## 동기화 대상

SCIM을 통해 동기화되는 정보는 다음과 같습니다.

| 동기화 항목 | 설명 | IdP → Databricks 방향 |
|------------|------|---------------------|
| **사용자 생성**| IdP에서 Databricks 앱에 사용자를 할당하면 계정이 생성됩니다 | 단방향 |
| **사용자 비활성화**| IdP에서 사용자를 비활성화하면 Databricks에서도 비활성화됩니다 | 단방향 |
| **사용자 속성**| 이름, 이메일, 부서 등 프로필 정보가 동기화됩니다 | 단방향 |
| **그룹 생성**| IdP의 그룹이 Databricks Account 그룹으로 생성됩니다 | 단방향 |
| **그룹 멤버십**| 그룹의 멤버 추가/제거가 자동 반영됩니다 | 단방향 |

> ⚠️ SCIM은 **IdP → Databricks 단방향** 동기화입니다. Databricks에서 직접 변경한 내용은 다음 동기화 시 IdP의 정보로 덮어씌워질 수 있습니다.

---

## Okta SCIM 설정

### 사전 준비

1. Databricks Account Admin 권한 필요
2. Okta Admin 권한 필요

### 단계 1: Databricks에서 SCIM 토큰 생성

1. **Databricks Account Console**> **Settings**> **User provisioning**
2. **Enable user provisioning** 클릭
3. **Generate token** 클릭하여 SCIM 토큰을 복사합니다

```
SCIM Base URL: https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2
Bearer Token:  dapi_xxxxxxxxxxxxxxxxxxxxxxxxx
```

### 단계 2: Okta에서 SCIM 앱 설정

1. **Okta Admin Console**> **Applications**> Databricks 앱 선택
2. **Provisioning**탭 > **Configure API Integration** 클릭
3. **Enable API Integration** 체크

| 항목 | 값 |
|------|---|
| **SCIM connector base URL**| `https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2` |
| **API Token**| 위에서 생성한 SCIM 토큰 |

4. **Test API Credentials** 클릭하여 연결 확인

### 단계 3: 프로비저닝 설정

**To App** 설정에서 다음을 활성화합니다:

| 기능 | 설명 | 권장 |
|------|------|------|
| **Create Users**| 새 사용자 자동 생성 | ✅ 활성화 |
| **Update User Attributes**| 프로필 변경 자동 반영 | ✅ 활성화 |
| **Deactivate Users**| 사용자 비활성화 자동 반영 | ✅ 활성화 |

### 단계 4: 사용자/그룹 할당

1. **Assignments** 탭에서 동기화할 사용자/그룹을 선택합니다
2. **Push Groups** 탭에서 Databricks로 동기화할 그룹을 설정합니다

---

## Azure AD (Entra ID) SCIM 설정

### 단계 1: Databricks SCIM 토큰 생성

Okta 설정과 동일하게 Account Console에서 SCIM 토큰을 생성합니다.

### 단계 2: Azure Portal에서 프로비저닝 설정

1. **Azure Portal**> **Microsoft Entra ID**> **Enterprise applications**
2. Databricks 앱 선택 > **Provisioning**> **Get started**
3. **Provisioning Mode**: `Automatic` 선택
4. **Admin Credentials** 설정:

| 항목 | 값 |
|------|---|
| **Tenant URL**| `https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2` |
| **Secret Token**| Databricks SCIM 토큰 |

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
| **토큰 갱신**| SCIM 토큰은 만료 기간이 있습니다. 만료 전 갱신해야 동기화가 중단되지 않습니다 |
| **삭제 vs 비활성화**| SCIM은 사용자를 **삭제하지 않고 비활성화** 합니다. 완전 삭제는 수동으로 처리합니다 |
| **그룹 이름 변경**| IdP에서 그룹 이름을 변경하면, Databricks에서 새 그룹으로 인식될 수 있습니다 |
| **동기화 지연**| IdP에서 변경 후 Databricks에 반영되기까지 최대 40분이 소요될 수 있습니다 |
| **중복 방지**| Account SCIM과 Workspace SCIM을 동시에 사용하지 마세요 |

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **Account 수준 SCIM**| 반드시 Account 수준에서 SCIM을 설정합니다 |
| **그룹 기반 할당**| 개별 사용자가 아닌 그룹 단위로 동기화합니다 |
| **네이밍 규칙**| IdP 그룹 이름을 Databricks 역할과 일치시킵니다 (예: `dbx-data-engineers`) |
| **토큰 모니터링**| SCIM 토큰 만료 알림을 설정합니다 |
| **정기 감사**| 동기화 상태를 주기적으로 확인합니다 |

---

## SCIM 프로토콜 내부 동작 상세

SCIM은 **REST API 기반의 CRUD(Create, Read, Update, Delete)** 프로토콜입니다. IdP가 Databricks의 SCIM 엔드포인트에 HTTP 요청을 보내는 방식으로 동기화가 이루어집니다.

### SCIM REST API 구조

| HTTP 메서드 | 엔드포인트 | 동작 | 예시 |
|------------|-----------|------|------|
| **POST**| `/scim/v2/Users` | 사용자 생성 | 신규 입사자 계정 생성 |
| **GET**| `/scim/v2/Users?filter=userName eq "user@company.com"` | 사용자 조회 | 동기화 전 존재 여부 확인 |
| **PUT**| `/scim/v2/Users/{id}` | 사용자 전체 업데이트 | 프로필 정보 전체 교체 |
| **PATCH**| `/scim/v2/Users/{id}` | 사용자 부분 업데이트 | 특정 속성만 변경 (활성/비활성) |
| **DELETE**| `/scim/v2/Users/{id}` | 사용자 삭제 | 완전 삭제 (대부분 IdP는 PATCH로 비활성화) |
| **POST**| `/scim/v2/Groups` | 그룹 생성 | 새 팀/역할 그룹 생성 |
| **PATCH**| `/scim/v2/Groups/{id}` | 그룹 멤버 변경 | 멤버 추가/제거 |

### SCIM 동기화 사이클 상세

IdP는 다음과 같은 사이클로 Databricks와 동기화합니다.

```
[초기 동기화 (Full Sync)]
1. IdP가 할당된 모든 사용자/그룹 목록을 생성
2. 각 사용자에 대해 GET → 존재하지 않으면 POST (생성)
3. 각 그룹에 대해 GET → 존재하지 않으면 POST (생성)
4. 그룹 멤버십을 PATCH로 설정
→ 수천 명 규모에서 20~40분 소요

[증분 동기화 (Incremental Sync)]
1. IdP가 마지막 동기화 이후 변경된 항목만 감지
2. 변경된 사용자: PATCH로 속성 업데이트
3. 비활성화된 사용자: PATCH로 active=false 설정
4. 그룹 변경: PATCH로 멤버 추가/제거
→ 변경 건수에 비례하여 수초~수분 소요
```

### SCIM 요청/응답 예시

```json
// POST /scim/v2/Users - 사용자 생성 요청
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "userName": "kim.minjun@company.com",
  "name": {
    "givenName": "민준",
    "familyName": "김"
  },
  "emails": [
    {
      "value": "kim.minjun@company.com",
      "type": "work",
      "primary": true
    }
  ],
  "active": true
}

// 응답: 201 Created
{
  "id": "1234567890",
  "userName": "kim.minjun@company.com",
  "active": true,
  ...
}
```

```json
// PATCH /scim/v2/Users/{id} - 사용자 비활성화 요청
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [
    {
      "op": "replace",
      "path": "active",
      "value": false
    }
  ]
}
```

---

## 대규모 조직(만 명 이상) SCIM 운영 시 고려사항

### 스케일링 문제와 해결

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| **초기 동기화 타임아웃**| 수만 명의 사용자를 한 번에 동기화하면 API 호출 한도 초과 | 그룹 단위로 분할하여 순차 동기화. 먼저 핵심 그룹부터 동기화 |
| **API Rate Limiting**| Databricks SCIM 엔드포인트의 초당 호출 제한에 도달 | IdP의 프로비저닝 속도 조절(Okta: provisioning rate limit 설정) |
| **동기화 지연 누적**| 변경 건수가 많으면 증분 동기화 시간이 길어짐 | IdP의 동기화 간격을 변경 빈도에 맞게 조정 |
| **토큰 만료**| 대규모 동기화 중 SCIM 토큰이 만료됨 | 토큰 수명을 충분히 길게 설정(최소 1년), 자동 갱신 프로세스 구축 |

### 동기화 범위 최적화

만 명 이상의 조직에서 모든 사용자를 Databricks에 동기화하는 것은 비효율적입니다.

| 전략 | 설명 | 적용 방법 |
|------|------|----------|
| **그룹 기반 필터링**| Databricks를 사용하는 그룹만 동기화합니다 | IdP에서 `dbx-users` 그룹에 속한 사용자만 Databricks 앱에 할당 |
| **역할 기반 필터링**| 데이터 관련 역할만 동기화합니다 | Data Engineer, Data Scientist, Analyst 역할만 포함 |
| **부서 기반 필터링**| 특정 부서만 동기화합니다 | 데이터팀, AI팀, 분석팀 등 |

> 💡 **실무 권장**: 전체 직원 5만 명 중 Databricks 실사용자가 500명이라면, 해당 500명이 속한 그룹만 동기화하세요. 불필요한 사용자 동기화는 API 호출 낭비이며, Databricks Account의 사용자 관리 화면도 복잡해집니다.

---

## 동기화 지연과 캐시

### 동기화 지연 구조

SCIM 동기화에서 발생하는 지연은 여러 단계에서 누적됩니다.

```
[IdP에서 변경 감지] → [SCIM API 호출] → [Databricks 반영] → [워크스페이스 반영]
   0~5분              0~40분           즉시              0~5분

총 지연: 최대 50분 (Entra ID 기준)
```

| IdP | 증분 동기화 간격 | 최대 지연 |
|-----|---------------|----------|
| **Okta**| 즉시~5분 (Push 방식) | 5~10분 |
| **Entra ID (Azure AD)**| 40분 고정 사이클 | 40~50분 |
| **OneLogin**| 즉시~10분 | 10~15분 |
| **Ping Identity**| 설정 가능 (5~60분) | 설정값 + 5분 |

### 긴급 상황에서의 수동 동기화

IdP의 자동 동기화를 기다릴 수 없는 긴급 상황(즉시 접근 차단이 필요한 보안 사고 등)에서는 Databricks API를 직접 호출합니다.

```bash
# 긴급 사용자 비활성화 (SCIM 동기화를 기다리지 않고 즉시 실행)
curl -X PATCH \
  "https://accounts.cloud.databricks.com/api/2.0/accounts/<account-id>/scim/v2/Users/<user-id>" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
    "Operations": [{"op": "replace", "path": "active", "value": false}]
  }'
```

> ⚠️ **주의**: Databricks에서 직접 변경한 내용은 다음 IdP 동기화 시 IdP의 상태로 덮어써질 수 있습니다. 반드시 **IdP에서도 동일하게 비활성화** 해야 합니다.

---

## 그룹 중첩(Nested Groups) 한계

### SCIM에서의 그룹 중첩 지원

| IdP | 중첩 그룹 지원 | Databricks 반영 |
|-----|-------------|----------------|
| **Entra ID**| IdP에서 중첩 가능 | SCIM은 **평면화(flatten)하여 전달**. 중첩 구조 유실 |
| **Okta**| Push Groups에서 중첩 불가 | 각 그룹을 개별적으로 Push해야 합니다 |
| **Databricks Account**| Account 그룹 내 그룹 중첩 가능 | 단, SCIM으로 자동 생성된 중첩은 지원 불안정 |

### 그룹 설계 모범 사례

**권장하지 않는 패턴**(중첩 의존):

| IdP 그룹 | 하위 그룹 | 문제 |
|----------|---------|------|
| Engineering | Data Engineering | 중첩 그룹은 SCIM 동기화 시 문제 발생 가능 |
| | - DE Bronze Team | - |
| | - DE Gold Team | - |
| | ML Engineering | - |
| | - MLE Training Team | - |
| | - MLE Serving Team | - |

> 💡 **실무 권장**: IdP의 조직 구조(부서별 중첩 그룹)와 Databricks의 접근 제어 그룹을 **분리하여 설계** 하세요. IdP에서 `dbx-` 접두사가 붙은 평면 그룹을 별도로 생성하고, 이 그룹만 SCIM으로 동기화하면 중첩 문제를 회피할 수 있습니다.

---

## IdP 장애 시 영향과 대응

### IdP 장애 시나리오별 영향

| 장애 시나리오 | SSO 영향 | SCIM 영향 | 기존 세션 |
|-------------|---------|----------|----------|
| **IdP 완전 다운**| 신규 로그인 불가 | 동기화 중단 | 기존 로그인 세션은 유지 |
| **IdP 부분 장애 (SCIM만)**| SSO 정상 | 동기화 중단 | 영향 없음 |
| **SCIM 토큰 만료**| SSO 정상 | 동기화 중단 | 영향 없음 |
| **IdP 인증서 만료**| SSO 실패 | SCIM은 토큰 기반이므로 정상 | 기존 세션 유지 |

### IdP 장애 대응 방안

| 대응 | 설명 | 사전 준비 |
|------|------|----------|
| **Emergency Access 계정**| 비밀번호 로그인이 가능한 Account Admin 계정 유지 | SSO 강제 활성화 시에도 예외 계정 설정 |
| **SCIM 동기화 모니터링**| 동기화 실패 시 알림 설정 | 감사 로그에서 SCIM 오류를 모니터링하는 알림 구축 |
| **토큰 갱신 자동화**| SCIM 토큰 만료 전 자동 갱신 | 토큰 만료일 모니터링 + 자동 갱신 스크립트 |
| **수동 사용자 관리 절차서**| IdP 장애 시 수동 관리 프로세스 | Databricks API를 사용한 긴급 사용자 관리 매뉴얼 |

```sql
-- SCIM 동기화 상태 모니터링 쿼리
-- 최근 24시간 내 SCIM 오류 확인
SELECT
  event_time,
  action_name,
  request_params.endpoint AS scim_endpoint,
  response.status_code,
  response.error_message
FROM system.access.audit
WHERE service_name = 'accounts'
  AND action_name LIKE '%scim%'
  AND response.status_code >= 400
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 24 HOURS
ORDER BY event_time DESC;

-- SCIM 동기화 정상 동작 여부 확인 (마지막 성공 시점)
SELECT
  MAX(event_time) AS last_successful_sync
FROM system.access.audit
WHERE service_name = 'accounts'
  AND action_name LIKE '%scim%'
  AND response.status_code BETWEEN 200 AND 299;
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SCIM**| 사용자/그룹을 IdP에서 Databricks로 자동 동기화하는 표준 프로토콜입니다 |
| **Account 수준**| Account 수준 SCIM을 사용하여 모든 워크스페이스에 적용합니다 |
| **단방향**| IdP → Databricks 방향으로만 동기화됩니다 |
| **비활성화**| 퇴사자를 IdP에서 비활성화하면 Databricks에서도 자동 비활성화됩니다 |
| **그룹 동기화**| IdP의 그룹 멤버십이 Databricks Account 그룹에 반영됩니다 |
| **REST API 기반**| SCIM은 HTTP POST/GET/PUT/PATCH/DELETE로 동작하는 REST 프로토콜입니다 |
| **동기화 지연**| IdP에 따라 5~50분의 지연이 발생합니다. 긴급 시 API 직접 호출로 해결합니다 |
| **그룹 중첩 한계**| SCIM은 중첩 그룹을 평면화하므로, 평면 그룹 구조를 권장합니다 |
| **IdP 장애 대응** | Emergency Access 계정과 모니터링 알림을 사전에 구축해야 합니다 |

---

## 참고 링크

- [Databricks: SCIM provisioning](https://docs.databricks.com/aws/en/admin/users-groups/scim/)
- [Databricks: Account-level SCIM](https://docs.databricks.com/aws/en/admin/users-groups/scim/aad.html)
- [Databricks: Okta SCIM](https://docs.databricks.com/aws/en/admin/users-groups/scim/okta.html)
- [Azure Databricks: Provision users with SCIM](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/scim/)
