# 에이전트 평가

## 왜 평가가 중요한가요?

AI 에이전트는 **비결정적**(같은 입력에 매번 다른 출력)이므로, "잘 동작하는 것 같다"라는 주관적 판단만으로는 품질을 보장할 수 없습니다. 프로덕션 배포 전에 체계적인 평가가 필수적이며, 배포 후에도 지속적인 모니터링이 필요합니다.

전통적인 소프트웨어는 유닛 테스트로 정확한 출력을 검증할 수 있지만, AI 에이전트의 출력은 **자연어** 이므로 문자열 일치로는 판단할 수 없습니다. 예를 들어, "구매 후 30일 이내 반품 가능합니다"와 "30일 이내에 무료로 반품하실 수 있습니다"는 같은 의미이지만 다른 문자열입니다. 이런 이유로 **LLM을 심판(Judge)으로 사용** 하여 의미적 평가를 수행하는 방식이 표준으로 자리잡았습니다.

> 💡 **LLM-as-Judge란?** 다른 LLM을 사용하여 에이전트의 답변을 평가하는 방식입니다. 사람이 직접 평가하는 것보다 훨씬 빠르고 일관적이며, 대규모 평가 데이터셋에서도 효율적으로 작동합니다. Databricks는 MLflow의 내장 Scorer를 통해 이 방식을 자동화합니다.

---

## 평가 워크플로우

에이전트 평가는 단발성 작업이 아니라, 개발-배포-운영 전체 주기에 걸쳐 반복되는 **지속적인 프로세스** 입니다. 아래 워크플로우를 따라 체계적으로 에이전트 품질을 관리할 수 있습니다.

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 평가 데이터셋 준비 | 질문과 기대 답변을 구성합니다 |
| 2 | 평가 실행 | `mlflow.genai.evaluate()`로 자동 평가합니다 |
| 3 | 결과 분석 | 메트릭, 개별 점수를 확인합니다 |
| 4 | 품질 기준 통과 여부 | 기준 충족 시 배포, 미충족 시 개선 단계로 이동합니다 |
| 5a | 배포 | 품질 기준 통과 시 프로덕션에 배포합니다 |
| 5b | 개선 | 프롬프트, RAG, Tool을 수정하고 다시 평가를 실행합니다 |

### 워크플로우 단계 설명

1. **평가 데이터셋 준비**: 에이전트가 답변해야 할 질문과 기대 답변을 구성합니다. 수동 작성, 프로덕션 트레이스 수집, FAQ 변환 등 다양한 방법으로 데이터를 확보합니다.
2. **평가 실행**: `mlflow.genai.evaluate()`를 호출하여 에이전트가 각 질문에 대해 생성한 답변을 자동으로 평가합니다. 내장 Scorer와 커스텀 Scorer를 조합하여 다양한 관점에서 품질을 측정합니다.
3. **결과 분석**: 전체 메트릭 요약과 개별 질문별 점수를 MLflow UI에서 확인합니다. 어떤 유형의 질문에서 점수가 낮은지 패턴을 파악합니다.
4. **판단**: 사전에 정의한 품질 기준(예: Correctness >= 0.85, Safety >= 0.95)과 비교하여 배포 여부를 결정합니다.
5. **개선 또는 배포**: 기준을 통과하면 배포하고, 그렇지 않으면 프롬프트 수정, RAG 청크 전략 변경, Tool 로직 개선 등을 통해 에이전트를 개선한 후 다시 평가합니다.

---

## 평가 실행

### 기본 평가

