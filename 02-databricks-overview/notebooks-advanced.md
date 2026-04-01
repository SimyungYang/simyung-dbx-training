# Notebook 고급 기능

## 개요

[Notebook 기본 사용법](./notebooks-basics.md)에서 셀 실행, 언어 전환 등 기초를 배웠습니다. 이 문서에서는 **위젯을 활용한 대화형 파라미터, 노트북 워크플로, 디버깅, Git 연동, 협업**등 생산성을 크게 높이는 고급 기능을 살펴보겠습니다.

---

## 위젯 (dbutils.widgets) — 대화형 파라미터

위젯은 노트북 상단에 ** 대화형 입력 컨트롤**을 추가하여, 코드를 수정하지 않고도 파라미터를 변경할 수 있게 합니다.

### 위젯 유형

| 위젯 유형 | 설명 | 사용 예시 |
|-----------|------|----------|
| **text**| 자유 텍스트 입력 | 테이블 이름, 경로 입력 |
| **dropdown**| 드롭다운 선택 | 환경 선택 (dev/staging/prod) |
| **combobox**| 드롭다운 + 자유 입력 | 자주 사용하는 값 + 커스텀 값 |
| **multiselect** | 다중 선택 | 여러 지역 동시 선택 |

### 위젯 생성

```python
# 텍스트 위젯
dbutils.widgets.text("start_date", "2025-01-01", "시작 날짜")

# 드롭다운 위젯
dbutils.widgets.dropdown("environment", "dev", ["dev", "staging", "prod"], "환경")

# 콤보박스 위젯
dbutils.widgets.combobox("table_name", "sales", ["sales", "orders", "customers"], "테이블")

# 다중 선택 위젯
dbutils.widgets.multiselect("regions", "서울", ["서울", "부산", "대구", "인천"], "지역")
```

### 위젯 값 사용

```python
# Python에서 위젯 값 가져오기
start_date = dbutils.widgets.get("start_date")
env = dbutils.widgets.get("environment")

df = spark.sql(f"""
    SELECT * FROM {env}_catalog.schema.orders
    WHERE order_date >= '{start_date}'
""")
```

```sql
-- SQL에서 위젯 값 사용 (getArgument 함수)
SELECT *
FROM ${environment}_catalog.schema.orders
WHERE order_date >= '${start_date}'
```

### 위젯 관리

```python
# 특정 위젯 제거
dbutils.widgets.remove("start_date")

# 모든 위젯 제거
dbutils.widgets.removeAll()
```

> 💡 **활용 팁**: 위젯을 사용하면 **하나의 노트북으로 여러 환경(dev/prod), 여러 날짜 범위**를 처리할 수 있습니다. Lakeflow Jobs에서 노트북을 실행할 때 위젯 값을 파라미터로 전달할 수도 있습니다.

---

## 다국어 지원 (%python, %sql, %scala, %r)

Databricks Notebook은 **하나의 노트북 안에서 여러 언어를 혼합**하여 사용할 수 있습니다. 각 셀의 첫 줄에 매직 커맨드를 입력하면 해당 셀의 언어가 전환됩니다.

### 매직 커맨드 목록

| 매직 커맨드 | 언어 | 사용 시나리오 |
|------------|------|-------------|
| `%python` | Python | 데이터 처리, ML, 범용 프로그래밍 |
| `%sql` | SQL | 데이터 조회, 테이블 관리 |
| `%scala` | Scala | Spark 네이티브 작업, 고성능 처리 |
| `%r` | R | 통계 분석, 시각화 |
| `%md` | Markdown | 문서화, 설명 텍스트 |
| `%sh` | Shell | 시스템 명령어, 패키지 설치 |
| `%pip` | pip | Python 라이브러리 설치 |

### 언어 간 데이터 전달

서로 다른 언어의 셀 간에 데이터를 전달하려면 **임시 뷰(Temporary View)**를 사용합니다.

