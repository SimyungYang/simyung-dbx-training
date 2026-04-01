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

## DLT에서 SDP로 — 리브랜딩 배경

Delta Live Tables(DLT)는 2021년에 출시된 Databricks의 선언적 파이프라인 프레임워크였습니다. 2024년부터 **Spark Declarative Pipelines(SDP)** 로 이름이 변경되었는데, 이 변화에는 전략적 의미가 있습니다.

| 시점 | 명칭 | 변화 배경 |
|------|------|----------|
| 2021 | **Delta Live Tables (DLT)** | Delta Lake 위의 ETL 프레임워크로 출시 |
| 2024 | **Lakeflow Declarative Pipelines** | Lakeflow 브랜드 통합 (Connect + Jobs + DLT) |
| 2025 | **Spark Declarative Pipelines (SDP)** | Apache Spark 프로젝트에 기여, 오픈소스화 방향 |

**리브랜딩의 핵심 이유:**

1. **오픈소스 전략**: Databricks는 SDP의 핵심 기술을 Apache Spark 프로젝트에 기여하여, 벤더 종속(lock-in) 우려를 해소하려 합니다. "Delta Live Tables"라는 이름은 Databricks 독점 느낌이 강했지만, "Spark Declarative Pipelines"는 Spark 생태계의 일부라는 인식을 줍니다.
2. **Lakeflow 브랜드 통합**: 수집(Connect), 변환(SDP), 오케스트레이션(Jobs)을 하나의 Lakeflow 제품군으로 묶어 데이터 엔지니어링 전체 라이프사이클을 어필합니다.
3. **Delta Lake 의존성 완화**: 향후 Iceberg 등 다른 테이블 포맷도 지원할 수 있는 확장성을 확보합니다.

> ⚠️ **실무 참고**: 기존 DLT 파이프라인 코드는 SDP에서 **100% 호환**됩니다. `dlt` 패키지가 `pyspark.pipelines`로 변경되었으나, 기존 `import dlt`도 계속 동작합니다. API 변경 사항은 마이그레이션 가이드를 참고하세요.

---

## SDP vs Apache Airflow 비교

데이터 파이프라인을 구축할 때 Apache Airflow와 SDP 중 어떤 것을 선택할지 고민하는 경우가 많습니다. 두 도구는 **해결하는 문제 자체가 다릅니다**.

| 비교 항목 | SDP | Apache Airflow |
|-----------|-----|---------------|
| **역할** | 데이터 **변환(Transformation)** 프레임워크 | 워크플로 **오케스트레이션(Orchestration)** 도구 |
| **추상화 수준** | "무엇을(What)" 만들지 선언 | "어떻게(How)" 실행할지 명령 |
| **데이터 품질** | Expectations 내장 | 별도 프레임워크 필요 (Great Expectations 등) |
| **증분 처리** | 자동 (체크포인트, 스키마 진화) | 수동 구현 (변경 감지 로직 직접 작성) |
| **의존성 관리** | SQL/Python 참조에서 자동 추론 | DAG에서 명시적으로 정의 |
| **스케일링** | 자동 (Enhanced Autoscaling) | Executor 설정 수동 관리 |
| **비용 모델** | Serverless DBU (사용한 만큼) | Airflow 인프라 상시 운영 비용 |
| **모니터링** | 이벤트 로그, 시스템 테이블 내장 | Airflow UI + 외부 모니터링 도구 |
| **외부 시스템 연동** | 제한적 (Databricks 생태계 중심) | 풍부한 Operator/Hook 생태계 |

**실무에서의 올바른 조합:**

```
SDP (변환) + Lakeflow Jobs (오케스트레이션) = Databricks 네이티브 스택
SDP (변환) + Airflow (오케스트레이션) = 하이브리드 스택 (기존 Airflow 투자 보존)
```

