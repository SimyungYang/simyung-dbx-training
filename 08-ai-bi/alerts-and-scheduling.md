# 알림과 스케줄링

## 왜 알림과 스케줄링이 필요한가?

데이터 기반 의사결정을 위해서는 **문제가 발생했을 때 즉시 알아차리는 것** 이 핵심입니다. 사람이 대시보드를 24시간 지켜볼 수 없으므로, 자동화된 알림과 스케줄링이 필수적입니다.

| 시나리오 | 수동 모니터링 | 자동 알림 |
|----------|-------------|-----------|
| 매출 급감 감지 | 매일 대시보드를 열어 확인해야 합니다 | 매출이 전주 대비 20% 이하로 떨어지면 즉시 Slack 알림을 받습니다 |
| ETL 파이프라인 지연 | 누군가 데이터가 오래되었다고 알려줄 때까지 모릅니다 | 테이블 갱신이 4시간 이상 지연되면 자동으로 이메일이 발송됩니다 |
| 비용 초과 | 월말 청구서를 받고 나서야 알 수 있습니다 | 일일 DBU 사용량이 임계치를 넘으면 PagerDuty로 긴급 알림을 보냅니다 |
| 경영진 보고 | 매주 수동으로 대시보드 PDF를 만들어 보냅니다 | 매주 월요일 오전 9시에 대시보드가 자동 갱신되고 PDF가 이메일로 전송됩니다 |

> 💡 **알림 피로(Alert Fatigue)란?** 너무 많은 알림이 발생하면 중요한 알림도 무시하게 되는 현상입니다. 적절한 임계치 설정과 알림 채널 분리가 중요합니다.

---

## SQL Alert 개요

> 💡 **Alert(알림)** 는 SQL 쿼리를 주기적으로 실행하여, 결과가 ** 특정 조건을 만족하면 자동으로 알림을 발송**하는 기능입니다. 이상 거래 감지, SLA 위반 모니터링, 재고 부족 경고 등에 활용됩니다.

### Alert의 동작 방식

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 스케줄 (매 15분) | 주기적으로 SQL 쿼리를 실행합니다 |
| 2 | SQL 쿼리 실행 | 모니터링할 값을 조회합니다 |
| 3 | 조건 평가 | value > threshold 여부를 확인합니다 |
| 4a | 조건 미충족 (No) | OK 상태를 유지하고 다음 스케줄을 기다립니다 |
| 4b | 조건 충족 (Yes) | 상태 변경 여부를 확인합니다 |
| 5a | OK → TRIGGERED | 알림을 발송합니다 |
| 5b | 이미 TRIGGERED | 알림 빈도 설정을 확인하여, 매번 알림이면 발송하고, 한번만 알림이면 추가 알림을 생략합니다 |

---

## Alert 상태 전이

Alert는 세 가지 상태를 가지며, 상태 변화에 따라 알림이 발송됩니다.

| 상태 | 아이콘 | 의미 | 알림 발송 |
|------|--------|------|-----------|
| **OK** | 🟢 | 조건을 만족하지 않는 정상 상태입니다 | 상태가 TRIGGERED에서 OK로 변경될 때 (설정 시) |
| **TRIGGERED** | 🔴 | 조건이 만족되어 알림이 발생한 상태입니다 | OK에서 TRIGGERED로 변경될 때 |
| **UNKNOWN** | ⚪ | 쿼리가 아직 실행되지 않았거나 실행에 실패한 상태입니다 | 없음 |

### 알림 빈도 설정

| 설정 | 동작 | 권장 상황 |
|------|------|-----------|
| **Just once** | 상태가 OK → TRIGGERED로 변경될 때 한 번만 알림을 보냅니다 | 한 번 인지하면 충분한 경우 |
| **Each time alert is evaluated** | 스케줄마다 조건을 만족하면 매번 알림을 보냅니다 | 지속적인 모니터링이 필요한 경우 |
| **At most every...** | 지정한 시간 간격으로 최대 한 번 알림을 보냅니다 | 알림 피로를 줄이면서도 추적이 필요한 경우 |

