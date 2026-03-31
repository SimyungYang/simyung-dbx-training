# 조건부 태스크와 흐름 제어

## 왜 흐름 제어가 필요한가?

실제 데이터 파이프라인은 단순한 일직선이 아닙니다. "데이터 검증에 성공하면 Gold 테이블을 갱신하고, 실패하면 알림을 보내라", "10개 테넌트에 대해 같은 처리를 반복하라" 같은 **동적 흐름 제어**가 필요합니다. Lakeflow Jobs의 **If/Else**, **For Each** 태스크, 그리고 **Task Values**를 활용하면 이러한 복잡한 워크플로를 구현할 수 있습니다.

---

## If/Else 태스크 (조건부 분기)

### 개념

If/Else 태스크는 **조건 표현식의 결과에 따라 다른 태스크 분기를 실행**합니다. 별도의 컴퓨트 리소스를 사용하지 않으며, DAG의 흐름을 제어하는 역할만 합니다.

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 데이터 수집 | 데이터를 수집합니다 |
| 2 | 데이터 검증 | 수집된 데이터를 검증합니다 |
| 3 | If/Else 분기 | `row_count > 0` 조건을 평가합니다 |
| 4a | True → Gold 테이블 갱신 | 데이터가 있으면 Gold 테이블을 갱신하고 성공 알림을 발송합니다 |
| 4b | False → 알림 발송 | 데이터가 없으면 경고 알림을 발송합니다 |

### 조건 표현식

If/Else 태스크에서 사용할 수 있는 조건 표현식은 다음과 같습니다.

| 표현식 유형 | 예시 | 설명 |
|------------|------|------|
| **Task Value 비교** | `{{tasks.validate.values.row_count}} > 0` | 선행 태스크의 값과 비교합니다 |
| **문자열 비교** | `{{tasks.validate.values.status}} == "PASS"` | 문자열 일치 여부를 확인합니다 |
| **Job 파라미터** | `{{job.parameters.mode}} == "full"` | Job 레벨 파라미터를 참조합니다 |
| **복합 조건** | `{{tasks.t1.values.count}} > 100 AND {{job.parameters.env}} == "prod"` | AND/OR로 조합합니다 |

### YAML 설정 예제

```yaml
tasks:
  # 1. 데이터 검증 태스크
  - task_key: "validate"
    notebook_task:
      notebook_path: "/Workspace/etl/validate"

  # 2. 조건 분기 태스크
  - task_key: "check_quality"
    depends_on:
      - task_key: "validate"
    condition_task:
      op: "GREATER_THAN"
      left: "{{tasks.validate.values.pass_rate}}"
      right: "95"

  # 3. 조건이 True일 때 실행
  - task_key: "build_gold"
    depends_on:
      - task_key: "check_quality"
        outcome: "true"
    notebook_task:
      notebook_path: "/Workspace/etl/build_gold"

  # 4. 조건이 False일 때 실행
  - task_key: "send_alert"
    depends_on:
      - task_key: "check_quality"
        outcome: "false"
    notebook_task:
      notebook_path: "/Workspace/etl/send_alert"
```

### 지원되는 연산자

| 연산자 | 설명 | 예시 |
|--------|------|------|
| `EQUAL_TO` | 같음 | `"status" == "PASS"` |
| `NOT_EQUAL` | 같지 않음 | `"status" != "FAIL"` |
| `GREATER_THAN` | 초과 | `count > 100` |
| `GREATER_THAN_OR_EQUAL` | 이상 | `count >= 100` |
| `LESS_THAN` | 미만 | `error_rate < 5` |
| `LESS_THAN_OR_EQUAL` | 이하 | `error_rate <= 5` |

---

## For Each 태스크 (반복 실행)

### 개념

For Each 태스크는 **입력 리스트의 각 항목에 대해 중첩된 태스크를 반복 실행**합니다. 멀티 테넌트 처리, 파티션별 처리, 여러 테이블에 대한 동일 작업 등에 활용됩니다.

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 테넌트 목록 생성 | `[tenant_A, tenant_B, tenant_C]` 목록을 생성합니다 |
| 2 | For Each 반복 | 각 테넌트에 대해 병렬로 처리를 실행합니다 |
| 3 | 개별 처리 | `process(tenant_A)`, `process(tenant_B)`, `process(tenant_C)` 가 병렬 실행됩니다 |
| 4 | 완료 | 모든 테넌트 처리가 완료됩니다 |

