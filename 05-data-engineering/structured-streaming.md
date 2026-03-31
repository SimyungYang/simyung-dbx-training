# Spark Structured Streaming 상세

## Structured Streaming이란?

> 💡 **Structured Streaming**은 Apache Spark의 **스트림 처리 엔진**으로, 끊임없이 들어오는 데이터를 마치 테이블에 계속 행이 추가되는 것처럼 처리합니다. 배치 처리와 동일한 DataFrame API를 사용하므로, 배치 코드를 거의 그대로 스트리밍으로 전환할 수 있습니다.

### 왜 Structured Streaming이 필요한가요?

전통적인 배치 처리는 데이터를 **일정 주기(예: 매시간, 매일)**로 모아서 처리합니다. 하지만 실시간 대시보드, 이상 거래 탐지, IoT 센서 모니터링 같은 시나리오에서는 데이터가 도착하는 즉시 처리해야 합니다.

| 처리 방식 | 지연 시간 | 적합한 시나리오 |
|-----------|----------|----------------|
| **배치 처리** | 분~시간 단위 | 일일 리포트, 월말 정산 |
| **마이크로 배치** | 초~분 단위 | 실시간 대시보드, 로그 분석 |
| **연속 처리 (Continuous)** | 밀리초 단위 | 이상 거래 탐지, 실시간 알림 |

Structured Streaming은 **마이크로 배치(기본)**와 **연속 처리** 모드를 모두 지원하여, 요구사항에 맞는 지연 시간을 선택할 수 있습니다.

---

## 핵심 개념: 무한 테이블 (Unbounded Table)

Structured Streaming의 핵심 아이디어는 **스트림 데이터를 끝없이 행이 추가되는 테이블**로 모델링하는 것입니다.

```
시간 →   t1        t2        t3
        ┌──────┐  ┌──────┐  ┌──────┐
입력     │ 행1  │  │ 행1  │  │ 행1  │
테이블   │ 행2  │  │ 행2  │  │ 행2  │
        │      │  │ 행3  │  │ 행3  │
        │      │  │ 행4  │  │ 행4  │
        │      │  │      │  │ 행5  │
        └──────┘  └──────┘  └──────┘
```

새로운 데이터가 도착할 때마다 입력 테이블에 행이 추가되고, Spark는 **증분(Incremental)**으로 쿼리를 실행하여 결과 테이블을 갱신합니다.

---

## readStream / writeStream API

### 스트림 읽기 (readStream)

```python
# Kafka에서 스트림 읽기
df_stream = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker1:9092,broker2:9092")
    .option("subscribe", "my-topic")
    .option("startingOffsets", "latest")
    .load()
)

# Delta 테이블에서 스트림 읽기 (CDC 등)
df_delta_stream = (
    spark.readStream
    .format("delta")
    .table("catalog.schema.source_table")
)

# Auto Loader로 클라우드 파일 스트림 읽기
df_autoloader = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/checkpoints/schema")
    .load("s3://my-bucket/raw-data/")
)
```

### 스트림 쓰기 (writeStream)

```python
# Delta 테이블로 스트림 쓰기
query = (
    df_stream
    .writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/checkpoints/my-stream")
    .toTable("catalog.schema.target_table")
)
```

### 출력 모드 (Output Mode)

| 모드 | 설명 | 사용 시나리오 |
|------|------|-------------|
| **append** (기본) | 새로 추가된 행만 출력합니다 | 로그 수집, 이벤트 적재 |
| **complete** | 전체 결과 테이블을 매번 출력합니다 | 집계 결과 (GROUP BY) |
| **update** | 변경된 행만 출력합니다 | 집계 결과의 증분 업데이트 |

---

## 트리거 모드 (Trigger)

트리거는 **얼마나 자주 새 데이터를 처리할지**를 결정합니다.

