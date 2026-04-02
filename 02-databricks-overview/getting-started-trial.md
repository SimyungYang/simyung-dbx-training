# 무료 체험 시작하기

## Databricks 무료 체험 (Free Trial)

Databricks를 직접 경험해 보는 가장 좋은 방법은 **무료 체험(Free Trial)** 을 시작하는 것입니다. 14일간 Databricks의 주요 기능을 무료로 사용해 볼 수 있습니다.

> 💡 무료 체험은 **신용카드 정보 없이** 도 시작할 수 있습니다. 체험 기간이 끝나면 자동으로 과금되지 않으니 안심하셔도 됩니다.

---

## Community Edition vs Full Trial

Databricks는 두 가지 무료 옵션을 제공합니다.

| 비교 항목 | Community Edition | Full Trial (14일) |
|-----------|-------------------|-------------------|
| **기간** | 무기한 | 14일 |
| **비용** | 무료 | 무료 (14일 이후 유료) |
| **클러스터** | 소규모 단일 노드만 | 다양한 크기의 클러스터 |
| **Unity Catalog** | ❌ | ✅ |
| **SQL Warehouse** | ❌ | ✅ (Serverless 포함) |
| **협업 기능** | 제한적 | 전체 기능 |
| **적합한 대상** | 개인 학습, 실습 | 팀 단위 평가, POC |

학습 목적이라면 **Community Edition** 으로도 충분하며, 기업 평가를 위해서는 **Full Trial** 을 권장합니다.

> 💡 **현업에서의 권장**: PoC(기술 검증)를 진행한다면, 반드시 **Full Trial** 을 사용하세요. Community Edition에서는 Unity Catalog, SQL Warehouse, Serverless 등 Databricks의 핵심 차별화 기능을 체험할 수 없습니다. Community Edition만 써보고 "Databricks는 Jupyter랑 비슷하네"라고 판단하는 것은 스마트폰의 전화 기능만 써보고 평가하는 것과 같습니다.

---

## Community Edition 시작하기

### Step 1: 가입 페이지 접속

Databricks 공식 사이트에서 Community Edition 가입 페이지에 접속합니다.

### Step 2: 정보 입력

- 이름, 이메일, 회사명(개인은 "Personal" 입력 가능)
- 비밀번호 설정

### Step 3: 이메일 인증

- 입력한 이메일로 인증 링크가 전송됩니다
- 이메일의 링크를 클릭하여 인증을 완료합니다

### Step 4: Workspace 접속

- 인증 완료 후 자동으로 Workspace에 접속됩니다
- 이제 노트북을 만들고, 코드를 실행할 수 있습니다

---

## 첫 번째 실습: "Hello, Databricks!"

Workspace에 접속했다면, 간단한 실습으로 Databricks를 체험해 보겠습니다.

### 1. 클러스터 생성 및 시작

1. 좌측 메뉴에서 **Compute** 클릭
2. **Create Compute** 클릭
3. 클러스터 이름 입력 (예: `my-first-cluster`)
4. 나머지는 기본값 유지 → **Create Compute** 클릭
5. 클러스터가 시작될 때까지 약 3~5분 대기 (상태가 "Running"으로 변경되면 완료)

### 2. 노트북 생성

1. 좌측 메뉴에서 **+ New**→ **Notebook** 클릭
2. 이름: `hello-databricks`
3. 기본 언어: **Python**
4. 방금 만든 클러스터를 연결

### 3. 코드 실행

```python
# 셀 1: Spark 버전 확인
print(f"Spark 버전: {spark.version}")
print(f"Python 버전: {sys.version}")
print("Hello, Databricks! 🎉")
```

```python
# 셀 2: 샘플 데이터 생성 및 분석
from pyspark.sql.functions import col, sum, avg, count

# 쇼핑몰 주문 데이터 생성
orders = spark.createDataFrame([
    (1, "노트북", "전자제품", 1200000, "2025-03-01"),
    (2, "키보드", "전자제품", 89000, "2025-03-01"),
    (3, "운동화", "패션", 159000, "2025-03-02"),
    (4, "텀블러", "생활용품", 25000, "2025-03-02"),
    (5, "모니터", "전자제품", 450000, "2025-03-03"),
    (6, "가방", "패션", 198000, "2025-03-03"),
    (7, "마우스", "전자제품", 65000, "2025-03-03"),
    (8, "쿠션", "생활용품", 35000, "2025-03-04"),
], ["order_id", "product", "category", "price", "order_date"])

# 카테고리별 매출 집계
summary = orders.groupBy("category").agg(
    count("*").alias("주문건수"),
    sum("price").alias("총매출"),
    avg("price").alias("평균단가")
).orderBy(col("총매출").desc())

display(summary)
```

