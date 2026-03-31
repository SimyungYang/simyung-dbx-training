# 작업 구성

## 왜 Job 구성이 중요한가?

데이터 파이프라인은 여러 단계(수집 → 변환 → 적재 → 검증)를 거칩니다. 각 단계를 **태스크(Task)** 로 정의하고, 이들의 실행 순서, 실패 시 대응 방식, 컴퓨트 자원 배분까지 체계적으로 설정해야 안정적인 운영이 가능합니다.

> 💡 **Job이란?** Databricks에서 하나 이상의 태스크를 묶어 실행하는 단위입니다. 단일 노트북 실행부터 수십 개 태스크가 의존 관계로 연결된 복잡한 워크플로까지 모두 Job으로 관리할 수 있습니다.

잘 구성된 Job은 다음과 같은 이점을 제공합니다:

| 이점 | 설명 |
|------|------|
| **안정성** | 재시도 정책과 타임아웃으로 일시적 장애를 자동 복구합니다 |
| **가시성** | 태스크 DAG를 통해 파이프라인 전체 흐름을 한눈에 파악할 수 있습니다 |
| **비용 효율** | Job Cluster와 Spot 인스턴스를 활용하여 비용을 최적화합니다 |
| **유지보수** | 파라미터화된 태스크로 코드 변경 없이 다양한 환경에서 실행할 수 있습니다 |

---

## 태스크 유형

Databricks Job은 다양한 유형의 태스크를 지원합니다. 각 태스크 유형은 특정 워크로드에 최적화되어 있습니다.

| 태스크 유형 | 설명 | 대표 사용 사례 |
|-------------|------|----------------|
| **Notebook** | Databricks 노트북을 실행합니다 | ETL 로직, 데이터 변환, ML 학습 |
| **Python Script** | `.py` 파일을 직접 실행합니다 | 범용 Python 스크립트, 배치 처리 |
| **SQL** | SQL 쿼리 또는 SQL 파일을 실행합니다 | 데이터 검증, 집계 테이블 갱신 |
| **dbt** | dbt 프로젝트의 태스크를 실행합니다 | dbt 모델 빌드, 테스트 |
| **Spark Submit** | Spark JAR 또는 Python을 spark-submit으로 실행합니다 | 레거시 Spark 애플리케이션 |
| **Pipeline (SDP)** | SDP(선언적 파이프라인)를 트리거합니다 | 스트리밍 수집, Medallion 파이프라인 |
| **JAR** | Java/Scala JAR 파일을 실행합니다 | Java 기반 ETL, 사내 라이브러리 |
| **Run Job** | 다른 Job을 트리거합니다 | 크로스 팀 워크플로 오케스트레이션 |
| **If/Else 조건** | 조건에 따라 분기 실행합니다 | 데이터 품질 기반 분기 처리 |
| **For Each** | 리스트의 각 항목에 대해 태스크를 반복 실행합니다 | 멀티 테넌트 처리, 파티션별 처리 |

---

## 태스크 의존성 설정

태스크 간 의존성을 설정하면 **DAG(Directed Acyclic Graph)** 형태로 실행 순서가 결정됩니다. Databricks는 세 가지 의존성 패턴을 지원합니다.

### 의존성 패턴

| 패턴 | 설명 | 사용 사례 |
|------|------|-----------|
| **선형(Sequential)** | A → B → C 순서대로 실행합니다 | 단계별 ETL 파이프라인 |
| **병렬(Fan-out/Fan-in)** | 여러 태스크를 동시에 실행한 후 합류합니다 | 독립적인 데이터 소스 동시 수집 |
| **조건부(Conditional)** | 선행 태스크 결과에 따라 분기합니다 | 데이터 검증 결과에 따른 처리 |

