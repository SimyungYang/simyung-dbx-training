# SDP 이벤트 로그 활용

## 이벤트 로그란?

SDP(Spark Declarative Pipelines) 파이프라인은 실행 과정에서 발생하는 모든 활동을 **이벤트 로그(Event Log)** 에 기록합니다. 데이터 흐름 진행 상황, 데이터 품질 검증 결과, 오류 정보, 성능 지표 등이 모두 포함됩니다.

> 💡 **이벤트 로그는 파이프라인의 "블랙박스"입니다.** 파이프라인에서 무슨 일이 일어났는지, 데이터 품질은 어떤지, 어디서 병목이 발생했는지를 모두 확인할 수 있습니다.

---

## 이벤트 로그 조회 방법

### event_log() 테이블 함수

Unity Catalog에 등록된 파이프라인의 이벤트 로그는 `event_log()` 함수로 조회할 수 있습니다.

```sql
-- 파이프라인 ID로 이벤트 로그 조회
SELECT * FROM event_log("pipeline-id-here")
ORDER BY timestamp DESC
LIMIT 100;

-- 파이프라인 이름으로 조회
SELECT * FROM event_log(TABLE(catalog.schema.my_table))
ORDER BY timestamp DESC;
```

### 이벤트 로그 스키마

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | STRING | 이벤트 고유 ID입니다 |
| `sequence` | STRUCT | 이벤트 순서 정보입니다 |
| `origin` | STRUCT | 이벤트 발생 원점 (파이프라인, 클러스터 등) 정보입니다 |
| `timestamp` | TIMESTAMP | 이벤트 발생 시각입니다 |
| `message` | STRING | 사람이 읽을 수 있는 이벤트 메시지입니다 |
| `level` | STRING | 이벤트 레벨입니다 (INFO, WARN, ERROR, METRICS) |
| `maturity_level` | STRING | API 안정성 수준입니다 (STABLE, EVOLVING) |
| `error` | STRUCT | 오류 발생 시 상세 정보입니다 |
| `details` | STRING | 이벤트 상세 정보 (JSON 형식)입니다 |
| `event_type` | STRING | 이벤트 유형입니다 |

---

## 주요 이벤트 유형

### 이벤트 유형 분류

| 이벤트 유형 | 설명 | 활용 |
|------------|------|------|
| **`user_action`** | 사용자가 파이프라인을 시작/중지한 이벤트입니다 | 실행 이력 추적 |
| **`flow_definition`** | 데이터 흐름(flow) 정의 정보입니다 | 파이프라인 구조 파악 |
| **`flow_progress`** | 데이터 흐름의 진행 상황입니다 | 처리량 모니터링, 병목 분석 |
| **`planning_information`** | 실행 계획 정보입니다 | 리소스 사용 분석 |
| **`dataset_definition`** | 데이터셋(테이블/뷰) 정의 정보입니다 | 스키마 변경 추적 |
| **`cluster_resources`** | 클러스터 리소스 사용 정보입니다 | 비용 최적화 |

### flow_progress 이벤트 분석

`flow_progress`는 각 데이터 흐름의 처리 결과를 담고 있어, 파이프라인 성능 분석에 가장 유용합니다.

```sql
-- 각 테이블별 처리 행 수와 소요 시간 조회
SELECT
    timestamp,
    details:flow_progress:data_quality:expectations AS expectations,
    details:flow_progress:metrics:num_output_rows AS output_rows,
    details:flow_progress:status AS status,
    details:flow_progress:flow_name AS flow_name
FROM event_log("pipeline-id-here")
WHERE event_type = 'flow_progress'
    AND details:flow_progress:status = 'COMPLETED'
ORDER BY timestamp DESC;
```

---

## Expectations (데이터 품질) 결과 조회

SDP의 Expectations(데이터 품질 규칙) 검증 결과는 이벤트 로그에 자동으로 기록됩니다. 이를 활용하면 데이터 품질 추이를 모니터링할 수 있습니다.

### Expectations 결과 조회 쿼리

