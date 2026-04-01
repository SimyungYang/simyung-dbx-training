# 스케줄링과 트리거

## 왜 스케줄링이 중요한가?

데이터 파이프라인은 "적시에, 안정적으로" 실행되어야 합니다. 매일 새벽 2시에 전일 매출을 집계해야 하고, 새 파일이 도착하면 즉시 수집해야 하며, 소스 테이블이 갱신되면 후속 파이프라인이 자동으로 시작되어야 합니다.

> 💡 **스케줄링(Scheduling)**이란 작업의 실행 시점과 빈도를 미리 정의하는 것이고, ** 트리거(Trigger)**란 특정 이벤트 발생 시 작업을 시작시키는 메커니즘입니다. Databricks는 시간 기반, 이벤트 기반, 연속 실행 등 다양한 방식을 지원합니다.

### 스케줄링 방식 비교

| 방식 | 설명 | 적합한 시나리오 |
|------|------|-----------------|
| **Cron 스케줄**| 고정된 시간에 주기적으로 실행합니다 | 일별/주별 배치 ETL, 리포트 생성 |
| ** 파일 도착 트리거**| 새 파일이 도착하면 실행합니다 | 외부 시스템 연동, 벤더 데이터 수집 |
| ** 테이블 트리거**| Delta 테이블이 갱신되면 실행합니다 | 파이프라인 체이닝, 이벤트 기반 처리 |
| ** 연속 실행**| 완료 즉시 다시 시작합니다 | 스트리밍 워크로드, 실시간 처리 |
| ** 수동 / API 트리거**| API 호출로 직접 실행합니다 | 임시 실행, 외부 오케스트레이터 연동 |

---

## Cron 스케줄링

### Cron 표현식 문법

> 💡 **Cron 표현식**은 작업의 실행 시점을 정의하는 표준 문법입니다. Databricks는 **Quartz Cron** 형식을 사용하며, 일반적인 Unix Cron과 약간 다릅니다.

```
Cron 필드 순서:  초  분  시  일  월  요일

| 필드 | 범위 |
|------|------|
| 초 | 0-59 |
| 분 | 0-59 |
| 시 | 0-23 |
| 일 | 1-31 |
| 월 | 1-12 |
| 요일 | 1-7 (1=일요일) |
* * * * * *
```

### 자주 사용하는 Cron 표현식

| Cron 표현식 | 의미 | 사용 사례 |
|-------------|------|-----------|
| `0 0 2 * * ?` | 매일 새벽 2시 | 일별 배치 ETL |
| `0 0 */6 * * ?` | 6시간마다 | 정기 데이터 동기화 |
| `0 0 9 ? * MON` | 매주 월요일 오전 9시 | 주간 리포트 생성 |
| `0 0 0 1 * ?` | 매월 1일 자정 | 월별 집계 |
| `0 */30 * * * ?` | 30분마다 | 빈번한 데이터 갱신 |
| `0 0 8-18 ? * MON-FRI` | 평일 매시간 (8시~18시) | 업무 시간 내 모니터링 |
| `0 0 2 ? * MON-FRI` | 평일 새벽 2시 | 영업일 기준 배치 처리 |
| `0 0 0 L * ?` | 매월 마지막 날 자정 | 월말 정산 |

### 특수 문자 설명

| 문자 | 의미 | 예시 |
|------|------|------|
| `*` | 모든 값 | `* * * * * ?` = 매초 |
| `?` | 지정하지 않음 (일/요일에 사용) | 일과 요일은 동시에 `*` 사용 불가 |
| `/` | 증분값 | `0/15` = 0분부터 15분 간격 |
| `-` | 범위 | `MON-FRI` = 월~금 |
| `,` | 목록 | `MON,WED,FRI` = 월, 수, 금 |
| `L` | 마지막 | `L` = 해당 월의 마지막 날 |
| `W` | 가장 가까운 평일 | `15W` = 15일에 가장 가까운 평일 |

### UI 및 YAML에서 설정

**UI 방식**: Workflows → Job → Schedule 탭에서 드롭다운으로 간편하게 설정하거나, 고급 옵션에서 Cron 표현식을 직접 입력할 수 있습니다.

```yaml
# Asset Bundles 설정
schedule:
  quartz_cron_expression: "0 0 2 * * ?"
  timezone_id: "Asia/Seoul"
  pause_status: "UNPAUSED"      # PAUSED로 설정하면 일시 중지
```

---

## 파일 도착 트리거 (File Arrival Trigger)

클라우드 스토리지(S3, ADLS, GCS)에 새 파일이 도착하면 자동으로 Job을 실행합니다. 외부 벤더가 FTP/S3로 데이터를 전송하는 시나리오에 적합합니다.

### 동작 원리

