# Notebook 사용법

## Notebook이란?

> 💡 **Notebook(노트북)**은 코드, 텍스트 설명, 시각화 결과를 하나의 문서에 담을 수 있는 **대화형 개발 환경** 입니다. 마치 실험 노트처럼, 코드를 작성하고 바로 실행하여 결과를 확인하면서 작업을 진행할 수 있습니다.

Jupyter Notebook과 유사하지만, Databricks Notebook은 다음과 같은 추가 기능을 제공합니다.

| 기능 | 설명 |
|------|------|
| **멀티 언어 지원**| 하나의 노트북 안에서 Python, SQL, Scala, R을 혼합하여 사용할 수 있습니다 |
| **실시간 협업**| 여러 사용자가 동시에 같은 노트북을 편집할 수 있습니다 (Google Docs처럼) |
| **클러스터 연결**| 클릭 한 번으로 분산 컴퓨팅 클러스터에 연결됩니다 |
| **버전 관리**| 노트북의 변경 이력이 자동으로 저장됩니다 |
| **Git 연동**| 외부 Git 저장소와 직접 연동하여 코드를 관리할 수 있습니다 |
| **시각화 내장**| 실행 결과를 바로 차트로 변환할 수 있습니다 |

---

## 셀(Cell)의 개념

Notebook은 여러 개의 **셀(Cell)** 로 구성되어 있습니다. 각 셀은 독립적으로 실행할 수 있습니다.

### 셀 유형

| 셀 유형 | 용도 | 표기 방법 |
|---------|------|-----------|
| **Code Cell**| 코드를 작성하고 실행합니다 | 기본 셀 (별도 표기 불필요) |
| **Markdown Cell**| 설명 텍스트를 작성합니다 | 셀 상단에 `%md` 입력 |
| **SQL Cell**| SQL 쿼리를 실행합니다 | 셀 상단에 `%sql` 입력 |
| **Shell Cell**| 쉘 명령어를 실행합니다 | 셀 상단에 `%sh` 입력 |

### 언어 전환 (Magic Command)

기본 언어가 Python인 노트북에서도, 셀 첫 줄에 **매직 커맨드** 를 입력하면 다른 언어를 사용할 수 있습니다.

```python
# 기본 언어가 Python인 노트북에서...

# Python 셀 (기본)
df = spark.read.table("catalog.schema.my_table")
df.show()
```

```sql
%sql
-- SQL 셀
SELECT * FROM catalog.schema.my_table LIMIT 10
```

```markdown
%md
## 분석 결과 요약
이 셀은 **Markdown** 으로 작성된 설명 텍스트입니다.
```

```bash
%sh
# Shell 명령어 실행
pip list | grep pandas
```

```python
%scala
// Scala 코드 실행
val df = spark.read.table("catalog.schema.my_table")
df.show()
```

---

## 기본 사용법

### 셀 실행

| 동작 | 단축키 | 설명 |
|------|--------|------|
| 현재 셀 실행 | `Shift + Enter` | 셀을 실행하고 다음 셀로 이동합니다 |
| 현재 셀만 실행 | `Ctrl/Cmd + Enter` | 셀을 실행하되 커서는 그 자리에 유지합니다 |
| 전체 실행 | 상단 **Run All** 버튼 | 노트북의 모든 셀을 순서대로 실행합니다 |
| 위쪽 전체 실행 | **Run All Above**| 현재 셀 위의 모든 셀을 실행합니다 |

### 셀 관리

| 동작 | 방법 |
|------|------|
| 새 셀 추가 | 셀 사이의 `+` 버튼 클릭 |
| 셀 삭제 | 셀 우측 `⋮` 메뉴 → Delete |
| 셀 이동 | 셀 좌측을 드래그하여 이동 |
| 셀 분할 | 커서 위치에서 `Ctrl/Cmd + Shift + -` |

---

## 실습: 첫 번째 Notebook 만들기

간단한 실습을 통해 Notebook 사용법을 익혀 보겠습니다.

### Step 1: Notebook 생성

1. Workspace에서 **+ New**→ **Notebook** 클릭
2. 이름: `my-first-notebook`
3. 기본 언어: **Python**
4. Cluster: **Serverless** 또는 사용 가능한 클러스터 선택

