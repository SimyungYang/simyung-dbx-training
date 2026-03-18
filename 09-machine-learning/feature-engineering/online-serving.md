# 실시간 피처 서빙

## Online Table

> 💡 **Online Table**은 Feature Table의 데이터를 **실시간 조회에 최적화된 형태**로 자동 동기화하는 기능입니다. Model Serving 엔드포인트에서 밀리초 수준으로 피처를 조회할 수 있습니다.

```sql
-- Online Table 생성
CREATE ONLINE TABLE catalog.schema.customer_features_online
FROM catalog.schema.customer_features;
```

---

## Model Serving과 연동

Online Table이 활성화되면, Model Serving 엔드포인트에서 피처를 자동으로 조회하여 모델에 전달합니다. 클라이언트는 Primary Key만 전송하면 됩니다.

```python
# 클라이언트: customer_id만 전송하면, 피처는 자동 조회됨
response = requests.post(endpoint_url, json={
    "dataframe_records": [{"customer_id": 12345}]
})
```

---

## 참고 링크

- [Databricks: Online tables](https://docs.databricks.com/aws/en/machine-learning/feature-store/online-tables.html)
