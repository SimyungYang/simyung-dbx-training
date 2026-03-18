# 알림과 스케줄링

## SQL Alerts

> 💡 **Alert**는 SQL 쿼리의 결과가 특정 조건을 만족하면 자동으로 알림을 보내는 기능입니다.

### 설정 예시

1. SQL 쿼리 작성: `SELECT COUNT(*) AS failed_orders FROM orders WHERE status = 'FAILED' AND order_date = CURRENT_DATE()`
2. 조건 설정: `failed_orders > 100`
3. 알림 대상: 이메일, Slack Webhook
4. 체크 빈도: 매 15분

---

## 대시보드 스케줄링

대시보드의 데이터를 자동으로 갱신하고, 결과를 이메일로 전송할 수 있습니다.

| 설정 | 옵션 |
|------|------|
| **갱신 빈도** | 매시간, 매일, 매주, 커스텀 Cron |
| **전송 대상** | 이메일 주소, Slack 채널 |
| **전송 형태** | 대시보드 링크 또는 PDF 첨부 |

---

## 참고 링크

- [Databricks: Alerts](https://docs.databricks.com/aws/en/sql/user/alerts/)
