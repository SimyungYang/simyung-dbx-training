# 권한 관리와 Model Serving 연동

## 왜 모델 권한 관리가 중요한가

머신러닝 모델은 단순한 코드 파일이 아닙니다. 프로덕션 환경에서 실행되는 모델은 **비즈니스 의사결정** 에 직접 영향을 미치며, 잘못된 모델이 배포되면 금전적 손실이나 규제 위반으로 이어질 수 있습니다.

### 무분별한 모델 배포의 위험성

- **검증되지 않은 모델 배포**: 실험 중인 모델이 실수로 프로덕션에 배포되어 예측 품질 저하
- **모델 오염 (Model Poisoning)**: 악의적 사용자가 학습 데이터나 모델 가중치를 변조
- **규제 위반**: 금융/의료 도메인에서 승인되지 않은 모델 사용 시 법적 제재 (GDPR, HIPAA 등)
- **추적 불가**: 누가 언제 어떤 모델을 배포했는지 기록이 없으면 사고 원인 분석 불가

### 규제 준수 (Compliance) 요구사항

| 규제 | 관련 요구사항 |
|------|--------------|
| **GDPR** | 자동화된 의사결정 모델의 설명 가능성, 접근 권한 기록 |
| **HIPAA** | 의료 데이터를 사용한 모델의 접근 제어 및 감사 로그 |
| **SOC 2** | 모델 변경 이력, 배포 승인 프로세스 문서화 |
| **금융 규제** | 신용 심사 모델의 변경 통제 및 독립적 검증 요구 |

### Unity Catalog (UC)가 제공하는 보호

Unity Catalog 기반 Model Registry는 **테이블과 동일한 거버넌스 체계** 를 모델에 적용합니다. 단일 접근 제어 계층에서 데이터 → 피처 → 모델 → 엔드포인트까지 일관된 권한 관리가 가능합니다.

---

## Unity Catalog 기반 모델 권한 체계

### 권한 계층 구조

UC 모델 권한은 **Catalog → Schema → Model** 3단계 네임스페이스를 따릅니다. 상위 레벨 권한이 없으면 하위 레벨에 접근할 수 없습니다.

```sql
-- 1단계: Catalog 접근 권한 (모든 하위 객체 접근의 전제 조건)
GRANT USE CATALOG ON CATALOG ml_catalog TO `ds_team`;

-- 2단계: Schema 접근 권한
GRANT USE SCHEMA ON SCHEMA ml_catalog.fraud_models TO `ds_team`;

-- 3단계: 모델 등록 권한 (새 모델 생성)
GRANT CREATE MODEL ON SCHEMA ml_catalog.fraud_models TO `ds_team`;

-- 3단계: 모델 실행 권한 (추론/예측 호출)
GRANT EXECUTE ON MODEL ml_catalog.fraud_models.fraud_detection TO `inference_service`;

-- 권한 확인
SHOW GRANTS ON MODEL ml_catalog.fraud_models.fraud_detection;

-- 권한 회수
REVOKE EXECUTE ON MODEL ml_catalog.fraud_models.fraud_detection FROM `inference_service`;
```

### 주요 권한 설명

| 권한 | 적용 대상 | 허용 작업 |
|------|----------|----------|
| `USE CATALOG` | Catalog | Catalog 내 객체 탐색 |
| `USE SCHEMA` | Schema | Schema 내 객체 탐색 |
| `CREATE MODEL` | Schema | 새 모델 등록 (새 버전 포함) |
| `EXECUTE` | Model | 모델 버전 조회, 추론 호출 |
| `APPLY TAG` | Model | 모델에 태그 추가/수정 |
| `ALL PRIVILEGES` | Model | 모든 작업 (소유자 수준) |

> **참고**: `EXECUTE` 권한은 모델 가중치 파일 다운로드를 허용하지 않습니다. 모델 아티팩트 (Artifact) 접근은 별도의 Storage Credential 권한으로 제어됩니다.

