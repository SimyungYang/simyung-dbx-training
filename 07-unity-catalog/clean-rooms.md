# Clean Rooms — 데이터를 공유하지 않는 공동 분석

## 개념

> 💡 **Clean Room(클린 룸)** 은 여러 조직이 **원본 데이터를 서로 노출하지 않으면서** 공동 분석을 수행할 수 있는 안전한 협업 환경입니다. 각 참여자의 데이터는 암호화된 상태로 유지되며, 사전에 합의된 연산만 실행할 수 있습니다.

### 왜 Clean Room이 필요한가요?

기업 간 데이터 협업은 큰 가치를 만들어낼 수 있지만, 현실적으로 여러 장벽이 존재합니다.

| 기존 협업 방법 | 문제점 |
|---------------|--------|
| **데이터 직접 전달** | 개인정보 유출 위험, GDPR/CCPA 위반 가능 |
| **API로 결과만 공유** | 개발 비용 높음, 유연성 부족 |
| **공유 데이터베이스** | 원본 데이터 노출, 권한 관리 어려움 |
| **수기 보고서 교환** | 실시간성 없음, 분석 범위 제한 |

> 💡 **GDPR**(General Data Protection Regulation)은 EU의 개인정보 보호 규정이고, **CCPA**(California Consumer Privacy Act)는 미국 캘리포니아주의 소비자 프라이버시 법입니다. 두 규정 모두 개인정보의 제3자 공유를 엄격히 제한합니다.

Clean Room은 이러한 문제를 해결합니다.

| Clean Room 장점 | 설명 |
|-----------------|------|
| **데이터 비노출** | 각 참여자의 원본 데이터는 상대방에게 보이지 않습니다 |
| **규제 준수** | GDPR, CCPA, HIPAA 등 프라이버시 규정을 충족합니다 |
| **합의된 연산만 실행** | 사전에 정의된 분석(집계, 매칭 등)만 허용됩니다 |
| **감사 추적** | 누가 어떤 분석을 실행했는지 모든 기록이 남습니다 |
| **중립적 환경** | 어느 한쪽의 인프라에 종속되지 않습니다 |

---

## Clean Room 아키텍처

Databricks Clean Room은 **Unity Catalog** 와 **Delta Sharing** 위에 구축된 보안 계층입니다.

### 참여 역할

| 역할 | 설명 |
|------|------|
| **생성자 (Creator)** | Clean Room을 만들고, 자신의 데이터를 등록하며, 허용 연산을 정의합니다 |
| **협업자 (Collaborator)** | Clean Room에 참여하여 자신의 데이터를 등록하고, 합의된 분석을 실행합니다 |

### 동작 원리

| 참여자 | 데이터 | 원본 노출 |
|--------|--------|---------|
| **조직 A**(생성자) | 고객 구매 데이터 | 비노출 |
| **조직 B**(협업자) | 광고 노출 데이터 | 비노출 |

**Databricks Clean Room 규칙**:
- 사전 합의된 연산만 실행
- 집계 결과만 반환
- 원본 데이터 접근 불가
- 모든 실행 기록 감사

> 최종 결과: **집계 결과만 공유**(개별 레코드 없음)

### 핵심 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **Clean Room** | 보안이 적용된 협업 공간입니다. Unity Catalog 메타스토어 위에 생성됩니다 |
| **Shared Table** | 각 참여자가 Clean Room에 등록한 테이블입니다. 상대방은 원본을 볼 수 없습니다 |
| **Notebook Template** | Clean Room 안에서 실행할 수 있는 사전 승인된 분석 코드입니다 |
| **Output Table** | 분석 결과가 저장되는 테이블입니다. 참여자가 확인할 수 있습니다 |

---

## Clean Room 생성 및 참여

### 1단계: Clean Room 생성 (생성자)

Databricks 워크스페이스의 **Data** 탭에서 Clean Room을 생성합니다.

```sql
-- Clean Room은 현재 UI 및 REST API를 통해 생성합니다.
-- 아래는 Clean Room에 데이터를 등록하는 개념적 흐름입니다.

-- 생성자: 공유할 테이블 준비
CREATE TABLE IF NOT EXISTS gold.customer_purchases (
  customer_id STRING,
  purchase_date DATE,
  product_category STRING,
  amount DOUBLE
);
```

