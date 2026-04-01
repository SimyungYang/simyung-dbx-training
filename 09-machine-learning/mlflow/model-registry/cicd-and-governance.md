# CI/CD 패턴과 거버넌스

## Alias 기반 CI/CD 패턴

### Champion/Challenger 패턴 상세

| Alias | 역할 | 사용 시점 |
|-------|------|----------|
| **champion**| 현재 프로덕션에서 서빙되는 모델 | 검증 완료 후 |
| **challenger**| 다음 프로덕션 후보. 테스트/A/B 테스트 중 | 학습 완료 직후 |
| **baseline**| 성능 비교 기준이 되는 기본 모델 | 최초 배포 시 설정 |
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
| **단일 Registry + Alias**| 하나의 UC Registry에서 Alias로 환경 구분 | 소규모~중규모 팀 |
| ** 환경별 카탈로그**| dev_catalog, staging_catalog, prod_catalog 분리 | 엔터프라이즈 (엄격한 격리) |
| ** 환경별 Workspace**| Dev/Staging/Prod Workspace + UC 공유 | 가장 엄격한 격리 |

### 환경별 카탈로그 패턴

| 환경 | 모델 경로 | 설명 |
|------|---------|------|
| ** 개발**| dev_catalog.ml.fraud_detection | 데이터 사이언티스트 자유 실험 (v1, v2, v3...) |
| ** 스테이징**| staging_catalog.ml.fraud_detection | 통합 테스트 (검증된 버전만 프로모션) |
| ** 프로덕션**| prod_catalog.ml.fraud_detection | 실서비스 (v1: champion, v2: challenger - A/B 테스트) |

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

Unity Catalog는 모델의 ** 전체 생애주기 리니지**를 자동으로 추적합니다.

### 리니지가 추적하는 정보

| 방향 | 구성 요소 | 설명 |
|------|----------|------|
| **소스 (upstream)** | catalog.ecommerce.gold_orders | 학습 데이터 |
| | catalog.ml.customer_features | 피처 테이블 |
| | catalog.ml.fraud_detection | 모델 (v1: deprecated, v2: champion, v3: challenger) |
| **소비자 (downstream)** | Serving Endpoint: fraud-detection-endpoint | 모델 서빙 |
| | Dashboard: fraud_monitoring_daily | 모니터링 대시보드 |
| | Job: daily_fraud_scoring_pipeline | 일일 스코어링 파이프라인 |

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
| **Alias 충돌**| 하나의 Alias는 하나의 버전에만 할당됩니다. 기존 Alias를 다른 버전으로 옮기면 이전 버전에서 자동 제거됩니다 |
| ** 삭제 정책**| 모델 버전을 삭제하면 복구할 수 없습니다. 중요한 버전에는 `do_not_delete` 태그를 붙여 실수를 방지하세요 |
| ** 서빙 참조 충돌**| Model Serving이 특정 버전을 직접 참조(by version number)하는 경우, 해당 버전을 삭제하면 서빙이 실패합니다. Alias 참조를 권장합니다 |
| ** 크로스 리전 모델 공유**| 다른 리전의 Workspace에서 모델을 사용하려면, Delta Sharing을 통해 모델을 공유해야 합니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Model Registry**| 모델 버전을 중앙에서 관리하는 저장소입니다 |
| **Version**| 모델의 각 버전에 자동으로 번호가 부여됩니다 |
| **Alias**| 특정 버전에 부여하는 별칭입니다 (champion, challenger) |
| **Tags**| 모델에 메타데이터를 부여하여 관리를 용이하게 합니다 |
| **Champion/Challenger**| 프로덕션과 후보 모델을 Alias로 관리하는 CI/CD 패턴입니다 |
| ** 멀티 환경 프로모션**| Dev → Staging → Prod로 모델을 안전하게 승격합니다 |
| ** 모델 리니지**| 소스 데이터 → 모델 → 서빙까지 전체 흐름을 추적합니다 |
| **Unity Catalog 통합** | 모델에도 GRANT/REVOKE, 리니지, 감사가 적용됩니다 |

---

## 참고 링크

- [Databricks: Model Registry (UC)](https://docs.databricks.com/aws/en/mlflow/model-registry.html)
- [MLflow: Model Registry](https://mlflow.org/docs/latest/model-registry.html)
