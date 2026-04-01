# 운영 대시보드와 활성화 방법

## 운영 대시보드 구성 가이드

시스템 테이블을 활용한 AI/BI 대시보드를 구축하면, 한눈에 플랫폼 운영 상황을 파악할 수 있습니다.

### 권장 대시보드 패널 구성

| 대시보드 | 주요 패널 | 데이터 소스 |
|---------|----------|-----------|
| **비용 관리** | 월별 DBU 추이, 팀별 비용, 비용 급증 알림 | `system.billing.usage` |
| **보안 감사** | 로그인 실패, 권한 변경, 대량 다운로드 | `system.access.audit` |
| **Job 모니터링** | 실패율, 실행시간 추이, SLA 위반 | `system.lakeflow.jobs` |
| **클러스터 현황** | 유휴 클러스터, 활용률, 비용 | `system.compute.clusters` |
| **데이터 거버넌스** | 리니지 맵, 미사용 테이블, 접근 패턴 | `system.lineage.*`, `system.access.audit` |

### 알림 설정 예시

```sql
-- SQL Alert: 일일 비용이 $500을 초과하면 알림
SELECT
    usage_date,
    SUM(usage_quantity) AS total_dbus,
    ROUND(SUM(usage_quantity) * 0.22, 2) AS estimated_cost_usd
FROM system.billing.usage
WHERE usage_date = CURRENT_DATE() - INTERVAL 1 DAY
GROUP BY usage_date
HAVING estimated_cost_usd > 500;
```

> 💡 위 쿼리를 **SQL Alert**로 등록하고, Slack/이메일로 알림을 설정하면 비용 이상을 즉시 감지할 수 있습니다.

---

## 활성화 방법

시스템 테이블은 **계정 관리자(Account Admin)** 가 활성화해야 합니다.

1. **Account Console** → **Settings** → **Feature Enablement**
2. **System Tables** → **Enable**
3. 활성화 후 데이터가 채워지기까지 최대 24시간이 소요될 수 있습니다

> 💡 시스템 테이블의 데이터는 **읽기 전용**이며, Unity Catalog의 `system` 카탈로그에 위치합니다. 조회하려면 `USE CATALOG system` 권한이 필요합니다.

---

## 정리

| 핵심 카테고리 | 대표 테이블 | 활용 |
|-------------|-----------|------|
| **감사** | `system.access.audit` | 누가 무엇을 했는지 추적 |
| **빌링** | `system.billing.usage` | 비용 분석, 예산 관리 |
| **컴퓨트** | `system.compute.clusters` | 유휴 리소스 감지 |
| **워크플로우** | `system.lakeflow.jobs` | Job 성공/실패 모니터링 |
| **리니지** | `system.lineage.table_lineage` | 데이터 흐름 추적 |

---

## 참고 링크

- [Databricks: System tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/)
- [Databricks: Billing usage](https://docs.databricks.com/aws/en/administration-guide/system-tables/billing.html)
- [Databricks: Audit logs](https://docs.databricks.com/aws/en/administration-guide/system-tables/audit-logs.html)
- [Azure Databricks: System tables](https://learn.microsoft.com/en-us/azure/databricks/administration-guide/system-tables/)
