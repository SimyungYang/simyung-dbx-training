# 모델 평가

## MLflow Evaluate

```python
import mlflow

# 평가 데이터셋 준비
eval_data = [
    {"inputs": {"question": "반품 정책이 뭔가요?"}, "expectations": {"expected_response": "30일 이내 무료 반품 가능"}},
    {"inputs": {"question": "배송 기간은?"}, "expectations": {"expected_response": "2~3 영업일"}},
]

# 내장 Scorer로 평가
results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=my_agent.predict,
    scorers=[
        mlflow.genai.scorers.Correctness(),
        mlflow.genai.scorers.Safety(),
        mlflow.genai.scorers.RetrievalGroundedness(),
    ]
)

print(results.metrics)
```

---

## 주요 내장 Scorer

| Scorer | 평가 내용 |
|--------|----------|
| **Correctness** | 답변이 기대 답변과 의미적으로 일치하는지 |
| **Safety** | 답변에 유해하거나 부적절한 내용이 없는지 |
| **RetrievalGroundedness** | 답변이 검색된 문서에 근거하는지 |
| **Guidelines** | 사용자 정의 가이드라인을 준수하는지 |

---

## 참고 링크

- [Databricks: MLflow Evaluation](https://docs.databricks.com/aws/en/mlflow/llm-evaluate.html)
