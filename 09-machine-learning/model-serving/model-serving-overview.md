# Model Serving 개요

## Model Serving이란?

> 💡 **Model Serving**은 학습된 ML 모델이나 AI 에이전트를 **실시간 추론 REST API 엔드포인트** 로 배포하는 Databricks의 서버리스 서비스입니다. 애플리케이션에서 HTTP 요청을 보내면, 밀리초~초 단위로 예측 결과를 반환합니다.

---

## 엔드포인트 유형

Databricks Model Serving은 세 가지 유형의 엔드포인트를 제공합니다.

| 유형 | 설명 | 적합한 사용 |
|------|------|-----------|
| **Foundation Model API**| Databricks가 호스팅하는 LLM (Llama, DBRX, Claude 등) | LLM 활용, GenAI 앱, 빠른 프로토타이핑 |
| **External Model Endpoint**| OpenAI, Anthropic, Google 등 **외부 LLM API를 프록시** 하여 통합 관리 | 외부 모델을 거버넌스 하에 관리, 비용 추적, Rate Limiting |
| **Custom Model Endpoint**| MLflow로 등록한 **사용자 커스텀 모델**(scikit-learn, PyTorch, ChatAgent 등) 배포 | 자체 ML 모델, AI 에이전트 배포 |

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **애플리케이션**| 클라이언트 | REST API로 Model Serving에 요청합니다 |
| **Foundation Model API**| 기본 LLM | Llama, DBRX, Claude 등을 제공합니다 |
| **External Model**| 외부 LLM 프록시 | OpenAI, Anthropic 등 외부 모델을 프록시합니다 |
| **Custom Model**| 사용자 모델 | MLflow 모델, Agent 등을 배포합니다 |

---

## Foundation Model API 상세

### Pay-per-token vs Provisioned Throughput

| 과금 모델 | 설명 | 적합한 사용 |
|-----------|------|-----------|
| **Pay-per-token**| 사용한 토큰 수만큼 과금. 별도 프로비저닝 불필요 | 개발, 프로토타이핑, 불규칙한 사용 |
| **Provisioned Throughput**| 예약된 처리량을 보장. 시간당 고정 과금 | 프로덕션, 안정적 처리량 필요, 대량 처리 |

### 사용 가능한 모델

| 제공사 | 모델 | 용도 |
|--------|------|------|
| **Meta**| Llama 3.3 70B Instruct | 범용 대화, 코드 생성 |
| **Databricks**| DBRX Instruct | 범용 대화 |
| **Databricks**| GTE Large / BGE Large | 텍스트 임베딩 |
| **OpenAI**| GPT-5.x 시리즈 | 고성능 대화, 코드, 분석 |
| **Anthropic**| Claude 4.x 시리즈 | 고성능 대화, 분석, 코딩 |
| **Google**| Gemini 3.x 시리즈 | 멀티모달, 대화 |

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

외부 LLM API를 Databricks Model Serving을 통해 **프록시** 합니다. 직접 외부 API를 호출하는 것 대비 다음 장점이 있습니다.

| 장점 | 설명 |
|------|------|
| **통합 인증**| Databricks 토큰으로 인증. 외부 API 키를 클라이언트에 노출하지 않습니다 |
| **비용 추적**| Inference Table로 모든 호출의 토큰 사용량을 추적합니다 |
| **Rate Limiting**| 팀별/사용자별 호출량을 제한할 수 있습니다 |
| **모델 전환**| 프록시 뒤의 모델을 변경해도 클라이언트 코드는 변경 불필요합니다 |
| **거버넌스**| MLflow Tracing으로 모든 호출을 추적합니다 |

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
| **Small**| 4 vCPU, 16 GB RAM | 경량 모델 (scikit-learn, XGBoost) |
| **Medium**| 8 vCPU, 32 GB RAM | 중간 모델, AI 에이전트 |
| **Large**| 16 vCPU, 64 GB RAM | 대형 모델 |
| **GPU (Small~Large)**| T4/A10/A100 GPU | 딥러닝, LLM 파인튜닝 모델 |

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

하나의 엔드포인트에 **여러 모델 버전** 을 배포하고, 트래픽 비율을 조절하여 A/B 테스트를 수행할 수 있습니다.

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

