# Delta Sharing — 조직 간 데이터 공유

## 개념

> 💡 **Delta Sharing** 은 조직 간에 데이터를 **안전하게 공유** 할 수 있는 **오픈 프로토콜** 입니다. 데이터를 복사하지 않고, 수신자에게 읽기 권한만 부여하여 원본 데이터에 직접 접근할 수 있게 합니다.

### 왜 Delta Sharing이 필요한가요?

전통적으로 조직 간 데이터 공유는 다음과 같은 방법으로 이루어졌습니다.

| 기존 방법 | 문제점 |
|-----------|--------|
| **파일 전송 (SFTP, Email)**| 데이터 복사 → 보안 위험, 버전 불일치, 대용량 불가 |
| **API 구축**| 개발 비용 높음, 유지보수 부담 |
| **공유 DB 접근 권한**| 보안 위험, 세밀한 제어 어려움 |
| **클라우드 스토리지 공유**| 플랫폼 종속, 권한 관리 복잡 |

Delta Sharing은 이 문제를 해결합니다.

| Delta Sharing 장점 | 설명 |
|--------------------|------|
| **데이터 복사 없음**| 수신자가 원본 데이터를 직접 읽습니다. 복사본이 생기지 않습니다 |
| **오픈 프로토콜**| Databricks 없이도 pandas, Spark, Power BI 등으로 읽을 수 있습니다 |
| **세밀한 제어**| 테이블/파티션 단위로 공유 범위를 지정합니다 |
| **실시간 최신 데이터**| 원본 테이블이 갱신되면 수신자도 최신 데이터를 봅니다 |
| **감사 추적**| 누가, 언제, 어떤 데이터에 접근했는지 기록됩니다 |

---

## Delta Sharing의 구성 요소

| 역할 | 구성 요소 | 설명 |
|------|-----------|------|
| **제공자 (Provider)**| Delta 테이블 | 공유할 원본 데이터입니다 |
|  | Share (공유 단위) | 테이블을 묶어 공유하는 단위입니다 |
| **수신자 (Recipient)**| Databricks (다른 워크스페이스) | 읽기 권한으로 데이터에 접근합니다 |
|  | Spark (외부 클러스터) | 읽기 권한으로 데이터에 접근합니다 |
|  | pandas (Python) | 읽기 권한으로 데이터에 접근합니다 |
|  | Power BI (BI 도구) | 읽기 권한으로 데이터에 접근합니다 |

| 구성 요소 | 설명 |
|-----------|------|
| **Provider (제공자)**| 데이터를 공유하는 쪽입니다. Share를 생성하고 테이블을 추가합니다 |
| **Share**| 공유 단위입니다. 하나의 Share에 여러 테이블을 포함할 수 있습니다 |
| **Recipient (수신자)**| 데이터를 받는 쪽입니다. Databricks 사용자일 수도, 외부 사용자일 수도 있습니다 |
| **Sharing Server**| Unity Catalog가 제공하는 공유 서버입니다 |

---

## 공유 방법

### Databricks-to-Databricks 공유

같은 Databricks 계정 내의 다른 워크스페이스 또는 다른 Databricks 계정과 공유합니다.

```sql
-- 1. Share 생성
CREATE SHARE IF NOT EXISTS customer_insights
COMMENT '고객 분석 데이터 공유';

-- 2. Share에 테이블 추가
ALTER SHARE customer_insights
ADD TABLE gold.customer_summary;

-- 특정 파티션만 공유 (선택)
ALTER SHARE customer_insights
ADD TABLE gold.regional_sales
PARTITION (region = '서울');

-- 3. 수신자 생성 (Databricks 계정)
CREATE RECIPIENT IF NOT EXISTS partner_analytics
USING ID '<수신자-sharing-identifier>';

-- 4. 수신자에게 Share 접근 권한 부여
GRANT SELECT ON SHARE customer_insights TO RECIPIENT partner_analytics;
```

### Open Sharing (외부 수신자)

Databricks를 사용하지 않는 외부 조직에도 데이터를 공유할 수 있습니다.

```sql
-- 외부 수신자 생성 (인증 토큰 방식)
CREATE RECIPIENT IF NOT EXISTS external_partner
COMMENT '외부 파트너사';

-- 활성화 링크/토큰 확인
DESCRIBE RECIPIENT external_partner;
-- → activation_link가 생성됩니다. 이 링크를 수신자에게 전달합니다.
```

수신자는 활성화 링크를 통해 **크레덴셜 파일** 을 다운로드하고, 이를 사용하여 데이터에 접근합니다.

---

## 수신자 측에서 데이터 접근

### Python (pandas)

