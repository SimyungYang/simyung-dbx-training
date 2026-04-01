# 모니터링과 알림

## 왜 모니터링이 중요한가?

데이터 파이프라인은 "실행되고 있다"는 것만으로는 충분하지 않습니다. **제시간에 완료되었는지, 데이터 품질은 괜찮은지, 비용은 예산 내인지** 를 지속적으로 확인해야 합니다. 모니터링이 없으면 장애를 사후에야 발견하게 되고, 이는 SLA 위반과 비즈니스 손실로 이어질 수 있습니다.

> 💡 ** 모니터링의 3대 축**: (1) ** 장애 감지** — 실패를 빠르게 발견하고 대응합니다. (2) ** 성능 추적** — 실행 시간 변화를 관찰하여 잠재적 문제를 예방합니다. (3) ** 비용 관리** — 리소스 사용량과 비용을 추적하여 최적화합니다.

---

## Job Run 상태 및 생명주기

Job Run은 다음과 같은 생명주기를 거칩니다.

| 상태 | 전이 조건 | 다음 상태 |
|------|----------|-----------|
| (시작) | 트리거 발생 | QUEUED |
| QUEUED | 대기열에서 선택 | PENDING |
| PENDING | 클러스터 준비 완료 | RUNNING |
| RUNNING | 실행 완료/실패 | TERMINATING |
| TERMINATING | 정상 종료 | TERMINATED |
| TERMINATING | 오류 발생 | FAILED |
| TERMINATING | 타임아웃 | TIMED_OUT |
| RUNNING | 사용자 취소 | CANCELLING → CANCELLED |
| FAILED | 재시도 (max_retries > 0) | PENDING |
| TIMED_OUT | retry_on_timeout = true | PENDING |

### 상태별 설명

| 상태 | 설명 | 모니터링 포인트 |
|------|------|-----------------|
| **QUEUED**| 대기열에서 실행 순서를 기다립니다 | `max_concurrent_runs` 초과 시 발생 |
| **PENDING**| 클러스터 시작을 기다립니다 | 시작 시간이 길면 인스턴스 부족 의심 |
| **RUNNING**| 태스크가 실행 중입니다 | 실행 시간이 평소보다 길면 경고 |
| **TERMINATED**| 정상적으로 완료되었습니다 | `result_state = SUCCESS` 확인 |
| **FAILED**| 오류로 인해 실패했습니다 | 에러 메시지 확인, 알림 발송 |
| **TIMED_OUT**| 설정된 시간 내에 완료되지 못했습니다 | 타임아웃 값 조정 검토 |
| **CANCELLED**| 사용자 또는 시스템에 의해 취소되었습니다 | 취소 원인 파악 |

### Result State (최종 결과)

| result_state | 설명 |
|-------------|------|
| `SUCCESS` | 모든 태스크가 성공적으로 완료되었습니다 |
| `FAILED` | 하나 이상의 태스크가 실패했습니다 |
| `TIMEDOUT` | 타임아웃으로 종료되었습니다 |
| `CANCELED` | 실행이 취소되었습니다 |
| `MAXIMUM_CONCURRENT_RUNS_REACHED` | 동시 실행 제한으로 스킵되었습니다 |
| `EXCLUDED` | 조건부 실행에서 제외되었습니다 |

---

## 실행 이력 확인

### Workflows UI

| 확인 항목 | 위치 | 용도 |
|-----------|------|------|
| ** 실행 이력 목록**| Workflows → Job → Runs 탭 | 최근 실행 상태 일괄 확인 |
| ** 태스크별 상태**| 개별 Run → DAG 뷰 | 어느 태스크에서 실패했는지 파악 |
| ** 실행 시간 추이**| Job → Runs 탭의 Duration 그래프 | 성능 변화 트렌드 분석 |
| ** 로그 확인**| 개별 Task → Output / Logs | 에러 메시지, 스택 트레이스 확인 |
| **Spark UI**| 개별 Task → Spark UI 링크 | 스테이지별 성능, 셔플 분석 |

### API로 실행 이력 조회

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 최근 실행 이력 조회
runs = w.jobs.list_runs(
    job_id=12345,
    limit=10,
    expand_tasks=True
)

for run in runs:
    print(f"Run {run.run_id}: {run.state.result_state} "
          f"({run.execution_duration / 1000:.0f}초)")

    # 실패한 태스크 확인
    if run.state.result_state == "FAILED":
        for task in run.tasks:
            if task.state.result_state == "FAILED":
                print(f"  실패 태스크: {task.task_key}")
                print(f"  에러: {task.state.state_message}")
