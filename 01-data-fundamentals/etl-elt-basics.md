# ETL과 ELT — 데이터 수집·변환·적재의 기본 패턴

## 왜 ETL/ELT를 알아야 하나요?

데이터 엔지니어링의 핵심은 "데이터를 수집하고, 변환하고, 적재하는 것"입니다. 이 세 단계를 **어떤 순서로, 어디서 수행하느냐** 에 따라 **ETL** 과 **ELT** 라는 두 가지 패턴으로 나뉩니다.

이 두 패턴의 차이를 이해하는 것은 단순한 이론이 아닙니다. 현업에서 파이프라인을 설계할 때 "ETL로 갈까, ELT로 갈까?"는 **아키텍처 전체를 결정하는 선택** 입니다. 잘못 선택하면 6개월 후 전체를 갈아엎어야 할 수도 있습니다.

---

## ETL (Extract → Transform → Load) — 20년간의 표준

### 개념

> 💡 **ETL(Extract-Transform-Load)** 은 데이터를 소스에서 **추출(Extract)** 한 후, 별도의 처리 서버에서 **변환(Transform)** 하고, 변환이 완료된 데이터를 최종 저장소에 **적재(Load)** 하는 패턴입니다.

### 왜 ETL이 20년간 표준이었나

2000년대를 생각해 보세요. 그 시절의 데이터 웨어하우스는 **Teradata, Oracle, Netezza** 같은 전용 어플라이언스였습니다. TB당 저장 비용이 **수천 달러** 에 달했습니다. 1TB를 저장하는 데 연간 수천만 원이 드는 상황에서, "일단 다 넣고 나중에 정리하자"라는 발상은 사치였습니다.

그래서 **변환을 먼저 하고, 깨끗한 데이터만 웨어하우스에 넣는**ETL 패턴이 표준이 되었습니다. 별도의 ETL 서버에서 정제, 필터링, 변환을 수행하고, 검증이 끝난 데이터만 비싼 웨어하우스에 적재했습니다.

### ETL 시대의 대표 도구들

이 시대를 함께한 도구들을 알아두면, 기존 시스템을 이해하는 데 도움이 됩니다:

| 도구 | 설명 | 현재 상태 |
|------|------|----------|
| **Informatica PowerCenter** | 엔터프라이즈 ETL의 대명사. GUI로 매핑을 그립니다 | 여전히 많은 대기업에서 사용 중 (유지보수 모드) |
| **SSIS (SQL Server Integration Services)** | Microsoft 생태계의 ETL 도구 | SQL Server와 함께 아직 사용됨 |
| **IBM DataStage** | IBM의 대용량 ETL 도구 | 금융권에서 레거시로 남아있음 |
| **Talend** | 오픈소스 기반 ETL | Cloud 버전으로 진화 중 |
| **Apache NiFi** | 실시간 데이터 흐름 관리 | 특수 목적으로 여전히 활용 |

> 💡 현업에서 기존 ETL 시스템을 마이그레이션하는 프로젝트를 많이 만나게 됩니다. 이 도구들의 이름을 알아두면 고객과의 대화가 수월해집니다.

### ETL의 치명적 단점 — "원본이 사라진다"

ETL의 가장 큰 문제는 **변환을 먼저 하면 원본 데이터가 보존되지 않는다** 는 것입니다. 이것이 왜 치명적인지 실제 사례로 설명하겠습니다.

**사례 1: 변환 로직이 잘못되었을 때**

어떤 팀이 ETL 파이프라인에서 "주문 금액이 0 이하인 행은 삭제"하는 변환을 넣었습니다. 6개월 후 알고 보니, 환불 처리된 주문(금액이 음수)이 모두 삭제되어 환불 분석이 불가능했습니다. 원본이 없으므로 **6개월 치 환불 데이터를 영구적으로 잃었습니다.**

**사례 2: 새로운 분석 요구사항이 생겼을 때**

마케팅팀이 "사용자의 클릭 로그에서 체류 시간을 분석하고 싶다"고 요청했습니다. 하지만 ETL 과정에서 "클릭 로그는 일별 집계만 저장"하도록 설계했기 때문에, 세션 단위의 원본 로그는 이미 사라졌습니다. 원본 시스템에 요청해서 데이터를 다시 받아야 했고, 과거 데이터는 이미 삭제된 상태였습니다.

