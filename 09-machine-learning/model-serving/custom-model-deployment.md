# 커스텀 모델 배포

## MLflow 모델을 엔드포인트로 배포

```python
import mlflow.deployments

client = mlflow.deployments.get_deploy_client("databricks")

# 엔드포인트 생성
endpoint = client.create_endpoint(
    name="fraud-detection-v2",
    config={
        "served_entities": [{
            "entity_name": "catalog.schema.fraud_model",
            "entity_version": "3",
            "workload_size": "Small",
            "scale_to_zero_enabled": True
        }]
    }
)
```

---

## 엔드포인트에 요청 보내기

```python
import requests

url = "https://<workspace-url>/serving-endpoints/fraud-detection-v2/invocations"
headers = {"Authorization": f"Bearer {token}"}
data = {"dataframe_records": [{"amount": 50000, "merchant": "online_store", "hour": 3}]}

response = requests.post(url, json=data, headers=headers)
print(response.json())
```

---

## A/B 테스트 (Traffic Split)

하나의 엔드포인트에 여러 모델 버전을 배포하고, 트래픽 비율을 조절하여 A/B 테스트를 수행할 수 있습니다.

---

## 참고 링크

- [Databricks: Deploy custom models](https://docs.databricks.com/aws/en/machine-learning/model-serving/create-manage-serving-endpoints.html)