### Step 2: Python으로 데이터 생성

```python
# 셀 1: 간단한 데이터 생성
data = [
    ("김철수", 28, "서울", 4500000),
    ("이영희", 34, "부산", 5200000),
    ("박민수", 25, "대구", 3800000),
    ("최지은", 31, "서울", 4800000),
    ("정하늘", 29, "인천", 4100000),
]

columns = ["이름", "나이", "도시", "연봉"]

df = spark.createDataFrame(data, columns)
df.show()
```

실행 결과:
```
+------+---+----+-------+
|  이름|나이|도시|   연봉|
+------+---+----+-------+
|김철수| 28|서울|4500000|
|이영희| 34|부산|5200000|
|박민수| 25|대구|3800000|
|최지은| 31|서울|4800000|
|정하늘| 29|인천|4100000|
+------+---+----+-------+
```

### Step 3: SQL로 분석

```sql
%sql
-- 셀 2: 임시 뷰를 만들어서 SQL로 분석
-- (Python에서 만든 df를 SQL에서 사용하려면 임시 뷰를 만들어야 합니다)
```

```python
# 셀 3: 임시 뷰 생성
df.createOrReplaceTempView("employees")
```

```sql
%sql
-- 셀 4: 도시별 평균 연봉 조회
SELECT
    도시,
    COUNT(*) AS 인원수,
    FORMAT_NUMBER(AVG(연봉), 0) AS 평균연봉
FROM employees
GROUP BY 도시
ORDER BY AVG(연봉) DESC
```

### Step 4: 시각화

SQL 셀의 실행 결과 아래에 나타나는 **차트 아이콘(📊)** 을 클릭하면, 결과를 다양한 차트로 변환할 수 있습니다.

| 차트 유형 | 적합한 데이터 |
|-----------|-------------|
| **Bar Chart**| 카테고리별 비교 (도시별 매출 등) |
| **Line Chart**| 시계열 데이터 (일별 매출 추이 등) |
| **Pie Chart**| 비율 비교 (카테고리 비중 등) |
| **Scatter Plot**| 두 변수 간의 관계 |
| **Map**| 지리 데이터 |

### Step 5: Markdown으로 설명 추가

```markdown
%md
## 분석 결과

위 데이터는 5명의 직원 샘플 데이터입니다.
- **서울** 근무자의 평균 연봉이 가장 높습니다.
- 전체 평균 나이는 약 **29세** 입니다.
```

---

## Jupyter에서 넘어온 사람이 처음 당황하는 5가지

Jupyter Notebook을 사용하던 분들이 Databricks Notebook으로 옮기면 겪는 흔한 혼란을 정리했습니다.

### 1. "pip install이 왜 다른가?"

Jupyter에서는 터미널에서 `pip install`을 하면 끝이지만, Databricks에서는 클러스터 전체에 라이브러리를 설치해야 합니다.

```python
# Databricks에서 라이브러리 설치 (노트북 내에서)
%pip install pandas==2.1.0 scikit-learn==1.3.0

# 반드시 설치 후 restart 필요 (Databricks가 자동으로 안내합니다)
dbutils.library.restartPython()
```

> ⚠️ **주의**: `%pip install`은 해당 노트북이 연결된 클러스터에만 적용됩니다. 다른 사용자가 같은 클러스터를 공유하는 경우, 라이브러리 버전 충돌이 발생할 수 있습니다. 프로덕션에서는 클러스터 설정의 **Libraries** 탭에서 관리하는 것이 안전합니다.

### 2. "변수가 왜 사라지나?"

Jupyter는 커널이 항상 살아있지만, Databricks 클러스터는 **Auto-termination**(자동 종료) 이후 재시작하면 모든 변수가 초기화됩니다. 또한 클러스터를 **Detach**했다가 다시 **Attach** 하면 상태가 초기화됩니다.

> 💡 **해결법**: 중간 결과는 Delta 테이블에 저장하세요. `df.write.saveAsTable("my_catalog.my_schema.temp_result")` 한 줄이면 됩니다. 변수가 사라져도 데이터는 남아 있습니다.