> 💡 **Inference Table**은 엔드포인트의 모든 요청/응답을 **자동으로 Delta 테이블에 기록** 하는 기능입니다.

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

## 심화: 프로덕션 Model Serving 운영

이 섹션에서는 프로덕션 환경에서 Model Serving을 운영할 때 알아야 할 지연시간 특성, 비용 최적화, 배포 전략, 캐싱, 멀티 리전 패턴을 다룹니다.

### 지연시간 특성 (Latency Profile)

#### Foundation Model API

| 메트릭 | Pay-per-token | Provisioned Throughput |
|--------|---------------|----------------------|
| **p50 지연시간**| 200~500ms (짧은 응답) | 100~300ms |
| **p99 지연시간**| 1~5초 | 500ms~2초 |
| **TTFT (Time to First Token)**| 100~300ms | 50~150ms |
| **처리량**| 공유 풀, 변동 가능 | 예약 처리량 보장 |
| **콜드스타트**| 없음 (항상 가동) | 없음 (항상 가동) |

> 💡 **TTFT(Time to First Token)** 란 요청 후 첫 번째 토큰이 생성되기까지의 시간입니다. Streaming 응답에서 사용자 체감 속도에 직접 영향을 줍니다.

#### Custom Model Endpoint

| 상태 | 지연시간 | 설명 |
|------|---------|------|
| **Warm (Scale > 0)**| p50: 10~100ms, p99: 200~500ms | 인스턴스가 가동 중 |
| **Cold Start (Scale-to-Zero)**| **30초~3분**| 컨테이너 시작 + 모델 로딩 + 헬스체크 |
| **GPU Cold Start**| **1~5분**| GPU 할당 + CUDA 초기화 + 대형 모델 로딩 |

> ⚠️ **Gotcha — Scale-to-Zero 콜드스타트**: Scale-to-Zero를 활성화하면 유휴 시 비용을 절약할 수 있지만, 첫 요청 시 30초~수 분의 콜드스타트가 발생합니다. **프로덕션 실시간 서빙에서는 Scale-to-Zero를 비활성화** 하는 것이 일반적입니다. 개발/스테이징 환경에서만 활성화하세요.

> ⚠️ **Gotcha — 모델 크기와 콜드스타트**: 모델 아티팩트가 클수록(예: PyTorch 모델 5GB+) 콜드스타트 시간이 길어집니다. MLflow 로깅 시 모델을 최적화(ONNX 변환, 양자화)하면 로딩 시간을 단축할 수 있습니다.

---

### 비용 최적화 전략

#### Pay-per-token vs Provisioned Throughput 손익분기점

```
월간 비용 비교 (Llama 3.3 70B 기준, 예시):

Pay-per-token:
  - 입력: $0.24 / 1M tokens
  - 출력: $0.24 / 1M tokens
  - 월 1억 토큰 사용 시: ~$24/월

Provisioned Throughput:
  - 1 단위: ~$6~8/시간 (모델, 리전에 따라 다름)
  - 24/7 운영: ~$4,300~5,800/월
  - 손익분기점: 대략 월 1.5~2억 토큰 이상 사용 시 Provisioned가 유리
```

| 사용 패턴 | 권장 과금 모델 | 이유 |
|-----------|--------------|------|
| 프로토타이핑, 불규칙 사용 | Pay-per-token | 사용한 만큼만 과금 |
| 일 1,000건 이하 요청 | Pay-per-token | 고정 비용 대비 사용량 적음 |
| 일 10,000건+ 안정적 트래픽 | Provisioned Throughput | 예측 가능한 비용, 안정적 지연시간 |
| 배치 추론 (대량 처리) | Pay-per-token + Batch Inference | 서빙 엔드포인트 대신 배치 추론 사용 |

#### Custom Model 비용 최적화

