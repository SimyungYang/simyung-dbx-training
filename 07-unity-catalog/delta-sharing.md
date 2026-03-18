# Delta Sharing

## 개념

> 💡 **Delta Sharing**은 조직 간에 데이터를 **안전하게 공유**할 수 있는 오픈 프로토콜입니다. 데이터를 복사하지 않고, 수신자에게 읽기 권한만 부여하여 원본 데이터에 직접 접근할 수 있게 합니다.

---

## 공유 방법

```sql
-- Share 생성
CREATE SHARE customer_insights;

-- Share에 테이블 추가
ALTER SHARE customer_insights ADD TABLE gold.customer_summary;

-- 수신자(Recipient) 생성
CREATE RECIPIENT partner_company
    USING ID 'partner-sharing-id';

-- 수신자에게 Share 부여
GRANT SELECT ON SHARE customer_insights TO RECIPIENT partner_company;
```

---

## 수신자 측에서 데이터 접근

수신자는 Databricks, Spark, pandas, Power BI 등 다양한 도구로 공유 받은 데이터에 접근할 수 있습니다.

```python
# Python (delta-sharing 패키지)
import delta_sharing
df = delta_sharing.load_as_pandas("profile.share#schema.table")
```

---

## 참고 링크

- [Databricks: Delta Sharing](https://docs.databricks.com/aws/en/data-sharing/)
- [Delta Sharing Official](https://delta.io/sharing/)
