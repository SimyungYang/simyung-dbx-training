# 09-2. Model Serving

> 학습된 모델을 실시간 추론 엔드포인트로 배포하는 방법을 학습합니다.

## 학습 목표
- Model Serving 엔드포인트의 종류와 선택 기준
- Foundation Model API를 통한 LLM 활용
- 커스텀 모델 배포 (PyFunc, Transformer)
- 엔드포인트 모니터링과 A/B 테스트

## 문서 목록

| 순서 | 파일명 | 내용 |
|------|--------|------|
| 1 | `model-serving-overview.md` | Model Serving 개요 — 엔드포인트 유형, 아키텍처 |
| 2 | `foundation-model-api.md` | Foundation Model API — Pay-per-token, Provisioned Throughput |
| 3 | `custom-model-deployment.md` | 커스텀 모델 배포 — MLflow 모델을 엔드포인트로 배포 |
| 4 | `endpoint-monitoring.md` | 모니터링 — 지연시간, 처리량, 드리프트 감지 |

## 참고 문서
- [Databricks: Model Serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/)