> ⚠️ **이것이 ETL → ELT 전환의 가장 큰 동기입니다.**"지금 당장 필요 없는 데이터"가 6개월 후에 반드시 필요해집니다. 원본을 보존하지 않으면 되돌릴 수 없습니다.

### ETL이 여전히 적합한 경우

ETL이 완전히 사라진 것은 아닙니다. 다음 경우에는 여전히 ETL 패턴이 맞습니다:

- **개인정보 마스킹이 적재 전에 필수인 경우**: GDPR/개인정보보호법 때문에 원본에 주민번호, 카드번호가 있으면 저장 자체가 불가능합니다. 변환(마스킹) 후 적재해야 합니다
- **소스 시스템의 데이터를 네트워크 밖으로 보내기 전에 집계해야 하는 경우**: 대역폭이 제한된 환경
- **임베디드/IoT 엣지 처리**: 센서 데이터를 모두 전송할 수 없어서 엣지에서 변환 후 전송

---

## ELT (Extract → Load → Transform) — 현대의 표준

### 개념

> 💡 **ELT(Extract-Load-Transform)** 은 데이터를 소스에서 **추출(Extract)** 한 후, 원본 그대로 먼저 저장소에 **적재(Load)** 하고, 저장소 내부의 강력한 컴퓨팅 파워를 활용하여 **변환(Transform)** 하는 패턴입니다.

### ETL → ELT 전환이 일어난 진짜 이유

ELT가 뜬 이유를 한 문장으로 요약하면: "** 스토리지가 거의 공짜가 되었고, 컴퓨트는 필요할 때만 쓸 수 있게 되었다**" 입니다.

| 변화 요인 | 2005년 | 2025년 | 영향 |
|----------|--------|--------|------|
| **스토리지 비용** | TB당 연 수천만 원 (Teradata) | TB당 월 23달러 (S3) | 원본을 그대로 저장해도 비용이 미미 |
| **컴퓨트 탄력성** | 고정 서버 (항상 비용 발생) | 서버리스 (쓸 때만 비용) | 변환이 필요할 때만 클러스터를 켜면 됨 |
| **컴퓨트 성능** | 단일 ETL 서버 (수 시간) | 수백 노드 Spark 클러스터 (수 분) | 적재 후 변환이 더 빠를 수 있음 |
| **데이터 다양성** | 정형 데이터 위주 | JSON, 로그, 이미지, 스트리밍 | 원본 형태가 다양해서 사전 변환이 어려움 |

### "일단 넣고, 나중에 정리" — 이것이 왜 합리적인가

ELT의 핵심 철학은 단순합니다:

1. **원본을 보존합니다**— 변환 로직이 잘못되어도 원본에서 다시 시작할 수 있습니다
2. **변환 로직을 나중에 바꿀 수 있습니다**— 비즈니스 요구사항은 반드시 변합니다
3. **여러 팀이 같은 원본에서 서로 다른 변환을 할 수 있습니다**— 마케팅팀과 재무팀이 같은 데이터를 다르게 가공

> 💡 **"원본을 보존한다"는 것은 보험입니다.** 보험을 안 들면 당장은 저축이지만, 사고가 나면 파산합니다. 데이터도 마찬가지입니다. 원본 보존 비용은 S3 기준 TB당 월 23달러 — 이 보험료는 누가 봐도 저렴합니다.

### ELT와 Medallion 아키텍처 — 자연스러운 연결

ELT 패턴을 구조화하면 자연스럽게 **Medallion 아키텍처(Bronze → Silver → Gold)** 가 됩니다. 이것은 우연이 아니라, ELT의 논리적 귀결입니다.

| ELT 단계 | Medallion 계층 | 역할 | 데이터 상태 |
|----------|--------------|------|-----------|
| **Extract + Load** | Bronze | 원본 그대로 적재 | 중복 있음, 스키마 불안정, 데이터 품질 미검증 |
| **Transform (1차)** | Silver | 정제, 중복 제거, 타입 변환 | 깨끗한 엔티티 테이블, 비즈니스 키 표준화 |
| **Transform (2차)** | Gold | 비즈니스 로직 적용, 집계 | 대시보드/ML에 바로 사용 가능한 상태 |

