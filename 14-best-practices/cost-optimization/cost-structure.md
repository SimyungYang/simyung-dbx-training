# 비용 구조 이해

## 이 문서에서 다루는 내용

Databricks 플랫폼의 비용을 체계적으로 관리하고 최적화하는 방법을 다룹니다. 비용 구조를 이해하는 것부터 시작하여, 컴퓨트/스토리지/SQL Warehouse 각 영역의 최적화 전략, 그리고 비용 모니터링 대시보드 구축까지 실전에서 바로 적용 가능한 가이드를 제공합니다.

---

## 1. 왜 비용 구조 이해가 중요한가

### 클라우드 비용 폭주 사례

클라우드 환경에서는 온프레미스와 달리 **잘못된 설정 하나가 수백만 원의 청구서** 로 이어질 수 있습니다. Databricks 환경에서 자주 발생하는 실제 패턴은 다음과 같습니다.

| 사례 | 원인 | 결과 |
|------|------|------|
| 주말 내내 켜진 대화형 클러스터 | Auto-termination 미설정 | 48시간 × All-Purpose DBU = 수백만 원 |
| BI 대시보드 자동 새로고침 | SQL Warehouse Always-On 설정 | 야간/주말에도 DBU 소모 |
| 실패한 Job의 무한 재시도 | max_retries 미설정 | DBU + 클라우드 스토리지 비용 누적 |
| 개발용 클러스터를 프로덕션에 사용 | All-Purpose를 Jobs 대신 사용 | 동일 작업 대비 2~3배 과금 |
| 리전 간 데이터 전송 | 멀티 리전 아키텍처 설계 오류 | 예상치 못한 Egress 비용 |

### 예상치 못한 청구 패턴

Databricks 청구서는 **DBU 비용(Databricks 청구) + 클라우드 인프라 비용(AWS/Azure/GCP 청구)** 두 가지로 나뉩니다. 많은 팀이 DBU 비용만 모니터링하다가 클라우드 청구서에서 충격을 받는 경우가 있습니다.

```
총 비용 = Databricks DBU 비용 + 클라우드 VM(EC2) 비용 + 스토리지 비용 + 네트워크 비용
```

> ⚠️ **주의**: Databricks 콘솔의 비용 모니터링은 DBU 소모량 중심입니다. EC2 인스턴스 비용, S3 스토리지 비용, NAT Gateway 비용은 AWS 청구서를 별도로 확인해야 합니다.

---

## 2. DBU(Databricks Unit) 단가 체계

### DBU란 무엇인가

Databricks의 과금은 **DBU(Databricks Unit)** 라는 정규화된 처리 단위를 기반으로 합니다. 동일한 작업이라도 어떤 SKU를 사용하느냐에 따라 DBU 소모량과 단가가 달라집니다.

> 💡 **DBU란?**Databricks의 과금 단위로, CPU 코어, 메모리, 컴퓨트 유형을 종합하여 정규화한 값입니다. 1 DBU는 약 1시간의 기본 컴퓨트 처리량에 해당합니다. 인스턴스 크기가 클수록 시간당 소모 DBU가 증가합니다.

### SKU별 가격 비교표 (AWS 기준, 상대 단가)

| SKU 유형 | 대표 용도 | 상대 단가 | 특징 |
|----------|----------|----------|------|
| **Jobs Compute** | 스케줄된 배치 작업 | ★☆☆☆☆ (최저) | 가장 경제적, 프로덕션 파이프라인 권장 |
| **Jobs Compute (Serverless)** | 서버리스 배치 작업 | ★★☆☆☆ | 인프라 관리 불필요, 빠른 시작 |
| **All-Purpose Compute** | 인터랙티브 개발, 노트북 | ★★★★☆ | 개발 시에만 사용, 프로덕션 금지 |
| **All-Purpose (Serverless)** | 서버리스 인터랙티브 | ★★★☆☆ | 유휴 비용 없음, 개발자 생산성 극대화 |
| **SQL Warehouse (Serverless)** | BI 쿼리, 대시보드 | ★★★☆☆ | 자동 스케일링, 캐싱 내장 |
| **SQL Warehouse (Pro)** | BI 쿼리 (클래식) | ★★★☆☆ | 수동 스케일링, 예측 가능한 비용 |
| **Model Serving** | 실시간 추론 | ★★★★☆ | GPU 사용 시 비용 급증 |
| **Foundation Model API** | DBRX, Llama 호출 | 토큰 기반 | 입력/출력 토큰 수에 비례 |

