# 접근 제어 설계

## 2. 접근 제어 설계

### 2.1 그룹 기반 권한 관리

**핵심 원칙**: 개별 사용자에게 직접 권한을 부여하지 않습니다. 반드시 **그룹**을 통해 관리합니다.

| 역할 그룹 | 카탈로그 권한 | 스키마 권한 | 테이블 권한 | 비고 |
|----------|-------------|-----------|-----------|------|
| `data-engineers` | USE CATALOG | USE SCHEMA, CREATE TABLE | SELECT, MODIFY | Bronze/Silver 쓰기 |
| `data-analysts` | USE CATALOG | USE SCHEMA | SELECT | Gold 읽기 전용 |
| `data-scientists` | USE CATALOG | USE SCHEMA, CREATE TABLE | SELECT, MODIFY | Silver 읽기, Sandbox 쓰기 |
| `platform-admins` | ALL PRIVILEGES | ALL PRIVILEGES | ALL PRIVILEGES | 최소 인원으로 제한 |
| `svc-prod-pipeline` | USE CATALOG | USE SCHEMA, CREATE TABLE | SELECT, MODIFY | Service Principal |

| 단계 | 원칙 | 설명 |
|------|------|------|
| 기본 | 모든 권한 거부 (Deny by Default) | 시작점 |
| 1단계 | 역할 기반 접근 제어 (RBAC) | 그룹별 필요 권한만 부여 |
| 2단계 | 데이터 수준 접근 제어 | 행/열 수준 필터링 |
| 3단계 | 정기 감사 | 분기별 권한 리뷰, 미사용 권한 회수 |

> 💡 **중요**: `USE CATALOG`과 `USE SCHEMA`는 해당 객체를 **탐색**할 수 있는 권한일 뿐, 데이터를 읽을 수 있는 `SELECT` 권한을 포함하지 않습니다. 반드시 별도로 부여해야 합니다.

### 2.3 Row Filter / Column Mask 적용 전략

민감한 데이터를 행 수준, 열 수준에서 제어합니다.

```sql
-- Row Filter: 사용자 그룹에 따라 볼 수 있는 행 제한
CREATE FUNCTION prod_sales.gold.region_filter(region STRING)
RETURN
  CASE
    WHEN IS_MEMBER('global-analysts') THEN TRUE
    WHEN IS_MEMBER('korea-team') AND region = 'KR' THEN TRUE
    WHEN IS_MEMBER('japan-team') AND region = 'JP' THEN TRUE
    ELSE FALSE
  END;

ALTER TABLE prod_sales.gold.sales_data
SET ROW FILTER prod_sales.gold.region_filter ON (region);

-- Column Mask: 민감한 컬럼 값을 마스킹
CREATE FUNCTION prod_sales.gold.mask_email(email STRING)
RETURN
  CASE
    WHEN IS_MEMBER('pii-authorized') THEN email
    ELSE CONCAT(LEFT(email, 2), '****@', SPLIT(email, '@')[1])
  END;

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN email SET MASK prod_sales.gold.mask_email;

-- Column Mask: 주민등록번호 마스킹
CREATE FUNCTION prod_sales.gold.mask_ssn(ssn STRING)
RETURN
  CASE
    WHEN IS_MEMBER('pii-authorized') THEN ssn
    ELSE CONCAT('***-**-', RIGHT(ssn, 4))
  END;

ALTER TABLE prod_sales.gold.customers
ALTER COLUMN ssn SET MASK prod_sales.gold.mask_ssn;
```

| PII/PHI 유형 | 마스킹 전략 | 예시 |
|-------------|-----------|------|
| 이메일 | 앞 2글자만 표시 | `si****@company.com` |
| 전화번호 | 뒷 4자리만 표시 | `***-****-5678` |
| 주민등록번호/SSN | 뒷 4자리만 표시 | `***-**-1234` |
| 신용카드 | 뒷 4자리만 표시 | `****-****-****-5678` |
| 주소 | 시/도만 표시 | `서울특별시 ***` |
| 진료기록(PHI) | 완전 숨김 | `[REDACTED]` |

### 2.4 Service Principal 관리 체계

| 용도 | Service Principal 명 | 권한 수준 | 비고 |
|------|---------------------|----------|------|
| 프로덕션 ETL | `svc-prod-etl` | Silver/Gold MODIFY | Jobs에서만 사용 |
| 프로덕션 ML | `svc-prod-ml` | Model Registry MANAGE | Serving 배포용 |
| CI/CD 파이프라인 | `svc-cicd` | Workspace 관리 | DAB 배포용 |
| 외부 연동 | `svc-external-api` | 특정 테이블 SELECT | API 접근용 |
| 모니터링 | `svc-monitoring` | 시스템 테이블 SELECT | 비용/감사 모니터링 |

```python
# Service Principal으로 프로덕션 Job 실행 (Terraform 예시)
resource "databricks_job" "daily_etl" {
  name = "daily-etl-pipeline"

  # Service Principal으로 실행 (개인 계정 아님)
  run_as {
    service_principal_name = "svc-prod-etl"
  }

  task {
    task_key = "etl_bronze_to_silver"
    pipeline_task {
      pipeline_id = databricks_pipeline.bronze_to_silver.id
    }
  }
}
```

> ⚠️ **핵심 원칙**: 프로덕션 워크로드는 반드시 **Service Principal** 으로 실행합니다. 개인 계정으로 실행하면 퇴사/이동 시 파이프라인이 중단될 수 있습니다.

---