### Mermaid 다이어그램: 복잡한 태스크 DAG 예제

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
- `ingest_orders`와 `ingest_products`는 **병렬**로 실행됩니다
- 두 수집 태스크가 모두 완료되면 `transform_orders`가 실행됩니다 (Fan-in)
- `validate_data` 결과에 따라 **조건부 분기**가 발생합니다
- `build_gold_tables` 완료 후 대시보드 갱신과 알림이 **병렬**로 실행됩니다 (Fan-out)

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

      # Job 레벨 파라미터 정의
      parameters:
        - name: "target_date"
          default: ""
        - name: "environment"
          default: "production"

      tasks:
        - task_key: "ingest_orders"
          pipeline_task:
            pipeline_id: "${var.pipeline_id}"

        - task_key: "ingest_products"
          pipeline_task:
            pipeline_id: "${var.products_pipeline_id}"

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
          depends_on:
            - task_key: "validate"
          notebook_task:
            notebook_path: "/Workspace/etl/send_notification"

      # 알림 설정
      email_notifications:
        on_failure:
          - "data-team@company.com"
        on_success:
          - "data-team@company.com"

      webhook_notifications:
        on_failure:
          - id: "${var.slack_webhook_id}"

      # 스케줄 설정
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"
        timezone_id: "Asia/Seoul"
```

---

## 클러스터 설정

### Job Cluster vs All-Purpose Cluster

태스크에 할당하는 컴퓨트 유형에 따라 비용과 성능이 크게 달라집니다.

| 구분 | Job Cluster | All-Purpose Cluster |
|------|-------------|---------------------|
| **생명주기** | Job 실행 시 생성, 완료 시 자동 종료됩니다 | 수동 시작/종료 또는 자동 종료 설정이 필요합니다 |
| **비용** | Job 컴퓨트 요금 적용 (더 저렴) | All-Purpose 요금 적용 (약 2~3배 비쌈) |
| **적합한 용도** | 프로덕션 배치 Job | 개발, 디버깅, 탐색적 분석 |
| **시작 시간** | 매 실행 시 클러스터 시작 대기 필요 (2~5분) | 이미 실행 중이면 즉시 사용 가능합니다 |
| **자원 공유** | 해당 Job 전용입니다 | 여러 사용자/노트북이 공유합니다 |

> ⚠️ **프로덕션 Job에는 반드시 Job Cluster 또는 Serverless를 사용하세요.** All-Purpose Cluster는 개발 단계에서만 사용하고, 운영 환경에서는 비용이 크게 증가할 수 있습니다.

### Serverless Compute

Serverless를 선택하면 클러스터 구성을 별도로 관리할 필요가 없습니다. Databricks가 자동으로 인프라를 프로비저닝하고, 실행이 끝나면 즉시 해제합니다.

| 장점 | 단점 |
|------|------|
| 클러스터 시작 대기 시간 없음 (수 초) | 커스텀 라이브러리 설치 제한 |
| 인프라 관리 불필요 | 특정 인스턴스 타입 지정 불가 |
| 자동 스케일링 | 일부 워크로드에서 비용이 더 높을 수 있음 |

### 인스턴스 타입 선택 가이드

| 워크로드 유형 | 권장 인스턴스 | 이유 |
|---------------|---------------|------|
| **ETL / 데이터 변환** | 메모리 최적화 (r5, r6i) | 셔플, 조인 시 메모리 사용량이 높습니다 |
| **ML 학습** | GPU 인스턴스 (p3, g5) | 딥러닝 모델 학습 가속화 |
| **가벼운 집계 / SQL** | 범용 (m5, m6i) | 비용 대비 균형 잡힌 성능 |
| **대용량 셔플** | 스토리지 최적화 (i3, i4i) | 로컬 NVMe SSD로 셔플 성능 극대화 |

---

## 파라미터 전달

### Job 레벨 파라미터

Job 레벨에서 파라미터를 정의하면 모든 태스크에서 참조할 수 있습니다.

```python
# 노트북에서 Job 파라미터 받기
dbutils.widgets.text("target_date", "2025-03-18")
dbutils.widgets.dropdown("mode", "full", ["full", "incremental"])

target_date = dbutils.widgets.get("target_date")
mode = dbutils.widgets.get("mode")

# 파라미터를 사용한 조건부 처리
if mode == "incremental":
    df = spark.sql(f"""
        SELECT * FROM silver.orders
        WHERE order_date = '{target_date}'
    """)
else:
    df = spark.sql("SELECT * FROM silver.orders")
```

### 태스크 간 값 전달 (Task Values)

선행 태스크에서 계산한 값을 후행 태스크로 전달할 수 있습니다.

```python
# [Task A] 값 설정
row_count = df.count()
dbutils.jobs.taskValues.set(key="processed_rows", value=row_count)
dbutils.jobs.taskValues.set(key="max_date", value="2025-03-18")
```

```python
# [Task B] 값 읽기 (Task A의 결과 참조)
processed_rows = dbutils.jobs.taskValues.get(
    taskKey="task_a",
    key="processed_rows",
    default=0
)
max_date = dbutils.jobs.taskValues.get(
    taskKey="task_a",
    key="max_date",
    default="1970-01-01"
)

