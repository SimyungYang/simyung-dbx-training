# 무료 체험 시작하기

## Databricks 무료 체험 (Free Trial)

Databricks를 직접 경험해 보는 가장 좋은 방법은 **무료 체험(Free Trial)**을 시작하는 것입니다. 14일간 Databricks의 주요 기능을 무료로 사용해 볼 수 있습니다.

> 💡 무료 체험은 **신용카드 정보 없이**도 시작할 수 있습니다. 체험 기간이 끝나면 자동으로 과금되지 않으니 안심하셔도 됩니다.

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

학습 목적이라면 **Community Edition**으로도 충분하며, 기업 평가를 위해서는 **Full Trial**을 권장합니다.

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

1. 좌측 메뉴에서 **+ New** → **Notebook** 클릭
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

SQL 셀의 결과 아래에서 **차트 아이콘(📊)**을 클릭하고:
- Chart Type: **Bar Chart** 선택
- X축: `주문일`
- Y축: `총매출`

이렇게 하면 일별 매출 추이를 막대 차트로 바로 확인하실 수 있습니다.

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

## 정리

| 핵심 내용 | 설명 |
|-----------|------|
| **Community Edition** | 무기한 무료, 개인 학습에 적합합니다. 일부 기능이 제한됩니다 |
| **Full Trial** | 14일 무료, 모든 기능을 사용할 수 있습니다. 팀 평가에 적합합니다 |
| **첫 단계** | 클러스터 생성 → 노트북 생성 → 코드 실행 순서로 시작하시면 됩니다 |

이것으로 **Databricks 개요** 섹션을 마치겠습니다. 다음 섹션에서는 Databricks의 핵심 아키텍처인 [레이크하우스](../03-lakehouse-architecture/README.md)에 대해 자세히 알아보겠습니다.

---

## 참고 링크

- [Databricks: Try Databricks (Free Trial)](https://docs.databricks.com/aws/en/getting-started/free-trial.html)
- [Databricks Community Edition](https://community.cloud.databricks.com/)
- [Azure Databricks: Quickstart](https://learn.microsoft.com/en-us/azure/databricks/getting-started/)