```sql
-- Expectations 결과만 추출
WITH expectations_data AS (
    SELECT
        timestamp,
        details:flow_progress:flow_name AS flow_name,
        EXPLODE(
            FROM_JSON(
                details:flow_progress:data_quality:expectations,
                'ARRAY<STRUCT<name: STRING, dataset: STRING, passed_records: BIGINT, failed_records: BIGINT>>'
            )
        ) AS expectation
    FROM event_log("pipeline-id-here")
    WHERE event_type = 'flow_progress'
        AND details:flow_progress:data_quality IS NOT NULL
)
SELECT
    DATE(timestamp) AS date,
    flow_name,
    expectation.name AS rule_name,
    expectation.passed_records,
    expectation.failed_records,
    ROUND(
        expectation.passed_records * 100.0 /
        (expectation.passed_records + expectation.failed_records), 2
    ) AS pass_rate_pct
FROM expectations_data
ORDER BY timestamp DESC;
```

### 결과 예시

| date | flow_name | rule_name | passed_records | failed_records | pass_rate_pct |
|------|-----------|-----------|---------------:|---------------:|--------------:|
| 2025-03-31 | silver_orders | valid_amount | 98,542 | 158 | 99.84 |
| 2025-03-31 | silver_orders | not_null_id | 98,700 | 0 | 100.00 |
| 2025-03-30 | silver_orders | valid_amount | 95,211 | 289 | 99.70 |

---

## 파이프라인 디버깅에 활용

### 오류 이벤트 조회

```sql
-- 오류 이벤트만 필터링
SELECT
    timestamp,
    level,
    message,
    error.exceptions AS error_details
FROM event_log("pipeline-id-here")
WHERE level = 'ERROR'
ORDER BY timestamp DESC
LIMIT 20;
```

### 실행 시간 분석

```sql
-- 파이프라인 실행별 소요 시간 분석
WITH update_events AS (
    SELECT
        origin.update_id,
        MIN(CASE WHEN event_type = 'user_action' AND details:user_action:action = 'START' THEN timestamp END) AS start_time,
        MAX(CASE WHEN event_type = 'user_action' AND details:user_action:action IN ('STOP', 'COMPLETE') THEN timestamp END) AS end_time
    FROM event_log("pipeline-id-here")
    GROUP BY origin.update_id
)
SELECT
    update_id,
    start_time,
    end_time,
    TIMESTAMPDIFF(MINUTE, start_time, end_time) AS duration_minutes
FROM update_events
WHERE start_time IS NOT NULL AND end_time IS NOT NULL
ORDER BY start_time DESC
LIMIT 10;
```

### 테이블별 처리량 추이

```sql
-- 일별 테이블별 처리 행 수 추이
SELECT
    DATE(timestamp) AS date,
    details:flow_progress:flow_name AS table_name,
    SUM(CAST(details:flow_progress:metrics:num_output_rows AS BIGINT)) AS total_rows
FROM event_log("pipeline-id-here")
WHERE event_type = 'flow_progress'
    AND details:flow_progress:status = 'COMPLETED'
GROUP BY 1, 2
ORDER BY date DESC, table_name;
```

---

## 이벤트 로그 기반 모니터링 대시보드

이벤트 로그 데이터를 활용하여 모니터링 대시보드를 구축할 수 있습니다.

### 권장 모니터링 지표

| 지표 | 쿼리 대상 | 임계값 예시 |
|------|----------|-----------|
| **파이프라인 실행 시간** | `user_action` 이벤트의 START/COMPLETE 간격 | 평소 대비 2배 초과 시 알림 |
| **Expectations 통과율** | `flow_progress`의 data_quality | 99% 미만 시 알림 |
| **처리된 행 수** | `flow_progress`의 num_output_rows | 0행 처리 시 알림 |
| **오류 발생 횟수** | `level = 'ERROR'` 이벤트 수 | 1건 이상 시 알림 |

### 알림 설정용 뷰 생성