```python
# 비용 최적화된 엔드포인트 설정
w.serving_endpoints.create(
    name="fraud-detection-prod",
    config=EndpointCoreConfigInput(
        served_entities=[ServedEntityInput(
            entity_name="catalog.schema.fraud_model",
            entity_version="3",
            workload_size="Small",          # 필요한 최소 사이즈 선택
            scale_to_zero_enabled=False,    # 프로덕션: 콜드스타트 방지
            workload_type="CPU",            # GPU가 불필요하면 CPU 사용
            min_provisioned_throughput=0,
            max_provisioned_throughput=1000  # 최대 처리량 제한으로 비용 상한 설정
        )]
    )
)
```

| 최적화 전략 | 절감 효과 | 트레이드오프 |
|------------|----------|------------|
| **Scale-to-Zero**(dev/staging) | 유휴 시 100% 절감 | 콜드스타트 30초~3분 |
| **CPU 사용**(전통 ML) | GPU 대비 70~90% 절감 | 딥러닝 모델은 추론 속도 저하 |
| **Small 사이즈**| Large 대비 ~75% 절감 | 동시 요청 처리량 감소 |
| **배치 추론 활용**| 서빙 대비 50~80% 절감 | 실시간 응답 불가 |

> ⚠️ **Gotcha — GPU 선택**: Custom Model에 GPU를 선택하면 비용이 10배 이상 증가합니다. scikit-learn, XGBoost, LightGBM 등 전통 ML 모델은 CPU로 충분합니다. GPU는 PyTorch/TensorFlow 딥러닝 모델이나 Fine-tuned LLM에만 사용하세요.

---

### 프로덕션 배포 패턴

#### Canary 배포 단계별 가이드

Canary 배포는 새 모델 버전을 **소량의 트래픽으로 먼저 검증** 한 후, 점진적으로 트래픽을 늘리는 전략입니다.

```python
# Step 1: Canary 시작 — 새 버전에 5% 트래픽 할당
w.serving_endpoints.update_config(
    name="fraud-detection-prod",
    served_entities=[
        ServedEntityInput(name="v3-champion", entity_name="catalog.schema.fraud_model",
                          entity_version="3", workload_size="Small"),
        ServedEntityInput(name="v4-canary", entity_name="catalog.schema.fraud_model",
                          entity_version="4", workload_size="Small"),
    ],
    traffic_config={"routes": [
        {"served_model_name": "v3-champion", "traffic_percentage": 95},
        {"served_model_name": "v4-canary", "traffic_percentage": 5},
    ]}
)

# Step 2: 24시간 모니터링 후 문제 없으면 → 25%로 증가
# Step 3: 48시간 모니터링 후 문제 없으면 → 50%로 증가
# Step 4: 안정 확인 후 → 100%로 전환 (이전 버전 제거)
```

#### 모니터링 기준 (Canary 판단)

| 메트릭 | 정상 범위 | 롤백 기준 |
|--------|----------|----------|
| **p99 지연시간**| 기존 버전 대비 ±20% | 50% 이상 증가 |
| **에러율**| < 0.1% | 0.5% 초과 |
| **모델 정확도**| 기존 버전 대비 ±2% | 5% 이상 하락 |
| **메모리 사용량**| 안정적 (증가 추세 없음) | 지속적 증가 (메모리 누수) |

#### 롤백 전략

```python
# 즉시 롤백: 이전 버전으로 100% 트래픽 전환
w.serving_endpoints.update_config(
    name="fraud-detection-prod",
    served_entities=[
        ServedEntityInput(name="v3-champion", entity_name="catalog.schema.fraud_model",
                          entity_version="3", workload_size="Small"),
    ],
    traffic_config={"routes": [
        {"served_model_name": "v3-champion", "traffic_percentage": 100},
    ]}
)
# 롤백 소요시간: 트래픽 설정 변경은 ~30초 이내 반영
```

> ⚠️ **Gotcha — 블루/그린 배포**: 블루/그린 방식으로 완전히 새로운 엔드포인트를 생성하여 전환하는 것도 가능하지만, **엔드포인트 URL이 변경** 되므로 클라이언트 측 수정이 필요합니다. Traffic Split 방식의 Canary 배포가 더 실용적입니다.

---

### 추론 캐싱과 Prompt Caching

#### AI Gateway Prompt Caching

Foundation Model API와 External Model Endpoint에서 **동일한 프롬프트에 대한 응답을 캐싱** 하여 비용과 지연시간을 줄일 수 있습니다.

