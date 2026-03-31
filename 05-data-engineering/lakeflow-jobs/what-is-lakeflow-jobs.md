# Lakeflow Jobs란?

## 개념

> 💡 **Lakeflow Jobs (Workflows)** 는 데이터 파이프라인과 작업을 **스케줄링하고, 의존성을 관리하며, 모니터링**하는 오케스트레이션 서비스입니다. 여러 개의 작업(Task)을 DAG(Directed Acyclic Graph) 형태로 연결하여, 정해진 순서대로 또는 조건에 따라 실행합니다.

비유하자면, 오케스트라의 **지휘자**와 같습니다. 각 연주자(Task)가 적절한 순서와 타이밍에 연주하도록 조율하고, 문제가 생기면 즉시 대응합니다. 데이터 팀에서 매일 반복되는 ETL 파이프라인, 정기 보고서 생성, ML 모델 재학습 등의 작업을 자동화하고 안정적으로 운영하는 데 핵심적인 역할을 합니다.

> 💡 **DAG(Directed Acyclic Graph)란?** 방향성이 있고 순환하지 않는 그래프를 뜻합니다. 쉽게 말해, 작업의 실행 순서를 화살표로 연결한 흐름도입니다. A → B → C처럼 진행 방향이 있고, C → A로 되돌아가는 순환이 없는 구조입니다. 데이터 파이프라인의 작업 의존성을 표현하는 데 가장 적합한 구조입니다.

---

## Job의 구성 요소

**Job: daily-sales-pipeline 구조**

| Task | 유형 | 의존성 | 설명 |
|------|------|--------|------|
| Task 1: 데이터 수집 | Pipeline | - | 데이터를 수집합니다 |
| Task 2: 데이터 정제 | Notebook | Task 1 | 데이터를 정제합니다 |
| Task 3: 품질 검증 | SQL | Task 2 | 품질을 검증합니다 |
| Task 4: 집계 생성 | Notebook | Task 3 (성공 시) | 집계 테이블을 생성합니다 |
| Task 5: 대시보드 갱신 | SQL | Task 4 | 대시보드를 갱신합니다 |
| Task 6: 알림 발송 | Python | Task 3 (실패 시) / Task 4 | 알림을 발송합니다 |

| 구성 요소 | 설명 |
|-----------|------|
| **Job** | 하나 이상의 Task를 묶은 워크플로우 단위입니다. 스케줄, 알림, 권한 설정의 단위이기도 합니다. 하나의 Job은 하나의 완결된 비즈니스 프로세스를 나타냅니다 |
| **Task** | 실제 실행되는 개별 작업입니다. 다양한 유형(노트북, SQL, 파이프라인 등)을 지원하며, 각 Task는 독립적인 컴퓨트에서 실행될 수 있습니다 |
| **Dependency** | Task 간의 실행 순서를 정의합니다. 선행 Task 완료 후 후행 Task가 실행됩니다. 성공/실패에 따른 조건부 의존성도 설정할 수 있습니다 |
| **Trigger** | Job이 실행되는 조건입니다 (시간, 파일 도착, API 호출, 테이블 변경). 하나의 Job에 여러 트리거를 설정할 수도 있습니다 |
| **Run** | Job의 한 번의 실행 인스턴스입니다. 각 Run에는 고유한 run_id가 부여되며, 실행 이력과 로그가 보존됩니다 |

---

## Job 라이프사이클

Job은 생성부터 실행, 모니터링까지 명확한 라이프사이클을 갖고 있습니다. 각 단계를 이해하면 운영 중 발생하는 문제를 더 효과적으로 대응할 수 있습니다.

```
[생성] → [트리거 대기] → [실행 시작] → [Task 순차/병렬 실행] → [완료/실패] → [알림/후처리]
```

### Run의 상태 흐름

| 상태 | 설명 |
|------|------|
| **PENDING** | 실행이 큐에 들어가서 리소스 할당을 기다리는 상태입니다 |
| **RUNNING** | 클러스터가 할당되어 Task가 실행 중인 상태입니다 |
| **TERMINATING** | 마지막 Task가 완료되어 리소스를 정리하는 상태입니다 |
| **TERMINATED** | 모든 Task가 성공적으로 완료된 상태입니다 |
| **SKIPPED** | 조건부 실행에서 건너뛴 상태입니다 |
| **INTERNAL_ERROR** | 시스템 오류로 실행이 실패한 상태입니다 |

### 재시도 정책

Task 실행이 실패했을 때 자동으로 재시도하도록 설정할 수 있습니다. 일시적인 네트워크 오류나 리소스 부족으로 인한 실패를 자동 복구하는 데 유용합니다.

