# 에이전트 배포

## Model Serving으로 배포

```python
import mlflow

# 에이전트를 MLflow 모델로 로깅
with mlflow.start_run():
    mlflow.pyfunc.log_model(
        artifact_path="agent",
        python_model=MyAgent(),
        registered_model_name="catalog.schema.customer_support_agent"
    )

# Model Serving 엔드포인트로 배포
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
w.serving_endpoints.create(
    name="customer-support-agent",
    config={
        "served_entities": [{
            "entity_name": "catalog.schema.customer_support_agent",
            "entity_version": "1",
            "workload_size": "Small",
            "environment_vars": {"VECTOR_SEARCH_ENDPOINT": "vs-endpoint"}
        }]
    }
)
```

---

## Review App

배포 전에 **Review App**을 통해 팀원들이 에이전트를 테스트하고 피드백을 남길 수 있습니다. 수집된 피드백은 평가 데이터셋으로 활용됩니다.

---

## 참고 링크

- [Databricks: Deploy agents](https://docs.databricks.com/aws/en/generative-ai/deploy-agent.html)