> 💡 **SA 관점**: 고객이 이미 Airflow를 사용 중이라면, Airflow에서 SDP 파이프라인을 트리거하는 하이브리드 패턴을 권장합니다. `DatabricksSubmitRunOperator`로 SDP 파이프라인 업데이트를 시작할 수 있습니다. 장기적으로는 Lakeflow Jobs로 마이그레이션하면 관리 포인트가 줄어듭니다.

---

## Enhanced Autoscaling 동작 원리

SDP는 일반 Spark 클러스터의 오토스케일링과 다른 **Enhanced Autoscaling** 알고리즘을 사용합니다. 이 차이를 이해하면 비용 최적화에 큰 도움이 됩니다.

### 일반 Spark 오토스케일링 vs Enhanced Autoscaling

| 비교 항목 | 일반 Spark Autoscaling | SDP Enhanced Autoscaling |
|-----------|----------------------|-------------------------|
| **스케일 업 기준** | 대기 중인 태스크 수 | 파이프라인 DAG 분석 + 데이터 볼륨 예측 |
| **스케일 다운** | 유휴 Worker 감지 (보수적) | Flow 완료 시 즉시 축소 (공격적) |
| **파이프라인 인식** | 없음 (태스크 단위) | 있음 (Flow/테이블 단위) |
| **비용 효율** | 보통 | 높음 (유휴 시간 최소화) |

### Enhanced Autoscaling의 핵심 동작

| Phase | 처리 내용 | Worker 할당 |
|-------|----------|------------|
| Phase 1 | Bronze 테이블 처리 | 데이터 양 분석 → 10대 할당 |
| Phase 2 | Silver 테이블 처리 | 의존성 충족 대기 → 5대로 축소 |
| Phase 3 | Gold 테이블 처리 | 집계 중심 → 3대로 축소 |
| 완료 | - | 모든 Worker 해제 |

Enhanced Autoscaling은 각 Flow의 **데이터 볼륨과 처리 복잡도를 사전 분석**하여 필요한 만큼만 Worker를 할당합니다. 일반 오토스케일링처럼 "느린 반응 → 과다 할당 → 느린 축소" 패턴이 발생하지 않습니다.

> 💡 **비용 영향**: Enhanced Autoscaling은 동일 워크로드 대비 일반 Spark 클러스터보다 **20~40% 낮은 DBU 소비**를 보이는 것이 일반적입니다. 특히 다수의 Flow가 순차적으로 실행되는 파이프라인에서 차이가 두드러집니다.

---

## Triggered vs Continuous 모드 — 성능/비용 심층 분석

### Triggered 모드 상세

| 속성 | 설명 |
|------|------|
| **동작** | 업데이트 요청 시 모든 Flow를 한 번 실행하고 종료합니다 |
| **리소스** | 실행 중에만 컴퓨트 할당, 완료 후 해제됩니다 |
| **지연 시간** | 스케줄 간격 + 컴퓨트 시작 시간 (서버리스: ~10초, Classic: 5~10분) |
| **비용 패턴** | 실행 시에만 과금. 간헐적 워크로드에 경제적입니다 |
| **적합한 경우** | 시간/일 단위 배치 ETL, 비용 민감한 환경, 데이터 신선도 요구가 낮은 경우 |

### Continuous 모드 상세

| 속성 | 설명 |
|------|------|
| **동작** | 클러스터가 상시 실행되며, 새 데이터가 도착하면 즉시 처리합니다 |
| **리소스** | 최소 Worker가 항상 할당되어 있습니다 |
| **지연 시간** | 초~분 단위 (데이터 도착 후 거의 즉시 처리) |
| **비용 패턴** | 24/7 과금. 유휴 시간에도 최소 비용이 발생합니다 |
| **적합한 경우** | 실시간 대시보드, IoT 이벤트 처리, 저지연 요구사항 |

### 비용 시뮬레이션 예시