```python
import delta_sharing

# 프로필 파일 경로 (활성화 시 다운로드한 파일)
profile_file = "/path/to/config.share"

# 사용 가능한 테이블 목록 확인
client = delta_sharing.SharingClient(profile_file)
print(client.list_all_tables())

# 데이터를 pandas DataFrame으로 로드
df = delta_sharing.load_as_pandas(
    f"{profile_file}#share_name.schema_name.table_name"
)
print(df.head())
```

### Apache Spark

```python
df = (spark.read
    .format("deltaSharing")
    .load(f"{profile_file}#share_name.schema_name.table_name")
)
df.show()
```

### Power BI

Power BI에서도 Delta Sharing 커넥터를 통해 공유 데이터에 직접 연결할 수 있습니다.

---

## 공유 관리

```sql
-- Share에 포함된 테이블 확인
SHOW ALL IN SHARE customer_insights;

-- Share에서 테이블 제거
ALTER SHARE customer_insights
REMOVE TABLE gold.customer_summary;

-- 수신자 목록 확인
SHOW RECIPIENTS;

-- 수신자의 접근 기록 확인
SHOW GRANTS ON SHARE customer_insights;

-- 수신자 삭제 (접근 권한 즉시 해제)
DROP RECIPIENT external_partner;

-- Share 삭제
DROP SHARE customer_insights;
```

---

## 활용 시나리오

| 시나리오 | 설명 |
|----------|------|
| **본사 ↔ 자회사**| 본사의 매출 데이터를 자회사에 공유합니다 |
| **데이터 제공 사업**| 데이터를 상품으로 외부 고객에게 판매합니다 |
| **파트너 협업**| 공급망 데이터를 파트너사와 실시간 공유합니다 |
| **규제 보고**| 규제 기관에 특정 데이터를 안전하게 공유합니다 |
| **멀티 리전**| 다른 리전/클라우드의 Databricks와 데이터를 공유합니다 |

---

## 보안 고려사항

| 항목 | 설명 |
|------|------|
| **읽기 전용**| 수신자는 데이터를 읽기만 할 수 있고, 수정/삭제할 수 없습니다 |
| **세밀한 범위**| 테이블 단위, 파티션 단위로 공유 범위를 제한합니다 |
| **감사 로그**| 모든 공유 접근이 `system.access.audit`에 기록됩니다 |
| **수신자 제거**| 수신자를 삭제하면 접근 권한이 즉시 해제됩니다 |
| **토큰 만료**| Open Sharing의 토큰에 만료 기간을 설정할 수 있습니다 |
| **IP 제한**| 수신자의 접근 IP를 제한할 수 있습니다 |

---

## 심화: Principal SA 레벨 운영 가이드

### 성능 특성

Delta Sharing은 수신자가 **Parquet 파일을 직접 읽는 구조** 이므로, 성능 특성을 정확히 이해해야 합니다.

| 항목 | 상세 |
|------|------|
| **파티션 프루닝**| 제공자가 파티션 단위로 공유하면, 수신자 측에서 불필요한 파티션을 스킵합니다. `date` 또는 `region` 기준 파티셔닝을 권장합니다 |
| **Parquet 통계 활용**| Parquet footer의 min/max 통계를 활용한 Data Skipping이 수신자 측에서도 동작합니다 |
| **네트워크 병목**| 수신자가 대용량 테이블(수 TB)을 읽을 때, 제공자의 클라우드 스토리지에서 직접 파일을 다운로드합니다. **Cross-region 읽기 시 네트워크 지연이 발생** 할 수 있으며, egress 비용이 제공자에게 청구됩니다 |
| **동시 읽기 제한**| 수신자가 대규모 병렬 읽기를 수행하면 제공자의 S3/ADLS API 호출 한도에 영향을 줄 수 있습니다. S3는 prefix당 초당 5,500 GET 요청까지 허용합니다 |
| **Change Data Feed**| Databricks-to-Databricks 공유 시 CDF(Change Data Feed)를 활용하면 수신자가 변경분만 읽을 수 있어 대폭 효율적입니다 |

> 💡 **성능 최적화 팁**: 대용량 공유 시 `Z-ORDER` 또는 `LIQUID CLUSTERING`이 적용된 테이블을 공유하면, 수신자의 쿼리가 Data Skipping 혜택을 극대화할 수 있습니다.

---

### 크로스 클라우드 및 리전 간 공유

Delta Sharing의 가장 강력한 사용 사례 중 하나는 **클라우드 경계를 넘는 데이터 공유** 입니다. 그러나 비용과 지연시간에 대한 명확한 이해가 필요합니다.

