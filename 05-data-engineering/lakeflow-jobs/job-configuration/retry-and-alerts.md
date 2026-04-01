# 재시도, 타임아웃, 알림

## 재시도 정책

태스크가 실패했을 때 자동으로 재시도하는 정책을 설정할 수 있습니다. 네트워크 오류, 일시적 리소스 부족 등 일시적 장애에 효과적입니다.

| 설정 | 설명 | 권장값 |
|------|------|--------|
| **Max Retries**| 실패 시 최대 재시도 횟수입니다 | 프로덕션: 1~3회 |
| **Min Retry Interval (초)**| 재시도 사이 최소 대기 시간입니다 | 60~300초 |
| **Max Retry Interval (초)**| 재시도 사이 최대 대기 시간입니다 | 600~1800초 |
| **Retry on Timeout**| 타임아웃으로 실패한 경우에도 재시도할지 여부입니다 | 상황에 따라 판단 |

```yaml
# Asset Bundles에서 재시도 설정
tasks:
  - task_key: "etl_transform"
    notebook_task:
      notebook_path: "/Workspace/etl/transform"
    max_retries: 2
    min_retry_interval_millis: 120000   # 2분
    retry_on_timeout: false
    timeout_seconds: 3600               # 1시간 타임아웃
```

> ⚠️ ** 재시도 시 멱등성(Idempotency) 확인**: 재시도되는 태스크는 여러 번 실행해도 같은 결과를 내야 합니다. `MERGE` 문이나 `CREATE OR REPLACE` 패턴을 사용하면 멱등성을 보장할 수 있습니다.

---

## 타임아웃 설정

각 태스크 및 전체 Job에 타임아웃을 설정하여, 무한히 실행되는 상황을 방지할 수 있습니다.

| 레벨 | 설정 | 설명 |
|------|------|------|
| ** 태스크 레벨** | `timeout_seconds` | 개별 태스크의 최대 실행 시간입니다 |
| **Job 레벨**| `health` 규칙 | 전체 Job의 건강 상태를 모니터링합니다 |

```yaml
tasks:
  - task_key: "heavy_etl"
    timeout_seconds: 7200    # 2시간 타임아웃

# Job 레벨 건강 규칙
health:
  rules:
    - metric: "RUN_DURATION_SECONDS"
      op: "GREATER_THAN"
      value: 10800           # 3시간 초과 시 경고
```

---

## 환경변수 및 시크릿 설정

민감한 정보(API 키, 데이터베이스 비밀번호 등)는 시크릿으로 관리해야 합니다.

```python
# Databricks Secrets를 사용한 자격 증명 관리
api_key = dbutils.secrets.get(scope="production", key="api-key")
db_password = dbutils.secrets.get(scope="production", key="db-password")

# 환경변수로 전달된 값 사용
import os
env = os.environ.get("ENVIRONMENT", "development")
```

```yaml
# Asset Bundles에서 환경변수 설정
tasks:
  - task_key: "data_sync"
    notebook_task:
      notebook_path: "/Workspace/etl/sync"
    new_cluster:
      spark_env_vars:
        ENVIRONMENT: "${bundle.target}"
        LOG_LEVEL: "INFO"
      spark_conf:
        "spark.databricks.delta.optimizeWrite.enabled": "true"
        "spark.databricks.delta.autoCompact.enabled": "true"
```

> ⚠️ ** 보안 주의**: 절대로 코드나 설정 파일에 비밀번호, API 키를 직접 입력하지 마세요. 반드시 `dbutils.secrets`를 사용하거나 Unity Catalog의 시크릿 관리 기능을 활용하세요.

---

## 알림 설정

Job의 상태 변화에 따라 다양한 채널로 알림을 보낼 수 있습니다.

| 알림 이벤트 | 설명 | 권장 설정 |
|-------------|------|-----------|
| **On Start**| Job 실행 시작 시 알림을 보냅니다 | 장시간 Job에 설정 |
| **On Success**| 성공적으로 완료 시 알림을 보냅니다 | 주요 파이프라인에 설정 |
| **On Failure**| 실패 시 알림을 보냅니다 | ** 모든 프로덕션 Job에 필수**|
| **On Duration Warning**| 설정 시간 초과 시 경고를 보냅니다 | SLA 관리용 |
| **On Streaming Backlog** | 스트리밍 백로그 임계치 초과 시 알림을 보냅니다 | 스트리밍 Job에 설정 |

```yaml
# 알림 설정 예제
email_notifications:
  on_start:
    - "oncall@company.com"
  on_success:
    - "data-team@company.com"
  on_failure:
    - "oncall@company.com"
    - "data-team-lead@company.com"
  on_duration_warning_threshold_exceeded:
    - "oncall@company.com"

webhook_notifications:
  on_failure:
    - id: "${var.slack_webhook_id}"
    - id: "${var.pagerduty_webhook_id}"

notification_settings:
  no_alert_for_skipped_runs: true      # 스킵된 실행에는 알림 안 보냄
  no_alert_for_canceled_runs: false    # 취소된 실행에는 알림 보냄
```

---
