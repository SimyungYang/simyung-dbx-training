# 기본 개념과 모델 등록

## 모델 레지스트리란?

> **Model Registry(모델 레지스트리)** 는 학습된 ML 모델의 **버전을 관리** 하고, 프로덕션 승격(Promotion)을 제어하는 중앙 저장소입니다. Databricks에서는 **Unity Catalog** 와 통합되어, 모델에도 테이블과 동일한 거버넌스(권한, 리니지, 감사)가 적용됩니다.

---

## 1. 왜 모델 레지스트리가 필요한가

### 모델 관리 없이 운영할 때의 문제

실제 ML 운영 현장에서 레지스트리 없이 모델을 관리하면 다음과 같은 문제가 반복적으로 발생합니다.

**버전 혼란 (Version Chaos)**
- 파일 시스템에 `model_v2_final_FINAL.pkl` 같은 파일이 쌓입니다.
- 어떤 파일이 실제 프로덕션에 배포된 것인지 확인할 방법이 없습니다.
- 팀원이 실수로 구버전 모델을 배포하는 사고가 발생합니다.

**재현 불가 (Non-reproducibility)**
- 6개월 후 "이 모델을 어떻게 학습했지?"라는 질문에 답하기 어렵습니다.
- 어떤 데이터셋, 어떤 하이퍼파라미터로 학습했는지 추적이 끊깁니다.
- 동일한 코드로 재학습해도 결과가 달라지는 환경 의존성 문제가 생깁니다.

**감사 불가 (No Auditability)**
- 금융·의료 등 규제 산업에서는 "누가 이 모델을 승인했는가"를 입증해야 합니다.
- 모델 변경 이력이 없으면 컴플라이언스(Compliance) 감사를 통과할 수 없습니다.
- 인시던트(Incident) 발생 시 원인 추적이 불가능합니다.

| 문제 | Registry의 해결 |
|------|----------------|
| 어떤 버전이 프로덕션인지 모름 | **Alias** (champion, challenger)로 명확히 표시 |
| 이전 버전으로 롤백이 어려움 | 모든 **버전이 보존** 됨. Alias만 변경하면 롤백 완료 |
| 모델의 출처를 모름 | **Lineage(리니지)** 추적 — 어떤 데이터, 어떤 실험에서 생성되었는지 |
| 권한 관리가 어려움 | Unity Catalog의 **GRANT/REVOKE** 로 접근 제어 |
| 모델 변경 이력 추적 | 모든 변경이 **Audit Log(감사 로그)** 에 기록 |

---

## 2. Unity Catalog 기반 모델 레지스트리

### Workspace 레지스트리에서 UC 레지스트리로의 전환

Databricks는 원래 **Workspace Model Registry** 를 제공했습니다. 이는 Workspace 범위 내에서만 동작하는 격리된 레지스트리였습니다. 2023년부터 **Unity Catalog(UC) 기반 Model Registry** 가 권장 방식이 되었으며, 이유는 다음과 같습니다.

| 비교 항목 | Workspace 레지스트리 | UC 레지스트리 |
|-----------|---------------------|---------------|
| 범위 | 단일 Workspace | Account 전체 (멀티 Workspace) |
| 네임스페이스 | `model_name` | `catalog.schema.model_name` |
| 권한 관리 | Workspace 수준 ACL | UC GRANT/REVOKE (세밀한 제어) |
| 리니지 추적 | 제한적 | UC Lineage와 완전 통합 |
| Delta Sharing | 미지원 | 외부 조직과 모델 공유 가능 |
| 감사 로그 | Workspace 감사 | Account-level 통합 감사 |

### 3-Level Namespace (3단계 네임스페이스)

UC의 모든 오브젝트는 `catalog.schema.object` 형태의 3단계 네임스페이스를 사용합니다. 모델도 동일합니다.

```
ml_catalog                    ← Catalog (데이터 도메인 단위)
  └── fraud_detection         ← Schema (프로젝트/팀 단위)
        ├── tables/           ← 테이블
        ├── volumes/          ← 파일
        └── models/           ← ML 모델
              └── fraud_model ← 등록된 모델
```

모델의 전체 이름은 `ml_catalog.fraud_detection.fraud_model` 이 됩니다. 이 이름 하나로 버전 관리, 권한 부여, 리니지 추적이 모두 연결됩니다.

---