```sql
%sql
-- 셀 3: SQL로 같은 분석 수행
-- (위에서 만든 DataFrame을 임시 뷰로 등록)
```

```python
# 셀 4: 임시 뷰 생성
orders.createOrReplaceTempView("orders")
```

```sql
%sql
-- 셀 5: SQL로 일별 매출 추이 조회
SELECT
    order_date AS 주문일,
    COUNT(*) AS 주문건수,
    FORMAT_NUMBER(SUM(price), 0) AS 총매출
FROM orders
GROUP BY order_date
ORDER BY order_date
```

### 4. 시각화

SQL 셀의 결과 아래에서 **차트 아이콘(📊)** 을 클릭하고:
- Chart Type: **Bar Chart** 선택
- X축: `주문일`
- Y축: `총매출`

이렇게 하면 일별 매출 추이를 막대 차트로 바로 확인하실 수 있습니다.

### 5. Delta 테이블 생성 및 타임 트래블 체험

```python
# 셀 6: Delta 테이블 생성
orders.write.mode("overwrite").saveAsTable("main.default.trial_orders")
print("Delta 테이블이 생성되었습니다!")
```

```sql
%sql
-- 셀 7: 데이터를 수정해봅니다
UPDATE main.default.trial_orders SET price = 999000 WHERE product = '노트북';

-- 셀 8: 변경 이력 확인 (Delta Lake의 타임 트래블)
DESCRIBE HISTORY main.default.trial_orders;

-- 셀 9: 수정 전 데이터를 조회 (버전 0 = 최초 상태)
SELECT * FROM main.default.trial_orders VERSION AS OF 0 WHERE product = '노트북';
-- price가 원래의 1200000으로 나옵니다!
```

> 💡 이것이 Delta Lake의 **타임 트래블** 기능입니다. 실수로 데이터를 잘못 수정해도 과거 버전으로 되돌릴 수 있습니다. 프로덕션 환경에서는 이 기능이 매우 중요합니다.

### 6. 샘플 데이터셋 활용

Databricks Full Trial에는 바로 사용할 수 있는 샘플 데이터가 내장되어 있습니다.

```sql
%sql
-- NYC 택시 데이터 (약 20만 건)
SELECT * FROM samples.nyctaxi.trips LIMIT 10;

-- TPC-H 벤치마크 데이터
SELECT * FROM samples.tpch.orders LIMIT 10;
```

이 샘플 데이터로 SQL 분석, 대시보드, 파이프라인을 별도의 데이터 준비 없이 바로 체험할 수 있습니다.

### 7. Databricks Assistant (AI 코딩 도우미)

Notebook에서 코드를 작성할 때, **Databricks Assistant** 를 활용해 보세요. 자연어로 "이 DataFrame에서 도시별 평균 연봉을 구해줘"라고 입력하면 코드를 자동으로 생성해 줍니다. 또한 에러 메시지를 붙여넣으면 원인과 해결 방법을 제안합니다.

> 💡 **현업에서의 활용**: Databricks Assistant는 특히 SQL 쿼리 작성, 에러 디버깅, 코드 설명 요청에서 유용합니다. "이 코드가 뭘 하는 건지 설명해줘"라고 물어보면 한국어로도 답변을 받을 수 있습니다.

---

## 체험판에서 반드시 시도해야 할 5가지

14일은 생각보다 짧습니다. Full Trial을 사용하신다면, 아래 5가지를 반드시 체험하시기 바랍니다. 이 5가지를 모두 해보면 Databricks가 우리 조직에 맞는 플랫폼인지 판단할 수 있습니다.

### 1. Unity Catalog 활성화 및 탐색

| 체험 내용 | 왜 중요한가? |
|-----------|-------------|
| Catalog > 스키마 > 테이블 생성 | 3-level namespace(카탈로그.스키마.테이블)를 직접 체험합니다 |
| 테이블에 권한(GRANT) 부여 | 데이터 거버넌스의 핵심 기능입니다 |
| Lineage 탭 확인 | "이 테이블이 어디서 왔는지" 자동 추적을 확인합니다 |

```sql
-- 카탈로그, 스키마, 테이블을 직접 만들어 보세요
CREATE CATALOG IF NOT EXISTS trial_catalog;
CREATE SCHEMA IF NOT EXISTS trial_catalog.my_project;
CREATE TABLE trial_catalog.my_project.sample_data AS
SELECT * FROM samples.nyctaxi.trips LIMIT 10000;

-- 다른 사용자에게 권한 부여
GRANT SELECT ON TABLE trial_catalog.my_project.sample_data TO `analyst@company.com`;
```