---

## Alert 생성 방법

### UI에서 생성

1. **SQL Editor** 또는 **Alerts** 페이지로 이동합니다
2. **Create Alert** 클릭합니다
3. 모니터링할 SQL 쿼리를 작성합니다
4. 조건(Condition)을 설정합니다
5. 알림 빈도(Trigger)를 선택합니다
6. 알림 대상(Destination)을 추가합니다
7. 스케줄을 설정합니다

### 조건 설정

| 설정 항목 | 설명 | 예시 |
|-----------|------|------|
| **Value Column** | 모니터링할 쿼리 결과 컬럼입니다 | `failed_orders` |
| **Condition** | 비교 연산자입니다 | `>`, `>=`, `<`, `<=`, `=`, `!=` |
| **Threshold** | 비교할 임계값입니다 | `100` |
| **Empty Result** | 쿼리 결과가 비어있을 때 동작입니다 | TRIGGERED 또는 OK |

### 쿼리 예시

```sql
-- 1. 실패 주문 수 모니터링
-- 조건: failed_orders > 100 이면 TRIGGERED
SELECT COUNT(*) AS failed_orders
FROM production.ecommerce.orders
WHERE status = 'FAILED'
  AND order_date >= CURRENT_TIMESTAMP() - INTERVAL 1 HOUR;

-- 2. 파이프라인 지연 감지
-- 조건: stale_tables > 0 이면 TRIGGERED
SELECT COUNT(*) AS stale_tables
FROM information_schema.tables
WHERE table_schema = 'gold'
  AND TIMESTAMPDIFF(HOUR, last_altered, CURRENT_TIMESTAMP()) > 4;

-- 3. 비용 급증 감지
-- 조건: pct_of_last_week > 150 이면 TRIGGERED (50% 이상 급증)
WITH today AS (
    SELECT SUM(usage_quantity) AS dbus
    FROM system.billing.usage
    WHERE usage_date = CURRENT_DATE()
),
last_week AS (
    SELECT SUM(usage_quantity) AS dbus
    FROM system.billing.usage
    WHERE usage_date = CURRENT_DATE() - 7
)
SELECT ROUND(today.dbus / NULLIF(last_week.dbus, 0) * 100, 1) AS pct_of_last_week
FROM today, last_week;

-- 4. 데이터 품질 이상 감지
-- 조건: null_ratio > 5 이면 TRIGGERED (NULL 비율 5% 초과)
SELECT
    ROUND(COUNT_IF(customer_id IS NULL) * 100.0 / COUNT(*), 2) AS null_ratio
FROM production.ecommerce.orders
WHERE order_date >= CURRENT_DATE() - INTERVAL 1 DAY;
```

---

## 알림 대상(Destination) 설정

Alert가 TRIGGERED되면 지정된 채널로 알림이 발송됩니다.

| 채널 | 설정 방법 | 페이로드 |
|------|-----------|----------|
| **이메일** | 사용자 또는 그룹 이메일 주소를 입력합니다 | 알림 이름, 상태, 쿼리 결과 포함 |
| **Slack** | Incoming Webhook URL을 등록합니다 | JSON 메시지로 알림 정보 전송 |
| **PagerDuty** | Integration Key를 등록합니다 | 인시던트 자동 생성 |
| **Microsoft Teams** | Webhook URL을 등록합니다 | Adaptive Card 형식 메시지 |
| **Generic Webhook** | 커스텀 URL에 HTTP POST를 보냅니다 | JSON 페이로드 전송 |

### Webhook 페이로드 구조

Generic Webhook으로 전송되는 JSON 페이로드는 다음과 같은 구조입니다.

```json
{
    "alert_id": "alert-12345",
    "alert_name": "Failed Orders Alert",
    "alert_state": "TRIGGERED",
    "alert_condition": {
        "column": "failed_orders",
        "operator": ">",
        "threshold": 100
    },
    "query_result": {
        "failed_orders": 157
    },
    "alert_url": "https://workspace.databricks.com/sql/alerts/12345",
    "triggered_at": "2025-01-15T09:30:00Z"
}
```