| 단계 | 발신 | 수신 | 내용 |
|------|------|------|------|
| 1 | 외부 벤더 | S3 Bucket | 파일 업로드 (orders_20250331.csv) |
| 2 | Databricks | S3 Bucket | 주기적 폴링 (새 파일 감지) |
| 3 | S3 Bucket | Databricks | 새 파일 발견 알림 |
| 4 | Databricks | Job | Job을 트리거합니다 |
| 5 | Job | S3 Bucket | 파일을 읽고 처리합니다 |
| 6 | Job | Databricks | 처리 완료 |

### 설정 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `url` | 감시할 클라우드 스토리지 경로입니다 | (필수) |
| `min_time_between_triggers_seconds` | 연속 트리거 사이 최소 대기 시간(초)입니다 | 60 |
| `wait_after_last_change_seconds` | 마지막 파일 변경 후 대기 시간(초)입니다. 여러 파일이 연속 도착하는 경우 유용합니다 | 0 |

```yaml
# Asset Bundles 설정
trigger:
  file_arrival:
    url: "s3://my-bucket/incoming/orders/"
    min_time_between_triggers_seconds: 300    # 최소 5분 간격
    wait_after_last_change_seconds: 60        # 마지막 변경 후 1분 대기
```

```yaml
# Azure Blob Storage 예제
trigger:
  file_arrival:
    url: "abfss://container@storage.dfs.core.windows.net/incoming/"
    min_time_between_triggers_seconds: 120
```

> ⚠️ **주의**: 파일 도착 트리거는 디렉토리 리스팅 방식으로 동작하므로, 파일이 매우 많은 디렉토리에서는 감지 지연이 발생할 수 있습니다. 날짜별 파티션 디렉토리를 사용하여 감시 대상을 좁히는 것을 권장합니다.

---

## 테이블 기반 트리거 (Table Trigger)

> 🆕 **[2024 릴리즈]**Delta 테이블의 데이터가 변경되면 자동으로 Job을 트리거하는 기능입니다. 파이프라인 체이닝에 매우 유용합니다.

소스 테이블이 갱신되면 후속 파이프라인이 자동으로 실행되어, 수동으로 스케줄을 맞출 필요가 없어집니다.

### 동작 원리

| 단계 | 발신 | 수신 | 내용 |
|------|------|------|------|
| 1 | Job A (수집) | Delta Table | 데이터 INSERT/UPDATE |
| 2 | Databricks | Delta Table | 테이블 변경 감지 |
| 3 | Databricks | Job B (변환) | Job B를 트리거합니다 |
| 4 | Job B | Delta Table | 변경된 데이터를 읽습니다 |
| 5 | Job B | Databricks | 처리 완료 |

### 설정 방법

```yaml
trigger:
  table:
    condition: "ANY_UPDATED"
    table_names:
      - "catalog.schema.bronze_orders"
      - "catalog.schema.bronze_customers"
    min_time_between_triggers_seconds: 300
    wait_after_last_change_seconds: 120
```

| 옵션 | 설명 |
|------|------|
| `condition` | 트리거 조건입니다. `ANY_UPDATED`(하나라도 갱신), `ALL_UPDATED`(모두 갱신) |
| `table_names` | 감시할 Delta 테이블 목록입니다 |
| `min_time_between_triggers_seconds` | 연속 트리거 사이 최소 대기 시간입니다 |
| `wait_after_last_change_seconds` | 마지막 변경 후 대기 시간입니다 |

> 💡 `ALL_UPDATED` 조건을 사용하면, 지정된 모든 테이블이 갱신되었을 때만 트리거됩니다. 여러 소스 테이블을 조인하는 변환 Job에 적합합니다.

---

## 연속 실행 (Continuous) 모드

Job이 완료되면 즉시 다시 시작하는 모드입니다. 스트리밍 워크로드나 매우 빈번한 배치 처리에 적합합니다.

```yaml
trigger:
  periodic:
    interval: 1
    unit: "MINUTES"
continuous:
  pause_status: "UNPAUSED"
```

| 특성 | 설명 |
|------|------|
| ** 동작 방식**| Job 완료 즉시 새 실행을 시작합니다 |
| ** 재시작 간격**| `periodic.interval`로 최소 간격을 설정할 수 있습니다 |
| ** 장애 복구**| 실패 시 자동 재시작됩니다 |
| ** 비용**| 24시간 클러스터가 실행되므로 비용이 높을 수 있습니다 |

> ⚠️ ** 연속 모드 vs Structured Streaming**: 단순히 "자주 실행"이 목적이라면 연속 모드가 아닌 짧은 Cron 간격(5~15분)이 더 비용 효율적일 수 있습니다. 진정한 스트리밍이 필요한 경우에만 연속 모드를 사용하세요.

---

## API 기반 수동 트리거

