# DBU와 가격 이해

## 가격 모델의 현실 — DBU 이해하기

Databricks 비용을 이해하려면 **DBU(Databricks Unit)** 개념을 알아야 합니다.

### DBU란?

> 💡 **DBU(Databricks Unit)** 는 Databricks에서 사용하는 과금 단위입니다. 클라우드 인프라 비용(VM, 스토리지)과는 별도로, Databricks 플랫폼 사용료를 DBU로 측정합니다. 1 DBU는 대략 "1시간 동안 가장 작은 컴퓨팅 리소스를 사용한 단위"라고 이해하시면 됩니다.

### 비용 구조

Databricks 비용은 크게 **두 가지** 로 구성됩니다.

| 비용 항목 | 누구에게 지불? | 설명 |
|-----------|-------------|------|
| **DBU 비용**| Databricks | 플랫폼 사용료. 워크로드 유형(Jobs, SQL, All-Purpose, Serverless)에 따라 DBU 단가가 다릅니다 |
| **클라우드 인프라 비용**| AWS / Azure / GCP | VM(가상 머신), 스토리지, 네트워크 비용. Databricks가 아닌 클라우드 사업자에게 지불합니다 |

### 워크로드별 DBU 단가 (AWS 기준, 2025년 참고가)

| 워크로드 | DBU 단가 (대략) | 설명 |
|----------|---------------|------|
| **Jobs Compute**| ~$0.10/DBU | 스케줄된 ETL 작업. 가장 저렴합니다 |
| **Jobs Compute (Serverless)**| ~$0.14/DBU | 서버리스 ETL. 인프라 비용 포함입니다 |
| **SQL Warehouse (Serverless)**| ~$0.22/DBU | SQL 분석 워크로드 |
| **All-Purpose Compute**| ~$0.22/DBU | 대화형 개발(노트북). 가장 비쌉니다 |
| **Model Serving**| 별도 과금 | 추론 요청 수 기반 |

> ⚠️ **현업에서의 주의사항**: 실제 단가는 클라우드, 리전, 계약 조건에 따라 다릅니다. 위 수치는 참고용이며, 정확한 단가는 Databricks 영업팀이나 [공식 가격 페이지](https://www.databricks.com/product/pricing)에서 확인하시기 바랍니다.

### 예산 수립 실무 가이드

| 단계 | 행동 | 팁 |
|------|------|-----|
| **1. 워크로드 분류**| ETL/SQL/ML/대화형 개발별로 사용량을 분리합니다 | ETL은 Jobs Compute로, 분석은 SQL Warehouse로 분리하면 비용이 30~50% 절감됩니다 |
| **2. 현재 비용 기준선**| 기존 EMR/Redshift/SageMaker 비용을 합산합니다 | 숨겨진 비용(운영 인력, 데이터 복사, 연동 개발)도 포함해야 합니다 |
| **3. PoC 비용 측정**| 14일 체험판에서 실제 워크로드를 돌려보고 DBU 소비량을 측정합니다 | System Table(`system.billing.usage`)에서 정확한 사용량을 확인할 수 있습니다 |
| **4. 연간 계약**| Commit 계약(사전 약정)을 하면 DBU 단가가 크게 할인됩니다 | 보통 1년 약정 시 15~25%, 3년 약정 시 30% 이상 할인 가능합니다 |

> 💡 **현업에서는 이렇게 합니다**: 가장 흔한 비용 실수는 "All-Purpose Compute(대화형 노트북)에서 ETL을 돌리는 것"입니다. All-Purpose는 DBU 단가가 Jobs Compute의 2배 이상이므로, 반복 실행되는 ETL 작업은 반드시 Jobs Compute로 전환해야 합니다. 이 한 가지만으로도 30~50%의 비용을 절감한 사례가 많습니다.

---

## Databricks가 지원하는 클라우드

Databricks는 주요 3대 클라우드 플랫폼 모두에서 사용할 수 있습니다.

| 클라우드 | 서비스명 | 데이터 저장소 |
|----------|----------|-------------|
| **AWS**| Databricks on AWS | Amazon S3 |
| **Azure**| Azure Databricks | Azure Data Lake Storage (ADLS) |
| **GCP**| Databricks on GCP | Google Cloud Storage (GCS) |

세 클라우드에서 제공하는 Databricks의 핵심 기능은 동일합니다. 다만 인프라 관리, 네트워크 설정, 결제 방식 등에서 클라우드별 차이가 있습니다.

> 💡 **클라우드 선택 팁**: 이미 사용 중인 클라우드가 있다면 해당 클라우드에서 Databricks를 사용하는 것이 가장 자연스럽습니다. 데이터가 S3에 있는데 GCP에서 Databricks를 사용하면 클라우드 간 데이터 전송 비용이 발생합니다. Azure를 주력으로 사용하는 기업은 **Azure Databricks** 가 Azure Active Directory, Azure DevOps 등과 통합이 더 긴밀하다는 장점이 있습니다. 특히 한국 기업에서는 Azure Databricks와 AWS Databricks 사용이 가장 많습니다.

### 멀티 클라우드 전략

일부 대기업은 두 개 이상의 클라우드에서 Databricks를 운영하기도 합니다. 이 경우 **Delta Sharing** 을 통해 클라우드 간 데이터를 안전하게 공유할 수 있습니다. 예를 들어, AWS의 Databricks에서 생성한 데이터를 Azure의 Databricks에서 직접 읽을 수 있습니다.

---

## 참고 링크

- [Databricks: Pricing](https://www.databricks.com/product/pricing)
- [Databricks: Get started with Databricks](https://docs.databricks.com/aws/en/getting-started/)
- [Azure Databricks: What is Azure Databricks?](https://learn.microsoft.com/en-us/azure/databricks/introduction/)
