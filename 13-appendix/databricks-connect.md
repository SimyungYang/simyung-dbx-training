| 로컬 개발 환경 | | 원격 클러스터 |
|-------------|---|-----------|
| VS Code, PyCharm, IntelliJ | → Spark 명령 → | 원격 클러스터 (또는 Serverless) |
| | ← 결과 반환 ← | 데이터 처리 실행 |# Databricks Connect — 로컬 IDE에서 원격 Spark 실행

## 개념

> 💡 **Databricks Connect**는 로컬 개발 환경(VS Code, PyCharm 등)에서 작성한 코드를 **원격 Databricks 클러스터에서 실행**할 수 있게 해주는 클라이언트 라이브러리입니다. 코드는 로컬에서 작성하지만, 실제 Spark 연산은 Databricks의 클러스터에서 처리됩니다.

### 왜 Databricks Connect가 필요한가요?

Databricks 노트북은 편리하지만, 전문 개발자에게는 익숙한 로컬 IDE의 기능(자동완성, 디버깅, 버전 관리 등)이 더 생산적입니다.

| 개발 방식 | 장점 | 단점 |
|-----------|------|------|
| **Databricks 노트북**| 즉시 사용 가능, 시각화 내장 | IDE 기능 제한, 대규모 프로젝트 관리 어려움 |
| ** 로컬 Spark (standalone)**| 빠른 테스트 | 소규모만 가능, 프로덕션 환경과 차이 |
| **Databricks Connect**| IDE 자유도, 디버깅, 테스트 프레임워크 | 네트워크 지연, 일부 API 제한 |

Databricks Connect를 사용하면 다음과 같은 워크플로가 가능합니다.

| 로컬 개발 환경 | 통신 | Databricks 클러스터 |
|-------------|---|-----------|
| VS Code, PyCharm, IntelliJ | → Spark 명령 → | 원격 클러스터 (또는 Serverless) |
| 코드 작성, 디버깅, 단위 테스트 | ← 결과 반환 ← | 데이터 처리 실행, Unity Catalog 접근 |

---

## 설치 및 설정

### 요구 사항

| 항목 | 요건 |
|------|------|
| **Python**| 3.10 이상 |
| **Databricks Runtime**| 13.3 LTS 이상 |
| ** 네트워크**| 로컬 머신에서 Databricks 워크스페이스로의 HTTPS 접근 |
| ** 인증** | PAT, OAuth, 또는 Azure AD 토큰 |

### 설치

```bash
# pip로 설치 (가장 일반적)
pip install databricks-connect==16.2.*

# 버전은 사용할 Databricks Runtime과 일치시켜야 합니다
# 예: DBR 16.2를 사용한다면 databricks-connect==16.2.*

# 설치 확인
python -c "from databricks.connect import DatabricksSession; print('OK')"
```

> ⚠️ **중요**: `databricks-connect`의 버전은 반드시 클러스터의 **Databricks Runtime 버전과 일치**해야 합니다. 버전이 맞지 않으면 호환성 오류가 발생합니다.

### 인증 설정

Databricks Connect는 `~/.databrickscfg` 파일 또는 환경 변수를 통해 인증합니다.

#### 방법 1: Databricks CLI 프로필 사용 (권장)

```bash
# Databricks CLI로 OAuth 인증 설정
databricks configure --profile CONNECT

# 입력:
# Databricks Host: https://dbc-abc123.cloud.databricks.com
# (브라우저에서 OAuth 인증 완료)
```

```python
# Python 코드에서 프로필 사용
from databricks.connect import DatabricksSession

spark = DatabricksSession.builder.profile("CONNECT").getOrCreate()
```

#### 방법 2: Personal Access Token

```python
from databricks.connect import DatabricksSession

spark = DatabricksSession.builder \
    .host("https://dbc-abc123.cloud.databricks.com") \
    .token("dapi...") \
    .clusterId("0123-456789-abcdefgh") \
    .getOrCreate()
```

#### 방법 3: 환경 변수

```bash
export DATABRICKS_HOST="https://dbc-abc123.cloud.databricks.com"
export DATABRICKS_TOKEN="dapi..."
export DATABRICKS_CLUSTER_ID="0123-456789-abcdefgh"
```

```python
# 환경 변수가 설정되면 별도 설정 없이 세션 생성 가능
from databricks.connect import DatabricksSession

spark = DatabricksSession.builder.getOrCreate()
```

---

## 지원 IDE

### VS Code

VS Code는 **Databricks Extension**과 함께 사용하면 가장 편리합니다.

