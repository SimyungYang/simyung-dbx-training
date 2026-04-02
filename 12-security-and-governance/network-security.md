# 네트워크 보안

## 네트워크 보안이 왜 중요한가요?

데이터 플랫폼은 대량의 민감한 데이터를 다루기 때문에, 외부로부터의 무단 접근을 차단하는 **네트워크 수준의 보안** 이 필수적입니다. 아무리 강력한 인증과 권한 시스템을 갖추더라도, 네트워크 자체가 노출되어 있으면 공격 표면(Attack Surface)이 넓어집니다.

> 💡 **네트워크 보안** 은 데이터에 도달하기 전의 **첫 번째 방어선** 입니다. "허용된 네트워크에서만 접근 가능한가?"라는 질문에 답하는 계층입니다.

특히 금융, 의료, 공공 분야에서는 규제 요건(PCI-DSS, HIPAA, GDPR 등)으로 인해 네트워크 격리가 **필수** 사항입니다.

---

## Databricks 네트워크 아키텍처

Databricks는 크게 **Control Plane** 과 **Data Plane** 이라는 두 영역으로 나뉩니다. 이 구조를 이해하는 것이 네트워크 보안 설정의 출발점입니다.

<!-- 📌 대체 예정: Databricks 네트워크 아키텍처 공식 이미지 확보 후 교체 -->
| 경로 | 설명 |
|------|------|
| ① 사용자 → Control Plane | HTTPS로 웹 UI/API 접근 |
| ② Control Plane → Data Plane | 클러스터 명령 전달 |
| ③ Data Plane → 데이터 저장소 | 데이터 읽기/쓰기 (S3/ADLS/GCS) |
| ④ Control Plane → 사용자 | 결과 반환 |

