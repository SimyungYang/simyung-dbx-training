# 09-2. Model Serving

> 학습된 모델을 실시간 추론 엔드포인트로 배포하는 방법을 학습합니다.

## 학습 목표
- Model Serving 엔드포인트의 종류와 선택 기준
- Foundation Model API를 통한 LLM 활용
- 커스텀 모델 배포 (PyFunc, Transformer)
- 엔드포인트 모니터링과 A/B 테스트

## 문서 목록

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [Model Serving 개요](model-serving-overview.md) | 엔드포인트 유형(Foundation/External/Custom)과 아키텍처를 설명합니다 |
| 2 | [Foundation Model API](foundation-model-api.md) | Pay-per-token, Provisioned Throughput, 지원 모델을 다룹니다 |
| 3 | [커스텀 모델 배포](custom-model-deployment.md) | MLflow 모델을 엔드포인트로 배포하고 A/B 테스트하는 방법을 안내합니다 |
| 4 | [엔드포인트 모니터링](endpoint-monitoring.md) | Inference Tables, 지연시간, 처리량, 에러율 모니터링을 설명합니다 |

## 참고 문서
- [Databricks: Model Serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/)