> ⚠️ **핵심 원칙**: 개발은 All-Purpose, 프로덕션은 Jobs Compute를 사용하면 **동일 작업 대비 2~3배 비용 절감** 이 가능합니다.

### Commit vs Pay-as-you-go

Databricks는 두 가지 구매 모델을 제공합니다.

| 구분 | Pay-as-you-go | Pre-purchase Commit (PUPM) |
|------|--------------|---------------------------|
| 단가 | 정가 (최고) | 최대 40~50% 할인 |
| 약정 | 없음 | 1년 또는 3년 약정 |
| 유연성 | 매우 높음 | 낮음 (미소진 시 소멸) |
| 적합 대상 | POC, 소규모, 불규칙 워크로드 | 안정적 프로덕션 워크로드 |
| 리스크 | 예산 초과 | 미소진 DBU 손실 |

> 💡 **전략**: 전체 DBU 소모량의 70~80%는 Commit으로 커버하고, 나머지 피크 부하는 Pay-as-you-go로 처리하는 혼합 전략이 일반적입니다.

---

## 3. 비용 발생 3대 영역

### 전체 비중

| 비용 카테고리 | 일반적 비중 | 세부 항목 |
|-------------|------------|---------|
| **컴퓨트** | 60~80% | 클러스터 DBU, SQL Warehouse DBU, Serverless DBU, Jobs DBU |
| **스토리지** | 10~20% | S3/ADLS 저장, Delta 트랜잭션 로그, 스냅샷 보존 |
| **네트워크** | 5~10% | 리전 간 전송, PrivateLink, NAT Gateway |

### 컴퓨트 비용: 클래식 vs 서버리스

**클래식 컴퓨트 (Classic Compute)**

- 사용자가 클러스터를 직접 생성하고 관리합니다.
- 클러스터가 기동 중인 시간 전체에 대해 DBU가 부과됩니다 (쿼리 실행 여부와 무관).
- Auto-termination을 반드시 설정해야 유휴 비용을 방지할 수 있습니다.
- EC2 인스턴스 비용은 AWS 청구서에 별도로 발생합니다.

**서버리스 컴퓨트 (Serverless Compute)**

- Databricks가 인프라를 완전히 관리합니다.
- 쿼리/작업 실행 시간에만 DBU가 부과됩니다 (유휴 비용 없음).
- EC2 인스턴스 비용이 DBU 단가에 포함되어 있습니다 (AWS 청구서에 별도 항목 없음).
- 단위 DBU 단가는 클래식보다 높지만, 유휴 시간 제거로 총비용은 낮아지는 경우가 많습니다.

```
클래식 총비용 = (기동 시간 × DBU/시간 × DBU 단가) + EC2 비용
서버리스 총비용 = (실행 시간 × DBU/시간 × 서버리스 DBU 단가)
```

### 스토리지 비용: Delta 로그 비용

Delta Lake는 모든 트랜잭션을 `_delta_log/` 디렉터리에 JSON 파일로 기록합니다. 테이블에 잦은 소규모 쓰기가 발생하면 Delta 로그 파일 수가 폭발적으로 증가하고, 이는 스토리지 비용과 쿼리 성능 저하로 이어집니다.

```python
# Delta 로그 정리 - OPTIMIZE + VACUUM 주기적 실행
spark.sql("OPTIMIZE my_catalog.my_schema.my_table")
spark.sql("VACUUM my_catalog.my_schema.my_table RETAIN 168 HOURS")
```

### 네트워크 비용: Egress 비용