이 구조가 좋은 이유는 **각 계층의 목적이 명확** 하기 때문입니다:
- Bronze가 잘못되면? 소스에서 다시 수집합니다
- Silver가 잘못되면? Bronze에서 다시 변환합니다
- Gold가 잘못되면? Silver에서 다시 집계합니다

**어떤 단계에서 문제가 생겨도 처음부터 다시 할 필요가 없습니다.**ETL에서는 전체 파이프라인을 다시 돌려야 했습니다.

---

## ETL vs ELT 비교

| 비교 항목 | ETL | ELT |
|-----------|-----|-----|
| **변환 시점** | 적재 **전**(별도 서버에서) | 적재 **후**(저장소 내에서) |
| **원본 데이터 보존** | ❌ 보통 보존하지 않음 | ✅ 원본을 그대로 보존 |
| **변환 처리 위치** | 별도 ETL 서버 (병목 지점) | 저장소의 컴퓨팅 엔진 (Spark, SQL) |
| **변환 로직 변경 시** | 원본부터 다시 수집해야 함 | 원본이 있으므로 변환만 다시 돌림 |
| **확장성** | ETL 서버 성능에 제한됨 | 클라우드 컴퓨팅으로 거의 무한 확장 |
| **초기 적재 속도** | 느림 (변환 후 적재) | 빠름 (원본 그대로 적재) |
| **적합한 환경** | 온프레미스, 보안 제약이 큰 환경 | 클라우드, 대규모 데이터 |
| **비용 구조** | ETL 서버 상시 운영 비용 | 스토리지(저렴) + 필요 시 컴퓨트 비용 |
| **시대적 흐름** | 전통적 방식 (2000~2015년) | 현대적 방식 (2015년~현재) |

---

## 실전: 기존 ETL을 ELT로 마이그레이션하는 전략

많은 기업이 Informatica나 SSIS 기반 ETL 파이프라인을 Databricks ELT로 마이그레이션하고 있습니다. 실무에서 효과적인 전략을 공유합니다.

### 빅뱅 vs 점진적 마이그레이션

| 전략 | 설명 | 장점 | 리스크 |
|------|------|------|--------|
| **빅뱅** | 전체 파이프라인을 한 번에 전환 | 빠른 전환, 이중 운영 비용 최소화 | 실패 시 전체 장애. **절대 권장하지 않습니다** |
| **점진적 (Strangler Fig 패턴)** | 파이프라인을 하나씩 전환 | 리스크 최소화, 팀 학습 시간 확보 | 이중 운영 기간 동안 비용 증가 |

> 💡 **현업에서는 반드시 점진적 마이그레이션을 합니다.** 영향도가 낮은 파이프라인부터 시작하여, 팀이 Databricks에 익숙해진 후 핵심 파이프라인을 전환합니다. "이번 주에 100개 파이프라인을 한 번에 옮기겠습니다"라고 하면, 99% 확률로 재앙입니다.

### 마이그레이션 우선순위

```
1단계: 리포트/대시보드용 배치 파이프라인 (영향도: 낮음)
   → 실패해도 리포트가 하루 늦어지는 정도

2단계: 데이터 사이언스 팀용 피처 파이프라인 (영향도: 중간)
   → ML 모델 재학습에 영향

3단계: 비즈니스 크리티컬 파이프라인 (영향도: 높음)
   → 실시간 매출, 재고 관리, 고객 대면 서비스
   → 반드시 이중 운영(parallel run)을 2~4주 진행
```

---

## Databricks에서의 ELT

Databricks는 현대적인 **ELT 패턴** 에 최적화된 플랫폼입니다. 그 이유를 구체적으로 살펴보겠습니다.

### 왜 Databricks가 ELT에 적합한가

