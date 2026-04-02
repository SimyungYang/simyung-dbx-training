# Structured Streaming 심화

## 윈도우 유형 (Window Types)

스트리밍 집계에서 **시간 기반 윈도우** 는 데이터를 시간 구간별로 그룹화하는 핵심 메커니즘입니다. Structured Streaming은 3가지 윈도우 유형을 지원합니다.

### Tumbling Window (고정 윈도우)

**겹치지 않는 고정 크기** 윈도우입니다. 각 이벤트는 정확히 하나의 윈도우에만 속합니다.

```python
from pyspark.sql.functions import window

# 5분 단위 고정 윈도우
df.withWatermark("event_time", "10 minutes") \
  .groupBy(window("event_time", "5 minutes"), "device_id") \
  .agg(avg("temperature").alias("avg_temp"))

# 결과 윈도우: [10:00-10:05), [10:05-10:10), [10:10-10:15), ...
```

| 항목 | 설명 |
|------|------|
| **사용 시기** | 5분/1시간/일별 집계 등 비중복 구간 통계 |
| **장점** | 단순하고 직관적, 메모리 효율적 |
| **단점** | 윈도우 경계에서 데이터가 분리됨 (10:04:59와 10:05:01이 다른 윈도우) |

### Sliding Window (슬라이딩 윈도우)

**윈도우가 겹치는** 구간 집계입니다. 하나의 이벤트가 여러 윈도우에 속할 수 있습니다.

```python
# 10분 윈도우, 5분마다 슬라이딩
df.withWatermark("event_time", "10 minutes") \
  .groupBy(window("event_time", "10 minutes", "5 minutes"), "device_id") \
  .agg(avg("temperature").alias("avg_temp"))

# 결과 윈도우: [10:00-10:10), [10:05-10:15), [10:10-10:20), ...
# → 각 이벤트가 2개 윈도우에 포함됨
```

| 항목 | 설명 |
|------|------|
| **사용 시기** | 이동 평균, 연속적인 이상 감지 |
| **장점** | 윈도우 경계 문제가 완화됨 |
| **단점** | Tumbling 대비 메모리 사용량 증가 (윈도우 수 = 윈도우 크기 / 슬라이드 간격) |

### Session Window (세션 윈도우)

**활동 간 간격(gap)** 기반으로 동적 크기의 윈도우를 생성합니다. 이벤트 간 gap이 임계값을 초과하면 새 세션이 시작됩니다.

```python
from pyspark.sql.functions import session_window

# 30분 이상 활동이 없으면 세션 종료
df.withWatermark("event_time", "1 hour") \
  .groupBy(session_window("event_time", "30 minutes"), "user_id") \
  .agg(
      count("*").alias("page_views"),
      min("event_time").alias("session_start"),
      max("event_time").alias("session_end")
  )
```

| 항목 | 설명 |
|------|------|
| **사용 시기** | 사용자 세션 분석, 웹 클릭스트림, IoT 장비 활동 구간 |
| **장점** | 실제 활동 패턴에 맞는 자연스러운 그룹화 |
| **단점** | 세션 종료를 감지하려면 워터마크가 필수이며, 메모리 사용이 예측하기 어려움 |

### 윈도우 유형 비교

| 비교 | Tumbling | Sliding | Session |
|------|---------|---------|---------|
| **윈도우 크기** | 고정 | 고정 | 가변 (gap 기반) |
| **겹침** | 없음 | 있음 | 없음 |
| **이벤트 소속** | 1개 윈도우 | 여러 윈도우 | 1개 세션 |
| **메모리** | 낮음 | 중간 | 높음 (가변) |
| **워터마크 필수** | 권장 | 권장 | **필수** |
| **대표 사용** | 일별/시간별 통계 | 이동 평균 | 사용자 세션 |

---

## Output Mode 심화

writeStream의 `outputMode`는 **결과 테이블의 어떤 행을 싱크에 쓸 것인지** 를 결정합니다.