| 트리거 | 설명 | 코드 |
|--------|------|------|
| **processingTime** | 지정 간격마다 마이크로 배치 실행 | `.trigger(processingTime="10 seconds")` |
| **availableNow** | 현재 가용한 데이터를 모두 처리 후 종료 | `.trigger(availableNow=True)` |
| **continuous** | 밀리초 단위 연속 처리 (실험적) | `.trigger(continuous="1 second")` |
| **기본값 (미지정)** | 이전 배치 완료 즉시 다음 배치 시작 | 트리거 옵션 생략 |

```python
# 30초마다 마이크로 배치 실행
query = (
    df_stream.writeStream
    .format("delta")
    .trigger(processingTime="30 seconds")
    .option("checkpointLocation", "/checkpoints/stream1")
    .toTable("catalog.schema.output")
)

# 가용한 데이터 모두 처리 후 종료 (스케줄링된 잡에 적합)
query = (
    df_stream.writeStream
    .format("delta")
    .trigger(availableNow=True)
    .option("checkpointLocation", "/checkpoints/stream2")
    .toTable("catalog.schema.output")
)
```

> 💡 **availableNow vs 배치**: `availableNow`는 배치처럼 동작하지만, **체크포인트를 유지**하여 이전에 처리한 데이터를 다시 처리하지 않습니다. 정기적으로 Lakeflow Jobs로 스케줄링할 때 매우 유용합니다.

---

## 워터마크와 Late Data 처리

스트림 데이터는 네트워크 지연, 시스템 장애 등으로 인해 **늦게 도착**할 수 있습니다. 워터마크(Watermark)는 **얼마나 늦은 데이터까지 허용할지**를 정의합니다.

```python
from pyspark.sql.functions import window, col

# 이벤트 타임 기준, 최대 10분 늦은 데이터까지 허용
df_windowed = (
    df_stream
    .withWatermark("event_time", "10 minutes")
    .groupBy(
        window("event_time", "5 minutes"),  # 5분 윈도우
        "device_id"
    )
    .agg({"temperature": "avg", "humidity": "avg"})
)
```

### 워터마크 동작 원리

```
현재 최대 이벤트 시간: 10:30
워터마크 임계값:      10분
워터마크:            10:20

→ 10:20 이전 이벤트 타임을 가진 데이터는 '너무 늦음'으로 판단하여 무시
→ 10:20 이후 데이터는 정상 처리
```

| 워터마크 설정 | 효과 |
|-------------|------|
| 짧은 워터마크 (예: 1분) | 메모리 절약, 빠른 결과. 늦은 데이터 누락 가능 |
| 긴 워터마크 (예: 1시간) | 늦은 데이터 수용. 메모리 사용량 증가, 결과 지연 |

---

## 스트림 조인

### 스트림-스태틱 조인

스트림 데이터와 **정적 테이블(Dimension 테이블)**을 조인합니다. 가장 일반적인 패턴입니다.

```python
# 스트림: 주문 이벤트
orders_stream = spark.readStream.table("catalog.schema.orders_raw")

# 스태틱: 고객 정보 (차원 테이블)
customers = spark.read.table("catalog.schema.customers")

# 조인: 주문에 고객 정보 추가
enriched_orders = orders_stream.join(
    customers,
    orders_stream.customer_id == customers.customer_id,
    "left"
)
```

### 스트림-스트림 조인

두 개의 스트림을 조인합니다. **워터마크가 필수**이며, 시간 범위 조건을 지정해야 합니다.

```python
# 스트림 1: 광고 노출
impressions = (
    spark.readStream
    .table("catalog.schema.impressions")
    .withWatermark("impression_time", "2 hours")
)

# 스트림 2: 광고 클릭
clicks = (
    spark.readStream
    .table("catalog.schema.clicks")
    .withWatermark("click_time", "3 hours")
)

# 조인: 노출 후 1시간 이내에 발생한 클릭 매칭
matched = impressions.join(
    clicks,
    (impressions.ad_id == clicks.ad_id) &
    (clicks.click_time >= impressions.impression_time) &
    (clicks.click_time <= impressions.impression_time + expr("INTERVAL 1 HOUR")),
    "leftOuter"
)
```

---

