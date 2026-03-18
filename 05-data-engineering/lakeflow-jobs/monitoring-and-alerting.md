# 모니터링과 알림

## Job 실행 모니터링

### Workflows UI

Job 실행 이력, 성공/실패 현황, 각 Task의 실행 시간을 Workflows UI에서 시각적으로 확인할 수 있습니다.

| 확인 항목 | 위치 |
|-----------|------|
| 실행 이력 | Workflows → Job → Runs 탭 |
| Task별 상태 | 개별 Run → DAG 뷰 |
| 로그 확인 | 개별 Task → Output / Logs |
| 실행 시간 추이 | Job → Runs 탭의 Duration 그래프 |

### 시스템 테이블 활용

```sql
-- Jobs 실행 이력 조회
SELECT
    job_id,
    run_id,
    result_state,
    start_time,
    end_time,
    TIMESTAMPDIFF(MINUTE, start_time, end_time) AS duration_minutes
FROM system.lakeflow.jobs
WHERE start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY start_time DESC;
```

---

## 비용 추적

```sql
-- Job별 비용 확인
SELECT
    usage_metadata.job_id,
    SUM(usage_quantity) AS total_dbus,
    COUNT(DISTINCT usage_date) AS active_days
FROM system.billing.usage
WHERE usage_metadata.job_id IS NOT NULL
    AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY usage_metadata.job_id
ORDER BY total_dbus DESC;
```

---

## 모범 사례

| 실천 사항 | 설명 |
|-----------|------|
| **실패 알림 필수 설정** | 모든 프로덕션 Job에 실패 알림을 설정합니다 |
| **Duration Warning** | 평소보다 오래 걸리면 경고를 보내도록 설정합니다 |
| **재시도 설정** | 일시적 오류에 대비하여 1~2회 재시도를 설정합니다 |
| **태그 활용** | Job에 팀명, 프로젝트명 태그를 달아 비용을 추적합니다 |

---

## 참고 링크

- [Databricks: Monitor jobs](https://docs.databricks.com/aws/en/jobs/monitor.html)
- [Databricks: System tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/)
