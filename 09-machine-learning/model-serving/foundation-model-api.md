# Foundation Model API

## 개념

> 💡 **Foundation Model API**는 Databricks가 호스팅하는 **대규모 언어 모델(LLM)** 을 즉시 사용할 수 있는 서비스입니다. 별도의 모델 배포 없이, API 호출만으로 텍스트 생성, 임베딩, 코드 생성 등을 수행할 수 있습니다.

클라우드 환경에서 LLM을 직접 배포하고 운영하려면 GPU 인스턴스 선택, 모델 로딩, 스케일링, 모니터링 등 상당한 인프라 전문 지식이 필요합니다. Foundation Model API는 이 모든 복잡성을 Databricks가 관리해 주므로, 개발자는 API 호출만으로 최신 LLM을 즉시 활용할 수 있습니다. OpenAI API와 호환되는 인터페이스를 제공하므로, 기존 코드의 마이그레이션도 간편합니다.

---

## 지원 모델

### Databricks 호스팅 모델

Databricks가 직접 호스팅하고 관리하는 모델입니다. 별도의 배포 과정 없이 즉시 API를 호출할 수 있으며, Databricks 인프라 내에서 처리되므로 데이터가 외부로 전송되지 않습니다.

| 제공사 | 모델 | 파라미터 | 용도 | 특징 |
|--------|------|---------|------|------|
| **Meta**| Llama 3.3 70B Instruct | 70B | 범용 대화, 코드, 분석 | 오픈소스, 다국어 지원, 가장 균형 잡힌 모델 |
| **Meta**| Llama 3.1 405B Instruct | 405B | 최고 성능 오픈소스 LLM | 복잡한 추론, 긴 컨텍스트 (128K) |
| **Meta**| Llama 3.1 8B Instruct | 8B | 경량 작업, 분류, 요약 | 빠른 응답, 낮은 비용 |
| **Databricks**| DBRX Instruct | 132B (MoE) | 범용 대화 | MoE 아키텍처로 효율적 |
| **Databricks**| GTE Large EN | - | 영어 텍스트 임베딩 | 1024차원 벡터, 검색에 최적화 |
| **Databricks**| BGE Large EN | - | 영어 텍스트 임베딩 | 1024차원 벡터, RAG에 적합 |

> 💡 **모델 선택 가이드**: 일반적인 대화와 분석에는 **Llama 3.3 70B**가 가장 좋은 균형(성능/비용)을 제공합니다. 매우 복잡한 추론이 필요하면 **405B**, 단순 분류나 요약은 **8B** 로 비용을 절약할 수 있습니다.

### 외부 모델 (External Model / 프록시)

External Model 엔드포인트를 통해 외부 LLM도 **동일한 Databricks API** 로 사용할 수 있습니다. 외부 API 키를 Databricks Secret에 안전하게 저장하고, Databricks의 거버넌스(로깅, 모니터링, 접근 제어)를 적용할 수 있다는 것이 핵심 장점입니다.

| 제공사 | 모델 | 설명 | 주요 용도 |
|--------|------|------|----------|
| **OpenAI**| GPT-4o, GPT-4o-mini 등 | OpenAI의 최신 모델입니다 | 범용 대화, 코드, 멀티모달 |
| **Anthropic**| Claude Sonnet/Haiku 등 | Anthropic의 최신 모델입니다 | 긴 문서 분석, 안전한 AI |
| **Google**| Gemini Pro/Flash 등 | Google의 멀티모달 모델입니다 | 멀티모달, 코드 |
| **Qwen**| Qwen3-Embedding | 다국어 임베딩 모델입니다 | 한국어 포함 다국어 임베딩 |
| **Cohere**| Command R+, Embed | 기업용 LLM입니다 | RAG, 검색, 다국어 |

### External Model 엔드포인트 생성

```python
import mlflow.deployments

client = mlflow.deployments.get_deploy_client("databricks")

# OpenAI GPT-4o를 Databricks에서 프록시
endpoint = client.create_endpoint(
    name="gpt-4o-proxy",
    config={
        "served_entities": [{
            "external_model": {
                "name": "gpt-4o",
                "provider": "openai",
                "openai_config": {
                    "openai_api_key": "{{secrets/llm-keys/openai-key}}"
                }
            }
        }]
    }
)
```

