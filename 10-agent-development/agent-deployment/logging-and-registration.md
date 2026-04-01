# 모델 로깅과 등록

## 왜 에이전트 배포가 중요한가?

노트북에서 프로토타입으로 동작하는 에이전트와, 실제 사용자가 접근할 수 있는 프로덕션 에이전트는 전혀 다릅니다. 프로덕션 배포에는 **확장성, 모니터링, 보안, 버전 관리**가 필요합니다.

> 💡 Databricks에서 AI 에이전트를 배포한다는 것은, 에이전트를 **Model Serving Endpoint**로 호스팅하여 REST API로 접근할 수 있게 만드는 것입니다. 배포된 에이전트는 웹 앱, 슬랙 봇, 내부 도구 등 다양한 채널에서 호출할 수 있습니다.

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 에이전트 개발 | 노트북에서 에이전트를 개발합니다 |
| 2 | MLflow 로깅 | 모델을 패키징하여 MLflow에 기록합니다 |
| 3 | UC 모델 등록 | Unity Catalog에 등록하여 버전 관리합니다 |
| 4 | Model Serving | 엔드포인트로 배포합니다 (핵심 단계) |
| 5a | Review App | 테스터가 피드백을 제공합니다 |
| 5b | 프로덕션 | REST API로 실제 서비스에 사용됩니다 |

---

## 배포 아키텍처 개요

Databricks의 에이전트 배포는 **MLflow + Unity Catalog + Model Serving**의 통합 구조로 동작합니다.

| 구성 요소 | 역할 |
|----------|------|
| **MLflow**| 에이전트를 모델로 패키징하고, 의존성/설정을 함께 로깅합니다 |
| **Unity Catalog**| 모델을 카탈로그에 등록하여 버전 관리와 거버넌스를 적용합니다 |
| **Model Serving**| 등록된 모델을 REST API 엔드포인트로 호스팅합니다 |
| **Review App**| 배포 전 테스터들이 에이전트를 사용해보고 피드백을 남기는 UI입니다 |
| **Inference Tables**| 모든 요청/응답을 자동으로 기록하여 모니터링에 활용합니다 |

---

## Step 1: 에이전트 모델 로깅

배포의 첫 단계는 에이전트를 **MLflow 모델**로 로깅하는 것입니다. 이때 에이전트 코드, 의존성, 설정이 모두 함께 패키징됩니다.

### mlflow.models.set_model() 사용법

에이전트 코드를 별도 Python 파일로 분리한 뒤, `set_model()`로 진입점을 지정합니다. 이 방식이 Databricks에서 권장하는 표준 패턴입니다.

**agent.py**(에이전트 코드):

```python
import mlflow
from databricks.sdk import WorkspaceClient
from langchain_community.chat_models import ChatDatabricks
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate

# 에이전트 정의
class CustomerSupportAgent(mlflow.pyfunc.PythonModel):
    def __init__(self):
        self.llm = ChatDatabricks(
            endpoint="databricks-meta-llama-3-3-70b-instruct",
            temperature=0.1
        )

    def predict(self, context, model_input, params=None):
        messages = model_input.get("messages", [])
        response = self.llm.invoke(messages)
        return {"content": response.content}

# 진입점 지정 — 이 파일이 모델로 사용될 때 이 클래스가 로드됩니다
mlflow.models.set_model(CustomerSupportAgent())
```

** 로깅 노트북**(모델 등록):

```python
import mlflow

# MLflow 실험 설정
mlflow.set_experiment("/Users/user@example.com/customer-support-agent")

# 에이전트 로깅
with mlflow.start_run():
    model_info = mlflow.pyfunc.log_model(
        artifact_path="agent",
        python_model="agent.py",  # 에이전트 코드 파일 경로
        registered_model_name="catalog.schema.customer_support_agent",
        pip_requirements=[
            "langchain>=0.3.0",
            "langchain-community>=0.3.0",
            "databricks-sdk>=0.30.0",
        ],
        input_example={
            "messages": [
                {"role": "user", "content": "주문 상태를 확인하고 싶습니다."}
            ]
        }
    )
    print(f"Model URI: {model_info.model_uri}")
```

### 의존성 관리

에이전트가 사용하는 모든 외부 패키지와 리소스를 명시해야 합니다.

| 의존성 유형 | 지정 방법 | 예시 |
|-----------|----------|------|
| **Python 패키지**| `pip_requirements` | `["langchain>=0.3.0", "tiktoken"]` |
| ** 환경 변수**| 엔드포인트 설정 시 지정 | `VECTOR_SEARCH_ENDPOINT`, `API_KEY` |
| ** 리소스 의존성**| `resources` 파라미터 | Vector Search Index, Serving Endpoint |
| ** 추가 파일**| `code_paths` 또는 `artifacts` | 프롬프트 템플릿, 설정 파일 |

```python
# 리소스 의존성 명시 (배포 시 자동으로 권한 설정)
from mlflow.models.resources import (
    DatabricksServingEndpoint,
    DatabricksVectorSearchIndex
)

with mlflow.start_run():
    mlflow.pyfunc.log_model(
        artifact_path="agent",
        python_model="agent.py",
        registered_model_name="catalog.schema.my_agent",
        resources=[
            DatabricksServingEndpoint(endpoint_name="databricks-meta-llama-3-3-70b-instruct"),
            DatabricksVectorSearchIndex(index_name="catalog.schema.docs_index"),
        ]
    )
```

> 💡 `resources`에 명시된 리소스들은 배포 시 자동으로 서비스 프린시펄(Service Principal)에게 필요한 권한이 부여됩니다. 수동으로 권한을 설정할 필요가 없습니다.

---

## Step 2: Unity Catalog에 모델 등록

`registered_model_name`을 지정하면 MLflow가 자동으로 Unity Catalog에 모델을 등록합니다. 등록된 모델은 버전 관리, 접근 제어, 리니지 추적이 가능합니다.

### 모델 버전 관리

| 개념 | 설명 |
|------|------|
| ** 모델 이름**| `catalog.schema.model_name` 형식의 3단계 네임스페이스 |
| ** 버전**| 로깅할 때마다 자동으로 버전 번호가 증가합니다 (v1, v2, v3...) |
| ** 앨리어스** | `champion`, `challenger` 같은 이름으로 특정 버전을 지칭할 수 있습니다 |

```python
from mlflow import MlflowClient

client = MlflowClient()

# 특정 버전에 앨리어스 지정
client.set_registered_model_alias(
    name="catalog.schema.customer_support_agent",
    alias="champion",
    version=3
)

# 앨리어스로 모델 조회
champion_version = client.get_model_version_by_alias(
    name="catalog.schema.customer_support_agent",
    alias="champion"
)
print(f"Champion version: {champion_version.version}")
```

---
