# Model Serving 개요

## Model Serving이란?

> 💡 **Model Serving**은 학습된 ML 모델이나 AI 에이전트를 **실시간 추론 REST API 엔드포인트**로 배포하는 Databricks의 서버리스 서비스입니다. 애플리케이션에서 HTTP 요청을 보내면, 밀리초~초 단위로 예측 결과를 반환합니다.

---

## 엔드포인트 유형

Databricks Model Serving은 세 가지 유형의 엔드포인트를 제공합니다.

| 유형 | 설명 | 적합한 사용 |
|------|------|-----------|
| **Foundation Model API** | Databricks가 호스팅하는 LLM (Llama, DBRX, Claude 등) | LLM 활용, GenAI 앱, 빠른 프로토타이핑 |
| **External Model Endpoint** | OpenAI, Anthropic, Google 등 **외부 LLM API를 프록시**하여 통합 관리 | 외부 모델을 거버넌스 하에 관리, 비용 추적, Rate Limiting |
| **Custom Model Endpoint** | MLflow로 등록한 **사용자 커스텀 모델**(scikit-learn, PyTorch, ChatAgent 등) 배포 | 자체 ML 모델, AI 에이전트 배포 |

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **애플리케이션** | 클라이언트 | REST API로 Model Serving에 요청합니다 |
| **Foundation Model API** | 기본 LLM | Llama, DBRX, Claude 등을 제공합니다 |
| **External Model** | 외부 LLM 프록시 | OpenAI, Anthropic 등 외부 모델을 프록시합니다 |
| **Custom Model** | 사용자 모델 | MLflow 모델, Agent 등을 배포합니다 |

---

## Foundation Model API 상세

### Pay-per-token vs Provisioned Throughput

| 과금 모델 | 설명 | 적합한 사용 |
|-----------|------|-----------|
| **Pay-per-token** | 사용한 토큰 수만큼 과금. 별도 프로비저닝 불필요 | 개발, 프로토타이핑, 불규칙한 사용 |
| **Provisioned Throughput** | 예약된 처리량을 보장. 시간당 고정 과금 | 프로덕션, 안정적 처리량 필요, 대량 처리 |

### 사용 가능한 모델

| 제공사 | 모델 | 용도 |
|--------|------|------|
| **Meta** | Llama 3.3 70B Instruct | 범용 대화, 코드 생성 |
| **Databricks** | DBRX Instruct | 범용 대화 |
| **Databricks** | GTE Large / BGE Large | 텍스트 임베딩 |
| **OpenAI** | GPT-5.x 시리즈 | 고성능 대화, 코드, 분석 |
| **Anthropic** | Claude 4.x 시리즈 | 고성능 대화, 분석, 코딩 |
| **Google** | Gemini 3.x 시리즈 | 멀티모달, 대화 |

```python
# Pay-per-token 사용 예시
import mlflow.deployments

client = mlflow.deployments.get_deploy_client("databricks")

response = client.predict(
    endpoint="databricks-meta-llama-3-3-70b-instruct",
    inputs={
        "messages": [
            {"role": "system", "content": "당신은 데이터 전문가입니다. 한국어로 답변해 주세요."},
            {"role": "user", "content": "Delta Lake의 ACID 트랜잭션이 왜 중요한가요?"}
        ],
        "max_tokens": 500,
        "temperature": 0.1
    }
)
print(response["choices"][0]["message"]["content"])
```

---

## External Model Endpoint

외부 LLM API를 Databricks Model Serving을 통해 **프록시**합니다. 직접 외부 API를 호출하는 것 대비 다음 장점이 있습니다.

| 장점 | 설명 |
|------|------|
| **통합 인증** | Databricks 토큰으로 인증. 외부 API 키를 클라이언트에 노출하지 않습니다 |
| **비용 추적** | Inference Table로 모든 호출의 토큰 사용량을 추적합니다 |
| **Rate Limiting** | 팀별/사용자별 호출량을 제한할 수 있습니다 |
| **모델 전환** | 프록시 뒤의 모델을 변경해도 클라이언트 코드는 변경 불필요합니다 |
| **거버넌스** | MLflow Tracing으로 모든 호출을 추적합니다 |

```python
# External Model 엔드포인트 생성 (OpenAI 프록시)
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    EndpointCoreConfigInput, ServedEntityInput, ExternalModel, OpenAiConfig
)

w = WorkspaceClient()

w.serving_endpoints.create(
    name="openai-gpt4-proxy",
    config=EndpointCoreConfigInput(
        served_entities=[ServedEntityInput(
            name="gpt4",
            external_model=ExternalModel(
                name="gpt-4",
                provider="openai",
                openai_config=OpenAiConfig(
                    openai_api_key="{{secrets/scope/openai-key}}"
                ),
                task="llm/v1/chat"
            )
        )]
    )
)
```

---

## Custom Model Endpoint

