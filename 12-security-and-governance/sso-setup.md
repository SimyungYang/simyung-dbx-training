# SSO (Single Sign-On) 설정 상세 가이드

## SSO란?

**SSO(Single Sign-On)** 는 사용자가 기업의 ID 공급자(IdP)에 한 번 로그인하면, Databricks를 포함한 여러 서비스에 ** 추가 로그인 없이** 접근할 수 있는 인증 방식입니다.

> 💡 기본 개념은 [인증과 접근 제어](./identity-and-access.md)의 "SSO" 섹션에서 소개했습니다. 이 문서에서는 실제 설정 절차와 트러블슈팅을 상세히 다룹니다.

---

## SAML 2.0 vs OIDC

Databricks는 두 가지 SSO 프로토콜을 지원합니다.

| 항목 | SAML 2.0 | OIDC (OpenID Connect) |
|------|----------|----------------------|
| ** 기반 기술** | XML 기반 | OAuth 2.0 + JSON 기반 |
| ** 성숙도** | 업계 표준 (오래됨) | 최신 표준 |
| ** 토큰 형식** | XML Assertion | JWT (JSON Web Token) |
| ** 설정 복잡도** | 중간 | 낮음 |
| ** 지원 IdP** | 대부분 지원 | 대부분 지원 |
| ** 권장 환경** | 기존 SAML 인프라가 있는 경우 | 신규 설정, 클라우드 네이티브 환경 |

> 💡 ** 어떤 프로토콜을 선택해야 하나요?** 이미 SAML 기반 SSO가 구성된 기업이라면 SAML을 사용하세요. 새로 설정하는 경우 OIDC가 더 간단하고 현대적인 선택입니다.

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
4. Okta에서 제공하는 **IdP metadata URL** 을 입력합니다
   - 또는 **IdP Issuer**, **IdP SSO URL**, **Certificate** 를 수동 입력합니다

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
2. **IdP metadata URL** 에 Azure AD의 메타데이터 URL 붙여넣기
3. **Test SSO** 실행

---

## SSO 강제 활성화

SSO가 정상 동작하면, 비밀번호 기반 로그인을 비활성화하여 보안을 강화합니다.

> ⚠️ ** 주의**: SSO 강제 활성화 전에 반드시 다음을 확인하세요.
> 1. SSO 테스트가 성공했는지 확인
> 2. ** 긴급 복구용 Account Admin** 계정이 비밀번호 로그인 가능하도록 유지
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
| ** 사용자를 찾을 수 없음** | 이메일 매핑 오류 | Name ID format이 `EmailAddress`이고, 실제 이메일이 전송되는지 확인합니다 |
| ** 인증서 만료** | IdP SAML 인증서 만료 | IdP에서 새 인증서를 발급하고, Databricks에 업데이트합니다 |
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
| ** 긴급 계정 유지** | 최소 1개의 Account Admin 계정은 비밀번호 로그인을 유지합니다 |
| ** 인증서 모니터링** | SAML 인증서 만료일을 모니터링하고 사전 갱신합니다 |
| **SCIM 연동** | SSO와 함께 SCIM을 설정하여 사용자 프로비저닝을 자동화합니다 |

---

## SAML Assertion 구조 상세

SAML SSO가 동작할 때, IdP는 **SAML Assertion** 이라는 XML 문서를 생성하여 Databricks(SP, Service Provider)에 전달합니다. 이 Assertion에는 인증 결과와 사용자 속성이 포함됩니다.

### SAML Assertion 주요 구성 요소

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **Issuer** | IdP 식별자 | 이 Assertion을 발행한 IdP의 고유 URI |
| **Subject/NameID** | 사용자 식별 | 인증된 사용자의 이메일 주소 (Databricks는 이메일로 사용자 매칭) |
| **Conditions** | 유효 조건 | Assertion의 유효 시간(NotBefore, NotOnOrAfter), 대상 SP(AudienceRestriction) |
| **AuthnStatement** | 인증 정보 | 인증 시각, 인증 방법(Password, MFA 등), 세션 인덱스 |
| **AttributeStatement** | 사용자 속성 | 이름, 이메일, 부서 등 추가 속성 정보 |
| **Signature** | 디지털 서명 | IdP의 X.509 인증서로 서명. Assertion의 무결성과 출처를 검증 |