### YAML 설정 예제

```yaml
tasks:
  # 1. 처리할 항목 리스트 생성
  - task_key: "generate_list"
    notebook_task:
      notebook_path: "/Workspace/etl/generate_tenant_list"

  # 2. For Each로 반복 실행
  - task_key: "process_tenants"
    depends_on:
      - task_key: "generate_list"
    for_each_task:
      inputs: "{{tasks.generate_list.values.tenant_list}}"
      concurrency: 5  # 최대 동시 실행 수
      task:
        task_key: "process_single_tenant"
        notebook_task:
          notebook_path: "/Workspace/etl/process_tenant"
          base_parameters:
            tenant_id: "{{input}}"
```

### 입력 리스트 생성 노트북

```python
# generate_tenant_list 노트북
import json

# 처리할 테넌트 목록 조회
tenants = spark.sql("""
    SELECT DISTINCT tenant_id
    FROM catalog.schema.tenant_config
    WHERE is_active = true
""").collect()

# Task Value로 리스트 전달
tenant_list = json.dumps([row.tenant_id for row in tenants])
dbutils.jobs.taskValues.set(key="tenant_list", value=tenant_list)
```

### For Each 태스크 설정 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `inputs` | 반복할 입력 리스트 (JSON 배열 문자열)입니다 | (필수) |
| `concurrency` | 동시에 실행할 최대 태스크 수입니다 | 1 |
| `task` | 각 반복에서 실행할 태스크 정의입니다 | (필수) |

> 💡 **concurrency 설정**: 리소스에 여유가 있다면 `concurrency`를 높여 병렬 처리 속도를 높일 수 있습니다. 단, 하류 시스템의 부하를 고려하여 적절한 값을 설정하세요.

---

## Task Values (태스크 간 값 전달)

### 개념

Task Values는 선행 태스크에서 계산한 결과를 후행 태스크로 전달하는 메커니즘입니다. `dbutils.jobs.taskValues`를 사용합니다.

### 값 설정 (선행 태스크)

```python
# Task A: 값 설정
row_count = spark.sql("SELECT COUNT(*) FROM silver.orders").collect()[0][0]
max_date = spark.sql("SELECT MAX(order_date) FROM silver.orders").collect()[0][0]
pass_rate = 99.5

# 다양한 타입의 값을 설정할 수 있습니다
dbutils.jobs.taskValues.set(key="row_count", value=row_count)          # 정수
dbutils.jobs.taskValues.set(key="max_date", value=str(max_date))       # 문자열
dbutils.jobs.taskValues.set(key="pass_rate", value=pass_rate)          # 실수
dbutils.jobs.taskValues.set(key="status", value="PASS")                # 문자열
dbutils.jobs.taskValues.set(key="tables", value='["orders","items"]')  # JSON 문자열
```

### 값 읽기 (후행 태스크)

```python
# Task B: Task A의 값 읽기
row_count = dbutils.jobs.taskValues.get(
    taskKey="task_a",
    key="row_count",
    default=0  # 값이 없을 때 기본값
)
max_date = dbutils.jobs.taskValues.get(
    taskKey="task_a",
    key="max_date",
    default="1970-01-01"
)
status = dbutils.jobs.taskValues.get(
    taskKey="task_a",
    key="status",
    default="UNKNOWN"
)

print(f"처리된 행 수: {row_count}")
print(f"최대 날짜: {max_date}")
print(f"상태: {status}")
```

### YAML에서 Task Values 참조

```yaml
tasks:
  - task_key: "downstream"
    depends_on:
      - task_key: "upstream"
    notebook_task:
      notebook_path: "/Workspace/etl/downstream"
      base_parameters:
        # Task Values를 파라미터로 전달
        row_count: "{{tasks.upstream.values.row_count}}"
        max_date: "{{tasks.upstream.values.max_date}}"
```

---

## DAG 의존성 패턴

### 1. 선형 패턴 (Sequential)