1. **Apache Spark**: 수 TB의 데이터를 수백 대의 서버에 분산 처리. ETL 서버 한 대에서 8시간 걸리던 작업이 Spark 클러스터에서 15분에 끝납니다
2. **Delta Lake**: 데이터 레이크 위에 트랜잭션, 스키마 관리를 추가. "데이터 늪(Data Swamp)"을 방지합니다
3. **Serverless 컴퓨팅**: 변환 작업이 필요할 때만 리소스를 할당하여 비용 최적화. 새벽 2시에 30분 돌리고 꺼집니다
4. **SDP (Spark Declarative Pipelines)**: 변환 로직을 선언적으로 정의하면, 실행 순서와 의존성을 자동 관리. "이 테이블은 저 테이블에서 만든다"만 정의하면 됩니다

> 🆕 **최신 트렌드**: Databricks **Lakeflow Connect** 를 통해 Extract(수집) 단계가 관리형 서비스로 제공됩니다. Salesforce, SAP, Oracle DB 등에서 데이터를 자동으로 수집하는 커넥터를 제공하여, 데이터 엔지니어가 변환 로직에만 집중할 수 있습니다. 예전에는 Extract 코드를 짜는 데 전체 작업 시간의 40%를 썼는데, 이제는 커넥터 설정만 하면 됩니다.

---

## 실무 예시: 온라인 쇼핑몰 데이터 파이프라인

온라인 쇼핑몰의 주문 데이터를 처리하는 ELT 파이프라인 예시입니다. 각 단계에서 왜 이렇게 하는지도 함께 설명합니다.

### 1단계: Extract & Load (Bronze)

```sql
-- Auto Loader로 S3에서 주문 JSON 파일을 원본 그대로 수집
-- 핵심: 데이터를 절대 변환하지 않습니다. 원본 그대로 넣습니다.
CREATE OR REFRESH STREAMING TABLE bronze_orders
AS SELECT
  *,
  _metadata.file_path AS source_file,      -- 어디서 온 파일인지 추적
  _metadata.file_modification_time AS ingested_at  -- 언제 수집했는지 기록
FROM STREAM read_files(
  's3://my-bucket/orders/',
  format => 'json'
);
```

> 💡 `source_file`과 `ingested_at`을 기록하는 이유: 나중에 "이 데이터가 어디서 왔는지", "특정 파일에 문제가 있었는지" 추적할 수 있어야 합니다. 이것을 **데이터 리니지(Data Lineage)** 라고 하며, 문제 발생 시 디버깅에 필수적입니다.

### 2단계: Transform (Silver)

```sql
-- 정제: 중복 제거, 타입 변환, 유효성 검증
-- 이 단계에서 "깨끗한 데이터"를 만듭니다
CREATE OR REFRESH STREAMING TABLE silver_orders (
  -- 데이터 품질 규칙: 위반 시 해당 행을 버립니다
  CONSTRAINT valid_order_id EXPECT (order_id IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT valid_amount EXPECT (amount > 0) ON VIOLATION DROP ROW
)
AS SELECT
  CAST(order_id AS BIGINT) AS order_id,
  CAST(customer_id AS BIGINT) AS customer_id,
  CAST(order_date AS TIMESTAMP) AS order_date,
  CAST(amount AS DECIMAL(10,2)) AS amount,
  UPPER(TRIM(status)) AS status  -- "pending", " PENDING", "Pending" → 모두 "PENDING"
FROM STREAM(bronze_orders);
```

> ⚠️ **amount > 0 제약 조건을 넣을 때 주의**: 앞서 말한 "환불 데이터 삭제" 실수를 기억하세요. 환불(음수 금액)이 비즈니스에서 발생하는지 반드시 확인한 후 규칙을 설정하세요. 확실하지 않으면 **DROP ROW 대신 태그만 달아두는 것** 이 안전합니다.

### 3단계: Transform (Gold)

```sql
-- 비즈니스 집계: 일별/상품별 매출 요약
-- 대시보드에서 바로 사용할 수 있는 형태
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_sales
AS SELECT
  DATE(order_date) AS sale_date,
  product_category,
  COUNT(*) AS order_count,
  SUM(amount) AS total_revenue,
  AVG(amount) AS avg_order_value
FROM silver_orders
GROUP BY DATE(order_date), product_category;
```

---

## 현업 사례: Informatica 라이선스 비용이 연 5억인 기업이 ELT로 전환한 이유

현업에서 가장 많이 접하는 마이그레이션 동기는 **비용** 입니다. 실제 사례를 바탕으로 설명합니다.

