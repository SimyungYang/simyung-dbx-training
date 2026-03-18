# 에이전트 평가

## 왜 평가가 중요한가요?

AI 에이전트는 비결정적(같은 입력에 매번 다른 출력)이므로, 체계적인 평가 없이는 품질을 보장할 수 없습니다. MLflow의 `evaluate()` 함수를 사용하여 자동화된 평가를 수행할 수 있습니다.

---

## 평가 워크플로우

```python
import mlflow

# 1. 평가 데이터셋 준비
eval_data = spark.table("catalog.schema.eval_dataset").toPandas()

# 2. 평가 실행
results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=my_agent.predict,
    scorers=[
        mlflow.genai.scorers.Correctness(),
        mlflow.genai.scorers.Safety(),
        mlflow.genai.scorers.RetrievalGroundedness(),
        mlflow.genai.scorers.Guidelines(
            guidelines="답변은 반드시 한국어로 작성되어야 합니다"
        )
    ]
)

# 3. 결과 확인
print(results.metrics)
display(results.tables["eval_results"])
```

---

## 참고 링크

- [Databricks: Evaluate agents](https://docs.databricks.com/aws/en/generative-ai/agent-evaluation/)
