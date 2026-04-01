# Databricks 아키텍처

## 왜 아키텍처를 이해해야 하나요?

Databricks를 사용하다 보면 "내 데이터는 어디에 저장되는 거지?", "클러스터는 누가 관리하지?", "보안은 어떻게 되는 거지?" 같은 질문이 생깁니다. 이 질문들에 답하려면 Databricks의 아키텍처를 이해해야 합니다.

Databricks 아키텍처의 핵심은 **Control Plane(제어 평면)** 과 **Data Plane(데이터 평면)** 의 분리입니다.

---

## Control Plane vs Data Plane

> 💡 **Control Plane(제어 평면)** 이란 Databricks가 직접 관리하는 영역으로, 사용자 인터페이스, 작업 스케줄링, 노트북 관리 등 "관리 기능"을 담당합니다.
>
> **Data Plane(데이터 평면)** 이란 고객의 클라우드 계정에서 실행되는 영역으로, 실제 데이터 처리와 저장이 이루어지는 곳입니다.

이 분리가 중요한 이유를 실무 관점에서 설명드리겠습니다.

**데이터 주권(Data Sovereignty)**: 고객의 데이터는 항상 **고객의 클라우드 계정(S3, ADLS, GCS)에 저장**됩니다. Databricks Control Plane은 메타데이터와 실행 명령만 관리하고, 실제 데이터는 절대 Databricks 측으로 이동하지 않습니다. 이것은 GDPR, HIPAA, 금융 규제에서 요구하는 데이터 레지던시 요건을 충족하는 핵심 설계입니다.

**경쟁사와의 차이**: Snowflake는 데이터를 자체 관리 스토리지에 저장하므로, 데이터 이동(egress) 비용과 벤더 종속이 발생합니다. Databricks는 오픈 포맷(Delta Lake/Parquet)으로 고객 스토리지에 저장하므로, 언제든 다른 도구에서 직접 읽을 수 있습니다.

**실전에서의 의미**: Control Plane 장애가 발생해도 데이터는 안전합니다. 고객의 S3 버킷에 그대로 있으므로, 다른 Spark 클러스터나 Trino, Snowflake에서도 Delta/Parquet 파일을 직접 읽을 수 있습니다. 이런 아키텍처 덕분에 특정 벤더에 완전히 종속되는 리스크가 크게 줄어듭니다.

### 아키텍처 다이어그램

