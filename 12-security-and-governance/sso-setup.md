# SSO (Single Sign-On) 설정 상세 가이드

## SSO란?

**SSO(Single Sign-On)**는 사용자가 기업의 ID 공급자(IdP)에 한 번 로그인하면, Databricks를 포함한 여러 서비스에 **추가 로그인 없이** 접근할 수 있는 인증 방식입니다.

> 💡 기본 개념은 [인증과 접근 제어](./identity-and-access.md)의 "SSO" 섹션에서 소개했습니다. 이 문서에서는 실제 설정 절차와 트러블슈팅을 상세히 다룹니다.

---

## SAML 2.0 vs OIDC

Databricks는 두 가지 SSO 프로토콜을 지원합니다.

| 항목 | SAML 2.0 | OIDC (OpenID Connect) |
|------|----------|----------------------|
| **기반 기술** | XML 기반 | OAuth 2.0 + JSON 기반 |
| **성숙도** | 업계 표준 (오래됨) | 최신 표준 |
| **토큰 형식** | XML Assertion | JWT (JSON Web Token) |
| **설정 복잡도** | 중간 | 낮음 |
| **지원 IdP** | 대부분 지원 | 대부분 지원 |
| **권장 환경** | 기존 SAML 인프라가 있는 경우 | 신규 설정, 클라우드 네이티브 환경 |

> 💡 **어떤 프로토콜을 선택해야 하나요?** 이미 SAML 기반 SSO가 구성된 기업이라면 SAML을 사용하세요. 새로 설정하는 경우 OIDC가 더 간단하고 현대적인 선택입니다.

---

## SSO 설정 흐름 개요

SSO 설정은 크게 3단계로 진행됩니다.

```
1. IdP에서 Databricks 앱 생성
   → SAML 메타데이터 또는 OIDC 클라이언트 정보 획득

2. Databricks Account Console에서 SSO 설정
   → IdP 정보 입력 및 연결

3. 테스트 및 강제 활성화
   → SSO 동작 확인 후 비밀번호 로그인 비활성화
```

---

## Okta SSO 설정 (SAML 2.0)

### 단계 1: Okta에서 Databricks 앱 생성

1. **Okta Admin Console** > **Applications** > **Create App Integration** 클릭
2. **Sign-in method**: `SAML 2.0` 선택
3. **App name**: `Databricks` 입력

### 단계 2: SAML 설정 입력

Okta의 SAML 설정에 다음 값을 입력합니다.

| 항목 | 값 |
|------|---|
| **Single sign-on URL** | `https://accounts.cloud.databricks.com/login/saml` |
| **Audience URI (SP Entity ID)** | `https://accounts.cloud.databricks.com` |
| **Name ID format** | `EmailAddress` |
| **Application username** | `Email` |

### 단계 3: Attribute Statements 설정

| Name | Value |
|------|-------|
| `email` | `user.email` |
| `firstName` | `user.firstName` |
| `lastName` | `user.lastName` |

### 단계 4: Databricks에서 SSO 구성