print(f"Task A에서 처리한 행 수: {processed_rows}")
print(f"최대 날짜: {max_date}")
```

> 💡 **동적 값 참조**: YAML 설정에서 `{{tasks.task_a.values.processed_rows}}` 형태로 다른 태스크의 Task Value를 참조할 수도 있습니다.

---

## 재시도 정책

태스크가 실패했을 때 자동으로 재시도하는 정책을 설정할 수 있습니다. 네트워크 오류, 일시적 리소스 부족 등 일시적 장애에 효과적입니다.

| 설정 | 설명 | 권장값 |
|------|------|--------|
| **Max Retries** | 실패 시 최대 재시도 횟수입니다 | 프로덕션: 1~3회 |
| **Min Retry Interval (초)** | 재시도 사이 최소 대기 시간입니다 | 60~300초 |
| **Max Retry Interval (초)** | 재시도 사이 최대 대기 시간입니다 | 600~1800초 |
| **Retry on Timeout** | 타임아웃으로 실패한 경우에도 재시도할지 여부입니다 | 상황에 따라 판단 |

```yaml
# Asset Bundles에서 재시도 설정
tasks:
  - task_key: "etl_transform"
    notebook_task:
      notebook_path: "/Workspace/etl/transform"
    max_retries: 2
    min_retry_interval_millis: 120000   # 2분
    retry_on_timeout: false
    timeout_seconds: 3600               # 1시간 타임아웃
```

> ⚠️ **재시도 시 멱등성(Idempotency) 확인**: 재시도되는 태스크는 여러 번 실행해도 같은 결과를 내야 합니다. `MERGE` 문이나 `CREATE OR REPLACE` 패턴을 사용하면 멱등성을 보장할 수 있습니다.

---

## 타임아웃 설정

각 태스크 및 전체 Job에 타임아웃을 설정하여, 무한히 실행되는 상황을 방지할 수 있습니다.

| 레벨 | 설정 | 설명 |
|------|------|------|
| **태스크 레벨** | `timeout_seconds` | 개별 태스크의 최대 실행 시간입니다 |
| **Job 레벨** | `health` 규칙 | 전체 Job의 건강 상태를 모니터링합니다 |

```yaml
tasks:
  - task_key: "heavy_etl"
    timeout_seconds: 7200    # 2시간 타임아웃

# Job 레벨 건강 규칙
health:
  rules:
    - metric: "RUN_DURATION_SECONDS"
      op: "GREATER_THAN"
      value: 10800           # 3시간 초과 시 경고
```

---

## 환경변수 및 시크릿 설정

민감한 정보(API 키, 데이터베이스 비밀번호 등)는 시크릿으로 관리해야 합니다.

```python
# Databricks Secrets를 사용한 자격 증명 관리
api_key = dbutils.secrets.get(scope="production", key="api-key")
db_password = dbutils.secrets.get(scope="production", key="db-password")

# 환경변수로 전달된 값 사용
import os
env = os.environ.get("ENVIRONMENT", "development")
```

```yaml
# Asset Bundles에서 환경변수 설정
tasks:
  - task_key: "data_sync"
    notebook_task:
      notebook_path: "/Workspace/etl/sync"
    new_cluster:
      spark_env_vars:
        ENVIRONMENT: "${bundle.target}"
        LOG_LEVEL: "INFO"
      spark_conf:
        "spark.databricks.delta.optimizeWrite.enabled": "true"
        "spark.databricks.delta.autoCompact.enabled": "true"
