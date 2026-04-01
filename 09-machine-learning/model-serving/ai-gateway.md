# AI Gateway

## AI Gateway란?

AI 애플리케이션을 프로덕션에 배포하면, **LLM 호출을 안전하고 효율적으로 관리** 하는 것이 큰 과제가 됩니다. 어떤 모델을 호출하고 있는지, 비용은 얼마나 발생하고 있는지, 부적절한 콘텐츠가 오가고 있지는 않은지 — 이 모든 것을 통제해야 합니다.

> 💡 **Databricks AI Gateway** 는 LLM 엔드포인트 앞에 위치하는 ** 관리형 프록시 계층** 으로, 트래픽 관리, 가드레일, 비용 제어, 사용량 추적을 중앙에서 처리합니다. 모든 Model Serving 엔드포인트에 자동으로 적용됩니다.

---

## AI Gateway의 핵심 기능

### 1. 통합 엔드포인트 (Unified Endpoint)

하나의 엔드포인트에서 ** 여러 LLM 제공자의 모델** 을 동일한 API 형식(OpenAI 호환)으로 호출할 수 있습니다.

```python
# 동일한 OpenAI 호환 API로 다양한 모델 호출
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DATABRICKS_TOKEN"],
    base_url=f"{os.environ['DATABRICKS_HOST']}/serving-endpoints"
)

# 엔드포인트 이름만 바꾸면 모델 전환
response = client.chat.completions.create(
    model="my-gateway-endpoint",  # AI Gateway 엔드포인트
    messages=[{"role": "user", "content": "매출 리포트를 요약해 주세요"}],
    max_tokens=500
)
```

** 지원 모델 제공자:**

| 제공자 | 모델 예시 | 연결 방식 |
|--------|---------|----------|
| **Databricks Foundation Models** | DBRX, Llama 3.1, Mixtral | 내장 (추가 설정 불필요) |
| **OpenAI** | GPT-4o, GPT-4o-mini | External Model 엔드포인트 |
| **Anthropic** | Claude 3.5 Sonnet, Claude 3 Opus | External Model 엔드포인트 |
| **Google** | Gemini 1.5 Pro | External Model 엔드포인트 |
| **Amazon Bedrock** | 여러 모델 | External Model 엔드포인트 |
| **Cohere** | Command R+ | External Model 엔드포인트 |
| ** 커스텀 모델** | 직접 학습한 모델 | MLflow Model Serving |

### External Model 엔드포인트 생성

외부 모델 제공자(OpenAI, Anthropic 등)의 API를 Databricks AI Gateway를 통해 호출하려면 **External Model 엔드포인트** 를 생성합니다.

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    EndpointCoreConfigInput,
    ServedEntityInput,
    ExternalModel,
    OpenAiConfig
)

w = WorkspaceClient()

# OpenAI GPT-4o를 AI Gateway 뒤에 배치
w.serving_endpoints.create(
    name="gpt4o-gateway",
    config=EndpointCoreConfigInput(
        served_entities=[
            ServedEntityInput(
                external_model=ExternalModel(
                    name="gpt-4o",
                    provider="openai",
                    task="llm/v1/chat",
                    openai_config=OpenAiConfig(
                        openai_api_key="{{secrets/my-scope/openai-key}}"
                    )
                )
            )
        ]
    )
)
```

> 💡 **API 키가 Secret Scope에 저장됩니다.** 애플리케이션 코드에 API 키가 노출되지 않으며, AI Gateway가 호출 시 자동으로 키를 주입합니다.

---

### 2. 가드레일 (Guardrails)

AI Gateway의 가드레일은 **LLM의 입력과 출력을 실시간으로 검사** 하여, 부적절한 콘텐츠를 차단합니다. 서버 측에서 동작하므로 클라이언트가 우회할 수 없습니다.

#### 가드레일 유형

| 유형 | 적용 대상 | 설명 |
|------|---------|------|
| **Safety Filter** | 입력 + 출력 | 유해 콘텐츠(폭력, 혐오, 성적 콘텐츠) 차단 |
| **Topic Filter** | 입력 | 허용되지 않은 주제의 질문을 차단 (예: "경쟁사 비방 글을 써줘") |
| **PII Detection** | 출력 | 개인정보(이름, 이메일, 전화번호, 주민번호)가 응답에 포함되면 마스킹 또는 차단 |
| **Keyword Filter** | 입력 + 출력 | 특정 금칙어가 포함된 요청/응답을 차단 |

#### 가드레일 설정 예시

```python
from databricks.sdk.service.serving import (
    AiGatewayConfig,
    AiGatewayGuardrails,
    AiGatewayGuardrailParameters,
    AiGatewayGuardrailPiiParameters
)