## 체크포인트 관리

체크포인트는 스트리밍 쿼리의 **진행 상태(오프셋, 상태 정보)**를 저장하여, 장애 발생 시 **마지막 처리 지점부터 재개**할 수 있게 합니다.

```python
query = (
    df_stream.writeStream
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/my-pipeline/stream1")
    .toTable("catalog.schema.output")
)
```

| 체크포인트 관리 규칙 | 설명 |
|-------------------|------|
| **1 쿼리 = 1 체크포인트** | 각 스트리밍 쿼리는 고유한 체크포인트 경로를 사용해야 합니다 |
| **체크포인트 삭제 주의** | 삭제하면 처음부터 다시 처리합니다. 의도하지 않은 중복이 발생할 수 있습니다 |
| **경로 변경 금지** | 같은 쿼리의 체크포인트 경로를 중간에 변경하면 안 됩니다 |
| **클라우드 스토리지 권장** | DBFS 대신 S3/ADLS/GCS에 저장하여 내구성을 확보합니다 |

---

## 외부 소스 연동

### Kafka / Confluent Cloud

```python
df_kafka = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "pkc-xxxxx.us-east-1.aws.confluent.cloud:9092")
    .option("kafka.security.protocol", "SASL_SSL")
    .option("kafka.sasl.mechanism", "PLAIN")
    .option("kafka.sasl.jaas.config",
        'org.apache.kafka.common.security.plain.PlainLoginModule required '
        'username="API_KEY" password="API_SECRET";')
    .option("subscribe", "orders")
    .option("startingOffsets", "earliest")
    .load()
)

# Kafka value를 JSON으로 파싱
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StringType, DoubleType

schema = StructType() \
    .add("order_id", StringType()) \
    .add("amount", DoubleType()) \
    .add("timestamp", StringType())

df_parsed = df_kafka.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")
```

### Amazon Kinesis

```python
df_kinesis = (
    spark.readStream
    .format("kinesis")
    .option("streamName", "my-stream")
    .option("region", "us-east-1")
    .option("initialPosition", "TRIM_HORIZON")
    .load()
)
```

### Azure Event Hubs

```python
connection_string = "Endpoint=sb://my-namespace.servicebus.windows.net/;..."
eh_conf = {
    "eventhubs.connectionString": sc._jvm.org.apache.spark.eventhubs
        .EventHubsUtils.encrypt(connection_string)
}

df_eh = (
    spark.readStream
    .format("eventhubs")
    .options(**eh_conf)
    .load()
)
```

---

## foreachBatch 패턴

`foreachBatch`를 사용하면 **각 마이크로 배치에 대해 임의의 DataFrame 연산**을 수행할 수 있습니다. MERGE(Upsert), 다중 출력 등 복잡한 로직에 적합합니다.

```python
def upsert_to_delta(batch_df, batch_id):
    """각 마이크로 배치를 Delta 테이블에 MERGE (Upsert)"""
    from delta.tables import DeltaTable

    target = DeltaTable.forName(spark, "catalog.schema.customers")

    (target.alias("t")
        .merge(batch_df.alias("s"), "t.customer_id = s.customer_id")
        .whenMatchedUpdateAll()
        .whenNotMatchedInsertAll()
        .execute()
    )

# foreachBatch로 Upsert 스트리밍
query = (
    df_stream.writeStream
    .foreachBatch(upsert_to_delta)
    .option("checkpointLocation", "/checkpoints/upsert-stream")
    .trigger(processingTime="1 minute")
    .start()
)
```

### foreachBatch 활용 사례

| 패턴 | 설명 |
|------|------|
| **MERGE (Upsert)** | 키 기반으로 업데이트 또는 삽입합니다 |
| **다중 출력** | 하나의 스트림을 여러 테이블에 동시에 씁니다 |
| **외부 시스템 연동** | REST API 호출, DB 직접 쓰기 등을 수행합니다 |
| **커스텀 검증** | 배치별 데이터 품질 검사를 실행합니다 |

---