```json
{
  "retry_on_timeout": true,
  "max_retries": 3,
  "min_retry_interval_millis": 60000,
  "unlimited_retries": false
}
```

---

## Task 유형

Lakeflow Jobs는 다양한 유형의 Task를 지원하여, 이기종 워크로드를 하나의 워크플로우에서 관리할 수 있습니다. 각 Task 유형의 특성을 이해하면 적절한 유형을 선택하는 데 도움이 됩니다.

| Task 유형 | 설명 | 적합한 사용 |
|-----------|------|-----------|
| **Notebook** | Databricks 노트북을 실행합니다. 가장 범용적인 Task 유형입니다 | Python/SQL 변환 작업 |
| **SQL** | SQL 쿼리 또는 SQL 파일을 실행합니다. SQL Warehouse에서 실행됩니다 | 데이터 검증, 집계 |
| **Python Script** | Python 스크립트 파일을 실행합니다. Workspace 또는 Repo의 파일을 지정합니다 | 커스텀 로직 |
| **JAR** | Java/Scala JAR 파일을 실행합니다. 기존 Spark 애플리케이션을 그대로 사용할 수 있습니다 | 레거시 Spark 작업 |
| **Pipeline** | SDP 파이프라인을 실행합니다. 선언적 파이프라인의 실행을 오케스트레이션합니다 | 선언적 ETL 파이프라인 |
| **dbt** | dbt 프로젝트를 실행합니다. dbt Core를 Databricks 위에서 네이티브로 실행합니다 | dbt 기반 변환 |
| **If/Else** | 조건에 따라 분기합니다. 이전 Task의 결과값이나 Job 파라미터를 조건으로 사용합니다 | 조건부 실행 로직 |
| **For Each** | 리스트의 각 항목에 대해 반복 실행합니다. 동적 병렬 처리가 가능합니다 | 배치 루프, 다중 테이블 처리 |

### If/Else 분기 예시

품질 검증 결과에 따라 다음 단계를 분기하는 패턴은 데이터 파이프라인에서 매우 자주 사용됩니다. 데이터 품질이 기준을 충족하지 않으면 파이프라인을 중단하고 담당자에게 알림을 보내는 방식으로 안전한 운영이 가능합니다.

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 품질 검증 (SQL Task) | 데이터 품질을 검증합니다 |
| 2 | 조건 분기 | 결과가 임계치 이상인지 판단합니다 |
| 3a | Yes → 다음 단계 진행 | 정상이면 후속 작업을 실행합니다 |
| 3b | No → 알림 발송 + 중지 | 이상이면 알림을 발송하고 중지합니다 |

### For Each 반복 예시

동일한 처리 로직을 여러 테이블이나 파티션에 반복 적용해야 할 때 For Each Task를 사용합니다. 각 반복은 **병렬로 실행**되므로 처리 시간을 크게 단축할 수 있습니다.

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 테이블 목록 | `['orders', 'customers', 'products']` |
| 2 | For Each Task | 각 테이블에 대해 병렬로 처리를 실행합니다 |
| 3 | 개별 처리 | 처리: orders, 처리: customers, 처리: products |

---

## 트리거 유형

Job을 실행하는 트리거는 다양한 방식으로 설정할 수 있습니다. 데이터의 특성과 비즈니스 요구에 맞는 트리거를 선택하면 효율적인 파이프라인 운영이 가능합니다.

| 트리거 유형 | 설명 | 사용 예시 |
|------------|------|----------|
| **Scheduled (크론)** | 정해진 시간에 주기적으로 실행합니다. 크론 표현식을 사용합니다 | 매일 새벽 2시 ETL, 매주 월요일 보고서 |
| **Continuous** | 이전 실행이 끝나면 즉시 다음 실행을 시작합니다 | 실시간에 가까운 데이터 처리 |
| **File Arrival** | 지정한 위치에 새 파일이 도착하면 실행합니다 | S3 버킷에 CSV 업로드 시 자동 처리 |
| **Table Update** | Unity Catalog 테이블이 갱신되면 실행합니다 | 상위 테이블 변경 시 하위 파이프라인 트리거 |
| **Manual / API** | 수동으로 또는 REST API를 통해 실행합니다 | 애드혹 실행, 외부 시스템 연동 |

### 스케줄 설정 예시

```python
# Python SDK로 크론 스케줄 설정
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

job = w.jobs.create(
    name="daily-sales-pipeline",
    tasks=[...],
    schedule={
        "quartz_cron_expression": "0 0 2 * * ?",  # 매일 새벽 2시
        "timezone_id": "Asia/Seoul",
        "pause_status": "UNPAUSED"
    },
    email_notifications={
        "on_failure": ["team@company.com"],
        "on_success": ["team@company.com"]
    }
)
```