### SAML Assertion XML 구조 예시

```xml
<saml:Assertion xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                ID="_abc123" IssueInstant="2025-03-31T09:00:00Z">

  <!-- 발행자 (IdP) -->
  <saml:Issuer>https://idp.company.com/saml/metadata</saml:Issuer>

  <!-- 디지털 서명 -->
  <ds:Signature>
    <ds:SignedInfo>...</ds:SignedInfo>
    <ds:SignatureValue>...</ds:SignatureValue>
    <ds:KeyInfo>
      <ds:X509Data>
        <ds:X509Certificate>MIIDpDCCAoy...</ds:X509Certificate>
      </ds:X509Data>
    </ds:KeyInfo>
  </ds:Signature>

  <!-- 인증 대상 사용자 -->
  <saml:Subject>
    <saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
      kim.minjun@company.com
    </saml:NameID>
  </saml:Subject>

  <!-- 유효 조건 -->
  <saml:Conditions NotBefore="2025-03-31T08:59:00Z"
                   NotOnOrAfter="2025-03-31T09:10:00Z">
    <saml:AudienceRestriction>
      <saml:Audience>https://accounts.cloud.databricks.com</saml:Audience>
    </saml:AudienceRestriction>
  </saml:Conditions>

  <!-- 사용자 속성 -->
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>kim.minjun@company.com</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="firstName">
      <saml:AttributeValue>민준</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="lastName">
      <saml:AttributeValue>김</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

### SAML 디버깅 시 확인 포인트

| 확인 항목 | 문제 시 증상 | 해결 |
|-----------|------------|------|
| **NameID 형식** | "User not found" 오류 | NameID가 이메일 형식이고, Databricks에 등록된 이메일과 일치하는지 확인 |
| **Audience 값** | "Invalid audience" 오류 | `https://accounts.cloud.databricks.com`과 정확히 일치해야 합니다 |
| **Conditions 시간** | "Assertion expired" 또는 "Not yet valid" | IdP와 Databricks 서버 간 시간 차이(Clock Skew)를 확인합니다. NTP 동기화 필수 |
| **Signature 검증** | "Signature validation failed" | IdP 인증서가 Databricks에 등록된 것과 일치하는지 확인 |
| **ACS URL** | 로그인 후 빈 페이지 또는 404 | Reply URL이 `https://accounts.cloud.databricks.com/login/saml`인지 확인 |

---

## OIDC Token Flow 상세

OIDC는 OAuth 2.0 위에 인증 레이어를 추가한 프로토콜로, SAML보다 현대적이고 가벼운 방식입니다.

### OIDC Authorization Code Flow (Databricks 사용 방식)

| 단계 | 방향 | 설명 |
|------|------|------|
| 1 | 사용자 → Databricks (SP) | 로그인 요청 |
| 2 | Databricks → IdP | Authorization Request |
| 3 | IdP → 사용자 | 로그인 페이지 표시 (ID/PW + MFA) |
| 4 | 사용자 → IdP | 인증 정보 입력 |
| 5 | IdP → Databricks | Authorization Code 전달 |
| 6 | Databricks → IdP | Code를 Token으로 교환 (백채널) |
| 7 | IdP → Databricks | Access Token + ID Token 발급 |
| 8 | Databricks → 사용자 | 로그인 완료, Workspace 표시 |

### ID Token (JWT) 구조

OIDC에서 반환되는 ID Token은 JWT(JSON Web Token) 형식입니다.

```json
// JWT Header
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "abc123"
}

// JWT Payload (Claims)
{
  "iss": "https://login.microsoftonline.com/<tenant-id>/v2.0",
  "sub": "AAAAABBBBBcccccc...",
  "aud": "<databricks-client-id>",
  "exp": 1743412800,
  "iat": 1743409200,
  "email": "kim.minjun@company.com",
  "name": "김민준",
  "preferred_username": "kim.minjun@company.com"
}

// JWT Signature (RS256으로 서명)
```

