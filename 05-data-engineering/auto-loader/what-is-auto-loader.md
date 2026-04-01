# Auto Loader란?

## 개념

> 💡 **Auto Loader** 는 클라우드 스토리지(S3, ADLS, GCS)에 ** 새로 도착한 파일을 자동으로 감지하고 수집** 하는 Databricks의 증분 파일 수집 기능입니다. Spark Structured Streaming의 `cloudFiles` 소스 형식으로 구현되어 있습니다.

비유하자면, 우편함을 계속 지켜보다가 새 편지가 도착하면 자동으로 꺼내서 정리해 주는 비서와 같습니다. 이미 읽은 편지는 다시 읽지 않고, 새 편지만 처리합니다.

---

## 왜 Auto Loader가 필요한가요?

일반적인 `spark.read`로 파일을 읽으면 매번 ** 모든 파일을 처음부터 다시 읽습니다**. 파일이 수천~수만 개로 늘어나면 매번 전체를 스캔하는 것은 매우 비효율적입니다.

| 비교 | spark.read (일반) | COPY INTO | Auto Loader |
|------|-------------------|-----------|-------------|
| 파일 감지 | 매번 전체 디렉토리 스캔 | 매번 전체 디렉토리 스캔 | 새 파일만 자동 감지 |
| 중복 처리 | 매번 모든 파일을 읽음 | 이미 처리한 파일 건너뜀 | 이미 처리한 파일 건너뜀 |
| 스키마 변경 | 수동 대응 필요 | 수동 대응 필요 | 자동 감지 및 진화 가능 |
| 확장성 | 파일 수 증가 시 성능 저하 | 파일 수 증가 시 성능 저하 | ** 수십억 파일도 효율적** |
| 처리 보장 | 보장 없음 | Exactly-Once | **Exactly-Once**|
| 권장 여부 | ❌ 파일 수집에 부적합 | 소규모 일회성 적재 | ✅ 대부분의 파일 수집 |

> 💡 **Exactly-Once 처리 보장**: Auto Loader는 RocksDB 기반 체크포인트에 처리 상태를 기록하여, 각 파일이 정확히 한 번만 처리되도록 보장합니다. 장애가 발생해도 체크포인트에서 재시작하여 중복이나 유실이 발생하지 않습니다.

---

## 지원 스토리지 소스

| 스토리지 | 설명 |
|----------|------|
| **Amazon S3**| AWS의 오브젝트 스토리지입니다 |
| **Azure Data Lake Storage (ADLS Gen2)**| Azure의 데이터 레이크 스토리지입니다 |
| **Azure Blob Storage**| Azure의 범용 오브젝트 스토리지입니다 |
| **Google Cloud Storage (GCS)**| GCP의 오브젝트 스토리지입니다 |
| **Unity Catalog Volumes**| Databricks 관리형 스토리지입니다 |
| **DBFS**| Databricks File System입니다 (레거시) |

---

## 지원 파일 포맷

| 포맷 | 설명 | 적합한 데이터 |
|------|------|-------------|
| **JSON**| JavaScript Object Notation | 웹 로그, API 응답, 이벤트 데이터 |
| **CSV**| Comma Separated Values | 전통적인 텍스트 데이터, 엑셀 내보내기 |
| **Parquet**| 컬럼 기반 바이너리 포맷 | 분석용 대용량 데이터, 다른 시스템에서 내보낸 데이터 |
| **Avro**| Apache Avro 포맷 | Kafka 메시지, 스키마 레지스트리 데이터 |
| **ORC**| Optimized Row Columnar | Hive 생태계 데이터 |
| **XML**| Extensible Markup Language | 레거시 시스템 데이터 |
| **Text**| 일반 텍스트 파일 | 로그 파일, 비구조화 텍스트 |
| **Binary**| 바이너리 파일 | 이미지, PDF, 모델 파일 |

---

## 파일 감지 모드

Auto Loader는 두 가지 파일 감지 모드를 제공합니다.

### Directory Listing (기본값)