*출처: [Databricks Docs](https://docs.databricks.com)*

| 영역 | 위치 | 역할 | 관리 주체 |
|------|------|------|----------|
| **Control Plane** | Databricks 클라우드 계정 | 웹 UI, REST API, 작업 스케줄링, 노트북 관리 | Databricks |
| **Data Plane (Classic)** | 고객 클라우드 계정 | Spark 클러스터 실행, 데이터 처리 | 고객 (VPC/VNet) |
| **Data Plane (Serverless)** | Databricks 관리 계정 | 서버리스 SQL/Notebook/Job 실행 | Databricks |

> 💡 **Classic Data Plane** 은 고객의 VPC에서 실행되므로 고객이 네트워크를 직접 제어할 수 있습니다. **Serverless Data Plane** 은 Databricks가 관리하지만, 네트워크 격리와 데이터 접근은 동일한 수준으로 보호됩니다.

---

## 고객 관리형 VPC (Customer-Managed VPC/VNet)

기본적으로 Databricks는 자체 관리 VPC를 생성하지만, 보안 요건이 높은 조직은 **직접 생성한 VPC** 를 사용할 수 있습니다.

| 항목 | Databricks-Managed VPC | Customer-Managed VPC |
|------|----------------------|---------------------|
| **VPC 생성** | Databricks가 자동 생성 | 고객이 직접 생성 |
| **서브넷 제어** | 제한적 | 완전한 제어 |
| **보안 그룹** | 기본 설정 적용 | 고객 커스텀 가능 |
| **VPC Peering** | 지원 | 지원 (더 유연) |
| **Transit Gateway** | 제한적 | 완전 지원 |
| **아웃바운드 제어** | 제한적 | 방화벽/NAT 게이트웨이 |
| **적합한 환경** | PoC, 개발 환경 | 프로덕션, 규제 산업 |

### Customer-Managed VPC 필수 요건

| 구성 요소 | 설명 |
|----------|------|
| **Private Subnet 1**(AZ-a) | Databricks 클러스터용 |
| **Private Subnet 2**(AZ-b) | Databricks 클러스터용 (HA) |
| (선택) NAT Gateway | 아웃바운드 인터넷 접근 |
| **NSG/Security Group** | 인바운드: Control Plane 트래픽 허용, 클러스터 간: 같은 SG 내 모든 포트 허용 |

---

## IP 접근 제한 목록 (IP Access Lists)

**IP Access List** 는 Databricks 워크스페이스에 접근할 수 있는 IP 주소를 제한합니다. 회사 VPN이나 사무실 IP만 허용하여 외부 접근을 차단하는 가장 기본적인 네트워크 보안 설정입니다.

### IP Access List 유형

| 유형 | 설명 | 용도 |
|------|------|------|
| **ALLOW** | 지정된 IP에서의 접근을 허용합니다 | 사무실, VPN IP 허용 |
| **BLOCK** | 지정된 IP에서의 접근을 차단합니다 | 특정 IP 차단 |

### 설정 방법

**1. Databricks CLI**
```bash
# IP Access List 생성 (허용 목록)
databricks ip-access-lists create \
  --label "Office-Network" \
  --list-type ALLOW \
  --ip-addresses "203.0.113.0/24,198.51.100.10"

# 목록 조회
databricks ip-access-lists list

# IP Access List 활성화 (Workspace 설정)
databricks workspace-conf set-status \
  --json '{"enableIpAccessLists": "true"}'
```

**2. REST API**
```bash
# IP Access List 생성
curl -X POST "https://<workspace-url>/api/2.0/ip-access-lists" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "label": "Corporate-VPN",
    "list_type": "ALLOW",
    "ip_addresses": ["10.0.0.0/8", "172.16.0.0/12"]
  }'
```

> ⚠️ **주의**: IP Access List를 잘못 설정하면 본인도 워크스페이스에 접근할 수 없게 됩니다. 반드시 현재 접속 중인 IP가 허용 목록에 포함되어 있는지 확인하세요. Account Console에서 복구할 수 있습니다.

---

## Private Link

> 💡 **Private Link** 는 Databricks와의 모든 통신을 **퍼블릭 인터넷을 거치지 않고** 클라우드 제공자의 백본 네트워크를 통해 수행하는 기술입니다. AWS에서는 **AWS PrivateLink**, Azure에서는 **Azure Private Link** 라고 합니다.

### Front-end vs Back-end Private Link

<!-- 📌 대체 예정: Private Link 아키텍처 공식 이미지 확보 후 교체 -->
![Backend PrivateLink](https://docs.databricks.com/aws/en/assets/images/pl-aws-be-89d73d019437bb90e32610dd5e82ade9.png)

![Frontend PrivateLink](https://docs.databricks.com/aws/en/assets/images/pl-aws-fe-84ca114d753c6130f407c6f9b776956d.png)

*출처: [Databricks Docs](https://docs.databricks.com)*

| 유형 | 경로 | 보호 대상 | 효과 |
|------|------|----------|------|
| **Front-end Private Link** | 사용자 → Control Plane | 사용자의 웹 UI/API 트래픽 | 인터넷 없이 Databricks에 접근 |
| **Back-end Private Link** | Data Plane → Control Plane | 클러스터와 Control Plane 간 통신 | 클러스터 트래픽의 내부화 |

### Private Link 설정 요약 (AWS)

| 단계 | 작업 | 상세 |
|------|------|------|
| 1 | **VPC Endpoint 생성** | AWS 콘솔에서 Interface Endpoint 생성 |
| 2 | **서비스 이름 지정** | Databricks 제공 PrivateLink 서비스 이름 입력 |
| 3 | **보안 그룹 설정** | HTTPS(443) 인바운드 허용 |
| 4 | **DNS 설정** | Private Hosted Zone 생성 및 연결 |
| 5 | **Workspace 설정** | Private Link 사용 Workspace 생성 또는 기존 Workspace 마이그레이션 |

> ⚠️ **Private Link는 Workspace 생성 시 설정하는 것이 권장됩니다.** 기존 Workspace의 마이그레이션도 가능하지만, 다운타임이 발생할 수 있습니다.

---

## VPC Peering

**VPC Peering** 은 Databricks의 Data Plane VPC와 고객의 다른 VPC(예: 데이터베이스 VPC, 애플리케이션 VPC)를 **프라이빗하게 연결** 합니다.

| VPC | CIDR |
|-----|------|
| 고객 데이터베이스 VPC | 10.1.0.0/16 |
| Databricks Data Plane VPC | 10.2.0.0/16 |
| 고객 애플리케이션 VPC | 10.3.0.0/16 |

> VPC Peering으로 상호 연결

| 고려사항 | 설명 |
|---------|------|
| **CIDR 겹침** | 피어링되는 VPC의 IP 대역이 겹치면 안 됩니다 |
| **Transitive Peering** | VPC Peering은 직접 연결만 지원합니다. A↔B, B↔C 피어링이 있어도 A↔C 통신은 불가합니다 |
| **Transit Gateway** | 여러 VPC 연결이 필요하면 Transit Gateway를 사용하세요 |
| **보안 그룹** | 피어링 후에도 보안 그룹에서 트래픽을 허용해야 합니다 |

---

## 아웃바운드 제어 및 데이터 유출 방지

### 방화벽/NAT Gateway를 통한 아웃바운드 제어

Customer-Managed VPC에서는 아웃바운드 트래픽을 제어하여 **데이터 유출(Data Exfiltration)** 을 방지할 수 있습니다.

| 허용해야 할 아웃바운드 대상 | 용도 |
|---------------------------|------|
| Databricks Control Plane | 클러스터 관리 통신 |
| S3/ADLS/GCS (고객 버킷) | 데이터 읽기/쓰기 |
| AWS STS (S3 경우) | 임시 자격증명 발급 |
| Databricks 아티팩트 저장소 | 라이브러리, Init Script |
| PyPI/Conda (선택) | Python 패키지 설치 |

### S3 버킷 정책을 이용한 데이터 유출 방지 (AWS)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictToSpecificVPC",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-data-bucket",
        "arn:aws:s3:::my-data-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpc": "vpc-0123456789abcdef0"
        }
      }
    }
  ]
}
```

> 💡 **데이터 유출 방지(Data Exfiltration Prevention)** 란, 허가되지 않은 경로로 데이터가 외부로 빠져나가는 것을 막는 보안 정책입니다. 예를 들어, Databricks 클러스터에서 임의의 외부 S3 버킷으로 데이터를 복사하는 것을 차단합니다.

---

## 클라우드별 네트워크 보안 비교

| 기능 | AWS | Azure | GCP |
|------|-----|-------|-----|
| **프라이빗 연결** | AWS PrivateLink | Azure Private Link | Private Service Connect |
| **고객 관리 네트워크** | Customer-Managed VPC | VNet Injection | Customer-Managed VPC |
| **네트워크 피어링** | VPC Peering | VNet Peering | VPC Peering |
| **방화벽** | Security Group + NACL | NSG + Azure Firewall | Firewall Rules |
| **IP 접근 제한** | IP Access Lists | IP Access Lists | IP Access Lists |
| **서비스 엔드포인트** | VPC Endpoint (Gateway/Interface) | Service Endpoint | Private Google Access |
| **아웃바운드 제어** | NAT Gateway + 방화벽 | NSG + UDR + Firewall | Cloud NAT + Firewall |
| **DNS** | Route 53 Private Hosted Zone | Azure Private DNS | Cloud DNS |

---

## 보안 수준별 네트워크 구성 가이드

### 기본 (Basic) — 개발/PoC 환경

```
✅ IP Access List 설정 (회사 VPN IP만 허용)
✅ Databricks-Managed VPC 사용
✅ TLS 1.2+ (기본 활성화)
```

### 표준 (Standard) — 일반 프로덕션 환경

```
✅ 기본 설정 포함
✅ Customer-Managed VPC
✅ VPC Peering (데이터 소스 연결)
✅ NAT Gateway (아웃바운드 제어)
✅ S3 VPC Endpoint (데이터 유출 방지)
```

### 강화 (Enhanced) — 규제 산업 (금융/의료/공공)

```
✅ 표준 설정 포함
✅ Front-end + Back-end Private Link
✅ 인터넷 접근 완전 차단
✅ 방화벽 규칙 (허용 목록 기반)
✅ S3 버킷 정책 (VPC 제한)
✅ CMK (고객 관리 암호화 키)
✅ 감사 로그 모니터링
```

---

## 컴플라이언스 체크리스트

| # | 항목 | 관련 규제 | 확인 |
|---|------|----------|------|
| 1 | IP Access List가 활성화되어 있는가? | 공통 | ☐ |
| 2 | 퍼블릭 인터넷 접근이 차단되어 있는가? | PCI-DSS, HIPAA | ☐ |
| 3 | Private Link가 구성되어 있는가? | HIPAA, FedRAMP | ☐ |
| 4 | 아웃바운드 트래픽이 제어되는가? | PCI-DSS | ☐ |
| 5 | 네트워크 로그가 수집되고 있는가? | SOC 2, ISO 27001 | ☐ |
| 6 | VPC Flow Logs가 활성화되어 있는가? | SOC 2 | ☐ |
| 7 | 암호화 키가 고객 관리(CMK)인가? | HIPAA, FedRAMP | ☐ |
| 8 | 데이터 유출 방지 정책이 적용되어 있는가? | GDPR, PCI-DSS | ☐ |

---

## 현업 사례: 퍼블릭 인터넷으로 Databricks 접근하다가 보안 감사에서 지적받은 사례

네트워크 보안은 "나중에 하자"고 미루다가 큰 낭패를 보는 대표적인 영역입니다.

### 사례: 금융사 보안 감사 지적

**사례: 금융사 네트워크 보안 강화**

| Phase | 기간 | 작업 |
|-------|------|------|
| **PoC** | 초기 | 네트워크 설정 없이 빠르게 구축 |
| **프로덕션 전환** | 6개월 후 | 보안 강화 필요 |
| 1단계 | - | VPC 구성 + SCC 활성화 |
| 2단계 | - | PrivateLink 구성 (UI/API + 백엔드) |
| 3단계 | - | S3 VPC Endpoint + 버킷 정책 |
| 4단계 | - | IP 접근 제어 + WAF |

### 이 사례에서 배우는 교훈

> ⚠️ **현업에서는 이렇게 합니다**: 프로덕션 환경은 **반드시 네트워크 보안을 먼저 설정** 한 후 워크로드를 배포합니다. 특히 금융, 의료, 공공 분야에서는 IP Access List와 Private Link가 "선택"이 아니라 "필수"입니다. PoC에서 프로덕션으로 전환할 때, 네트워크 아키텍처를 반드시 재검토하세요.

---

## Private Link 설정이 현실적으로 얼마나 복잡한지

Private Link는 "설정하면 끝"이 아닙니다. 현업에서 겪는 현실적인 복잡성을 솔직하게 공유합니다.

### Private Link 설정의 실제 소요 시간

| 단계 | 예상 시간 | 실제 소요 시간 | 병목 원인 |
|------|---------|-------------|----------|
| VPC Endpoint 생성 | 1시간 | 1~2시간 | AWS 콘솔 작업 자체는 간단 |
| DNS 설정 (Private Hosted Zone) | 2시간 | 1~3일 | 기업의 DNS 관리팀 승인 필요 |
| 보안 그룹 조정 | 1시간 | 1~2일 | 네트워크팀 검토/승인 프로세스 |
| Workspace 마이그레이션 | 2시간 | 1~2일 | 다운타임 윈도우 조율 필요 |
| 테스트 및 검증 | 2시간 | 3~5일 | 모든 연결 경로 확인 필요 |
| **합계 (이론)** | **~8시간** | — | — |
| **합계 (현실)** | — | **2~4주** | 조직 프로세스가 병목 |

### 현실적으로 어려운 부분들

```
[어려움 1: DNS 지옥]
- Private Link를 설정하면 Databricks URL이 Private IP로 해석되어야 함
- 기업 내부 DNS 서버, VPN DNS, 클라우드 DNS가 복잡하게 얽혀 있음
- "사무실에서는 되는데 VPN에서는 안 됩니다" 문제가 빈번
- 해결: Split-horizon DNS 설정 필요 (내부/외부 DNS 분리)