| SAML vs OIDC 토큰 비교 | SAML Assertion | OIDC ID Token (JWT) |
|----------------------|----------------|---------------------|
| ** 포맷** | XML (수 KB) | JSON (수백 바이트) |
| ** 서명 알고리즘** | RSA-SHA256 (XML 서명) | RS256 (JWT 서명) |
| ** 전송 크기** | 크다 | 작다 |
| ** 파싱 복잡도** | XML 파서 필요 | JSON 파서 (더 간단) |
| ** 유효 기간 관리** | Conditions 요소 | exp/iat 클레임 |

---

## Conditional Access Policy (조건부 접근)

IdP 측에서 ** 조건부 접근 정책**을 설정하면, 특정 조건에서만 Databricks 접근을 허용할 수 있습니다. 이는 SSO를 넘어선 추가 보안 계층입니다.

### 주요 조건부 접근 시나리오

| 조건 | 정책 예시 | 적용 방법 |
|------|----------|----------|
| ** 위치 기반** | 사내 네트워크(VPN) 밖에서는 접근 차단 | Entra ID: Named Locations + Conditional Access Policy |
| ** 디바이스 상태** | 관리 디바이스(MDM 등록)에서만 접근 허용 | Intune + Conditional Access. 개인 기기 차단 |
| **MFA 강제** | 모든 Databricks 접근에 MFA 필수 | IdP에서 Databricks 앱에 MFA 정책 적용 |
| ** 위험 수준** | 비정상 로그인 패턴 감지 시 차단 | Entra ID Identity Protection: Risk-based Conditional Access |
| ** 시간 기반** | 업무 시간(9AM~6PM)에만 접근 허용 | IdP의 시간 조건 정책 |
| ** 역할 기반** | Admin 역할은 MFA + 관리 디바이스 필수, 일반 사용자는 MFA만 | 역할별 차등 정책 |

### Entra ID Conditional Access 설정 예시

```
정책 이름: "Databricks - MFA + 관리 디바이스 필수"

대상 앱: Databricks SSO (Enterprise Application)
대상 사용자: 모든 사용자
조건:
  - 디바이스 플랫폼: Windows, macOS
  - 위치: 모든 위치 (신뢰할 수 있는 위치 제외 시 더 엄격)
접근 제어:
  - MFA 필수
  - Intune 준수 디바이스 필수
  - 세션: 로그인 빈도 12시간 (12시간마다 재인증)
```

> 💡 ** 실무 팁**: Conditional Access Policy를 처음 적용할 때는 반드시 **"Report-only" 모드** 로 시작하세요. 실제 차단 없이 정책의 영향 범위를 확인한 후 Enforcement로 전환합니다. 갑작스러운 정책 적용은 전체 사용자의 Databricks 접근을 차단할 수 있습니다.

---

## SSO 장애 시 Emergency Access 방법

### Emergency Access 설계

SSO가 실패하면 전체 사용자가 Databricks에 접근할 수 없게 됩니다. 이를 대비한 Emergency Access 체계가 반드시 필요합니다.

| 대비 항목 | 설정 방법 | 주의사항 |
|-----------|----------|---------|
| **Break-glass 계정** | Account Admin 2개 이상 계정에 비밀번호 로그인 유지 | MFA도 IdP 독립적인 방식(TOTP) 사용. 비밀번호는 금고에 보관 |
| ** 비밀번호 로그인 허용** | Account Console > SSO > "Allow password login" | 특정 계정만 예외. 전체 허용은 보안 위험 |
| **Emergency Access 절차서** | SSO 장애 시 수행할 단계를 문서화 | 연락 체계, 에스컬레이션 경로 포함 |
| ** 정기 테스트** | 분기 1회 Emergency Access 로그인 테스트 | 계정이 유효한지, 비밀번호가 만료되지 않았는지 확인 |

### Emergency Access 절차

