# 실전 파이프라인 오케스트레이션 패턴

## 패턴 1: Lakeflow Jobs로 End-to-End 파이프라인 오케스트레이션

Lakeflow Jobs는 **여러 종류의 태스크를 DAG(방향성 비순환 그래프)로 조합** 하여 전체 파이프라인을 하나의 워크플로로 관리합니다. 수집 → 변환 → 품질 검증 → 서빙까지 하나의 Job으로 구성할 수 있습니다.

```
[Job: daily-data-pipeline]
│
├─ Task 1: 수집 (SDP Pipeline — Lakeflow Connect CDC)
│   └─ Bronze 테이블에 원본 데이터 적재
│
├─ Task 2: 변환 (SDP Pipeline — Bronze → Silver → Gold)
│   ├─ Silver: 정제, 중복 제거, SCD Type 1/2
│   └─ Gold: 비즈니스 집계
│
├─ Task 3: 품질 검증 (SQL Task)
│   └─ Gold 테이블의 행 수, NULL 비율, 전일 대비 변화량 체크
│
├─ Task 4-A: 성공 시 → ML 피처 갱신 (Notebook Task)
│   └─ Feature Table 업데이트 → Online Table 동기화
│
├─ Task 4-B: 성공 시 → 대시보드 새로고침 (SQL Task)
│   └─ REFRESH MATERIALIZED VIEW gold_daily_kpi
│
└─ Task 5: 실패 시 → Slack 알림 (Webhook)
```

```python
# Lakeflow Jobs SDK로 위 파이프라인 생성
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import *

w = WorkspaceClient()

w.jobs.create(
    name="daily-data-pipeline",
    tasks=[
        Task(
            task_key="ingest",
            pipeline_task=PipelineTask(pipeline_id="<ingest-pipeline-id>"),
        ),
        Task(
            task_key="transform",
            depends_on=[TaskDependency(task_key="ingest")],
            pipeline_task=PipelineTask(pipeline_id="<sdp-pipeline-id>"),
        ),
        Task(
            task_key="quality_check",
            depends_on=[TaskDependency(task_key="transform")],
            sql_task=SqlTask(
                query=SqlTaskQuery(query_id="<quality-check-query-id>"),
                warehouse_id="<warehouse-id>"
            ),
        ),
        Task(
            task_key="update_features",
            depends_on=[TaskDependency(task_key="quality_check")],
            notebook_task=NotebookTask(
                notebook_path="/Workspace/pipelines/update_features"
            ),
        ),
        Task(
            task_key="refresh_dashboard",
            depends_on=[TaskDependency(task_key="quality_check")],
            sql_task=SqlTask(
                query=SqlTaskQuery(query_id="<refresh-mv-query-id>"),
                warehouse_id="<warehouse-id>"
            ),
        ),
    ],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 6 * * ?",  # 매일 06:00
        timezone_id="Asia/Seoul"
    ),
    email_notifications=JobEmailNotifications(
        on_failure=["data-team@company.com"]
    )
)
```

---

## 패턴 2: CDC 파이프라인 — SCD Type 1 vs Type 2

CDC(Change Data Capture) 데이터를 처리할 때 가장 중요한 결정은 **SCD(Slowly Changing Dimension) 전략**입니다.

### SCD Type 1: 최신 값으로 덮어쓰기

변경 이력을 보존하지 않고, **항상 최신 값만 유지**합니다.

```sql
-- SDP에서 SCD Type 1 구현
CREATE OR REFRESH STREAMING TABLE silver_customers;

APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customers)
KEYS (customer_id)
SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 1;
```

| 항목 | 설명 |
|------|------|
| **결과 테이블** | customer_id당 항상 1개 행 (최신) |
| **이전 값** | 삭제됨 (복구 불가, Delta Time Travel로만 가능) |
| **적합한 경우** | 최신 상태만 필요 (주소, 이메일, 전화번호 등) |
| **스토리지** | 효율적 (행 수 = 고유 키 수) |

### SCD Type 2: 변경 이력 보존

모든 변경 이력을 **별도 행으로 보존**합니다. `__START_AT`, `__END_AT` 컬럼으로 유효 기간을 관리합니다.

```sql
-- SDP에서 SCD Type 2 구현
CREATE OR REFRESH STREAMING TABLE silver_customers_history;

APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_customers)
KEYS (customer_id)
SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 2;

-- 결과 테이블 구조:
-- | customer_id | name   | city  | __START_AT     | __END_AT       |
-- |-------------|--------|-------|----------------|----------------|
-- | 1001        | 김철수 | 서울  | 2025-01-01     | 2025-03-15     |
-- | 1001        | 김철수 | 부산  | 2025-03-15     | NULL           | ← 현재 값
```

| 항목 | 설명 |
|------|------|
| **결과 테이블** | customer_id당 여러 행 (변경 이력) |
| **현재 값 조회** | `WHERE __END_AT IS NULL` |
| **특정 시점 조회** | `WHERE __START_AT <= '2025-02-01' AND (__END_AT > '2025-02-01' OR __END_AT IS NULL)` |
| **적합한 경우** | 이력 추적 필수 (고객 등급 변화, 가격 변동, 규제 감사) |
| **스토리지** | 변경 빈도에 비례하여 증가 |

### SCD Type 1 vs Type 2 선택 기준

| 기준 | Type 1 | Type 2 |
|------|--------|--------|
| **"이전 값이 필요한가?"** | 아니오 → Type 1 | 예 → Type 2 |
| **규제 감사 요건** | 없음 → Type 1 | 있음 (변경 이력 보존) → Type 2 |
| **분석 요구** | 현재 상태 분석 → Type 1 | 시간에 따른 변화 분석 → Type 2 |
| **스토리지 비용** | 낮음 | 높음 (이력 누적) |
| **쿼리 복잡도** | 단순 (항상 최신) | 복잡 (`__START_AT`/`__END_AT` 필터) |
| **대표 사용** | 고객 연락처, 제품 정보 | 고객 등급, 가격 이력, 재고 변동 |