```

> ⚠️ **보안 주의**: 절대로 코드나 설정 파일에 비밀번호, API 키를 직접 입력하지 마세요. 반드시 `dbutils.secrets`를 사용하거나 Unity Catalog의 시크릿 관리 기능을 활용하세요.

---

## 알림 설정

Job의 상태 변화에 따라 다양한 채널로 알림을 보낼 수 있습니다.

| 알림 이벤트 | 설명 | 권장 설정 |
|-------------|------|-----------|
| **On Start** | Job 실행 시작 시 알림을 보냅니다 | 장시간 Job에 설정 |
| **On Success** | 성공적으로 완료 시 알림을 보냅니다 | 주요 파이프라인에 설정 |
| **On Failure** | 실패 시 알림을 보냅니다 | **모든 프로덕션 Job에 필수** |
| **On Duration Warning** | 설정 시간 초과 시 경고를 보냅니다 | SLA 관리용 |
| **On Streaming Backlog** | 스트리밍 백로그 임계치 초과 시 알림을 보냅니다 | 스트리밍 Job에 설정 |

```yaml
# 알림 설정 예제
email_notifications:
  on_start:
    - "oncall@company.com"
  on_success:
    - "data-team@company.com"
  on_failure:
    - "oncall@company.com"
    - "data-team-lead@company.com"
  on_duration_warning_threshold_exceeded:
    - "oncall@company.com"

webhook_notifications:
  on_failure:
    - id: "${var.slack_webhook_id}"
    - id: "${var.pagerduty_webhook_id}"

notification_settings:
  no_alert_for_skipped_runs: true      # 스킵된 실행에는 알림 안 보냄
  no_alert_for_canceled_runs: false    # 취소된 실행에는 알림 보냄
```

---

## 실습: 멀티태스크 Job 생성

다음은 E-Commerce 매출 파이프라인을 위한 완전한 Job 설정 예제입니다.

### JSON 설정 (API 방식)

```json
{
  "name": "ecommerce-daily-pipeline",
  "tags": {
    "team": "data-engineering",
    "project": "ecommerce",
    "cost_center": "DE-001"
  },
  "tasks": [
    {
      "task_key": "ingest_orders",
      "description": "S3에서 주문 데이터 수집",
      "pipeline_task": {
        "pipeline_id": "abc-123-def"
      },
      "timeout_seconds": 1800
    },
    {
      "task_key": "ingest_customers",
      "description": "고객 마스터 데이터 동기화",
      "notebook_task": {
        "notebook_path": "/Workspace/etl/ingest_customers",
        "base_parameters": {
          "mode": "incremental"
        }
      },
      "new_cluster": {
        "spark_version": "15.4.x-scala2.12",
        "num_workers": 2,
        "node_type_id": "m5.xlarge",
        "aws_attributes": {
          "availability": "SPOT_WITH_FALLBACK",
          "first_on_demand": 1
        }
      },
      "timeout_seconds": 1800,
      "max_retries": 2,
      "min_retry_interval_millis": 60000
    },
    {
      "task_key": "transform_and_join",
      "description": "주문-고객 조인 및 변환",
      "depends_on": [
        {"task_key": "ingest_orders"},
        {"task_key": "ingest_customers"}
      ],
      "notebook_task": {
        "notebook_path": "/Workspace/etl/transform_orders",
        "base_parameters": {
          "target_date": "{{job.parameters.target_date}}"
        }
      },
      "new_cluster": {
        "spark_version": "15.4.x-scala2.12",
        "num_workers": 4,
        "node_type_id": "r5.xlarge"
      },
      "timeout_seconds": 3600,
      "max_retries": 1
    },
    {
      "task_key": "validate_results",
      "description": "데이터 품질 검증",
      "depends_on": [
        {"task_key": "transform_and_join"}
      ],
      "sql_task": {
        "query": {
          "query_text": "SELECT CASE WHEN COUNT(*) > 0 THEN 'PASS' ELSE 'FAIL' END AS result FROM gold.daily_revenue WHERE sale_date = CURRENT_DATE()"
        },
        "warehouse_id": "warehouse-id-here"
      }
    },
    {
      "task_key": "build_aggregates",
      "description": "Gold 집계 테이블 생성",
      "depends_on": [
        {"task_key": "validate_results"}
      ],
      "dbt_task": {
        "project_directory": "/Workspace/dbt/ecommerce",
        "commands": ["dbt run --select gold_models"],
        "warehouse_id": "warehouse-id-here"
      }
    }
  ],
  "parameters": [
    {
      "name": "target_date",
      "default": ""
    }
  ],
  "email_notifications": {
    "on_failure": ["data-team@company.com"]
  },
  "webhook_notifications": {
    "on_failure": [{"id": "slack-webhook-id"}]
  },
  "schedule": {
    "quartz_cron_expression": "0 0 2 * * ?",
    "timezone_id": "Asia/Seoul"
  },
  "max_concurrent_runs": 1,
  "health": {
    "rules": [
      {
        "metric": "RUN_DURATION_SECONDS",
        "op": "GREATER_THAN",
        "value": 7200
      }
    ]
  }
}
```

---

## 비용 최적화 팁

### Spot 인스턴스 활용

| 전략 | 설정 | 효과 |
|------|------|------|
| **SPOT_WITH_FALLBACK** | Spot 우선, 불가 시 On-Demand로 전환 | 비용 60~90% 절감, 안정성 유지 |
| **first_on_demand** | Driver는 On-Demand, Worker는 Spot | Driver 장애 방지 |
| **spot_bid_max_price** | Spot 최대 입찰가 설정 | 예산 제어 |

```yaml
new_cluster:
  aws_attributes:
    availability: "SPOT_WITH_FALLBACK"
    first_on_demand: 1        # Driver 노드만 On-Demand
    spot_bid_max_price: -1    # 시장 가격 사용
    zone_id: "auto"           # 가용 영역 자동 선택
