# Delta Lake 성능 최적화

## 이 문서에서 다루는 내용

Databricks 플랫폼에서 쿼리, 파이프라인, ML 워크로드의 성능을 극대화하는 방법을 다룹니다. Delta Lake 데이터 레이아웃부터 Spark 쿼리 튜닝, SQL Warehouse 최적화, 파이프라인 처리량 개선, ML/AI 지연시간 최적화까지 실전 트러블슈팅 가이드를 제공합니다.

---

## 1. Delta Lake 성능 최적화

### 1.1 Liquid Clustering 키 선택 전략

Liquid Clustering은 기존의 파티셔닝과 Z-ORDER를 대체하는 현대적인 데이터 레이아웃 기술입니다. 올바른 키 선택이 성능을 좌우합니다.

**키 선택 의사결정 매트릭스**

| 고려 요소 | 좋은 키 | 나쁜 키 |
|----------|--------|--------|
| **쿼리 필터 빈도** | WHERE절에 자주 등장 | 거의 필터링되지 않음 |
| **카디널리티** | 중~고 (수천~수백만) | 극저 (boolean, 2-3값) 또는 극고 (UUID) |
| **상관관계** | 다른 키와 독립적 | 다른 키와 높은 상관 (둘 다 선택 불필요) |
| **데이터 타입** | Date, String, Integer | Array, Map, Struct |
| **키 개수** | 1~4개 | 5개 이상 (최대 4개 제한) |

```sql
-- 사례 1: 이커머스 트랜잭션 테이블
-- 쿼리 패턴: 날짜 범위 + 지역 필터가 대부분
ALTER TABLE sales.silver.transactions
CLUSTER BY (transaction_date, region);

-- 사례 2: IoT 센서 데이터
-- 쿼리 패턴: 디바이스별 + 시간 범위 조회
ALTER TABLE iot.silver.sensor_readings
CLUSTER BY (device_id, event_timestamp);

-- 사례 3: 쿼리 패턴이 자주 변경되는 테이블
-- Databricks가 자동으로 최적 키를 분석하여 결정
ALTER TABLE analytics.gold.user_behavior
CLUSTER BY AUTO;
```

**기존 테이블 마이그레이션 가이드**

| 현재 방식 | 마이그레이션 전략 |
|----------|----------------|
| `PARTITIONED BY (date)` | `CLUSTER BY (date)` — 파티션 키를 클러스터링 키로 |
| `ZORDER BY (col_a, col_b)` | `CLUSTER BY (col_a, col_b)` — Z-ORDER 키를 그대로 |
| 파티션 + Z-ORDER | `CLUSTER BY (partition_col, zorder_col)` — 둘 다 포함 |
| 없음 (최적화 안 함) | `CLUSTER BY AUTO` — 자동 분석 후 적용 |

> 💡 **핵심 장점**: Liquid Clustering은 키를 변경해도 **기존 데이터를 즉시 다시 쓸 필요가 없습니다**. 이후 OPTIMIZE 실행 시 점진적으로 새 키에 맞게 재배치됩니다.

### 1.2 파일 사이즈 최적화

Delta Lake의 최적 파일 크기는 **128MB ~ 1GB** 입니다. 이 범위를 벗어나면 성능이 저하됩니다.

| 문제 | 원인 | 증상 | 해결 방법 |
|------|------|------|----------|
| **Small File 문제** | 잦은 스트리밍 쓰기, 작은 배치 | 파일 수 과다, 메타데이터 오버헤드 | `OPTIMIZE` 실행 |
| **Large File 문제** | 과도한 OPTIMIZE, 큰 배치 | 프루닝 효과 감소, 메모리 압박 | 파일 크기 상한 설정 |

```sql
-- Small File 진단: 파일 수와 평균 크기 확인
DESCRIBE DETAIL catalog.schema.table_name;
-- numFiles, sizeInBytes를 확인하여 평균 파일 크기 계산
-- 평균 < 32MB면 OPTIMIZE 필요

-- OPTIMIZE로 파일 병합
OPTIMIZE catalog.schema.table_name;

-- 특정 파티션만 OPTIMIZE (대용량 테이블)
OPTIMIZE catalog.schema.table_name
WHERE event_date >= current_date() - INTERVAL 7 DAYS;
```

```python
# 스트리밍 쓰기 시 파일 크기 제어
(df.writeStream
  .option("maxBytesPerTrigger", "100m")  # 트리거당 최대 100MB 읽기
  .option("maxFilesPerTrigger", "1000")   # 트리거당 최대 파일 수
  .trigger(processingTime="30 seconds")   # 30초 간격 → 적당한 파일 크기
  .table("catalog.schema.streaming_table")
)
```

### 1.3 통계 수집 및 Data Skipping

Data Skipping은 쿼리 실행 시 불필요한 파일을 자동으로 건너뛰는 기능입니다. 정확한 통계 정보가 필수적입니다.

```sql
-- Delta 통계 재수집 (Runtime 14.3 LTS+)
ANALYZE TABLE catalog.schema.table_name COMPUTE DELTA STATISTICS;

-- 특정 컬럼에 대한 통계 수집
ALTER TABLE catalog.schema.table_name
SET TBLPROPERTIES('delta.dataSkippingStatsColumns' = 'event_date, user_id, region');

-- 통계 수집 대상 컬럼 수 확인 (기본: 처음 32개 컬럼)
-- Unity Catalog 관리형 테이블은 Predictive Optimization이 자동으로 관리
```

| 기능 | 동작 방식 | 성능 효과 |
|------|----------|----------|
| **Min/Max 통계** | 파일별 최소/최대값 기록 | WHERE절 조건으로 파일 단위 스킵 |
| **Null Count** | 파일별 NULL 수 기록 | IS NOT NULL 필터 최적화 |
| **Liquid Clustering** | 관련 데이터를 같은 파일에 배치 | Data Skipping 효과 극대화 |
| **ANALYZE TABLE** | 통계 강제 재수집 | 테이블 속성 변경 후 반드시 실행 |

---