| 시나리오 | 데이터 전송 비용 | 예상 지연시간 | 비고 |
|----------|----------------|-------------|------|
| **같은 리전 (us-east-1 → us-east-1)**| 무료 (S3) / 무료 (ADLS 같은 리전) | 낮음 (수십 ms) | 최적 |
| **리전 간 (us-east-1 → eu-west-1)**| $0.01~0.02/GB (AWS), $0.02/GB (Azure) | 중간 (100-300 ms) | 1TB 공유 시 $10~20 |
| **AWS → Azure**| $0.09/GB (인터넷 egress) | 높음 (200-500 ms) | 1TB 공유 시 약 $90. **PrivateLink 불가**|
| **AWS → GCP**| $0.09/GB (인터넷 egress) | 높음 | 크로스 클라우드 최적화 없음 |

> ⚠️ **비용 주의**: 크로스 클라우드 공유 시 **데이터 전송 비용은 제공자에게 청구** 됩니다. 10TB 테이블을 AWS에서 Azure로 공유하면 매번 전체 읽기 시 약 $900의 egress 비용이 발생합니다. 파티션 필터링으로 전송량을 최소화하는 것이 필수입니다.

**데이터 레지던시 고려사항:**
- EU GDPR, 한국 데이터 3법 등 규제에 따라 데이터가 특정 리전을 벗어나면 안 되는 경우가 있습니다
- Delta Sharing은 수신자가 **원본 스토리지에서 직접 읽는 구조** 이므로, 데이터가 물리적으로 이동하지 않습니다. 단, 수신자 측 캐싱/복사가 발생할 수 있으므로 수신자와의 계약에서 이를 명시해야 합니다

---

### Credential 라이프사이클 관리

Open Sharing 방식의 토큰 관리는 운영의 핵심입니다.

| 단계 | 관리 항목 | 권장 사항 |
|------|----------|----------|
| **토큰 생성**| `CREATE RECIPIENT` 시 활성화 링크 생성 | 수신자별 고유 Recipient 생성 (공유 금지) |
| **토큰 만료**| 기본 만료 없음, 수동 설정 필요 | `RECIPIENT_TOKEN_LIFETIME_IN_SECONDS` 속성으로 만료 기간 설정 (90일 권장) |
| **토큰 로테이션**| 기존 토큰 무효화 + 새 토큰 발급 | `ALTER RECIPIENT ... ROTATE TOKEN`으로 주기적 로테이션 |
| **수신자 비활성화**| 접근을 일시 중지 | `DROP RECIPIENT`으로 즉시 비활성화, 필요 시 재생성 |
| **IP 제한** | 수신자의 접근 IP 대역 지정 | `CREATE RECIPIENT ... PROPERTIES ('ip_access_list' = '10.0.0.0/8')` |

```sql
-- 토큰 로테이션 (기존 토큰 즉시 무효화)
ALTER RECIPIENT external_partner ROTATE TOKEN;

-- 토큰 만료 기간 설정 (90일)
ALTER RECIPIENT external_partner
SET PROPERTIES ('token_lifetime_in_seconds' = '7776000');

-- IP 접근 제한 설정
ALTER RECIPIENT external_partner
SET PROPERTIES ('ip_access_list' = '203.0.113.0/24,198.51.100.0/24');
```

---

### 공유 모니터링 및 감사

Delta Sharing 접근은 Unity Catalog의 감사 로그와 시스템 테이블을 통해 추적할 수 있습니다.

```sql
-- 감사 로그에서 Delta Sharing 접근 이벤트 조회
SELECT
    event_time,
    user_identity.email AS accessor,
    action_name,
    request_params.share_name,
    request_params.recipient_name,
    response.status_code
FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND action_name IN (
    'deltaSharingQueryTable',
    'deltaSharingGetTableVersion',
    'deltaSharingGetShare',
    'deltaSharingListShares'
  )
  AND event_date >= current_date() - INTERVAL 30 DAYS
ORDER BY event_time DESC;

-- 수신자별 데이터 접근 빈도 분석
SELECT
    request_params.recipient_name,
    request_params.share_name,
    COUNT(*) AS query_count,
    MIN(event_time) AS first_access,
    MAX(event_time) AS last_access
FROM system.access.audit
WHERE action_name = 'deltaSharingQueryTable'
  AND event_date >= current_date() - INTERVAL 90 DAYS
GROUP BY 1, 2
ORDER BY query_count DESC;
```

> 💡 **운영 팁**: 90일 이상 접근이 없는 수신자는 비활성화를 검토하세요. 불필요한 Recipient를 방치하면 보안 위험이 됩니다.

---

### Open Sharing vs Databricks-to-Databricks 비교

두 공유 방식의 차이는 생각보다 큽니다. 요구사항에 따라 올바른 방식을 선택해야 합니다.