```
VS Code 설정 순서:
1. "Databricks" Extension 설치 (Marketplace)
2. Extension에서 Workspace 연결 설정
3. 클러스터 선택
4. Python 인터프리터를 databricks-connect가 설치된 환경으로 설정
5. 코드 작성 후 F5로 실행/디버깅
```

| VS Code 기능 | Databricks Connect 지원 |
|-------------|------------------------|
| **코드 자동완성**| ✅ PySpark API 자동완성 |
| ** 디버깅 (Breakpoint)**| ✅ 로컬 코드 디버깅 가능 |
| ** 변수 탐색기**| ✅ DataFrame 내용 확인 |
| ** 통합 터미널**| ✅ CLI 명령 실행 |
| **Jupyter 노트북** | ✅ .ipynb 파일에서도 사용 가능 |

### PyCharm / IntelliJ

```
PyCharm 설정 순서:
1. Project Interpreter에서 databricks-connect 설치된 venv 선택
2. Run/Debug Configuration 생성
3. 환경 변수 또는 코드 내에서 인증 설정
4. 코드 실행 시 자동으로 원격 클러스터에서 처리
```

---

## 로컬 코드에서 원격 실행

### 기본 사용법

```python
from databricks.connect import DatabricksSession

# 세션 생성 (원격 클러스터에 연결)
spark = DatabricksSession.builder.profile("CONNECT").getOrCreate()

# Unity Catalog 테이블 읽기
df = spark.read.table("catalog.schema.my_table")

# 로컬에서 작성한 변환 로직이 원격에서 실행됩니다
result = (
    df
    .filter(df.status == "active")
    .groupBy("region")
    .agg({"amount": "sum", "customer_id": "count"})
    .orderBy("sum(amount)", ascending=False)
)

# 결과를 로컬로 가져오기
result.show()
```

### DataFrame 작업

```python
# 로컬 데이터로 DataFrame 생성 후 원격에서 처리
from pyspark.sql import Row

local_data = [
    Row(name="김철수", department="엔지니어링", salary=80000),
    Row(name="이영희", department="마케팅", salary=75000),
    Row(name="박민수", department="엔지니어링", salary=85000),
]

df = spark.createDataFrame(local_data)

# 원격 클러스터에서 집계 실행
dept_stats = (
    df.groupBy("department")
    .agg(
        {"salary": "avg", "*": "count"}
    )
)
dept_stats.show()
```

### SQL 실행

```python
# Unity Catalog 테이블에 SQL 쿼리 실행
result = spark.sql("""
    SELECT
        region,
        COUNT(*) AS order_count,
        SUM(amount) AS total_revenue
    FROM catalog.gold.orders
    WHERE order_date >= '2025-01-01'
    GROUP BY region
    ORDER BY total_revenue DESC
""")

# pandas DataFrame으로 변환 (로컬로 가져오기)
pandas_df = result.toPandas()
print(pandas_df)
```

### 단위 테스트 연동

```python
# pytest와 함께 사용
import pytest
from databricks.connect import DatabricksSession

@pytest.fixture(scope="session")
def spark():
    """테스트 세션용 Spark 세션"""
    return DatabricksSession.builder.profile("CONNECT").getOrCreate()

def test_data_quality(spark):
    """데이터 품질 검증 테스트"""
    df = spark.read.table("catalog.gold.customers")

    # Null 체크
    null_count = df.filter(df.customer_id.isNull()).count()
    assert null_count == 0, f"customer_id에 {null_count}개의 Null 값이 있습니다"

    # 유니크 체크
    total = df.count()
    distinct = df.select("customer_id").distinct().count()
    assert total == distinct, "customer_id에 중복이 있습니다"

def test_transformation_logic(spark):
    """변환 로직 단위 테스트"""
    from my_project.transforms import calculate_revenue

    input_df = spark.createDataFrame([
        {"product_id": "A", "quantity": 10, "price": 100.0},
        {"product_id": "B", "quantity": 5, "price": 200.0},
    ])

    result = calculate_revenue(input_df)
    rows = result.collect()

    assert rows[0]["revenue"] == 1000.0
    assert rows[1]["revenue"] == 1000.0
```

---

## Serverless 컴퓨트와의 연동

Databricks Connect는 **Serverless Compute**에도 연결할 수 있습니다. 클러스터를 미리 시작할 필요 없이, 코드 실행 시 자동으로 서버리스 리소스가 할당됩니다.

```python
from databricks.connect import DatabricksSession

# Serverless에 연결 (cluster_id 대신 serverless 사용)
spark = DatabricksSession.builder \
    .host("https://dbc-abc123.cloud.databricks.com") \
    .token("dapi...") \
    .serverless(True) \
    .getOrCreate()

# 이후 사용법은 동일합니다
df = spark.read.table("catalog.schema.my_table")
df.show()
```

