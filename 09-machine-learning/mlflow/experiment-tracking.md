# 실험 추적

## MLflow Tracking 기본

```python
import mlflow
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score

mlflow.set_experiment("/Workspace/experiments/fraud-detection")

with mlflow.start_run(run_name="rf-baseline"):
    # 하이퍼파라미터 기록
    mlflow.log_param("n_estimators", 100)
    mlflow.log_param("max_depth", 10)
    mlflow.log_param("random_state", 42)

    # 모델 학습
    model = RandomForestClassifier(n_estimators=100, max_depth=10, random_state=42)
    model.fit(X_train, y_train)
    predictions = model.predict(X_test)

    # 메트릭 기록
    mlflow.log_metric("accuracy", accuracy_score(y_test, predictions))
    mlflow.log_metric("f1_score", f1_score(y_test, predictions))

    # 모델 저장
    mlflow.sklearn.log_model(model, "model")

    # 추가 아티팩트 (그래프, 데이터 등)
    mlflow.log_artifact("confusion_matrix.png")
```

---

## 자동 로깅 (Autolog)

```python
# 자동 로깅 활성화 - 파라미터/메트릭/모델이 자동으로 기록됩니다
mlflow.autolog()

model = RandomForestClassifier(n_estimators=100)
model.fit(X_train, y_train)
# → 자동으로 run이 생성되고 모든 정보가 기록됩니다
```

지원 프레임워크: scikit-learn, XGBoost, LightGBM, PyTorch, TensorFlow, Keras, Spark ML 등

---

## 실험 결과 비교

MLflow UI에서 여러 Run을 선택하여 파라미터와 메트릭을 비교할 수 있습니다. 최적의 하이퍼파라미터 조합을 찾는 데 유용합니다.

---

## 참고 링크

- [Databricks: MLflow Tracking](https://docs.databricks.com/aws/en/mlflow/tracking.html)