| 전송 유형 | 비용 발생 여부 | 비고 |
|-----------|-------------|------|
| 동일 AZ 내 전송 | 없음 | 클러스터와 S3가 같은 AZ |
| 리전 내 AZ 간 전송 | 발생 | 약 $0.01/GB |
| 리전 간(Cross-Region) 전송 | 발생 | 약 $0.02~0.09/GB |
| 인터넷 Egress | 가장 비쌈 | 약 $0.09/GB |
| PrivateLink 사용 | 시간 + 데이터 요금 | 엔드포인트당 월 고정 비용 발생 |

> 💡 **팁**: 클러스터와 S3 버킷을 같은 리전/AZ에 배치하면 네트워크 비용을 크게 줄일 수 있습니다.

---

## 4. 숨겨진 비용 (Hidden Costs)

많은 조직이 예상치 못하게 만나는 숨겨진 비용 항목들입니다.

### Unity Catalog 서버리스 컴퓨트

Unity Catalog의 일부 기능(Data Lineage, Automated Data Quality 등)은 내부적으로 서버리스 컴퓨트를 사용합니다. 이 비용은 `system.billing.usage` 테이블에서 `sku_name`이 `ENTERPRISE_ALL_PURPOSE_COMPUTE_SERVERLESS`로 기록됩니다.

### Photon 추가 DBU

Photon 엔진을 활성화하면 동일 인스턴스에서 더 많은 DBU가 소모됩니다. Photon은 벡터화 실행 엔진으로 쿼리 속도를 높이지만, **DBU 소모량도 함께 증가** 합니다. 따라서 Photon의 이점(처리 속도 향상 → 실행 시간 단축)이 DBU 단가 증가를 상쇄하는지 워크로드별로 검증해야 합니다.

### 실패한 Job 재시도 비용

Job이 실패하더라도 클러스터가 기동되고 일부 코드가 실행된 시간만큼 DBU가 과금됩니다. `max_retries`가 높게 설정된 경우 실패한 Job이 반복 재시도되면서 비용이 누적됩니다.

```json
// Job 설정 예시 - 재시도 횟수 제한
{
  "max_retries": 1,
  "min_retry_interval_millis": 60000
}
```

### DBFS 루트 스토리지

DBFS(Databricks File System) 루트 스토리지는 워크스페이스가 프로비저닝될 때 자동으로 생성되는 S3 버킷입니다. 여기에 저장된 데이터(오래된 체크포인트, 임시 파일, 실험 결과 등)는 명시적으로 삭제하지 않으면 계속 스토리지 비용이 발생합니다. Unity Catalog 도입 후에는 DBFS 루트 사용을 최소화하고 External Location으로 마이그레이션하는 것을 권장합니다.

### Delta Live Tables (DLT) 오버헤드

DLT 파이프라인은 내부적으로 추가 오케스트레이션 컴퓨트를 사용합니다. 소규모 파이프라인에서는 오케스트레이션 오버헤드 비율이 높아질 수 있으므로, DLT가 적합한 워크로드인지 사전에 검토가 필요합니다.

---

## 5. 비용 최적화 핵심 전략

### Jobs Compute vs All-Purpose Compute

가장 즉각적인 비용 절감 효과를 낼 수 있는 전략입니다.

| 항목 | All-Purpose | Jobs Compute |
|------|------------|-------------|
| 주요 용도 | 인터랙티브 개발, 노트북 | 스케줄된 파이프라인 |
| DBU 단가 비율 | 1.0 (기준) | ~0.4~0.5 |
| 클러스터 재사용 | 가능 | 작업당 새 클러스터 |
| Multi-task 지원 | 제한적 | 완전 지원 |
| **권장 사용처** | 개발/디버깅 전용 | 프로덕션 전용 |

### Spot Instance (스팟 인스턴스) 활용

AWS Spot Instance를 Worker 노드에 적용하면 On-Demand 대비 **60~90% EC2 비용 절감** 이 가능합니다.

```python
# 클러스터 설정에서 Spot Instance 활성화
{
  "aws_attributes": {
    "availability": "SPOT_WITH_FALLBACK",  # Spot 불가 시 On-Demand로 자동 전환
    "spot_bid_price_percent": 100           # On-Demand 가격의 100%까지 입찰
  }
}
```

