# Foundation Model API

## 개념

> 💡 **Foundation Model API**는 Databricks가 호스팅하는 **대규모 언어 모델(LLM)**을 즉시 사용할 수 있는 서비스입니다. 별도의 모델 배포 없이, API 호출만으로 텍스트 생성, 임베딩, 코드 생성 등을 수행할 수 있습니다.

---

## 지원 모델

### Databricks 호스팅 모델

| 제공사 | 모델 | 파라미터 | 용도 |
|--------|------|---------|------|
| **Meta** | Llama 3.3 70B Instruct | 70B | 범용 대화, 코드, 분석 |
| **Meta** | Llama 3.1 405B Instruct | 405B | 최고 성능 오픈소스 LLM |
| **Databricks** | DBRX Instruct | 132B (MoE) | 범용 대화 |
| **Databricks** | GTE Large EN | - | 영어 텍스트 임베딩 |
| **Databricks** | BGE Large EN | - | 영어 텍스트 임베딩 |

### 외부 모델 (프록시)

External Model 엔드포인트를 통해 외부 LLM도 동일한 API로 사용할 수 있습니다.

| 제공사 | 모델 | 설명 |
|--------|------|------|
| **OpenAI** | GPT-5.x 시리즈 | OpenAI의 최신 모델입니다 |
| **Anthropic** | Claude 4.x 시리즈 | Anthropic의 최신 모델입니다 |
| **Google** | Gemini 3.x 시리즈 | Google의 멀티모달 모델입니다 |
| **Qwen** | Qwen3-Embedding | 다국어 임베딩 모델입니다 |

---

## 과금 모델

| 방식 | 설명 | 비용 구조 | 적합한 사용 |
|------|------|----------|-----------|
| **Pay-per-token** | 사용한 토큰 수만큼 과금합니다. 별도 프로비저닝이 필요 없습니다 | 입력 토큰 단가 + 출력 토큰 단가 | 개발, 프로토타이핑, 불규칙한 사용 |
| **Provisioned Throughput** | 예약된 처리량을 보장합니다. 시간당 고정 과금됩니다 | 시간당 DBU 고정 비용 | 프로덕션, SLA 보장 필요, 대량 처리 |

### Pay-per-token 예시

```
입력: 500 토큰 × $0.001/1K토큰 = $0.0005
출력: 200 토큰 × $0.002/1K토큰 = $0.0004
합계: $0.0009/요청
```

### Provisioned Throughput 예시

```
예약: 모델 토큰/초 보장
비용: 시간당 고정 DBU (예: 50 DBU/시간)
장점: 일정한 응답 시간 보장, 대량 처리 시 단가 저렴
```

---

## API 사용법

### Chat Completions (대화형)

```python
import mlflow.deployments

client = mlflow.deployments.get_deploy_client("databricks")

# 기본 대화
response = client.predict(
    endpoint="databricks-meta-llama-3-3-70b-instruct",
    inputs={
        "messages": [
            {"role": "system", "content": "당신은 데이터 전문가입니다. 한국어로 답변하세요."},
            {"role": "user", "content": "Delta Lake의 ACID 트랜잭션을 설명해 주세요."}
        ],
        "max_tokens": 1000,
        "temperature": 0.1
    }
)

print(response["choices"][0]["message"]["content"])
```

### Embeddings (임베딩)

```python
# 텍스트를 벡터로 변환
response = client.predict(
    endpoint="databricks-gte-large-en",
    inputs={
        "input": ["Delta Lake is an open-source storage layer",
                  "Apache Spark is a distributed processing engine"]
    }
)

# 결과: 각 텍스트의 1024차원 벡터
embeddings = [item["embedding"] for item in response["data"]]
```

### SQL에서 사용 (ai_query)

```sql
SELECT ai_query(
    'databricks-meta-llama-3-3-70b-instruct',
    CONCAT('다음 리뷰를 한국어로 요약해 주세요: ', review_text)
) AS summary
FROM customer_reviews;
```

---

## Foundation Model API vs 외부 직접 호출 비교

| 비교 | Foundation Model API | 외부 API 직접 호출 |
|------|---------------------|------------------|
| **인증** | Databricks 토큰 | 외부 API 키 관리 필요 |
| **비용 추적** | Inference Table 자동 기록 | 수동 추적 필요 |
| **거버넌스** | MLflow Tracing 통합 | 별도 구현 필요 |
| **Rate Limiting** | Databricks에서 관리 | 외부 서비스에 의존 |
| **데이터 보안** | VPC 내 처리 가능 | 데이터가 외부로 전송 |
| **모델 전환** | 엔드포인트만 변경 | 코드 수정 필요 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Foundation Model API** | Databricks가 호스팅하는 LLM을 즉시 사용할 수 있는 서비스입니다 |
| **Pay-per-token** | 사용한 토큰만큼 과금. 개발/프로토타입에 적합합니다 |
| **Provisioned Throughput** | 처리량 보장. 프로덕션에 적합합니다 |
| **External Model** | 외부 LLM을 Databricks를 통해 프록시합니다 |

---

## 참고 링크

- [Databricks: Foundation Model APIs](https://docs.databricks.com/aws/en/machine-learning/foundation-models/)
- [Databricks: Pay-per-token](https://docs.databricks.com/aws/en/machine-learning/foundation-models/pay-per-token.html)
- [Databricks: Provisioned Throughput](https://docs.databricks.com/aws/en/machine-learning/foundation-models/provisioned-throughput.html)