REST API를 통해 외부 시스템에서 Job을 실행할 수 있습니다. CI/CD 파이프라인, Airflow 같은 외부 오케스트레이터와의 연동에 사용됩니다.

### REST API 호출

```bash
# Job 즉시 실행 (Run Now)
curl -X POST "https://<workspace-url>/api/2.1/jobs/run-now" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "job_id": 12345,
    "job_parameters": {
      "target_date": "2025-03-31",
      "mode": "incremental"
    }
  }'
```

```bash
# 일회성 실행 (Submit Run) - Job 정의 없이 바로 실행
curl -X POST "https://<workspace-url>/api/2.1/jobs/runs/submit" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "run_name": "ad-hoc-backfill",
    "tasks": [
      {
        "task_key": "backfill",
        "notebook_task": {
          "notebook_path": "/Workspace/etl/backfill",
          "base_parameters": {
            "start_date": "2025-01-01",
            "end_date": "2025-03-31"
          }
        },
        "new_cluster": {
          "spark_version": "15.4.x-scala2.12",
          "num_workers": 4,
          "node_type_id": "r5.xlarge"
        }
      }
    ]
  }'
```

### Databricks SDK (Python)

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Job 즉시 실행
run = w.jobs.run_now(
    job_id=12345,
    job_parameters={
        "target_date": "2025-03-31",
        "mode": "incremental"
    }
)

# 실행 결과 대기
result = w.jobs.get_run(run_id=run.run_id)
print(f"상태: {result.state.result_state}")
```

---

## 타임존 설정

스케줄의 타임존을 올바르게 설정하지 않으면 예상과 다른 시간에 Job이 실행될 수 있습니다.

| 설정 | 설명 |
|------|------|
| `timezone_id` | IANA 타임존 문자열입니다 (예: `Asia/Seoul`, `UTC`, `US/Eastern`) |
| **서머타임(DST)** | 서머타임이 적용되는 타임존에서는 시간이 자동 조정됩니다 |

```yaml
schedule:
  quartz_cron_expression: "0 0 2 * * ?"
  timezone_id: "Asia/Seoul"        # KST (UTC+9)
```

> 💡 **권장사항**: 한국에서 운영하는 경우 `Asia/Seoul`을 사용하세요. 글로벌 팀이라면 `UTC`로 통일하고, 각 팀에서 로컬 시간으로 변환하는 것이 관리하기 쉽습니다.

---

## 동시 실행 제어

### max_concurrent_runs

같은 Job이 동시에 여러 번 실행되는 것을 제어합니다.

| 값 | 동작 |
|----|------|
| `1` (기본값) | 이전 실행이 완료될 때까지 새 실행을 대기열에 넣습니다 |
| `2 이상` | 지정 수까지 동시 실행을 허용합니다 |
| `0` | 동시 실행 수 제한이 없습니다 |

```yaml
max_concurrent_runs: 1    # 동시에 하나만 실행

# 큐 설정 (동시 실행 초과 시 동작)
queue:
  enabled: true           # 대기열에 넣음 (false면 스킵)
```

> ⚠️ **데이터 정합성 주의**: `max_concurrent_runs`가 2 이상이면 같은 데이터를 동시에 처리하여 충돌이 발생할 수 있습니다. 멱등성이 보장되지 않는 Job에는 반드시 `1`로 설정하세요.

### 대기열(Queue) 동작

| 조건 | 상태 | 동작 |
|------|------|------|
| 새 트리거 발생 + 현재 미실행 | 즉시 실행 | Job을 바로 실행합니다 |
| 새 트리거 발생 + 현재 실행 중 + `queue.enabled=true` | 대기열에 추가 | 순서대로 실행합니다 |
| 새 트리거 발생 + 현재 실행 중 + `queue.enabled=false` | 실행 스킵 | 알림을 발송합니다 |

---

## 스케줄 일시 중지 / 재개

유지보수, 장애 대응, 휴일 등의 상황에서 스케줄을 일시 중지할 수 있습니다.

### UI 방식
Workflows → Job → Schedule 탭 → **Pause** 버튼 클릭

### API 방식

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 스케줄 일시 중지
w.jobs.update(
    job_id=12345,
    new_settings={
        "schedule": {
            "quartz_cron_expression": "0 0 2 * * ?",
            "timezone_id": "Asia/Seoul",
            "pause_status": "PAUSED"
        }
    }
)

# 스케줄 재개
w.jobs.update(
    job_id=12345,
    new_settings={
        "schedule": {
            "quartz_cron_expression": "0 0 2 * * ?",
            "timezone_id": "Asia/Seoul",
            "pause_status": "UNPAUSED"
        }
    }
)
```

---

## SLA 관리 전략

