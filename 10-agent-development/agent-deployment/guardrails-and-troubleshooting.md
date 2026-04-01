# Guardrails와 트러블슈팅

## Guardrails 설정

에이전트의 안전한 운영을 위해 입출력에 대한 제한을 설정할 수 있습니다.

### AI Gateway Guardrails

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

# Guardrails가 포함된 엔드포인트 설정
w.serving_endpoints.put_ai_gateway(
    name="customer-support-agent",
    guardrails=AiGatewayGuardrails(
        input=AiGatewayGuardrailParameters(
            safety=True,       # 유해 콘텐츠 필터링
            pii={"behavior": "BLOCK"},  # PII 정보 차단
            valid_topics=["customer support", "order inquiry", "product info"],
            invalid_topics=["politics", "religion", "competitor products"]
        ),
        output=AiGatewayGuardrailParameters(
            safety=True,
            pii={"behavior": "BLOCK"}
        )
    ),
    rate_limits=[
        AiGatewayRateLimit(
            calls=100,
            renewal_period="minute",
            key="user"  # 사용자별 분당 100회 제한
        )
    ],
    usage_tracking_config=AiGatewayUsageTrackingConfig(enabled=True)
)
```

### Guardrails 유형

| 유형 | 설명 |
|------|------|
| **Safety Filter**| 유해하거나 부적절한 콘텐츠를 자동으로 필터링합니다 |
| **PII Detection**| 개인정보(이름, 전화번호, 주민번호 등)를 감지하고 차단합니다 |
| **Topic Control**| 허용된 주제만 응답하고, 금지된 주제는 거부합니다 |
| **Rate Limiting**| 사용자별/전체 요청 수를 제한하여 남용을 방지합니다 |

---

## 트러블슈팅 가이드

### 자주 발생하는 문제

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| 엔드포인트가 시작되지 않음 | 의존성 누락 또는 버전 충돌 | `pip_requirements`에 모든 패키지와 정확한 버전을 명시합니다 |
| `ModuleNotFoundError` | 패키지가 설치되지 않음 | `pip_requirements`에 해당 패키지를 추가합니다 |
| 환경 변수 접근 실패 | 환경 변수가 설정되지 않음 | 엔드포인트 설정에서 `environment_vars`를 확인합니다 |
| Vector Search 권한 오류 | 서비스 프린시펄 권한 부족 | `resources`에 VS Index를 명시하거나 수동으로 권한을 부여합니다 |
| Scale to Zero 후 지연 | 콜드 스타트 시간 | 프로덕션에서는 `scale_to_zero_enabled=False` 권장 |
| 응답 시간 초과 | 에이전트 로직이 너무 복잡 | 도구 호출 수 제한, LLM 응답 스트리밍 활성화 |

### 엔드포인트 상태 확인

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 엔드포인트 상태 조회
endpoint = w.serving_endpoints.get("customer-support-agent")
print(f"State: {endpoint.state.ready}")
print(f"Config update: {endpoint.state.config_update}")

# 엔드포인트 로그 조회
logs = w.serving_endpoints.logs(
    name="customer-support-agent",
    served_model_name="customer_support_agent-3"
)
print(logs.logs)
```

### 배포 전 로컬 테스트

배포 전에 로깅된 모델을 로컬에서 먼저 테스트하는 것을 권장합니다.

```python
import mlflow

# 로깅된 모델 로드 및 테스트
model = mlflow.pyfunc.load_model(model_info.model_uri)

# 테스트 입력
test_input = {
    "messages": [
        {"role": "user", "content": "주문번호 12345의 배송 상태를 알려주세요."}
    ]
}

# 예측 실행
response = model.predict(test_input)
print(response)
```

---

## 실습: 에이전트 배포 전체 워크플로

전체 배포 과정을 순서대로 정리하면 다음과 같습니다.

```python
# ============================================
# 1. 에이전트 코드 작성 (agent.py)
# ============================================
# (위의 agent.py 참고)

# ============================================
# 2. MLflow로 모델 로깅 + UC 등록
# ============================================
import mlflow
from mlflow.models.resources import (
    DatabricksServingEndpoint,
    DatabricksVectorSearchIndex
)

with mlflow.start_run():
    model_info = mlflow.pyfunc.log_model(
        artifact_path="agent",
        python_model="agent.py",
        registered_model_name="catalog.schema.customer_support_agent",
        pip_requirements=["langchain>=0.3.0", "databricks-sdk>=0.30.0"],
        resources=[
            DatabricksServingEndpoint(endpoint_name="databricks-meta-llama-3-3-70b-instruct"),
            DatabricksVectorSearchIndex(index_name="catalog.schema.docs_index"),
        ],
        input_example={"messages": [{"role": "user", "content": "안녕하세요"}]}
    )

# ============================================
# 3. 로컬 테스트
# ============================================
model = mlflow.pyfunc.load_model(model_info.model_uri)
result = model.predict({"messages": [{"role": "user", "content": "테스트"}]})
print(f"로컬 테스트 결과: {result}")

# ============================================
# 4. 배포 (엔드포인트 + Review App)
# ============================================
from databricks import agents

deployment = agents.deploy(
    model_name="catalog.schema.customer_support_agent",
    model_version=model_info.registered_model_version
)
print(f"배포 완료!")
print(f"   Endpoint: {deployment.endpoint_name}")
print(f"   Review App: {deployment.review_app_url}")

# ============================================
# 5. Review App 설정
# ============================================
agents.set_permissions(
    model_name="catalog.schema.customer_support_agent",
    users=["tester@company.com"],
    permission="CAN_QUERY"
)

# ============================================
# 6. 프로덕션 엔드포인트 호출 테스트
# ============================================
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
response = w.serving_endpoints.query(
    name=deployment.endpoint_name,
    input={
        "messages": [
            {"role": "user", "content": "최근 주문 상태를 확인하고 싶습니다."}
        ]
    }
)
print(f"에이전트 응답: {response.choices[0].message.content}")
```

---

## 정리

| 핵심 포인트 | 설명 |
|------------|------|
| ** 배포 파이프라인**| 개발 → MLflow 로깅 → UC 등록 → Model Serving → 모니터링 |
| **agents.deploy()**| 엔드포인트, Review App, Inference Tables를 한 번에 설정하는 가장 간편한 방법입니다 |
| **set_model()**| 에이전트 코드를 별도 파일로 분리하고, 진입점을 지정하는 권장 패턴입니다 |
| **Review App**| 배포 전 테스터 피드백을 수집하고, 평가 데이터셋으로 활용합니다 |
| **Inference Tables**| 모든 요청/응답을 자동 기록하여 모니터링과 디버깅에 활용합니다 |
| **Guardrails**| Safety, PII, Topic Control, Rate Limiting으로 안전한 운영을 보장합니다 |
| ** 버전 관리** | UC 앨리어스(champion/challenger)로 무중단 업데이트와 롤백이 가능합니다 |

---

## 참고 링크

- [Databricks: Deploy agents](https://docs.databricks.com/aws/en/generative-ai/deploy-agent.html)
- [Databricks: Agent Framework](https://docs.databricks.com/aws/en/generative-ai/agent-framework/index.html)
- [Databricks: Review App](https://docs.databricks.com/aws/en/generative-ai/agent-evaluation/review-app.html)
- [Databricks: MLflow Tracing](https://docs.databricks.com/aws/en/mlflow/mlflow-tracing.html)
- [Databricks: AI Gateway Guardrails](https://docs.databricks.com/aws/en/ai-gateway/guardrails.html)
- [Databricks: Inference Tables](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