## 3. 모델 등록 방법

### 방법 1: 학습 중 직접 등록 (MLflow API)

```python
import mlflow
from mlflow import MlflowClient

# Unity Catalog를 레지스트리로 지정
mlflow.set_registry_uri("databricks-uc")

# 실험(Experiment) 설정
mlflow.set_experiment("/Users/user@company.com/fraud-detection")

with mlflow.start_run(run_name="xgboost-v3") as run:
    # 모델 학습
    model.fit(X_train, y_train)

    # 메트릭 로깅
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("f1_score", f1)

    # 모델 로깅 + 레지스트리 등록을 한 번에
    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="model",
        registered_model_name="ml_catalog.fraud_detection.fraud_model"  # UC 3-level 이름
    )
```

### 방법 2: 기존 Run에서 사후 등록

```python
# 이미 완료된 Run의 모델을 나중에 레지스트리에 등록
run_id = "a1b2c3d4e5f6..."
model_uri = f"runs:/{run_id}/model"

model_version = mlflow.register_model(
    model_uri=model_uri,
    name="ml_catalog.fraud_detection.fraud_model"
)
print(f"등록된 버전: {model_version.version}")
```

### 방법 3: UI에서 등록

1. Databricks 좌측 메뉴 → **Experiments** 진입
2. 실험 Run 클릭 → **Artifacts** 탭 선택
3. `model` 폴더 클릭 → **Register Model** 버튼 클릭
4. 모델 이름을 `catalog.schema.model_name` 형식으로 입력
5. **Register** 클릭 → 새 버전 자동 생성

### 방법 4: 자동 등록 (autolog)

```python
# MLflow autolog 사용 시 모델도 자동 로깅
mlflow.sklearn.autolog(
    registered_model_name="ml_catalog.fraud_detection.fraud_model"
)

with mlflow.start_run():
    model.fit(X_train, y_train)
    # 학습이 끝나면 자동으로 모델 등록
```

---

## 4. 모델 버전 관리

### 버전 번호 체계

모델을 동일한 이름으로 등록할 때마다 **버전 번호가 자동으로 증가** 합니다. 버전은 삭제하지 않는 한 영구 보존됩니다.

| 버전 | 등록일 | accuracy | f1_score | 상태 |
|------|--------|----------|----------|------|
| v1 | 2025-01-15 | 0.89 | 0.87 | 아카이브 |
| v2 | 2025-02-20 | 0.92 | 0.91 | 아카이브 |
| v3 | 2025-03-10 | 0.95 | 0.94 | **champion** (프로덕션) |
| v4 | 2025-04-01 | 0.96 | 0.95 | challenger (테스트 중) |

### Alias (별칭) 관리

**Alias** 는 특정 버전에 부여하는 **가변 포인터** 입니다. 버전 번호 대신 Alias를 참조하면, 내부 버전을 바꿔도 참조 코드를 수정할 필요가 없습니다.

```python
from mlflow import MlflowClient

client = MlflowClient()

# "champion" 별칭을 버전 3에 부여 (프로덕션 버전)
client.set_registered_model_alias(
    name="ml_catalog.fraud_detection.fraud_model",
    alias="champion",
    version=3
)

# "challenger" 별칭을 버전 4에 부여 (A/B 테스트용)
client.set_registered_model_alias(
    name="ml_catalog.fraud_detection.fraud_model",
    alias="challenger",
    version=4
)

# champion 모델 로드 — 내부 버전이 바뀌어도 이 코드는 변경 불필요
champion_model = mlflow.pyfunc.load_model(
    "models:/ml_catalog.fraud_detection.fraud_model@champion"
)

# 특정 버전 번호로 직접 로드
v2_model = mlflow.pyfunc.load_model(
    "models:/ml_catalog.fraud_detection.fraud_model/2"
)
```

### 버전별 메타데이터와 태그

```python
# 모델(전체) 레벨 태그
client.set_registered_model_tag(
    name="ml_catalog.fraud_detection.fraud_model",
    key="team",
    value="risk-analytics"
)

# 버전 레벨 태그 — 검토·승인 워크플로에 활용
client.set_model_version_tag(
    name="ml_catalog.fraud_detection.fraud_model",
    version=3,
    key="validation_status",
    value="approved"
)

client.set_model_version_tag(
    name="ml_catalog.fraud_detection.fraud_model",
    version=3,
    key="approved_by",
    value="kim@company.com"
)

client.set_model_version_tag(
    name="ml_catalog.fraud_detection.fraud_model",
    version=3,
    key="approval_date",
    value="2025-03-08"
)
```