```

### 클러스터 사이징 가이드

| 데이터 크기 | Workers 수 | 인스턴스 타입 | 비용 |
|-----------|-----------|-------------|------|
| < 10GB | 2 Workers | m5.xlarge | $ |
| 10~100GB | 4~8 Workers | r5.xlarge | $$ |
| 100GB~1TB | 8~16 Workers | r5.2xlarge | $$$ |
| > 1TB | Auto-scaling 16~64 Workers | - | $$$$ |

### 추가 비용 최적화 체크리스트

- **Auto-scaling 활용**: `autoscale.min_workers`와 `autoscale.max_workers`를 설정하여 워크로드에 맞게 자동 조절합니다
- **Photon 엔진 사용**: SQL 및 DataFrame 워크로드에서 성능이 2~8배 향상됩니다
- **태스크별 클러스터 공유**: 동일한 컴퓨트가 필요한 태스크는 `job_cluster_key`로 클러스터를 공유할 수 있습니다
- **불필요한 셔플 제거**: `broadcast join`, `partition pruning` 등으로 데이터 이동을 최소화합니다
- **Delta 최적화**: `optimizeWrite`, `autoCompact`를 활성화하여 소규모 파일 문제를 방지합니다

```yaml
# 태스크 간 클러스터 공유 예제
job_clusters:
  - job_cluster_key: "shared_etl_cluster"
    new_cluster:
      spark_version: "15.4.x-scala2.12"
      num_workers: 4
      node_type_id: "r5.xlarge"
      runtime_engine: "PHOTON"

tasks:
  - task_key: "transform_a"
    job_cluster_key: "shared_etl_cluster"
    notebook_task:
      notebook_path: "/Workspace/etl/transform_a"

  - task_key: "transform_b"
    job_cluster_key: "shared_etl_cluster"
    depends_on:
      - task_key: "transform_a"
    notebook_task:
      notebook_path: "/Workspace/etl/transform_b"
```

---

## 정리

| 구성 요소 | 핵심 포인트 |
|-----------|-------------|
| **태스크 유형** | Notebook, SQL, dbt, Pipeline 등 워크로드에 맞는 유형을 선택합니다 |
| **의존성** | DAG로 선형, 병렬, 조건부 흐름을 설계합니다 |
| **클러스터** | Job Cluster + Spot 인스턴스로 비용을 최적화합니다 |
| **파라미터** | `dbutils.widgets`와 Task Values로 태스크 간 데이터를 전달합니다 |
| **재시도/타임아웃** | 일시적 장애에 대비하되, 멱등성을 보장합니다 |
| **알림** | 프로덕션 Job에는 반드시 실패 알림을 설정합니다 |

---

## 참고 링크

- [Databricks: Configure jobs](https://docs.databricks.com/aws/en/jobs/create-run-jobs.html)
- [Databricks: Task types](https://docs.databricks.com/aws/en/jobs/task-types.html)
- [Databricks: Share clusters across tasks](https://docs.databricks.com/aws/en/jobs/create-run-jobs.html#use-shared-job-clusters)
- [Databricks: Pass parameters to jobs](https://docs.databricks.com/aws/en/jobs/parameters.html)
- [Databricks: Serverless compute for jobs](https://docs.databricks.com/aws/en/jobs/serverless.html)