![Databricks 아키텍처 — Control Plane과 Data Plane](https://docs.databricks.com/aws/en/_images/databricks-architecture-aws.png)

> 출처: [Databricks 공식 문서 — Architecture overview](https://docs.databricks.com/aws/en/getting-started/overview.html)

<!-- 📌 이미지가 표시되지 않는 경우: Databricks 공식 문서의 Architecture overview 페이지에서 최신 이미지 URL을 확인하세요 -->

### 각 영역의 상세 구성

#### Control Plane (Databricks 관리 영역)

| 구성 요소 | 역할 |
|-----------|------|
| **Web Application** | 브라우저에서 접속하는 Workspace UI를 제공합니다 |
| **Job Scheduler** | 작업의 스케줄링, 실행, 재시도를 관리합니다 |
| **Notebook Service** | 노트북의 저장, 버전 관리, 실행 결과 관리를 담당합니다 |
| **Cluster Manager** | 클러스터의 생성, 스케일링, 종료를 제어합니다 |
| **Unity Catalog Metastore** | 테이블, 뷰, 함수 등의 메타데이터를 저장합니다 |
| **IAM (Identity & Access)** | 사용자 인증과 권한 관리를 수행합니다 |

#### Data Plane (고객 클라우드 영역)

| 구성 요소 | 역할 |
|-----------|------|
| **Compute Clusters** | 실제 Spark 작업을 실행하는 가상 머신(VM) 클러스터입니다 |
| **Cloud Storage** | Delta Lake 테이블, 파일 등 실제 데이터가 저장되는 곳입니다 |
| **Network (VPC/VNet)** | 고객의 가상 네트워크 안에서 안전하게 통신합니다 |

> 💡 **VPC(Virtual Private Cloud)란?** 클라우드 안에서 논리적으로 격리된 네트워크 공간입니다. 마치 큰 건물(클라우드) 안에 전용 사무실(VPC)을 빌리는 것과 같습니다. 외부에서 함부로 접근할 수 없고, 허용된 경로로만 통신할 수 있습니다. AWS에서는 VPC, Azure에서는 VNet(Virtual Network)이라고 부릅니다.

---

## Serverless Data Plane

최근 Databricks는 **Serverless** 옵션을 통해 Data Plane의 관리 부담을 더욱 줄이고 있습니다.

> 💡 **서버리스(Serverless)란?** 사용자가 서버(컴퓨팅 리소스)를 직접 생성하거나 관리할 필요 없이, 작업을 실행하면 시스템이 알아서 적절한 리소스를 할당해 주는 방식입니다. "서버가 없다"는 뜻이 아니라, "서버 관리를 신경 쓰지 않아도 된다"는 의미입니다.

![Serverless Architecture](https://docs.databricks.com/aws/en/assets/images/serverless-workspaces-1634da84fc840966875dfee6e9f613e0.png)

*출처: [Databricks Docs](https://docs.databricks.com)*

| 비교 항목 | 클래식 (Customer-Managed) | 서버리스 (Serverless) |
|-----------|--------------------------|----------------------|
| 리소스 관리 | 고객이 직접 클러스터 설정 | Databricks가 자동 관리 |
| 시작 시간 | 수 분 (클러스터 시작) | 수 초 (즉시 시작) |
| 비용 모델 | 클러스터 실행 시간 기준 | 실제 처리량 기준 |
| 적합한 경우 | 세밀한 제어가 필요할 때 | 빠른 시작, 간편 운영 |

> 🆕 **최신 동향**: Databricks는 SQL Warehouse, Notebooks, Jobs, SDP 등 대부분의 워크로드에서 Serverless 옵션을 지원하고 있으며, 점차 서버리스를 기본(default) 모드로 전환하고 있습니다. 특히 Serverless SQL Warehouse는 수 초 만에 시작되어 대화형 SQL 분석에 매우 적합합니다.

---

## 클라우드별 아키텍처

세 클라우드에서의 기본 구조는 동일하지만, 사용되는 클라우드 서비스 이름이 다릅니다.

| 구성 요소 | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| 오브젝트 스토리지 | S3 | ADLS Gen2 | GCS |
| 네트워크 | VPC | VNet | VPC |
| 컴퓨트 VM | EC2 | Azure VM | Compute Engine |
| IAM | AWS IAM | Azure AD (Entra ID) | Cloud IAM |
| 네트워크 격리 | PrivateLink | Private Endpoint | Private Service Connect |

---

## Workspace의 개념

> 💡 **Workspace(워크스페이스)** 란 Databricks에서 작업을 수행하는 **독립적인 작업 환경**입니다. 하나의 조직에서 여러 개의 Workspace를 만들 수 있으며, 각 Workspace는 고유한 URL을 가집니다.

Workspace는 다음과 같은 구성 요소를 포함합니다.

| 구성 요소 | 설명 |
|-----------|------|
| Notebooks | 코드 작성 및 실행 |
| Repos | Git 저장소 연동 |
| Clusters | 컴퓨팅 리소스 |
| Jobs | 스케줄된 작업 |
| SQL Editor | SQL 쿼리 편집기 |
| Dashboards | 시각화 대시보드 |

**Unity Catalog**는 여러 Workspace에서 공유됩니다.

*출처: [Databricks Docs](https://docs.databricks.com)*

### Workspace 구성 모범 사례

| 전략 | 설명 | 적합한 경우 |
|------|------|------------|
| **환경별 분리** | 개발(Dev), 스테이징(Staging), 프로덕션(Prod) 별도 Workspace | 대부분의 기업에 권장됩니다 |
| **팀별 분리** | 데이터 엔지니어링 팀, 분석 팀, ML 팀 별도 Workspace | 팀 간 격리가 필요할 때 |
| **단일 Workspace** | 모든 팀이 하나의 Workspace 사용 | 소규모 조직 |

> 💡 **Unity Catalog**는 여러 Workspace에 걸쳐 공유될 수 있습니다. 따라서 Workspace를 분리하더라도 데이터에 대한 통합 거버넌스를 유지할 수 있습니다.

---

## 데이터 흐름의 전체 그림

지금까지 배운 내용을 종합하여, Databricks에서 데이터가 흘러가는 전체 과정을 살펴보겠습니다.

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 데이터 소스 | 운영 DB, 클라우드 스토리지, SaaS 앱, 스트리밍(Kafka) | 원본 데이터 발생지 |
| 수집 | Lakeflow Connect, Auto Loader | 데이터를 레이크하우스로 수집 |
| 레이크하우스 | Bronze → Silver → Gold (Delta Lake) | Medallion 아키텍처로 데이터 정제 |
| 변환 | SDP (선언적 파이프라인) | 데이터 변환 및 품질 관리 |
| 소비 | Databricks SQL, MLflow, 외부 BI 도구 | 분석, ML, 리포팅 |
| 거버넌스 | Unity Catalog | 전체 관리 |

*출처: [Databricks Docs](https://docs.databricks.com)*

---

## Control Plane 내부 구성 요소 심화

Control Plane은 단순한 웹 서버가 아니라, 여러 **마이크로서비스**로 구성된 복합 시스템입니다. Principal SA 수준에서 이 내부 구조를 이해하면 장애 대응과 아키텍처 설계에 큰 도움이 됩니다.

### Control Plane 마이크로서비스 아키텍처

| 서비스 | 역할 | 상세 설명 |
|--------|------|-----------|
| **Webapp** | UI/API Gateway | REST API 요청을 라우팅하고, Workspace UI를 제공합니다. 모든 API 호출의 진입점입니다 |
| **Shard (Metastore)** | 메타데이터 저장 | Workspace 설정, 노트북 내용, 클러스터 설정 등을 MySQL 호환 DB에 저장합니다 |
| **Cluster Manager** | 클러스터 오케스트레이션 | 클라우드 VM 프로비저닝, Auto Scaling, Spot Instance 관리, 비정상 노드 교체를 수행합니다 |
| **Jobs Service** | 작업 스케줄링 | Cron 기반 스케줄, 트리거 기반 실행, 재시도 로직, DAG 의존성 관리를 담당합니다 |
| **Unity Catalog Service** | 거버넌스 엔진 | 메타데이터 CRUD, 권한 평가(Policy Evaluation), 리니지 수집, 감사 로그 생성을 수행합니다 |
| **Token Service** | 인증/인가 | PAT(Personal Access Token), OAuth 토큰, 서비스 프린시펄 인증을 관리합니다 |
| **Secure Cluster Connectivity (SCC)** | 네트워크 터널 | Data Plane → Control Plane 방향의 안전한 역방향 터널을 제공합니다 |

### Control Plane의 데이터 저장

> 💡 **중요 포인트**: Control Plane에는 **고객의 비즈니스 데이터가 저장되지 않습니다**. 다만 다음과 같은 메타데이터는 Control Plane에 존재합니다.

| 저장 항목 | 위치 | 민감도 |
|-----------|------|--------|
| 노트북 소스 코드 | Control Plane DB | 중간 (코드에 민감 정보 포함 가능) |
| 클러스터/작업 설정 | Control Plane DB | 낮음 |
| 쿼리 결과 (1MB 미만 캐시) | Control Plane 메모리 | 높음 (결과에 민감 데이터 포함 가능) |
| UC 메타데이터 | Control Plane DB | 낮음 (테이블 이름, 스키마 등) |
| Git 자격증명 (암호화) | Control Plane KMS | 높음 |

> ⚠️ **보안 주의**: 노트북에 비밀번호나 API 키를 하드코딩하면 Control Plane에 저장됩니다. 반드시 **Databricks Secrets**를 사용하세요.

---

## Classic Data Plane vs Serverless Data Plane 아키텍처 심화

### Classic Data Plane 상세

Classic Data Plane에서는 클러스터가 **고객의 VPC/VNet** 안에서 실행됩니다. 이 모델의 핵심 특징을 살펴보겠습니다.

| 구성 요소 | 상세 설명 |
|-----------|-----------|
| **Driver Node** | Spark Application의 마스터 노드입니다. SparkContext를 생성하고, DAG 스케줄링, 태스크 분배를 수행합니다 |
| **Worker Nodes** | 실제 데이터 처리를 수행하는 Executor가 실행됩니다. Auto Scaling으로 동적 증감합니다 |
| **NAT Gateway** | Worker → 인터넷 통신 시 NAT Gateway를 경유합니다. Control Plane과의 통신도 이 경로를 사용합니다 |
| **SCC Relay** | Secure Cluster Connectivity 활성화 시, Data Plane에서 Control Plane으로의 아웃바운드 연결만 허용됩니다 |
| **DBFS Root** | Workspace 생성 시 자동으로 만들어지는 S3/ADLS 버킷입니다. 클러스터 로그, 라이브러리 등이 저장됩니다 |

### Serverless Data Plane 상세

Serverless 모드에서는 컴퓨팅이 **Databricks가 관리하는 클라우드 계정**에서 실행됩니다. 이 아키텍처에는 중요한 차이가 있습니다.

| 항목 | Classic | Serverless |
|------|---------|------------|
| **VM 위치** | 고객 클라우드 계정 | Databricks 관리 계정 |
| **네트워크 격리** | 고객 VPC에 배포 | Databricks 관리 VPC + 전용 네트워크 격리 |
| **데이터 접근** | VPC 내부에서 직접 접근 | Secure Channel을 통해 고객 스토리지에 접근 |
| **비용 구조** | DBU + 클라우드 인프라 비용 별도 | DBU에 인프라 비용 포함 (올인원) |
| **Cold Start** | 3~10분 (VM 프로비저닝) | 5~15초 (Warm Pool에서 즉시 할당) |
| **패치/업데이트** | 고객이 DBR 버전 관리 | Databricks가 자동 업데이트 |
| **커스터마이징** | Init Script, 커스텀 AMI, 라이브러리 자유 | 제한적 (표준화된 환경) |

> 💡 **Serverless Warm Pool**: Databricks는 미리 프로비저닝된 VM 풀(Warm Pool)을 유지하여 Serverless 컴퓨트의 시작 시간을 수 초로 줄입니다. 이는 마치 "대기 중인 택시"와 같아서, 요청이 오면 즉시 할당됩니다.

### Serverless 보안 모델

Serverless Data Plane의 보안에 대해 고객이 자주 우려하는 부분을 정리합니다.

| 보안 메커니즘 | 설명 |
|-------------|------|
| **Workload Isolation** | 각 고객의 워크로드는 전용 VM에서 실행되며, 다른 고객과 하드웨어를 공유하지 않습니다 (단일 테넌트 VM) |
| **네트워크 격리** | 각 워크로드에 전용 네트워크 네임스페이스가 할당됩니다 |
| **데이터 암호화** | 전송 중(TLS 1.2+) + 저장 시(AES-256) 암호화됩니다 |
| **Credential Passthrough** | Serverless에서 고객 스토리지 접근 시 임시 자격증명(STS Token)을 사용하여 장기 키를 노출하지 않습니다 |
| **감사 로그** | 모든 Serverless 작업이 고객의 감사 로그에 기록됩니다 |

---

## 네트워크 트래픽 흐름 상세

### 주요 통신 경로

엔터프라이즈 환경에서 네트워크 구성을 이해하는 것은 매우 중요합니다.

| 경로 | 방향 | 프로토콜 | 용도 |
|------|------|----------|------|
| **User → Control Plane** | Inbound | HTTPS (443) | UI 접속, API 호출 |
| **Control Plane → Data Plane** | Inbound (Classic) | SSH, HTTPS | 클러스터 관리, 명령 전달 |
| **Data Plane → Control Plane** | Outbound | HTTPS (443) | SCC 터널, 결과 전송, 메타데이터 조회 |
| **Data Plane → Cloud Storage** | Outbound | HTTPS (443) | 데이터 읽기/쓰기 (S3, ADLS, GCS) |
| **Data Plane → External** | Outbound | 다양함 | PyPI 패키지 다운로드, 외부 DB 연결 등 |
| **Data Plane 내부** | Internal | 다양함 | Worker 간 Shuffle 통신, Driver-Worker 통신 |

### PrivateLink / Private Endpoint 구성

엔터프라이즈 고객은 **인터넷을 경유하지 않는** 전용 연결을 요구합니다.

| 연결 유형 | AWS | Azure | 용도 |
|-----------|-----|-------|------|
| **프론트엔드 연결** | VPC Endpoint (Interface) | Private Endpoint | 사용자 → Workspace UI/API 접근 |
| **백엔드 연결** | VPC Endpoint (Interface) | Private Endpoint | Data Plane → Control Plane 통신 |
| **스토리지 연결** | VPC Endpoint (Gateway/Interface) | Private Endpoint | Data Plane → S3/ADLS 통신 |

> ⚠️ **비용 고려사항**: PrivateLink/Private Endpoint는 시간당 과금 + 데이터 처리량 과금이 발생합니다. AWS에서 VPC Interface Endpoint는 AZ당 약 $0.01/hr + $0.01/GB입니다. 대용량 데이터 처리 시 상당한 비용이 될 수 있으므로, S3 Gateway Endpoint(무료)를 우선 사용하는 것이 좋습니다.

### 방화벽/NSG 최소 규칙

Classic Data Plane을 사용하는 경우, 다음과 같은 네트워크 규칙이 필요합니다.

```
# Outbound 규칙 (Data Plane → 외부)
1. Control Plane (Webapp)     → HTTPS 443  — 필수
2. Control Plane (SCC Relay)  → HTTPS 443  — 필수
3. S3/ADLS/GCS               → HTTPS 443  — 필수
4. Unity Catalog Metastore    → HTTPS 443  — 필수
5. 패키지 저장소 (PyPI 등)     → HTTPS 443  — 선택
6. Worker 간 Shuffle          → 내부 통신   — 필수 (Security Group 내부)

# Inbound 규칙 (외부 → Data Plane)
7. SCC 사용 시: 없음 (모든 인바운드 차단 가능)
8. SCC 미사용 시: Control Plane → SSH 2200, HTTPS 443
```

---

## 리전별 가용성과 고가용성 (HA)

### Databricks 리전별 가용성

Databricks는 주요 클라우드 리전에서 서비스를 제공합니다.

| 클라우드 | 주요 리전 | 비고 |
|----------|----------|------|
| **AWS** | us-east-1, us-west-2, eu-west-1, ap-northeast-1(도쿄), ap-southeast-1(싱가포르) 등 20+ 리전 | ap-northeast-2(서울) 지원 |
| **Azure** | East US, West Europe, Japan East, Korea Central, Southeast Asia 등 30+ 리전 | Korea Central(서울) 지원 |
| **GCP** | us-central1, europe-west1, asia-northeast1(도쿄) 등 15+ 리전 | 한국 리전 미지원 (도쿄 사용) |

> 💡 **한국 고객**: AWS ap-northeast-2(서울)과 Azure Korea Central을 사용할 수 있습니다. GCP 사용 시에는 asia-northeast1(도쿄)이 가장 가까운 리전입니다.

### Control Plane 고가용성

| HA 메커니즘 | 설명 |
|-------------|------|
| **멀티 AZ 배포** | Control Plane 서비스는 최소 3개 가용 영역(AZ)에 분산 배포됩니다 |
| **데이터 복제** | 메타데이터 DB는 동기식 복제로 데이터 손실을 방지합니다 |
| **자동 장애 조치** | Control Plane 노드 장애 시 자동으로 다른 AZ의 노드로 전환됩니다 |
| **SLA** | Databricks Premium/Enterprise Plan에서 99.95% 가용성 SLA를 제공합니다 |

> ⚠️ **Control Plane 장애 시 영향**: Control Plane이 다운되면 **새 클러스터 시작, 작업 스케줄링, UI 접속**이 불가합니다. 단, **이미 실행 중인 클러스터와 작업은 계속 실행**됩니다(Data Plane은 독립적). 이것이 Control/Data Plane 분리 아키텍처의 핵심 이점입니다.

---

## 재해 복구 (DR) 패턴

### Databricks DR 전략

Databricks 환경의 재해 복구는 크게 세 가지 수준으로 나뉩니다.

| DR 수준 | RPO | RTO | 비용 | 설명 |
|---------|-----|-----|------|------|
| **Level 1: 데이터 DR** | 수 분 | 수 시간 | 낮음 | 데이터(Delta Lake)만 복제, Workspace는 수동 재구성 |
| **Level 2: 설정 포함 DR** | 수 분 | 1~2시간 | 중간 | 데이터 + Workspace 설정(Terraform/DABs)을 코드로 관리 |
| **Level 3: Active-Active** | ~0 | 수 분 | 높음 | 두 리전에 동시 운영, 즉시 전환 가능 |

> 💡 **RPO(Recovery Point Objective)** 는 "데이터를 얼마나 잃어도 되는가", **RTO(Recovery Time Objective)** 는 "복구에 얼마나 걸려도 되는가"입니다.

### 실전 DR 아키텍처 패턴

| 구성 요소 | Primary Region (서울) | Secondary Region (도쿄) | 복제 방식 |
|-----------|----------------------|------------------------|-----------|
| **Workspace** | Active | Standby / Active | Terraform / DABs (Git) |
| **Delta Lake (S3/ADLS)** | 원본 | 복제본 | DEEP CLONE |
| **Unity Catalog Metastore** | 원본 | 공유 또는 동기화 | 리전 간 Metastore 공유 |
| **설정 관리** | Terraform / DABs (Git 저장소) | 동일 코드 기반 배포 | Git 저장소 |

### DR 구현 핵심 요소

| 요소 | 권장 방법 | 설명 |
|------|----------|------|
| **Delta 테이블 복제** | `DEEP CLONE` + 스케줄링 | 주기적으로 DR 리전에 테이블을 복제합니다 |
| **스토리지 복제** | S3 Cross-Region Replication / Azure GRS | 클라우드 네이티브 복제를 활용합니다 |
| **Workspace 설정** | Terraform Provider for Databricks | 모든 Workspace 설정을 IaC로 관리합니다 |
| **노트북/코드** | Git 연동 (GitHub/Azure DevOps) | 코드는 항상 Git에 저장하여 리전 독립적으로 관리합니다 |
| **UC 메타데이터** | 리전 간 Metastore 공유 또는 동기화 | 같은 클라우드 내에서 메타스토어를 공유할 수 있습니다 |
| **시크릿** | Vault/AWS Secrets Manager 멀티 리전 | 자격증명도 DR 리전에 동기화합니다 |

> ⚠️ **DR 비용 고려**: Active-Active 구성은 **거의 2배의 비용**이 발생합니다. 대부분의 고객에게는 Level 2(설정 포함 DR)가 비용 대비 효과가 가장 좋습니다. 금융, 의료 등 규제 산업에서만 Level 3를 권장합니다.

### Delta Lake DEEP CLONE을 활용한 DR

```sql
-- 테이블 단위 DR 복제
CREATE OR REPLACE TABLE dr_catalog.schema.orders
DEEP CLONE production.ecommerce.orders
LOCATION 's3://dr-bucket/ecommerce/orders/';

-- 증분 복제 (변경분만)
CREATE OR REPLACE TABLE dr_catalog.schema.orders
DEEP CLONE production.ecommerce.orders;
-- DEEP CLONE은 자동으로 마지막 복제 이후 변경분만 복사합니다
```

---

## 비용 최적화 관점

아키텍처 선택은 비용에 직접적인 영향을 미칩니다.

| 최적화 영역 | 전략 | 예상 절감 |
|-------------|------|----------|
| **Serverless vs Classic** | 간헐적 워크로드는 Serverless, 상시 워크로드는 Classic | 20~50% |
| **Spot/Preemptible Instance** | Classic 모드에서 Worker에 Spot Instance 사용 | 60~80% (Worker 비용) |
| **Auto Scaling** | min/max worker 적절히 설정, 유휴 시 자동 축소 | 30~50% |
| **Auto Termination** | 클러스터 비활성 시 자동 종료 (기본 120분 → 30분 권장) | 가변적 |
| **Photon 엔진** | SQL 워크로드에서 Photon 활성화 (처리 속도↑, 총 비용↓) | 20~40% |
| **적절한 VM 유형** | 메모리 집약 vs 컴퓨팅 집약 워크로드에 맞는 VM 선택 | 15~30% |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Control Plane** | Databricks가 관리하는 영역. UI, 스케줄링, 메타데이터 관리를 담당합니다 |
| **Data Plane** | 고객 클라우드에 위치한 영역. 실제 데이터 처리와 저장이 이루어집니다 |
| **Serverless** | 리소스를 자동 관리하여 사용자가 인프라를 신경 쓰지 않아도 되는 방식입니다 |
| **Workspace** | Databricks에서 작업을 수행하는 독립적인 환경입니다 |
| **VPC/VNet** | 클라우드에서 네트워크를 격리하여 보안을 확보하는 기술입니다 |
| **PrivateLink** | 인터넷을 경유하지 않는 전용 네트워크 연결입니다 |
| **SCC** | Secure Cluster Connectivity — 인바운드 포트 없이 안전하게 통신합니다 |
| **DR (재해 복구)** | DEEP CLONE, IaC, 스토리지 복제를 결합한 멀티 리전 복구 전략입니다 |

다음 문서에서는 실제로 Databricks Workspace에 접속하여 **UI를 둘러보는 방법**을 안내해 드리겠습니다.

---

## 참고 링크

- [Databricks: Architecture overview](https://docs.databricks.com/aws/en/getting-started/overview.html)
- [Azure Databricks: Architecture](https://learn.microsoft.com/en-us/azure/databricks/getting-started/overview)
- [Databricks: Serverless compute](https://docs.databricks.com/aws/en/serverless-compute/)
