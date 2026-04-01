# Lakeflow Jobs 태스크 유형 상세

## 왜 태스크 유형을 이해해야 하는가?

Lakeflow Jobs는 다양한 워크로드를 하나의 워크플로로 통합합니다. **Notebook, SQL, Python Script, dbt, SDP 파이프라인** 등 10가지 태스크 유형을 제공하며, 각 유형은 특정 용도에 최적화되어 있습니다. 올바른 태스크 유형을 선택하면 개발 생산성과 운영 효율성이 크게 향상됩니다.

---

## 10가지 태스크 유형 총정리

| 태스크 유형 | 설명 | 컴퓨트 | 대표 사용 사례 |
|------------|------|--------|-------------|
| **Notebook**| Databricks 노트북을 실행합니다 | Job Cluster / Serverless | ETL 변환, ML 학습 |
| **Python Script**| `.py` 파일을 실행합니다 | Job Cluster / Serverless | 범용 Python 처리 |
| **SQL**| SQL 쿼리/파일을 실행합니다 | SQL Warehouse | 데이터 검증, 집계 |
| **dbt**| dbt 프로젝트 태스크를 실행합니다 | SQL Warehouse | dbt 모델 빌드 |
| **Pipeline (SDP)**| SDP 파이프라인을 트리거합니다 | 파이프라인 자체 클러스터 | 스트리밍 수집 |
| **JAR**| Java/Scala JAR을 실행합니다 | Job Cluster | Java 기반 ETL |
| **Spark Submit**| spark-submit으로 실행합니다 | Job Cluster | 레거시 Spark 앱 |
| **Run Job**| 다른 Job을 트리거합니다 | 대상 Job의 컴퓨트 | 크로스 팀 오케스트레이션 |
| **If/Else**| 조건에 따라 분기합니다 | 없음 (제어 흐름만) | 조건부 처리 |
| **For Each**| 리스트 항목별 반복 실행합니다 | 없음 (제어 흐름만) | 멀티 테넌트 처리 |

---

## Notebook 태스크

가장 자주 사용되는 태스크 유형입니다. Databricks 노트북을 실행하며, **Python, Scala, SQL, R** 언어를 지원합니다.

```yaml
tasks:
  - task_key: "transform_orders"
    notebook_task:
      notebook_path: "/Workspace/etl/transform_orders"
      base_parameters:
        target_date: "{{job.parameters.target_date}}"
        mode: "incremental"
      source: WORKSPACE  # 또는 GIT
    new_cluster:
      spark_version: "15.4.x-scala2.12"
      num_workers: 4
      node_type_id: "r5.xlarge"
```

| 설정 | 설명 |
|------|------|
| `notebook_path` | 실행할 노트북의 Workspace 경로입니다 |
| `base_parameters` | 노트북에 전달할 파라미터입니다 (`dbutils.widgets`로 수신) |
| `source` | 노트북 소스입니다 (WORKSPACE 또는 GIT) |

---

## Python Script 태스크

`.py` 파일을 직접 실행합니다. Notebook 태스크와 달리 **순수 Python 스크립트** 를 실행하므로, CI/CD 파이프라인과의 연동이 쉽습니다.

```yaml
tasks:
  - task_key: "data_validation"
    python_wheel_task:
      package_name: "my_etl_package"
      entry_point: "validate"
      parameters: ["--date", "{{job.parameters.target_date}}"]
    # 또는 spark_python_task로 .py 파일 직접 실행
    spark_python_task:
      python_file: "/Workspace/scripts/validate.py"
      parameters: ["--date", "{{job.parameters.target_date}}"]
```

---

## SQL 태스크

SQL 쿼리를 SQL Warehouse에서 실행합니다. 데이터 검증, 집계 테이블 갱신, 리포트 생성에 적합합니다.

```yaml
tasks:
  - task_key: "validate_results"
    sql_task:
      query:
        query_text: |
          SELECT
            COUNT(*) AS total_rows,
            COUNT(CASE WHEN amount < 0 THEN 1 END) AS negative_amounts
          FROM gold.daily_revenue
          WHERE sale_date = CURRENT_DATE()
      warehouse_id: "warehouse-id-here"
```

