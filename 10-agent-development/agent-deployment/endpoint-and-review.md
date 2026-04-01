# 엔드포인트 생성과 Review App

## Step 3: 엔드포인트 생성 및 배포

### deploy() 함수 사용 (권장)

`databricks.agents` 패키지의 `deploy()` 함수가 가장 간편한 배포 방법입니다. 엔드포인트 생성, Review App 활성화, Inference Tables 설정을 한 번에 처리합니다.

```python
from databricks import agents

# 에이전트 배포 (엔드포인트 + Review App + Inference Tables 자동 설정)
deployment = agents.deploy(
    model_name="catalog.schema.customer_support_agent",
    model_version=3,  # 또는 특정 버전 번호
    environment_vars={
        "VECTOR_SEARCH_ENDPOINT": "vs-endpoint-name",
        "TEMPERATURE": "0.1"
    }
)

print(f"Endpoint name: {deployment.endpoint_name}")
print(f"Query endpoint: {deployment.query_endpoint}")
print(f"Review App URL: {deployment.review_app_url}")
```

### SDK를 사용한 세부 설정

더 세밀한 제어가 필요한 경우 Databricks SDK를 직접 사용합니다.

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    EndpointCoreConfigInput,
    ServedEntityInput,
    AutoCaptureConfigInput
)

w = WorkspaceClient()

# 엔드포인트 생성 (세부 설정 포함)
endpoint = w.serving_endpoints.create(
    name="customer-support-agent",
    config=EndpointCoreConfigInput(
        served_entities=[
            ServedEntityInput(
                entity_name="catalog.schema.customer_support_agent",
                entity_version="3",
                workload_size="Small",       # Small, Medium, Large
                scale_to_zero_enabled=True,  # 비활성 시 0으로 축소
                environment_vars={
                    "VECTOR_SEARCH_ENDPOINT": {"type": "plain", "value": "vs-endpoint"},
                    "API_KEY": {"type": "secret", "key": "scope/key"}  # 시크릿 참조
                }
            )
        ],
        auto_capture_config=AutoCaptureConfigInput(
            enabled=True,
            catalog_name="catalog",
            schema_name="schema",
            table_name_prefix="agent_logs"
        )
    )
)
```

### 엔드포인트 설정 옵션

| 설정 | 옵션 | 설명 |
|------|------|------|
| **워크로드 크기**| Small / Medium / Large | 동시 요청 처리 용량을 결정합니다 |
| **Scale to Zero**| true / false | 트래픽이 없을 때 인스턴스를 0으로 축소하여 비용을 절감합니다 |
| **환경 변수**| plain / secret | 민감 정보는 `secret` 타입으로 안전하게 전달합니다 |
| **GPU**| 엔터티별 설정 | GPU가 필요한 모델(임베딩 등)에 GPU 인스턴스를 할당합니다 |
| **Inference Tables**| auto_capture_config | 모든 요청/응답을 Delta Table에 자동 기록합니다 |

---

## Step 4: Review App으로 테스트

### Review App이란?

Review App은 배포된 에이전트를 **웹 기반 채팅 UI**로 테스트하고, 테스터들이 **피드백을 남길 수 있는** 도구입니다. 수집된 피드백은 에이전트 평가와 개선에 활용됩니다.

### 테스터 추가 및 피드백 수집

```python
from databricks import agents

# Review App에 테스터 추가
agents.set_review_instructions(
    model_name="catalog.schema.customer_support_agent",
    instructions="""
    다음 시나리오를 테스트해주세요:
    1. "최근 주문 상태를 알려주세요" 같은 일반적인 질문
    2. 존재하지 않는 주문번호로 질문
    3. 한국어/영어 혼용 질문
    4. 에이전트가 답변할 수 없는 범위 외 질문

    각 응답에 대해 피드백과 코멘트를 남겨주세요.
    """
)

# 특정 사용자에게 Review App 접근 권한 부여
agents.set_permissions(
    model_name="catalog.schema.customer_support_agent",
    users=["tester1@company.com", "tester2@company.com"],
    permission="CAN_QUERY"
)
```

### 피드백 활용

수집된 피드백은 Unity Catalog의 Inference Tables에 저장되며, 다음과 같이 활용할 수 있습니다:

| 활용 방법 | 설명 |
|----------|------|
| **평가 데이터셋 구축**| 실제 사용자 질문 + 피드백을 Agent Evaluation의 골드 데이터셋으로 활용합니다 |
| **문제 패턴 분석**| 부정적 피드백이 많은 질문 유형을 분석하여 개선 방향을 도출합니다 |
| **A/B 테스트 근거** | 새 버전과 기존 버전의 피드백 점수를 비교합니다 |

---
