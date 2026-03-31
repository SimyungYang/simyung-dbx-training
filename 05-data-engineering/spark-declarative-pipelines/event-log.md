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