| SQL 태스크 옵션 | 설명 |
|----------------|------|
| `query.query_text` | 인라인 SQL 쿼리를 직접 지정합니다 |
| `file.path` | SQL 파일 경로를 지정합니다 |
| `dashboard.dashboard_id` | 대시보드를 새로고침합니다 |
| `alert.alert_id` | SQL 알림을 실행합니다 |

---

## dbt 태스크

dbt Core 프로젝트를 Databricks SQL Warehouse에서 실행합니다.

```yaml
tasks:
  - task_key: "build_gold_models"
    dbt_task:
      project_directory: "/Workspace/dbt/ecommerce"
      commands:
        - "dbt run --select gold_models"
        - "dbt test --select gold_models"
      warehouse_id: "warehouse-id-here"
      catalog: "production"
      schema: "gold"
```

> 💡 **dbt 태스크의 장점**: dbt 프로젝트를 Databricks 워크플로에 네이티브로 통합할 수 있습니다. dbt Cloud 없이도 스케줄링, 모니터링, 재시도가 가능합니다.

---

## Pipeline (SDP) 태스크

SDP 파이프라인의 업데이트를 트리거합니다. 파이프라인 ID만 지정하면 됩니다.

```yaml
tasks:
  - task_key: "ingest_raw_data"
    pipeline_task:
      pipeline_id: "abc-123-def-456"
      full_refresh: false  # true로 설정하면 전체 재처리
```

| 옵션 | 설명 |
|------|------|
| `pipeline_id` | 실행할 SDP 파이프라인 ID입니다 |
| `full_refresh` | `true`이면 전체 데이터를 재처리합니다 |

---

## JAR 태스크

Java/Scala로 작성된 JAR 파일을 실행합니다.

```yaml
tasks:
  - task_key: "java_etl"
    spark_jar_task:
      main_class_name: "com.company.etl.MainJob"
      parameters: ["--config", "production"]
    libraries:
      - jar: "dbfs:/jars/etl-job-1.0.jar"
    new_cluster:
      spark_version: "15.4.x-scala2.12"
      num_workers: 4
```

---

## Spark Submit 태스크

`spark-submit` 명령을 직접 실행합니다. 기존 Spark 애플리케이션을 그대로 사용할 수 있습니다.

```yaml
tasks:
  - task_key: "legacy_spark_app"
    spark_submit_task:
      parameters:
        - "--class"
        - "com.company.LegacyApp"
        - "dbfs:/jars/legacy-app.jar"
        - "--input"
        - "s3://bucket/data/"
```

> ⚠️ **Spark Submit은 레거시 호환을 위한 것입니다.** 새로운 워크로드에는 Notebook 또는 Python Script 태스크를 권장합니다.

---

## Run Job 태스크

다른 Job을 트리거하여 크로스 팀/프로젝트 오케스트레이션을 구현합니다.

```yaml
tasks:
  - task_key: "trigger_downstream"
    run_job_task:
      job_id: 987654321
      job_parameters:
        source_table: "gold.daily_revenue"
        execution_date: "{{job.parameters.target_date}}"
```

| 설정 | 설명 |
|------|------|
| `job_id` | 트리거할 대상 Job의 ID입니다 |
| `job_parameters` | 대상 Job에 전달할 파라미터입니다 |

---

## 현업 사례: 태스크 유형 잘못 선택해서 3배 비용을 지불한 경험

> 🔥 **실전 경험담**
>
> 한 고객사의 데이터 엔지니어가 **단순 SQL 집계 작업(일별 매출 요약 테이블 생성)을 Notebook 태스크로 구현** 했습니다. 노트북에서 PySpark로 `GROUP BY` 집계를 수행한 후 Delta 테이블에 저장하는 간단한 로직이었습니다. 문제는 이 Notebook 태스크가 **매번 Job Cluster를 새로 생성** 한다는 점이었습니다.
>
> - **Notebook 태스크 비용**: Job Cluster 시작(약 5분) + 실행(약 3분) + 종료 = **총 8분, 4 Worker 노드 사용**
> - **SQL 태스크로 전환 후**: 이미 실행 중인 SQL Warehouse에서 즉시 실행, **약 30초 만에 완료**
>
> 같은 결과를 내는 작업인데, **Notebook 태스크는 Job Cluster 시작 시간과 노드 비용 때문에 SQL 태스크 대비 약 3배의 비용** 이 발생했습니다. 이 고객사에는 비슷한 패턴의 태스크가 50개 이상 있었고, 전체를 SQL 태스크로 전환한 후 **월 컴퓨트 비용이 약 $2,000 절감** 되었습니다.