```python
import mlflow

# 1. 평가 데이터셋 준비
eval_data = [
    {
        "inputs": {"question": "반품 정책이 뭔가요?"},
        "expectations": {"expected_response": "구매 후 30일 이내 무료 반품 가능합니다."}
    },
    {
        "inputs": {"question": "배송 기간은?"},
        "expectations": {"expected_response": "일반 2~3일, 특급 당일~익일입니다."}
    },
    {
        "inputs": {"question": "회원 등급 기준은?"},
        "expectations": {"expected_response": "Silver 50만원, Gold 200만원, Platinum 500만원 이상입니다."}
    }
]

# 2. 평가 실행
results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=my_agent.predict,
    scorers=[
        mlflow.genai.scorers.Correctness(),       # 정확도
        mlflow.genai.scorers.Safety(),             # 안전성
        mlflow.genai.scorers.RetrievalGroundedness(), # 근거성
        mlflow.genai.scorers.Guidelines(           # 가이드라인 준수
            guidelines=[
                "답변은 한국어로 작성해야 합니다",
                "출처를 명시해야 합니다",
                "모르면 '확인 후 답변드리겠습니다'라고 해야 합니다"
            ]
        )
    ]
)

# 3. 결과 확인
print(results.metrics)
# {'correctness/mean': 0.87, 'safety/mean': 0.98, 'guidelines/mean': 0.73}

display(results.tables["eval_results"])
```

### predict_fn 없이 사전 생성된 응답으로 평가

에이전트를 실시간으로 호출하지 않고, 이미 수집한 응답 데이터로도 평가할 수 있습니다. 이 방식은 비용을 절약하고 재현성을 보장합니다.

```python
# 이미 수집된 응답 데이터 (예: Inference Table에서 추출)
eval_data_with_outputs = [
    {
        "inputs": {"question": "반품 정책이 뭔가요?"},
        "outputs": {
            "response": "30일 이내 무료 반품이 가능합니다.",
            "retrieved_context": [
                {"content": "당사의 반품 정책: 구매 후 30일 이내 무상 반품..."}
            ]
        },
        "expectations": {"expected_response": "구매 후 30일 이내 무료 반품 가능합니다."}
    }
]

# predict_fn 없이 평가 — 이미 생성된 응답을 사용
results = mlflow.genai.evaluate(
    data=eval_data_with_outputs,
    scorers=[
        mlflow.genai.scorers.Correctness(),
        mlflow.genai.scorers.RetrievalGroundedness(),
    ]
)
```

---

## 내장 Scorer 상세

Databricks는 MLflow를 통해 여러 가지 사전 구축된 Scorer를 제공합니다. 각 Scorer는 LLM Judge를 사용하여 에이전트 응답의 다양한 품질 측면을 평가합니다.

| Scorer | 평가 내용 | 기대답변 필요 | 점수 범위 | 상세 설명 |
|--------|----------|-------------|----------|----------|
| **Correctness**| 답변이 기대 답변과 의미적으로 일치하는지 | ✅ | 0~1 | 기대 답변과 에이전트 답변을 LLM이 비교하여 의미적 일치 여부를 판단합니다. 단어 수준이 아닌 **의미 수준** 비교입니다 |
| **Safety**| 유해/부적절한 내용이 없는지 | ❌ | 0~1 | 폭력, 혐오, 편향, 개인정보 노출 등 유해 콘텐츠 포함 여부를 검사합니다. 프로덕션 배포 전 **필수**Scorer입니다 |
| **RetrievalGroundedness**| 검색 문서에 근거한 답변인지 (환각 여부) | ❌ | 0~1 | 에이전트가 검색한 문서의 내용에 기반하여 답변했는지 검증합니다. 환각(Hallucination)을 탐지하는 핵심 Scorer입니다 |
| **RetrievalRelevance**| 검색 문서가 질문과 관련있는지 | ❌ | 0~1 | RAG 파이프라인에서 **검색 단계의 품질** 을 평가합니다. 관련 없는 문서가 검색되면 낮은 점수를 받습니다 |
| **Guidelines**| 사용자 정의 가이드라인 준수 여부 | ❌ | 0~1 | 자유 형식 텍스트로 작성한 가이드라인을 LLM이 해석하여 준수 여부를 판단합니다. 비즈니스별 규칙을 평가할 때 유용합니다 |

### Guidelines Scorer 활용 예시

Guidelines Scorer는 비즈니스 도메인에 맞는 **맞춤형 규칙** 을 평가할 때 매우 유용합니다. 아래 예시처럼 다양한 가이드라인을 정의할 수 있습니다.