---

## API 유형 상세

Foundation Model API는 세 가지 주요 API 유형을 제공합니다. 각 API는 OpenAI API와 호환되는 형식을 사용하므로 기존 코드를 쉽게 마이그레이션할 수 있습니다.

### Chat Completions (대화형)

가장 많이 사용되는 API로, 대화 형식의 메시지를 주고받습니다. 시스템 프롬프트로 LLM의 역할을 정의하고, 사용자 메시지에 대한 응답을 생성합니다.

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

#### 주요 파라미터

| 파라미터 | 설명 | 기본값 | 권장 범위 |
|---------|------|--------|----------|
| **max_tokens**| 생성할 최대 토큰 수입니다 | 모델별 상이 | 100~4000 |
| **temperature**| 응답의 무작위성입니다. 낮을수록 결정적 | 1.0 | 분석: 0.0~0.3 / 창작: 0.7~1.0 |
| **top_p**| nucleus sampling의 확률 임계값입니다 | 1.0 | 0.1~0.95 |
| **stop**| 생성을 중단할 시퀀스입니다 | None | 필요 시 설정 |
| **stream**| 스트리밍 응답 여부입니다 | false | 실시간 UI에서 true |

### Completions (텍스트 완성)

프롬프트 뒤에 이어지는 텍스트를 생성합니다. Chat 형식이 아닌 단순 텍스트 완성 작업에 적합합니다.

```python
response = client.predict(
    endpoint="databricks-meta-llama-3-3-70b-instruct",
    inputs={
        "prompt": "다음 Python 코드를 완성하세요:\ndef calculate_revenue(orders):\n",
        "max_tokens": 200,
        "temperature": 0.0
    }
)
```

### Embeddings (임베딩)

텍스트를 고정 차원의 벡터로 변환합니다. 텍스트의 의미를 수치화하여 유사도 검색, 분류, 클러스터링 등에 활용합니다. RAG 파이프라인에서 문서 인덱싱과 쿼리 임베딩에 필수적입니다.

```python
# 텍스트를 벡터로 변환
response = client.predict(
    endpoint="databricks-gte-large-en",
    inputs={
        "input": [
            "Delta Lake is an open-source storage layer",
            "Apache Spark is a distributed processing engine"
        ]
    }
)

# 결과: 각 텍스트의 1024차원 벡터
embeddings = [item["embedding"] for item in response["data"]]
print(f"벡터 차원: {len(embeddings[0])}")  # 1024
```

### SQL에서 사용 (ai_query)

Databricks SQL에서 `ai_query()` 함수를 사용하면, SQL 쿼리 안에서 직접 LLM을 호출할 수 있습니다. 대량의 데이터에 대해 일괄적으로 LLM 처리를 적용할 때 매우 유용합니다.

```sql
-- 고객 리뷰를 한국어로 요약
SELECT
    review_id,
    review_text,
    ai_query(
        'databricks-meta-llama-3-3-70b-instruct',
        CONCAT('다음 리뷰를 한국어로 한 문장으로 요약해 주세요: ', review_text)
    ) AS summary
FROM customer_reviews
LIMIT 100;

-- 텍스트 감성 분석
SELECT
    review_id,
    ai_query(
        'databricks-meta-llama-3-3-70b-instruct',
        CONCAT('다음 리뷰의 감성을 "긍정", "부정", "중립" 중 하나로 분류하세요: ', review_text)
    ) AS sentiment
FROM customer_reviews;

-- 임베딩 생성
SELECT
    doc_id,
    ai_query(
        'databricks-gte-large-en',
        content,
        returnType => 'ARRAY<FLOAT>'
    ) AS embedding
FROM documents;
```

---

## 과금 모델

Foundation Model API는 두 가지 과금 방식을 제공합니다. 사용 패턴에 따라 적절한 방식을 선택하면 비용을 최적화할 수 있습니다.

| 방식 | 설명 | 비용 구조 | 적합한 사용 |
|------|------|----------|-----------|
| **Pay-per-token**| 사용한 토큰 수만큼 과금합니다. 별도 프로비저닝이 필요 없습니다 | 입력 토큰 단가 + 출력 토큰 단가 | 개발, 프로토타이핑, 불규칙한 사용 |
| **Provisioned Throughput**| 예약된 처리량을 보장합니다. 시간당 고정 과금됩니다 | 시간당 DBU 고정 비용 | 프로덕션, SLA 보장 필요, 대량 처리 |