### 실전: 하이브리드 전략 (Type 1 + Type 2 동시 운영)

대부분의 프로덕션 환경에서는 **같은 소스에서 Type 1과 Type 2를 동시에 생성**합니다.

```sql
-- 같은 Bronze 소스에서 두 가지 Silver 테이블 생성

-- Silver 1: 현재 상태 (조인용, 대시보드용) — SCD Type 1
CREATE OR REFRESH STREAMING TABLE silver_customers_current;
APPLY CHANGES INTO silver_customers_current
FROM STREAM(bronze_customers)
KEYS (customer_id) SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 1;

-- Silver 2: 전체 이력 (감사, 시계열 분석용) — SCD Type 2
CREATE OR REFRESH STREAMING TABLE silver_customers_history;
APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_customers)
KEYS (customer_id) SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 2;

-- Gold: 현재 상태 기반 집계 (Type 1에서 읽음 → 빠른 조인)
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_revenue AS
SELECT c.customer_id, c.name, c.city,
       COUNT(o.order_id) AS total_orders,
       SUM(o.amount) AS total_revenue
FROM silver_customers_current c
JOIN silver_orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city;

-- 감사 리포트: 이력 기반 조회 (Type 2에서 읽음)
-- "2025년 1분기에 서울에 거주했던 고객 목록"
SELECT * FROM silver_customers_history
WHERE city = '서울'
  AND __START_AT <= '2025-03-31'
  AND (__END_AT > '2025-01-01' OR __END_AT IS NULL);
```

> 💡 **왜 하이브리드인가?** Type 2만 사용하면 Gold 계층의 JOIN이 복잡해집니다 (`__END_AT IS NULL` 조건이 매번 필요). Type 1의 "현재 상태" 테이블을 별도로 유지하면 **Gold 집계가 단순하고 빠릅니다.**

---

## 패턴 3: CDC → CDF → 다운스트림 전파

외부 DB의 CDC를 수집하고, Delta CDF(Change Data Feed)로 다운스트림에 전파하는 전체 흐름입니다.

```
[외부 MySQL]
    │ Lakeflow Connect (CDC — binlog)
    ▼
[Bronze: bronze_customers]  ← 원본 CDC 이벤트 보존
    │ SDP: APPLY CHANGES INTO
    ▼
[Silver: silver_customers]  ← SCD Type 1 (CDF 활성화)
    │ Delta CDF (readChangeFeed)
    ├──▶ [Online Table] ← 실시간 Feature Serving
    ├──▶ [Gold: gold_customer_360] ← MV 증분 갱신
    └──▶ [외부 시스템] ← Reverse ETL (foreachBatch)
```

```sql
-- Silver 테이블에 CDF 활성화 (다운스트림 전파용)
ALTER TABLE silver_customers
SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');
```

```python
# Silver의 변경사항을 외부 시스템에 Reverse ETL
def push_to_crm(batch_df, batch_id):
    updates = batch_df.filter("_change_type IN ('insert', 'update_postimage')")
    # CRM API 호출로 고객 정보 동기화
    for row in updates.collect():
        crm_api.update_customer(row.customer_id, row.name, row.email)

(spark.readStream
    .format("delta")
    .option("readChangeFeed", "true")
    .table("silver_customers")
    .writeStream
    .foreachBatch(push_to_crm)
    .option("checkpointLocation", "/checkpoints/reverse-etl")
    .trigger(availableNow=True)
    .start()
)
```

---

## 파이프라인 오케스트레이션 모범 사례

| # | 원칙 | 설명 |
|---|------|------|
| 1 | **수집과 변환을 분리** | Lakeflow Connect(수집)과 SDP(변환)를 **별도 파이프라인**으로 구성합니다. 수집이 실패해도 변환은 마지막 성공 데이터로 동작합니다 |
| 2 | **Job으로 전체 오케스트레이션** | 수집 파이프라인 → SDP 파이프라인 → 품질 검증 → 후처리를 하나의 Lakeflow Job DAG로 묶습니다 |
| 3 | **품질 게이트** | Silver → Gold 사이에 품질 검증 태스크를 삽입합니다. 실패 시 Gold 갱신을 차단합니다 |
| 4 | **멱등성 보장** | 모든 태스크는 재실행해도 같은 결과를 보장해야 합니다. MERGE, APPLY CHANGES는 기본적으로 멱등적입니다 |
| 5 | **환경별 분리** | Dev/Staging/Prod에 동일한 파이프라인을 Declarative Automation Bundles로 배포합니다 |
| 6 | **SCD 전략은 비즈니스 요구로 결정** | 이력이 필요하면 Type 2, 최신값만 필요하면 Type 1. 대부분 하이브리드 |
| 7 | **CDF로 다운스트림 연결** | Silver 테이블에 CDF를 활성화하면 Online Table, MV, Reverse ETL이 증분으로 동작합니다 |

---

## 참고 링크

- [Databricks: Data Engineering](https://docs.databricks.com/aws/en/data-engineering)
- [Databricks: Lakeflow Jobs](https://docs.databricks.com/aws/en/jobs/)
- [Databricks: SDP CDC and SCD](https://docs.databricks.com/aws/en/delta-live-tables/cdc.html)
- [Databricks Blog: Introducing Lakeflow](https://www.databricks.com/blog/introducing-databricks-lakeflow)