```
[SSO 장애 감지]
1. 사용자 로그인 실패 보고 확인
2. IdP 상태 페이지(예: status.okta.com) 확인
3. Emergency 여부 판단

[Emergency Access 실행]
1. Break-glass 계정으로 https://accounts.cloud.databricks.com 직접 접속
2. 이메일 + 비밀번호로 로그인 (SSO 우회)
3. Account Console > Settings > SSO에서 상태 확인
4. 필요 시 "Allow password login"을 일시적으로 Enabled 설정
5. 전체 사용자에게 비밀번호 재설정 링크 안내

[복구 후]
1. IdP 정상화 확인
2. SSO 테스트 수행
3. "Allow password login"을 다시 Disabled로 변경
4. 인시던트 보고서 작성
```

---

## 멀티 IdP 구성 패턴

대기업이나 M&A 이후 여러 IdP를 사용해야 하는 경우가 있습니다.

### Databricks의 멀티 IdP 제한

| 수준 | 지원 | 설명 |
|------|------|------|
| **Account SSO** | IdP 1개만 | Account Console에서 하나의 IdP만 설정 가능 |
| **Workspace SSO** | Workspace별 다른 IdP 가능 | 각 Workspace에 별도 SSO를 설정할 수 있습니다 (레거시 방식) |

### 멀티 IdP 대응 전략

| 전략 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **IdP 통합** | 하나의 마스터 IdP로 통합 | 가장 깔끔한 구조 | M&A 후 통합에 시간이 오래 걸림 |
| **IdP 페더레이션** | IdP 간 Trust를 설정하여 하나의 IdP를 프록시로 사용 | Account SSO 1개로 통합 가능 | IdP 설정이 복잡 |
| **Entra ID B2B** | Azure AD B2B 초대로 외부 사용자를 자사 테넌트에 추가 | Entra ID 하나로 관리 | 외부 사용자 경험이 다소 복잡 |
| **Workspace별 SSO** | 각 Workspace에 다른 IdP SSO 설정 (레거시) | 즉시 적용 가능 | Account 수준 관리 불가, Unity Catalog 그룹 관리 어려움 |

### IdP 페더레이션 예시 (Okta → Azure AD Trust)

| 회사 | IdP | 연동 방식 |
|------|-----|---------|
| 회사 A | Okta | SAML 2.0 + SCIM |
| 회사 B | Entra ID | OIDC + SCIM |

> 💡 **M&A 시나리오 권장**: 인수된 회사의 IdP를 마스터 IdP에 페더레이션으로 연결하고, Databricks Account SSO는 마스터 IdP 하나만 설정합니다. 시간이 지나 IdP 통합이 완료되면 페더레이션을 해제합니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SAML 2.0** | XML 기반 SSO 프로토콜. 대부분의 IdP에서 지원합니다 |
| **OIDC** | OAuth 2.0 기반 최신 SSO 프로토콜입니다 |
| **Account Console** | SSO 설정은 Account Console에서 수행합니다 |
| **SSO 강제** | 테스트 완료 후 비밀번호 로그인을 비활성화합니다 |
| ** 긴급 복구** | IdP 장애 대비 비밀번호 로그인 가능한 관리자 계정을 유지합니다 |
| **SAML Assertion** | XML 기반의 인증 토큰으로, 사용자 정보와 디지털 서명을 포함합니다 |
| **OIDC JWT** | JSON 기반의 경량 인증 토큰으로, SAML보다 파싱이 간편합니다 |
| **Conditional Access** | IdP 측에서 위치, 디바이스, MFA 등 조건부 접근을 제어합니다 |
| **Emergency Access** | Break-glass 계정으로 SSO 장애 시에도 관리 접근이 가능합니다 |
| ** 멀티 IdP** | Account SSO는 1개만 지원하므로 IdP 페더레이션으로 통합합니다 |

---

## 참고 링크

- [Databricks: Configure SSO](https://docs.databricks.com/aws/en/admin/account-settings/sso.html)
- [Databricks: SAML SSO](https://docs.databricks.com/aws/en/admin/account-settings/sso/saml.html)
- [Databricks: OIDC SSO](https://docs.databricks.com/aws/en/admin/account-settings/sso/oidc.html)
- [Azure Databricks: SSO with Azure AD](https://learn.microsoft.com/en-us/azure/databricks/admin/account-settings/sso)
