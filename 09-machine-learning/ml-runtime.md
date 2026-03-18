# ML Runtime

## ML Runtime이란?

> 💡 **ML Runtime**은 머신러닝 작업에 필요한 라이브러리가 **사전 설치된** Databricks Runtime입니다. TensorFlow, PyTorch, scikit-learn, XGBoost 등 주요 ML 프레임워크가 포함되어 있어, 별도 설치 없이 바로 모델 학습을 시작할 수 있습니다.

---

## 포함된 주요 라이브러리

| 카테고리 | 라이브러리 |
|----------|-----------|
| **ML 프레임워크** | scikit-learn, XGBoost, LightGBM |
| **딥러닝** | PyTorch, TensorFlow, Keras |
| **데이터 처리** | pandas, NumPy, SciPy |
| **시각화** | matplotlib, seaborn, plotly |
| **NLP** | Hugging Face Transformers |
| **ML 관리** | MLflow |

---

## GPU Runtime

딥러닝이나 LLM 파인튜닝에는 **GPU Runtime**을 사용합니다. GPU 드라이버(CUDA, cuDNN)가 사전 설치되어 있습니다.

| 용도 | 권장 GPU 인스턴스 |
|------|-----------------|
| 소규모 학습/추론 | g4dn (NVIDIA T4) |
| 중규모 학습 | p3 (NVIDIA V100) |
| LLM 파인튜닝 | p4d (NVIDIA A100) |

---

## 참고 링크

- [Databricks: ML Runtime](https://docs.databricks.com/aws/en/release-notes/runtime/ml.html)
