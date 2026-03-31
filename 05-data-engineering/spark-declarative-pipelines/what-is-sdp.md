# SDP란? — 선언적 파이프라인

## 개념

> 💡 **Spark Declarative Pipelines (SDP)** 는 데이터 변환 파이프라인을 **"무엇을(What)"** 만들지 선언하면, Databricks가 **"어떻게(How)"** 실행할지를 자동으로 관리해 주는 프레임워크입니다. 이전에는 **Delta Live Tables (DLT)** 라는 이름으로 불렸습니다.

Apache Spark Structured Streaming 위에 구축된 추상화 레이어로, 배치와 스트리밍 데이터 파이프라인을 SQL 또는 Python으로 생성할 수 있습니다. 클라우드 스토리지 파일 수집, 메시지 버스 소비, 증분 배치/스트리밍 변환을 모두 지원합니다.

---

## 명령형 vs 선언형 비교

| 방식 | 명령형 (Imperative) | 선언형 (Declarative) |
|------|-------------------|-------------------|
| **비유** | "달걀을 깨고, 팬을 달구고, 기름을 두르고..." | "스크램블 에그를 만들어 주세요" |
| **코드** | 읽기→변환→쓰기→에러처리→재시도 직접 구현 | 결과 테이블의 정의만 작성 |
| **의존성** | 개발자가 실행 순서를 직접 관리 | 시스템이 자동으로 파악하고 실행 |
| **에러 처리** | 개발자가 체크포인트, 재시도 직접 구현 | 시스템이 자동 관리 및 복구 |
| **인프라** | 클러스터 생성, 라이브러리 설치 직접 수행 | 서버리스로 자동 프로비저닝 |

```python
# ❌ 명령형: 모든 것을 직접 관리해야 합니다
df = spark.readStream.format("delta").load("/bronze/orders")
cleaned = df.filter(col("order_id").isNotNull()).dropDuplicates(["order_id"])
query = (cleaned.writeStream
    .format("delta")
    .option("checkpointLocation", "/checkpoints/silver_orders")
    .option("mergeSchema", "true")
    .outputMode("append")
    .trigger(availableNow=True)
    .toTable("silver_orders"))
query.awaitTermination()
```

```sql
-- ✅ 선언형 (SDP): 결과만 정의합니다. 나머지는 시스템이 알아서 합니다.
CREATE OR REFRESH STREAMING TABLE silver_orders (
    CONSTRAINT valid_id EXPECT (order_id IS NOT NULL) ON VIOLATION DROP ROW
)
AS SELECT * FROM STREAM(bronze_orders);
```

---

## SDP의 핵심 구성 요소

### 데이터 객체

| 구성 요소 | 설명 | SQL 키워드 |
|-----------|------|-----------|
| **Streaming Table** | 새 데이터만 증분 처리하는 추가 전용 테이블입니다. 각 레코드가 정확히 한 번(Exactly-Once) 처리됩니다 | `CREATE STREAMING TABLE` |
| **Materialized View** | 전체 데이터를 대상으로 결과를 계산하고 캐싱하는 뷰입니다. 스케줄에 따라 자동 갱신됩니다 | `CREATE MATERIALIZED VIEW` |
| **Temporary View** | 파이프라인 내에서만 사용할 수 있는 중간 결과입니다. 외부에서 조회할 수 없습니다 | `CREATE TEMPORARY VIEW` (SQL) / `@dp.temporary_view` (Python) |

### 처리 단위

| 구성 요소 | 설명 |
|-----------|------|
| **Flow** | 소스에서 타겟으로 데이터를 이동시키는 처리 단위입니다. Append, Append-Once, CDC 세 가지 유형이 있습니다 |
| **Sink** | 외부 시스템으로 데이터를 내보내는 메커니즘입니다 |
| **Pipeline** | 여러 테이블, 뷰, Flow를 묶어서 실행하는 배포 단위입니다 |

### 실행 모드

| 모드 | 설명 | 적합한 사용 |
|------|------|-----------|
| **Triggered** | 수동으로 실행하거나 스케줄에 따라 실행합니다. 처리 완료 후 리소스를 해제합니다 | 배치 ETL, 정기 갱신 |
| **Continuous** | 지속적으로 실행되며, 새 데이터가 도착하면 즉시 처리합니다 | 실시간 스트리밍 |
| **Development Mode** | 빠른 반복 개발을 위한 모드입니다. 실패 시 클러스터를 유지하여 디버깅이 용이합니다 | 개발/테스트 |

---

## SDP의 장점

| 장점 | 설명 |
|------|------|
| **자동 의존성 관리** | 테이블 간 참조를 분석하여 실행 순서를 자동으로 결정합니다. DAG(Directed Acyclic Graph)를 시각적으로 표시합니다 |
| **자동 증분 처리** | Streaming Table은 변경된 데이터만 처리하여 비용과 시간을 절약합니다 |
| **데이터 품질 모니터링** | Expectations로 데이터 품질을 자동으로 검증하고, 위반 건수를 리포팅합니다 |
| **자동 에러 복구** | 체크포인트에서 자동으로 재시작합니다. 일시적 오류에 대한 재시도도 자동입니다 |
| **서버리스 지원** | 클러스터 관리 없이 서버리스로 실행할 수 있습니다. 자동 스케일링을 제공합니다 |
| **스키마 자동 관리** | Auto Loader와 결합 시 스키마 추론, 진화, 체크포인트가 모두 자동입니다 |
| **Unity Catalog 통합** | 생성된 테이블이 자동으로 Unity Catalog에 등록되어 거버넌스가 적용됩니다 |