### Slack 연동 설정

1. Slack 앱에서 **Incoming Webhook** 을 생성합니다
2. Webhook URL을 복사합니다 (예: `https://hooks.slack.com/services/T.../B.../xxx`)
3. Databricks의 **SQL > Alert Destinations** 에서 **New Destination** 클릭합니다
4. **Slack** 을 선택하고 Webhook URL을 입력합니다
5. Alert 설정에서 해당 Destination을 추가합니다

---

## 대시보드 스케줄링

### 자동 새로고침 설정

Lakeview 대시보드는 **스케줄을 설정하여 자동으로 데이터를 갱신** 할 수 있습니다.

| 설정 항목 | 옵션 | 설명 |
|-----------|------|------|
| **갱신 빈도** | 매시간, 매일, 매주, 커스텀 Cron | 데이터를 얼마나 자주 새로고침할지 설정합니다 |
| **SQL Warehouse** | Serverless, Pro, Classic | 쿼리를 실행할 Warehouse를 선택합니다 |
| ** 타임존** | UTC, Asia/Seoul 등 | 스케줄 기준 시간대를 설정합니다 |

### 구독자 관리 및 이메일 전송

대시보드 스케줄에 구독자를 추가하면, 갱신될 때마다 자동으로 결과를 전달할 수 있습니다.

| 전송 형식 | 설명 | 적합한 상황 |
|-----------|------|-------------|
| ** 링크** | 대시보드 URL만 이메일로 전송합니다 | 항상 최신 데이터를 직접 확인하는 사용자 |
| **PDF 첨부** | 대시보드를 PDF로 렌더링하여 첨부합니다 | 오프라인에서도 보고서를 확인하는 경영진 |
| ** 인라인 이미지** | 차트를 이미지로 이메일 본문에 삽입합니다 | 빠르게 핵심 지표를 확인하려는 사용자 |

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 스케줄 트리거 | 매일 오전 9시에 실행됩니다 |
| 2 | 대시보드 데이터 갱신 | 최신 데이터로 대시보드를 갱신합니다 |
| 3 | 구독자 확인 | 구독자가 있는지 확인합니다 |
| 4a | 구독자 있음 (Yes) | 이메일을 전송합니다 (PDF/링크/이미지) |
| 4b | 구독자 없음 (No) | 갱신만 완료합니다 |

### 파라미터화된 대시보드 스케줄

대시보드에 파라미터가 있는 경우, 스케줄마다 ** 다른 파라미터 값**을 지정하여 여러 버전의 보고서를 자동 생성할 수 있습니다.

```
예시: 지역별 매출 보고서
- 스케줄 1: region = 'APAC', 수신자: APAC 팀
- 스케줄 2: region = 'EMEA', 수신자: EMEA 팀
- 스케줄 3: region = 'NA', 수신자: NA 팀
```

---

## 쿼리 스케줄링

SQL 쿼리 자체에도 스케줄을 설정하여 주기적으로 실행할 수 있습니다.

| 설정 | 설명 |
|------|------|
| ** 실행 주기** | 분, 시간, 일, 주, 월, 커스텀 Cron 표현식을 지원합니다 |
| **SQL Warehouse** | 쿼리를 실행할 Warehouse를 지정합니다 |
| ** 실패 알림** | 쿼리 실행 실패 시 알림을 받을 수 있습니다 |

> 💡 ** 쿼리 스케줄 vs Jobs**: 단순한 SQL 실행은 쿼리 스케줄이 편리하고, 여러 단계의 복잡한 워크플로가 필요하면 **Workflows(Jobs)** 를 사용하는 것이 좋습니다.

---

## 실습: 알림 생성부터 Slack 연동까지

### Step 1: 모니터링 쿼리 작성