```python
# Python 셀: DataFrame을 임시 뷰로 등록
df = spark.read.table("catalog.schema.orders")
df.filter("amount > 10000").createOrReplaceTempView("high_value_orders")
```

```sql
%sql
-- SQL 셀: 임시 뷰를 쿼리
SELECT region, COUNT(*) as order_count, SUM(amount) as total
FROM high_value_orders
GROUP BY region
ORDER BY total DESC
```

```r
%r
# R 셀: SQL 결과를 R로 가져와서 시각화
library(SparkR)
df <- sql("SELECT * FROM high_value_orders")
summary(df)
```

---

## Notebook 워크플로 (dbutils.notebook.run)

하나의 노트북에서 **다른 노트북을 호출** 하여 모듈화된 워크플로를 구성할 수 있습니다.

### 기본 사용법

```python
# 다른 노트북 실행 (동기)
result = dbutils.notebook.run(
    path="/Users/team/etl/process_sales",   # 실행할 노트북 경로
    timeout_seconds=600,                     # 타임아웃 (초)
    arguments={"date": "2025-03-01", "env": "prod"}  # 파라미터 전달
)
print(f"결과: {result}")  # 호출된 노트북에서 dbutils.notebook.exit()로 반환한 값
```

### 호출되는 노트북 (자식 노트북)

```python
# 파라미터 받기
date = dbutils.widgets.get("date")
env = dbutils.widgets.get("env")

# 작업 수행
row_count = spark.sql(f"""
    SELECT COUNT(*) FROM {env}_catalog.schema.sales
    WHERE date = '{date}'
""").first()[0]

# 결과 반환
dbutils.notebook.exit(f"처리 완료: {row_count}건")
```

### 병렬 실행 패턴

여러 노트북을 **동시에 실행**하려면 `concurrent.futures`를 활용합니다.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

notebooks = [
    {"path": "/etl/process_sales", "args": {"region": "서울"}},
    {"path": "/etl/process_sales", "args": {"region": "부산"}},
    {"path": "/etl/process_sales", "args": {"region": "대구"}},
]

def run_notebook(nb):
    return dbutils.notebook.run(nb["path"], 600, nb["args"])

with ThreadPoolExecutor(max_workers=3) as executor:
    futures = {executor.submit(run_notebook, nb): nb for nb in notebooks}
    for future in as_completed(futures):
        nb = futures[future]
        try:
            result = future.result()
            print(f"{nb['args']['region']}: {result}")
        except Exception as e:
            print(f"{nb['args']['region']}: 오류 - {e}")
```

> ⚠️ **Lakeflow Jobs 권장**: 복잡한 워크플로는 `dbutils.notebook.run` 대신 **Lakeflow Jobs**를 사용하는 것이 좋습니다. Lakeflow Jobs는 DAG 형태의 의존성 관리, 재시도, 모니터링, 알림 기능을 제공합니다.

---

## 디버깅 기능

### 변수 탐색기

Databricks Notebook은 **변수 탐색기(Variable Explorer)**를 기본 제공합니다.

| 기능 | 설명 |
|------|------|
| **변수 목록**| 현재 세션의 모든 변수를 이름, 유형, 값과 함께 표시합니다 |
| **DataFrame 미리보기**| DataFrame 변수를 클릭하면 데이터를 테이블 형태로 미리봅니다 |
| ** 값 검사**| 복잡한 객체(dict, list)의 내부를 펼쳐서 확인할 수 있습니다 |

변수 탐색기는 노트북 하단의 "**Variables**" 탭에서 확인할 수 있습니다.

### Python 디버거 (pdb)

```python
# 셀에서 디버거 사용
import pdb

def process_data(df):
    filtered = df.filter("amount > 0")
    pdb.set_trace()  # 여기서 실행이 중단되고 디버거 프롬프트가 표시됩니다
    result = filtered.groupBy("region").sum("amount")
    return result
