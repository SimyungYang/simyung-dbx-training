# 권한 관리와 Model Serving 연동

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

Databricks에는 두 가지 Model Registry가 존재합니다. 현재 **UC Model Registry** 가 표준이며, Workspace Registry는 레거시로 분류됩니다.

| 항목 | Workspace Registry (레거시) | UC Model Registry (현재 표준) |
|------|---------------------------|-------------------------------|
| **네임스페이스** | `model_name` (flat) | `catalog.schema.model_name` (3-Level) |
| ** 접근 제어** | Workspace ACL (제한적) | UC GRANT/REVOKE (세밀한 권한) |
| ** 크로스 워크스페이스** | 불가 | 가능 (UC 공유) |
| ** 리니지** | 제한적 | 자동 리니지 (소스 데이터 → 모델) |
| ** 감사 로그** | Workspace 로그 | UC 시스템 테이블 (system.access.audit) |
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

> ⚠️ ** 마이그레이션 주의사항**: Workspace Registry에서 UC Registry로 마이그레이션 시, Model Serving 엔드포인트의 참조도 함께 업데이트해야 합니다. 엔드포인트가 `models:/old_name/Production`을 참조하고 있다면, `models:/catalog.schema.new_name@champion`으로 변경합니다.

---
