# 모델 레지스트리

## 모델 레지스트리란?

> 💡 **Model Registry(모델 레지스트리)** 는 학습된 ML 모델의 **버전을 관리**하고, 프로덕션 승격(Promotion)을 제어하는 중앙 저장소입니다. Databricks에서는 **Unity Catalog**와 통합되어, 모델에도 테이블과 동일한 거버넌스(권한, 리니지, 감사)가 적용됩니다.

---

## 왜 모델 레지스트리가 필요한가요?

모델 학습을 반복하다 보면, "어떤 버전이 프로덕션에 배포되어 있지?", "이전 버전으로 롤백하려면?", "누가 이 모델을 승인했지?" 같은 질문이 생깁니다. Model Registry는 이 문제를 체계적으로 해결합니다.

| 문제 | Registry의 해결 |
|------|----------------|
| 어떤 버전이 프로덕션인지 모름 | **Alias** (champion, challenger)로 명확히 표시 |
| 이전 버전으로 롤백이 어려움 | 모든 **버전이 보존**. Alias만 변경하면 롤백 완료 |
| 모델의 출처를 모름 | **리니지** 추적 — 어떤 데이터, 어떤 실험에서 생성되었는지 |
| 권한 관리가 어려움 | Unity Catalog의 **GRANT/REVOKE**로 접근 제어 |
| 모델 변경 이력 추적 | 모든 변경이 **감사 로그**에 기록 |

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

```
Unity Catalog:
  catalog
    └── schema
         ├── tables (테이블)
         ├── volumes (파일)
         └── models (ML 모델)  ← 여기에 등록됩니다
              └── fraud_detection
                   ├── Version 1 (2025-01-15)
                   ├── Version 2 (2025-02-20)
                   └── Version 3 (2025-03-10) ← champion
```

---

## 버전 관리

모델을 등록할 때마다 자동으로 **새 버전**이 생성됩니다.

| 버전 | 등록일 | 메트릭 | 상태 |
|------|--------|--------|------|
| v1 | 2025-01-15 | accuracy: 0.89 | 아카이브 |
| v2 | 2025-02-20 | accuracy: 0.92 | 아카이브 |
| v3 | 2025-03-10 | accuracy: 0.95 | **champion** (프로덕션) |

---

## Alias (별칭)

> 💡 **Alias**는 특정 모델 버전에 부여하는 **별칭**입니다. "champion"은 현재 프로덕션에 배포된 버전, "challenger"는 테스트 중인 다음 버전을 의미하는 것이 일반적입니다.

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

## 모델 권한 관리

Unity Catalog에 등록된 모델은 테이블과 동일한 권한 체계로 관리됩니다.

```sql
-- 특정 팀에게 모델 사용 권한 부여
GRANT EXECUTE ON MODEL catalog.schema.fraud_detection TO `ml_engineers`;

-- 모델 등록 권한 부여
GRANT CREATE MODEL ON SCHEMA catalog.schema TO `data_scientists`;

-- 현재 권한 확인
SHOW GRANTS ON MODEL catalog.schema.fraud_detection;
```

---

## Model Serving 연동

등록된 모델을 Model Serving 엔드포인트에 배포합니다.

```python
# Alias를 참조하여 배포 → Alias 변경 시 자동 전환
w.serving_endpoints.create(
    name="fraud-detection",
    config={
        "served_entities": [{
            "entity_name": "catalog.schema.fraud_detection",
            "entity_version": "champion",  # Alias 사용
            "workload_size": "Small",
            "scale_to_zero_enabled": True
        }]
    }
)
```

---

## UC Model Registry vs Legacy Workspace Registry

### 비교

Databricks에는 두 가지 Model Registry가 존재합니다. 현재 **UC Model Registry**가 표준이며, Workspace Registry는 레거시로 분류됩니다.

| 항목 | Workspace Registry (레거시) | UC Model Registry (현재 표준) |
|------|---------------------------|-------------------------------|
| **네임스페이스** | `model_name` (flat) | `catalog.schema.model_name` (3-Level) |
| **접근 제어** | Workspace ACL (제한적) | UC GRANT/REVOKE (세밀한 권한) |
| **크로스 워크스페이스** | 불가 | 가능 (UC 공유) |
| **리니지** | 제한적 | 자동 리니지 (소스 데이터 → 모델) |
| **감사 로그** | Workspace 로그 | UC 시스템 테이블 (system.access.audit) |
| **Stage** | Staging/Production/Archived | Alias (자유 정의) |
| **Delta Sharing** | 불가 | 모델을 외부와 공유 가능 |
| **Serverless Model Serving** | 지원 | 지원 |