```

---

## 알림 설정 상세

### 알림 이벤트

| 이벤트 | 설명 | 권장 대상 |
|--------|------|-----------|
| **On Start**| Job 실행 시작 시 알림을 보냅니다 | 장시간 실행 Job |
| **On Success**| 성공적으로 완료 시 알림을 보냅니다 | SLA 관리 필요 Job |
| **On Failure**| 실패 시 알림을 보냅니다 | ** 모든 프로덕션 Job (필수)**|
| **On Duration Warning**| 설정 시간 초과 시 경고를 보냅니다 | SLA 마감이 있는 Job |
| **On Retry**| 태스크 재시도 시 알림을 보냅니다 | 불안정한 Job 모니터링 |
| **On Streaming Backlog**| 스트리밍 백로그 초과 시 알림을 보냅니다 | 스트리밍 Job |

### Webhook 연동

Slack, Microsoft Teams, PagerDuty 등 외부 서비스와 Webhook으로 연동할 수 있습니다.

**Slack Webhook 설정 절차**:

1. Databricks Workspace → **Settings**→ **Notifications destinations**→ **Add destination**
2. **Destination type**: Slack 선택
3. **Slack Webhook URL** 입력
4. 테스트 메시지 발송 확인
5. Job 설정에서 해당 Webhook ID 참조

```yaml
# 알림 목적지 설정 후 Job에 연동
webhook_notifications:
  on_start:
    - id: "${var.slack_info_webhook}"
  on_success:
    - id: "${var.slack_info_webhook}"
  on_failure:
    - id: "${var.slack_alert_webhook}"
    - id: "${var.pagerduty_webhook}"
  on_duration_warning_threshold_exceeded:
    - id: "${var.slack_alert_webhook}"

email_notifications:
  on_failure:
    - "data-oncall@company.com"
    - "team-lead@company.com"
  on_duration_warning_threshold_exceeded:
    - "data-oncall@company.com"