### 3. "spark라는 변수는 어디서 온 거지?"

Jupyter에서는 `SparkSession.builder.getOrCreate()`를 직접 호출해야 하지만, Databricks에서는 **`spark` 변수가 자동으로 생성** 되어 있습니다. 마찬가지로 `sc`(SparkContext), `sqlContext`, `dbutils`도 미리 선언되어 있습니다.

```python
# Jupyter에서는 이렇게 해야 하지만...
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("my_app").getOrCreate()

# Databricks에서는 그냥 바로 사용합니다
df = spark.read.table("my_catalog.my_schema.my_table")
```

### 4. "display()가 뭐지?"

Databricks에는 `display()`라는 고유 함수가 있습니다. Jupyter의 `df.show()`와 달리, **인터랙티브한 테이블**과 **차트 시각화** 를 제공합니다.

```python
# df.show()는 텍스트 출력 (Jupyter 스타일)
df.show(10)

# display()는 인터랙티브 테이블 + 차트 (Databricks 스타일) — 이것을 쓰세요
display(df)
```

> ⚠️ **대형 테이블 주의**: `display(df)`는 기본적으로 1,000행만 표시하지만, 내부적으로 쿼리를 실행합니다. 수십억 건의 테이블에 `display()`를 호출하면 집계 없이 전체 스캔이 발생할 수 있습니다. 반드시 `display(df.limit(100))` 또는 `display(df.filter(...))` 처럼 범위를 제한하세요. 현업에서 실수로 `display(spark.read.table("10TB_table"))`을 호출하여 클러스터가 OOM(메모리 부족)으로 죽는 사고가 실제로 발생합니다.

### 5. "노트북 출력이 저장되지 않는다?"

Jupyter는 `.ipynb` 파일에 출력 결과가 함께 저장되지만, Databricks Notebook은 기본적으로 **코드만 Git에 저장**됩니다. 출력 결과를 보존하려면 **HTML로 내보내기** 하거나, Databricks의 자체 리비전 히스토리를 사용해야 합니다.

---

## %sql 결과를 Python DataFrame으로 변환하는 실전 패턴

노트북에서 `%sql`과 `%python`을 혼합할 때, SQL 결과를 Python에서 사용하는 방법이 필요합니다.

### 방법 1: 임시 뷰 활용 (가장 일반적)

```sql
%sql
-- SQL에서 복잡한 쿼리 실행
CREATE OR REPLACE TEMP VIEW daily_sales AS
SELECT order_date, SUM(amount) AS total_sales
FROM catalog.schema.orders
GROUP BY order_date
ORDER BY order_date;
```

```python
# Python에서 임시 뷰를 DataFrame으로 읽기
df = spark.table("daily_sales")

# 이제 Python으로 추가 처리 가능
import pandas as pd
pandas_df = df.toPandas()  # 데이터가 작을 때만 사용
```

### 방법 2: spark.sql() 함수 활용

```python
# Python 셀에서 SQL을 직접 실행하고 결과를 DataFrame으로 받기
df = spark.sql("""
    SELECT order_date, SUM(amount) AS total_sales
    FROM catalog.schema.orders
    WHERE order_date >= '2025-01-01'
    GROUP BY order_date
""")
display(df)
```

### 방법 3: _sqldf 매직 변수 (간편하지만 비공식)

```sql
%sql
SELECT * FROM catalog.schema.orders LIMIT 100
```

```python
# 직전 %sql 셀의 결과가 _sqldf에 자동 저장됩니다
display(_sqldf)
```

> 💡 **현업에서는 이렇게 합니다**: SQL이 가독성이 좋은 경우(집계, 조인 등)에는 `%sql`로 쿼리를 작성하고, 그 결과를 Python에서 ML 모델에 넣거나 복잡한 로직을 처리할 때 혼합 사용합니다. "SQL을 잘 하는 분석가가 쿼리를 작성하고, ML 엔지니어가 Python으로 모델을 학습하는" 협업 패턴이 매우 흔합니다.

---

## Notebook 고급 기능

### 위젯(Widgets) — 동적 매개변수