### 마이그레이션 방법

```python
import mlflow

# 1. Workspace Registry의 모델 정보 확인
client = mlflow.MlflowClient()
model = client.get_registered_model("legacy_fraud_detection")

# 2. UC Registry로 복사
for version in client.search_model_versions("name='legacy_fraud_detection'"):
    # 해당 버전의 모델 아티팩트 URI
    source_uri = f"models:/legacy_fraud_detection/{version.version}"

    # UC Registry에 등록
    mlflow.set_registry_uri("databricks-uc")
    mlflow.register_model(
        model_uri=source_uri,
        name="catalog.schema.fraud_detection"  # UC 3-Level 이름
    )
    print(f"Migrated version {version.version}")

# 3. 새 모델에 Alias 설정
client.set_registered_model_alias(
    name="catalog.schema.fraud_detection",
    alias="champion",
    version=version.version  # 마지막 프로덕션 버전
)
```

> ⚠️ **마이그레이션 주의사항**: Workspace Registry에서 UC Registry로 마이그레이션 시, Model Serving 엔드포인트의 참조도 함께 업데이트해야 합니다. 엔드포인트가 `models:/old_name/Production`을 참조하고 있다면, `models:/catalog.schema.new_name@champion`으로 변경합니다.

---

## Alias 기반 CI/CD 패턴

### Champion/Challenger 패턴 상세

| Alias | 역할 | 사용 시점 |
|-------|------|----------|
| **champion** | 현재 프로덕션에서 서빙되는 모델 | 검증 완료 후 |
| **challenger** | 다음 프로덕션 후보. 테스트/A/B 테스트 중 | 학습 완료 직후 |
| **baseline** | 성능 비교 기준이 되는 기본 모델 | 최초 배포 시 설정 |
| **rollback** | 즉시 롤백할 수 있는 이전 안정 버전 | champion 변경 직전 |

### CI/CD 자동화 파이프라인

```python
# 1단계: 모델 학습 및 challenger 등록
with mlflow.start_run():
    model = train_model(training_data)
    mlflow.sklearn.log_model(model, "model",
        registered_model_name="catalog.schema.fraud_detection")

# 새 버전에 challenger alias 부여
latest_version = client.get_latest_versions("catalog.schema.fraud_detection")[0].version
client.set_registered_model_alias(
    "catalog.schema.fraud_detection", "challenger", latest_version
)

# 2단계: 자동 검증 (Validation Job)
challenger = mlflow.pyfunc.load_model("models:/catalog.schema.fraud_detection@challenger")
champion = mlflow.pyfunc.load_model("models:/catalog.schema.fraud_detection@champion")

# 검증 데이터셋으로 성능 비교
challenger_metrics = evaluate_model(challenger, validation_data)
champion_metrics = evaluate_model(champion, validation_data)

# 3단계: 프로모션 판단
if challenger_metrics["f1_score"] > champion_metrics["f1_score"] * 1.02:  # 2% 이상 개선
    # 현재 champion을 rollback으로 보존
    champion_version = client.get_model_version_by_alias(
        "catalog.schema.fraud_detection", "champion"
    ).version
    client.set_registered_model_alias(
        "catalog.schema.fraud_detection", "rollback", champion_version
    )

    # challenger를 champion으로 승격
    client.set_registered_model_alias(
        "catalog.schema.fraud_detection", "champion", latest_version
    )
    print(f"✅ Promoted v{latest_version} to champion (F1: {challenger_metrics['f1_score']:.4f})")
else:
    print(f"❌ Challenger did not pass threshold (F1: {challenger_metrics['f1_score']:.4f} vs {champion_metrics['f1_score']:.4f})")
```

### A/B 테스트 패턴

```python
# Model Serving에서 트래픽 분할 A/B 테스트
w.serving_endpoints.update_config(
    name="fraud-detection",
    served_entities=[
        {
            "entity_name": "catalog.schema.fraud_detection",
            "entity_version": "champion",     # 기존 모델: 90% 트래픽
            "workload_size": "Small",
            "scale_to_zero_enabled": True,
            "traffic_percentage": 90
        },
        {
            "entity_name": "catalog.schema.fraud_detection",
            "entity_version": "challenger",   # 새 모델: 10% 트래픽
            "workload_size": "Small",
            "scale_to_zero_enabled": True,
            "traffic_percentage": 10
        }
    ]
)
```