**참고 링크**: [Unity Catalog privileges for ML models](https://docs.databricks.com/en/data-governance/unity-catalog/manage-privileges/privileges.html#ml-model-privileges)

---

## 역할별 권한 설계

실제 조직에서는 역할 (Role) 기반으로 권한을 그룹화하는 것이 권장됩니다.

### 데이터 사이언티스트 (Data Scientist)

실험을 수행하고 모델을 등록하는 역할. 프로덕션 배포 권한은 없습니다.

```sql
-- DS 팀 그룹에 최소 권한 부여
GRANT USE CATALOG ON CATALOG ml_catalog TO `data_scientists`;
GRANT USE SCHEMA ON SCHEMA ml_catalog.experiments TO `data_scientists`;
GRANT USE SCHEMA ON SCHEMA ml_catalog.staging_models TO `data_scientists`;

-- 실험용 Schema에 모델 등록 허용
GRANT CREATE MODEL ON SCHEMA ml_catalog.staging_models TO `data_scientists`;

-- 스테이징 모델 읽기/실행 허용
GRANT EXECUTE ON SCHEMA ml_catalog.staging_models TO `data_scientists`;
```

### ML 엔지니어 (ML Engineer)

승인된 모델을 프로덕션에 배포하고 모니터링하는 역할.

```sql
-- ML 엔지니어는 프로덕션 Schema에 대한 쓰기 권한 보유
GRANT USE CATALOG ON CATALOG ml_catalog TO `ml_engineers`;
GRANT USE SCHEMA ON SCHEMA ml_catalog.production_models TO `ml_engineers`;
GRANT CREATE MODEL ON SCHEMA ml_catalog.production_models TO `ml_engineers`;
GRANT EXECUTE ON SCHEMA ml_catalog.production_models TO `ml_engineers`;

-- Alias 관리를 위한 ALL PRIVILEGES (champion/challenger 전환)
GRANT ALL PRIVILEGES ON SCHEMA ml_catalog.production_models TO `ml_engineers`;
```

### 모델 소비자 (Model Consumer)

추론 API를 호출하는 애플리케이션 또는 팀. 읽기 전용 권한만 부여합니다.

```sql
-- 추론만 허용 (모델 수정 불가)
GRANT USE CATALOG ON CATALOG ml_catalog TO `analytics_team`;
GRANT USE SCHEMA ON SCHEMA ml_catalog.production_models TO `analytics_team`;
GRANT EXECUTE ON MODEL ml_catalog.production_models.fraud_detection TO `analytics_team`;
```

### 관리자 (Admin)

전체 모델 생명주기를 관리하는 역할. 서비스 프린시펄 (Service Principal) 계정으로 운영하는 것을 권장합니다.

```sql
-- Catalog 소유자 지정 (일반적으로 관리자 그룹)
ALTER CATALOG ml_catalog SET OWNER TO `ml_platform_admins`;

-- Schema 소유자 지정
ALTER SCHEMA ml_catalog.production_models SET OWNER TO `ml_platform_admins`;
```

---

## Model Serving 연동

### 모델 → 엔드포인트 배포 워크플로

```
[Model Registry (UC)]
    ↓ Alias: "champion"
[Model Serving Endpoint]
    ↓ REST API
[애플리케이션 / 대시보드]
```

Alias (별칭) 를 활용하면 엔드포인트 설정을 변경하지 않고 **새 모델 버전으로 무중단 전환** 이 가능합니다.

```python
import mlflow
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Alias를 참조하여 배포 → Alias 변경 시 자동 전환
w.serving_endpoints.create(
    name="fraud-detection-endpoint",
    config={
        "served_entities": [{
            "entity_name": "ml_catalog.production_models.fraud_detection",
            "entity_version": "champion",  # Alias 사용 (버전 번호 대신)
            "workload_size": "Small",       # Small / Medium / Large
            "scale_to_zero_enabled": True   # 비용 최적화
        }]
    }
)
```

### 서비스 프린시펄 (Service Principal) 설정

엔드포인트가 UC 모델에 접근하려면 **서비스 프린시펄** 에 권한을 부여해야 합니다. 개인 계정 대신 서비스 프린시펄을 사용하면 담당자 이직/퇴사 시에도 서비스가 중단되지 않습니다.

```python
# 1. 서비스 프린시펄 ID 확인
sp = w.service_principals.get(application_id="<app-id>")
print(f"Service Principal ID: {sp.id}")
```

```sql
-- 2. 서비스 프린시펄에 모델 접근 권한 부여
GRANT USE CATALOG ON CATALOG ml_catalog TO `<service-principal-name>`;
GRANT USE SCHEMA ON SCHEMA ml_catalog.production_models TO `<service-principal-name>`;
GRANT EXECUTE ON MODEL ml_catalog.production_models.fraud_detection TO `<service-principal-name>`;
```

**참고 링크**: [Model Serving with Unity Catalog](https://docs.databricks.com/en/machine-learning/model-serving/model-serving-intro.html)

---

## 엔드포인트 권한 관리

Model Serving 엔드포인트 자체에도 별도의 권한 체계가 있습니다. UC 모델 권한과는 독립적으로 관리됩니다.

### 엔드포인트 권한 레벨

| 권한 | 허용 작업 |
|------|----------|
| `CAN_MANAGE` | 엔드포인트 생성/수정/삭제, 권한 관리 |
| `CAN_QUERY` | 엔드포인트에 추론 요청 전송 (REST API 호출) |
| `CAN_VIEW` | 엔드포인트 상태 및 설정 조회 |

```python
from databricks.sdk.service.serving import ServingEndpointAccessControlRequest, ServingEndpointPermissionLevel

# 엔드포인트 권한 설정
w.serving_endpoints.set_permissions(
    serving_endpoint_id="fraud-detection-endpoint",
    access_control_list=[
        ServingEndpointAccessControlRequest(
            group_name="analytics_team",
            permission_level=ServingEndpointPermissionLevel.CAN_QUERY
        ),
        ServingEndpointAccessControlRequest(
            group_name="ml_engineers",
            permission_level=ServingEndpointPermissionLevel.CAN_MANAGE
        )
    ]
)
```

### PAT (Personal Access Token) vs OAuth 토큰

| 방식 | 장점 | 단점 |
|------|------|------|
| **PAT** | 설정 간단 | 만료 관리 필요, 개인 계정에 종속 |
| **OAuth M2M** | 서비스 프린시펄 기반, 자동 갱신 | 초기 설정 복잡 |
| **Instance Profile (AWS)** | IAM 기반, 자격증명 불필요 | EC2/클러스터에서만 사용 가능 |

프로덕션 환경에서는 **OAuth M2M (Machine-to-Machine)** 방식을 권장합니다.

```python
# OAuth M2M 토큰으로 엔드포인트 호출
import requests

# 1. 서비스 프린시펄 클라이언트 자격증명으로 OAuth 토큰 발급
token_response = requests.post(
    "https://<workspace-url>/oidc/v1/token",
    data={
        "grant_type": "client_credentials",
        "client_id": "<client-id>",
        "client_secret": "<client-secret>",
        "scope": "all-apis"
    }
)
access_token = token_response.json()["access_token"]

# 2. 토큰으로 엔드포인트 호출
response = requests.post(
    "https://<workspace-url>/serving-endpoints/fraud-detection-endpoint/invocations",
    headers={"Authorization": f"Bearer {access_token}"},
    json={"inputs": [{"feature_1": 0.5, "feature_2": 1.2}]}
)
```

---

## CI/CD와 자동화

### 서비스 프린시펄 기반 자동 배포 패턴

수동 배포 대신 CI/CD 파이프라인으로 모델 배포를 자동화하면 일관성과 추적성이 향상됩니다.

```
[GitHub PR 머지]
    ↓
[GitHub Actions: 모델 학습 & 평가]
    ↓ (평가 통과 시)
[MLflow: UC Registry에 새 버전 등록]
    ↓
[GitHub Actions: Alias "champion" 업데이트]
    ↓
[Model Serving 엔드포인트: 자동 전환]
```

### GitHub Actions 연동 예시

```yaml
# .github/workflows/deploy-model.yml
name: Deploy ML Model

on:
  push:
    branches: [main]
    paths: ['models/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: pip install databricks-sdk mlflow

      - name: Train and register model
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_CLIENT_ID: ${{ secrets.DATABRICKS_SP_CLIENT_ID }}
          DATABRICKS_CLIENT_SECRET: ${{ secrets.DATABRICKS_SP_CLIENT_SECRET }}
        run: |
          python scripts/train_and_register.py \
            --model-name "ml_catalog.production_models.fraud_detection" \
            --experiment-name "/Shared/fraud-detection-experiment"

      - name: Promote to champion
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_CLIENT_ID: ${{ secrets.DATABRICKS_SP_CLIENT_ID }}
          DATABRICKS_CLIENT_SECRET: ${{ secrets.DATABRICKS_SP_CLIENT_SECRET }}
        run: |
          python scripts/promote_model.py \
            --model-name "ml_catalog.production_models.fraud_detection" \
            --alias "champion"
```

```python
# scripts/promote_model.py
import mlflow
import argparse
from databricks.sdk import WorkspaceClient

def promote_model(model_name: str, alias: str):
    """평가를 통과한 최신 모델 버전에 Alias를 설정합니다."""
    client = mlflow.MlflowClient()

    # 최신 버전 조회
    versions = client.search_model_versions(f"name='{model_name}'")
    latest_version = sorted(versions, key=lambda v: int(v.version))[-1]

    # 메트릭 검증 (예: F1-score > 0.85)
    run = mlflow.get_run(latest_version.run_id)
    f1_score = run.data.metrics.get("f1_score", 0)

    if f1_score < 0.85:
        raise ValueError(f"F1 score {f1_score:.3f} is below threshold 0.85. Deployment aborted.")

    # Alias 업데이트 → Model Serving 엔드포인트 자동 전환
    client.set_registered_model_alias(
        name=model_name,
        alias=alias,
        version=latest_version.version
    )
    print(f"Promoted version {latest_version.version} to '{alias}' (F1: {f1_score:.3f})")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--model-name", required=True)
    parser.add_argument("--alias", default="champion")
    args = parser.parse_args()
    promote_model(args.model_name, args.alias)
```

> **보안 주의**: GitHub Actions Secret에 `DATABRICKS_SP_CLIENT_SECRET`을 저장하고, 서비스 프린시펄에는 배포에 필요한 최소 권한만 부여합니다.

---

## 감사와 컴플라이언스

### system.access.audit 활용

Unity Catalog는 모든 모델 접근 이벤트를 `system.access.audit` 테이블에 자동으로 기록합니다. 별도의 로깅 코드 없이 감사 추적이 가능합니다.

```sql
-- 최근 7일간 모델 접근 로그 조회
SELECT
    event_time,
    user_identity.email AS user,
    action_name,
    request_params,
    response.status_code
FROM system.access.audit
WHERE
    service_name = 'unityCatalog'
    AND action_name IN (
        'getModel', 'createModelVersion', 'updateModelVersion',
        'setRegisteredModelAlias', 'deleteModelVersion'
    )
    AND event_time >= CURRENT_TIMESTAMP - INTERVAL 7 DAYS
ORDER BY event_time DESC;
```

```sql
-- 특정 모델의 배포 이력 추적 (Alias 변경 이력)
SELECT
    event_time,
    user_identity.email AS deployed_by,
    request_params:name::STRING AS model_name,
    request_params:alias::STRING AS alias,
    request_params:version::STRING AS version
FROM system.access.audit
WHERE
    action_name = 'setRegisteredModelAlias'
    AND request_params:name::STRING LIKE '%fraud_detection%'
ORDER BY event_time DESC;
```

```sql
-- 모델 엔드포인트 호출 통계 (이상 접근 탐지)
SELECT
    DATE_TRUNC('hour', event_time) AS hour,
    user_identity.email AS caller,
    COUNT(*) AS call_count
FROM system.access.audit
WHERE
    service_name = 'modelServing'
    AND action_name = 'serveEndpoint'
    AND event_time >= CURRENT_TIMESTAMP - INTERVAL 24 HOURS
GROUP BY 1, 2
HAVING call_count > 1000  -- 비정상적으로 많은 호출 탐지
ORDER BY call_count DESC;
```

**참고 링크**: [Monitor Unity Catalog activity with system tables](https://docs.databricks.com/en/administration-guide/system-tables/audit.html)

---

## UC Model Registry vs Legacy Workspace Registry

Databricks에는 두 가지 Model Registry가 존재합니다. 현재 **UC Model Registry** 가 표준이며, Workspace Registry는 레거시로 분류됩니다.

| 항목 | Workspace Registry (레거시) | UC Model Registry (현재 표준) |
|------|---------------------------|-------------------------------|
| **네임스페이스** | `model_name` (flat) | `catalog.schema.model_name` (3-Level) |
| **접근 제어** | Workspace ACL (제한적) | UC GRANT/REVOKE (세밀한 권한) |
| **크로스 워크스페이스** | 불가 | 가능 (UC 공유) |
| **리니지** | 제한적 | 자동 리니지 (소스 데이터 → 모델) |
| **감사 로그** | Workspace 로그 | UC 시스템 테이블 (system.access.audit) |
| **Stage** | Staging/Production/Archived | Alias (자유 정의) |
| **Delta Sharing** | 불가 | 모델을 외부와 공유 가능 |

### Workspace Registry → UC Registry 마이그레이션

```python
import mlflow

mlflow.set_registry_uri("databricks-uc")
client = mlflow.MlflowClient()

# 1. Workspace Registry의 모델 버전 목록 조회
# (임시로 레거시 URI 사용)
legacy_client = mlflow.MlflowClient(registry_uri="databricks")
versions = legacy_client.search_model_versions("name='legacy_fraud_detection'")

# 2. 각 버전을 UC Registry로 복사
for version in sorted(versions, key=lambda v: int(v.version)):
    source_uri = f"models:/legacy_fraud_detection/{version.version}"

    new_version = mlflow.register_model(
        model_uri=source_uri,
        name="ml_catalog.production_models.fraud_detection"
    )
    print(f"Migrated version {version.version} → UC version {new_version.version}")

# 3. 프로덕션 버전에 champion Alias 설정
client.set_registered_model_alias(
    name="ml_catalog.production_models.fraud_detection",
    alias="champion",
    version=new_version.version
)
```

> ⚠️ **마이그레이션 주의사항**: Model Serving 엔드포인트의 참조도 함께 업데이트해야 합니다. `models:/legacy_fraud_detection/Production` → `ml_catalog.production_models.fraud_detection@champion` 으로 변경합니다.

---

## 베스트 프랙티스와 흔한 실수

### 권장 사항

| 항목 | 권장 | 피해야 할 것 |
|------|------|-------------|
| **계정 관리** | 서비스 프린시펄 사용 | 개인 PAT 토큰으로 자동화 |
| **권한 부여** | 최소 권한 원칙 (Least Privilege) | `ALL PRIVILEGES`를 모든 팀에 부여 |
| **배포 방식** | Alias 기반 무중단 전환 | 버전 번호 하드코딩 |
| **승인 프로세스** | CI/CD 파이프라인 + 메트릭 검증 게이트 | 수동 노트북 실행으로 배포 |
| **토큰 관리** | OAuth M2M, 짧은 만료 시간 | 만료 없는 PAT 토큰 사용 |

### 흔한 실수와 해결 방법

**실수 1: USE CATALOG/USE SCHEMA 없이 EXECUTE만 부여**
```sql
-- 잘못된 예 (접근 불가)
GRANT EXECUTE ON MODEL ml_catalog.models.fraud_detection TO `user@company.com`;

-- 올바른 예 (상위 계층 권한 포함)
GRANT USE CATALOG ON CATALOG ml_catalog TO `user@company.com`;
GRANT USE SCHEMA ON SCHEMA ml_catalog.models TO `user@company.com`;
GRANT EXECUTE ON MODEL ml_catalog.models.fraud_detection TO `user@company.com`;
```

**실수 2: 엔드포인트 권한과 모델 권한 혼동**
```python
# Model Serving 엔드포인트 호출 권한 (CAN_QUERY)과
# UC 모델 접근 권한 (EXECUTE)은 별도로 관리됩니다.
# 두 권한 모두 없으면 엔드포인트 호출이 실패합니다.
```

**실수 3: 개인 계정으로 엔드포인트 생성**
```
문제: 담당자 퇴사 시 엔드포인트가 비활성화되거나 오류 발생
해결: 서비스 프린시펄 계정으로 엔드포인트 생성 및 관리
```

**실수 4: Alias 없이 버전 번호로 배포**
```python
# 잘못된 예 - 새 버전 배포 시 엔드포인트 설정 변경 필요
"entity_version": "5"  # 버전 번호 하드코딩

# 올바른 예 - Alias 변경만으로 자동 전환
"entity_version": "champion"  # Alias 사용
```

**참고 링크**:
- [MLflow Model Registry with Unity Catalog](https://docs.databricks.com/en/mlflow/models-in-uc.html)
- [Set model serving endpoint permissions](https://docs.databricks.com/en/machine-learning/model-serving/manage-endpoint-permissions.html)
- [Service principals for CI/CD](https://docs.databricks.com/en/dev-tools/ci-cd/ci-cd-sp.html)
- [Audit log reference](https://docs.databricks.com/en/administration-guide/system-tables/audit.html)