[어려움 2: 기존 연결 깨짐]
- Private Link로 전환하면 기존 퍼블릭 접근이 차단됨
- BI 도구(Tableau, Power BI), CI/CD 파이프라인, 외부 연동 서비스가 영향
- 사전에 모든 접근 경로를 파악해야 함 (놓치면 장애)

[어려움 3: 멀티 리전/계정]
- 여러 리전에 Databricks를 운영하면 각 리전마다 Private Link 설정
- Cross-account/Cross-region 연결이 필요하면 Transit Gateway 추가 비용
```

> 💡 **현업 팁**: Private Link는 **새 Workspace를 생성할 때 처음부터 설정** 하는 것이 10배 쉽습니다. 기존 Workspace를 마이그레이션하면 다운타임, DNS 변경, 기존 연결 끊김 등의 리스크가 있습니다. 신규 프로젝트라면 처음부터 Private Link를 켜세요.

---

## IP Access List만으로 충분한 경우

Private Link가 항상 필요한 것은 아닙니다. 비용과 복잡성을 고려하면 IP Access List만으로도 충분한 경우가 있습니다.

### 판단 기준

| 조건 | IP Access List만으로 충분 | Private Link 필요 |
|------|--------------------------|------------------|
| **규제 요건** | 명시적 Private Link 요구 없음 | 금감원, HIPAA 등에서 인터넷 차단 요구 |
| **데이터 민감도** | 내부 분석용 데이터 | 고객 PII, 금융 거래, 의료 정보 |
| **조직 규모** | 소~중규모 (사용자 100명 이하) | 대규모 (사용자 100명+) |
| **접근 경로** | 사무실 + VPN만 | 여러 네트워크에서 다양한 접근 |
| **보안 감사 수준** | 일반적 (SOC 2) | 강화 (PCI-DSS, HIPAA, FedRAMP) |

### IP Access List로 충분한 시나리오

```
[시나리오: 중소기업 데이터 분석 팀]
- 사용자 30명, 사무실 + VPN 접속
- 고객 PII 없음, 내부 운영 데이터 분석
- SOC 2 정도의 보안 수준 필요