---

## 모델 거버넌스 심화

### 접근 제어 레벨

```sql
-- 카탈로그 레벨: ML 카탈로그 전체에 대한 기본 권한
GRANT USE CATALOG ON CATALOG ml_catalog TO `all_data_scientists`;

-- 스키마 레벨: 모델 생성 권한
GRANT CREATE MODEL ON SCHEMA ml_catalog.production TO `senior_ml_engineers`;

-- 모델 레벨: 개별 모델 사용 권한
GRANT EXECUTE ON MODEL ml_catalog.production.fraud_detection TO `serving_service_principal`;

-- 읽기 전용 (메타데이터 조회만)
GRANT SELECT ON MODEL ml_catalog.production.fraud_detection TO `ml_auditors`;
```

### 감사 로그 활용

```sql
-- 모델 접근 감사 로그 확인
SELECT
    event_time,
    user_identity.email AS user,
    action_name,
    request_params.full_name_arg AS model_name,
    request_params.version AS model_version,
    response.status_code
FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND action_name IN ('getRegisteredModel', 'createModelVersion', 'setModelAlias')
  AND request_params.full_name_arg LIKE '%fraud_detection%'
ORDER BY event_time DESC
LIMIT 50;
```

### 모델 승인 워크플로우

태그를 활용한 수동 승인 프로세스를 구현할 수 있습니다.

```python
# 모델 승인 상태 관리
def request_approval(model_name, version):
    """모델 배포 승인 요청"""
    client.set_model_version_tag(model_name, version, "approval_status", "pending")
    client.set_model_version_tag(model_name, version, "requested_by", "kim@company.com")
    client.set_model_version_tag(model_name, version, "requested_at", str(datetime.now()))
    # Slack 알림 전송 등

def approve_model(model_name, version, approver):
    """모델 배포 승인"""
    client.set_model_version_tag(model_name, version, "approval_status", "approved")
    client.set_model_version_tag(model_name, version, "approved_by", approver)
    client.set_model_version_tag(model_name, version, "approved_at", str(datetime.now()))

def reject_model(model_name, version, reviewer, reason):
    """모델 배포 거부"""
    client.set_model_version_tag(model_name, version, "approval_status", "rejected")
    client.set_model_version_tag(model_name, version, "rejected_by", reviewer)
    client.set_model_version_tag(model_name, version, "rejection_reason", reason)
```

---

## 멀티 환경 프로모션 (Dev → Staging → Prod)

### 환경 분리 전략

대규모 조직에서는 모델을 여러 환경에 걸쳐 프로모션합니다.

| 전략 | 설명 | 적합한 경우 |
|------|------|-----------|
| **단일 Registry + Alias** | 하나의 UC Registry에서 Alias로 환경 구분 | 소규모~중규모 팀 |
| **환경별 카탈로그** | dev_catalog, staging_catalog, prod_catalog 분리 | 엔터프라이즈 (엄격한 격리) |
| **환경별 Workspace** | Dev/Staging/Prod Workspace + UC 공유 | 가장 엄격한 격리 |

### 환경별 카탈로그 패턴

```
dev_catalog.ml.fraud_detection      ← 개발 환경 (데이터 사이언티스트 자유 실험)
  └── v1, v2, v3... (실험 버전)

staging_catalog.ml.fraud_detection  ← 스테이징 (자동 테스트)
  └── v1 @challenger (검증 중)

prod_catalog.ml.fraud_detection     ← 프로덕션 (승인된 모델만)
  └── v1 @champion (서빙 중)
  └── v0 @rollback (롤백용)
```

### 환경 간 모델 프로모션