---

## SQL과 Python 지원

SDP는 SQL과 Python 두 가지 언어를 지원하며, **하나의 파이프라인에서 두 언어를 혼합**하여 사용할 수 있습니다 (단, 파일 단위로는 단일 언어).

### SQL 방식

```sql
-- Streaming Table
CREATE OR REFRESH STREAMING TABLE bronze_orders
AS SELECT * FROM STREAM read_files('s3://bucket/orders/', format => 'json');

-- Materialized View
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_revenue
AS SELECT DATE(order_date) AS day, SUM(amount) AS revenue
FROM silver_orders GROUP BY 1;
```

### Python 방식

```python
from pyspark import pipelines as dp

@dp.table(comment="Bronze 주문 데이터")
def bronze_orders():
    return (spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .load("s3://bucket/orders/"))

@dp.materialized_view(comment="일별 매출 집계")
def gold_daily_revenue():
    return spark.sql("""
        SELECT DATE(order_date) AS day, SUM(amount) AS revenue
        FROM silver_orders GROUP BY 1
    """)
```

---

## 파이프라인 생성 및 실행

### UI에서 생성

1. **Pipelines** → **Create Pipeline**
2. Pipeline 이름 입력
3. Source Code: 노트북 또는 파일 경로 지정 (최대 100개 소스 파일)
4. Destination: Unity Catalog의 카탈로그.스키마 지정
5. Compute: **Serverless** (권장) 또는 Classic
6. Pipeline Mode: **Triggered** 또는 **Continuous**
7. **Start** 클릭

### Asset Bundles (IaC)

```yaml
resources:
  pipelines:
    sales_pipeline:
      name: "sales-medallion-pipeline"
      catalog: "production"
      schema: "ecommerce"
      serverless: true
      continuous: false
      libraries:
        - notebook:
            path: "/Workspace/pipelines/bronze.sql"
        - notebook:
            path: "/Workspace/pipelines/silver.sql"
        - notebook:
            path: "/Workspace/pipelines/gold.sql"
      configuration:
        "pipelines.trigger.interval": "1 hour"
```

---

## 시스템 제약 사항

공식 문서에 명시된 파이프라인 제약 사항을 정리합니다.

| 제약 | 값 |
|------|------|
| 워크스페이스당 최대 동시 파이프라인 업데이트 | **1,000** |
| 파이프라인당 최대 소스 파일 수 | **100** (50개 파일/폴더 조합, 간접적으로 최대 1,000개 파일) |
| Materialized View의 외부 스트리밍 소스 사용 | ❌ 불가 (같은 파이프라인 내의 Streaming Table만 소스로 사용 가능) |
| Hive Metastore에서 Unity Catalog로 업그레이드 | ❌ 불가 (새 파이프라인 생성 필요) |
| JAR 라이브러리 사용 | ❌ 불가 (Python 라이브러리만 지원) |
| `pivot()` 함수 사용 | ❌ 불가 (Eager Loading 이슈) |
| Delta Lake 타임 트래블 | Streaming Table만 가능. Materialized View에서는 불가 |
| Iceberg 읽기 | Streaming Table, Materialized View에서 미지원 |

---

## 이벤트 로그 모니터링

파이프라인의 실행 이벤트, 데이터 품질 결과, 성능 메트릭을 이벤트 로그를 통해 SQL로 조회할 수 있습니다.

```sql
-- 파이프라인 이벤트 로그 조회
SELECT
    timestamp,
    event_type,
    message,
    details
FROM event_log(TABLE(catalog.schema.my_streaming_table))
WHERE event_type IN ('flow_progress', 'dataset_quality')
ORDER BY timestamp DESC;
```

---

## 최신 기능

> 🆕 **TRIGGER ON UPDATE**: 소스 테이블이 변경되면 파이프라인을 자동으로 갱신하는 기능입니다.

> 🆕 **Dry Run**: 실제 데이터를 업데이트하지 않고 파이프라인의 정확성을 검증하는 기능입니다. 개발 중 빠른 피드백에 유용합니다.

> 🆕 **Serverless Performance Mode**: 서버리스 SDP의 성능을 최적화하는 모드가 베타로 출시되었습니다.

> 🆕 **Failure Notifications**: 파이프라인 실패 시 자동 알림을 보내는 기능이 추가되었습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SDP** | 선언적 데이터 파이프라인 프레임워크입니다 (구 Delta Live Tables) |
| **Streaming Table** | 새 데이터만 증분 처리합니다. Exactly-Once 보장됩니다 |
| **Materialized View** | 전체 데이터를 재계산하여 캐싱합니다 |
| **Flow** | 소스→타겟 데이터 이동 단위입니다 (Append, CDC) |
| **Expectations** | 데이터 품질 규칙을 선언적으로 정의합니다 |
| **Triggered/Continuous** | 배치 또는 실시간 실행 모드를 선택합니다 |

---

## 참고 링크

- [Databricks: Spark Declarative Pipelines](https://docs.databricks.com/aws/en/sdp/)
- [Databricks: SDP SQL reference](https://docs.databricks.com/aws/en/sdp/sql-ref.html)
- [Databricks: SDP Python reference](https://docs.databricks.com/aws/en/sdp/python-ref.html)
- [Databricks: Pipeline limitations](https://docs.databricks.com/aws/en/sdp/limitations.html)
- [Azure Databricks: Delta Live Tables](https://learn.microsoft.com/en-us/azure/databricks/delta-live-tables/)