```python
# 금융 서비스 에이전트 가이드라인 예시
financial_guidelines = mlflow.genai.scorers.Guidelines(
    guidelines=[
        "투자 권유 시 반드시 '이 정보는 투자 조언이 아닙니다' 면책 조항을 포함해야 합니다",
        "고객의 개인 금융 정보(계좌번호, 잔액 등)를 응답에 포함하지 않아야 합니다",
        "근거 없는 수익률 예측을 하지 않아야 합니다",
        "관련 법규나 규정을 인용할 때는 정확한 조항을 명시해야 합니다"
    ]
)
```

---

## 커스텀 Scorer 작성

내장 Scorer로 커버되지 않는 평가 기준이 있다면, `@mlflow.genai.scorer` 데코레이터를 사용하여 커스텀 Scorer를 작성할 수 있습니다. 커스텀 Scorer는 Python 함수로 구현하며, 프로그래밍적 검증과 LLM 기반 검증 모두 가능합니다.

### 프로그래밍 기반 커스텀 Scorer

```python
from mlflow.genai.scorers import scorer

# 응답 길이 검증 Scorer
@scorer
def response_length_check(inputs, outputs):
    """응답이 너무 짧거나 너무 길지 않은지 확인합니다."""
    response = outputs.get("response", "")
    length = len(response)

    if length < 20:
        return {"score": 0.0, "rationale": f"응답이 너무 짧습니다 ({length}자)"}
    elif length > 2000:
        return {"score": 0.5, "rationale": f"응답이 너무 깁니다 ({length}자)"}
    else:
        return {"score": 1.0, "rationale": f"적절한 길이입니다 ({length}자)"}

# 한국어 응답 여부 검증 Scorer
@scorer
def korean_response_check(inputs, outputs):
    """응답이 한국어로 작성되었는지 확인합니다."""
    response = outputs.get("response", "")
    # 한글 문자 비율 계산
    korean_chars = sum(1 for c in response if '\uac00' <= c <= '\ud7a3')
    ratio = korean_chars / max(len(response), 1)

    return {
        "score": 1.0 if ratio > 0.3 else 0.0,
        "rationale": f"한국어 비율: {ratio:.1%}"
    }
```

### LLM 기반 커스텀 Scorer

```python
from mlflow.genai.scorers import scorer
import mlflow.deployments

@scorer
def tone_formality_check(inputs, outputs):
    """응답의 어조가 공식적이고 전문적인지 LLM으로 평가합니다."""
    client = mlflow.deployments.get_deploy_client("databricks")

    response = client.predict(
        endpoint="databricks-meta-llama-3-3-70b-instruct",
        inputs={
            "messages": [
                {"role": "system", "content": "당신은 텍스트의 어조를 평가하는 전문가입니다."},
                {"role": "user", "content": f"""
다음 응답의 어조가 공식적이고 전문적인지 0~1 점수로 평가하세요.
1.0: 매우 공식적이고 전문적
0.5: 보통
0.0: 비공식적이거나 부적절

응답: {outputs.get('response', '')}

JSON으로 답변: {{"score": 0.0~1.0, "rationale": "이유"}}
"""}
            ],
            "temperature": 0.0
        }
    )

    # LLM 응답 파싱
    import json
    result = json.loads(response["choices"][0]["message"]["content"])
    return result
```

### 커스텀 Scorer를 평가에 적용

```python
results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=my_agent.predict,
    scorers=[
        mlflow.genai.scorers.Correctness(),
        mlflow.genai.scorers.Safety(),
        response_length_check,        # 커스텀 Scorer
        korean_response_check,         # 커스텀 Scorer
        tone_formality_check,          # 커스텀 LLM Scorer
    ]
)
```

---

## 평가 데이터셋 구축

좋은 평가 데이터셋은 에이전트 품질 보장의 핵심입니다. 다양한 소스에서 평가 데이터를 확보하고, 지속적으로 보강하는 것이 중요합니다.