---

## 5. 모델 배포 워크플로

### 개발 → 스테이징 → 프로덕션 (Alias 활용)

UC 레지스트리에서는 Stage 개념 대신 **Alias** 로 배포 단계를 표현합니다. 일반적인 워크플로는 다음과 같습니다.

```
[실험/개발]        [스테이징]          [프로덕션]
  Run #1      →  @staging        →  @champion
  Run #2
  Run #3  ────────────────────────────────────
                  @challenger    (A/B 테스트)
```

```python
# 1단계: 새 버전을 스테이징으로 승격
client.set_registered_model_alias(
    name="ml_catalog.fraud_detection.fraud_model",
    alias="staging",
    version=4
)

# 2단계: 검증 통과 후 프로덕션(champion)으로 승격
client.set_registered_model_alias(
    name="ml_catalog.fraud_detection.fraud_model",
    alias="champion",
    version=4
)

# 3단계: 이전 champion(v3)의 staging alias 제거 (정리)
client.delete_registered_model_alias(
    name="ml_catalog.fraud_detection.fraud_model",
    alias="staging"
)
```

### 롤백 (Rollback)

문제가 발생하면 Alias만 변경하여 즉시 이전 버전으로 롤백합니다.

```python
# v4에 문제 발생 → v3으로 즉시 롤백
client.set_registered_model_alias(
    name="ml_catalog.fraud_detection.fraud_model",
    alias="champion",
    version=3  # 안정적인 이전 버전으로 변경
)
# Model Serving이 @champion을 참조하면 서빙 코드 수정 없이 즉시 롤백됩니다.
```

### Model Serving 엔드포인트와 연결

```python
import requests

# Model Serving 엔드포인트 생성 시 Alias로 모델 지정
endpoint_config = {
    "name": "fraud-detection-endpoint",
    "config": {
        "served_models": [
            {
                "model_name": "ml_catalog.fraud_detection.fraud_model",
                "model_version": "champion",  # Alias 참조
                "workload_size": "Small",
                "scale_to_zero_enabled": True
            }
        ]
    }
}
# → champion Alias가 가리키는 버전이 자동으로 서빙됩니다.
# → 롤백 시 Alias만 변경하면 엔드포인트 재배포 없이 반영됩니다.
```

---

## 6. 권한과 거버넌스

### UC 기반 권한 체계

UC 레지스트리는 **SQL GRANT 문법** 으로 권한을 부여합니다. 테이블 권한과 동일한 체계를 사용하므로 데이터 거버넌스와 일관성 있게 관리됩니다.

```sql
-- 카탈로그 사용 권한 부여
GRANT USE CATALOG ON CATALOG ml_catalog TO `data-science-team`;

-- 스키마 사용 권한 부여
GRANT USE SCHEMA ON SCHEMA ml_catalog.fraud_detection TO `data-science-team`;

-- 모델 실행(추론) 권한 — 서빙 서비스 계정에 부여
GRANT EXECUTE ON MODEL ml_catalog.fraud_detection.fraud_model TO `serving-sp`;

-- 모델 등록·수정 권한 — ML 엔지니어에게 부여
GRANT CREATE MODEL ON SCHEMA ml_catalog.fraud_detection TO `ml-engineer`;

-- 읽기 전용 — 비즈니스 분석팀에 부여
GRANT SELECT ON MODEL ml_catalog.fraud_detection.fraud_model TO `ba-team`;
```

### 감사 로그 (Audit Log)

UC 레지스트리의 모든 작업은 **자동으로 감사 로그** 에 기록됩니다.

```sql
-- 모델 레지스트리 감사 로그 조회
SELECT
    event_time,
    user_identity.email AS user,
    action_name,
    request_params
FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND action_name IN (
      'registerModel', 'updateRegisteredModel',
      'setRegisteredModelAlias', 'deleteModelVersion'
  )
  AND request_params:full_name LIKE 'ml_catalog.fraud_detection%'
ORDER BY event_time DESC;
```

기록되는 주요 이벤트는 다음과 같습니다.