### Pay-per-token 상세

Pay-per-token은 사용한 만큼만 비용을 지불하는 방식입니다. 초기 설정이 필요 없어 바로 시작할 수 있지만, 트래픽이 많을 때 비용이 예측하기 어려울 수 있습니다.

```
예시 계산:
입력: 500 토큰 × $0.001/1K토큰 = $0.0005
출력: 200 토큰 × $0.002/1K토큰 = $0.0004
합계: $0.0009/요청

월 10만 건 요청 시: 약 $90/월
```

### Provisioned Throughput 상세

Provisioned Throughput은 일정한 처리 성능을 보장받는 방식입니다. 프로덕션 환경에서 응답 지연시간의 SLA를 보장해야 할 때, 또는 대량 배치 처리가 필요할 때 적합합니다.

```
예약: 특정 모델에 대해 토큰/초 처리량 보장
비용: 시간당 고정 DBU (예: 50 DBU/시간)
장점: 일정한 응답 시간 보장, 대량 처리 시 단가 저렴
특징: 커스텀 파인튜닝 모델도 배포 가능
```

### 과금 방식 선택 가이드

| 시나리오 | 권장 방식 | 이유 |
|---------|----------|------|
| 개발/테스트 | Pay-per-token | 초기 비용 없이 바로 시작 |
| PoC/프로토타입 | Pay-per-token | 불확실한 사용량에 유연하게 대응 |
| 프로덕션 (소량) | Pay-per-token | 월 요청 수만 건 이하라면 더 경제적 |
| 프로덕션 (대량) | Provisioned Throughput | 안정적 성능 + 대량 처리 시 단가 절감 |
| SLA 보장 필요 | Provisioned Throughput | 응답 시간 보장이 필수 |
| 파인튜닝 모델 배포 | Provisioned Throughput | 커스텀 모델은 PT로만 배포 가능 |

---

## Rate Limiting (요청 제한)

Foundation Model API에는 안정적인 서비스를 위해 Rate Limiting이 적용됩니다. 제한에 걸리면 HTTP 429 오류가 반환되므로, 애플리케이션에서 적절한 재시도 로직을 구현해야 합니다.

| 과금 방식 | 제한 방식 | 설명 |
|----------|----------|------|
| **Pay-per-token**| 분당 토큰 수 제한 | 워크스페이스 단위로 분당 처리 가능한 토큰 수가 제한됩니다 |
| **Provisioned Throughput**| 프로비저닝된 용량 | 예약한 처리량 범위 내에서 제한 없이 사용 가능합니다 |

### Rate Limiting 대응 방법

```python
import time
import mlflow.deployments

client = mlflow.deployments.get_deploy_client("databricks")

def call_with_retry(endpoint, inputs, max_retries=5):
    """지수 백오프(Exponential Backoff)로 재시도합니다."""
    for attempt in range(max_retries):
        try:
            return client.predict(endpoint=endpoint, inputs=inputs)
        except Exception as e:
            if "429" in str(e) and attempt < max_retries - 1:
                wait_time = 2 **attempt  # 1, 2, 4, 8, 16초
                print(f"Rate limited. {wait_time}초 후 재시도... ({attempt + 1}/{max_retries})")
                time.sleep(wait_time)
            else:
                raise
```

---

## Foundation Model API vs 외부 직접 호출 비교

| 비교 | Foundation Model API | 외부 API 직접 호출 |
|------|---------------------|------------------|
| **인증**| Databricks 토큰 하나로 통합 | 외부 API 키 별도 관리 필요 |
| **비용 추적**| Inference Table 자동 기록 | 수동 추적 필요 |
| **거버넌스**| MLflow Tracing + UC 통합 | 별도 구현 필요 |
| **Rate Limiting**| Databricks에서 관리 | 외부 서비스에 의존 |
| **데이터 보안**| VPC 내 처리 가능 (호스팅 모델) | 데이터가 외부로 전송 |
| **모델 전환**| 엔드포인트명만 변경 | SDK/코드 수정 필요 |
| **장애 대응**| Databricks SLA 적용 | 외부 서비스 SLA에 의존 |
| **감사/규정**| 시스템 테이블로 자동 감사 로그 | 별도 로깅 구현 필요 |