```yaml
tasks:
  - task_key: "ingest"
    notebook_task: {notebook_path: "/etl/ingest"}
  - task_key: "transform"
    depends_on: [{task_key: "ingest"}]
    notebook_task: {notebook_path: "/etl/transform"}
  - task_key: "validate"
    depends_on: [{task_key: "transform"}]
    notebook_task: {notebook_path: "/etl/validate"}
```

### 2. 팬아웃/팬인 패턴 (Fan-out/Fan-in)

```yaml
tasks:
  - task_key: "ingest_orders"
    notebook_task: {notebook_path: "/etl/ingest_orders"}
  - task_key: "ingest_customers"
    notebook_task: {notebook_path: "/etl/ingest_customers"}
  - task_key: "ingest_products"
    notebook_task: {notebook_path: "/etl/ingest_products"}
  # 팬인: 세 태스크가 모두 완료되어야 실행
  - task_key: "join_all"
    depends_on:
      - {task_key: "ingest_orders"}
      - {task_key: "ingest_customers"}
      - {task_key: "ingest_products"}
    notebook_task: {notebook_path: "/etl/join_all"}
```

### 3. 조건부 패턴 (Conditional)

```yaml
tasks:
  - task_key: "validate"
    notebook_task: {notebook_path: "/etl/validate"}
  - task_key: "check_result"
    depends_on: [{task_key: "validate"}]
    condition_task:
      op: "EQUAL_TO"
      left: "{{tasks.validate.values.status}}"
      right: "PASS"
  - task_key: "on_success"
    depends_on: [{task_key: "check_result", outcome: "true"}]
    notebook_task: {notebook_path: "/etl/build_gold"}
  - task_key: "on_failure"
    depends_on: [{task_key: "check_result", outcome: "false"}]
    notebook_task: {notebook_path: "/etl/send_alert"}
```

---

## 실전 예제: 멀티 테넌트 ETL 워크플로

```yaml
tasks:
  # 1. 활성 테넌트 목록 조회
  - task_key: "get_tenants"
    notebook_task:
      notebook_path: "/Workspace/etl/get_active_tenants"

  # 2. 테넌트별 데이터 처리 (병렬)
  - task_key: "process_all_tenants"
    depends_on: [{task_key: "get_tenants"}]
    for_each_task:
      inputs: "{{tasks.get_tenants.values.tenant_list}}"
      concurrency: 10
      task:
        task_key: "process_tenant"
        notebook_task:
          notebook_path: "/Workspace/etl/process_tenant"
          base_parameters:
            tenant_id: "{{input}}"

  # 3. 결과 검증
  - task_key: "validate_all"
    depends_on: [{task_key: "process_all_tenants"}]
    notebook_task:
      notebook_path: "/Workspace/etl/validate_results"

  # 4. 조건부 후처리
  - task_key: "check_quality"
    depends_on: [{task_key: "validate_all"}]
    condition_task:
      op: "GREATER_THAN_OR_EQUAL"
      left: "{{tasks.validate_all.values.pass_rate}}"
      right: "99"

  - task_key: "publish_report"
    depends_on: [{task_key: "check_quality", outcome: "true"}]
    sql_task:
      query:
        query_text: "REFRESH MATERIALIZED VIEW gold.tenant_summary"
      warehouse_id: "wh-123"
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **If/Else 태스크** | 조건 표현식에 따라 DAG 내에서 분기를 수행합니다 |
| **For Each 태스크** | 입력 리스트의 각 항목에 대해 태스크를 반복 실행합니다 |
| **Task Values** | `dbutils.jobs.taskValues`로 태스크 간 데이터를 전달합니다 |
| **의존성 패턴** | 선형, 팬아웃/팬인, 조건부 패턴을 조합하여 복잡한 DAG를 구성합니다 |
| **concurrency** | For Each의 동시 실행 수를 제어하여 리소스를 관리합니다 |

---

## 참고 링크

- [Databricks: Conditional tasks](https://docs.databricks.com/aws/en/jobs/conditional-tasks.html)
- [Databricks: For each tasks](https://docs.databricks.com/aws/en/jobs/for-each-tasks.html)
- [Databricks: Task values](https://docs.databricks.com/aws/en/jobs/task-values.html)
- [Databricks: Job parameters](https://docs.databricks.com/aws/en/jobs/parameters.html)