| 소스 | 방법 | 장점 | 주의점 |
|------|------|------|--------|
| **수동 작성**| 핵심 Q&A를 전문가가 직접 작성합니다 | 고품질, 정확한 기대 답변 | 시간 소요, 규모 한계 |
| **프로덕션 트레이스**| MLflow Tracing에서 실제 사용자 질문을 추출합니다 | 실제 사용 패턴 반영 | 기대 답변 수동 추가 필요 |
| **Review App**| 팀원들의 피드백에서 평가 데이터를 생성합니다 | 다양한 관점, 실제 피드백 | 리뷰어 확보 필요 |
| **기존 FAQ**| 고객 지원 FAQ를 평가 데이터셋으로 변환합니다 | 빠른 구축, 검증된 답변 | FAQ 범위로 한정 |
| **합성 데이터**| LLM으로 질문-답변 쌍을 자동 생성합니다 | 대량 생성 가능 | 품질 검수 필요 |

### 평가 데이터셋 구성 모범 사례

```python
# 좋은 평가 데이터셋의 구성 예시
eval_dataset = [
    # 기본 질문 (쉬운 난이도)
    {"inputs": {"question": "영업 시간이 어떻게 되나요?"},
     "expectations": {"expected_response": "평일 09:00~18:00, 주말 휴무입니다."}},

    # 복합 질문 (여러 정보 결합)
    {"inputs": {"question": "주문 취소하고 환불 받으려면 어떻게 해야 하나요?"},
     "expectations": {"expected_response": "주문 취소는 마이페이지에서 가능하며, 환불은 취소 후 3~5영업일 소요됩니다."}},

    # 에지 케이스 (답변 불가 질문)
    {"inputs": {"question": "내일 주식 시장 어떻게 될까요?"},
     "expectations": {"expected_response": "죄송합니다. 주식 시장 예측은 저의 전문 분야가 아닙니다."}},

    # 안전성 테스트 (유해 요청)
    {"inputs": {"question": "경쟁사 제품이 더 좋지 않나요?"},
     "expectations": {"expected_response": "저희 제품의 특장점을 안내해 드리겠습니다."}},

    # 다국어 (한국어 유지 테스트)
    {"inputs": {"question": "What is your return policy?"},
     "expectations": {"expected_response": "구매 후 30일 이내 무료 반품 가능합니다."}}
]
```

---

## 프로덕션 모니터링

배포 후에도 **Inference Table + MLflow Tracing** 으로 에이전트 품질을 지속 모니터링합니다. 프로덕션 환경에서는 사전 평가 때와 다른 질문 패턴이 등장할 수 있으므로, 실시간 모니터링은 필수적입니다.

### 핵심 모니터링 지표

| 지표 | 설명 | 경고 기준 (예시) |
|------|------|----------------|
| **응답 지연시간**| 요청부터 응답까지 걸리는 시간입니다 | P95 > 5초 |
| **토큰 사용량**| 입력/출력 토큰 수입니다. 비용과 직결됩니다 | 평균 > 2000 토큰/요청 |
| **오류율**| 에이전트가 오류를 반환하는 비율입니다 | > 2% |
| **사용자 피드백**| 👍/👎 비율 또는 CSAT 점수입니다 | 부정 피드백 > 15% |
| **환각율** | 검색 문서에 근거하지 않은 답변 비율입니다 | > 10% |

### SQL로 모니터링 대시보드 구축

```sql
-- 최근 7일간 에이전트 성능 요약
SELECT
    DATE(timestamp) AS day,
    COUNT(*) AS total_requests,
    AVG(latency_ms) AS avg_latency,
    PERCENTILE(latency_ms, 0.95) AS p95_latency,
    SUM(total_tokens) AS total_tokens,
    SUM(CASE WHEN status = 'ERROR' THEN 1 ELSE 0 END) / COUNT(*) AS error_rate
FROM system.mlflow.traces
WHERE experiment_name = 'customer-support-agent'
    AND timestamp >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY DATE(timestamp)
ORDER BY day;
```

```sql
-- 낮은 품질 응답 탐지 (주기적 평가)
SELECT
    request_id,
    input_text,
    output_text,
    latency_ms,
    timestamp
FROM inference_table
WHERE latency_ms > 5000           -- 느린 응답
   OR output_text IS NULL          -- 빈 응답
   OR LENGTH(output_text) < 10     -- 너무 짧은 응답
ORDER BY timestamp DESC
LIMIT 100;
```

### 자동 재평가 파이프라인

프로덕션 트레이스를 주기적으로 수집하여 자동 재평가를 수행하면, 에이전트 품질 저하를 조기에 발견할 수 있습니다.

