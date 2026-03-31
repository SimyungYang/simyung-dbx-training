# Lakeflow Jobs 태스크 유형 상세

## 왜 태스크 유형을 이해해야 하는가?

Lakeflow Jobs는 다양한 워크로드를 하나의 워크플로로 통합합니다. **Notebook, SQL, Python Script, dbt, SDP 파이프라인** 등 10가지 태스크 유형을 제공하며, 각 유형은 특정 용도에 최적화되어 있습니다. 올바른 태스크 유형을 선택하면 개발 생산성과 운영 효율성이 크게 향상됩니다.

---

## 10가지 태스크 유형 총정리

| 태스크 유형 | 설명 | 컴퓨트 | 대표 사용 사례 |
|------------|------|--------|-------------|
| **Notebook** | Databricks 노트북을 실행합니다 | Job Cluster / Serverless | ETL 변환, ML 학습 |
| **Python Script** | `.py` 파일을 실행합니다 | Job Cluster / Serverless | 범용 Python 처리 |
| **SQL** | SQL 쿼리/파일을 실행합니다 | SQL Warehouse | 데이터 검증, 집계 |
| **dbt** | dbt 프로젝트 태스크를 실행합니다 | SQL Warehouse | dbt 모델 빌드 |
| **Pipeline (SDP)** | SDP 파이프라인을 트리거합니다 | 파이프라인 자체 클러스터 | 스트리밍 수집 |
| **JAR** | Java/Scala JAR을 실행합니다 | Job Cluster | Java 기반 ETL |
| **Spark Submit** | spark-submit으로 실행합니다 | Job Cluster | 레거시 Spark 앱 |
| **Run Job** | 다른 Job을 트리거합니다 | 대상 Job의 컴퓨트 | 크로스 팀 오케스트레이션 |
| **If/Else** | 조건에 따라 분기합니다 | 없음 (제어 흐름만) | 조건부 처리 |
| **For Each** | 리스트 항목별 반복 실행합니다 | 없음 (제어 흐름만) | 멀티 테넌트 처리 |

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

`.py` 파일을 직접 실행합니다. Notebook 태스크와 달리 **순수 Python 스크립트**를 실행하므로, CI/CD 파이프라인과의 연동이 쉽습니다.

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

## 태스크 유형 선택 가이드

| 시나리오 | 권장 태스크 유형 | 이유 |
|----------|---------------|------|
| Spark DataFrame 변환 | **Notebook** | 대화형 개발 + Spark 지원 |
| 단순 SQL 집계/검증 | **SQL** | SQL Warehouse에서 직접 실행 |
| dbt 모델 빌드 | **dbt** | 네이티브 dbt 통합 |
| 스트리밍 데이터 수집 | **Pipeline (SDP)** | SDP의 증분 처리 활용 |
| CI/CD 파이프라인 연동 | **Python Script** | Git 기반 코드 관리 |
| 다른 팀 Job 호출 | **Run Job** | 독립적인 Job 연결 |
| 조건부 분기 | **If/Else** | DAG 내 동적 흐름 제어 |
| 반복 처리 | **For Each** | 리스트 기반 병렬 실행 |

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
| **10가지 태스크 유형** | 다양한 워크로드를 하나의 워크플로에서 통합 관리할 수 있습니다 |
| **Notebook** | 가장 범용적이며, Spark + Python/SQL/Scala 모두 지원합니다 |
| **SQL** | SQL Warehouse에서 직접 실행되어 가볍고 빠릅니다 |
| **Pipeline** | SDP 파이프라인을 워크플로의 한 단계로 통합합니다 |
| **If/Else, For Each** | DAG 내에서 동적 흐름 제어를 가능하게 합니다 |

---

## 참고 링크

- [Databricks: Create and run jobs](https://docs.databricks.com/aws/en/jobs/create-run-jobs.html)
- [Databricks: Task types](https://docs.databricks.com/aws/en/jobs/task-types.html)
- [Databricks: dbt task](https://docs.databricks.com/aws/en/jobs/dbt.html)
- [Azure Databricks: Jobs](https://learn.microsoft.com/en-us/azure/databricks/jobs/)