| Output Mode | 동작 | 사용 가능 조건 | 적합한 싱크 |
|------------|------|-------------|-----------|
| **Append**(기본) | 새로 추가된 행만 출력 | 집계 없는 쿼리, 또는 워터마크가 있는 집계 | Delta Table, Kafka |
| **Update** | 변경된 행만 출력 | 집계, mapGroupsWithState | Delta Table (MERGE와 결합) |
| **Complete** | 전체 결과 테이블 출력 | 집계 쿼리만 | 소규모 결과, 메모리 싱크 |

```python
# Append Mode: 새 행만 쓰기 (가장 일반적)
df.writeStream.outputMode("append").toTable("output")

# Update Mode: 변경된 행만 쓰기
df.groupBy("device_id").agg(avg("temp")) \
  .writeStream.outputMode("update").toTable("device_avg")

# Complete Mode: 전체 결과 덮어쓰기 (소규모만)
df.groupBy("status").count() \
  .writeStream.outputMode("complete") \
  .format("memory").queryName("status_count").start()
```

> ⚠️ **Complete Mode 주의**: 결과 테이블 전체를 매번 다시 쓰므로, **결과가 소규모(수천 행 이하)인 경우에만** 사용하세요. 대규모 결과에 사용하면 성능이 급격히 저하됩니다.

---

## Change Data Feed (CDF)

**Change Data Feed(CDF)** 는 Delta 테이블에서 발생한 행 수준 변경사항(INSERT, UPDATE, DELETE)을 스트리밍으로 읽을 수 있게 해주는 기능입니다. CDC 파이프라인의 핵심 기술입니다.

### CDF가 필요한 이유

일반적인 `readStream`으로 Delta 테이블을 읽으면 **새로 추가된 행(Append)만** 볼 수 있습니다. UPDATE나 DELETE는 감지할 수 없습니다. CDF를 활성화하면 **모든 변경 유형** 을 스트림으로 처리할 수 있습니다.

| 시나리오 | readStream (기본) | readStream + CDF |
|---------|------------------|------------------|
| INSERT 감지 | ✅ | ✅ |
| UPDATE 감지 | ❌ | ✅ (before/after 이미지) |
| DELETE 감지 | ❌ | ✅ |
| SCD Type 2 구현 | 불가 | 가능 |

### CDF 활성화

```sql
-- 테이블에 CDF 활성화
ALTER TABLE catalog.schema.customers
SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');

-- 새 테이블 생성 시 활성화
CREATE TABLE catalog.schema.orders (
    order_id BIGINT,
    customer_id BIGINT,
    amount DECIMAL(10,2),
    status STRING
)
TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');
```

### CDF 스트림 읽기

```python
# CDF가 활성화된 Delta 테이블에서 변경사항 읽기
changes_df = (
    spark.readStream
    .format("delta")
    .option("readChangeFeed", "true")      # CDF 활성화
    .option("startingVersion", 0)           # 또는 "startingTimestamp"
    .table("catalog.schema.customers")
)

# CDF 메타데이터 컬럼
# _change_type: "insert", "update_preimage", "update_postimage", "delete"
# _commit_version: 변경이 발생한 Delta 버전
# _commit_timestamp: 변경 시각

changes_df.select(
    "_change_type",
    "_commit_version",
    "customer_id",
    "name",
    "email"
).display()
```

### CDF 변경 유형

| `_change_type` | 설명 | 데이터 |
|---------------|------|--------|
| `insert` | 새 행 삽입 | 삽입된 행 |
| `update_preimage` | 업데이트 **이전** 값 | 변경 전 행 |
| `update_postimage` | 업데이트 **이후** 값 | 변경 후 행 |
| `delete` | 행 삭제 | 삭제된 행 |

### CDF 활용: SCD Type 2 파이프라인