| 이벤트 | 설명 |
|--------|------|
| `registerModel` | 새 모델/버전 등록 |
| `setRegisteredModelAlias` | Alias 변경 (프로덕션 배포 이력) |
| `updateRegisteredModel` | 모델 메타데이터·태그 수정 |
| `deleteModelVersion` | 버전 삭제 |
| `getRegisteredModel` | 모델 조회 (누가 언제 조회했는지) |

---

## 7. 장단점과 트레이드오프

### UC 레지스트리 vs Workspace 레지스트리

| 기준 | UC 레지스트리 | Workspace 레지스트리 |
|------|--------------|---------------------|
| **장점** | 멀티 Workspace 공유, 세밀한 권한, 감사 로그, Delta Sharing | 설정 간단, UC 미사용 환경에서도 동작 |
| **단점** | UC 활성화 필수, 마이그레이션 작업 필요 | 단일 Workspace 격리, 권한 체계 제한 |
| **추천 대상** | 신규 프로젝트, 엔터프라이즈 환경 | 레거시 환경, 빠른 프로토타이핑 |

### 마이그레이션 고려사항

Workspace 레지스트리에서 UC 레지스트리로 마이그레이션할 때 주의할 점입니다.

- **네임스페이스 변경**: `model_name` → `catalog.schema.model_name` 으로 모든 참조를 업데이트해야 합니다.
- **Stage → Alias 전환**: `Production`, `Staging` Stage 개념이 없어지고 Alias로 대체됩니다.
- **Model Serving 엔드포인트 재생성**: Workspace 레지스트리를 참조하는 엔드포인트는 재배포가 필요합니다.
- **권한 재설정**: Workspace ACL을 UC GRANT 문법으로 재작성해야 합니다.
- **공식 마이그레이션 도구**: `mlflow.models.migrate_to_uc()` API 또는 Databricks UI의 마이그레이션 위저드를 활용합니다.

---

## 8. 베스트 프랙티스

**네이밍 컨벤션을 팀 전체에서 통일합니다.**
- Catalog: 데이터 도메인 단위 (`ml_prod`, `ml_dev`)
- Schema: 프로젝트/팀 단위 (`fraud_detection`, `recommendation`)
- Model: 모델 역할 중심 (`transaction_scorer`, `item_ranker`)

**버전을 절대 삭제하지 않습니다.**
- 디버깅과 규제 감사를 위해 모든 버전을 보존합니다.
- 불필요한 버전은 Alias를 제거하는 것으로 충분합니다.

**Alias를 코드의 진입점으로 사용합니다.**
- 서빙 코드, 배치 추론 코드 모두 버전 번호 대신 `@champion` 같은 Alias를 참조합니다.
- 롤백·업그레이드가 코드 변경 없이 가능해집니다.

**태그로 검토 워크플로를 구현합니다.**
- `validation_status: pending → approved → rejected` 태그로 모델 심사 프로세스를 추적합니다.
- `approved_by`, `approval_date` 태그로 규제 감사 요건을 충족합니다.

**CI/CD 파이프라인에서 자동 등록을 구현합니다.**
- PR 머지 시 재학습 → 메트릭 검증 → 자동 등록 → `@staging` Alias 부여까지 자동화합니다.
- 사람은 `@staging` → `@champion` 승격 단계에만 개입합니다.

**UC 권한을 최소 권한 원칙으로 설정합니다.**
- 학습 파이프라인: `CREATE MODEL` 권한만 부여
- 서빙 서비스 계정: `EXECUTE` 권한만 부여
- 감사자·분석팀: `SELECT` 권한만 부여

---

## 참고 링크

- [MLflow Model Registry — Databricks 공식 문서](https://docs.databricks.com/en/mlflow/model-registry.html)
- [Unity Catalog로 모델 관리](https://docs.databricks.com/en/machine-learning/manage-model-lifecycle/index.html)
- [Workspace 레지스트리 → UC 레지스트리 마이그레이션](https://docs.databricks.com/en/mlflow/migrate-models-to-uc.html)
- [MLflow Model Registry API 레퍼런스](https://mlflow.org/docs/latest/model-registry.html)
- [UC 권한 관리 (Privileges)](https://docs.databricks.com/en/data-governance/unity-catalog/manage-privileges/privileges.html)
- [Audit Log — system.access.audit](https://docs.databricks.com/en/administration-guide/account-settings/audit-logs.html)
