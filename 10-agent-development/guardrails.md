# AI 가드레일(Guardrails) 상세

## 가드레일이란?

**가드레일(Guardrails)** 은 AI 에이전트의 입력과 출력에 대한 **안전 장치**입니다. 에이전트가 유해한 콘텐츠를 생성하거나, 개인정보를 유출하거나, 허용되지 않은 주제에 대해 답변하는 것을 방지합니다. Databricks의 **AI Gateway**를 통해 Model Serving 엔드포인트에 가드레일을 적용할 수 있습니다.

> 💡 **비유**: 가드레일은 도로의 "안전 가드레일"과 같습니다. 차량(AI)이 도로(허용 범위) 밖으로 나가지 않도록 물리적 장벽을 세우는 것입니다.

---

## AI Gateway 가드레일 개념

AI Gateway는 Model Serving 엔드포인트 앞에 위치하는 **프록시 계층**으로, 모든 요청과 응답을 모니터링하고 제어합니다.

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | 사용자 | 요청을 AI Gateway로 전송합니다 |
| 2 | AI Gateway | 모든 요청/응답을 모니터링하고 제어하는 프록시 계층입니다 |
| 3 | 입력 필터 | 사용자 요청을 검사합니다 (Safety, Topic, PII) |
| 4 | Model Serving Endpoint | 에이전트가 요청을 처리합니다 |
| 5 | 출력 필터 | 에이전트 응답을 검사합니다 (Safety, PII, 키워드) |
| 6 | 사용자 | 필터링된 응답을 수신합니다 |

---

## 입력 필터 (Input Guardrails)

사용자의 요청이 에이전트에 도달하기 전에 필터링합니다.

### Safety Filter (안전 필터)

유해하거나 부적절한 콘텐츠를 자동으로 감지하고 차단합니다.

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    AiGatewayConfig,
    AiGatewayGuardrails,
    AiGatewayGuardrailParameters
)

w = WorkspaceClient()

w.serving_endpoints.put_ai_gateway(
    name="my-agent-endpoint",
    guardrails=AiGatewayGuardrails(
        input=AiGatewayGuardrailParameters(
            safety=True  # 안전 필터 활성화
        )
    )
)
```

Safety 필터가 감지하는 항목:

| 카테고리 | 설명 |
|---------|------|
| **혐오 발언** | 인종, 성별, 종교 등에 대한 차별적 발언입니다 |
| **폭력/자해** | 폭력적 행위나 자해를 조장하는 내용입니다 |
| **성적 콘텐츠** | 부적절한 성적 콘텐츠입니다 |
| **불법 활동** | 불법 행위를 조장하는 내용입니다 |

### Topic Filter (주제 필터)

에이전트가 답변해야 할 주제와 답변하지 말아야 할 주제를 명시합니다.

```python
w.serving_endpoints.put_ai_gateway(
    name="customer-support-agent",
    guardrails=AiGatewayGuardrails(
        input=AiGatewayGuardrailParameters(
            safety=True,
            valid_topics=[
                "customer support",
                "order inquiry",
                "product information",
                "shipping status",
                "return and refund"
            ],
            invalid_topics=[
                "politics",
                "religion",
                "competitor products",
                "investment advice",
                "medical advice"
            ]
        )
    )
)
```

| 설정 | 동작 |
|------|------|
| `valid_topics` | 이 목록에 해당하는 주제의 질문만 허용합니다 |
| `invalid_topics` | 이 목록에 해당하는 주제의 질문을 차단합니다 |

> 💡 **valid_topics와 invalid_topics**: 둘 다 설정할 수 있으며, invalid_topics에 해당하면 valid_topics에 포함되더라도 차단됩니다.

---

## 출력 필터 (Output Guardrails)

에이전트의 응답이 사용자에게 전달되기 전에 필터링합니다.

### PII 필터 (개인정보 보호)

응답에 포함된 개인식별정보(PII)를 감지하고 처리합니다.

```python
w.serving_endpoints.put_ai_gateway(
    name="customer-support-agent",
    guardrails=AiGatewayGuardrails(
        output=AiGatewayGuardrailParameters(
            safety=True,
            pii={
                "behavior": "BLOCK"  # PII 감지 시 응답을 차단합니다
            }
        )
    )
)
```

| PII behavior | 동작 |
|-------------|------|
| `BLOCK` | PII가 감지되면 전체 응답을 차단합니다 |
| `MASK` | PII를 마스킹하여 전달합니다 (예: `010-****-5678`) |
| `NONE` | PII 감지를 비활성화합니다 |

감지되는 PII 유형:

| PII 유형 | 예시 |
|---------|------|
| **이름** | 홍길동 |
| **전화번호** | 010-1234-5678 |
| **이메일** | user@example.com |
| **주소** | 서울시 강남구 ... |
| **주민등록번호** | 901015-1234567 |
| **신용카드 번호** | 4532-XXXX-XXXX-1234 |

### 키워드 필터

특정 키워드가 포함된 응답을 차단합니다.

```python
w.serving_endpoints.put_ai_gateway(
    name="customer-support-agent",
    guardrails=AiGatewayGuardrails(
        output=AiGatewayGuardrailParameters(
            safety=True,
            pii={"behavior": "MASK"},
            invalid_keywords=[
                "경쟁사_이름",
                "내부기밀",
                "비공개"
            ]
        )
    )
)
```

---

## Rate Limiting (속도 제한)

에이전트의 남용을 방지하기 위해 요청 수를 제한합니다.

```python
from databricks.sdk.service.serving import AiGatewayRateLimit

