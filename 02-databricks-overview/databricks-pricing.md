# DBU와 가격 이해

## 1. 왜 가격 모델 이해가 중요한가

클라우드 서비스 도입 시 가장 흔한 실수는 **비용 예측 실패** 입니다. 온프레미스(On-premises)는 서버 구매 비용이 명확하지만, 클라우드는 사용한 만큼 과금되는 구조이기 때문에 사용 패턴에 따라 월 청구액이 수 배까지 달라질 수 있습니다.

Databricks는 특히 다음과 같은 이유로 비용 구조를 이해하지 않으면 **예상치 못한 청구** 가 발생할 수 있습니다.

- **이중 과금 구조**: Databricks 플랫폼 사용료(DBU)와 클라우드 인프라 비용(VM, 네트워크, 스토리지)이 **별도로 청구** 됩니다.
- **워크로드별 단가 차이**: 같은 작업이라도 어떤 Compute 유형을 사용하느냐에 따라 DBU 단가가 2~3배 이상 차이납니다.
- **유휴 클러스터 비용**: 개발자가 노트북 작업을 마치고 클러스터를 종료하지 않으면, 아무 작업이 없어도 VM 비용과 DBU가 계속 발생합니다.
- **플랜에 따른 기능 제한**: 저렴한 Standard 플랜에서는 Unity Catalog, 역할 기반 접근 제어(RBAC) 등 일부 기능을 사용할 수 없습니다.

> 💡 **현업 조언**: Databricks 도입을 검토 중인 조직은 파일럿(Pilot) 프로젝트 단계에서 반드시 실제 워크로드를 기반으로 비용을 측정해야 합니다. 카탈로그 수준의 가격표만 보고 예산을 수립하면 실사용 비용과 큰 괴리가 생깁니다.

---

## 2. DBU 기반 과금 체계

### DBU(Databricks Unit)란?

**DBU(Databricks Unit)** 는 Databricks에서 컴퓨팅 자원 사용량을 측정하는 **표준 과금 단위** 입니다. 전기 요금에서 "kWh(킬로와트시)"가 전력 사용량의 단위인 것처럼, DBU는 Databricks 플랫폼의 컴퓨팅 사용량을 표현합니다.

- 1 DBU는 고정된 물리 자원을 의미하는 것이 아니라, **워크로드 유형과 인스턴스 크기에 따라 소비율이 다릅니다**.
- 예를 들어, 4코어 VM은 1시간에 1 DBU를 소비하지만, 16코어 VM은 동일 조건에서 4 DBU를 소비합니다.
- VM 인스턴스 유형(m5.xlarge, m5.4xlarge 등)마다 시간당 DBU 소비량이 **AWS 공식 문서에 정의** 되어 있습니다.

### 비용의 두 가지 구성 요소

Databricks를 사용하면 두 가지 비용이 각각 발생합니다.

| 비용 항목 | 지불 대상 | 설명 |
|-----------|-----------|------|
| **DBU 비용** | Databricks | 플랫폼 소프트웨어 사용료. SKU(워크로드 유형)에 따라 단가가 다릅니다 |
| **클라우드 인프라 비용** | AWS / Azure / GCP | VM(가상 머신), EBS 스토리지, 네트워크 전송 비용. Databricks가 아닌 클라우드 사업자에게 지불합니다 |

> ⚠️ **주의**: AWS Marketplace를 통해 구독하면 두 비용을 AWS 청구서 하나로 통합할 수 있습니다. 하지만 내부적으로는 여전히 별도 항목으로 집계됩니다.

### SKU별 DBU 소비율 비교 (AWS 기준)

같은 인스턴스(예: m5.xlarge)를 사용해도 **SKU(Stock Keeping Unit)** — 즉 워크로드 유형에 따라 DBU 소비율 계수가 다릅니다.

