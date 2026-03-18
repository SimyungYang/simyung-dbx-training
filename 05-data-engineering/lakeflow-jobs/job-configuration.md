# 작업 구성

## Job 생성 방법

### UI에서 생성

1. 좌측 메뉴 **Workflows** → **Create Job**
2. Job 이름 입력
3. Task 추가 (노트북 경로, SQL 쿼리 등 지정)
4. Task 간 의존성 설정 (드래그&드롭)
5. 컴퓨트 설정 (Serverless 또는 Job Cluster)

### Asset Bundles (IaC)

```yaml
resources:
  jobs:
    daily_pipeline:
      name: "daily-sales-pipeline"
      tasks:
        - task_key: "ingest"
          pipeline_task:
            pipeline_id: "${var.pipeline_id}"
        - task_key: "transform"
          depends_on:
            - task_key: "ingest"
          notebook_task:
            notebook_path: "/Workspace/etl/transform"
          environment_key: "default"
        - task_key: "validate"
          depends_on:
            - task_key: "transform"
          sql_task:
            query:
              query_text: "SELECT COUNT(*) FROM gold.daily_revenue WHERE sale_date = CURRENT_DATE()"
            warehouse_id: "${var.warehouse_id}"
```

---

## 매개변수 전달

Task 간에 값을 전달하거나, Job 실행 시 외부에서 매개변수를 주입할 수 있습니다.

```python
# 노트북에서 매개변수 받기
dbutils.widgets.text("date", "2025-03-18")
target_date = dbutils.widgets.get("date")

# 매개변수를 사용한 처리
df = spark.sql(f"SELECT * FROM silver.orders WHERE order_date = '{target_date}'")
```

---

## 재시도 설정

| 설정 | 설명 |
|------|------|
| **Max Retries** | 실패 시 최대 재시도 횟수 |
| **Min Retry Interval** | 재시도 간 최소 대기 시간 |
| **Retry on Timeout** | 타임아웃 시에도 재시도 여부 |

---

## 참고 링크

- [Databricks: Configure jobs](https://docs.databricks.com/aws/en/jobs/create-run-jobs.html)