---

## DAG 오케스트레이션 패턴

실무에서 자주 사용되는 DAG 패턴을 소개합니다. 이 패턴들을 조합하여 복잡한 데이터 파이프라인을 구성할 수 있습니다.

### 패턴 1: 순차 실행 (Linear)

가장 단순한 형태로, Task를 하나씩 순서대로 실행합니다.

```
[수집] → [정제] → [집계] → [보고서]
```

### 패턴 2: 팬아웃/팬인 (Fan-out / Fan-in)

하나의 Task 이후 여러 Task를 병렬로 실행하고, 모두 완료된 후 다음 단계로 진행합니다. 독립적인 작업을 병렬 처리하여 전체 실행 시간을 단축합니다.

```
              ┌─ [주문 집계] ──┐
[데이터 수집] ─┼─ [고객 집계] ──┼─ [통합 보고서]
              └─ [상품 집계] ──┘
```

### 패턴 3: 조건부 분기 (Conditional)

If/Else Task를 사용하여 이전 Task의 결과에 따라 다른 경로를 실행합니다.

```
[데이터 검증] → {품질 OK?}
                 ├─ Yes → [프로덕션 적재]
                 └─ No  → [오류 알림] → [대체 로직]
```

### 패턴 4: 동적 반복 (Dynamic Loop)

For Each Task로 동적 목록을 반복 처리합니다. 처리 대상이 런타임에 결정되는 경우에 유용합니다.

```
[대상 목록 조회] → [For Each: 테이블별 처리] → [완료 알림]
```

---

## 서버리스 Jobs

> 🆕 **Serverless Jobs**는 클러스터를 미리 프로비저닝하지 않고, Task 실행 시 **자동으로 컴퓨트가 할당**되는 방식입니다. 클러스터 시작 대기 시간이 대폭 줄어들어 빠른 실행이 가능합니다.

| 비교 | 기존 Job Cluster | Serverless Jobs |
|------|-----------------|-----------------|
| **시작 시간** | 수 분 (클러스터 프로비저닝) | 수 초~수십 초 |
| **인프라 관리** | 인스턴스 타입, 노드 수 직접 설정 | 자동 (Databricks가 관리) |
| **비용** | 클러스터 실행 시간 기준 | Task 실행 시간 기준 (더 정밀) |
| **스케일링** | 수동 또는 오토스케일링 | 자동 스케일링 |
| **적합한 경우** | 장시간 실행, GPU, 특수 라이브러리 | 짧은 배치, SQL, 표준 Python/Spark |

---

## 비용 최적화 가이드

Lakeflow Jobs의 비용을 최적화하려면 다음 전략을 고려하세요.

| 전략 | 설명 | 절감 효과 |
|------|------|----------|
| **Serverless Jobs** | 짧은 Task에 서버리스를 사용하면 클러스터 대기 시간 비용이 절감됩니다 | 높음 |
| **Job Cluster 사용** | All-Purpose 대신 Job Cluster를 사용하면 단가가 낮습니다 | 중간~높음 |
| **Spot 인스턴스** | Worker에 Spot 인스턴스를 적용하면 60~90% 비용 절감이 가능합니다 | 높음 |
| **Task별 컴퓨트 분리** | 가벼운 Task는 작은 클러스터, 무거운 Task는 큰 클러스터를 할당합니다 | 중간 |
| **불필요한 스케줄 정리** | 더 이상 사용하지 않는 Job의 스케줄을 비활성화합니다 | 낮음~중간 |
| **Photon 활성화** | SQL 중심 Task에 Photon을 적용하면 실행 시간이 단축되어 비용이 줄어듭니다 | 중간 |

---

## 시스템 제약 사항

공식 문서에 명시된 Lakeflow Jobs의 시스템 제약입니다. 대규모 운영 시 참고해야 합니다.

| 제약 | 값 |
|------|------|
| 워크스페이스당 최대 동시 Task 실행 수 | **2,000** |
| 시간당 최대 Job 생성 수 | **10,000** |
| 워크스페이스당 최대 저장 Job 수 | **12,000** |
| Job당 최대 Task 수 | **1,000** |
| 최대 동시 Job Run 수 (동일 Job) | 설정 가능 (기본 무제한) |

---

## 프로그래밍 방식 관리

Lakeflow Jobs는 UI 외에도 다양한 방법으로 관리할 수 있습니다. 특히 CI/CD 파이프라인에서는 코드 기반 관리가 필수적입니다.

