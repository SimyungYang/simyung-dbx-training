# 실습과 비용 최적화

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
| **SPOT_WITH_FALLBACK**| Spot 우선, 불가 시 On-Demand로 전환 | 비용 60~90% 절감, 안정성 유지 |
| **first_on_demand**| Driver는 On-Demand, Worker는 Spot | Driver 장애 방지 |
| **spot_bid_max_price**| Spot 최대 입찰가 설정 | 예산 제어 |

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