> 💡 **핵심 이점**: Foundation Model API를 통해 외부 모델을 프록시하면, 모든 LLM 호출에 대해 **비용 추적, 접근 제어, 감사 로그** 가 자동으로 적용됩니다. 직접 외부 API를 호출하면 이러한 거버넌스를 별도로 구현해야 합니다.

---

## 현업 가이드: Pay-per-token vs Provisioned Throughput 비용 역전점

현업에서 "어느 시점에 Provisioned Throughput으로 전환해야 하는가?"는 가장 많이 받는 질문 중 하나입니다. 구체적인 숫자로 설명합니다.

### 비용 역전점 분석

| 월간 요청 수 | 예상 비용 | 권장 |
|------------|---------|------|
| 1만 건 | $9 | Pay-per-token 압승 |
| 10만 건 | $90 | Pay-per-token 유리 |
| 50만 건 | $450 | 비슷한 구간 (역전점) |
| 100만 건 | $900 | Provisioned Throughput 유리 |
| 500만 건 | $4,500 | Provisioned Throughput 압승 |

> Llama 3.3 70B 기준 예시 단가: 입력 $0.001/1K 토큰, 출력 $0.002/1K 토큰, 평균 요청당 입력 500 + 출력 200 토큰

> 💡 **현업에서는 이렇게 합니다**: **월 50만 건 이상**의 요청이 예상되면 Provisioned Throughput을 검토합니다. 단, 단순 비용 외에 **응답 지연시간(Latency)** 도 고려하세요. Pay-per-token은 공유 인프라이므로 피크 시간에 지연이 길어질 수 있습니다. 실시간 챗봇처럼 P95 응답 시간이 중요한 경우, 비용이 더 들더라도 Provisioned Throughput을 선택합니다.

### 과금 방식 전환 판단 체크리스트

```
다음 중 3개 이상 해당하면 Provisioned Throughput으로 전환을 고려하세요:

☐ 월 50만 건 이상의 API 요청이 발생한다
☐ P95 응답 시간 SLA (예: 3초 이내)가 필요하다
☐ 트래픽이 예측 가능하고 일정하다 (하루 종일 균등)
☐ 커스텀 파인튜닝 모델을 서빙해야 한다
☐ Pay-per-token에서 Rate Limiting에 자주 걸린다
```

---

## 모델 선택 가이드: 어떤 상황에 어떤 모델을 쓸 것인가

현업에서 "어떤 모델을 써야 하나요?"는 기술 리더가 가장 어려워하는 질문입니다. 모델 선택은 **성능, 비용, 데이터 보안, 라이선스**4가지 축으로 판단합니다.

### 호스팅 모델 vs 외부 모델 선택 기준

| 판단 기준 | 호스팅 모델 (Llama 등) | 외부 모델 (GPT-4o, Claude 등) |
|----------|----------------------|---------------------------|
| **데이터 보안**| 데이터가 Databricks 내부에서만 처리 | 데이터가 외부 API로 전송 |
| **규제 산업**| 금융/의료/공공에서 필수 | 데이터 외부 전송 불가 시 사용 불가 |
| **성능 (복잡한 추론)**| 70B도 GPT-4o 대비 약간 낮음 | GPT-4o, Claude Sonnet이 최고 수준 |
| **비용**| 호스팅 모델이 대체로 저렴 | 외부 모델은 토큰 단가가 더 높음 |
| **한국어 성능**| Llama 3.3 70B도 한국어 준수 | GPT-4o, Claude가 한국어 최고 |
| **파인튜닝**| 가능 (Provisioned Throughput) | 제한적 (OpenAI만 일부 지원) |

### 용도별 모델 추천

| 용도 | 1순위 추천 | 2순위 추천 | 이유 |
|------|-----------|-----------|------|
| **내부 문서 요약/분류**| Llama 3.3 70B | Llama 3.1 8B | 데이터 보안 + 비용. 분류는 8B로도 충분 |
| **고객 대면 챗봇**| GPT-4o (외부) | Claude Sonnet (외부) | 한국어 자연스러움이 핵심 |
| **코드 생성/리뷰**| Claude Sonnet (외부) | GPT-4o (외부) | 코드 품질은 외부 모델이 우수 |
| **대량 배치 처리 (ai_query)**| Llama 3.1 8B | Llama 3.3 70B | 비용이 핵심. 8B로 충분한 작업이 많음 |
| **RAG 임베딩**| GTE Large EN | Qwen3-Embedding (외부) | 한국어가 중요하면 Qwen3 |
| **규제 산업 (금융/의료)**| Llama 3.3 70B | DBRX | 데이터 외부 전송 불가 |