| SKU (워크로드 유형) | DBU 소비율 계수 | 비고 |
|--------------------|--------------|------|
| **Jobs Compute** (배치 작업) | 1.0x | 기준값 |
| **Jobs Compute + Photon** | 1.5x | Photon 가속 엔진 적용 |
| **All-Purpose Compute** (대화형) | 2.0x ~ 2.5x | 노트북 개발용. 가장 비쌉니다 |
| **SQL Warehouse Classic** | 1.0x ~ 2.0x | 크기(2X-Small~4X-Large)에 따라 다름 |
| **SQL Warehouse Serverless** | 인프라 포함 | VM을 직접 관리하지 않음 |
| **Serverless Jobs** | 인프라 포함 | VM 없이 실행 |

> 💡 **핵심**: All-Purpose Compute는 Jobs Compute 대비 약 2배의 DBU를 소비합니다. 반복 실행되는 ETL 파이프라인을 All-Purpose에서 실행하면 비용이 2배로 올라갑니다.

---

## 3. SKU별 가격 비교

### 워크로드별 대략적 DBU 단가 (AWS, Pay-as-you-go 기준, 2025년 참고가)

| SKU | 대략적 단가 | 인프라 포함 여부 | 주요 용도 |
|-----|-----------|----------------|----------|
| **Jobs Compute** | ~$0.10/DBU | 별도 | 예약된 ETL, 배치 처리 |
| **Jobs Compute + Photon** | ~$0.15/DBU | 별도 | 고성능 ETL |
| **Serverless Jobs** | ~$0.20/DBU | 포함 | 인프라 관리 없는 ETL |
| **All-Purpose Compute** | ~$0.22/DBU | 별도 | 노트북 개발, 탐색적 분석 |
| **SQL Warehouse Classic** | ~$0.22/DBU | 별도 | SQL 분석, BI 연동 |
| **SQL Warehouse Serverless** | ~$0.70/DBU | 포함 | 서버리스 SQL 분석 |
| **Model Serving** | 요청 수 기반 | 포함 | LLM/ML 모델 추론 |