1. **Databricks Account Console** (https://accounts.cloud.databricks.com) 로그인
2. **Settings** > **Single sign-on** 이동
3. **SSO type**: `SAML 2.0` 선택
4. Okta에서 제공하는 **IdP metadata URL**을 입력합니다
   - 또는 **IdP Issuer**, **IdP SSO URL**, **Certificate**를 수동 입력합니다

### 단계 5: 테스트

1. **Test SSO** 버튼 클릭
2. 새 브라우저 탭에서 Okta 로그인 페이지가 열립니다
3. 로그인 성공 시 Databricks로 리디렉션됩니다
4. "SSO test successful" 메시지를 확인합니다

---

## Azure AD (Entra ID) SSO 설정

### 단계 1: Azure Portal에서 엔터프라이즈 앱 생성

1. **Azure Portal** > **Microsoft Entra ID** > **Enterprise applications** 이동
2. **New application** > **Create your own application** 클릭
3. 이름: `Databricks SSO` 입력

### 단계 2: SAML 설정

1. **Single sign-on** > **SAML** 선택
2. **Basic SAML Configuration** 편집:

| 항목 | 값 |
|------|---|
| **Identifier (Entity ID)** | `https://accounts.cloud.databricks.com` |
| **Reply URL (ACS URL)** | `https://accounts.cloud.databricks.com/login/saml` |
| **Sign on URL** | `https://accounts.cloud.databricks.com` |

### 단계 3: 사용자 속성 매핑

| Claim Name | Source Attribute |
|-----------|-----------------|
| `emailaddress` | `user.mail` |
| `givenname` | `user.givenname` |
| `surname` | `user.surname` |

### 단계 4: 인증서 및 URL 복사

**SAML Signing Certificate** 섹션에서:
- **App Federation Metadata URL** 복사

### 단계 5: Databricks에 설정

1. **Databricks Account Console** > **Settings** > **SSO**
2. **IdP metadata URL**에 Azure AD의 메타데이터 URL 붙여넣기
3. **Test SSO** 실행

---

## SSO 강제 활성화

SSO가 정상 동작하면, 비밀번호 기반 로그인을 비활성화하여 보안을 강화합니다.

> ⚠️ **주의**: SSO 강제 활성화 전에 반드시 다음을 확인하세요.
> 1. SSO 테스트가 성공했는지 확인
> 2. **긴급 복구용 Account Admin** 계정이 비밀번호 로그인 가능하도록 유지
> 3. IdP 장애 시 대비 계획 수립

### 강제 활성화 설정

1. **Account Console** > **Settings** > **Single sign-on**
2. **Allow password login**: `Disabled` 설정
3. **Emergency access**: 최소 1개의 Account Admin 계정은 비밀번호 로그인을 유지합니다

---

## OIDC SSO 설정

OIDC 기반 SSO는 SAML보다 설정이 간단합니다.

### Databricks 설정

1. **Account Console** > **Settings** > **SSO**
2. **SSO type**: `OpenID Connect` 선택
3. 다음 정보를 입력합니다:

| 항목 | 설명 |
|------|------|
| **Issuer URL** | IdP의 OIDC discovery URL (예: `https://login.microsoftonline.com/<tenant-id>/v2.0`) |
| **Client ID** | IdP에서 발급한 클라이언트 ID |
| **Client Secret** | IdP에서 발급한 클라이언트 시크릿 |

---

## 트러블슈팅

### 자주 발생하는 문제

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| **SAML Response 유효하지 않음** | ACS URL이 일치하지 않음 | IdP의 Reply URL이 `https://accounts.cloud.databricks.com/login/saml`인지 확인합니다 |
| **사용자를 찾을 수 없음** | 이메일 매핑 오류 | Name ID format이 `EmailAddress`이고, 실제 이메일이 전송되는지 확인합니다 |
| **인증서 만료** | IdP SAML 인증서 만료 | IdP에서 새 인증서를 발급하고, Databricks에 업데이트합니다 |
| **SSO 루프** | ACS URL 중복 설정 | IdP에 등록된 Databricks 앱이 하나인지 확인합니다 |
| **OIDC 토큰 오류** | Client Secret 불일치 | IdP에서 새 Secret을 생성하고 Databricks에 업데이트합니다 |

### 디버깅 방법

```sql
-- 감사 로그에서 SSO 관련 이벤트 조회
SELECT
  event_time,
  user_identity.email,
  action_name,
  response.status_code,
  response.error_message,
  source_ip_address
FROM system.access.audit
WHERE action_name LIKE '%sso%' OR action_name LIKE '%login%'
  AND event_time > CURRENT_TIMESTAMP() - INTERVAL 1 HOUR
ORDER BY event_time DESC;
```

---

## 모범 사례

| 원칙 | 설명 |
|------|------|
| **SSO 필수화** | 테스트 완료 후 비밀번호 로그인을 비활성화합니다 |
| **MFA 연동** | IdP 측에서 MFA를 강제하여 이중 인증을 적용합니다 |
| **긴급 계정 유지** | 최소 1개의 Account Admin 계정은 비밀번호 로그인을 유지합니다 |
| **인증서 모니터링** | SAML 인증서 만료일을 모니터링하고 사전 갱신합니다 |
| **SCIM 연동** | SSO와 함께 SCIM을 설정하여 사용자 프로비저닝을 자동화합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SAML 2.0** | XML 기반 SSO 프로토콜. 대부분의 IdP에서 지원합니다 |
| **OIDC** | OAuth 2.0 기반 최신 SSO 프로토콜입니다 |
| **Account Console** | SSO 설정은 Account Console에서 수행합니다 |
| **SSO 강제** | 테스트 완료 후 비밀번호 로그인을 비활성화합니다 |
| **긴급 복구** | IdP 장애 대비 비밀번호 로그인 가능한 관리자 계정을 유지합니다 |

---

## 참고 링크

- [Databricks: Configure SSO](https://docs.databricks.com/aws/en/admin/account-settings/sso.html)
- [Databricks: SAML SSO](https://docs.databricks.com/aws/en/admin/account-settings/sso/saml.html)
- [Databricks: OIDC SSO](https://docs.databricks.com/aws/en/admin/account-settings/sso/oidc.html)
- [Azure Databricks: SSO with Azure AD](https://learn.microsoft.com/en-us/azure/databricks/admin/account-settings/sso)
