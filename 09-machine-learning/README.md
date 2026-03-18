# 09. 머신러닝 (Machine Learning)

> Databricks 위에서 ML 모델을 개발, 추적, 배포하는 전체 과정을 학습합니다.

## 학습 목표
- MLflow를 활용한 실험 추적과 모델 관리
- Unity Catalog 기반 모델 레지스트리 사용법
- Feature Engineering과 Feature Store 활용
- Model Serving을 통한 모델 배포와 실시간 추론

## 문서 목록 (이 폴더)

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [Databricks ML 개요](ml-on-databricks-overview.md) | ML 워크플로우 전체 그림과 핵심 컴포넌트를 소개합니다 |
| 2 | [ML Runtime](ml-runtime.md) | 사전 설치 라이브러리, GPU 클러스터 구성을 다룹니다 |

## 하위 섹션

### 📂 [MLflow](mlflow/README.md)
MLflow를 활용한 실험 관리, 모델 추적, GenAI 트레이싱

| 문서 | 내용 |
|------|------|
| [MLflow란?](mlflow/what-is-mlflow.md) | 개념, 구성 요소, Databricks 통합을 설명합니다 |
| [실험 추적](mlflow/experiment-tracking.md) | 파라미터, 메트릭, 아티팩트 로깅과 Autolog을 다룹니다 |
| [모델 레지스트리](mlflow/model-registry.md) | Unity Catalog 기반 모델 등록과 버전 관리를 안내합니다 |
| [MLflow Tracing](mlflow/mlflow-tracing.md) | GenAI 앱의 호출 흐름 추적과 디버깅을 설명합니다 |
| [모델 평가](mlflow/mlflow-evaluation.md) | Scorer, 자동 평가, 프롬프트 최적화를 다룹니다 |

### 📂 [Model Serving](model-serving/README.md)
모델 서빙 — 실시간 추론 엔드포인트, Foundation Model API

| 문서 | 내용 |
|------|------|
| [Model Serving 개요](model-serving/model-serving-overview.md) | 엔드포인트 유형과 아키텍처를 설명합니다 |
| [Foundation Model API](model-serving/foundation-model-api.md) | Pay-per-token, Provisioned Throughput을 다룹니다 |
| [커스텀 모델 배포](model-serving/custom-model-deployment.md) | MLflow 모델을 엔드포인트로 배포하는 방법을 안내합니다 |
| [엔드포인트 모니터링](model-serving/endpoint-monitoring.md) | Inference Tables, 지연시간, 처리량 모니터링을 설명합니다 |

### 📂 [Feature Engineering](feature-engineering/README.md)
Feature Engineering — Feature Table, Online Store, Point-in-Time Lookup

| 문서 | 내용 |
|------|------|
| [Feature Engineering 개요](feature-engineering/feature-engineering-overview.md) | 개념, 피처 테이블, 워크플로우를 설명합니다 |
| [피처 테이블 관리](feature-engineering/feature-table-management.md) | 생성, 업데이트, 학습 데이터 결합을 다룹니다 |
| [실시간 피처 서빙](feature-engineering/online-serving.md) | Online Table, Model Serving 연동을 안내합니다 |

## 참고 문서
- [Databricks: AI and Machine Learning](https://docs.databricks.com/aws/en/machine-learning/)
- [Azure Databricks: AI and Machine Learning](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/)
