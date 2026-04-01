# SQL Warehouse와 파이프라인 최적화

## 3. SQL Warehouse 성능 최적화

### 3.1 쿼리 프로필 분석

SQL Warehouse의 Query Profile에서 주목해야 할 핵심 지표입니다.

| 단계 | 확인 항목 | 최적화 방향 |
|------|----------|-----------|
| **Scan**| 읽은 바이트 수 vs 테이블 크기 | 비율 높으면 클러스터링 부족 |
| **Filter**| 필터 전후 행 수 비율 | 비율 높으면 프루닝 미작동 |
| **Join**| 조인 유형 (Broadcast vs SortMerge) | 작은 테이블은 Broadcast로 |
| **Aggregate**| 집계 전후 행 수 | Pre-aggregate 고려 |
| **Exchange (Shuffle)**| Shuffle 데이터량 | 조인 키 최적화, 클러스터링 |
| **Spill** | Disk Spill 발생 여부 | Warehouse 사이즈 업 |

### 3.2 인덱스 없이 성능 높이기

Databricks는 전통적인 B-Tree 인덱스 대신 다음 기법들로 성능을 달성합니다.

```sql
-- 1. Liquid Clustering: 쿼리 패턴에 맞는 데이터 배치
ALTER TABLE catalog.schema.orders
CLUSTER BY (order_date, customer_id);

-- 2. 통계 수집: 옵티마이저에게 정확한 정보 제공
ANALYZE TABLE catalog.schema.orders COMPUTE DELTA STATISTICS;

-- 3. Materialized View: 복잡한 집계 사전 계산
CREATE MATERIALIZED VIEW catalog.schema.mv_daily_orders AS
SELECT order_date, region, COUNT(*) AS order_count, SUM(amount) AS total
FROM catalog.schema.orders
GROUP BY order_date, region;

-- 4. 적절한 데이터 타입 사용
-- ❌ 나쁜 예: 날짜를 문자열로 저장
-- CREATE TABLE ... (order_date STRING, ...)
-- ✅ 좋은 예: 네이티브 날짜 타입 사용 (프루닝/통계 활용)
-- CREATE TABLE ... (order_date DATE, ...)
```

### 3.3 쿼리 캐싱 전략

```text
요청 → Result Cache 확인 → 히트? → 즉시 반환 (0초)
                        ↓ 미스
        Disk Cache 확인 → 히트? → 로컬 SSD에서 읽기 (빠름)
                        ↓ 미스
        리모트 스토리지 (S3/ADLS) 에서 읽기 (느림)
```

| 캐시 유형 | 유효 기간 | 무효화 조건 | 설정 방법 |
|----------|----------|-----------|----------|
| **Result Cache**| 기본 활성화 | 테이블 데이터 변경 시 자동 무효화 | 자동 (수동 비활성화 가능) |
| **Disk Cache**| Warehouse 재시작까지 | Warehouse 재시작 또는 만료 | Storage-optimized 인스턴스 자동 |

```sql
-- Result Cache 확인: 동일 쿼리 재실행 시 Query Profile에서
-- "Result Cache Hit" 표시 여부 확인

-- Disk Cache 상태 확인 (노트북에서)
-- spark.catalog.isCached("table_name")

-- Cache를 최대 활용하는 쿼리 패턴
-- ✅ 동일한 쿼리를 파라미터만 바꿔 실행 (Result Cache 히트율 향상)
-- ✅ 자주 조인하는 디멘션 테이블은 자동으로 Disk Cache에 로드
-- ❌ CURRENT_TIMESTAMP() 등 비결정적 함수 사용 시 Cache 무효화
```

---

## 4. 파이프라인 성능 최적화

### 4.1 SDP(Spark Declarative Pipelines) 처리량 최적화

```python
# SDP 파이프라인 성능 설정
import dlt

@dlt.table(
  comment="최적화된 실버 테이블",
  table_properties={
    "pipelines.autoOptimize.managed": "true",      # 자동 최적화
    "pipelines.autoOptimize.zOrderCols": "user_id"  # 자동 Z-ORDER (레거시)
  }
)
def silver_events():
    return (
        dlt.read_stream("bronze_events")
        .withWatermark("event_time", "1 hour")  # 늦은 데이터 처리 한도
        .dropDuplicatesWithinWatermark(["event_id"])  # 중복 제거
    )
```

| 설정 | 기본값 | 성능 팁 |
|------|-------|--------|
| **pipelines.trigger.interval**| `"once"` 또는 연속 | 배치: `"once"`, 실시간: `"5 seconds"` |
| **Serverless 스케일링**| 자동 | Enhanced Autoscaling이 워크로드에 맞게 자동 조절 |
| **테이블 속성**| 기본값 | Liquid Clustering 적용 시 읽기 성능 대폭 향상 |

### 4.2 Auto Loader 파일 감지 모드 선택

| 모드 | 동작 방식 | 장점 | 단점 | 적합한 상황 |
|------|----------|------|------|-----------|
| **Directory Listing**| 주기적 디렉토리 스캔 | 설정 간단, 추가 인프라 불필요 | 파일 수 증가 시 스캔 비용 증가 | 파일 수 < 100만, 간단한 구조 |
| **File Notification**| SNS/SQS 이벤트 기반 | 즉시 감지, 대규모에 효율적 | 클라우드 리소스 설정 필요 | 파일 수 > 100만, 실시간 요구 |

```python
# Directory Listing 모드 (기본, 간단한 설정)
df = (spark.readStream
  .format("cloudFiles")
  .option("cloudFiles.format", "json")
  .option("cloudFiles.schemaLocation", "/checkpoints/schema")
  .load("s3://bucket/incoming/")
)

# File Notification 모드 (대규모, 실시간)
df = (spark.readStream
  .format("cloudFiles")
  .option("cloudFiles.format", "json")
  .option("cloudFiles.useNotifications", "true")  # 이벤트 기반
  .option("cloudFiles.schemaLocation", "/checkpoints/schema")
  .load("s3://bucket/incoming/")
)
```

### 4.3 Structured Streaming 지연시간 최적화

| 지연시간 목표 | 트리거 설정 | 적합한 워크로드 |
|-------------|-----------|---------------|
| **< 1초**| `trigger(processingTime="0 seconds")` | 실시간 대시보드, 알림 |
| **1~10초**| `trigger(processingTime="5 seconds")` | 준실시간 분석 |
| **10초~5분**| `trigger(processingTime="1 minute")` | 마이크로 배치 ETL |
| **5분+** | `trigger(availableNow=True)` | 스케줄 기반 배치 |

```python
# 지연시간 최적화 설정 조합
(df.writeStream
  .trigger(processingTime="5 seconds")
  .option("checkpointLocation", "/checkpoints/my_stream")
  .option("maxFilesPerTrigger", "100")        # 트리거당 파일 수 제한
  .option("maxBytesPerTrigger", "50m")         # 트리거당 데이터량 제한
  .outputMode("append")
  .toTable("catalog.schema.realtime_events")
)
```

---