| 항목 | Classic Compute | Serverless Compute |
|------|----------------|-------------------|
| **시작 시간**| 수 분 (클러스터 시작 대기) | 수 초 |
| ** 비용 모델**| 클러스터 가동 시간 기준 | 실행 시간 기준 |
| ** 설정**| cluster_id 필요 | serverless=True |
| ** 관리**| 클러스터 사양 직접 설정 | Databricks가 관리 |
| ** 적합한 경우**| 장시간 작업, 특수 설정 필요 | 간헐적 개발, 빠른 테스트 |

> 🆕 **[2025 업데이트]**Serverless Compute와의 연동이 GA되어, 클러스터를 관리할 필요 없이 즉시 개발을 시작할 수 있습니다.

---

## 제한사항

Databricks Connect는 모든 Spark API를 지원하지는 않습니다. 다음 제한사항을 확인하세요.

| 카테고리 | 지원되지 않는 기능 |
|----------|-------------------|
| **Streaming**| `foreachBatch`의 일부 기능, 특정 소스/싱크 제한 |
| **RDD API**| RDD 기반 연산은 지원하지 않음 (DataFrame API 사용 권장) |
| **SparkContext**| `sc.parallelize()`, `sc.textFile()` 등 직접 사용 불가 |
| **UDF**| 복잡한 UDF에서 일부 제한 (로컬 라이브러리 참조 시) |
| **Delta Lake**| `MERGE`, `DELETE`, `UPDATE` 등은 SQL로 실행 가능 |
| **MLlib** | 분산 ML 학습은 제한적 (추론은 가능) |

### 제한 우회 방법

```python
# RDD 대신 DataFrame API 사용
# ❌ 지원하지 않음
# rdd = spark.sparkContext.parallelize([1, 2, 3])

# ✅ DataFrame API 사용
from pyspark.sql import Row
df = spark.createDataFrame([Row(value=1), Row(value=2), Row(value=3)])

# DML(MERGE 등)은 SQL로 실행
# ❌ DataFrame Writer의 merge는 제한적
# ✅ SQL 사용
spark.sql("""
    MERGE INTO catalog.schema.target t
    USING catalog.schema.source s
    ON t.id = s.id
    WHEN MATCHED THEN UPDATE SET *
    WHEN NOT MATCHED THEN INSERT *
""")
```

---

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| `ConnectionError` | 네트워크 차단 | VPN 연결 확인, 방화벽 규칙 확인 |
| `AuthenticationError` | 토큰 만료 또는 잘못된 인증 | PAT 재발급 또는 OAuth 재인증 |
| `VersionMismatch` | Connect 버전과 DBR 불일치 | `pip install databricks-connect==<DBR버전>.*` |
| `ClusterNotRunning` | 클러스터가 종료된 상태 | 클러스터 시작 또는 Serverless 사용 |
| `Timeout` | 대량 데이터를 로컬로 전송 시도 | `.limit()` 또는 `.toPandas()` 전에 필터링 |

```python
# 연결 상태 확인
from databricks.connect import DatabricksSession

spark = DatabricksSession.builder.profile("CONNECT").getOrCreate()

# 간단한 연산으로 연결 테스트
spark.range(10).count()
print("연결 성공!")
```

---

## 정리

| 핵심 개념 | 요약 |
|-----------|------|
| **Databricks Connect**| 로컬 IDE에서 원격 Databricks 클러스터로 코드를 실행하는 클라이언트 라이브러리 |
| ** 설치** | `pip install databricks-connect==<버전>.*` (DBR 버전과 일치 필수) |
| **인증**| OAuth 프로필(권장), PAT, 환경 변수 |
| **Serverless**| 클러스터 없이 즉시 실행 가능 |
| ** 제한사항** | RDD API, SparkContext 직접 사용 불가. DataFrame API와 SQL 사용 권장 |

---

## 참고 링크

- [Databricks Connect 공식 문서](https://docs.databricks.com/aws/en/dev-tools/databricks-connect/)
- [Databricks Connect 설치 가이드](https://docs.databricks.com/aws/en/dev-tools/databricks-connect/install.html)
- [VS Code용 Databricks Extension](https://docs.databricks.com/aws/en/dev-tools/vscode-ext/)
- [Serverless Compute](https://docs.databricks.com/aws/en/compute/serverless.html)
- [지원되지 않는 기능 목록](https://docs.databricks.com/aws/en/dev-tools/databricks-connect/limitations.html)
