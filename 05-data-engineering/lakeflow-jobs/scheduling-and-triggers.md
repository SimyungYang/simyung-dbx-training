# 스케줄링과 트리거

## 스케줄링 방식

### Cron 기반 스케줄

> 💡 **Cron 표현식**은 작업의 실행 시점을 "분 시 일 월 요일" 형식으로 정의하는 표준 문법입니다.

| Cron 표현식 | 의미 |
|-------------|------|
| `0 2 * * *` | 매일 새벽 2시 |
| `0 */6 * * *` | 6시간마다 |
| `0 9 * * 1` | 매주 월요일 오전 9시 |
| `0 0 1 * *` | 매월 1일 자정 |

### 파일 도착 트리거

클라우드 스토리지에 새 파일이 도착하면 자동으로 Job을 실행합니다.

```yaml
trigger:
  file_arrival:
    url: "s3://my-bucket/incoming/"
    min_time_between_triggers_seconds: 60
```

### Continuous 모드

중단 없이 **계속 실행**되는 모드입니다. 스트리밍 워크로드에 적합합니다.

```yaml
trigger:
  periodic:
    interval: 1
    unit: "MINUTES"
continuous: true
```

---

## 알림 설정

| 알림 이벤트 | 설명 |
|------------|------|
| **On Start** | Job 시작 시 알림 |
| **On Success** | 성공적으로 완료 시 알림 |
| **On Failure** | 실패 시 알림 |
| **On Duration Warning** | 설정 시간 초과 시 알림 |

알림 대상: 이메일, Slack Webhook, PagerDuty, Microsoft Teams 등

---

## 참고 링크

- [Databricks: Schedule jobs](https://docs.databricks.com/aws/en/jobs/schedule.html)
- [Databricks: File arrival triggers](https://docs.databricks.com/aws/en/jobs/file-arrival-triggers.html)
