# 모델 레지스트리

## Unity Catalog 기반 모델 관리

```python
# 모델을 Unity Catalog에 등록
mlflow.set_registry_uri("databricks-uc")

model_uri = f"runs:/{run_id}/model"
mlflow.register_model(model_uri, "catalog.schema.fraud_detection_model")
```

---

## 모델 버전 관리

| 개념 | 설명 |
|------|------|
| **Version** | 모델의 각 버전에 자동으로 번호가 부여됩니다 (v1, v2, v3...) |
| **Alias** | 특정 버전에 "champion", "challenger" 같은 별칭을 부여합니다 |
| **Tags** | 모델에 메타데이터 태그를 달아 검색과 관리를 용이하게 합니다 |

```python
from mlflow import MlflowClient

client = MlflowClient()

# "champion" 별칭을 버전 3에 부여
client.set_registered_model_alias("catalog.schema.fraud_model", "champion", version=3)

# champion 모델 로드
model = mlflow.pyfunc.load_model("models:/catalog.schema.fraud_model@champion")
```

---

## 참고 링크

- [Databricks: Model Registry](https://docs.databricks.com/aws/en/mlflow/model-registry.html)
