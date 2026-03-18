# 엔드포인트 모니터링

## Inference Tables

> 💡 **Inference Table**은 Model Serving 엔드포인트의 모든 요청과 응답을 **자동으로 Delta 테이블에 기록**하는 기능입니다.

활성화하면 다음 정보가 기록됩니다.

| 기록 항목 | 설명 |
|-----------|------|
| 요청 입력 | 모델에 전달된 입력 데이터 |
| 응답 출력 | 모델의 예측 결과 |
| 타임스탬프 | 요청/응답 시간 |
| 지연 시간 | 추론에 걸린 시간 |
| 상태 코드 | 성공/실패 여부 |

---

## 모니터링 메트릭

| 메트릭 | 설명 |
|--------|------|
| **처리량 (QPS)** | 초당 처리 요청 수 |
| **지연 시간 (Latency)** | P50, P99 응답 시간 |
| **에러율** | 실패한 요청 비율 |
| **GPU 사용률** | GPU 엔드포인트의 리소스 활용도 |

---

## 참고 링크

- [Databricks: Monitor model serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