```sql
-- SQL Editor에서 새 쿼리 생성
-- 이름: "Failed Orders Monitor"
SELECT
    COUNT(*) AS failed_count,
    MAX(order_date) AS latest_failure
FROM production.ecommerce.orders
WHERE status = 'FAILED'
  AND order_date >= CURRENT_TIMESTAMP() - INTERVAL 1 HOUR;
```

### Step 2: Alert 생성

1. 쿼리 결과 패널에서 **Create Alert** 클릭합니다
2. 조건 설정:
   - Value Column: `failed_count`
   - Condition: `>`
   - Threshold: `50`
3. 알림 빈도: **At most every 30 minutes** 선택합니다
4. **Save** 클릭합니다

### Step 3: Slack Destination 추가

1. **Alert Destinations** 탭에서 Slack Destination을 선택합니다
2. 알림 메시지가 지정된 Slack 채널로 전송됩니다

### Step 4: 스케줄 설정

1. **Schedule** 탭에서 **Every 15 minutes** 선택합니다
2. SQL Warehouse: **Serverless** 선택합니다
3. **Save** 클릭합니다

---

## 모범 사례

### 알림 설계 원칙

| 원칙 | 설명 | 예시 |
|------|------|------|
| ** 의미 있는 임계치** | 비즈니스 영향도를 고려하여 임계치를 설정합니다 | "1건의 실패"보다는 "전체 대비 5% 이상 실패"가 의미 있습니다 |
| ** 알림 채널 분리** | 긴급도에 따라 채널을 분리합니다 | 긴급: PagerDuty, 주의: Slack, 정보성: 이메일 |
| ** 알림 피로 방지** | 너무 자주, 너무 많은 알림은 무시됩니다 | 빈도 제한, 적절한 임계치, 그룹화 활용 |
| ** 문맥 포함** | 알림에 충분한 정보를 포함합니다 | "실패 50건" 대신 "지난 1시간 실패 50건 (평소 5건)" |
| ** 해결 방법 포함** | 알림에 다음 단계를 안내합니다 | 관련 대시보드 링크, 런북 URL 포함 |

### 스케줄 설정 모범 사례

| 항목 | 권장 사항 |
|------|-----------|
| ** 대시보드 갱신** | 사용자가 출근하기 전(예: 오전 8시)에 갱신이 완료되도록 합니다 |
| ** 알림 체크 주기** | 비즈니스 SLA에 맞춰 설정합니다 (예: SLA 1시간 → 알림 15분 간격) |
| **Warehouse 선택** | 스케줄 작업에는 Serverless를 사용하여 유휴 비용을 줄입니다 |
| ** 타임존** | 팀의 근무 시간대에 맞춰 설정합니다 |
| **PDF 보고서** | 용량이 크므로 일/주 단위로 전송합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SQL Alert** | 쿼리 결과가 조건을 만족하면 자동으로 알림을 발송하는 기능입니다 |
| **Alert 상태** | OK, TRIGGERED, UNKNOWN 세 가지 상태로 전이됩니다 |
| **Destination** | 이메일, Slack, PagerDuty, Teams, Webhook 등으로 알림을 전송합니다 |
| ** 대시보드 스케줄** | 대시보드 데이터를 자동 갱신하고 구독자에게 전달합니다 |
| ** 파라미터 스케줄** | 파라미터별로 다른 스케줄을 설정하여 맞춤 보고서를 생성합니다 |
| ** 알림 피로 방지** | 적절한 임계치와 빈도 설정으로 중요한 알림에 집중합니다 |

---

## 참고 링크

- [Databricks: Alerts](https://docs.databricks.com/aws/en/sql/user/alerts/)
- [Databricks: Dashboard schedules](https://docs.databricks.com/aws/en/aibi/dashboards/schedule.html)
- [Databricks: Alert destinations](https://docs.databricks.com/aws/en/sql/admin/alert-destinations.html)
- [Databricks: Query schedules](https://docs.databricks.com/aws/en/sql/user/queries/schedule-query.html)
- [Azure Databricks: Alerts](https://learn.microsoft.com/en-us/azure/databricks/sql/user/alerts/)