```python
# Dev → Staging 프로모션
def promote_to_staging(dev_model_name, dev_version, staging_model_name):
    """개발 환경에서 스테이징으로 모델 복사"""
    source_uri = f"models:/{dev_model_name}/{dev_version}"

    mlflow.set_registry_uri("databricks-uc")
    result = mlflow.register_model(source_uri, staging_model_name)

    client.set_registered_model_alias(
        staging_model_name, "challenger", result.version
    )

    # 태그에 출처 기록
    client.set_model_version_tag(
        staging_model_name, result.version,
        "source", f"{dev_model_name}/v{dev_version}"
    )
    return result.version

# Staging → Prod 프로모션 (승인 후)
def promote_to_prod(staging_model_name, staging_version, prod_model_name):
    """스테이징에서 프로덕션으로 모델 복사 (승인 필요)"""
    # 승인 상태 확인
    version_info = client.get_model_version(staging_model_name, staging_version)
    tags = {t.key: t.value for t in version_info.tags}

    if tags.get("approval_status") != "approved":
        raise ValueError("모델이 승인되지 않았습니다")

    source_uri = f"models:/{staging_model_name}/{staging_version}"
    result = mlflow.register_model(source_uri, prod_model_name)

    # 현재 champion을 rollback으로 보존
    try:
        current = client.get_model_version_by_alias(prod_model_name, "champion")
        client.set_registered_model_alias(prod_model_name, "rollback", current.version)
    except:
        pass

    # 새 버전을 champion으로 설정
    client.set_registered_model_alias(prod_model_name, "champion", result.version)
    return result.version
```

---

## 모델 리니지

Unity Catalog는 모델의 **전체 생애주기 리니지**를 자동으로 추적합니다.

### 리니지가 추적하는 정보

```
소스 데이터 (upstream)
  ├─ catalog.ecommerce.gold_orders (학습 데이터)
  ├─ catalog.ml.customer_features (Feature Table)
  └─ catalog.ml.product_features (Feature Table)
       │
       ▼
MLflow Experiment Run
  ├─ 하이퍼파라미터: max_depth=10, n_estimators=200
  ├─ 메트릭: accuracy=0.95, f1=0.93
  └─ 아티팩트: model.pkl, requirements.txt
       │
       ▼
Model Registry (catalog.ml.fraud_detection/v3 @champion)
       │
       ▼
Model Serving Endpoint (fraud-detection)
       │
       ▼
Inference Table (catalog.ml.fraud_detection_inference_logs)
```

### 리니지 SQL 조회

```sql
-- 특정 모델이 어떤 테이블에서 학습되었는지
SELECT
    source_table_full_name,
    source_type
FROM system.lineage.table_lineage
WHERE target_table_full_name = 'catalog.ml.fraud_detection'
  AND target_type = 'MODEL';

-- 특정 테이블이 변경되면 영향받는 모델
SELECT
    target_table_full_name AS affected_model,
    target_type
FROM system.lineage.table_lineage
WHERE source_table_full_name = 'catalog.ecommerce.gold_orders'
  AND target_type = 'MODEL';
```

---

## Edge Case와 주의사항

| 주의사항 | 설명 |
|---------|------|
| **모델 아티팩트 크기** | UC Registry에 등록 가능한 모델 크기에는 실질적 제한이 없지만, 수 GB 이상의 대형 모델은 등록/로드 시간이 오래 걸릴 수 있습니다 |
| **Alias 충돌** | 하나의 Alias는 하나의 버전에만 할당됩니다. 기존 Alias를 다른 버전으로 옮기면 이전 버전에서 자동 제거됩니다 |
| **삭제 정책** | 모델 버전을 삭제하면 복구할 수 없습니다. 중요한 버전에는 `do_not_delete` 태그를 붙여 실수를 방지하세요 |
| **서빙 참조 충돌** | Model Serving이 특정 버전을 직접 참조(by version number)하는 경우, 해당 버전을 삭제하면 서빙이 실패합니다. Alias 참조를 권장합니다 |
| **크로스 리전 모델 공유** | 다른 리전의 Workspace에서 모델을 사용하려면, Delta Sharing을 통해 모델을 공유해야 합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Model Registry** | 모델 버전을 중앙에서 관리하는 저장소입니다 |
| **Version** | 모델의 각 버전에 자동으로 번호가 부여됩니다 |
| **Alias** | 특정 버전에 부여하는 별칭입니다 (champion, challenger) |
| **Tags** | 모델에 메타데이터를 부여하여 관리를 용이하게 합니다 |
| **Champion/Challenger** | 프로덕션과 후보 모델을 Alias로 관리하는 CI/CD 패턴입니다 |
| **멀티 환경 프로모션** | Dev → Staging → Prod로 모델을 안전하게 승격합니다 |
| **모델 리니지** | 소스 데이터 → 모델 → 서빙까지 전체 흐름을 추적합니다 |
| **Unity Catalog 통합** | 모델에도 GRANT/REVOKE, 리니지, 감사가 적용됩니다 |

---

## 참고 링크

- [Databricks: Model Registry (UC)](https://docs.databricks.com/aws/en/mlflow/model-registry.html)
- [MLflow: Model Registry](https://mlflow.org/docs/latest/model-registry.html)