# 가드레일이 적용된 엔드포인트 설정
w.serving_endpoints.update_ai_gateway(
    name="customer-support-agent",
    guardrails=AiGatewayGuardrails(
        input=AiGatewayGuardrailParameters(
            # 입력에서 부적절한 콘텐츠 차단
            safety=True,
            # 허용되지 않은 주제 차단
            invalid_keywords=["경쟁사", "해킹", "불법"]
        ),
        output=AiGatewayGuardrailParameters(
            # 출력에서 부적절한 콘텐츠 차단
            safety=True,
            # PII 자동 마스킹
            pii=AiGatewayGuardrailPiiParameters(
                behavior="BLOCK"  # BLOCK 또는 MASK
            ),
            # 금칙어 차단
            invalid_keywords=["내부 전용", "기밀"]
        )
    )
)
```

#### 가드레일 동작 흐름

| 단계 | 구성 요소 | 설명 |
|------|----------|------|
| 1 | 사용자 요청 | 입력 |
| 2 | ** 입력 가드레일** | Safety Filter + Topic Filter + Keyword Filter (차단 시 거부) |
| 3 | **LLM 호출** | Foundation Model / External Model (Rate Limit + Fallback 적용) |
| 4 | ** 출력 가드레일** | PII Filter + Safety Filter + Custom Validator (부적절 시 필터링) |
| 5 | 응답 반환 | 로깅 (Inference Table에 전수 기록) |

> ⚠️ ** 가드레일은 100% 완벽하지 않습니다.** LLM 기반 필터링이므로 엣지 케이스가 존재할 수 있습니다. 민감한 애플리케이션에서는 가드레일과 함께 ** 추론 테이블 모니터링** 을 병행하여, 사후 검토도 수행하는 것을 권장합니다.

---

### 3. Rate Limiting (사용량 제어)

팀별, 사용자별, 엔드포인트별로 ** 분당/시간당 요청 수** 를 제한하여 비용 폭주를 방지합니다.

```python
from databricks.sdk.service.serving import (
    AiGatewayRateLimit,
    AiGatewayRateLimitRenewalPeriod,
    AiGatewayRateLimitKey
)

w.serving_endpoints.update_ai_gateway(
    name="customer-support-agent",
    rate_limits=[
        # 사용자별 분당 10회 제한
        AiGatewayRateLimit(
            calls=10,
            renewal_period=AiGatewayRateLimitRenewalPeriod.MINUTE,
            key=AiGatewayRateLimitKey.USER
        ),
        # 엔드포인트 전체 분당 100회 제한
        AiGatewayRateLimit(
            calls=100,
            renewal_period=AiGatewayRateLimitRenewalPeriod.MINUTE,
            key=AiGatewayRateLimitKey.ENDPOINT
        )
    ]
)
```

| Rate Limit Key | 설명 | 용도 |
|---------------|------|------|
| **USER** | 개별 사용자별 제한 | 특정 사용자의 과도한 사용 방지 |
| **ENDPOINT** | 엔드포인트 전체 제한 | 전체 비용 상한 |

> 💡 Rate Limit에 도달하면 HTTP 429 (Too Many Requests) 응답이 반환됩니다. 클라이언트에서 exponential backoff로 재시도하도록 구현하세요.

---

### 4. 사용량 추적 (Usage Tracking)

AI Gateway는 모든 LLM 호출을 ** 자동으로 기록** 합니다. 추론 테이블과 시스템 테이블에서 사용량, 비용, 성능을 분석할 수 있습니다.

#### 추론 테이블 (Inference Table)

```sql
-- 엔드포인트별 일일 사용량 및 비용 추적
SELECT
    DATE(timestamp) AS date,
    COUNT(*) AS total_requests,
    SUM(request:usage:prompt_tokens) AS total_input_tokens,
    SUM(request:usage:completion_tokens) AS total_output_tokens,
    SUM(request:usage:total_tokens) AS total_tokens,
    -- 대략적 비용 추정 (모델별 단가는 다를 수 있음)
    ROUND(SUM(request:usage:total_tokens) / 1000000 * 2.5, 2) AS estimated_cost_usd