---

## 각 유형별 실전 비용 비교

아래는 "일별 매출 집계(100만 행 → 집계 결과 1,000행)"라는 동일한 작업을 각 태스크 유형으로 실행했을 때의 비용 비교입니다.

| 태스크 유형 | 클러스터 시작 시간 | 실행 시간 | 총 소요 시간 | 상대 비용 | 적합도 |
|------------|:----------------:|:--------:|:-----------:|:---------:|:------:|
| **SQL**| 0초 (Warehouse 상시 대기) | 30초 | **30초**| **1x**(기준) | ★★★★★ |
| **Notebook (Serverless)**| 30초~1분 | 1분 | **~2분**| **2x**| ★★★☆☆ |
| **Notebook (Job Cluster)**| 3~5분 | 1분 | **~6분**| **3x**| ★★☆☆☆ |
| **Python Script**| 3~5분 | 1분 | **~6분**| **3x**| ★★☆☆☆ |
| **Pipeline (SDP)**| 2~4분 | 1분 | **~5분**| **2.5x**| ★★☆☆☆ |

> 💡 **현업 팁**: SQL로 해결 가능한 작업은 무조건 SQL 태스크를 사용하세요. SQL Warehouse가 이미 실행 중이라면 추가 비용이 거의 없습니다. **"이 작업이 꼭 PySpark여야 하는가?"를 항상 먼저 질문하세요.**

---

## Notebook vs Python Script: 언제 무엇을 쓸까?

현업에서 가장 많이 받는 질문 중 하나입니다. 두 태스크 유형 모두 Python 코드를 실행하지만, 용도가 다릅니다.

| 비교 항목 | Notebook 태스크 | Python Script 태스크 |
|-----------|----------------|---------------------|
| **코드 위치**| Workspace 내 노트북 | Git 리포지토리의 `.py` 파일 |
| **디버깅**| 셀 단위 실행, 중간 결과 확인 가능 | 전체 스크립트 한 번에 실행 |
| **CI/CD**| Git 연동 가능하지만 노트북 형태 유지 | 표준 Python 프로젝트 구조 |
| **단위 테스트**| 어려움 (노트북 특성상) | 쉬움 (pytest 등 표준 도구 사용) |
| **파라미터 전달**| `dbutils.widgets` 사용 | `argparse`, `sys.argv` 사용 |
| **팀 협업**| 노트북 내 주석/마크다운 활용 | 코드 리뷰(PR) 친화적 |
| **라이브러리 관리**| 클러스터에 설치 | `requirements.txt` + Wheel 패키징 |

### 선택 기준 의사결정 트리

**질문: "이 코드를 어떻게 개발하고 관리할 것인가?"**

| 상황 | 권장 태스크 유형 | 예시 |
|------|----------------|------|
| 빠르게 프로토타이핑, 데이터 탐색하면서 개발 | **Notebook 태스크**| 새로운 ETL 로직 실험, 데이터 분석 |
| CI/CD 파이프라인에서 자동 배포, 테스트 코드 포함 | **Python Script (Wheel) 태스크**| 프로덕션 ETL, ML 파이프라인 |
| SQL만으로 충분한 변환/집계 작업 | **SQL 태스크**| 집계 테이블 갱신, 데이터 검증 |
| 기존 dbt 프로젝트가 있음 | **dbt 태스크**| dbt 모델 빌드 |

> 🔥 **현업에서는**: **프로토타이핑은 Notebook → 안정화되면 Python Script로 전환** 하는 패턴이 가장 많습니다. 처음부터 Python Script로 시작하면 개발 속도가 느리고, 끝까지 Notebook으로 가면 테스트와 코드 리뷰가 어렵습니다. 실제로 성숙한 조직에서는 프로덕션 Job의 70% 이상을 Python Wheel 태스크로 운영합니다.

