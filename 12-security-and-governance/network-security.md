# 네트워크 보안

## 네트워크 보안이 왜 중요한가요?

데이터 플랫폼은 대량의 민감한 데이터를 다루기 때문에, 외부로부터의 무단 접근을 차단하는 **네트워크 수준의 보안**이 필수적입니다.

---

## VPC / VNet

> 💡 **VPC(Virtual Private Cloud)**는 클라우드 안에서 논리적으로 격리된 네트워크 공간입니다. Databricks Data Plane은 고객의 VPC 안에서 실행됩니다.

| 방식 | 설명 |
|------|------|
| **Databricks-Managed VPC** | Databricks가 자동 생성·관리합니다 |
| **Customer-Managed VPC** | 고객이 직접 생성하고 Databricks에 연결합니다 |

## Private Link

> 💡 **Private Link**는 Databricks와의 통신을 퍼블릭 인터넷이 아닌 **클라우드 내부 네트워크**를 통해 수행합니다.

| 유형 | 경로 |
|------|------|
| **Front-end** | 사용자 → Control Plane |
| **Back-end** | Control Plane → Data Plane |

## IP Access List

특정 IP 주소에서만 Workspace에 접근할 수 있도록 제한합니다.

## 데이터 암호화

| 유형 | 설명 | 기본 |
|------|------|------|
| 전송 중 (In-Transit) | TLS 1.2+ | ✅ |
| 저장 시 (At-Rest) | AES-256 | ✅ |
| 고객 관리 키 (CMK) | 고객 소유 키로 암호화 | 선택 |

---

## 참고 링크

- [Databricks: Network security](https://docs.databricks.com/aws/en/security/network/)
- [Databricks: Private Link](https://docs.databricks.com/aws/en/security/network/classic/privatelink.html)
