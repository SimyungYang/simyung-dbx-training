# MLflow란?

## 개념

> 💡 **MLflow**는 머신러닝 라이프사이클 전체를 관리하는 **오픈소스 플랫폼**입니다. 실험 추적, 모델 패키징, 모델 레지스트리, GenAI 트레이싱까지 지원합니다.

---

## MLflow의 핵심 컴포넌트

| 컴포넌트 | 역할 |
|----------|------|
| **Tracking** | 실험의 파라미터, 메트릭, 아티팩트를 기록합니다 |
| **Models** | 모델을 표준 형식으로 패키징하여 다양한 환경에 배포합니다 |
| **Model Registry** | 모델의 버전을 관리하고, 스테이지(Staging→Production)를 관리합니다 |
| **Tracing** | GenAI 앱의 호출 흐름(LLM 호출, Tool 호출 등)을 추적합니다 |
| **Evaluate** | 모델/에이전트의 품질을 자동으로 평가합니다 |

---

## Databricks와의 통합

Databricks에서 MLflow는 별도 설치 없이 **기본 내장**되어 있으며, Unity Catalog와 자동으로 연동됩니다.

```python
import mlflow

# Databricks에서는 자동으로 Tracking Server가 설정됩니다
mlflow.set_experiment("/Users/user@company.com/my-experiment")

with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_metric("accuracy", 0.95)
    mlflow.sklearn.log_model(model, "model")
```

> 🆕 **MLflow Traces in Unity Catalog**: MLflow 트레이스를 Unity Catalog에 저장하여 SQL로 조회할 수 있는 기능이 Preview로 출시되었습니다. 무제한 용량으로 트레이스를 보존하고 분석할 수 있습니다.

---

## 참고 링크

- [Databricks: MLflow](https://docs.databricks.com/aws/en/mlflow/)
- [MLflow Official](https://mlflow.org/)