---

## 태스크 유형 선택 가이드

| 시나리오 | 권장 태스크 유형 | 이유 |
|----------|---------------|------|
| Spark DataFrame 변환 | **Notebook**| 대화형 개발 + Spark 지원 |
| 단순 SQL 집계/검증 | **SQL**| SQL Warehouse에서 직접 실행 |
| dbt 모델 빌드 | **dbt**| 네이티브 dbt 통합 |
| 스트리밍 데이터 수집 | **Pipeline (SDP)**| SDP의 증분 처리 활용 |
| CI/CD 파이프라인 연동 | **Python Script**| Git 기반 코드 관리 |
| 다른 팀 Job 호출 | **Run Job**| 독립적인 Job 연결 |
| 조건부 분기 | **If/Else**| DAG 내 동적 흐름 제어 |
| 반복 처리 | **For Each** | 리스트 기반 병렬 실행 |

### 자주 하는 실수와 해결 방법

| 실수 | 결과 | 해결 |
|------|------|------|
| SQL로 충분한 작업을 Notebook으로 수행 | 불필요한 Job Cluster 비용 | SQL 태스크로 전환 |
| 모든 태스크에 별도 Job Cluster 할당 | 클러스터 시작 시간 누적 | 공유 Job Cluster 사용 |
| For Each 내부에 무거운 태스크 배치 | 병렬 실행으로 동시 클러스터 폭증 | `max_concurrency` 제한 설정 |
| Run Job으로 순차 실행 구현 | 불필요한 Job 간 오버헤드 | 하나의 Job 내 태스크 의존성 사용 |
| Pipeline 태스크에 `full_refresh: true` 고정 | 매번 전체 재처리로 비용 폭증 | `false`로 설정, 필요시만 수동 전체 갱신 |

---

## 실습: 다양한 태스크 유형을 활용한 워크플로

```yaml
tasks:
  # 1. SDP로 원시 데이터 수집
  - task_key: "ingest"
    pipeline_task:
      pipeline_id: "pipeline-abc-123"

  # 2. Notebook으로 데이터 변환
  - task_key: "transform"
    depends_on: [{task_key: "ingest"}]
    notebook_task:
      notebook_path: "/Workspace/etl/transform"

  # 3. SQL로 데이터 검증
  - task_key: "validate"
    depends_on: [{task_key: "transform"}]
    sql_task:
      query:
        query_text: "SELECT COUNT(*) AS cnt FROM gold.orders WHERE order_date = CURRENT_DATE()"
      warehouse_id: "wh-123"

  # 4. dbt로 Gold 모델 빌드
  - task_key: "build_gold"
    depends_on: [{task_key: "validate"}]
    dbt_task:
      project_directory: "/Workspace/dbt/project"
      commands: ["dbt run --select gold"]
      warehouse_id: "wh-123"

  # 5. 다운스트림 팀 Job 트리거
  - task_key: "notify_downstream"
    depends_on: [{task_key: "build_gold"}]
    run_job_task:
      job_id: 999888777
```

---

## 정리

| 핵심 포인트 | 설명 |
|------------|------|
| **10가지 태스크 유형**| 다양한 워크로드를 하나의 워크플로에서 통합 관리할 수 있습니다 |
| **Notebook**| 가장 범용적이며, Spark + Python/SQL/Scala 모두 지원합니다 |
| **SQL**| SQL Warehouse에서 직접 실행되어 가볍고 빠릅니다 |
| **Pipeline**| SDP 파이프라인을 워크플로의 한 단계로 통합합니다 |
| **If/Else, For Each** | DAG 내에서 동적 흐름 제어를 가능하게 합니다 |

---

## 참고 링크

- [Databricks: Create and run jobs](https://docs.databricks.com/aws/en/jobs/create-run-jobs.html)
- [Databricks: Task types](https://docs.databricks.com/aws/en/jobs/task-types.html)
- [Databricks: dbt task](https://docs.databricks.com/aws/en/jobs/dbt.html)
- [Azure Databricks: Jobs](https://learn.microsoft.com/en-us/azure/databricks/jobs/)