### 2. SQL Warehouse로 SQL 분석

Serverless SQL Warehouse를 생성하고, SQL Editor에서 쿼리를 실행해 보세요. Snowflake나 Redshift와 비교할 수 있는 좋은 기회입니다.

```sql
-- 샘플 데이터셋으로 SQL 분석 (Full Trial에 기본 포함)
SELECT
    pickup_zip,
    COUNT(*) AS trip_count,
    ROUND(AVG(fare_amount), 2) AS avg_fare
FROM samples.nyctaxi.trips
GROUP BY pickup_zip
ORDER BY trip_count DESC
LIMIT 20;
```

### 3. Auto Loader로 파일 수집

클라우드 스토리지(S3, ADLS)에 파일을 업로드하고, Auto Loader가 자동으로 감지하여 Delta 테이블에 적재하는 것을 확인해 보세요.

### 4. SDP(Spark Declarative Pipelines)로 파이프라인 구축

SDP를 사용하면 SQL 또는 Python으로 데이터 변환 파이프라인을 선언적으로 정의할 수 있습니다. Bronze → Silver → Gold 파이프라인을 직접 만들어 보세요.

### 5. Genie로 자연어 데이터 질의

AI/BI Genie에 테이블을 등록하고, "지난 달 매출이 가장 높은 지역은?"처럼 자연어로 질문해 보세요. 비개발자도 데이터를 탐색할 수 있는 강력한 기능입니다.

---

## 체험판 제한사항 상세

### Community Edition 제한

| 제한 항목 | 상세 |
|-----------|------|
| **클러스터** | 단일 노드(15GB RAM)만 사용 가능합니다. 멀티 노드 클러스터를 만들 수 없습니다 |
| **Unity Catalog** | 사용할 수 없습니다. HMS(Hive Metastore)만 사용 가능합니다 |
| **SQL Warehouse** | 사용할 수 없습니다 |
| **Serverless** | 사용할 수 없습니다 |
| **AI/BI Dashboard** | 사용할 수 없습니다 |
| **협업** | 실시간 공동 편집이 불가능합니다 |
| **SDP** | 사용할 수 없습니다 |

### Full Trial (14일) 제한

| 제한 항목 | 상세 |
|-----------|------|
| **기간** | 14일 후 자동 만료됩니다. 연장은 영업팀에 문의해야 합니다 |
| **크레딧** | 약 $400~$500 상당의 DBU 크레딧이 제공됩니다 (변경될 수 있습니다) |
| **리전** | 가입 시 선택한 리전에서만 사용 가능합니다 |
| **기능** | 대부분의 기능이 제공되지만, 일부 엔터프라이즈 기능(Private Link, BYOK 등)은 제한될 수 있습니다 |

> 💡 **크레딧 절약 팁**: 체험판 크레딧은 한정되어 있으므로, 사용하지 않는 클러스터는 반드시 종료하세요. Auto-termination을 10~15분으로 설정하면 불필요한 비용을 줄일 수 있습니다. SQL Warehouse의 Serverless는 자동으로 스케일 다운되므로 비교적 효율적입니다.

---

## PoC에서 프로덕션으로 — 전환 시 고려사항

체험판이나 PoC(Proof of Concept)가 성공적이었다면, 프로덕션으로 전환할 때 다음 사항들을 미리 계획해야 합니다.

### 인프라 및 네트워크

| 항목 | 체험판 | 프로덕션 | 비고 |
|------|--------|---------|------|
| **네트워크** | 퍼블릭 접근 | VPC Peering / Private Link | 보안 필수. 대부분의 기업에서 요구합니다 |
| **인증** | 이메일/비밀번호 | SSO (SAML/OIDC) | 기업 IdP(Okta, Azure AD 등)와 연동합니다 |
| **데이터 저장** | Databricks 관리형 | 고객 관리형 S3/ADLS | 데이터 소유권과 보안을 위해 자체 스토리지를 사용합니다 |

### 거버넌스

| 항목 | 조치 |
|------|------|
| **Unity Catalog 설계** | 카탈로그 명명 규칙, 스키마 구조, 접근 권한 정책을 사전에 설계합니다 |
| **Cluster Policy** | 인스턴스 타입, 최대 노드 수, auto-termination을 정책으로 제한합니다 |
| **비용 모니터링** | System Table(`system.billing.usage`)을 기반으로 비용 대시보드를 구축합니다 |
| **감사 로그** | `system.access.audit`로 누가 어떤 데이터에 접근했는지 추적합니다 |

### 개발 프로세스