주기적으로 디렉토리를 스캔하여 새 파일을 찾습니다.

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.useNotifications", "false")  # 기본값
    .load("s3://my-bucket/data/")
)
```

| 항목 | 설명 |
|------|------|
| ** 동작 방식**| 디렉토리를 반복 스캔하여 이전에 없던 파일을 감지합니다 |
| ** 설정**| 별도 설정 없이 바로 사용 가능합니다 |
| ** 성능**| 디렉토리의 파일 수에 비례하여 스캔 비용이 증가합니다 |
| ** 적합한 경우**| 파일 수가 수만 개 이하인 경우 |
| ** 장점**| 설정이 간단하고, 클라우드 이벤트 인프라가 불필요합니다 |

### File Notification (File Events)

클라우드 이벤트 서비스(S3 Events, Azure Event Grid, GCS Pub/Sub)를 활용하여 새 파일 알림을 받습니다.

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.useNotifications", "true")
    .load("s3://my-bucket/data/")
)
```

| 항목 | 설명 |
|------|------|
| ** 동작 방식**| 클라우드 이벤트를 통해 새 파일 알림을 수신합니다 |
| ** 설정**| 클라우드 이벤트 서비스 설정이 필요합니다 (Auto Loader가 자동 생성 가능) |
| ** 성능**| 디렉토리 크기와 무관하게 일정한 성능을 제공합니다 |
| ** 적합한 경우**| 파일 수가 수백만 개 이상인 경우 |
| ** 장점**| 대규모에서도 감지 비용이 일정합니다 |

> 🆕 **Auto Loader with File Events (GA)**: 최근 GA된 새로운 모드로, File Notification의 효율성과 Directory Listing의 간편함을 결합했습니다. 별도의 클라우드 이벤트 설정 없이도 높은 성능을 제공합니다.

### 모드 선택 가이드

| 디렉토리 파일 수 | 권장 모드 | 설명 |
|-----------------|----------|------|
| 수만 개 이하 | Directory Listing (기본값) | 간편한 설정으로 시작할 수 있습니다 |
| 수십만~수백만 개 | File Events (GA, 권장) | 효율적인 파일 감지를 제공합니다 |
| 수백만 개 이상 + 극한 성능 필요 | File Notification | 클라우드 이벤트를 사용하여 최고 성능을 제공합니다 |

---

## 스키마 추론과 진화

Auto Loader의 가장 강력한 기능 중 하나는 ** 스키마를 자동으로 추론하고 변화에 대응** 하는 것입니다.

### 스키마 추론 (Schema Inference)

처음 수집할 때, Auto Loader가 파일 샘플을 읽어서 ** 자동으로 스키마(컬럼과 타입)를 추론** 합니다. 추론된 스키마는 `schemaLocation`에 저장되어, 이후 수집 시 재사용됩니다.

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")   # 타입까지 추론 (기본: 모두 STRING)
    .option("cloudFiles.schemaLocation", "/checkpoints/orders/schema")  # 스키마 저장 위치
    .load("s3://my-bucket/orders/")
)
```

### 스키마 진화 (Schema Evolution)

새 파일에 ** 새로운 컬럼이 추가** 되거나 ** 타입이 변경** 되면, Auto Loader가 이를 감지하고 대응합니다.

| 진화 모드 | 설명 |
|-----------|------|
| `addNewColumns` (기본) | 새 컬럼이 발견되면 스키마에 자동으로 추가합니다 |
| `rescue` | 새 컬럼을 추가하지 않고, 맞지 않는 데이터를 rescue 컬럼에 저장합니다 |
| `failOnNewColumns` | 새 컬럼이 발견되면 스트림을 중지합니다 |
| `none` | 스키마 진화를 비활성화합니다 |

```python
.option("cloudFiles.schemaEvolutionMode", "addNewColumns")
```

### 스키마 힌트 (Schema Hints)

자동 추론이 부정확한 경우, 특정 컬럼의 타입을 ** 수동으로 지정** 할 수 있습니다.

```python
.option("cloudFiles.schemaHints",
    "order_id BIGINT, amount DECIMAL(12,2), order_date TIMESTAMP, tags MAP<STRING,STRING>")