> ⚠️ **주의**: Driver 노드는 On-Demand로 유지하세요. Spot 중단 시 전체 작업이 실패합니다.

### Auto-termination 설정

All-Purpose 클러스터는 반드시 Auto-termination을 설정해야 합니다.

```python
# 권장: 10~30분 유휴 시 자동 종료
{
  "autotermination_minutes": 20
}
```

| 클러스터 유형 | 권장 Auto-termination |
|-------------|----------------------|
| 개인 개발 클러스터 | 10~20분 |
| 공유 개발 클러스터 | 30~60분 |
| 프로덕션 Jobs | 해당 없음 (작업 완료 후 자동 종료) |

### Serverless 활용 전략

서버리스 컴퓨트는 유휴 비용이 없으므로, **간헐적 워크로드** 에 특히 효과적입니다.

- **SQL Warehouse Serverless**: BI 대시보드, 임시 쿼리 — 비사용 시 자동 중지
- **Jobs Serverless**: 일별/주별 배치 — 클러스터 시작 시간(~2분) 절약
- **All-Purpose Serverless**: 개발자 노트북 — 유휴 시간 비용 제거

### Photon 활용 판단 기준

Photon은 모든 워크로드에 적합하지 않습니다.

| 워크로드 유형 | Photon 권장 여부 | 이유 |
|-------------|----------------|------|
| 대용량 집계/조인 쿼리 | 강력 권장 | 벡터화 실행으로 10x 속도 향상 가능 |
| SQL 분석 워크로드 | 권장 | SQL Warehouse에 기본 포함 |
| Python UDF 중심 워크로드 | 비권장 | Photon은 Python UDF 미지원 |
| 소규모 데이터셋 | 비권장 | 오버헤드 대비 이점 미미 |
| ML 학습(GPU) | 비권장 | GPU 워크로드와 무관 |

---

## 6. Commit vs Pay-as-you-go 전략

### 언제 Commit을 선택해야 하는가

**Pre-purchase Commit(사전 구매 약정)** 은 다음 조건을 모두 충족할 때 선택합니다.

1. 월간 DBU 소모량이 예측 가능하고 안정적입니다.
2. 최소 6개월 이상 플랫폼을 지속적으로 사용할 계획이 있습니다.
3. 약정 DBU의 80% 이상을 실제로 소진할 수 있습니다.

### 혼합 전략 (권장)

```
전체 예상 DBU = 기본 워크로드 (예측 가능) + 피크 워크로드 (불규칙)

Commit 대상 = 기본 워크로드의 70~80%
Pay-as-you-go = 나머지 20~30% + 피크 워크로드
```

### 조직 유형별 권장 모델

| 조직 유형 | 권장 모델 | 이유 |
|-----------|---------|------|
| 스타트업 / POC 단계 | Pay-as-you-go | 사용량 불확실, 유연성 필요 |
| 성장기 기업 | 혼합 (Commit 50~70%) | 안정적 기반 + 유연성 유지 |
| 엔터프라이즈 / 안정 운영 | Commit 중심 (80%+) | 최대 할인 효과 |
| 계절적 워크로드 | Pay-as-you-go 또는 단기 Commit | 성수기/비수기 편차 대응 |

---

## 7. 비용 가시성 확보

### system.billing.usage 테이블

Unity Catalog의 **System Tables** 는 Databricks 사용량 데이터를 SQL로 직접 조회할 수 있는 가장 강력한 도구입니다.

```sql
-- 최근 30일 SKU별 DBU 소모량 조회
SELECT
  sku_name,
  SUM(usage_quantity) AS total_dbu,
  COUNT(DISTINCT workspace_id) AS workspace_count
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE - INTERVAL 30 DAYS
GROUP BY sku_name
ORDER BY total_dbu DESC;
```

```sql
-- 팀별 비용 추적 (클러스터 태그 활용)
SELECT
  usage_metadata.cluster_id,
  custom_tags['team'] AS team,
  custom_tags['project'] AS project,
  SUM(usage_quantity) AS total_dbu
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE - INTERVAL 7 DAYS
  AND custom_tags['team'] IS NOT NULL
GROUP BY 1, 2, 3
ORDER BY total_dbu DESC;
```