### 배경

국내 대형 금융사의 데이터 팀이 운영하던 환경은 다음과 같았습니다:

| 항목 | 기존 ETL 환경 |
|------|-------------|
| 인프라 | Informatica PowerCenter 서버 3대 (라이선스: 연 5억 원) |
| DW | Oracle DW (Exadata: 연 유지보수 8억 원) |
| ETL 매핑 | 2,300개 (10년간 누적) |
| 인력 | ETL 개발자 4명 (Informatica 전문가, 희소 인력) |
| 처리 시간 | 일일 12시간 (새벽 1시~오후 1시) |
| 소스 추가 | 평균 3주 (매핑 설계 → 개발 → 테스트 → 배포) |

### 전환 동기 — "비용"만이 아니었습니다

| 동기 | 구체적 상황 | 영향 |
|------|-----------|------|
| **라이선스 비용** | Informatica + Oracle 연 13억 원 | 매년 5~10% 인상 통보 |
| **인력 종속** | Informatica 전문가 4명 중 2명 이직 예정 | 대체 인력 채용 불가 (시장에 없음) |
| **처리 시간** | 12시간 ETL 윈도우가 포화 | 신규 데이터 소스 추가 불가 |
| **유연성 부재** | 새 리포트 요청 시 3주 소요 | 비즈니스 팀 불만 폭증 |
| **ML/AI 불가** | DW에서 Python 실행 불가 | 데이터를 복사해서 별도 환경으로 이동 필요 |

### ELT 전환 후 결과 (18개월 후)

| 항목 | 전환 후 ELT 환경 |
|------|---------------|
| 인프라 | Databricks Lakehouse (Lakeflow Jobs + SDP) |
| 스토리지 | Delta Lake on S3 |
| 파이프라인 | 1,500개 (불필요한 700개 정리) |
| 인력 | 데이터 엔지니어 4명 (SQL + Python, 채용 용이) |
| 처리 시간 | 일일 2시간 (서버리스 클러스터 자동 확장) |
| 소스 추가 | 평균 3일 |

| 지표 | Before | After | 개선 |
|------|--------|-------|------|
| 연간 플랫폼 비용 | 13억 원 | 5억 원 | **62% 절감** |
| 일일 ETL 시간 | 12시간 | 2시간 | **83% 단축** |
| 신규 소스 추가 | 3주 | 3일 | **7배 빠름** |
| 인력 채용 난이도 | 극난 (Informatica) | 보통 (SQL/Python) | **인력풀 100배** |

---

## ETL에서 ELT로 전환할 때 가장 어려운 점

기술적 전환보다 **조직적 전환** 이 10배 어렵습니다. 현업에서 겪는 실제 저항과 해결법을 공유합니다.

### 1. 조직 문화의 저항

```
DBA/ETL 팀의 전형적인 반응:
"10년간 잘 돌아가는 ETL을 왜 바꿉니까?"
"Spark는 배워본 적이 없는데요"
"원본을 그대로 넣으면 데이터 품질은 누가 보장합니까?"
```

> 💡 **현업에서는 이렇게 합니다**: 기존 ETL 팀을 "적"이 아니라 "주인공"으로 만들어야 합니다. 이 팀이 가장 잘 아는 것은 비즈니스 로직입니다. Informatica 매핑에 들어있는 변환 로직을 SQL로 재작성하는 역할을 맡기면, 자연스럽게 Databricks에 익숙해지면서 동시에 기존 지식이 보존됩니다.

### 2. 기존 코드 자산의 처리

2,300개의 Informatica 매핑을 어떻게 할 것인가? 현실적인 접근법:

| 전략 | 설명 | 적합한 상황 |
|------|------|-----------|
| **자동 변환 도구** | Informatica → Spark/SQL 자동 변환 | 단순 매핑 (전체의 40~60%) |
| **수동 재작성** | 복잡한 로직을 SQL/Python으로 재구현 | 비즈니스 크리티컬 매핑 |
| **폐기** | 더 이상 사용되지 않는 매핑 제거 | 사용자가 없는 매핑 (보통 30%) |