> ⚠️ **현업에서는 이렇게 합니다**: "하나의 모델로 모든 것을 해결하자"는 접근은 비용 낭비입니다. 현업에서는 **용도별로 모델을 다르게** 사용합니다. 단순 분류는 8B 모델, 복잡한 추론은 70B, 고객 대면은 외부 모델. External Model 엔드포인트를 활용하면 코드 변경 없이 모델만 교체할 수 있으므로, A/B 테스트로 최적 모델을 찾는 것이 좋습니다.

---

## Rate Limiting 대응 실전 전략

Pay-per-token에서 Rate Limiting은 "겪어보면 아는" 고통입니다. 실전 대응 전략을 공유합니다.

### Rate Limiting이 문제가 되는 전형적인 시나리오

```
[시나리오: 대량 배치 처리]
- 100만 건의 고객 리뷰를 ai_query()로 감성 분석
- 분당 토큰 제한에 걸려서 처리가 수 시간 → 수일로 늘어남
- 중간에 실패하면 어디까지 처리했는지 추적 어려움
```

### 대량 처리를 위한 실전 패턴

```python
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

def batch_process_with_rate_limit(records, endpoint, max_workers=5, delay=0.1):
    """
    Rate Limiting을 고려한 대량 배치 처리 패턴
    - max_workers: 동시 요청 수 (너무 높으면 429 에러)
    - delay: 요청 간 대기 시간 (Rate Limit 여유)
    """
    results = []
    failed = []

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {}
        for i, record in enumerate(records):
            future = executor.submit(call_with_retry, endpoint, record)
            futures[future] = i
            time.sleep(delay)  # 요청 간격 조절

        for future in as_completed(futures):
            idx = futures[future]
            try:
                results.append((idx, future.result()))
            except Exception as e:
                failed.append((idx, str(e)))

    print(f"성공: {len(results)}, 실패: {len(failed)}")
    return results, failed
```

> 💡 **현업 팁**: 대량 배치 처리(100만 건+)에서는 `ai_query()`를 직접 사용하는 것보다, **Spark UDF로 래핑** 하고 파티션별로 병렬 처리하는 것이 효율적입니다. 각 파티션에서 적절한 Rate Limiting을 적용하면, Spark의 분산 처리 능력을 활용하면서도 429 에러를 최소화할 수 있습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Foundation Model API**| Databricks가 호스팅하는 LLM을 즉시 사용할 수 있는 서비스입니다. OpenAI 호환 API를 제공합니다 |
| **호스팅 모델**| Llama, DBRX 등 Databricks 인프라에서 직접 실행되는 모델입니다. 데이터가 외부로 전송되지 않습니다 |
| **External Model**| 외부 LLM(GPT, Claude 등)을 Databricks를 통해 프록시합니다. 거버넌스가 자동 적용됩니다 |
| **Pay-per-token**| 사용한 토큰만큼 과금. 개발/프로토타입에 적합합니다 |
| **Provisioned Throughput**| 처리량 보장. 프로덕션과 SLA가 필요한 경우에 적합합니다 |
| **ai_query()** | SQL에서 직접 LLM을 호출할 수 있는 함수입니다. 대량 배치 처리에 유용합니다 |

---

## 참고 링크

- [Databricks: Foundation Model APIs](https://docs.databricks.com/aws/en/machine-learning/foundation-models/)
- [Databricks: Pay-per-token](https://docs.databricks.com/aws/en/machine-learning/foundation-models/pay-per-token.html)
- [Databricks: Provisioned Throughput](https://docs.databricks.com/aws/en/machine-learning/foundation-models/provisioned-throughput.html)
- [Databricks: External Models](https://docs.databricks.com/aws/en/generative-ai/external-models/)
- [Databricks: AI Functions (ai_query)](https://docs.databricks.com/aws/en/large-language-models/ai-functions.html)