### 2단계: 협업자 초대

```
Clean Room 생성 후:
1. "Add Collaborator" 클릭
2. 협업자의 Databricks Sharing Identifier 입력
3. 협업자에게 초대 전송
```

### 3단계: 데이터 등록

각 참여자는 자신의 테이블을 Clean Room에 등록합니다.

| 역할 | 테이블 | 설명 |
|------|--------|------|
| **생성자** | gold.customer_purchases | 고객 구매 기록 |
| | gold.customer_segments | 고객 세그먼트 |
| **협업자** | gold.ad_impressions | 광고 노출 기록 |
| | gold.campaign_metadata | 캠페인 메타데이터 |

### 4단계: 노트북 템플릿 정의 및 실행

```python
# Clean Room 노트북 템플릿 예시
# 생성자가 작성하고, 양측이 합의한 분석만 실행 가능

# 두 조직의 데이터를 매칭하여 집계 결과만 반환
result = spark.sql("""
    SELECT
        c.product_category,
        a.campaign_name,
        COUNT(DISTINCT c.customer_id) AS matched_customers,
        SUM(c.amount) AS total_revenue
    FROM creator.customer_purchases c
    JOIN collaborator.ad_impressions a
        ON c.customer_id = a.user_id
    JOIN collaborator.campaign_metadata cm
        ON a.campaign_id = cm.campaign_id
    GROUP BY c.product_category, a.campaign_name
    HAVING COUNT(DISTINCT c.customer_id) >= 50  -- 최소 집계 단위 보장
""")
```

> ⚠️ **중요**: Clean Room에서는 개별 레코드를 반환하는 쿼리는 허용되지 않습니다. 반드시 집계(GROUP BY)를 통해 결과를 생성해야 하며, 최소 집계 단위(k-anonymity)를 설정하여 개인 식별을 방지합니다.

---

## 허용 연산 정의

Clean Room의 핵심은 **어떤 연산을 허용할 것인가** 를 사전에 합의하는 것입니다.

### 연산 유형별 허용 여부

| 연산 유형 | 허용 여부 | 설명 |
|-----------|-----------|------|
| **집계 (Aggregation)** | ✅ 허용 | COUNT, SUM, AVG 등 그룹별 통계 |
| **매칭 (Overlap Analysis)** | ✅ 허용 | 두 데이터셋의 공통 사용자 수 계산 |
| **세그먼트 분석** | ✅ 허용 | 특정 조건의 그룹 크기와 통계 |
| **개별 레코드 조회** | ❌ 차단 | SELECT * 같은 원본 데이터 열람 |
| **데이터 내보내기** | ❌ 차단 | 원본 데이터를 외부로 복사 |
| **역추론 쿼리** | ❌ 차단 | WHERE 조건으로 개인을 특정하는 시도 |

### 프라이버시 보호 메커니즘

| 프라이버시 보호 계층 | 설명 |
|-------------------|------|
| **k-Anonymity** | 최소 k명 이상의 그룹만 결과에 포함 |
| **집계 강제** | GROUP BY 없는 쿼리 차단 |
| **컬럼 접근 제한** | 민감 컬럼은 매칭 키로만 사용 가능 |
| **감사 로그** | 모든 쿼리와 결과를 기록 |

---

## 활용 사례

### 사례 1: 광고 성과 분석

소매업체와 광고 플랫폼이 개인정보를 공유하지 않고 광고 효과를 측정합니다.

```python
# 소매업체(생성자): 구매 데이터 보유
# 광고 플랫폼(협업자): 광고 노출 데이터 보유

# Clean Room에서 실행하는 분석
overlap_analysis = spark.sql("""
    SELECT
        ad.campaign_name,
        COUNT(DISTINCT purchase.customer_id) AS converters,
        AVG(purchase.amount) AS avg_order_value,
        SUM(purchase.amount) AS total_attributed_revenue
    FROM creator.purchases purchase
    JOIN collaborator.ad_exposures ad
        ON purchase.hashed_email = ad.hashed_email
    WHERE purchase.purchase_date BETWEEN ad.exposure_date
        AND DATEADD(DAY, 14, ad.exposure_date)
    GROUP BY ad.campaign_name
    HAVING COUNT(DISTINCT purchase.customer_id) >= 100
""")
```