FROM catalog.schema.customer_support_agent_inference_log
WHERE DATE(timestamp) >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

#### 시스템 테이블에서 서빙 메트릭 조회

```sql
-- Model Serving 엔드포인트별 비용 추적
SELECT
    usage_date,
    usage_metadata.endpoint_name,
    SUM(usage_quantity) AS total_dbus,
    ROUND(SUM(usage_quantity) * 0.07, 2) AS estimated_cost_usd
FROM system.billing.usage
WHERE billing_origin_product = 'MODEL_SERVING'
    AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY total_dbus DESC;
```

---

### 5. 모델 라우팅 및 Fallback

하나의 엔드포인트에서 **여러 모델 간 트래픽을 분배** 하거나, 주 모델 장애 시 ** 대체 모델로 자동 전환** 할 수 있습니다.

#### 트래픽 분할 (A/B 테스트)

```python
# GPT-4o 80%, Claude 20%으로 트래픽 분할
w.serving_endpoints.update_config(
    name="my-gateway",
    served_entities=[
        ServedEntityInput(
            external_model=ExternalModel(name="gpt-4o", provider="openai", ...),
            traffic_percentage=80
        ),
        ServedEntityInput(
            external_model=ExternalModel(name="claude-3-5-sonnet", provider="anthropic", ...),
            traffic_percentage=20
        )
    ]
)
```

#### Fallback 시나리오

| 상황 | AI Gateway 동작 |
|------|----------------|
| 주 모델(GPT-4o) 응답 지연 | 설정된 타임아웃 후 대체 모델로 전환 |
| 주 모델 API 에러 (5xx) | 자동으로 대체 모델에 요청 |
| Rate Limit 초과 | 429 응답 반환 (클라이언트 측 재시도) |

---

### 6. Payload 로깅 및 감사

모든 요청/응답의 전체 페이로드를 ** 추론 테이블** 에 자동 기록합니다. 이 데이터는 다음 용도로 활용됩니다:

| 활용 | 설명 |
|------|------|
| ** 품질 모니터링** | 응답의 품질을 사후 평가합니다 (Correctness, Groundedness) |
| ** 비용 분석** | 토큰 사용량 기반 정확한 비용을 산출합니다 |
| ** 규제 감사** | 누가, 언제, 어떤 질문을 했고, 어떤 응답을 받았는지 전체 이력을 보관합니다 |
| ** 평가 데이터셋 구축** | 실제 사용자 질문을 수집하여 에이전트 평가에 활용합니다 |
| ** 파인튜닝 데이터** | 고품질 Q&A 쌍을 수집하여 모델 개선에 활용합니다 |

```python
# 추론 테이블 자동 캡처 설정
w.serving_endpoints.update_config(
    name="my-gateway",
    auto_capture_config=AutoCaptureConfigInput(
        catalog_name="ml_catalog",
        schema_name="serving_logs",
        enabled=True
    )
)
```

---

## 실전 구성 예시

### 고객 지원 챗봇의 AI Gateway 설정

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 1. 엔드포인트 생성 (Foundation Model 사용)
endpoint = w.serving_endpoints.create(
    name="cs-chatbot-prod",
    config=EndpointCoreConfigInput(
        served_entities=[
            ServedEntityInput(
                entity_name="databricks-meta-llama-3-1-70b-instruct",
                workload_size="Small",
                scale_to_zero_enabled=False  # 프로덕션은 콜드 스타트 방지
            )
        ],
        auto_capture_config=AutoCaptureConfigInput(
            catalog_name="prod_catalog",
            schema_name="serving",
            enabled=True
        )
    )
)