```
시나리오: 일 4회 실행, 회당 15분 소요, 10 DBU/시간

Triggered 모드:
  4회 × 15분 × 10 DBU/시간 = 10 DBU/일 = 300 DBU/월

Continuous 모드:
  24시간 × 10 DBU/시간 = 240 DBU/일 = 7,200 DBU/월
  (→ Triggered 대비 24배 비용!)

손익 분기점: 하루 약 16회 이상 실행 시 Continuous가 유리
  (단, 실시간 처리의 비즈니스 가치를 별도 평가해야 합니다)
```

### `pipelines.trigger.interval` 설정

Triggered 모드에서 이 설정은 **파이프라인 내부 마이크로배치 간격**을 제어합니다. Continuous 모드에서는 새 데이터 폴링 간격을 의미합니다.

```yaml
configuration:
  "pipelines.trigger.interval": "5 minutes"   # 5분마다 새 데이터 확인
```

> ⚠️ **Gotcha**: `pipelines.trigger.interval`은 Lakeflow Jobs의 스케줄과 다릅니다. Jobs 스케줄은 "파이프라인 업데이트를 언제 시작할지"를 제어하고, `trigger.interval`은 "실행 중인 파이프라인 내에서 얼마나 자주 새 데이터를 확인할지"를 제어합니다.

---

## 서버리스 SDP — 장점과 한계

### 서버리스 SDP의 장점

| 장점 | 설명 |
|------|------|
| **즉시 시작** | 클러스터 프로비저닝 대기 없이 ~10초 이내 시작합니다 |
| **자동 스케일링** | Enhanced Autoscaling이 기본 적용됩니다 |
| **제로 관리** | DBR 버전, 라이브러리, 패치 관리가 불필요합니다 |
| **비용 효율** | 실행 시간에 대해서만 정확히 과금됩니다 |

### 서버리스 SDP의 한계와 주의사항

| 한계 | 상세 설명 | 대안 |
|------|----------|------|
| **커스텀 라이브러리 제한** | PyPI 패키지만 설치 가능, C 확장이 있는 라이브러리는 제한적 | Classic 클러스터 + init script |
| **Init Script 미지원** | 클러스터 초기화 스크립트를 사용할 수 없습니다 | Classic 클러스터 사용 |
| **네트워크 제한** | 외부 네트워크 접근이 제한될 수 있습니다 (VPC/VNet Peering 불가) | Classic + VPC Peering |
| **GPU 미지원** | GPU 인스턴스를 지정할 수 없습니다 | Classic 클러스터 + GPU |
| **리전 제한** | 일부 리전에서는 서버리스가 미지원입니다 | Classic 클러스터 사용 |
| **DBU 단가** | 서버리스 DBU 단가가 Classic보다 높습니다 (~1.5~2배) | 장시간 실행 시 Classic이 저렴할 수 있음 |
| **실행 시간 제한** | 단일 업데이트의 최대 실행 시간이 48시간입니다 | 워크로드 분할 |

### Classic vs Serverless SDP 선택 가이드

```
서버리스 SDP를 선택하세요:
  ✅ 파이프라인 실행 시간이 짧은 경우 (< 2시간)
  ✅ 간헐적으로 실행되는 배치 ETL
  ✅ 특별한 라이브러리/네트워크 요구사항이 없는 경우
  ✅ 빠른 개발 반복이 필요한 경우

Classic SDP를 선택하세요:
  ✅ 외부 네트워크 접근이 필요한 경우 (DB 직접 연결 등)
  ✅ GPU가 필요한 ML 파이프라인
  ✅ 커스텀 C 라이브러리가 필요한 경우
  ✅ 매우 긴 실행 시간 (> 8시간, Spot 인스턴스 활용 시 비용 유리)
```

---

## 엔터프라이즈 패턴과 실무 가이드

### 파이프라인 분리 전략

대규모 환경에서는 파이프라인을 어떻게 분리할지가 중요한 아키텍처 결정입니다.