위젯을 사용하면 노트북 상단에 드롭다운, 텍스트 입력 등을 추가하여, 코드를 수정하지 않고도 매개변수를 변경할 수 있습니다.

```python
# 드롭다운 위젯 생성
dbutils.widgets.dropdown("city", "서울", ["서울", "부산", "대구", "인천"])

# 위젯 값 사용
selected_city = dbutils.widgets.get("city")
display(df.filter(df.도시 == selected_city))
```

### dbutils — Databricks 유틸리티

> 💡 **dbutils** 는 Databricks Notebook에서 사용할 수 있는 유틸리티 모음입니다. 파일 시스템 접근, 비밀 키 관리, 위젯 제어 등의 기능을 제공합니다.

| 유틸리티 | 용도 | 예시 |
|----------|------|------|
| `dbutils.fs` | 파일 시스템 조작 | `dbutils.fs.ls("/mnt/data")` |
| `dbutils.widgets` | 매개변수 위젯 | `dbutils.widgets.text("name", "default")` |
| `dbutils.secrets` | 비밀 키 접근 | `dbutils.secrets.get("scope", "key")` |
| `dbutils.notebook` | 다른 노트북 실행 | `dbutils.notebook.run("./other", 60)` |

### dbutils 핵심 기능 상세

`dbutils`는 Databricks Notebook에서만 사용할 수 있는 강력한 유틸리티입니다. 각 모듈별로 실전에서 가장 많이 쓰이는 기능을 상세히 살펴보겠습니다.

#### dbutils.secrets — 비밀 키 관리

비밀번호, API 키, 접속 정보를 코드에 직접 입력하면 안 됩니다. `dbutils.secrets`를 사용하면 안전하게 관리할 수 있습니다.

```python
# 절대 이렇게 하지 마세요!!
password = "my_super_secret_123"  # 코드에 비밀번호 하드코딩 ❌

# 이렇게 하세요
password = dbutils.secrets.get(scope="my-scope", key="db-password")
jdbc_url = f"jdbc:postgresql://host:5432/mydb?user=admin&password={password}"
```

> ⚠️ **현업에서의 사고 사례**: 노트북에 AWS Access Key를 하드코딩한 채로 Git에 push하여 보안 사고가 발생한 경우가 실제로 있습니다. `dbutils.secrets`를 쓰면 값이 `[REDACTED]`로 표시되어 실수로 노출될 위험이 없습니다.

#### dbutils.widgets — 매개변수화된 노트북

위젯은 노트북을 재사용 가능한 템플릿으로 만들어 줍니다.

```python
# 위젯 생성
dbutils.widgets.text("start_date", "2025-01-01", "시작 날짜")
dbutils.widgets.text("end_date", "2025-03-31", "종료 날짜")
dbutils.widgets.dropdown("env", "dev", ["dev", "staging", "prod"], "환경")

# 위젯 값 사용
start = dbutils.widgets.get("start_date")
end = dbutils.widgets.get("end_date")
env = dbutils.widgets.get("env")

catalog = f"catalog_{env}"  # catalog_dev, catalog_staging, catalog_prod
df = spark.sql(f"""
    SELECT * FROM {catalog}.sales.orders
    WHERE order_date BETWEEN '{start}' AND '{end}'
""")
```

> 💡 위젯은 **Lakeflow Jobs에서 파라미터로 전달** 할 수 있습니다. 개발 시에는 위젯으로 값을 입력하고, 운영 시에는 Jobs에서 자동으로 파라미터를 주입하는 패턴이 매우 유용합니다.

#### dbutils.notebook.run — 노트북 간 호출

```python
# 다른 노트북을 호출하고 결과를 받기
result = dbutils.notebook.run(
    path="./etl/load_customers",
    timeout_seconds=600,
    arguments={"date": "2025-03-15", "env": "prod"}
)
print(f"결과: {result}")  # 호출된 노트북의 dbutils.notebook.exit() 반환값
```

> 💡 **현업에서는 이렇게 합니다**: `dbutils.notebook.run`은 간단한 워크플로우에는 유용하지만, 복잡한 파이프라인에서는 **Lakeflow Jobs** 를 사용하는 것이 좋습니다. Jobs는 병렬 실행, 재시도, 알림, 의존성 관리 등 운영에 필요한 기능을 모두 제공합니다.

