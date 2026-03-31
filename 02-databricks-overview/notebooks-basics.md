# Notebook 사용법

## Notebook이란?

> 💡 **Notebook(노트북)** 은 코드, 텍스트 설명, 시각화 결과를 하나의 문서에 담을 수 있는 **대화형 개발 환경**입니다. 마치 실험 노트처럼, 코드를 작성하고 바로 실행하여 결과를 확인하면서 작업을 진행할 수 있습니다.

Jupyter Notebook과 유사하지만, Databricks Notebook은 다음과 같은 추가 기능을 제공합니다.

| 기능 | 설명 |
|------|------|
| **멀티 언어 지원** | 하나의 노트북 안에서 Python, SQL, Scala, R을 혼합하여 사용할 수 있습니다 |
| **실시간 협업** | 여러 사용자가 동시에 같은 노트북을 편집할 수 있습니다 (Google Docs처럼) |
| **클러스터 연결** | 클릭 한 번으로 분산 컴퓨팅 클러스터에 연결됩니다 |
| **버전 관리** | 노트북의 변경 이력이 자동으로 저장됩니다 |
| **Git 연동** | 외부 Git 저장소와 직접 연동하여 코드를 관리할 수 있습니다 |
| **시각화 내장** | 실행 결과를 바로 차트로 변환할 수 있습니다 |

---

## 셀(Cell)의 개념

Notebook은 여러 개의 **셀(Cell)** 로 구성되어 있습니다. 각 셀은 독립적으로 실행할 수 있습니다.

### 셀 유형

| 셀 유형 | 용도 | 표기 방법 |
|---------|------|-----------|
| **Code Cell** | 코드를 작성하고 실행합니다 | 기본 셀 (별도 표기 불필요) |
| **Markdown Cell** | 설명 텍스트를 작성합니다 | 셀 상단에 `%md` 입력 |
| **SQL Cell** | SQL 쿼리를 실행합니다 | 셀 상단에 `%sql` 입력 |
| **Shell Cell** | 쉘 명령어를 실행합니다 | 셀 상단에 `%sh` 입력 |

### 언어 전환 (Magic Command)

기본 언어가 Python인 노트북에서도, 셀 첫 줄에 **매직 커맨드**를 입력하면 다른 언어를 사용할 수 있습니다.

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
| 위쪽 전체 실행 | **Run All Above** | 현재 셀 위의 모든 셀을 실행합니다 |

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

1. Workspace에서 **+ New** → **Notebook** 클릭
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
| **Bar Chart** | 카테고리별 비교 (도시별 매출 등) |
| **Line Chart** | 시계열 데이터 (일별 매출 추이 등) |
| **Pie Chart** | 비율 비교 (카테고리 비중 등) |
| **Scatter Plot** | 두 변수 간의 관계 |
| **Map** | 지리 데이터 |

### Step 5: Markdown으로 설명 추가

```markdown
%md
## 분석 결과

위 데이터는 5명의 직원 샘플 데이터입니다.
- **서울** 근무자의 평균 연봉이 가장 높습니다.
- 전체 평균 나이는 약 **29세**입니다.
```

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

> 💡 **dbutils**는 Databricks Notebook에서 사용할 수 있는 유틸리티 모음입니다. 파일 시스템 접근, 비밀 키 관리, 위젯 제어 등의 기능을 제공합니다.

| 유틸리티 | 용도 | 예시 |
|----------|------|------|
| `dbutils.fs` | 파일 시스템 조작 | `dbutils.fs.ls("/mnt/data")` |
| `dbutils.widgets` | 매개변수 위젯 | `dbutils.widgets.text("name", "default")` |
| `dbutils.secrets` | 비밀 키 접근 | `dbutils.secrets.get("scope", "key")` |
| `dbutils.notebook` | 다른 노트북 실행 | `dbutils.notebook.run("./other", 60)` |

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
| **셀 기반 실행** | 코드를 셀 단위로 나누어 실행하고, 결과를 바로 확인할 수 있습니다 |
| **매직 커맨드** | `%sql`, `%md`, `%sh` 등으로 하나의 노트북에서 여러 언어를 혼합 사용할 수 있습니다 |
| **실시간 협업** | 여러 사용자가 동시에 편집하고, 변경 이력이 자동 저장됩니다 |
| **시각화** | SQL/DataFrame 결과를 클릭 한 번으로 차트로 변환할 수 있습니다 |
| **dbutils** | 파일 접근, 비밀 키 관리, 위젯 등 유틸리티를 제공합니다 |

다음 문서에서는 Databricks **무료 체험**을 시작하는 방법을 안내해 드리겠습니다.

---

## 참고 링크

- [Databricks: Notebooks](https://docs.databricks.com/aws/en/notebooks/)
- [Azure Databricks: Notebooks](https://learn.microsoft.com/en-us/azure/databricks/notebooks/)
- [Databricks: dbutils](https://docs.databricks.com/aws/en/dev-tools/databricks-utils.html)