| 패턴 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **단일 파이프라인** | 모든 테이블을 하나의 파이프라인에서 관리 | 의존성 관리 용이 | 장애 시 전체 영향, 대규모에서 느림 |
| **레이어별 분리** | Bronze/Silver/Gold를 별도 파이프라인으로 분리 | 독립적 스케줄, 장애 격리 | 파이프라인 간 의존성 관리 필요 |
| **도메인별 분리** | 비즈니스 도메인(주문, 고객, 재고)별 파이프라인 | 팀별 독립 운영, 확장 용이 | 도메인 간 조인이 필요한 Gold 처리 복잡 |
| **하이브리드** | 수집(Bronze)은 통합, 변환(Silver/Gold)은 도메인별 | 균형 잡힌 접근 | 설계 복잡도 증가 |

> 💡 **SA 권장 패턴**: 대부분의 엔터프라이즈 고객에게는 **하이브리드 패턴**을 권장합니다. Bronze 레이어는 중앙 데이터 엔지니어링 팀이 관리하고, Silver/Gold는 각 도메인 팀이 독립적으로 운영하는 구조가 가장 효과적입니다.

### Expectations 고급 활용

```sql
-- 단일 규칙: NULL 체크
CONSTRAINT valid_id EXPECT (order_id IS NOT NULL) ON VIOLATION DROP ROW

-- 비율 기반 규칙: 95% 이상 통과해야 함
CONSTRAINT valid_amount EXPECT (amount > 0) ON VIOLATION DROP ROW

-- 치명적 규칙: 위반 시 파이프라인 실패
CONSTRAINT critical_date EXPECT (order_date IS NOT NULL) ON VIOLATION FAIL UPDATE

-- 복합 규칙
CONSTRAINT valid_status EXPECT (status IN ('pending', 'completed', 'cancelled'))
    ON VIOLATION DROP ROW
```

**Expectations 전략:**

| 전략 | 적용 위치 | ON VIOLATION |
|------|----------|-------------|
| **데이터 존재성** | Bronze → Silver | DROP ROW (불량 데이터 제거) |
| **비즈니스 규칙** | Silver → Gold | DROP ROW (비즈니스 로직 위반 제거) |
| **치명적 이상** | 전체 레이어 | FAIL UPDATE (파이프라인 중단) |
| **통계적 이상** | Gold | 별도 모니터링 (Lakehouse Monitor) |

### 성능 최적화 팁

1. **소스 파일 수 최적화**: 파이프라인당 소스 파일을 50개 이하로 유지합니다. 파일이 많으면 DAG 분석 시간이 증가합니다.
2. **Streaming Table 우선 사용**: 가능하면 Materialized View 대신 Streaming Table을 사용합니다. 증분 처리가 훨씬 효율적입니다.
3. **Photon 활성화**: SDP는 Photon 엔진과 결합 시 SQL 변환 성능이 2~5배 향상됩니다.
4. **APPLY CHANGES 최적화**: CDC 처리 시 `KEYS`를 적절히 지정하고, `SEQUENCE BY`를 사용하여 순서를 보장합니다.
5. **Partition Pruning**: 날짜 기반 파티셔닝을 적용하면 증분 처리 효율이 높아집니다.

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
| **Enhanced Autoscaling** | DAG 인식 기반 자동 스케일링으로 비용을 최적화합니다 |
| **서버리스 SDP** | 즉시 시작, 제로 관리이지만 네트워크/라이브러리 제약이 있습니다 |

---

## 참고 링크

- [Databricks: Spark Declarative Pipelines](https://docs.databricks.com/aws/en/sdp/)
- [Databricks: SDP SQL reference](https://docs.databricks.com/aws/en/sdp/sql-ref.html)
- [Databricks: SDP Python reference](https://docs.databricks.com/aws/en/sdp/python-ref.html)
- [Databricks: Pipeline limitations](https://docs.databricks.com/aws/en/sdp/limitations.html)
- [Azure Databricks: Delta Live Tables](https://learn.microsoft.com/en-us/azure/databricks/delta-live-tables/)
