# 기본 개념과 모델 등록

## 모델 레지스트리란?

> 💡 **Model Registry(모델 레지스트리)** 는 학습된 ML 모델의 **버전을 관리** 하고, 프로덕션 승격(Promotion)을 제어하는 중앙 저장소입니다. Databricks에서는 **Unity Catalog** 와 통합되어, 모델에도 테이블과 동일한 거버넌스(권한, 리니지, 감사)가 적용됩니다.

---

## 왜 모델 레지스트리가 필요한가요?

모델 학습을 반복하다 보면, "어떤 버전이 프로덕션에 배포되어 있지?", "이전 버전으로 롤백하려면?", "누가 이 모델을 승인했지?" 같은 질문이 생깁니다. Model Registry는 이 문제를 체계적으로 해결합니다.

| 문제 | Registry의 해결 |
|------|----------------|
| 어떤 버전이 프로덕션인지 모름 | **Alias**(champion, challenger)로 명확히 표시 |
| 이전 버전으로 롤백이 어려움 | 모든 **버전이 보존**. Alias만 변경하면 롤백 완료 |
| 모델의 출처를 모름 | **리니지** 추적 — 어떤 데이터, 어떤 실험에서 생성되었는지 |
| 권한 관리가 어려움 | Unity Catalog의 **GRANT/REVOKE** 로 접근 제어 |
| 모델 변경 이력 추적 | 모든 변경이 **감사 로그** 에 기록 |

---

## 모델 등록

### MLflow에서 모델 등록

```python
import mlflow

# 방법 1: 학습 중 직접 등록
mlflow.set_registry_uri("databricks-uc")  # Unity Catalog 사용

with mlflow.start_run():
    mlflow.sklearn.log_model(
        model,
        artifact_path="model",
        registered_model_name="catalog.schema.fraud_detection"  # UC 3-Level 이름
    )

# 방법 2: 기존 Run에서 나중에 등록
model_uri = f"runs:/{run_id}/model"
mlflow.register_model(model_uri, "catalog.schema.fraud_detection")
```

### 등록된 모델의 위치

| Unity Catalog 위치 | 오브젝트 | 설명 |
|-------------------|---------|------|
| catalog > schema | tables | 테이블 |
| | volumes | 파일 |
| | **models**| ML 모델 (여기에 등록) |
| | └ fraud_detection | 모델 예시 |

| Version | Date | Status |
|---------|------|--------|
| v1 | 2025-01-15 | - |
| v2 | 2025-02-20 | - |
| v3 | 2025-03-10 | champion |
```

---

## 버전 관리

모델을 등록할 때마다 자동으로 **새 버전** 이 생성됩니다.

| 버전 | 등록일 | 메트릭 | 상태 |
|------|--------|--------|------|
| v1 | 2025-01-15 | accuracy: 0.89 | 아카이브 |
| v2 | 2025-02-20 | accuracy: 0.92 | 아카이브 |
| v3 | 2025-03-10 | accuracy: 0.95 | **champion**(프로덕션) |

---

## Alias (별칭)

> 💡 **Alias** 는 특정 모델 버전에 부여하는 **별칭** 입니다. "champion"은 현재 프로덕션에 배포된 버전, "challenger"는 테스트 중인 다음 버전을 의미하는 것이 일반적입니다.

```python
from mlflow import MlflowClient

client = MlflowClient()

# "champion" 별칭을 버전 3에 부여
client.set_registered_model_alias(
    name="catalog.schema.fraud_detection",
    alias="champion",
    version=3
)

# "challenger" 별칭을 버전 4에 부여 (A/B 테스트용)
client.set_registered_model_alias(
    name="catalog.schema.fraud_detection",
    alias="challenger",
    version=4
)

# champion 모델 로드 (프로덕션 배포 시)
champion_model = mlflow.pyfunc.load_model(
    "models:/catalog.schema.fraud_detection@champion"
)

# 버전 번호로 로드
specific_model = mlflow.pyfunc.load_model(
    "models:/catalog.schema.fraud_detection/3"
)
```

### 롤백

문제가 발생하면 Alias만 변경하여 즉시 롤백합니다.

```python
# v3에 문제 발생 → v2로 즉시 롤백
client.set_registered_model_alias(
    name="catalog.schema.fraud_detection",
    alias="champion",
    version=2  # 이전 버전으로 변경
)
# → Model Serving이 @champion을 참조하고 있다면, 즉시 v2로 전환됩니다
```

---

## Tags (태그)

모델에 메타데이터 태그를 달아 검색과 관리를 용이하게 합니다.

```python
# 모델 태그 추가
client.set_registered_model_tag(
    name="catalog.schema.fraud_detection",
    key="team",
    value="risk-analytics"
)

client.set_model_version_tag(
    name="catalog.schema.fraud_detection",
    version=3,
    key="validation_status",
    value="approved"
)

client.set_model_version_tag(
    name="catalog.schema.fraud_detection",
    version=3,
    key="approved_by",
    value="kim@company.com"
)
```

---