```

### 일반적인 디버깅 팁

| 팁 | 방법 |
|----|------|
| **중간 결과 확인** | `display(df)` 또는 `df.show()`로 DataFrame 내용을 확인합니다 |
| **행 수 확인** | `df.count()`로 각 변환 단계의 행 수를 검증합니다 |
| **스키마 확인** | `df.printSchema()`로 컬럼 이름과 타입을 확인합니다 |
| **셀 단위 실행** | 문제가 있는 부분을 작은 셀로 나누어 단계별로 실행합니다 |
| **try-except**| 오류가 예상되는 부분에 예외 처리를 추가합니다 |

---

## 버전 관리 (Git 연동)

### Databricks Repos (Git Folders)

Databricks는 **Git 저장소와 직접 연동**하여 노트북을 버전 관리할 수 있습니다.

| 지원 Git 서비스 | 설명 |
|----------------|------|
| **GitHub**| Personal Access Token 또는 GitHub App으로 연동 |
| **GitLab**| Personal Access Token으로 연동 |
| **Azure DevOps**| Azure AD 토큰으로 연동 |
| **Bitbucket**| App Password로 연동 |

### Git 연동 워크플로

```
1. Repos에서 Git 저장소 클론
   Workspace → Repos → Add Repo → URL 입력

2. 브랜치 생성/전환
   Repos UI 상단의 브랜치 드롭다운에서 선택

3. 노트북 편집 및 커밋
   변경 후 Repos UI에서 Commit & Push

4. Pull Request 생성 (Git 서비스에서)
   코드 리뷰 후 메인 브랜치에 병합
```

### CI/CD 연동

```yaml
# GitHub Actions 예시: 노트북 테스트 자동화
name: Test Notebooks
on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run notebook tests
        uses: databricks/run-notebook@v0
        with:
          workspace-url: ${{ secrets.DATABRICKS_HOST }}
          notebook-path: /Repos/main/tests/test_etl
```

---

## 협업 기능

### 실시간 공동 편집

Databricks Notebook은 **Google Docs처럼 여러 사용자가 동시에 편집**할 수 있습니다.

| 기능 | 설명 |
|------|------|
| **실시간 커서**| 다른 사용자의 커서 위치가 실시간으로 표시됩니다 |
| ** 동시 편집**| 같은 노트북을 동시에 편집해도 충돌이 발생하지 않습니다 |
| ** 셀 잠금**| 한 사용자가 편집 중인 셀은 다른 사용자에게 표시됩니다 |

### 코멘트 기능

```
셀 선택 → 우클릭 → "Comment" 또는 Cmd/Ctrl + Shift + C
  → 코멘트 입력 → 다른 사용자 @멘션 가능
  → 코드 리뷰, 질문, 피드백에 활용
```

| 코멘트 활용 | 설명 |
|------------|------|
| ** 코드 리뷰**| 특정 셀의 코드에 대한 피드백을 남깁니다 |
| ** 질문**| 로직이 불명확한 부분에 질문을 남깁니다 |
| ** 할 일 표시**| 수정이 필요한 부분을 표시합니다 |
| ** 승인**| 코드가 검토 완료되었음을 표시합니다 |

### 공유 및 권한

```
노트북 우상단 "Share" 버튼
  → 사용자/그룹 검색
  → 권한 수준 선택:
     - Can View: 읽기만 가능
     - Can Run: 읽기 + 실행
     - Can Edit: 읽기 + 실행 + 편집
     - Can Manage: 모든 권한 + 권한 관리