## 모니터링 (StreamingQueryListener)

### 기본 쿼리 상태 확인

```python
# 실행 중인 스트리밍 쿼리 목록
for q in spark.streams.active:
    print(f"쿼리: {q.name}, 상태: {q.status}")

# 특정 쿼리의 진행 상황 확인
query.lastProgress  # 마지막 마이크로 배치 정보
query.recentProgress  # 최근 몇 개 배치의 진행 정보
```

### StreamingQueryListener를 활용한 커스텀 모니터링

```python
from pyspark.sql.streaming import StreamingQueryListener

class MyListener(StreamingQueryListener):
    def onQueryStarted(self, event):
        print(f"쿼리 시작: {event.name} (ID: {event.id})")

    def onQueryProgress(self, event):
        progress = event.progress
        print(f"처리 행: {progress.numInputRows}, "
              f"처리 시간: {progress.batchDuration}ms, "
              f"입력 속도: {progress.inputRowsPerSecond} rows/sec")

    def onQueryTerminated(self, event):
        if event.exception:
            print(f"쿼리 오류 종료: {event.exception}")
        else:
            print("쿼리 정상 종료")

# 리스너 등록
spark.streams.addListener(MyListener())
```

### 주요 모니터링 메트릭

| 메트릭 | 설명 | 주의 기준 |
|--------|------|----------|
| `inputRowsPerSecond` | 초당 입력 행 수 | 급격한 증가/감소 시 확인 |
| `processedRowsPerSecond` | 초당 처리 행 수 | 입력보다 낮으면 처리 지연 발생 |
| `numInputRows` | 배치당 입력 행 수 | 0이면 새 데이터 없음 |
| `batchDuration` | 배치 처리 시간 (ms) | 트리거 간격보다 길면 병목 |
| `stateOperators.numRowsTotal` | 상태 저장소 크기 | 지속 증가 시 메모리 부족 위험 |

---

## 모범 사례

| 항목 | 권장 사항 |
|------|----------|
| **체크포인트** | 클라우드 스토리지에 저장하고, 쿼리별로 고유 경로를 사용합니다 |
| **워터마크** | 상태 기반 연산(윈도우 집계, 스트림 조인)에는 반드시 워터마크를 설정합니다 |
| **트리거 선택** | 실시간이 필요하면 `processingTime`, 스케줄링이면 `availableNow`를 사용합니다 |
| **스키마 관리** | Kafka JSON 등 스키마 없는 소스는 명시적 스키마를 정의합니다 |
| **모니터링** | StreamingQueryListener로 처리 지연을 감시합니다 |
| **Auto Loader 활용** | 파일 기반 스트리밍은 Auto Loader를 우선 사용합니다 |
| **SDP 연동** | 복잡한 파이프라인은 SDP(Spark Declarative Pipelines)를 고려합니다 |

---

## 정리

| 개념 | 핵심 내용 |
|------|----------|
| Structured Streaming | 스트림을 무한 테이블로 모델링하여 DataFrame API로 처리합니다 |
| readStream / writeStream | 배치 API와 동일한 패턴으로 스트림을 읽고 씁니다 |
| 트리거 모드 | processingTime(주기적), availableNow(일회성), continuous(실시간) |
| 워터마크 | 늦은 데이터 허용 범위를 정의하여 상태를 관리합니다 |
| foreachBatch | 마이크로 배치 단위로 MERGE 등 복잡한 로직을 실행합니다 |
| 체크포인트 | 장애 복구를 위해 처리 진행 상태를 저장합니다 |

---

## 참고 링크

- [Structured Streaming 공식 문서](https://docs.databricks.com/aws/en/structured-streaming/)
- [Structured Streaming 프로그래밍 가이드 (Apache Spark)](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [Auto Loader 공식 문서](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)
- [Kafka 연동 가이드](https://docs.databricks.com/aws/en/structured-streaming/kafka.html)
- [foreachBatch를 활용한 스트리밍 Upsert](https://docs.databricks.com/aws/en/structured-streaming/foreach.html)