| 도구 | 설명 | 적합한 용도 |
|------|------|-----------|
| **Databricks CLI** | `databricks jobs create`, `databricks jobs run-now` 등 | 개발/테스트, 스크립트 자동화 |
| **REST API** | `/api/2.1/jobs/create`, `/api/2.1/jobs/run-now` 등 | 외부 시스템 연동 |
| **Python SDK** | `databricks.sdk.WorkspaceClient().jobs.create()` | Python 애플리케이션에서 직접 관리 |
| **Asset Bundles** | YAML로 선언적 정의, `databricks bundle deploy`로 배포 | CI/CD, 환경별 배포 (권장) |
| **Apache Airflow** | `DatabricksRunNowOperator`로 외부 오케스트레이션 | 멀티 플랫폼 오케스트레이션 |
| **Terraform** | `databricks_job` 리소스로 IaC 관리 | 인프라 코드화 (IaC) |

### Asset Bundles로 Job 정의 (YAML)

```yaml
# databricks.yml
bundle:
  name: sales-pipeline

resources:
  jobs:
    daily_sales:
      name: "daily-sales-pipeline"
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"
        timezone_id: "Asia/Seoul"

      tasks:
        - task_key: ingest
          pipeline_task:
            pipeline_id: ${resources.pipelines.bronze.id}

        - task_key: transform
          depends_on:
            - task_key: ingest
          notebook_task:
            notebook_path: ./notebooks/silver_transform.py

        - task_key: aggregate
          depends_on:
            - task_key: transform
          sql_task:
            query: ./sql/gold_aggregate.sql
            warehouse_id: ${var.warehouse_id}
```

---

## 다른 오케스트레이션 도구와의 비교

| 비교 항목 | Lakeflow Jobs | Apache Airflow | AWS Step Functions | dbt Cloud |
|-----------|-------------|----------------|-------------------|-----------|
| **설치/관리** | 관리형 (설치 불필요) | 직접 설치/운영 | AWS 관리형 | SaaS |
| **UI** | Databricks 내장 | Airflow 웹 UI | AWS 콘솔 | dbt Cloud UI |
| **Task 유형** | 노트북, SQL, JAR, 파이프라인, dbt | Python 오퍼레이터 무한 확장 | Lambda, ECS, Glue 등 | SQL 변환 전용 |
| **Databricks 통합** | 네이티브 (완벽) | 커넥터 필요 | 커넥터 필요 | 어댑터 필요 |
| **비용** | DBU에 포함 | 별도 인프라 비용 | 상태 전환 단위 과금 | 별도 SaaS 비용 |
| **학습 곡선** | 낮음 | 높음 (Python 코드) | 중간 (JSON 정의) | 낮음 (SQL 전용) |
| **적합한 경우** | Databricks 중심 파이프라인 | 복잡한 멀티 플랫폼 오케스트레이션 | AWS 네이티브 워크플로우 | SQL 변환 중심 |

> 💡 **권장**: Databricks 내의 워크로드만 오케스트레이션한다면 **Lakeflow Jobs**가 가장 간편하고 비용 효율적입니다. 여러 플랫폼(Databricks + 외부 시스템)을 아우르는 복잡한 오케스트레이션이 필요하다면 **Apache Airflow**를, SQL 변환 중심이라면 **dbt**를 추가로 고려할 수 있습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Job** | Task들의 워크플로우 묶음입니다. 스케줄링과 모니터링의 단위입니다 |
| **Task** | 실제 실행 단위. Notebook, SQL, Pipeline, dbt, If/Else, For Each 등을 지원합니다 |
| **Dependency** | Task 간 실행 순서를 정의합니다. 성공/실패에 따른 조건부 의존성도 지원합니다 |
| **Run** | Job의 한 번의 실행 인스턴스입니다. 실행 이력과 로그가 보존됩니다 |
| **Trigger** | 시간, 파일 도착, 테이블 변경, API 호출 등 Job 실행 조건입니다 |
| **Serverless** | 클러스터 프로비저닝 없이 빠르게 실행됩니다. 짧은 배치에 적합합니다 |
| **Asset Bundles** | YAML로 Job을 선언적으로 정의하고, CI/CD 파이프라인에서 배포합니다 |

---

## 참고 링크

- [Databricks: Lakeflow Jobs](https://docs.databricks.com/aws/en/jobs/)
- [Databricks: Jobs API](https://docs.databricks.com/api/workspace/jobs)
- [Databricks: Serverless Jobs](https://docs.databricks.com/aws/en/jobs/serverless.html)
- [Databricks: Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)
- [Azure Databricks: Workflows](https://learn.microsoft.com/en-us/azure/databricks/workflows/)