```python
# CDF로 SCD Type 2 구현 (foreachBatch 패턴)
def upsert_to_scd2(batch_df, batch_id):
    from delta.tables import DeltaTable

    # UPDATE: 기존 행의 end_date를 설정 (이력 닫기)
    updates = batch_df.filter("_change_type = 'update_postimage'")

    target = DeltaTable.forName(spark, "catalog.schema.customers_scd2")

    target.alias("t").merge(
        updates.alias("s"),
        "t.customer_id = s.customer_id AND t.is_current = true"
    ).whenMatchedUpdate(set={
        "is_current": "false",
        "end_date": "s._commit_timestamp"
    }).whenNotMatchedInsert(values={
        "customer_id": "s.customer_id",
        "name": "s.name",
        "email": "s.email",
        "start_date": "s._commit_timestamp",
        "end_date": "null",
        "is_current": "true"
    }).execute()

# 스트림 실행
(changes_df
    .writeStream
    .foreachBatch(upsert_to_scd2)
    .option("checkpointLocation", "/checkpoints/scd2")
    .trigger(availableNow=True)
    .start()
)
```

### CDF와 다른 기능의 관계

| 기능 | CDF 필요 여부 | 설명 |
|------|-------------|------|
| **Online Table 동기화** | ✅ 필수 (TRIGGERED/CONTINUOUS) | Online Table이 Feature Table의 변경을 추적 |
| **SDP APPLY CHANGES** | ❌ 불필요 (내부 처리) | SDP는 자체 CDC 메커니즘 사용 |
| **Lakeflow Connect** | ❌ 불필요 | 소스 DB의 CDC를 직접 캡처 |
| **Materialized View 증분** | ✅ 활용 | MV가 소스 변경을 효율적으로 감지 |

> 💡 **CDF vs CDC**: CDF(Change Data Feed)는 **Delta 테이블 내부의 변경 추적** 기능이고, CDC(Change Data Capture)는 **외부 DB의 변경을 캡처** 하는 기술입니다. Lakeflow Connect가 외부 CDC를 캡처하면, 그 결과를 CDF가 활성화된 Delta 테이블에 저장하여 다운스트림에 전파하는 패턴이 일반적입니다.

---

## State Store 관리

윈도우 집계, 스트림 조인 등 **상태 기반 연산** 은 내부적으로 **State Store** 에 중간 상태를 저장합니다. 프로덕션에서 가장 흔한 스트리밍 문제는 **State Store 크기 증가** 입니다.

### State Store가 커지는 원인

| 원인 | 설명 | 해결 |
|------|------|------|
| **워터마크 미설정** | 오래된 상태가 정리되지 않음 | 반드시 `withWatermark()` 설정 |
| **워터마크가 너무 큼** | 1시간 워터마크 = 1시간치 상태 유지 | 비즈니스 요구 최소한으로 설정 |
| **높은 카디널리티 키** | user_id별 집계 시 수억 개 상태 | 키 카디널리티 검토, 사전 집계 |
| **Session Window** | 세션이 닫히기 전까지 상태 유지 | gap duration을 짧게 |

### State Store 모니터링

```python
# StreamingQueryListener로 상태 크기 모니터링
query = df.writeStream...start()

# 진행 상태에서 State Store 메트릭 확인
progress = query.lastProgress
if progress and progress.get("stateOperators"):
    for op in progress["stateOperators"]:
        print(f"State rows: {op['numRowsTotal']:,}")
        print(f"State memory: {op['memoryUsedBytes'] / 1024 / 1024:.1f} MB")
        print(f"Rows dropped (late): {op.get('numRowsDroppedByWatermark', 0):,}")
```

> ⚠️ **State Store OOM**: State Store가 메모리를 초과하면 RocksDB State Store Backend로 디스크에 spill됩니다. Databricks는 기본적으로 RocksDB를 사용하므로 OOM은 드물지만, **디스크 I/O 증가로 지연시간이 늘어날 수** 있습니다.

---

## 참고 링크

- [Structured Streaming 공식 문서](https://docs.databricks.com/aws/en/structured-streaming/)
- [Structured Streaming 프로그래밍 가이드 (Apache Spark)](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [Databricks: Change Data Feed](https://docs.databricks.com/aws/en/delta/delta-change-data-feed.html)