| 비교 항목 | Open Sharing | Databricks-to-Databricks |
|----------|-------------|------------------------|
| **수신자 플랫폼**| 제한 없음 (pandas, Spark, Power BI 등) | Databricks 워크스페이스 필수 |
| **인증 방식**| Bearer Token (활성화 파일) | Unity Catalog Sharing Identifier |
| **Change Data Feed**| ❌ 지원 안 함 | ✅ 변경분만 읽기 가능 |
| **Notebook 공유**| ❌ | ✅ 노트북도 Share에 포함 가능 |
| **Volume 공유**| ❌ | ✅ 파일(비정형 데이터) 공유 가능 |
| **AI 모델 공유**| ❌ | ✅ ML 모델도 공유 가능 |
| **실시간 뷰**| ❌ (스냅샷만) | ✅ 실시간 원본 접근 |
| **보안 수준**| 중간 (토큰 유출 시 위험) | 높음 (UC 기반 ID 인증) |
| **거버넌스**| 제한적 (감사 로그만) | 완전 (리니지, 태깅, 접근 제어) |
| **네트워크 보안**| 인터넷 경유 | PrivateLink 지원 가능 |

> 💡 **선택 가이드**: 수신자가 Databricks를 사용한다면 **반드시 Databricks-to-Databricks** 방식을 사용하세요. Open Sharing은 수신자가 Databricks를 사용하지 않는 경우에만 권장합니다. 보안, 기능, 거버넌스 모든 면에서 D2D가 우월합니다.

---

### 실전 패턴

#### 패턴 1: 데이터 제품 판매 (Data as a Product)

데이터 마켓플레이스 형태로 외부 고객에게 데이터를 판매하는 패턴입니다.

```sql
-- 고객별 Share 생성 (고객 격리)
CREATE SHARE gold_weather_api_customer_a
COMMENT '고객 A 전용 - 날씨 API 데이터';

-- 파티션 기반 데이터 범위 제한 (계약 기간만 공유)
ALTER SHARE gold_weather_api_customer_a
ADD TABLE gold.weather_daily
PARTITION (date >= '2025-01-01' AND region IN ('KR', 'JP'));

-- 고객별 수신자 생성 + IP 제한
CREATE RECIPIENT customer_a_prod
PROPERTIES ('ip_access_list' = '203.0.113.0/24');

GRANT SELECT ON SHARE gold_weather_api_customer_a TO RECIPIENT customer_a_prod;
```

| 운영 포인트 | 설명 |
|-----------|------|
| **고객별 Share 분리**| 공유 범위를 고객별로 독립적으로 관리할 수 있습니다 |
| **파티션 기반 과금**| 계약 기간/지역별로 데이터 범위를 제한하여 과금 모델을 구현합니다 |
| **접근 모니터링**| 감사 로그로 고객별 사용량을 추적하여 과금에 활용합니다 |

#### 패턴 2: 파트너 간 양방향 데이터 교환

본사-자회사, 또는 공급망 파트너 간 데이터를 교환하는 패턴입니다.

```
[본사 Workspace]           [자회사 Workspace]
  Share: hq_to_sub    →      Recipient: hq
  (매출 데이터)               (읽기)

  Recipient: sub      ←      Share: sub_to_hq
  (읽기)                     (재고 데이터)
```

> ⚠️ **양방향 공유 주의사항**: 양쪽 모두 Provider이자 Recipient가 되므로, Share/Recipient 네이밍 컨벤션을 명확히 정의해야 합니다. `{source}_{destination}_{domain}` 형식을 권장합니다 (예: `hq_sub01_sales`).

#### 패턴 3: 규제 기관 보고

금융감독원, 국세청 등 규제 기관에 데이터를 안전하게 공유하는 패턴입니다.

| 요구사항 | 구현 방법 |
|----------|----------|
| **최소 권한**| 규제 보고에 필요한 컬럼만 포함한 View를 공유합니다 |
| **데이터 마스킹**| 민감 정보(주민번호, 계좌번호)를 마스킹한 View를 생성하여 공유합니다 |
| **접근 기간 제한**| 토큰 만료를 감사 기간에 맞춰 설정합니다 |
| **감사 증적**| 감사 로그를 별도로 보존하여 규제 준수를 입증합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Delta Sharing**| 조직 간 데이터를 안전하게 공유하는 오픈 프로토콜입니다 |
| **Share**| 공유할 테이블의 묶음입니다 |
| **Recipient**| 데이터를 받는 수신자입니다 |
| **데이터 복사 없음**| 수신자가 원본을 직접 읽습니다 |
| **오픈 프로토콜** | Databricks 없이도 pandas, Spark, Power BI로 접근 가능합니다 |

---

## 참고 링크

- [Databricks: Delta Sharing](https://docs.databricks.com/aws/en/data-sharing/)
- [Databricks: Share data](https://docs.databricks.com/aws/en/data-sharing/share-data.html)
- [Delta Sharing Official](https://delta.io/sharing/)
- [Azure Databricks: Delta Sharing](https://learn.microsoft.com/en-us/azure/databricks/data-sharing/)