MLflow로 로깅한 모델을 배포합니다. 전통적 ML 모델(scikit-learn, XGBoost)부터 AI 에이전트(ChatAgent)까지 배포할 수 있습니다.

### 엔드포인트 생성

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    EndpointCoreConfigInput, ServedEntityInput, AutoCaptureConfigInput
)

w = WorkspaceClient()

w.serving_endpoints.create(
    name="fraud-detection-v2",
    config=EndpointCoreConfigInput(
        served_entities=[ServedEntityInput(
            entity_name="catalog.schema.fraud_model",  # UC 등록 모델
            entity_version="3",
            workload_size="Small",                     # Small, Medium, Large
            scale_to_zero_enabled=True,                # 유휴 시 0으로 스케일다운
            workload_type="CPU"                        # CPU 또는 GPU
        )],
        auto_capture_config=AutoCaptureConfigInput(    # Inference Table 자동 활성화
            enabled=True,
            catalog_name="catalog",
            schema_name="schema",
            table_name_prefix="fraud_model"
        )
    )
)
```

### Workload Size

| 사이즈 | 사양 | 적합한 모델 |
|--------|------|-----------|
| **Small** | 4 vCPU, 16 GB RAM | 경량 모델 (scikit-learn, XGBoost) |
| **Medium** | 8 vCPU, 32 GB RAM | 중간 모델, AI 에이전트 |
| **Large** | 16 vCPU, 64 GB RAM | 대형 모델 |
| **GPU (Small~Large)** | T4/A10/A100 GPU | 딥러닝, LLM 파인튜닝 모델 |

### 엔드포인트 호출

```python
import requests

# REST API로 호출
url = f"https://{workspace_url}/serving-endpoints/fraud-detection-v2/invocations"
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

# 단건 예측
response = requests.post(url, json={
    "dataframe_records": [
        {"amount": 50000, "merchant_category": "online", "hour": 3, "is_foreign": True}
    ]
}, headers=headers)

print(response.json())  # {"predictions": [0.87]}  (사기 확률 87%)
```

```python
# Databricks SDK로 호출
w = WorkspaceClient()
response = w.serving_endpoints.query(
    name="fraud-detection-v2",
    dataframe_records=[
        {"amount": 50000, "merchant_category": "online", "hour": 3}
    ]
)
```

---

## A/B 테스트 (Traffic Split)

하나의 엔드포인트에 **여러 모델 버전**을 배포하고, 트래픽 비율을 조절하여 A/B 테스트를 수행할 수 있습니다.

```python
w.serving_endpoints.update_config(
    name="fraud-detection-v2",
    served_entities=[
        ServedEntityInput(
            name="champion",
            entity_name="catalog.schema.fraud_model",
            entity_version="3",
            workload_size="Small",
            scale_to_zero_enabled=True
        ),
        ServedEntityInput(
            name="challenger",
            entity_name="catalog.schema.fraud_model",
            entity_version="4",
            workload_size="Small",
            scale_to_zero_enabled=True
        )
    ],
    traffic_config={
        "routes": [
            {"served_model_name": "champion", "traffic_percentage": 80},
            {"served_model_name": "challenger", "traffic_percentage": 20}
        ]
    }
)
```

---

## Inference Tables (추론 로깅)

> 💡 **Inference Table**은 엔드포인트의 모든 요청/응답을 **자동으로 Delta 테이블에 기록**하는 기능입니다.

| 기록 항목 | 설명 |
|-----------|------|
| 요청 입력 | 모델에 전달된 입력 데이터 |
| 응답 출력 | 모델의 예측 결과 |
| 타임스탬프 | 요청/응답 시간 |
| 지연 시간 | 추론에 걸린 시간 (밀리초) |
| 상태 코드 | 성공/실패 여부 |
| 모델 버전 | 어떤 모델 버전이 응답했는지 |

```sql
-- Inference Table에서 모니터링 쿼리
SELECT
    DATE(timestamp) AS day,
    COUNT(*) AS total_requests,
    AVG(latency_ms) AS avg_latency,
    SUM(CASE WHEN status_code != 200 THEN 1 ELSE 0 END) AS error_count
FROM catalog.schema.fraud_model_inference_log
GROUP BY DATE(timestamp)
ORDER BY day DESC;
```

---

## 정리

| 엔드포인트 유형 | 용도 | 과금 |
|----------------|------|------|
| **Foundation Model API** | Databricks 호스팅 LLM 사용 | Pay-per-token 또는 Provisioned |
| **External Model** | 외부 LLM (OpenAI 등) 프록시 | 외부 API 비용 + DBU |
| **Custom Model** | 자체 ML 모델/에이전트 배포 | 엔드포인트 실행 시간 기준 |

---

## 참고 링크

- [Databricks: Model Serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/)
- [Databricks: Foundation Model APIs](https://docs.databricks.com/aws/en/machine-learning/foundation-models/)
- [Databricks: External models](https://docs.databricks.com/aws/en/machine-learning/model-serving/external-models.html)
- [Databricks: Inference tables](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