→ IP Access List 설정:
  - 사무실 고정 IP: 203.0.113.0/24
  - VPN 출구 IP: 198.51.100.0/24
  - 비용: $0 (Databricks 기본 기능)
  - 설정 시간: 30분
  - 효과: 허용된 네트워크 외 접근 완전 차단
```

### IP Access List + 추가 보안으로 강화하는 방법

Private Link 없이도 보안을 강화할 수 있는 조합:

```
[보안 강화 조합 — Private Link 없이]
☑ IP Access List (필수) — 허용된 IP만 접근
☑ 조건부 접근 (Conditional Access) — Azure AD 조건부 정책
☑ MFA (Multi-Factor Authentication) — 2차 인증 필수
☑ 세션 타임아웃 — 비활동 시 자동 로그아웃
☑ API 토큰 수명 제한 — 토큰 유효 기간 단축
☑ S3 VPC Endpoint — 데이터 경로만 프라이빗으로 설정

→ 이 조합으로도 대부분의 비규제 산업 보안 요건을 충족
→ Private Link 대비 비용 절감: 월 수백만 원
```

> ⚠️ **현업에서는 이렇게 합니다**: "Private Link가 좋은 건 알지만, 설정/유지 비용이 부담"이라면, 위 조합으로 시작하세요. 규제 당국이 명시적으로 Private Link를 요구하거나, 보안 감사에서 지적받은 경우에만 Private Link로 전환해도 늦지 않습니다. 단, **IP Access List는 반드시 설정** 하세요. 이것조차 안 하면 워크스페이스 URL만 알면 전 세계 누구나 로그인 시도가 가능합니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Control Plane / Data Plane** | Databricks 아키텍처의 두 축. Data Plane을 고객 VPC에서 실행하여 네트워크를 제어합니다 |
| **IP Access List** | 가장 기본적인 네트워크 보안. 허용된 IP에서만 접근할 수 있도록 제한합니다 |
| **Private Link** | 퍼블릭 인터넷 없이 Databricks와 통신합니다. Front-end(사용자)와 Back-end(클러스터) 두 유형이 있습니다 |
| **Customer-Managed VPC** | 고객이 네트워크를 완전히 제어합니다. 프로덕션 환경에 권장됩니다 |
| **데이터 유출 방지** | S3 버킷 정책, 아웃바운드 방화벽으로 데이터가 외부로 나가는 것을 차단합니다 |

---

## 참고 링크

- [Databricks: Network security](https://docs.databricks.com/aws/en/security/network/)
- [Databricks: Private Link](https://docs.databricks.com/aws/en/security/network/classic/privatelink.html)
- [Databricks: Customer-managed VPC](https://docs.databricks.com/aws/en/security/network/classic/customer-managed-vpc.html)
- [Databricks: IP Access Lists](https://docs.databricks.com/aws/en/security/network/front-end/ip-access-list.html)
- [Azure Databricks: Virtual network injection](https://learn.microsoft.com/en-us/azure/databricks/security/network/)