| 항목 | 권장 사항 |
|------|-----------|
| **환경 분리** | dev / staging / prod Workspace를 분리합니다 (또는 카탈로그로 분리) |
| **CI/CD** | Databricks Asset Bundles로 코드를 배포합니다 |
| **Git 연동** | 모든 프로덕션 코드를 Repos를 통해 관리합니다 |
| **테스트** | 파이프라인에 데이터 품질 테스트(expectations)를 포함합니다 |

> 💡 **현업에서는 이렇게 합니다**: PoC에서 프로덕션으로 전환할 때 가장 많이 실수하는 것은 "PoC 코드를 그대로 프로덕션에 올리는 것"입니다. PoC 노트북은 탐색적으로 작성되었기 때문에, 프로덕션에서는 에러 처리, 재시도 로직, 데이터 품질 검증, 모니터링을 반드시 추가해야 합니다. 보통 PoC 코드를 프로덕션 품질로 리팩토링하는 데 PoC 기간의 1.5~2배가 소요됩니다.

---

## 체험 중 해볼 만한 것들

무료 체험을 최대한 활용하기 위해, 다음 순서로 기능들을 체험해 보시는 것을 권장합니다.

| 순서 | 활동 | 관련 섹션 |
|------|------|-----------|
| 1 | Notebook에서 Python/SQL 실행해 보기 | 현재 문서 |
| 2 | 샘플 데이터셋으로 SQL 분석해 보기 | [06. 데이터 웨어하우징](../06-data-warehousing/) |
| 3 | Delta 테이블 생성 및 조회해 보기 | [03. 레이크하우스](../03-lakehouse-architecture/) |
| 4 | 간단한 대시보드 만들어 보기 (Full Trial) | [08. AI/BI](../08-ai-bi/) |
| 5 | Unity Catalog 탐색해 보기 (Full Trial) | [07. Unity Catalog](../07-unity-catalog/) |

> 💡 **Community Edition 제한 사항**: Community Edition에서는 Unity Catalog, SQL Warehouse, AI/BI Dashboard 등 일부 기능을 사용할 수 없습니다. 이러한 기능을 체험하시려면 Full Trial 또는 기업 계정이 필요합니다.

---

## 체험판에서 흔히 겪는 문제와 해결

| 문제 | 원인 | 해결 방법 |
|------|------|-----------|
| 클러스터가 시작되지 않음 | 리전의 인스턴스 용량 부족 | 다른 인스턴스 타입을 선택하거나, Serverless를 사용합니다 |
| "Permission denied" 에러 | Unity Catalog 권한 미설정 | Catalog에서 해당 스키마에 대한 USE, SELECT 권한을 확인합니다 |
| `samples.nyctaxi.trips` 테이블이 안 보임 | Community Edition 사용 중 | Community Edition에서는 samples 카탈로그가 없습니다. Full Trial이 필요합니다 |
| 노트북 실행이 느림 | 클러스터 웜업 중 | 첫 실행은 클러스터 시작 시간이 포함됩니다. 두 번째 실행부터 빨라집니다 |
| 크레딧이 빨리 소진됨 | All-Purpose 클러스터를 방치 | Auto-termination을 10분으로 설정하고, 사용 후 수동 종료합니다 |

> 💡 **체험판 연장**: 14일이 부족하다면, Databricks 영업팀에 연락하여 연장을 요청할 수 있습니다. 구체적인 평가 계획을 공유하면 연장 승인이 수월합니다.

---

## 정리

| 핵심 내용 | 설명 |
|-----------|------|
| **Community Edition** | 무기한 무료, 개인 학습에 적합합니다. 일부 기능이 제한됩니다 |
| **Full Trial** | 14일 무료, 모든 기능을 사용할 수 있습니다. 팀 평가에 적합합니다 |
| **필수 체험** | Unity Catalog, SQL Warehouse, Auto Loader, SDP, Genie를 반드시 시도해 보세요 |
| **PoC → 프로덕션** | 네트워크 보안, SSO, 환경 분리, CI/CD를 계획해야 합니다 |
| **첫 단계** | 클러스터 생성 → 노트북 생성 → 코드 실행 순서로 시작하시면 됩니다 |

이것으로 **Databricks 개요** 섹션을 마치겠습니다. 다음 섹션에서는 Databricks의 핵심 아키텍처인 [레이크하우스](../03-lakehouse-architecture/README.md)에 대해 자세히 알아보겠습니다.

---

## 참고 링크

- [Databricks: Try Databricks (Free Trial)](https://docs.databricks.com/aws/en/getting-started/free-trial.html)
- [Databricks Community Edition](https://community.cloud.databricks.com/)
- [Azure Databricks: Quickstart](https://learn.microsoft.com/en-us/azure/databricks/getting-started/)