# 2. 가드레일 설정
w.serving_endpoints.update_ai_gateway(
    name="cs-chatbot-prod",
    guardrails=AiGatewayGuardrails(
        input=AiGatewayGuardrailParameters(
            safety=True,
            invalid_keywords=["해킹", "불법", "경쟁사"]
        ),
        output=AiGatewayGuardrailParameters(
            safety=True,
            pii=AiGatewayGuardrailPiiParameters(behavior="MASK")
        )
    ),
    rate_limits=[
        AiGatewayRateLimit(
            calls=20,
            renewal_period=AiGatewayRateLimitRenewalPeriod.MINUTE,
            key=AiGatewayRateLimitKey.USER
        )
    ]
)
```

---

## AI Gateway 도입 체크리스트

| # | 항목 | 설명 |
|---|------|------|
| 1 | **External Model 엔드포인트 생성** | 외부 모델(OpenAI, Anthropic) 사용 시 API 키를 Secret Scope에 저장 |
| 2 | ** 가드레일 설정** | Safety, PII, Topic, Keyword 필터 중 필요한 것 활성화 |
| 3 | **Rate Limiting 설정** | 사용자별/엔드포인트별 요청 수 제한 |
| 4 | ** 추론 테이블 활성화** | 모든 요청/응답을 자동 기록하여 모니터링 |
| 5 | **Unity Catalog 권한 설정** | 엔드포인트에 대한 CAN_QUERY 권한을 필요한 그룹에만 부여 |
| 6 | ** 비용 모니터링 대시보드** | 시스템 테이블로 일별/팀별 비용 추적 |
| 7 | **Fallback 모델 구성** | 주 모델 장애 시 대체 모델 설정 (선택) |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **AI Gateway** | LLM 엔드포인트 앞의 관리형 프록시로, 트래픽 관리·가드레일·비용 제어를 중앙 처리합니다 |
| ** 클라이언트 측 vs 서버 측** | LiteLLM 등 클라이언트 측 구현은 키 분산·거버넌스 부재·우회 가능 등의 위험이 있습니다. AI Gateway는 서버 측에서 동작하여 이를 방지합니다 |
| ** 가드레일** | Safety, PII, Topic, Keyword 4가지 필터로 입출력을 실시간 검사합니다 |
| **Rate Limiting** | 사용자별/엔드포인트별 요청 수를 제한하여 비용 폭주를 방지합니다 |
| ** 추론 테이블** | 모든 요청/응답을 자동 기록하여 모니터링·감사·평가에 활용합니다 |
| ** 모델 라우팅** | 트래픽 분할(A/B 테스트)과 Fallback을 지원합니다 |

---

## 부록: 클라이언트 측 LLM 관리의 한계

LiteLLM, LangChain Router 등 클라이언트 측 라이브러리로 LLM 호출을 관리하는 팀이 있습니다. 소규모 프로토타입에서는 유용하지만, 프로덕션 환경에서는 다음과 같은 한계가 있습니다.

| 문제 | 설명 |
|------|------|
| **API 키 분산** | 각 앱이 OpenAI/Anthropic API 키를 직접 보유합니다. 유출 시 무제한 과금 위험이 있습니다 |
| ** 거버넌스 부재** | 누가, 언제, 어떤 모델을 호출했는지 중앙 추적이 불가합니다 |
| ** 비용 통제 불가** | 팀별 사용량 제한이 없어, 한 앱의 폭주가 전체 예산을 소진할 수 있습니다 |
| ** 가드레일 우회** | 클라이언트 측 필터는 코드 수정으로 쉽게 우회할 수 있습니다 |
| ** 버전 파편화** | 각 팀이 다른 버전의 라이브러리를 사용하여 동작이 일관되지 않습니다 |
| ** 모니터링 사각지대** | 로깅이 분산되어 전체 사용 패턴을 파악할 수 없습니다 |

### AI Gateway vs 클라이언트 측 비교

| 항목 | 클라이언트 측 (LiteLLM 등) | Databricks AI Gateway |
|------|--------------------------|----------------------|
| **API 키 관리** | 각 앱이 직접 보유 (분산) | Databricks가 중앙 관리 |
| ** 가드레일** | 코드 레벨 (우회 가능) | 플랫폼 레벨 (우회 불가) |
| ** 비용 제어** | 자체 구현 필요 | Rate Limiting 내장 |
| ** 모니터링** | 자체 로깅 구축 | 추론 테이블 자동 기록 |
| ** 모델 전환** | 코드 수정 + 재배포 | 설정 변경만으로 전환 |
| **Unity Catalog** | ❌ | ✅ 권한, 감사, 리니지 |
| ** 운영 부담** | 높음 | 없음 (관리형) |

---

## 참고 링크

- [Databricks: AI Gateway](https://docs.databricks.com/aws/en/ai-gateway/)
- [Databricks: AI Gateway Guardrails](https://docs.databricks.com/aws/en/ai-gateway/guardrails.html)
- [Databricks: AI Gateway Rate Limits](https://docs.databricks.com/aws/en/ai-gateway/rate-limits.html)
- [Databricks: External Models](https://docs.databricks.com/aws/en/generative-ai/external-models/)
- [Databricks: Inference Tables](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
