# Foundation Model API

## 지원 모델

Databricks는 주요 LLM을 **Pay-per-token** 방식으로 제공합니다.

| 제공사 | 모델 예시 |
|--------|----------|
| **Meta** | Llama 3.3 70B, Llama 3.1 405B |
| **Databricks** | DBRX |
| **OpenAI** | GPT-5.x 시리즈 |
| **Anthropic** | Claude Sonnet/Opus 4.x |
| **Google** | Gemini 3.x |

---

## 과금 모델

| 방식 | 설명 | 적합한 경우 |
|------|------|-----------|
| **Pay-per-token** | 사용한 토큰 수만큼 과금 | 불규칙한 사용, 프로토타이핑 |
| **Provisioned Throughput** | 예약된 처리량 보장, 시간당 과금 | 프로덕션, 안정적 처리량 필요 |

---

## 참고 링크

- [Databricks: Foundation Model APIs](https://docs.databricks.com/aws/en/machine-learning/foundation-models/)
