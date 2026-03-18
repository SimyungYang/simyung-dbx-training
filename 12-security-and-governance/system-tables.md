# 시스템 테이블

## 시스템 테이블이란?

> 💡 **시스템 테이블(System Tables)**은 Databricks 플랫폼의 운영 데이터(감사 로그, 빌링, 리니지, 사용량 등)를 **SQL로 직접 조회**할 수 있는 내장 테이블입니다.

---

## 주요 시스템 테이블

| 스키마 | 테이블 | 내용 |
|--------|--------|------|
| `system.access` | `audit` | 누가 어떤 작업을 했는지 감사 로그 |
| `system.billing` | `usage` | DBU 사용량, 비용 정보 |
| `system.billing` | `list_prices` | DBU 단가 정보 |
| `system.compute` | `clusters` | 클러스터 설정 및 이벤트 |
| `system.lakeflow` | `jobs` | Job 실행 이력 |
| `system.lineage` | `table_lineage` | 테이블 간 리니지 |
| `system.lineage` | `column_lineage` | 컬럼 간 리니지 |

---

## 활용 예시

```sql
-- 지난 30일간 사용자별 DBU 사용량 TOP 10
SELECT
    identity_metadata.run_as AS user,
    SUM(usage_quantity) AS total_dbus
FROM system.billing.usage
WHERE usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;

-- 최근 실패한 Job 확인
SELECT
    job_id, run_id, result_state, error_message, start_time
FROM system.lakeflow.jobs
WHERE result_state = 'FAILED'
    AND start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY start_time DESC;
```

---

## 참고 링크

- [Databricks: System tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/)
