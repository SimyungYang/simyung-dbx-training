# 태스크 유형과 의존성

## 왜 태스크 의존성 관리가 중요한가?

### 단일 노트북 실행의 한계

초기 데이터 파이프라인은 흔히 하나의 노트북에 수집 → 변환 → 적재 → 알림 로직을 모두 몰아넣는 방식으로 작성됩니다. 이 접근 방식은 다음과 같은 문제를 초래합니다.

| 문제 | 설명 |
|------|------|
| **재실행 비효율** | 적재 단계에서 실패해도 수집부터 전부 다시 실행해야 합니다 |
| **컴퓨트 낭비** | 병렬화가 불가능하여 모든 단계가 직렬로 실행됩니다 |
| **가시성 부재** | 어느 단계에서 얼마나 걸렸는지 파악하기 어렵습니다 |
| **유지보수 어려움** | 단계별 독립 배포, 테스트, 재시도가 불가능합니다 |

### DAG 기반 오케스트레이션의 필요성

**DAG(Directed Acyclic Graph)** 기반 오케스트레이션은 각 단계를 독립 태스크로 분리하고, 태스크 간 의존 관계를 방향성 있는 그래프로 정의합니다. Databricks LakeFlow Jobs는 이 DAG를 시각적으로 편집하고, 실행 상태를 실시간으로 모니터링할 수 있습니다.

핵심 이점:
- **부분 재실행**: 실패한 태스크부터 재시작 가능합니다
- **병렬 실행**: 의존성 없는 태스크는 동시에 실행됩니다
- **세밀한 컴퓨트 제어**: 태스크별로 클러스터 크기를 다르게 설정할 수 있습니다
- **조건부 분기**: 이전 태스크 결과에 따라 실행 경로를 동적으로 결정합니다