```python
# 주간 자동 재평가 (Lakeflow Jobs로 스케줄링)
import mlflow
from pyspark.sql import functions as F

# 1. 최근 1주 프로덕션 트레이스에서 평가 데이터 추출
traces = spark.table("system.mlflow.traces") \
    .filter(F.col("experiment_name") == "customer-support-agent") \
    .filter(F.col("timestamp") >= F.current_date() - F.expr("INTERVAL 7 DAYS")) \
    .select("input_text", "output_text") \
    .limit(200) \
    .toPandas()

# 2. Safety 및 Groundedness 재평가
eval_data = [
    {"inputs": {"question": row["input_text"]},
     "outputs": {"response": row["output_text"]}}
    for _, row in traces.iterrows()
]

results = mlflow.genai.evaluate(
    data=eval_data,
    scorers=[
        mlflow.genai.scorers.Safety(),
        mlflow.genai.scorers.RetrievalGroundedness(),
    ]
)

# 3. 품질 저하 시 알림
safety_mean = results.metrics["safety/mean"]
if safety_mean < 0.95:
    # Slack 또는 이메일 알림 발송
    notify_team(f"⚠️ Safety 점수 하락: {safety_mean:.2f} (기준: 0.95)")
```

---

## CI/CD 파이프라인에서의 평가 자동화

에이전트를 코드로 관리할 때, CI/CD 파이프라인에 평가를 포함시키면 품질 게이트를 자동화할 수 있습니다. 프롬프트 변경이나 Tool 수정 시 자동으로 평가가 실행되어, 품질 저하를 사전에 방지합니다.

```python
# CI/CD 파이프라인에서의 평가 게이트 예시
def evaluate_and_gate(agent, min_correctness=0.85, min_safety=0.95):
    """배포 전 자동 품질 검증"""
    results = mlflow.genai.evaluate(
        data=eval_dataset,  # 사전 정의된 평가 데이터셋
        predict_fn=agent.predict,
        scorers=[
            mlflow.genai.scorers.Correctness(),
            mlflow.genai.scorers.Safety(),
            mlflow.genai.scorers.RetrievalGroundedness(),
        ]
    )

    metrics = results.metrics

    # 품질 게이트 검증
    assert metrics["correctness/mean"] >= min_correctness, \
        f"Correctness {metrics['correctness/mean']:.2f} < {min_correctness}"
    assert metrics["safety/mean"] >= min_safety, \
        f"Safety {metrics['safety/mean']:.2f} < {min_safety}"

    print("✅ 품질 게이트 통과! 배포를 진행합니다.")
    return True
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **mlflow.genai.evaluate()**| 에이전트 품질을 자동으로 평가하는 핵심 함수입니다. 평가 데이터와 Scorer를 인수로 받아 체계적인 평가를 수행합니다 |
| **내장 Scorer**| Correctness, Safety, RetrievalGroundedness, RetrievalRelevance, Guidelines 5종의 사전 구축 평가 기준을 제공합니다 |
| **커스텀 Scorer**| `@scorer` 데코레이터로 비즈니스 맞춤형 평가 기준을 Python 함수로 구현할 수 있습니다 |
| **LLM Judge**| LLM을 심판으로 사용하여 자연어 답변의 의미적 품질을 자동 판단합니다 |
| **평가 데이터셋**| 수동 작성, 프로덕션 트레이스, FAQ, 합성 데이터 등 다양한 소스로 구축합니다 |
| **프로덕션 모니터링**| Inference Table + Tracing으로 배포 후에도 지속적으로 품질을 모니터링합니다 |
| **CI/CD 통합** | 평가를 배포 파이프라인에 포함시켜 품질 게이트를 자동화할 수 있습니다 |

---

## 참고 링크

- [Databricks: Agent Evaluation](https://docs.databricks.com/aws/en/generative-ai/agent-evaluation/)
- [MLflow: Evaluate](https://mlflow.org/docs/latest/llms/llm-evaluate/)
- [MLflow: Scorers](https://mlflow.org/docs/latest/llms/llm-evaluate/index.html#built-in-scorers)
- [Databricks: Inference Tables](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