> ⚠️ **면책 사항**: 위 단가는 개념적 이해를 위한 참고값입니다. 실제 단가는 클라우드, 리전, 계약 조건에 따라 달라집니다. 정확한 견적은 [Databricks 가격 페이지](https://www.databricks.com/product/pricing) 또는 영업팀을 통해 확인하시기 바랍니다.

### 상대적 비용 비교

같은 작업을 서로 다른 Compute 유형으로 실행할 때의 비용 비율입니다.

```
Jobs Compute          ████░░░░░░░░  (1배, 기준)
Serverless Jobs       ██████░░░░░░  (약 2배, 인프라 포함)
All-Purpose           ████████░░░░  (약 2.2배)
SQL Warehouse Classic ████████░░░░  (약 2.2배)
SQL Warehouse Server. ████████████  (약 7배, 인프라 포함)
```

Serverless 계열은 단가가 높아 보이지만, **VM 관리 오버헤드 제거**, **자동 스케일링**, **유휴 비용 없음** 등의 운영 이점도 함께 고려해야 합니다.

---

## 4. 플랜별 차이 (Standard / Premium / Enterprise)

Databricks는 세 가지 구독 플랜을 제공합니다. 플랜마다 사용 가능한 기능이 다르고, DBU 단가도 차이가 납니다.

### 플랜별 주요 기능 비교

| 기능 | Standard | Premium | Enterprise |
|------|----------|---------|------------|
| **Apache Spark** 기반 컴퓨팅 | O | O | O |
| **Delta Lake** | O | O | O |
| **Databricks SQL** (SQL Warehouse) | O | O | O |
| **MLflow** 기본 실험 추적 | O | O | O |
| **Unity Catalog** (데이터 거버넌스) | X | O | O |
| **역할 기반 접근 제어(RBAC)** | 기본 | 세분화 | 고급 |
| **데이터 계보(Data Lineage)** | X | O | O |
| **감사 로그(Audit Log)** | 기본 | 상세 | 완전 |
| **IP 액세스 목록** | X | O | O |
| **SSO / SAML 2.0** | X | O | O |
| **전용 SLA / 지원** | 커뮤니티 | 비즈니스 | 엔터프라이즈 |
| **Databricks Marketplace** | O | O | O |

### 어떤 플랜을 선택해야 할까?

- **Standard**: 개인 학습, 소규모 팀의 빠른 프로토타이핑. 보안 요건이 낮고 데이터 거버넌스가 필요 없는 경우에 적합합니다.
- **Premium**: 대부분의 기업 환경에 권장됩니다. Unity Catalog와 RBAC이 포함되어 데이터 거버넌스를 체계적으로 구성할 수 있습니다.
- **Enterprise**: 금융, 의료, 공공 등 규제 산업. SLA 보장, 고급 감사 기능, 전담 지원이 필요한 대형 조직에 적합합니다.

> 💡 **현업 팁**: 한국의 대부분 엔터프라이즈 고객은 **Premium** 플랜에서 시작합니다. Standard는 Unity Catalog를 사용할 수 없어, 멀티 워크스페이스 환경에서 데이터 거버넌스를 구성하기 어렵습니다.

---

## 5. Commit vs Pay-as-you-go

### 두 가지 결제 방식

| 구분 | Pay-as-you-go (종량제) | Commit (사전 약정) |
|------|----------------------|------------------|
| **결제 방식** | 사용한 DBU만큼 월별 청구 | 연간 또는 3년 단위로 DBU 사용량 사전 약정 |
| **DBU 단가** | 정가 | 약정 규모에 따라 15~40% 할인 |
| **유연성** | 언제든지 증감 가능 | 약정 기간 동안 최소 사용량 보장 필요 |
| **위험도** | 예상보다 많이 쓰면 비용 급증 | 약정량 미달 시 손실 발생 가능 |
| **적합한 조직** | 도입 초기, 사용량 불확실한 경우 | 사용량이 안정적이고 예측 가능한 경우 |

### 할인율 가이드라인 (참고)

| 약정 규모 | 약정 기간 | 예상 할인율 |
|-----------|---------|-----------|
| 소규모 | 1년 | 10~20% |
| 중규모 | 1년 | 20~30% |
| 대규모 | 3년 | 30~40%+ |

> ⚠️ **실제 할인율은 Databricks 영업팀과 협상으로 결정** 됩니다. 위 수치는 시장 평균 참고값이며, 조직 규모, 산업, 파트너 채널에 따라 크게 달라질 수 있습니다.

### Commit 계약 시 체크리스트

Commit 계약을 고려하고 있다면 다음 사항을 사전에 점검합니다.

1. **기존 사용량 데이터 수집**: 최소 3~6개월의 DBU 소비 이력이 있어야 합적 약정량을 산출할 수 있습니다.
2. **워크로드 성장률 예측**: 향후 1~3년간의 데이터 처리량 증가를 감안해야 합니다.
3. **SKU 믹스 파악**: Jobs, SQL, All-Purpose 각각의 비율을 파악해야 SKU별 약정 배분이 가능합니다.
4. **미사용 DBU 처리 규정 확인**: 약정 기간 내 미사용 DBU가 이월되는지, 만료되는지 계약서에서 확인합니다.

---

## 6. 비용 최적화 전략

### 핵심 전략 요약

| 전략 | 예상 절감 효과 | 난이도 |
|------|-------------|------|
| **All-Purpose → Jobs Compute 전환** | 30~50% | 낮음 |
| **Spot Instance(스팟 인스턴스) 사용** | 50~70% | 중간 |
| **Auto-termination(자동 종료) 설정** | 10~30% | 낮음 |
| **클러스터 자동 스케일링 활성화** | 15~40% | 낮음 |
| **SQL Warehouse 자동 정지 설정** | 10~20% | 낮음 |
| **Serverless로 전환** | 운영 비용 절감 | 중간 |
| **Photon 선택적 적용** | 성능 향상 → 실행 시간 단축 | 높음 |

### 전략 1: Jobs Compute vs All-Purpose Compute

**반복 실행되는 파이프라인은 반드시 Jobs Compute를 사용합니다.**

- All-Purpose Compute는 대화형 노트북 개발을 위한 것입니다. 여러 사용자가 동일 클러스터를 공유하며 탐색적 분석을 수행하는 데 적합합니다.
- Lakeflow Jobs(구 Databricks Jobs)로 스케줄링된 작업은 자동으로 Jobs Compute를 사용하도록 설정할 수 있습니다.
- All-Purpose에서 Jobs로 전환하는 것만으로도 **동일한 워크로드 대비 DBU 비용을 절반으로 줄일 수 있습니다**.

### 전략 2: Spot Instance(스팟 인스턴스) 활용

AWS의 Spot Instance, Azure의 Spot VM, GCP의 Preemptible VM은 온디맨드(On-demand) 대비 **50~80% 저렴한 유휴 컴퓨팅 자원** 입니다.

- Databricks는 Spot Instance 중단(인터럽트) 시 자동으로 온디맨드 인스턴스로 전환하는 폴백(Fallback) 기능을 제공합니다.
- 배치 ETL 작업처럼 **재실행이 가능한 워크로드** 에 특히 적합합니다.
- 실시간 스트리밍이나 SLA가 엄격한 작업에는 온디맨드와 Spot을 혼합(Mix) 설정합니다.

### 전략 3: Auto-termination(자동 종료) 설정

All-Purpose Cluster에 **자동 종료(Auto-termination)** 를 반드시 설정합니다.

```
권장 설정:
- 개발 클러스터: 30분 유휴 시 자동 종료
- 탐색적 분석 클러스터: 60분 유휴 시 자동 종료
```

유휴 클러스터 하나가 종료되지 않으면 하루 수십~수백 달러의 VM 비용이 낭비될 수 있습니다.

### 전략 4: Serverless 전환 검토

**Serverless Compute** 는 VM을 직접 관리하지 않고 Databricks가 인프라를 자동으로 프로비저닝·종료합니다.

- Serverless Jobs: 클러스터 시작 시간이 일반 클러스터(2~5분)보다 빠릅니다(수십 초).
- Serverless SQL Warehouse: SQL 쿼리 실행 후 수 초 내로 자원을 반납합니다.
- 단가는 높지만, **유휴 비용 제거 + 관리 오버헤드 감소** 효과로 총비용(TCO)이 오히려 낮아지는 경우가 많습니다.

---

## 7. 비용 추정 방법

### Step 1: Databricks 가격 계산기 활용

[Databricks 공식 가격 계산기](https://www.databricks.com/product/pricing)에서 다음을 입력하면 월 예상 비용을 산출할 수 있습니다.

- 클라우드 공급자 (AWS / Azure / GCP)
- 리전 (예: ap-northeast-2 서울)
- 플랜 (Standard / Premium / Enterprise)
- 워크로드 유형 및 예상 DBU 소비량

### Step 2: 파일럿 프로젝트 기반 실측

가격 계산기는 어디까지나 추정치입니다. **실제 비용은 파일럿을 통해 직접 측정하는 것이 가장 정확합니다.**

1. 대표 워크로드 2~3개를 선정합니다 (가장 빈번히 실행되는 ETL, 핵심 SQL 쿼리, ML 학습 작업).
2. 14일 ~ 30일간 실제 환경에서 실행합니다.
3. **System Table** `system.billing.usage` 를 조회해 SKU별 DBU 소비량을 정확하게 집계합니다.

```sql
-- DBU 소비량 조회 (System Table 활용)
SELECT
  sku_name,
  SUM(usage_quantity) AS total_dbu,
  usage_date
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE - INTERVAL 30 DAYS
GROUP BY sku_name, usage_date
ORDER BY usage_date DESC;
```

### Step 3: 기존 시스템 비용과 비교

Databricks 도입 전후 비용을 비교할 때는 **숨겨진 비용** 도 반드시 포함합니다.

| 기존 비용 항목 | 포함 여부 |
|--------------|---------|
| EMR / Redshift / SageMaker 사용료 | O |
| 데이터 파이프라인 개발 인력 비용 | O |
| ETL 도구 라이선스 (Informatica, Talend 등) | O |
| 클라우드 간 데이터 전송 비용 | O |
| 운영 및 유지보수 인력 비용 | O |

---

## 8. 흔한 비용 함정

### 함정 1: All-Purpose Compute 남용

**가장 흔하고 가장 비싼 실수** 입니다.

- 증상: 개발자가 노트북에서 작업하던 코드를 그대로 스케줄링합니다. 클러스터 유형을 Jobs로 변경하지 않습니다.
- 결과: 동일한 ETL 작업에 Jobs Compute 대비 2배 이상의 DBU 비용 발생합니다.
- 해결책: Lakeflow Jobs(Databricks Jobs) 작업 생성 시 기본 컴퓨팅 유형을 Jobs Compute로 설정하고, Cluster Policy로 강제합니다.

### 함정 2: 유휴 클러스터 방치

- 증상: 개발자가 퇴근 후 클러스터를 종료하지 않습니다. 주말 내내 VM이 켜져 있습니다.
- 결과: 아무 작업 없이 VM 비용과 최소 DBU가 계속 발생합니다.
- 해결책: **Auto-termination을 기본값으로 강제 설정** (Cluster Policy 활용), 관리자가 정기적으로 유휴 클러스터를 모니터링합니다.

### 함정 3: 과도한 인스턴스 크기 선택

- 증상: "더 큰 VM이 더 빠를 것"이라는 직관으로 무조건 대형 인스턴스를 선택합니다.
- 결과: 실제 메모리·CPU 사용률은 20~30%인데 비용만 낭비됩니다.
- 해결책: **Autoscaling** 을 활성화해 실제 작업 부하에 따라 클러스터가 자동으로 스케일 업/다운하도록 설정합니다.

### 함정 4: Photon 무분별한 적용

**Photon** 은 Delta Lake에 최적화된 벡터화 실행 엔진으로, 쿼리 성능을 2~10배 향상시킬 수 있습니다. 하지만:

- Photon이 적용된 SKU는 DBU 소비율이 1.5배 높습니다.
- 단순 Python 코드나 UDF(사용자 정의 함수) 위주의 작업에는 Photon 가속이 적용되지 않습니다.
- **SQL 집약적이거나 Delta 읽기/쓰기가 많은 워크로드** 에만 Photon을 적용합니다.

### 함정 5: SQL Warehouse를 항상 켜두기

- 증상: SQL Warehouse를 특정 크기로 생성한 후 **Auto-stop** 을 설정하지 않습니다.
- 결과: BI 도구에서 쿼리가 없는 시간에도 SQL Warehouse가 계속 실행됩니다.
- 해결책: Auto-stop을 10~30분으로 설정합니다. 첫 쿼리 시 웜업(Warm-up) 시간이 다소 걸리지만, 비용 절감 효과가 훨씬 큽니다.

---

## 참고 링크

- [Databricks: Pricing](https://www.databricks.com/product/pricing)
- [Databricks: DBU Pricing Overview (AWS)](https://aws.amazon.com/marketplace/pp/prodview-wtyi7772cyh5m)
- [Databricks Docs: Monitor usage with system tables](https://docs.databricks.com/en/admin/system-tables/billing.html)
- [Databricks Docs: Configure clusters](https://docs.databricks.com/en/compute/configure.html)
- [Databricks Docs: Serverless compute](https://docs.databricks.com/en/serverless-compute/index.html)
- [Databricks Docs: Cost management best practices](https://docs.databricks.com/en/admin/account-settings/usage.html)
- [Databricks: Get started with Databricks](https://docs.databricks.com/aws/en/getting-started/)
- [Azure Databricks: What is Azure Databricks?](https://learn.microsoft.com/en-us/azure/databricks/introduction/)