```

### Rescue Data Column

스키마에 맞지 않는 데이터를 별도 컬럼에 **JSON 형태로 보존** 하여 데이터 유실을 방지합니다.

```python
.option("rescuedDataColumn", "_rescued_data")
```

```
정상 데이터: order_id=1, amount=50000, _rescued_data=NULL
불량 데이터: order_id=NULL, amount=NULL, _rescued_data='{"order_id":"ABC","amount":"invalid"}'
```

> 💡 **SDP와의 통합**: SDP(Spark Declarative Pipelines) 안에서 Auto Loader를 사용하면, 스키마 위치와 체크포인트를 ** 수동으로 지정할 필요가 없습니다**. SDP가 자동으로 관리합니다. 이것이 Auto Loader를 SDP와 함께 사용하는 것을 권장하는 이유입니다.

---

## 사용 방법

### SQL 방식 (SDP 내에서 — 권장)

```sql
-- read_files() 함수를 사용 (SDP 전용)
CREATE OR REFRESH STREAMING TABLE bronze_orders
AS SELECT
    *,
    _metadata.file_path AS _source_file,
    _metadata.file_name AS _file_name,
    _metadata.file_size AS _file_size,
    _metadata.file_modification_time AS _file_modified_at,
    current_timestamp() AS _ingested_at
FROM STREAM read_files(
    's3://my-bucket/raw/orders/',
    format => 'json',
    inferColumnTypes => true,
    rescuedDataColumn => '_rescued_data'
);
```

### Python 방식 (독립 실행)

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaLocation", "/checkpoints/orders/schema")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("rescuedDataColumn", "_rescued_data")
    .load("s3://my-bucket/raw/orders/")
)

# 메타데이터 컬럼 추가
from pyspark.sql.functions import current_timestamp, input_file_name

enriched = (df
    .withColumn("_source_file", input_file_name())
    .withColumn("_ingested_at", current_timestamp())
)

# Delta 테이블로 저장
(enriched.writeStream
    .format("delta")
    .option("checkpointLocation", "/checkpoints/orders/data")
    .option("mergeSchema", "true")
    .outputMode("append")
    .trigger(availableNow=True)  # 현재 사용 가능한 파일만 처리 후 종료
    .toTable("catalog.schema.bronze_orders")
)
```

### Trigger 옵션

| Trigger | 설명 | 적합한 사용 |
|---------|------|-----------|
| `availableNow=True` | 현재 사용 가능한 모든 파일을 처리하고 종료합니다 | 스케줄된 배치 작업 |
| `processingTime='10 seconds'` | 10초 간격으로 새 파일을 확인하고 처리합니다 | 니어 리얼타임 수집 |
| `once=True` | 한 번 실행 후 종료합니다 (레거시) | 일회성 수집 |
| 없음 (Continuous) | 지속적으로 실행되며 새 파일을 즉시 처리합니다 | 실시간 수집 |

---

## 메타데이터 컬럼 (_metadata)

Auto Loader는 수집된 파일의 메타데이터를 자동으로 제공합니다.

| 컬럼 | 설명 |
|------|------|
| `_metadata.file_path` | 파일의 전체 경로 (예: `s3://bucket/path/file.json`) |
| `_metadata.file_name` | 파일 이름만 (예: `file.json`) |
| `_metadata.file_size` | 파일 크기 (바이트) |
| `_metadata.file_modification_time` | 파일의 최종 수정 시간 |
| `_metadata.file_block_start` | 파일 내 블록 시작 위치 |
| `_metadata.file_block_length` | 블록 길이 |

이 메타데이터는 Bronze 테이블에 함께 저장하면, 나중에 **어떤 파일에서 어떤 데이터가 왔는지** 추적하는 데 매우 유용합니다.

---

## Auto Loader vs COPY INTO 비교