```

---

## 시스템 테이블 활용

Databricks는 Job 실행 이력, 비용, 클러스터 메트릭을 ** 시스템 테이블** 에 자동으로 기록합니다. SQL로 분석하여 커스텀 대시보드를 만들 수 있습니다.

> 💡 ** 시스템 테이블(System Tables)** 은 Databricks가 자동으로 수집하는 운영 메타데이터입니다. `system` 카탈로그 아래에 위치하며, Unity Catalog를 통해 접근합니다.

### 주요 시스템 테이블

| 테이블 | 설명 |
|--------|------|
| `system.lakeflow.job_run_timeline` | Job 실행 타임라인 (시작, 종료, 상태) |
| `system.lakeflow.job_task_run_timeline` | 태스크 레벨 실행 타임라인 |
| `system.billing.usage` | 리소스 사용량 및 비용 |
| `system.compute.clusters` | 클러스터 정보 |

### 실행 이력 분석 쿼리

```sql
-- 최근 7일간 Job 실행 현황 요약
SELECT
    job_id,
    job_name,
    COUNT(*) AS total_runs,
    SUM(CASE WHEN result_state = 'SUCCESS' THEN 1 ELSE 0 END) AS success_count,
    SUM(CASE WHEN result_state = 'FAILED' THEN 1 ELSE 0 END) AS failure_count,
    ROUND(
        SUM(CASE WHEN result_state = 'SUCCESS' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1
    ) AS success_rate_pct,
    AVG(TIMESTAMPDIFF(MINUTE, period_start_time, period_end_time)) AS avg_duration_min,
    MAX(TIMESTAMPDIFF(MINUTE, period_start_time, period_end_time)) AS max_duration_min
FROM system.lakeflow.job_run_timeline
WHERE period_start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY job_id, job_name
ORDER BY failure_count DESC;
```

### 실패 패턴 분석 쿼리

```sql
-- 자주 실패하는 태스크 Top 10
SELECT
    t.job_id,
    j.job_name,
    t.task_key,
    COUNT(*) AS failure_count,
    COLLECT_SET(t.result_state) AS failure_types,
    MAX(t.period_start_time) AS last_failure_time
FROM system.lakeflow.job_task_run_timeline t
JOIN system.lakeflow.job_run_timeline j
    ON t.run_id = j.run_id
WHERE t.result_state IN ('FAILED', 'TIMEDOUT')
    AND t.period_start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY t.job_id, j.job_name, t.task_key
ORDER BY failure_count DESC
LIMIT 10;
```

---

## 비용 추적 및 분석

### Job별 비용 확인

```sql
-- 최근 30일 Job별 DBU 사용량 (비용 상위 20)
SELECT
    usage_metadata.job_id,
    j.job_name,
    SUM(usage_quantity) AS total_dbus,
    COUNT(DISTINCT usage_date) AS active_days,
    ROUND(SUM(usage_quantity) / COUNT(DISTINCT usage_date), 1) AS avg_daily_dbus,
    SUM(usage_quantity * list_price) AS estimated_cost_usd
FROM system.billing.usage u
LEFT JOIN system.lakeflow.job_run_timeline j
    ON u.usage_metadata.job_id = j.job_id
WHERE u.usage_metadata.job_id IS NOT NULL
    AND u.usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY usage_metadata.job_id, j.job_name
ORDER BY total_dbus DESC
LIMIT 20;
```

### 비용 추이 분석

```sql
-- 주간 비용 추이 (Job 컴퓨트)
SELECT
    DATE_TRUNC('WEEK', usage_date) AS week,
    sku_name,
    SUM(usage_quantity) AS total_dbus,
    SUM(usage_quantity * list_price) AS estimated_cost_usd
FROM system.billing.usage
WHERE usage_metadata.job_id IS NOT NULL
    AND usage_date >= CURRENT_DATE() - INTERVAL 90 DAYS
GROUP BY DATE_TRUNC('WEEK', usage_date), sku_name
ORDER BY week DESC, total_dbus DESC;
```

### 비용 최적화 기회 발견

```sql
-- Spot 인스턴스 미사용 Job 탐지 (비용 절감 기회)
SELECT DISTINCT
    usage_metadata.job_id,
    sku_name,
    SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE usage_metadata.job_id IS NOT NULL
    AND sku_name LIKE '%ALL_PURPOSE%'   -- Job Cluster가 아닌 All-Purpose 사용
    AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY usage_metadata.job_id, sku_name
HAVING SUM(usage_quantity) > 100
ORDER BY total_dbus DESC;
```

---

## 로그 확인 방법

### Driver Log

각 태스크의 Driver Log에서 에러 메시지, 스택 트레이스를 확인할 수 있습니다.

| 로그 유형 | 확인 방법 | 내용 |
|-----------|-----------|------|
| **Standard Output**| Task → Output 탭 | `print()` 출력, 노트북 셀 결과 |
| **Standard Error**| Task → Logs → stderr | 에러 메시지, 경고 |
| **Log4j**| Task → Logs → log4j-active | Spark 내부 로그 |
| **Cluster Event Log** | Cluster → Event Log | 클러스터 시작/종료, 노드 추가/제거 |

### Spark UI 활용

| 증상 | Spark UI 탭 | 확인 항목 |
|------|-------------|----------|
| Job 단계에서 느림 | Jobs 탭 | 스테이지별 시간을 확인합니다 |
| 셔플 문제 | Stages 탭 | Shuffle Read/Write를 확인합니다 |
| 데이터 편향 | Tasks 탭 | 태스크별 시간 편차를 확인합니다 |
| 메모리 문제 | Executors 탭 | GC 시간, 메모리 사용량을 확인합니다 |

---

## 실습: 모니터링 대시보드 쿼리

아래 쿼리들을 Databricks SQL 대시보드에 배치하면 종합적인 모니터링 환경을 구축할 수 있습니다.

### 대시보드 패널 1: 실행 현황 요약

```sql
-- 오늘의 Job 실행 현황 (대시보드 카드용)
SELECT
    COUNT(*) AS total_runs,
    SUM(CASE WHEN result_state = 'SUCCESS' THEN 1 ELSE 0 END) AS succeeded,
    SUM(CASE WHEN result_state = 'FAILED' THEN 1 ELSE 0 END) AS failed,
    SUM(CASE WHEN result_state IS NULL THEN 1 ELSE 0 END) AS running
FROM system.lakeflow.job_run_timeline
WHERE period_start_time >= CURRENT_DATE();
```

### 대시보드 패널 2: 실행 시간 이상 감지

```sql
-- 평균 대비 2배 이상 오래 걸린 실행 (최근 24시간)
WITH job_stats AS (
    SELECT
        job_id,
        job_name,
        AVG(TIMESTAMPDIFF(MINUTE, period_start_time, period_end_time)) AS avg_duration,
        STDDEV(TIMESTAMPDIFF(MINUTE, period_start_time, period_end_time)) AS stddev_duration
    FROM system.lakeflow.job_run_timeline
    WHERE period_start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
        AND result_state = 'SUCCESS'
    GROUP BY job_id, job_name
)
SELECT
    r.job_id,
    r.job_name,
    r.run_id,
    TIMESTAMPDIFF(MINUTE, r.period_start_time, r.period_end_time) AS actual_duration_min,
    ROUND(s.avg_duration, 1) AS avg_duration_min,
    ROUND(
        TIMESTAMPDIFF(MINUTE, r.period_start_time, r.period_end_time) / s.avg_duration, 1
    ) AS duration_ratio
FROM system.lakeflow.job_run_timeline r
JOIN job_stats s ON r.job_id = s.job_id
WHERE r.period_start_time >= CURRENT_DATE() - INTERVAL 1 DAY
    AND TIMESTAMPDIFF(MINUTE, r.period_start_time, r.period_end_time) > s.avg_duration * 2
ORDER BY duration_ratio DESC;
```

### 대시보드 패널 3: 일별 성공률 추이

```sql
-- 최근 30일 일별 성공률 (라인 차트용)
SELECT
    DATE(period_start_time) AS run_date,
    COUNT(*) AS total_runs,
    ROUND(
        SUM(CASE WHEN result_state = 'SUCCESS' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1
    ) AS success_rate_pct
FROM system.lakeflow.job_run_timeline
WHERE period_start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
    AND result_state IS NOT NULL
GROUP BY DATE(period_start_time)
ORDER BY run_date;
```

---

## 트러블슈팅 체크리스트

### Job이 실패했을 때

| 순서 | 확인 사항 | 조치 |
|------|-----------|------|
| 1 | **에러 메시지 확인**| Task → Output/Logs에서 에러 확인 |
| 2 | ** 최근 코드 변경 확인**| 최근 배포된 변경이 원인인지 확인 |
| 3 | ** 데이터 변경 확인**| 소스 데이터의 스키마 변경, 볼륨 급증 확인 |
| 4 | ** 리소스 부족 확인**| OOM, Disk Space 부족 여부 확인 |
| 5 | ** 외부 의존성 확인**| API, 데이터베이스 연결 장애 확인 |
| 6 | ** 재시도 로그 확인**| 재시도 횟수 소진 여부 확인 |

### Job이 느려졌을 때

| 순서 | 확인 사항 | 조치 |
|------|-----------|------|
| 1 | ** 데이터 볼륨 증가**| 소스 데이터 크기 변화 확인 |
| 2 | **Spark UI: Shuffle**| 과도한 셔플이 발생하는지 확인 |
| 3 | **Spark UI: Skew**| 데이터 편향(Skew)이 있는지 확인 |
| 4 | ** 클러스터 이벤트 로그**| 노드 손실, 리사이즈 이벤트 확인 |
| 5 | **Spot 인스턴스 회수**| Spot 회수로 인한 재시작 확인 |
| 6 | ** 동시 실행 경합**| 같은 리소스를 사용하는 다른 Job 확인 |

### Job이 시작되지 않을 때

| 순서 | 확인 사항 | 조치 |
|------|-----------|------|
| 1 | ** 스케줄 상태**| Pause 상태가 아닌지 확인 |
| 2 | **max_concurrent_runs**| 이전 실행이 아직 진행 중인지 확인 |
| 3 | ** 클러스터 할당량**| 계정의 클러스터 할당량 초과 여부 확인 |
| 4 | ** 인스턴스 가용성**| 선택한 인스턴스 타입의 가용성 확인 |
| 5 | ** 권한 확인**| Job 소유자의 권한이 유효한지 확인 |

---

## 모범 사례 요약

| 영역 | 모범 사례 |
|------|-----------|
| ** 알림**| 모든 프로덕션 Job에 실패 알림을 설정합니다. Slack + 이메일 이중화를 권장합니다 |
| **Duration Warning**| 평소 실행 시간의 1.5~2배를 임계치로 설정합니다 |
| ** 재시도**| 일시적 오류에 대비하여 1~2회 재시도를 설정합니다 |
| ** 태그**| Job에 팀명, 프로젝트명, 환경 태그를 달아 비용과 실행을 추적합니다 |
| ** 대시보드**| 시스템 테이블 기반 모니터링 대시보드를 구축합니다 |
| ** 로그 보존**| 중요한 로그는 별도 테이블에 저장하여 장기 분석에 활용합니다 |
| ** 정기 점검** | 주간/월간 단위로 비용, 성공률, 실행 시간 추이를 리뷰합니다 |

---

## 참고 링크

- [Databricks: Monitor jobs](https://docs.databricks.com/aws/en/jobs/monitor.html)
- [Databricks: System tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/)
- [Databricks: Lakeflow system tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/lakeflow.html)
- [Databricks: Billing system tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/billing.html)
- [Databricks: Notification destinations](https://docs.databricks.com/aws/en/jobs/notifications.html)