w.serving_endpoints.put_ai_gateway(
    name="customer-support-agent",
    guardrails=AiGatewayGuardrails(
        input=AiGatewayGuardrailParameters(safety=True),
        output=AiGatewayGuardrailParameters(safety=True, pii={"behavior": "BLOCK"})
    ),
    rate_limits=[
        # 사용자별 분당 20회 제한
        AiGatewayRateLimit(
            calls=20,
            renewal_period="minute",
            key="user"
        ),
        # 전체 분당 1000회 제한
        AiGatewayRateLimit(
            calls=1000,
            renewal_period="minute",
            key="endpoint"
        )
    ]
)
```

### Rate Limit 설정 옵션

| 설정 | 설명 |
|------|------|
| `calls` | 허용되는 최대 요청 수입니다 |
| `renewal_period` | 제한 기간입니다 (`minute`, `hour`, `day`) |
| `key` | 제한 단위입니다 (`user`: 사용자별, `endpoint`: 전체) |

---

## 전체 설정 예제

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    AiGatewayConfig,
    AiGatewayGuardrails,
    AiGatewayGuardrailParameters,
    AiGatewayRateLimit,
    AiGatewayUsageTrackingConfig
)

w = WorkspaceClient()

# 종합 가드레일 설정
w.serving_endpoints.put_ai_gateway(
    name="customer-support-agent",

    # 가드레일 설정
    guardrails=AiGatewayGuardrails(
        # 입력 필터
        input=AiGatewayGuardrailParameters(
            safety=True,
            valid_topics=["customer support", "order inquiry", "product info"],
            invalid_topics=["politics", "religion", "medical advice"]
        ),
        # 출력 필터
        output=AiGatewayGuardrailParameters(
            safety=True,
            pii={"behavior": "BLOCK"},
            invalid_keywords=["경쟁사A", "경쟁사B"]
        )
    ),

    # 속도 제한
    rate_limits=[
        AiGatewayRateLimit(calls=30, renewal_period="minute", key="user"),
        AiGatewayRateLimit(calls=2000, renewal_period="minute", key="endpoint")
    ],

    # 사용량 추적
    usage_tracking_config=AiGatewayUsageTrackingConfig(enabled=True)
)
```

---

## 가드레일 차단 시 동작

가드레일에 의해 요청이 차단되면, 사용자에게 안전한 기본 응답이 반환됩니다.

| 차단 유형 | 반환 응답 예시 |
|----------|-------------|
| **Safety 차단** | "죄송합니다, 해당 요청은 처리할 수 없습니다." |
| **Topic 차단** | "죄송합니다, 해당 주제에 대해서는 답변을 드리기 어렵습니다." |
| **PII 차단** | "응답에 민감한 개인정보가 포함되어 전달할 수 없습니다." |
| **Rate Limit** | HTTP 429 (Too Many Requests) |

---

## 모니터링

가드레일의 활동은 Inference Tables에 자동으로 기록됩니다.

```sql
-- 가드레일 차단 이벤트 분석
SELECT
    DATE(timestamp) AS date,
    COUNT(*) AS total_requests,
    COUNT(CASE WHEN guardrail_action = 'BLOCKED' THEN 1 END) AS blocked_count,
    ROUND(
        COUNT(CASE WHEN guardrail_action = 'BLOCKED' THEN 1 END) * 100.0 / COUNT(*), 2
    ) AS block_rate_pct
FROM catalog.schema.agent_logs_payload
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **AI Gateway** | Model Serving 앞에 위치하여 요청/응답을 필터링하는 프록시입니다 |
| **Safety Filter** | 유해하고 부적절한 콘텐츠를 자동으로 감지하고 차단합니다 |
| **Topic Filter** | 허용/금지 주제를 설정하여 에이전트의 응답 범위를 제한합니다 |
| **PII Filter** | 개인정보를 감지하여 차단(BLOCK) 또는 마스킹(MASK)합니다 |
| **Rate Limiting** | 사용자별/엔드포인트별 요청 수를 제한하여 남용을 방지합니다 |

---

## 참고 링크

- [Databricks: AI Gateway guardrails](https://docs.databricks.com/aws/en/ai-gateway/guardrails.html)
- [Databricks: AI Gateway configuration](https://docs.databricks.com/aws/en/ai-gateway/)
- [Databricks: Rate limiting](https://docs.databricks.com/aws/en/ai-gateway/rate-limits.html)
- [Azure Databricks: AI Gateway](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/guardrails)