| 비교 | Auto Loader | COPY INTO |
|------|-------------|-----------|
| ** 방식**| 스트리밍 (증분) | 배치 (전체 스캔 후 건너뛰기) |
| ** 확장성**| 수십억 파일 지원 | 파일 수 증가 시 느려짐 |
| ** 스키마 진화**| ✅ 자동 지원 | ❌ 미지원 |
| ** 처리 보장**| Exactly-Once (체크포인트) | Exactly-Once (파일 상태 테이블) |
| ** 실시간 수집**| ✅ 연속 실행 가능 | ❌ 배치만 |
| ** 권장**| ✅ 대부분의 경우 | 소규모, 일회성 적재 |

---

## 모범 사례

| 항목 | 권장 사항 |
|------|----------|
| **SDP와 함께 사용**| SDP 내의 `read_files()` 사용을 권장합니다. 스키마/체크포인트 자동 관리됩니다 |
| ** 스키마 힌트 활용**| 추론이 부정확한 핵심 컬럼에는 명시적 타입을 지정합니다 |
| **Rescue Data 활성화**| `rescuedDataColumn`을 설정하여 데이터 유실을 방지합니다 |
| ** 메타데이터 저장**| `_metadata` 컬럼을 Bronze 테이블에 포함하여 데이터 리니지를 확보합니다 |
| ** 파일 정리**| 처리 완료된 파일을 아카이브하거나 삭제하여 디렉토리 크기를 관리합니다 |
| ** 체크포인트 보존**| 체크포인트 디렉토리를 삭제하면 전체 파일이 재처리됩니다. 주의가 필요합니다 |

---

## 현업 사례: COPY INTO에서 Auto Loader로 전환한 후 파이프라인 안정성이 극적으로 개선된 사례

현업에서 Auto Loader의 가치를 가장 체감하는 순간은 "**COPY INTO에서 전환한 직후**" 입니다.

### 사례: 물류 회사의 IoT 센서 데이터 파이프라인

```
[기존 환경: COPY INTO]
- 데이터: 물류 차량 GPS 센서 데이터 (JSON)
- 파일 수: S3 디렉토리에 하루 50만 파일 생성
- 누적: 1년 후 디렉토리에 1.8억 파일

** 문제 발생 (COPY INTO)**:
- COPY INTO 실행 시 매번 1.8억 파일 디렉토리를 전체 스캔
- 파일 목록 조회만 45분 소요 (S3 LIST API 한계)
- 실제 처리할 신규 파일은 50만 개인데, 1.8억 개를 확인
- 일일 파이프라인이 4시간 → 점점 길어져 12시간까지 증가
- 간헐적으로 S3 LIST API 타임아웃 → 파이프라인 실패
- 실패 시 수동 재시작 → 새벽 3시 콜 빈발

**Auto Loader 전환 후**:
- 체크포인트에 마지막 처리 위치 기록
- 새 파일 50만 개만 감지하여 처리
- 디렉토리 크기와 무관하게 일정한 성능
- 일일 처리 시간: 35분 (12시간 → 35분)
- 파이프라인 실패: 월 평균 15회 → 0회
- 새벽 3시 콜: 사라짐
```

### 체크포인트의 진짜 가치

Auto Loader의 체크포인트는 단순한 "어디까지 읽었는지" 기록이 아닙니다. 현업에서의 진짜 가치를 설명합니다.

| 가치 | 설명 | COPY INTO와의 차이 |
|------|------|------------------|
| ** 장애 복구** | 파이프라인이 중간에 실패해도, 마지막 성공 지점부터 재시작 | COPY INTO는 전체를 다시 스캔해야 함 |
| ** 중복 방지** | 같은 파일을 두 번 처리하지 않음 (Exactly-Once) | COPY INTO도 중복 방지하지만 스캔 비용이 큼 |
| ** 점진적 확장** | 파일이 10억 개가 되어도 성능 일정 | COPY INTO는 파일 수에 비례하여 느려짐 |
| ** 스키마 이력** | 스키마 변경 이력이 체크포인트에 기록 | COPY INTO는 스키마 진화 미지원 |