```sql
-- 데이터 품질 모니터링 뷰
CREATE OR REPLACE VIEW catalog.schema.v_pipeline_quality AS
WITH latest_expectations AS (
    SELECT
        timestamp,
        details:flow_progress:flow_name AS flow_name,
        EXPLODE(
            FROM_JSON(
                details:flow_progress:data_quality:expectations,
                'ARRAY<STRUCT<name: STRING, passed_records: BIGINT, failed_records: BIGINT>>'
            )
        ) AS exp
    FROM event_log("pipeline-id-here")
    WHERE event_type = 'flow_progress'
        AND details:flow_progress:data_quality IS NOT NULL
        AND DATE(timestamp) = CURRENT_DATE()
)
SELECT
    flow_name,
    exp.name AS rule_name,
    SUM(exp.passed_records) AS total_passed,
    SUM(exp.failed_records) AS total_failed,
    ROUND(
        SUM(exp.passed_records) * 100.0 /
        NULLIF(SUM(exp.passed_records) + SUM(exp.failed_records), 0), 2
    ) AS pass_rate_pct
FROM latest_expectations
GROUP BY flow_name, exp.name;
```

---

## 현업 사례: 이벤트 로그로 원인을 찾아낸 경험

> 🔥 **실전 경험담**
>
> 한 이커머스 고객사에서 매일 새벽 3시에 돌아가는 SDP 파이프라인이 **2주간 간헐적으로 실패**하고 있었습니다. Spark UI를 봐도 "Task failed" 정도의 메시지만 보였고, 노트북 코드 자체에는 문제가 없어 보였습니다. 개발팀은 "서버 문제 아닌가요?"라며 인프라 팀에 문의했지만, 인프라 팀도 원인을 찾지 못했습니다.
>
> 이벤트 로그를 분석하자 패턴이 보였습니다:
>
> ```sql
> -- 실패 시간대와 flow별 분석
> SELECT
>     DATE(timestamp) AS date,
>     HOUR(timestamp) AS hour,
>     details:flow_progress:flow_name AS flow_name,
>     details:flow_progress:status AS status,
>     message
> FROM event_log("pipeline-id")
> WHERE level = 'ERROR'
>     AND timestamp > '2025-03-01'
> ORDER BY timestamp DESC;
> ```
>
> 결과를 보니, 실패하는 flow는 항상 `silver_product_inventory`였고, 에러 메시지에 "Schema mismatch" 관련 내용이 있었습니다. 소스 시스템(ERP)에서 **매일 새벽 2시 50분에 배치 작업을 돌리면서 임시로 컬럼을 추가했다가 삭제하는 패턴**이 있었고, SDP가 그 타이밍에 데이터를 읽으면 스키마 불일치가 발생하는 것이었습니다.
>
> **해결**: 파이프라인 스케줄을 새벽 3시 30분으로 변경하여 ERP 배치 작업이 완료된 후 실행되도록 조정했습니다. 이벤트 로그가 없었으면 **수개월간 원인을 못 찾았을 수도 있는 문제**였습니다.

---

## 이벤트 로그 분석 실전 패턴

### 패턴 1: 파이프라인 성능 추이 분석

일별 파이프라인 실행 시간 추이를 추적하면, 성능 저하를 조기에 발견할 수 있습니다.

```sql
-- 파이프라인 일별 실행 시간 추이 (점점 느려지고 있는지 확인)
WITH daily_runs AS (
    SELECT
        DATE(timestamp) AS run_date,
        origin.update_id,
        MIN(timestamp) AS start_time,
        MAX(timestamp) AS end_time
    FROM event_log("pipeline-id-here")
    WHERE event_type = 'user_action'
    GROUP BY 1, 2
)
SELECT
    run_date,
    COUNT(DISTINCT update_id) AS num_runs,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, start_time, end_time)), 1) AS avg_duration_min,
    MAX(TIMESTAMPDIFF(MINUTE, start_time, end_time)) AS max_duration_min
FROM daily_runs
WHERE start_time IS NOT NULL AND end_time IS NOT NULL
GROUP BY run_date
ORDER BY run_date DESC
LIMIT 30;
```