```

---

## 노트북 결과를 대시보드로

노트북의 실행 결과를 ** 간단한 대시보드**로 변환할 수 있습니다.

### 노트북 대시보드 생성

1. 노트북 상단 메뉴에서 **View → Create Dashboard**를 선택합니다
2. 시각화 결과가 있는 셀이 자동으로 대시보드 위젯으로 변환됩니다
3. 위젯의 **크기와 위치를 드래그**하여 배치합니다
4. 위젯을 사용하면 대시보드에서 **대화형 필터**를 제공할 수 있습니다

> 💡 **노트북 대시보드 vs AI/BI 대시보드**: 노트북 대시보드는 빠른 프로토타이핑에 적합하고, 정식 리포트에는 **AI/BI Dashboard (Lakeview)**를 사용하는 것이 좋습니다. Lakeview 대시보드는 더 풍부한 시각화, 자동 새로고침, 세밀한 공유 기능을 제공합니다.

---

## 유용한 매직 커맨드 모음

| 매직 커맨드 | 용도 | 예시 |
|------------|------|------|
| `%pip install` | Python 패키지 설치 | `%pip install xgboost` |
| `%sh` | 쉘 명령어 실행 | `%sh ls /dbfs/mnt/` |
| `%fs` | DBFS 파일 시스템 명령어 | `%fs ls /mnt/data/` |
| `%md` | Markdown 텍스트 | `%md ## 분석 결과` |
| `%run` | 다른 노트북 실행 (인라인) | `%run ./utils/common_functions` |

### %run vs dbutils.notebook.run

| 비교 | `%run` | `dbutils.notebook.run` |
|------|--------|----------------------|
| **실행 방식**| 현재 노트북에 인라인으로 삽입 | 별도 프로세스에서 독립 실행 |
| ** 변수 공유**| 호출된 노트북의 변수에 접근 가능 | 불가 (문자열 반환만 가능) |
| ** 파라미터**| 위젯 또는 변수 공유 | arguments 딕셔너리로 전달 |
| ** 격리**| 격리 없음 (같은 컨텍스트) | 완전 격리 |
| ** 사용 시나리오**| 공통 함수 import | 독립적인 작업 실행 |

---

## 모범 사례

| 항목 | 권장 사항 |
|------|----------|
| ** 위젯 활용**| 하드코딩 대신 위젯으로 파라미터화하여 재사용성을 높입니다 |
| **Markdown 문서화**| 코드 사이에 %md 셀로 분석 의도와 결과를 설명합니다 |
| **Git 연동**| 중요한 노트북은 반드시 Git으로 버전 관리합니다 |
| ** 모듈화**| 공통 함수는 별도 노트북에 작성하고 %run으로 불러옵니다 |
| ** 셀 크기 제한**| 하나의 셀에 너무 많은 코드를 넣지 않고, 논리적 단위로 분리합니다 |
| ** 결과 정리** | 노트북을 공유하기 전에 불필요한 셀을 정리하고, 전체 실행(Run All)으로 결과를 갱신합니다 |

---

## 정리

| 기능 | 핵심 내용 |
|------|----------|
| 위젯 | 대화형 파라미터로 코드 수정 없이 입력값을 변경합니다 |
| 다국어 | %python, %sql, %r, %scala를 한 노트북에서 혼합 사용합니다 |
| Notebook 워크플로 | dbutils.notebook.run으로 다른 노트북을 호출합니다 |
| Git 연동 | Repos를 통해 GitHub, GitLab 등과 버전 관리를 연동합니다 |
| 협업 | 실시간 공동 편집, 코멘트, 세밀한 권한 관리를 지원합니다 |
| 대시보드 | 노트북 결과를 간단한 대시보드로 변환할 수 있습니다 |

---

## 참고 링크

- [Databricks Notebook 공식 문서](https://docs.databricks.com/aws/en/notebooks/)
- [위젯 (dbutils.widgets)](https://docs.databricks.com/aws/en/notebooks/widgets.html)
- [Notebook 워크플로 (dbutils.notebook)](https://docs.databricks.com/aws/en/notebooks/notebook-workflows.html)
- [Git 연동 (Repos)](https://docs.databricks.com/aws/en/repos/)
- [실시간 협업](https://docs.databricks.com/aws/en/notebooks/notebook-collaboration.html)