> ⚠️ ** 체크포인트를 삭제하면 벌어지는 일**: 체크포인트 디렉토리를 삭제하면, Auto Loader는 "처음부터 다시 시작"합니다. 1.8억 파일을 전부 다시 읽게 되며, 타겟 테이블에 중복 데이터가 적재됩니다. ** 체크포인트는 절대로 삭제하지 마세요.** 현업에서 이 실수를 하면 데이터 정합성 복구에 수일이 걸립니다.

### 수백만~수억 파일 처리 시 성능 특성

| 디렉토리 파일 수 | COPY INTO 스캔 시간 | Auto Loader (Directory Listing) | Auto Loader (File Events) |
|----------------|-------------------|-----------------------------|--------------------------|
| 1만 개 | 10초 | 10초 | 2초 |
| 100만 개 | 15분 | 3분 | 2초 |
| 1,000만 개 | 2시간+ | 20분 | 2초 |
| 1억 개 | 실패 (타임아웃) | 2시간 | 2초 |
| 10억 개 | 불가능 | 느림 (비권장) | **5초**|

> 💡 ** 현업에서는 이렇게 합니다**: 파일 수가 100만 개를 넘어가면 반드시 **File Events 모드** 로 전환합니다. S3 Event Notification이나 Azure Event Grid를 설정하면, Auto Loader가 자동으로 이벤트 큐를 구독하여 새 파일만 즉시 감지합니다. 초기 설정에 30분이 더 걸리지만, 이후 수년간 일정한 성능을 보장합니다.

### Auto Loader 운영 시 꼭 알아야 할 팁

| # | 팁 | 설명 |
|---|-----|------|
| 1 | **SDP와 함께 사용**| `read_files()` 함수를 사용하면 체크포인트/스키마 위치를 수동 관리할 필요 없음 |
| 2 | **rescuedDataColumn 반드시 설정**| 스키마에 맞지 않는 데이터를 보존. 나중에 원인 분석 가능 |
| 3 | **_metadata 컬럼 저장**| 데이터 리니지(어떤 파일에서 왔는가) 추적에 필수 |
| 4 | **availableNow=True 활용**| 스케줄 배치에서는 이 트리거를 사용하면 비용 최적화 (연속 실행 불필요) |
| 5 | ** 스키마 힌트로 핵심 타입 고정**| 자동 추론이 BIGINT를 STRING으로 잡는 경우가 있음. 핵심 ID 컬럼은 힌트 필수 |
| 6 | ** 파일 정리 전략 수립**| 처리 완료된 파일을 아카이브/삭제하여 S3 비용 관리 (체크포인트에 영향 없음) |

> ⚠️ ** 이것을 안 하면**: rescuedDataColumn을 설정하지 않으면, 스키마에 맞지 않는 데이터가 ** 조용히 무시** 됩니다. 소스 시스템이 JSON 구조를 변경했는데, 몇 주 후에야 "데이터가 빠져있다"는 것을 발견하게 됩니다. 항상 `_rescued_data` 컬럼을 모니터링하여 null이 아닌 행이 있으면 알림을 설정하세요.

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| ** 자동 파일 감지** | 새 파일만 자동으로 감지하여 중복 처리를 방지합니다 |
| **Exactly-Once**| 체크포인트를 통해 각 파일이 정확히 한 번만 처리됩니다 |
| ** 스키마 추론/진화**| 파일 구조 변경에 자동으로 대응합니다 |
| **Rescue Data**| 스키마에 맞지 않는 데이터도 유실 없이 보존합니다 |
| ** 수십억 파일 확장성**| File Events/Notification 모드로 대규모도 효율적으로 처리합니다 |
| **8가지 파일 포맷** | JSON, CSV, Parquet, Avro, ORC, XML, Text, Binary를 지원합니다 |

---

## 참고 링크

- [Databricks: Auto Loader](https://docs.databricks.com/aws/en/ingestion/auto-loader/)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html)
- [Databricks: Auto Loader schema inference](https://docs.databricks.com/aws/en/ingestion/auto-loader/schema.html)
- [Azure Databricks: Auto Loader](https://learn.microsoft.com/en-us/azure/databricks/ingestion/auto-loader/)