| 캐싱 유형 | 설명 | 효과 |
|-----------|------|------|
| **Exact Match Cache**| 완전히 동일한 요청에 대해 캐시된 응답 반환 | 지연시간 ~10ms, 토큰 비용 0 |
| **KV Cache (Provisioned)**| 긴 시스템 프롬프트의 KV Cache를 재사용 | TTFT 50~80% 감소 |

```python
# 캐싱 활성화 (AI Gateway 설정)
# temperature=0으로 설정하면 캐시 적중률이 높아집니다
response = client.predict(
    endpoint="databricks-meta-llama-3-3-70b-instruct",
    inputs={
        "messages": [
            {"role": "system", "content": "...긴 시스템 프롬프트..."},
            {"role": "user", "content": "매출 요약해줘"}
        ],
        "temperature": 0  # 결정적 응답 → 캐시 적중률 ↑
    }
)
```

> ⚠️ **Gotcha**: `temperature > 0`이면 동일 입력에도 다른 출력이 나오므로 Exact Match Cache의 효과가 제한적입니다. 캐싱을 적극 활용하려면 `temperature=0`을 사용하세요.

---

### 멀티 리전 서빙

대규모 글로벌 서비스에서는 지연시간 최적화와 장애 대응을 위해 멀티 리전 아키텍처를 고려해야 합니다.

#### 패턴 1: DNS 기반 지역 라우팅

```
사용자 (한국) → Azure Korea Central Workspace → 해당 리전 엔드포인트
사용자 (미국) → AWS us-east-1 Workspace → 해당 리전 엔드포인트
```

- 각 리전에 동일한 모델을 배포한 엔드포인트를 운영합니다
- DNS 또는 API Gateway(예: AWS API Gateway, Azure Front Door)로 가까운 리전으로 라우팅합니다
- MLflow Model Registry에서 동일 모델 버전을 관리하여 일관성을 보장합니다

#### 패턴 2: Failover 구성

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

ENDPOINTS = [
    "https://primary-workspace.cloud.databricks.com/serving-endpoints/model/invocations",
    "https://secondary-workspace.cloud.databricks.com/serving-endpoints/model/invocations",
]

def predict_with_failover(payload, headers):
    for endpoint in ENDPOINTS:
        try:
            response = requests.post(endpoint, json=payload, headers=headers, timeout=5)
            if response.status_code == 200:
                return response.json()
        except (requests.Timeout, requests.ConnectionError):
            continue  # 다음 엔드포인트로 Failover
    raise Exception("All endpoints failed")
```

| 고려 사항 | 설명 |
|-----------|------|
| **모델 동기화**| 양쪽 리전에 동일 버전의 모델이 배포되어야 합니다 |
| **데이터 일관성**| Feature Store 데이터가 리전 간 동기화되어야 합니다 |
| **비용**| 두 리전 모두 엔드포인트를 유지하므로 비용이 2배 |
| **Inference Table**| 각 리전별로 별도 Inference Table이 생성됩니다 |

> ⚠️ **Gotcha**: Databricks의 Model Serving은 **단일 워크스페이스 내에서만** 서빙됩니다. 멀티 리전 구성을 위해서는 각 리전에 별도 워크스페이스와 엔드포인트를 구성해야 하며, 모델 배포 파이프라인도 멀티 리전을 지원하도록 설계해야 합니다.

---

## 정리

| 엔드포인트 유형 | 용도 | 과금 |
|----------------|------|------|
| **Foundation Model API**| Databricks 호스팅 LLM 사용 | Pay-per-token 또는 Provisioned |
| **External Model**| 외부 LLM (OpenAI 등) 프록시 | 외부 API 비용 + DBU |
| **Custom Model** | 자체 ML 모델/에이전트 배포 | 엔드포인트 실행 시간 기준 |

---

## 참고 링크

- [Databricks: Model Serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/)
- [Databricks: Foundation Model APIs](https://docs.databricks.com/aws/en/machine-learning/foundation-models/)
- [Databricks: External models](https://docs.databricks.com/aws/en/machine-learning/model-serving/external-models.html)
- [Databricks: Inference tables](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables.html)