> 참고: [Databricks Jobs 공식 문서](https://docs.databricks.com/en/jobs/index.html)

---

## 태스크 유형

Databricks Job은 다양한 유형의 태스크를 지원합니다. 각 태스크 유형은 특정 워크로드에 최적화되어 있습니다.

### 태스크 유형 요약

| 태스크 유형 | 설명 | 대표 사용 사례 |
|-------------|------|----------------|
| **Notebook** | Databricks 노트북을 실행합니다 | ETL 로직, 데이터 변환, ML 학습 |
| **Python Script** | `.py` 파일을 직접 실행합니다 | 범용 Python 스크립트, 배치 처리 |
| **SQL** | SQL 쿼리 또는 SQL 파일을 실행합니다 | 데이터 검증, 집계 테이블 갱신 |
| **dbt** | dbt 프로젝트의 태스크를 실행합니다 | dbt 모델 빌드, 테스트 |
| **Spark Submit** | Spark JAR 또는 Python을 spark-submit으로 실행합니다 | 레거시 Spark 애플리케이션 |
| **Pipeline (DLT)** | Delta Live Tables 파이프라인을 트리거합니다 | 스트리밍 수집, Medallion 파이프라인 |
| **JAR** | Java/Scala JAR 파일을 실행합니다 | Java 기반 ETL, 사내 라이브러리 |
| **Run Job** | 다른 Job을 트리거합니다 | 크로스 팀 워크플로 오케스트레이션 |
| **If/Else 조건** | 조건에 따라 분기 실행합니다 | 데이터 품질 기반 분기 처리 |
| **For Each** | 리스트의 각 항목에 대해 태스크를 반복 실행합니다 | 멀티 테넌트 처리, 파티션별 처리 |

### 주요 태스크 유형 상세

**Notebook 태스크** 는 가장 범용적인 유형입니다. Python, SQL, Scala 노트북을 모두 지원하며, `dbutils.widgets`로 파라미터를 전달받을 수 있습니다. 개발 및 프로토타이핑 단계에서 가장 많이 활용됩니다.

**Python Script 태스크** 는 Git 또는 Workspace에 저장된 `.py` 파일을 실행합니다. 노트북보다 버전 관리가 명확하고, CI/CD 파이프라인과 통합하기 용이합니다. 프로덕션 환경에 권장됩니다.

**SQL 태스크** 는 SQL Warehouse에서 실행되며, Unity Catalog 테이블에 대한 DDL/DML 작업이나 데이터 품질 검사에 적합합니다. Serverless SQL Warehouse와 함께 사용하면 비용 효율이 높습니다.

**Pipeline (DLT) 태스크** 는 Delta Live Tables 파이프라인을 Job 내에서 트리거합니다. 수집 완료 후 변환 파이프라인을 순차 실행하는 패턴에 유용합니다.

**dbt 태스크** 는 dbt Cloud 또는 dbt Core 프로젝트와 통합됩니다. `dbt run`, `dbt test`, `dbt snapshot` 명령을 Job 태스크로 실행할 수 있습니다.

> 참고: [태스크 유형별 공식 문서](https://docs.databricks.com/en/workflows/jobs/create-run-jobs.html#add-tasks-to-a-job)

---

## 의존성 설정

태스크 간 의존성을 설정하면 **DAG(Directed Acyclic Graph)** 형태로 실행 순서가 결정됩니다. Databricks는 세 가지 핵심 의존성 패턴을 지원합니다.

### 의존성 패턴

| 패턴 | 설명 | 사용 사례 |
|------|------|-----------|
| **선형 (Sequential)** | A → B → C 순서대로 실행합니다 | 단계별 ETL 파이프라인 |
| **팬아웃/팬인 (Fan-out/Fan-in)** | 여러 태스크를 동시에 실행한 후 합류합니다 | 독립적인 데이터 소스 동시 수집 |
| **조건부 (Conditional)** | 선행 태스크 결과에 따라 분기합니다 | 데이터 검증 결과에 따른 처리 |

### 복잡한 태스크 DAG 예제

```
ingest_orders ──┐
                ├── transform_orders ── validate_data ──┬── build_gold_tables ──┬── update_dashboard
ingest_products─┘                                       │                       └── notify_success
                                                        └── send_alert (실패 시)
```

| Task | 유형 | 의존성 | 설명 |
|------|------|--------|------|
| ingest_orders | Pipeline Task | - | 주문 데이터를 수집합니다 |
| ingest_products | Pipeline Task | - | 상품 데이터를 수집합니다 |
| transform_orders | Notebook Task | ingest_orders, ingest_products | 데이터를 변환합니다 |
| validate_data | SQL Task | transform_orders | 데이터 품질을 검증합니다 |
| build_gold_tables | dbt Task | validate_data (성공 시) | Gold 테이블을 생성합니다 |
| send_alert | Notebook Task | validate_data (실패 시) | 알림을 발송합니다 |
| update_dashboard | SQL Task | build_gold_tables | 대시보드를 갱신합니다 |
| notify_success | Notebook Task | build_gold_tables | 성공 알림을 발송합니다 |

위 DAG에서:
- `ingest_orders`와 `ingest_products`는 **병렬** 로 실행됩니다
- 두 수집 태스크가 모두 완료되면 `transform_orders`가 실행됩니다 (팬인)
- `validate_data` 결과에 따라 **조건부 분기** 가 발생합니다
- `build_gold_tables` 완료 후 대시보드 갱신과 알림이 **병렬** 로 실행됩니다 (팬아웃)

### If/Else 조건부 태스크

**If/Else 태스크** 는 이전 태스크의 결과값 또는 Job 파라미터를 기반으로 실행 경로를 동적으로 분기합니다. 예를 들어 데이터 검증 태스크가 반환한 행 수가 임계값 이하이면 경고 경로로, 초과하면 정상 경로로 분기할 수 있습니다.

```yaml
- task_key: "route_by_quality"
  condition_task:
    op: "GREATER_THAN"
    left: "{{tasks.validate_data.values.row_count}}"
    right: "0"

- task_key: "build_gold_tables"
  depends_on:
    - task_key: "route_by_quality"
      outcome: "true"
  dbt_task:
    project_directory: "/dbt"
    commands: ["dbt run --select gold.*"]

- task_key: "send_alert"
  depends_on:
    - task_key: "route_by_quality"
      outcome: "false"
  notebook_task:
    notebook_path: "/Workspace/utils/alert"
```

> 참고: [조건부 태스크 공식 문서](https://docs.databricks.com/en/workflows/jobs/conditional-tasks.html)

---

## 태스크 값 전달 (Task Values)

태스크 간 데이터를 전달하려면 **`dbutils.jobs.taskValues`** API를 사용합니다. 이 메커니즘으로 한 태스크의 출력을 후속 태스크의 입력으로 활용할 수 있습니다.

> 주의: 태스크 값의 최대 크기는 **48KB** 입니다. 대용량 데이터는 Delta 테이블 또는 외부 스토리지를 경유하십시오.

### 태스크 값 설정 (upstream 태스크)

```python
# validate_data 태스크 (노트북)에서 행 수를 downstream으로 전달합니다
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

row_count = spark.table("silver.orders").filter("sale_date = current_date()").count()
error_count = spark.table("silver.orders").filter("status = 'ERROR'").count()

# 태스크 값 설정
dbutils.jobs.taskValues.set(key="row_count", value=row_count)
dbutils.jobs.taskValues.set(key="error_count", value=error_count)
dbutils.jobs.taskValues.set(key="validation_status", value="passed" if error_count == 0 else "failed")

print(f"row_count={row_count}, error_count={error_count}")
```

### 태스크 값 읽기 (downstream 태스크)

```python
# build_gold_tables 태스크에서 upstream의 값을 읽습니다
row_count = dbutils.jobs.taskValues.get(
    taskKey="validate_data",   # upstream 태스크 key
    key="row_count",           # 읽을 값의 key
    default=0,                 # 값이 없을 경우 기본값
    debugValue=9999            # 노트북을 독립 실행할 때 사용할 디버그 값
)

validation_status = dbutils.jobs.taskValues.get(
    taskKey="validate_data",
    key="validation_status",
    default="unknown"
)

print(f"수신된 행 수: {row_count}, 검증 상태: {validation_status}")

if validation_status == "failed":
    raise Exception("Upstream validation failed. Aborting gold table build.")
```

### Asset Bundle YAML에서 태스크 값 참조

```yaml
- task_key: "notify_success"
  depends_on:
    - task_key: "build_gold_tables"
  notebook_task:
    notebook_path: "/Workspace/utils/notify"
    base_parameters:
      row_count: "{{tasks.validate_data.values.row_count}}"
      status: "{{tasks.validate_data.values.validation_status}}"
```

> 참고: [Task values 공식 문서](https://docs.databricks.com/en/workflows/jobs/share-task-context.html)

---

## 실패 처리

### 재시도 설정

각 태스크에 재시도 횟수와 대기 시간을 설정하면 일시적인 장애(네트워크 오류, 클러스터 시작 지연 등)를 자동으로 복구할 수 있습니다.

```yaml
- task_key: "ingest_orders"
  pipeline_task:
    pipeline_id: "${var.pipeline_id}"
  max_retries: 3                # 최대 3회 재시도합니다
  min_retry_interval_millis: 60000   # 재시도 전 1분 대기합니다
  retry_on_timeout: true        # 타임아웃 시에도 재시도합니다
  timeout_seconds: 3600         # 1시간 후 타임아웃 처리합니다
```

### 태스크 실패 시 전체 Job 동작

기본적으로 하나의 태스크가 실패하면 해당 태스크에 **의존하는 후속 태스크들은 실행되지 않습니다.** 그러나 의존성이 없는 병렬 태스크는 계속 실행됩니다.

`run_if` 속성으로 후속 태스크의 실행 조건을 명시적으로 제어할 수 있습니다:

| `run_if` 값 | 설명 |
|-------------|------|
| `ALL_SUCCESS` (기본값) | 모든 선행 태스크가 성공해야 실행됩니다 |
| `ALL_DONE` | 선행 태스크의 성공/실패 여부와 무관하게 실행됩니다 |
| `AT_LEAST_ONE_SUCCESS` | 선행 태스크 중 하나 이상이 성공하면 실행됩니다 |
| `ALL_FAILED` | 모든 선행 태스크가 실패해야 실행됩니다 |
| `AT_LEAST_ONE_FAILED` | 선행 태스크 중 하나 이상이 실패하면 실행됩니다 |

```yaml
# 실패 여부와 관계없이 항상 클린업을 실행하는 예시
- task_key: "cleanup"
  run_if: "ALL_DONE"
  depends_on:
    - task_key: "transform_orders"
  notebook_task:
    notebook_path: "/Workspace/utils/cleanup"
```

### 조건부 실행으로 에러 핸들링

```python
# transform_orders 태스크: 에러가 발생해도 빈 값을 설정하여 후속 태스크가 판단할 수 있게 합니다
try:
    df = spark.table("bronze.orders").transform(clean_orders)
    df.write.mode("overwrite").saveAsTable("silver.orders")
    dbutils.jobs.taskValues.set(key="transform_status", value="success")
    dbutils.jobs.taskValues.set(key="record_count", value=df.count())
except Exception as e:
    dbutils.jobs.taskValues.set(key="transform_status", value="failed")
    dbutils.jobs.taskValues.set(key="error_message", value=str(e))
    raise  # Job 레벨에서도 실패를 감지할 수 있도록 예외를 다시 던집니다
```

> 참고: [재시도 및 타임아웃 공식 문서](https://docs.databricks.com/en/workflows/jobs/job-task-settings.html#retry-policy)

---

## For Each 태스크

**For Each 태스크** 는 입력 배열의 각 요소에 대해 내부 태스크를 반복 실행합니다. 동일한 로직을 여러 테넌트, 여러 리전, 여러 파티션에 적용하는 패턴에 매우 유용합니다.

### 구조

- **For Each 태스크** (외부): 반복할 파라미터 배열을 정의합니다
- **내부 태스크 (nested task)**: 각 반복에서 실행할 태스크를 정의합니다
- **`concurrency`**: 동시 실행할 최대 태스크 수를 제어합니다

### Asset Bundle YAML 예시

```yaml
- task_key: "process_all_regions"
  for_each_task:
    inputs: '["us-east-1", "eu-west-1", "ap-northeast-2"]'
    concurrency: 3   # 3개 리전을 동시에 처리합니다
    task:
      task_key: "process_region"
      notebook_task:
        notebook_path: "/Workspace/etl/process_region"
        base_parameters:
          region: "{{input}}"   # 배열의 현재 요소가 주입됩니다
```

### 동적 배열 사용 (upstream 태스크 값 활용)

```python
# discover_tenants 태스크: 처리할 테넌트 목록을 동적으로 생성합니다
import json

tenants = spark.table("catalog.meta.active_tenants").select("tenant_id").collect()
tenant_list = [row.tenant_id for row in tenants]

dbutils.jobs.taskValues.set(key="tenant_list", value=json.dumps(tenant_list))
```

```yaml
- task_key: "process_tenants"
  depends_on:
    - task_key: "discover_tenants"
  for_each_task:
    inputs: "{{tasks.discover_tenants.values.tenant_list}}"
    concurrency: 5
    task:
      task_key: "process_single_tenant"
      python_wheel_task:
        package_name: "etl_pipeline"
        entry_point: "run_tenant"
        parameters: ["--tenant-id", "{{input}}"]
```

> 참고: [For Each 태스크 공식 문서](https://docs.databricks.com/en/workflows/jobs/for-each-task.html)

---

## Job 생성 방법

### UI에서 생성

1. 좌측 메뉴 **Workflows** → **Create Job** 클릭
2. Job 이름 입력
3. Task 추가 (노트북 경로, SQL 쿼리 등 지정)
4. Task 간 의존성 설정 (드래그&드롭 또는 `depends_on` 선택)
5. 컴퓨트 설정 (Serverless, Job Cluster, 또는 기존 All-Purpose Cluster)
6. 스케줄, 알림, 재시도 정책 설정

### Asset Bundles (IaC)

Infrastructure as Code 방식으로 Job을 정의하면 버전 관리와 환경별 배포가 쉬워집니다.

```yaml
resources:
  jobs:
    daily_pipeline:
      name: "daily-sales-pipeline"
      tags:
        team: "data-engineering"
        project: "sales"
        environment: "${bundle.target}"

      parameters:
        - name: "target_date"
          default: ""
        - name: "environment"
          default: "production"

      tasks:
        - task_key: "ingest_orders"
          pipeline_task:
            pipeline_id: "${var.pipeline_id}"
          max_retries: 2
          min_retry_interval_millis: 60000
          timeout_seconds: 1800

        - task_key: "ingest_products"
          pipeline_task:
            pipeline_id: "${var.products_pipeline_id}"
          max_retries: 2

        - task_key: "transform"
          depends_on:
            - task_key: "ingest_orders"
            - task_key: "ingest_products"
          notebook_task:
            notebook_path: "/Workspace/etl/transform"
            base_parameters:
              date: "{{job.parameters.target_date}}"
          new_cluster:
            spark_version: "15.4.x-scala2.12"
            num_workers: 4
            node_type_id: "i3.xlarge"
            aws_attributes:
              availability: "SPOT_WITH_FALLBACK"

        - task_key: "validate"
          depends_on:
            - task_key: "transform"
          sql_task:
            query:
              query_text: >
                SELECT COUNT(*) AS row_count
                FROM gold.daily_revenue
                WHERE sale_date = CURRENT_DATE()
            warehouse_id: "${var.warehouse_id}"

        - task_key: "notify"
          run_if: "ALL_DONE"
          depends_on:
            - task_key: "validate"
          notebook_task:
            notebook_path: "/Workspace/etl/send_notification"

      email_notifications:
        on_failure:
          - "data-team@company.com"
        on_success:
          - "data-team@company.com"

      webhook_notifications:
        on_failure:
          - id: "${var.slack_webhook_id}"

      schedule:
        quartz_cron_expression: "0 0 2 * * ?"
        timezone_id: "Asia/Seoul"
```

---

## 베스트 프랙티스

### 태스크 분리 기준

- **단일 책임 원칙**: 하나의 태스크는 하나의 명확한 역할만 수행합니다 (수집, 변환, 검증 등)
- **재실행 단위**: 실패 후 재실행하고 싶은 최소 단위로 태스크를 나눕니다
- **컴퓨트 요구사항**: 리소스 요구가 크게 다른 단계는 별도 태스크로 분리하여 클러스터를 개별 최적화합니다

### 의존성 최소화

- 실제로 필요한 의존성만 선언합니다. 불필요한 직렬화는 전체 Job 실행 시간을 늘립니다
- 논리적으로 독립적인 데이터 소스 수집은 항상 병렬 태스크로 설계합니다
- 팬인 지점(합류 태스크)은 반드시 필요한 경우에만 추가합니다

### 멱등성 설계

모든 태스크는 **멱등성(idempotency)** 을 갖도록 설계합니다. 같은 태스크를 여러 번 실행해도 결과가 동일해야 합니다.

```python
# 멱등성 보장: 덮어쓰기 모드 + 파티션 교체
spark.table("silver.orders") \
    .filter(f"sale_date = '{target_date}'") \
    .write \
    .mode("overwrite") \
    .option("replaceWhere", f"sale_date = '{target_date}'") \
    .saveAsTable("gold.daily_revenue")
```

---

## 흔한 실수

### 과도한 의존성

모든 태스크를 직렬로 연결하면 병렬화의 이점을 전혀 얻지 못합니다. 태스크 A의 출력이 태스크 B에 실제로 필요한지 먼저 검토합니다.

```
# 잘못된 패턴: 모든 태스크 직렬화
ingest_orders → ingest_products → transform → validate → build_gold

# 올바른 패턴: 독립 태스크 병렬화
ingest_orders ──┐
                ├── transform → validate → build_gold
ingest_products─┘
```

### 태스크 값 크기 제한 초과

`dbutils.jobs.taskValues`의 최대 크기는 **48KB** 입니다. DataFrame, 대형 리스트, 바이너리 데이터를 태스크 값으로 전달하지 마십시오.

```python
# 잘못된 패턴: 대용량 데이터를 태스크 값으로 전달
rows = spark.table("silver.orders").collect()
dbutils.jobs.taskValues.set(key="data", value=str(rows))  # 크기 초과 위험

# 올바른 패턴: Delta 테이블 경로 또는 메타데이터만 전달
dbutils.jobs.taskValues.set(key="output_table", value="silver.orders")
dbutils.jobs.taskValues.set(key="row_count", value=spark.table("silver.orders").count())
```

### 타임아웃 미설정

타임아웃을 설정하지 않으면 무한 대기 상태의 태스크가 클러스터를 점유합니다. 예상 실행 시간의 2-3배를 타임아웃으로 설정합니다.

```yaml
- task_key: "long_running_transform"
  timeout_seconds: 7200   # 예상 1시간 → 2시간 타임아웃 설정
  max_retries: 1
```

### 디버그 시 `debugValue` 미활용

노트북을 독립 실행할 때 `dbutils.jobs.taskValues.get`은 Job 컨텍스트가 없어 오류를 발생시킵니다. `debugValue` 파라미터를 반드시 설정합니다.

```python
# debugValue 없이 독립 실행 시 오류 발생
row_count = dbutils.jobs.taskValues.get(taskKey="validate", key="row_count")

# debugValue를 설정하면 독립 실행 시 해당 값을 사용합니다
row_count = dbutils.jobs.taskValues.get(
    taskKey="validate",
    key="row_count",
    default=0,
    debugValue=100   # 노트북 단독 실행 시 사용할 값
)
```

---

## 참고 링크

- [Databricks Jobs 공식 문서](https://docs.databricks.com/en/jobs/index.html)
- [태스크 유형 및 설정](https://docs.databricks.com/en/workflows/jobs/create-run-jobs.html)
- [Task values (태스크 간 데이터 전달)](https://docs.databricks.com/en/workflows/jobs/share-task-context.html)
- [조건부 태스크](https://docs.databricks.com/en/workflows/jobs/conditional-tasks.html)
- [For Each 태스크](https://docs.databricks.com/en/workflows/jobs/for-each-task.html)
- [재시도 및 타임아웃 설정](https://docs.databricks.com/en/workflows/jobs/job-task-settings.html#retry-policy)
- [Asset Bundles Job 리소스 스펙](https://docs.databricks.com/en/dev-tools/bundles/reference.html#jobs)