> ⚠️ **이것을 안 하면**: 2,300개를 전부 1:1로 변환하려다가 2년이 걸리고, 결국 이중 운영 비용만 늘어납니다. **반드시 사용 빈도를 분석** 하세요. 대부분의 기업에서 전체 ETL 매핑의 30%는 아무도 사용하지 않는 좀비 파이프라인입니다.

### 3. 데이터 정합성 신뢰 확보

가장 오래 걸리는 단계입니다. 현업에서는 최소 4주간 이중 운영(parallel run)을 합니다.

```sql
-- 정합성 비교 쿼리 패턴
-- 기존 DW와 신규 Lakehouse의 결과를 비교합니다
SELECT
    'DW' AS source,
    COUNT(*) AS row_count,
    SUM(amount) AS total_amount,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM oracle_dw.sales.daily_summary
WHERE date = '2025-03-30'

UNION ALL

SELECT
    'Lakehouse' AS source,
    COUNT(*) AS row_count,
    SUM(amount) AS total_amount,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM catalog.gold.daily_summary
WHERE date = '2025-03-30';
```

---

## Databricks에서 ELT 구현 실전 패턴

### 패턴 1: CDC 기반 실시간 ELT

운영 DB의 변경분만 캡처하여 적재하는 패턴입니다. 전체를 매번 덤프하는 것보다 **100배 효율적** 입니다.

```sql
-- Lakeflow Connect로 MySQL CDC 수집 (Bronze)
-- 변경분만 자동으로 캡처하여 Delta 테이블에 적재

-- Silver: CDC 이벤트를 최신 상태로 MERGE
CREATE OR REFRESH STREAMING TABLE silver_customers
AS SELECT *
FROM STREAM(bronze_customers_cdc)
-- SDP APPLY CHANGES INTO를 사용하면 CDC MERGE를 자동 처리
```

### 패턴 2: 멀티 소스 조인 ELT

여러 소스의 데이터를 Lakehouse에서 조인하는 패턴입니다.

```sql
-- Gold: 여러 Silver 테이블을 조인하여 비즈니스 뷰 생성
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_360
AS SELECT
    c.customer_id,
    c.name,
    c.segment,
    o.total_orders,
    o.total_revenue,
    o.last_order_date,
    s.nps_score,
    s.last_survey_date
FROM silver_customers c
LEFT JOIN (
    SELECT customer_id,
           COUNT(*) AS total_orders,
           SUM(amount) AS total_revenue,
           MAX(order_date) AS last_order_date
    FROM silver_orders
    GROUP BY customer_id
) o ON c.customer_id = o.customer_id
LEFT JOIN silver_surveys s ON c.customer_id = s.customer_id;
```

> 💡 **현업 팁**: ELT에서 가장 중요한 것은 **Bronze를 절대 건드리지 않는 것** 입니다. Bronze는 "금고에 넣어둔 원본"입니다. 변환 로직이 잘못되면 Silver를 Drop하고 Bronze에서 다시 만들면 됩니다. ETL에서는 이것이 불가능했습니다.

---

## 정리

| 핵심 개념 | 실무 핵심 |
|-----------|----------|
| **ETL** | 변환 후 적재. 스토리지가 비쌌던 시대의 표준. 원본이 사라지는 치명적 단점 |
| **ELT** | 적재 후 변환. 원본 보존 + 클라우드 컴퓨트 활용. 현대의 표준 |
| **Medallion** | ELT의 구조화된 구현. Bronze(원본) → Silver(정제) → Gold(집계) |
| **마이그레이션** | 반드시 점진적으로. 영향도 낮은 것부터 시작 |
| **원본 보존** | ELT의 가장 큰 장점. TB당 월 23달러짜리 보험 |

다음 문서에서는 데이터 처리의 두 가지 방식인 **배치 처리** 와 **스트리밍 처리** 의 차이점을 살펴보겠습니다.

---

## 참고 링크

- [Databricks: Medallion Architecture](https://docs.databricks.com/aws/en/lakehouse/medallion.html)
- [Databricks: Data Engineering](https://docs.databricks.com/aws/en/data-engineering)
- [Databricks Blog: ETL vs ELT](https://www.databricks.com/glossary/etl-vs-elt)
- [Databricks: Lakeflow Connect](https://docs.databricks.com/aws/en/connect/)
- [Databricks: Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)
