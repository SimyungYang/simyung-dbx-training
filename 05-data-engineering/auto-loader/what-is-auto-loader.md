# Auto Loader란?

## 개념

> 💡 **Auto Loader**는 클라우드 스토리지(S3, ADLS, GCS)에 **새로 도착한 파일을 자동으로 감지하고 수집**하는 Databricks의 파일 수집 기능입니다.

비유하자면, 우편함을 계속 지켜보다가 새 편지가 도착하면 자동으로 꺼내서 정리해 주는 비서와 같습니다. 이미 읽은 편지는 다시 읽지 않고, 새 편지만 처리합니다.

---

## 왜 Auto Loader가 필요한가요?

일반적인 `spark.read`로 파일을 읽으면 매번 **모든 파일을 처음부터 다시 읽습니다**. 파일이 수천~수만 개로 늘어나면 매번 전체를 스캔하는 것은 매우 비효율적입니다.

| 비교 | spark.read (일반) | Auto Loader |
|------|-------------------|-------------|
| 파일 감지 | 매번 전체 디렉토리 스캔 | 새 파일만 자동 감지 |
| 중복 처리 | 매번 모든 파일을 읽음 | 이미 처리한 파일은 건너뜀 |
| 스키마 변경 | 수동 대응 필요 | 자동 감지 및 진화 가능 |
| 확장성 | 파일 수 증가 시 성능 저하 | 수백만 파일도 효율적 처리 |

---

## 파일 감지 방식

Auto Loader는 두 가지 파일 감지 모드를 제공합니다.

| 모드 | 설명 | 적합한 경우 |
|------|------|-----------|
| **Directory Listing** | 주기적으로 디렉토리를 스캔하여 새 파일을 찾습니다 (기본값) | 대부분의 경우에 적합합니다 |
| **File Notification** | 클라우드 이벤트(S3 Events, Event Grid)를 활용하여 새 파일 알림을 받습니다 | 파일 수가 매우 많을 때 (수백만 개 이상) |

> 🆕 **Auto Loader with File Events (GA)**: 최근 GA된 File Events 모드는 File Notification의 효율성과 Directory Listing의 간편함을 결합한 새로운 방식입니다. 별도의 클라우드 이벤트 설정 없이도 높은 성능을 제공합니다.

---

## 기본 사용법

### SQL 방식 (SDP에서 사용)

```sql
CREATE OR REFRESH STREAMING TABLE bronze_orders
AS SELECT *
FROM STREAM read_files(
  's3://my-bucket/orders/',
  format => 'json',
  inferColumnTypes => true
);
```

### Python 방식

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaLocation", "/checkpoints/orders/schema")
    .load("s3://my-bucket/orders/")
)

(df.writeStream
    .format("delta")
    .option("checkpointLocation", "/checkpoints/orders")
    .outputMode("append")
    .toTable("catalog.schema.bronze_orders")
)
```

---

## 지원 파일 포맷

| 포맷 | 설명 |
|------|------|
| **JSON** | 웹 로그, API 응답 등 |
| **CSV** | 전통적인 텍스트 데이터 |
| **Parquet** | 컬럼 기반 바이너리 포맷 |
| **Avro** | Kafka에서 자주 사용되는 포맷 |
| **ORC** | Hive 생태계에서 사용되는 포맷 |
| **Text** | 일반 텍스트 파일 (로그 등) |
| **Binary** | 이미지, PDF 등 바이너리 파일 |

---

## 스키마 추론과 진화

Auto Loader의 강력한 기능 중 하나는 **스키마를 자동으로 추론하고 변화에 대응**하는 것입니다.

| 기능 | 설명 |
|------|------|
| **스키마 추론** | 첫 번째 파일에서 자동으로 스키마(컬럼과 타입)를 추론합니다 |
| **스키마 진화** | 새 파일에 새로운 컬럼이 추가되면 자동으로 감지하고 스키마를 업데이트합니다 |
| **스키마 힌트** | 추론이 부정확한 경우, 특정 컬럼의 타입을 지정할 수 있습니다 |
| **Rescue Data Column** | 스키마에 맞지 않는 데이터를 별도 컬럼(`_rescued_data`)에 저장하여 데이터 유실을 방지합니다 |

```python
# 스키마 힌트와 Rescue Data Column 사용
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaHints", "amount DECIMAL(10,2), order_date TIMESTAMP")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("rescuedDataColumn", "_rescued_data")
    .load("s3://my-bucket/orders/")
)
```

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **자동 파일 감지** | 새 파일만 자동으로 감지하여 중복 처리를 방지합니다 |
| **스키마 추론/진화** | 파일 구조 변경에 자동으로 대응합니다 |
| **Rescue Data** | 스키마에 맞지 않는 데이터도 유실 없이 보존합니다 |
| **확장성** | 수백만 파일도 효율적으로 처리합니다 |

---

## 참고 링크

- [Databricks: Auto Loader](https://docs.databricks.com/aws/en/ingestion/auto-loader/)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html)