서비스 수준 협약(SLA)을 준수하기 위한 스케줄링 전략을 수립해야 합니다.

### SLA 관리 체크리스트

| 전략 | 구현 방법 | 설명 |
|------|-----------|------|
| **충분한 버퍼 확보**| SLA 마감 2~3시간 전에 실행 시작 | 재시도, 지연 등에 대비합니다 |
| **Duration Warning 설정**| `health` 규칙으로 실행 시간 모니터링 | 예상보다 오래 걸리면 사전 경고합니다 |
| ** 다운스트림 트리거**| 테이블 트리거로 파이프라인 체이닝 | 불필요한 대기 시간을 제거합니다 |
| ** 장애 알림 즉시 발송**| Slack/PagerDuty Webhook 연동 | 장애 발생 시 빠르게 대응합니다 |
| ** 백필(Backfill) 계획** | 파라미터화된 Job으로 날짜 범위 지정 | 누락된 데이터를 신속하게 복구합니다 |

### SLA 관리 예제

```yaml
# SLA: 매일 오전 8시까지 Gold 테이블 갱신 완료
schedule:
  quartz_cron_expression: "0 0 2 * * ?"  # 새벽 2시 시작 (6시간 버퍼)
  timezone_id: "Asia/Seoul"

# 3시간 초과 시 경고
health:
  rules:
    - metric: "RUN_DURATION_SECONDS"
      op: "GREATER_THAN"
      value: 10800

# 실패 시 즉시 재시도 (최대 2회)
tasks:
  - task_key: "main_etl"
    max_retries: 2
    min_retry_interval_millis: 120000
    timeout_seconds: 14400    # 4시간 타임아웃

# 알림 설정
email_notifications:
  on_failure:
    - "oncall@company.com"
  on_duration_warning_threshold_exceeded:
    - "data-team@company.com"

webhook_notifications:
  on_failure:
    - id: "${var.pagerduty_webhook_id}"
```

---

## 실습: 다양한 트리거 설정 예제

### 시나리오 1: 일별 배치 파이프라인

```yaml
# 매일 새벽 2시 KST 실행, 동시 실행 방지
resources:
  jobs:
    daily_etl:
      name: "daily-sales-etl"
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"
        timezone_id: "Asia/Seoul"
      max_concurrent_runs: 1
      queue:
        enabled: true
      tasks:
        - task_key: "etl_main"
          notebook_task:
            notebook_path: "/Workspace/etl/daily_sales"
```

### 시나리오 2: 파일 도착 기반 수집

```yaml
# S3에 파일이 도착하면 5분 간격으로 감지
resources:
  jobs:
    file_ingestion:
      name: "vendor-data-ingestion"
      trigger:
        file_arrival:
          url: "s3://data-lake/incoming/vendor-a/"
          min_time_between_triggers_seconds: 300
          wait_after_last_change_seconds: 120
      max_concurrent_runs: 1
      tasks:
        - task_key: "ingest"
          notebook_task:
            notebook_path: "/Workspace/ingest/vendor_a"
```

### 시나리오 3: 테이블 갱신 기반 체이닝

```yaml
# bronze 테이블이 갱신되면 silver 변환 자동 실행
resources:
  jobs:
    silver_transform:
      name: "silver-orders-transform"
      trigger:
        table:
          condition: "ALL_UPDATED"
          table_names:
            - "catalog.bronze.orders"
            - "catalog.bronze.customers"
          min_time_between_triggers_seconds: 600
          wait_after_last_change_seconds: 180
      tasks:
        - task_key: "transform"
          notebook_task:
            notebook_path: "/Workspace/etl/silver_orders"
```

---

## 정리

| 트리거 방식 | 최적 시나리오 | 핵심 설정 |
|-------------|---------------|-----------|
| **Cron 스케줄**| 정기 배치 처리 | `quartz_cron_expression`, `timezone_id` |
| ** 파일 도착**| 외부 데이터 수집 | `url`, `min_time_between_triggers_seconds` |
| ** 테이블 트리거**| 파이프라인 체이닝 | `table_names`, `condition` |
| ** 연속 실행**| 스트리밍 워크로드 | `continuous.pause_status` |
| **API 호출** | 외부 오케스트레이션 | `run-now`, `runs/submit` |

---

## 참고 링크

- [Databricks: Schedule jobs](https://docs.databricks.com/aws/en/jobs/schedule.html)
- [Databricks: File arrival triggers](https://docs.databricks.com/aws/en/jobs/file-arrival-triggers.html)
- [Databricks: Table triggers](https://docs.databricks.com/aws/en/jobs/triggers.html)
- [Databricks: Continuous jobs](https://docs.databricks.com/aws/en/jobs/continuous.html)
- [Databricks: Jobs API 2.1](https://docs.databricks.com/api/workspace/jobs)
