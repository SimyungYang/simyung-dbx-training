# Model Serving 개요

## 개념

> 💡 **Model Serving**은 학습된 ML 모델을 **실시간 추론 API 엔드포인트**로 배포하는 서비스입니다. REST API로 예측 요청을 보내고 응답을 받을 수 있습니다.

---

## 엔드포인트 유형

| 유형 | 설명 | 적합한 사용 |
|------|------|-----------|
| **Foundation Model API** | Databricks가 호스팅하는 LLM (Llama, DBRX 등) | LLM 활용, 빠른 프로토타이핑 |
| **External Model** | OpenAI, Anthropic 등 외부 LLM을 프록시 | 외부 LLM 통합 관리 |
| **Custom Model** | MLflow로 등록한 사용자 모델 배포 | 커스텀 ML 모델 서빙 |

---

## Foundation Model API

```python
import mlflow.deployments

client = mlflow.deployments.get_deploy_client("databricks")

response = client.predict(
    endpoint="databricks-meta-llama-3-3-70b-instruct",
    inputs={
        "messages": [
            {"role": "user", "content": "데이터 레이크하우스란 무엇인가요?"}
        ],
        "max_tokens": 500
    }
)
print(response["choices"][0]["message"]["content"])
```

---

## 참고 링크

- [Databricks: Model Serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/)