### 사례 2: 의료 데이터 공동 연구

병원과 제약사가 환자 데이터를 노출하지 않고 임상 연구를 수행합니다.

```python
# 병원(생성자): 진료 기록 보유
# 제약사(협업자): 약물 투여 기록 보유

clinical_study = spark.sql("""
    SELECT
        pharma.drug_name,
        hospital.age_group,
        COUNT(*) AS patient_count,
        AVG(hospital.recovery_days) AS avg_recovery_days,
        STDDEV(hospital.recovery_days) AS stddev_recovery_days
    FROM creator.patient_outcomes hospital
    JOIN collaborator.drug_administration pharma
        ON hospital.anonymized_patient_id = pharma.anonymized_patient_id
    GROUP BY pharma.drug_name, hospital.age_group
    HAVING COUNT(*) >= 30  -- 통계적 유의성 및 프라이버시 보장
""")
```

### 사례 3: 금융 사기 탐지

여러 금융 기관이 거래 패턴을 공유하지 않고 사기 의심 계정을 탐지합니다.

| 참여자 | 제공 데이터 | 목적 |
|--------|-------------|------|
| **은행 A** | 이상 거래 패턴 | 교차 검증 |
| **은행 B** | 이상 거래 패턴 | 교차 검증 |
| **분석 결과** | 공통 의심 계정 수 (집계) | 사기 탐지 강화 |

---

## Delta Sharing과의 비교

| 항목 | Delta Sharing | Clean Room |
|------|--------------|------------|
| **데이터 가시성** | 수신자가 원본 데이터를 볼 수 있음 | 원본 데이터를 볼 수 없음 |
| **연산 제어** | 수신자가 자유롭게 분석 | 사전 합의된 연산만 가능 |
| **방향** | 단방향 (제공자 → 수신자) | 양방향 (양측 데이터 결합) |
| **프라이버시** | 데이터 공유 기반 | 데이터 비공유 기반 |
| **적합한 상황** | 파트너에게 데이터 제공 | 민감 데이터 공동 분석 |

---

## 주의사항 및 제한

| 항목 | 내용 |
|------|------|
| **Unity Catalog 필수** | Clean Room은 Unity Catalog가 활성화된 워크스페이스에서만 사용 가능합니다 |
| **Databricks 간 전용** | 현재 양측 모두 Databricks를 사용해야 합니다 |
| **노트북 승인** | 노트북 템플릿은 양측이 합의해야 실행할 수 있습니다 |
| **컴퓨팅 비용** | Clean Room 실행 시 생성자의 컴퓨팅 리소스가 사용됩니다 |
| **리전 제한** | 동일 클라우드 리전 내에서의 협업이 가장 효율적입니다 |

> 🆕 **[2025 업데이트]**Clean Room은 지속적으로 기능이 확장되고 있습니다. 최신 지원 범위와 제한사항은 공식 문서를 확인하세요.

---

## 정리

| 핵심 개념 | 요약 |
|-----------|------|
| **Clean Room** | 데이터를 공유하지 않고 공동 분석하는 보안 환경 |
| **생성자/협업자** | Clean Room을 만드는 쪽과 참여하는 쪽 |
| **노트북 템플릿** | 양측이 합의한 분석 코드만 실행 가능 |
| **프라이버시 보장** | 집계 결과만 반환, 개별 레코드 접근 불가 |
| **활용 분야** | 광고 성과, 의료 연구, 금융 사기 탐지 등 |

---

## 참고 링크

- [Databricks Clean Rooms 공식 문서](https://docs.databricks.com/aws/en/clean-rooms/)
- [Unity Catalog 개요](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)
- [Delta Sharing 문서](https://docs.databricks.com/aws/en/delta-sharing/)
- [Databricks Blog: Clean Rooms](https://www.databricks.com/blog/announcing-databricks-clean-rooms)