### 태그 기반 비용 추적 (Tag-based Cost Tracking)

태그(Tag)는 클러스터, Job, SQL Warehouse에 키-값 쌍으로 부착하여 비용을 분류하는 핵심 수단입니다.

**권장 태그 체계:**

```
team: data-engineering
project: customer-360
env: production
cost-center: DE-001
owner: john.doe@company.com
```

**클러스터 정책에서 태그 강제화:**

```json
{
  "custom_tags.team": {
    "type": "fixed",
    "value": "data-engineering"
  },
  "custom_tags.project": {
    "type": "allowlist",
    "values": ["customer-360", "fraud-detection", "recommendation"]
  }
}
```

### 클러스터 정책 (Cluster Policy)

클러스터 정책은 사용자가 생성할 수 있는 클러스터의 사양을 제한하여 **비용 폭주를 구조적으로 방지** 합니다.

```json
{
  "autotermination_minutes": {
    "type": "fixed",
    "value": 30
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["m5.xlarge", "m5.2xlarge", "m5.4xlarge"]
  },
  "num_workers": {
    "type": "range",
    "maxValue": 10
  }
}
```

---

## 8. 흔한 실수와 베스트 프랙티스

### 흔한 실수 Top 5

| 순위 | 실수 | 결과 | 해결책 |
|------|------|------|--------|
| 1 | All-Purpose로 프로덕션 파이프라인 실행 | 2~3배 과금 | Jobs Compute로 전환 |
| 2 | Auto-termination 미설정 | 주말 내내 클러스터 가동 | 정책으로 강제화 |
| 3 | 클러스터 과다 프로비저닝 | EC2 비용 낭비 | Auto-scaling 활성화 |
| 4 | 태그 없이 클러스터 생성 | 비용 추적 불가 | 클러스터 정책으로 태그 필수화 |
| 5 | VACUUM 미실행 | Delta 로그/오래된 파일로 스토리지 낭비 | 주기적 OPTIMIZE + VACUUM |

### 베스트 프랙티스 체크리스트

**컴퓨트 최적화**
- [ ] 프로덕션 Job은 Jobs Compute SKU 사용
- [ ] All-Purpose 클러스터에 Auto-termination 설정 (20분 이하)
- [ ] Worker 노드에 Spot Instance 적용 (`SPOT_WITH_FALLBACK`)
- [ ] Auto-scaling 설정 (min: 2, max: 적절히 제한)
- [ ] 클러스터 정책으로 최대 노드 수 제한

**스토리지 최적화**
- [ ] Delta 테이블에 주간 OPTIMIZE 실행
- [ ] VACUUM으로 오래된 파일 정리 (최소 7일 보존 후 삭제)
- [ ] DBFS 루트 대신 External Location 사용
- [ ] 테이블 파티셔닝 전략 검토 (과도한 소규모 파티션 방지)

**비용 가시성**
- [ ] 모든 클러스터/Job에 team, project, env 태그 부착
- [ ] `system.billing.usage` 기반 주간 비용 리포트 자동화
- [ ] 이상 비용 발생 시 Slack 알림 설정
- [ ] 월별 SKU별 비용 추이 대시보드 유지

---

## 참고 링크

- [Databricks Pricing 공식 페이지](https://www.databricks.com/product/pricing)
- [Databricks DBU 소비 최적화 가이드](https://docs.databricks.com/en/lakehouse-architecture/cost-optimization/index.html)
- [System Tables — billing.usage](https://docs.databricks.com/en/admin/system-tables/billing.html)
- [클러스터 정책 설정](https://docs.databricks.com/en/administration-guide/clusters/policies.html)
- [Delta Lake OPTIMIZE 및 VACUUM](https://docs.databricks.com/en/delta/optimize.html)
- [Serverless Compute 개요](https://docs.databricks.com/en/compute/serverless/index.html)
- [Spot Instance 설정 (AWS)](https://docs.databricks.com/en/compute/configure.html#aws-availability)
- [Unity Catalog System Tables](https://docs.databricks.com/en/admin/system-tables/index.html)