> 💡 **현업 팁**: 파이프라인 실행 시간이 **주 단위로 5% 이상 증가하는 추세**가 보이면, 데이터 볼륨 증가에 따른 클러스터 스케일업이 필요한 신호입니다. 이 추세를 무시하면 어느 날 갑자기 SLA를 위반하게 됩니다.

### 패턴 2: 데이터 품질 추이 대시보드

Expectations 통과율을 일별로 추적하면, 소스 데이터의 품질 변화를 감지할 수 있습니다.

```sql
-- 주간 데이터 품질 리포트
WITH weekly_quality AS (
    SELECT
        DATE_TRUNC('WEEK', timestamp) AS week_start,
        details:flow_progress:flow_name AS flow_name,
        EXPLODE(
            FROM_JSON(
                details:flow_progress:data_quality:expectations,
                'ARRAY<STRUCT<name: STRING, passed_records: BIGINT, failed_records: BIGINT>>'
            )
        ) AS exp
    FROM event_log("pipeline-id-here")
    WHERE event_type = 'flow_progress'
        AND details:flow_progress:data_quality IS NOT NULL
)
SELECT
    week_start,
    flow_name,
    exp.name AS rule_name,
    SUM(exp.passed_records) AS total_passed,
    SUM(exp.failed_records) AS total_failed,
    ROUND(
        SUM(exp.passed_records) * 100.0 /
        NULLIF(SUM(exp.passed_records) + SUM(exp.failed_records), 0), 2
    ) AS pass_rate_pct,
    -- 전주 대비 변화
    LAG(
        ROUND(SUM(exp.passed_records) * 100.0 /
        NULLIF(SUM(exp.passed_records) + SUM(exp.failed_records), 0), 2)
    ) OVER (PARTITION BY flow_name, exp.name ORDER BY week_start) AS prev_week_rate
FROM weekly_quality
GROUP BY week_start, flow_name, exp.name
ORDER BY week_start DESC, flow_name;
```

> 🔥 **이것을 안 하면**: 데이터 품질 문제는 즉시 드러나지 않습니다. 통과율이 99.9%에서 99.5%로 떨어져도 아무도 눈치채지 못하다가, 어느 날 갑자기 95%로 급락합니다. **주간 추이를 모니터링해야 점진적 악화를 조기에 발견**할 수 있습니다.

### 패턴 3: 특정 flow의 병목 분석

```sql
-- 각 flow별 평균 처리 시간과 처리량 (병목 구간 식별)
SELECT
    details:flow_progress:flow_name AS flow_name,
    COUNT(*) AS total_runs,
    ROUND(AVG(CAST(details:flow_progress:metrics:num_output_rows AS BIGINT)), 0) AS avg_rows,
    ROUND(AVG(
        TIMESTAMPDIFF(SECOND,
            LAG(timestamp) OVER (PARTITION BY details:flow_progress:flow_name ORDER BY timestamp),
            timestamp
        )
    ), 1) AS avg_interval_sec
FROM event_log("pipeline-id-here")
WHERE event_type = 'flow_progress'
    AND details:flow_progress:status = 'COMPLETED'
    AND timestamp > CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY details:flow_progress:flow_name
ORDER BY avg_rows DESC;
```

---

## 자동 모니터링 구축

현업에서는 이벤트 로그를 수동으로 조회하지 않습니다. **자동 알림 시스템**을 구축하여 문제가 발생하면 즉시 대응합니다.

### Step 1: 모니터링 뷰 생성

```sql
-- 파이프라인 건강 상태 종합 뷰
CREATE OR REPLACE VIEW catalog.schema.v_pipeline_health AS
SELECT
    'error_count' AS metric_name,
    COUNT(*) AS metric_value,
    CURRENT_TIMESTAMP() AS checked_at
FROM event_log("pipeline-id-here")
WHERE level = 'ERROR'
    AND DATE(timestamp) = CURRENT_DATE()

UNION ALL

SELECT
    'avg_duration_min',
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, start_time, end_time)), 1),
    CURRENT_TIMESTAMP()
FROM (
    SELECT
        origin.update_id,
        MIN(timestamp) AS start_time,
        MAX(timestamp) AS end_time
    FROM event_log("pipeline-id-here")
    WHERE DATE(timestamp) = CURRENT_DATE()
    GROUP BY origin.update_id
);
```

