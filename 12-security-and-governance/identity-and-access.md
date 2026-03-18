# 인증과 접근 제어

## 인증 (Authentication)

| 방식 | 설명 | 적합한 사용 |
|------|------|-----------|
| **SSO (Single Sign-On)** | 기업 IdP(Okta, Azure AD 등)와 연동 | 사용자 로그인 |
| **MFA** | 다중 인증 (2단계 인증) | 보안 강화 |
| **PAT (Personal Access Token)** | 개인 토큰 기반 인증 | API, CLI 호출 |
| **OAuth (M2M)** | 서비스 간 인증 | 자동화, CI/CD |
| **Service Principal** | 서비스 계정 (사람이 아닌 시스템용) | 프로덕션 파이프라인 |

> 🆕 **Scoped Personal Access Tokens**: 토큰의 권한 범위를 API 유형별로 제한할 수 있는 기능이 Preview로 출시되었습니다.

---

## SCIM 프로비저닝

> 💡 **SCIM(System for Cross-domain Identity Management)**은 사용자와 그룹 정보를 IdP에서 Databricks로 **자동 동기화**하는 표준 프로토콜입니다. Okta, Azure AD에서 사용자를 추가/삭제하면 Databricks에도 자동으로 반영됩니다.

---

## 참고 링크

- [Databricks: Authentication](https://docs.databricks.com/aws/en/security/auth/)