#### dbutils.fs — 파일 시스템 조작

```python
# 파일 목록 확인
dbutils.fs.ls("/Volumes/my_catalog/my_schema/my_volume/")

# 파일 복사
dbutils.fs.cp(
    "dbfs:/Volumes/catalog/schema/raw/data.csv",
    "dbfs:/Volumes/catalog/schema/archive/data_20250315.csv"
)

# 파일/폴더 삭제 (recurse=True면 하위 전체 삭제)
dbutils.fs.rm("/Volumes/catalog/schema/temp/", recurse=True)
```

---

### 노트북 버전 관리 전략

노트북의 버전 관리는 크게 두 가지 방법이 있습니다.

#### 방법 1: Databricks 내장 리비전 히스토리

노트북 우측 상단의 **Revision history** 버튼을 클릭하면, 자동 저장된 모든 버전을 확인하고 복원할 수 있습니다.

| 장점 | 단점 |
|------|------|
| 별도 설정 불필요 | 브랜치, PR 등의 협업 기능이 없습니다 |
| 자동 저장 | 다른 프로젝트와 코드를 공유하기 어렵습니다 |
| 특정 시점으로 복원 가능 | 코드 리뷰 프로세스가 없습니다 |

**적합한 경우**: 개인 실험, 일회성 분석, 빠른 프로토타이핑

#### 방법 2: Git 연동 (Repos)

Databricks의 **Repos** 기능을 사용하면 GitHub, GitLab, Azure DevOps 등의 Git 저장소와 연동할 수 있습니다.

| 장점 | 단점 |
|------|------|
| 브랜치, PR, 코드 리뷰 가능 | 초기 설정이 필요합니다 |
| CI/CD 파이프라인 연동 | Git 사용법을 알아야 합니다 |
| 팀 협업에 필수적 | 노트북 출력 결과는 저장되지 않습니다 |
| 다른 IDE(VS Code 등)와 함께 사용 가능 | |

**적합한 경우**: 팀 프로젝트, 프로덕션 코드, 코드 리뷰가 필요한 모든 경우

> 💡 **현업에서는 이렇게 합니다**: "개인 실험은 Workspace 폴더에서, 프로덕션 코드는 반드시 Repos(Git)로" 라는 원칙을 팀 내에서 명확히 합니다. 또한 Databricks에서는 **노트북을 `.py` 파일로 저장하는 방식**(Databricks Notebook 포맷)을 지원하므로, Git에서 diff(변경 사항 비교)가 깔끔하게 표시됩니다.

---

### 결과 내보내기

노트북 실행 결과는 다양한 형태로 내보낼 수 있습니다.

- **DBC 파일**: Databricks 전용 포맷 (다른 Workspace로 가져오기 가능)
- **IPython Notebook (.ipynb)**: Jupyter Notebook 포맷
- **HTML**: 실행 결과를 포함한 웹 페이지
- **Python/SQL 파일**: 코드만 추출

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **셀 기반 실행**| 코드를 셀 단위로 나누어 실행하고, 결과를 바로 확인할 수 있습니다 |
| **매직 커맨드**| `%sql`, `%md`, `%sh` 등으로 하나의 노트북에서 여러 언어를 혼합 사용할 수 있습니다 |
| **실시간 협업**| 여러 사용자가 동시에 편집하고, 변경 이력이 자동 저장됩니다 |
| **시각화**| SQL/DataFrame 결과를 클릭 한 번으로 차트로 변환할 수 있습니다 |
| **dbutils**| 파일 접근, 비밀 키 관리, 위젯 등 유틸리티를 제공합니다 |

다음 문서에서는 Databricks **무료 체험** 을 시작하는 방법을 안내해 드리겠습니다.

---

## 참고 링크

- [Databricks: Notebooks](https://docs.databricks.com/aws/en/notebooks/)
- [Azure Databricks: Notebooks](https://learn.microsoft.com/en-us/azure/databricks/notebooks/)
- [Databricks: dbutils](https://docs.databricks.com/aws/en/dev-tools/databricks-utils.html)