### Step 2: SQL 알림(Alert) 설정

1. **Databricks SQL** > **Alerts** > **Create Alert**
2. 위 뷰에서 `error_count > 0` 조건으로 알림 생성
3. 알림 대상: Slack 채널 또는 이메일
4. 체크 주기: 15분

> 💡 **현업 팁**: 파이프라인 모니터링은 **"에러 발생"과 "SLA 위반 예상" 두 가지를 동시에 감시**해야 합니다. 에러가 없어도 실행 시간이 평소의 2배를 넘으면 알림을 보내는 것이 좋습니다. 실행 시간 증가는 장애의 전조 증상인 경우가 많습니다.

### Step 3: 장기 보관 자동화 Job

```python
# Lakeflow Jobs로 매일 이벤트 로그를 장기 보관 테이블에 적재
# pipeline_event_archive 테이블은 별도로 생성해 두어야 합니다

spark.sql("""
    INSERT INTO catalog.schema.pipeline_event_archive
    SELECT
        *,
        CURRENT_TIMESTAMP() AS archived_at
    FROM event_log("pipeline-id-here")
    WHERE DATE(timestamp) = CURRENT_DATE() - INTERVAL 1 DAY
""")

# 장기 보관 테이블에서 월간 리포트 생성 가능
# 30일 기본 보존 기간 이후에도 이력을 추적할 수 있습니다
```

> 🔥 **이것을 안 하면**: 이벤트 로그는 기본 30일만 보존됩니다. 한 고객사에서 "3개월 전에도 같은 문제가 있었는데 그때 어떻게 해결했지?"라는 상황이 발생했을 때, 이벤트 로그가 이미 삭제되어 원인 분석을 처음부터 다시 해야 했습니다. **장기 보관은 선택이 아니라 필수**입니다.

---

## 이벤트 로그 보존 및 관리

| 항목 | 설명 |
|------|------|
| **보존 기간** | 이벤트 로그는 기본적으로 30일간 보존됩니다 |
| **장기 보관** | 장기 분석이 필요하면 이벤트 로그를 별도 Delta 테이블로 복사하세요 |
| **접근 권한** | 이벤트 로그 조회에는 파이프라인에 대한 VIEW 권한이 필요합니다 |

```sql
-- 이벤트 로그를 장기 보관 테이블로 복사
INSERT INTO catalog.schema.pipeline_event_archive
SELECT * FROM event_log("pipeline-id-here")
WHERE DATE(timestamp) = CURRENT_DATE() - INTERVAL 1 DAY;
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **이벤트 로그** | SDP 파이프라인의 모든 활동이 기록되는 감사 로그입니다 |
| **event_log()** | 이벤트 로그를 조회하는 테이블 함수입니다 |
| **flow_progress** | 데이터 흐름의 처리 결과와 품질 지표를 담고 있습니다 |
| **Expectations 결과** | 데이터 품질 규칙의 통과/실패 건수가 자동 기록됩니다 |
| **디버깅 활용** | 오류 상세, 실행 시간, 처리량 분석에 활용할 수 있습니다 |

---

## 참고 링크

- [Databricks: SDP event log](https://docs.databricks.com/aws/en/delta-live-tables/observability.html)
- [Databricks: Query the event log](https://docs.databricks.com/aws/en/delta-live-tables/observability.html#query-the-event-log)
- [Databricks: Monitor data quality](https://docs.databricks.com/aws/en/delta-live-tables/expectations.html)
- [Azure Databricks: Pipeline observability](https://learn.microsoft.com/en-us/azure/databricks/delta-live-tables/observability)
