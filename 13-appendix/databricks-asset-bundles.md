# Declarative Automation Bundles (DABs)

## 개념

> **Databricks Declarative Automation Bundles (DABs)** 는 Databricks 리소스(Jobs, Pipelines, 대시보드 등)를 **코드로 정의하고 배포** 하는 IaC(Infrastructure as Code) 도구입니다. YAML 파일로 리소스를 선언하고, `databricks bundle` CLI 명령어로 배포합니다.

> **명칭 변경**: 2025년부터 **Databricks Asset Bundles (DABs)** 가 **Declarative Automation Bundles** 로 리브랜딩되었습니다. CLI 명령어(`databricks bundle`)와 설정 파일 구조는 동일합니다. 기존 문서나 블로그에서 "DABs"나 "Asset Bundles"로 언급되는 것은 같은 기능을 가리킵니다.

> **IaC(Infrastructure as Code)란?** 인프라(서버, 데이터베이스, 파이프라인 등)를 수동으로 설정하는 대신, **코드(설정 파일)로 정의** 하여 자동으로 배포하는 방식입니다. 코드이므로 Git으로 버전 관리할 수 있고, 코드 리뷰를 통해 변경 사항을 검토할 수 있으며, 여러 환경(dev, staging, prod)에 동일한 구성을 일관되게 배포할 수 있습니다.

---

## 1. 왜 DABs가 필요한가

### 수동 배포(UI 클릭)의 문제점

실제 엔터프라이즈 환경에서 UI로만 Databricks 리소스를 관리하면 다음과 같은 문제가 발생합니다.

| 문제 | 구체적인 현상 |
|------|-------------|
| **재현 불가** | "누가, 언제, 어떤 설정으로 만들었지?" 추적이 어렵습니다 |
| **환경 불일치** | 개발 환경과 프로덕션 환경의 설정이 미묘하게 달라 버그가 발생합니다 |
| **온보딩 비용** | 신규 팀원이 리소스 구조를 파악하려면 UI를 일일이 탐색해야 합니다 |
| **롤백 불가** | 잘못된 변경을 되돌리는 공식적인 방법이 없습니다 |
| **감사(Audit) 곤란** | 누가 어떤 Job 설정을 바꿨는지 추적하기 어렵습니다 |
| **코드 리뷰 불가** | 인프라 변경에 대한 팀 리뷰 프로세스가 없습니다 |

### IaC가 이 문제를 해결하는 방법

DABs는 모든 리소스 정의를 YAML 파일로 관리합니다. Git에 커밋되는 순간 다음 모든 것이 자동으로 확보됩니다.

- **버전 이력** — 언제, 누가, 무엇을 바꿨는지 `git log`로 확인
- **코드 리뷰** — PR(Pull Request)을 통해 변경 사항을 팀이 검토
- **재현 가능성** — 동일 YAML로 어느 환경에나 동일한 리소스 생성
- **자동화** — CI/CD 파이프라인에서 배포 자동 실행

### DABs vs Terraform — 무엇을 선택해야 하나

| 비교 항목 | DABs | Terraform |
|----------|------|-----------|
| **관리 대상** | Databricks 리소스 전용 | 모든 클라우드 인프라 |
| **설정 언어** | YAML | HCL (HashiCorp Configuration Language) |
| **학습 곡선** | 낮음 (Databricks 개념만 알면 됨) | 중간~높음 (Terraform 개념 추가 필요) |
| **상태 관리** | Databricks 플랫폼이 자동 관리 | `terraform.tfstate` 파일 직접 관리 필요 |
| **Databricks 신기능** | 즉시 반영 (네이티브) | Provider 업데이트 후 사용 가능 |
| **적합한 경우** | Databricks 리소스만 관리 | VPC, S3, IAM + Databricks 통합 관리 |
| **혼합 사용** | 가능 — 실제로 권장되는 패턴 | 가능 — Terraform으로 클라우드 인프라, DABs로 워크로드 |

> **권장 패턴**: 클라우드 인프라(VPC, 서브넷, IAM Role)는 Terraform으로, Databricks 워크로드(Jobs, Pipelines, Dashboards)는 DABs로 관리합니다.

---

## 2. 핵심 개념

### Bundle 구조

DABs 프로젝트의 핵심은 `databricks.yml`이라는 메인 설정 파일입니다. 이 파일을 중심으로 전체 번들이 구성됩니다.

```
my-project/
├── databricks.yml          # 메인 설정 (필수)
├── resources/
│   ├── jobs.yml            # Job 정의
│   ├── pipelines.yml       # DLT Pipeline 정의
│   └── dashboards.yml      # 대시보드 정의
├── src/
│   ├── notebooks/          # 노트북 소스
│   ├── pipelines/          # DLT 파이프라인 소스
│   └── python/             # Python 모듈
└── tests/                  # 테스트 코드
```

### 핵심 4대 구성 요소

| 구성 요소 | 설명 |
|----------|------|
| **bundle** | 번들 이름, 버전 등 메타데이터를 정의합니다 |
| **resources** | Jobs, Pipelines, Dashboards 등 Databricks 리소스를 선언합니다 |
| **targets** | dev, staging, prod 등 배포 환경별 설정을 정의합니다 |
| **variables** | 환경별로 달라지는 값(카탈로그명, 웨어하우스 ID 등)을 변수로 관리합니다 |

### Variables (변수)

변수는 두 가지 방식으로 값을 받습니다.

```yaml
variables:
  # 1. 기본값 지정
  catalog:
    description: "Unity Catalog 카탈로그 이름"
    default: "dev_catalog"

  # 2. 런타임에 CLI 인자로 전달 (기본값 없음)
  warehouse_id:
    description: "SQL Warehouse ID"
```

변수 참조는 `${var.변수명}` 형식을 사용합니다. Bundle 자체 메타데이터는 `${bundle.name}`, `${bundle.target}` 으로 참조할 수 있습니다.

```bash
# CLI에서 변수 오버라이드
databricks bundle deploy -t prod --var="catalog=prod_catalog"
```

---

## 3. 프로젝트 초기화

### databricks bundle init

Databricks CLI가 제공하는 공식 템플릿으로 프로젝트를 빠르게 시작할 수 있습니다.

```bash
# 대화형 템플릿 선택 (권장)
databricks bundle init

# 특정 템플릿 직접 지정
databricks bundle init default-python
databricks bundle init default-sql

# GitHub 저장소의 커스텀 템플릿 사용
databricks bundle init https://github.com/my-org/my-dab-template
```

실행 후 템플릿 선택 대화가 진행됩니다.

```
Template to use [default-python]:
> default-python
> default-sql
> dbt-sql
> mlops-stacks

Project name: my-etl-pipeline
Include a stub (sample) notebook [yes]: yes
```

### 공식 내장 템플릿

| 템플릿 | 적합한 경우 |
|-------|----------|
| `default-python` | Python 노트북/Job 위주의 일반적인 파이프라인 |
| `default-sql` | SQL 위주, DLT + SQL 작업 조합 |
| `dbt-sql` | dbt Core와 Databricks를 연동하는 프로젝트 |
| `mlops-stacks` | ML 모델 훈련~서빙 전체 MLOps 워크플로 |

---

## 4. 주요 리소스 유형

### 4-1. Jobs

```yaml
# resources/jobs.yml
resources:
  jobs:
    daily_etl:
      name: "daily-sales-etl-${bundle.target}"
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"  # 매일 새벽 2시 KST
        timezone_id: "Asia/Seoul"
      email_notifications:
        on_failure:
          - "data-team@company.com"

      tasks:
        - task_key: "ingest"
          pipeline_task:
            pipeline_id: "${resources.pipelines.medallion.id}"

        - task_key: "validate"
          depends_on:
            - task_key: "ingest"
          sql_task:
            query:
              query_text: >
                SELECT COUNT(*) AS cnt
                FROM ${var.catalog}.ecommerce.silver_orders
                WHERE order_date = CURRENT_DATE() - INTERVAL 1 DAY
            warehouse_id: "${var.warehouse_id}"

        - task_key: "notify"
          depends_on:
            - task_key: "validate"
          notebook_task:
            notebook_path: "src/notebooks/send_report.py"
          environment_key: "default"

      environments:
        - environment_key: "default"
          spec:
            client: "1"
            dependencies:
              - "requests>=2.28"
              - "pandas>=2.0"
```

**핵심 포인트**: `${resources.pipelines.medallion.id}` 처럼 다른 리소스를 참조할 수 있습니다. DABs가 배포 순서를 자동으로 결정합니다.

### 4-2. Pipelines (DLT, Delta Live Tables)

```yaml
# resources/pipelines.yml
resources:
  pipelines:
    medallion:
      name: "medallion-pipeline-${bundle.target}"
      catalog: "${var.catalog}"
      schema: "ecommerce"
      serverless: true
      continuous: false
      channel: "CURRENT"
      libraries:
        - notebook:
            path: "src/pipelines/bronze_layer.sql"
        - notebook:
            path: "src/pipelines/silver_layer.py"
        - notebook:
            path: "src/pipelines/gold_layer.py"
      notifications:
        - on_failure:
            - "data-team@company.com"
      configuration:
        source_path: "/Volumes/${var.catalog}/raw/landing"
```

### 4-3. Dashboards (AI/BI 대시보드)

대시보드도 YAML로 코드화하여 Git 관리가 가능합니다.

```yaml
# resources/dashboards.yml
resources:
  dashboards:
    sales_overview:
      display_name: "Sales Overview - ${bundle.target}"
      file_path: "src/dashboards/sales_overview.lvdash.json"
      warehouse_id: "${var.warehouse_id}"
      embed_credentials: false
```

대시보드 파일(`*.lvdash.json`)은 Databricks UI에서 "Export as code" 기능으로 추출합니다.

### 4-4. Model Serving Endpoints

```yaml
# resources/model_serving.yml
resources:
  model_serving_endpoints:
    fraud_detector:
      name: "fraud-detector-${bundle.target}"
      config:
        served_models:
          - model_name: "${var.catalog}.ml.fraud_model"
            model_version: "1"
            workload_size: "Small"
            scale_to_zero_enabled: true
      rate_limits:
        - calls: 1000
          renewal_period: "minute"
```

### 4-5. Quality Monitors & Alerts

```yaml
# resources/quality.yml
resources:
  quality_monitors:
    orders_monitor:
      table_name: "${var.catalog}.ecommerce.silver_orders"
      assets_dir: "/Shared/monitors/orders"
      output_schema_name: "${var.catalog}.monitoring"
      inference_log:
        granularities: ["1 day"]
        model_id_col: "model_version"
        prediction_col: "prediction"
        timestamp_col: "scored_at"

  alerts:
    pipeline_failure_alert:
      display_name: "Pipeline Failure - ${bundle.target}"
      condition:
        op: "GREATER_THAN"
        operand:
          column:
            name: "failed_count"
        threshold:
          value:
            double_value: 0
      query_id: "${var.failure_query_id}"
      seconds_to_retrigger: 3600
```

---

## 5. 환경별 배포 (Targets)

### dev / staging / prod 타겟 설정

```yaml
# databricks.yml
bundle:
  name: "ecommerce-data-pipeline"

variables:
  catalog:
    default: "dev_catalog"
  warehouse_id:
    default: ""
  notification_email:
    default: "dev-team@company.com"

include:
  - "resources/*.yml"

targets:
  # 개발 환경 — 기본 타겟
  dev:
    mode: development       # 리소스 이름에 [dev 사용자명] 접두사 자동 추가
    default: true
    workspace:
      host: "https://dbc-dev.cloud.databricks.com"
    variables:
      catalog: "dev_catalog"
      warehouse_id: "aaa111bbb"

  # 스테이징 환경
  staging:
    workspace:
      host: "https://dbc-staging.cloud.databricks.com"
    variables:
      catalog: "staging_catalog"
      warehouse_id: "ccc222ddd"
      notification_email: "staging-alerts@company.com"

  # 프로덕션 환경
  prod:
    mode: production        # 실수로 destroy 시 확인 요구, run_as 강제
    workspace:
      host: "https://dbc-prod.cloud.databricks.com"
    variables:
      catalog: "prod_catalog"
      warehouse_id: "eee333fff"
      notification_email: "oncall@company.com"
    run_as:
      service_principal_name: "sp-prod-data-pipeline"
```

### mode: development vs mode: production

| mode | 동작 |
|------|------|
| `development` | 리소스 이름에 `[dev 사용자명]` 접두사가 자동으로 붙어 개인 개발 공간 격리 |
| `production` | `run_as` 설정 필수, `bundle destroy` 시 확인 프롬프트 표시 |
| (미설정) | 기본 동작, 이름 변경 없음 |

### 권한(Permissions) 설정

```yaml
# resources/jobs.yml — 리소스별 권한
resources:
  jobs:
    daily_etl:
      name: "daily-sales-etl-${bundle.target}"
      permissions:
        - level: "CAN_MANAGE"
          group_name: "data-engineers"
        - level: "CAN_VIEW"
          group_name: "data-analysts"
        - level: "IS_OWNER"
          service_principal_name: "sp-prod-data-pipeline"
```

---

## 6. 배포 워크플로

### 배포 3단계

```bash
# 1단계: 검증 — 실제 배포 없이 설정 오류 사전 확인 (Dry Run)
databricks bundle validate

# 2단계: 배포 — 리소스 생성/업데이트
databricks bundle deploy -t staging

# 3단계: 실행 — 배포된 Job/Pipeline 즉시 실행
databricks bundle run daily_etl -t staging

# 리소스 삭제 (개발 환경 정리)
databricks bundle destroy -t dev
```

### validate 출력 예시

```
Name: ecommerce-data-pipeline
Target: staging
Workspace:
  Host: https://dbc-staging.cloud.databricks.com
  User: data-engineer@company.com
  Path: /Workspace/Users/data-engineer@company.com/.bundle/ecommerce-data-pipeline/staging

Validation OK!
```

### GitHub Actions CI/CD 연동

```yaml
# .github/workflows/deploy.yml
name: Deploy Data Pipeline

on:
  push:
    branches: [main]        # main 머지 시 prod 자동 배포
  pull_request:
    branches: [main]        # PR 생성 시 staging 검증

jobs:
  validate:
    name: Validate Bundle
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - name: Install Databricks CLI
        uses: databricks/setup-cli@main

      - name: Validate (staging)
        run: databricks bundle validate -t staging
        env:
          DATABRICKS_HOST: ${{ secrets.STAGING_DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.STAGING_DATABRICKS_TOKEN }}

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production   # GitHub Environment 보호 규칙 적용
    steps:
      - uses: actions/checkout@v4

      - name: Install Databricks CLI
        uses: databricks/setup-cli@main

      - name: Deploy to Production
        run: |
          databricks bundle validate -t prod
          databricks bundle deploy -t prod
        env:
          DATABRICKS_HOST: ${{ secrets.PROD_DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.PROD_DATABRICKS_TOKEN }}
```

> **인증 방법**: 프로덕션 배포에는 Personal Access Token 대신 **Service Principal + OAuth M2M** 인증을 사용합니다. GitHub Secrets에 `DATABRICKS_CLIENT_ID`, `DATABRICKS_CLIENT_SECRET`을 저장하고 `DATABRICKS_TOKEN` 대신 사용합니다.

---

## 7. 실전 패턴

### 7-1. 모노레포(Monorepo) 구조

여러 팀/도메인의 파이프라인을 하나의 Git 저장소에서 관리하는 패턴입니다.

```
data-platform/
├── domains/
│   ├── ecommerce/
│   │   ├── databricks.yml
│   │   ├── resources/
│   │   └── src/
│   ├── marketing/
│   │   ├── databricks.yml
│   │   ├── resources/
│   │   └── src/
│   └── finance/
│       ├── databricks.yml
│       └── ...
└── shared/
    └── libraries/          # 공통 유틸리티 모듈
```

각 도메인 디렉토리에서 독립적으로 `databricks bundle deploy`를 실행합니다. GitHub Actions에서는 `paths` 필터로 변경된 도메인만 선택적으로 배포합니다.

### 7-2. 다중 Job 오케스트레이션

한 번들 내에서 여러 Job이 서로를 참조할 수 있습니다.

```yaml
resources:
  jobs:
    ingestion_job:
      name: "ingestion-${bundle.target}"
      tasks:
        - task_key: "run_pipeline"
          pipeline_task:
            pipeline_id: "${resources.pipelines.raw_ingest.id}"

    transformation_job:
      name: "transformation-${bundle.target}"
      # 다른 Job 완료 후 트리거하는 패턴 (run_job_task)
      tasks:
        - task_key: "transform"
          run_job_task:
            job_id: "${resources.jobs.ingestion_job.id}"
```

### 7-3. 대시보드 코드화

```bash
# UI에서 대시보드를 JSON으로 추출
databricks dashboards export <dashboard-id> > src/dashboards/sales_overview.lvdash.json

# 번들에 포함하여 배포
databricks bundle deploy -t prod
```

이렇게 하면 대시보드도 Git 이력 관리가 가능하고, 환경 간 일관된 대시보드 배포가 가능합니다.

### 7-4. include로 설정 분리

큰 프로젝트에서는 리소스 파일을 분리하여 가독성을 높입니다.

```yaml
# databricks.yml
include:
  - "resources/jobs/*.yml"
  - "resources/pipelines/*.yml"
  - "resources/dashboards/*.yml"
```

---

## 8. 장단점과 트레이드오프

### DABs vs 수동 배포(UI)

| | DABs | 수동 배포 |
|---|------|---------|
| **초기 설정** | YAML 학습 필요, 초기 비용 있음 | 즉시 시작 가능 |
| **반복 배포** | 명령어 한 줄 | 매번 UI 클릭 반복 |
| **환경 일관성** | 완벽히 보장 | 사람에 의존, 오류 발생 가능 |
| **감사/추적** | Git 이력으로 완전 추적 | 불가 |
| **팀 협업** | PR 리뷰 기반 변경 관리 | 변경 충돌 발생 가능 |
| **적합한 규모** | 중규모 이상 팀 | 개인/소규모 PoC |

### DABs vs Terraform

| | DABs | Terraform |
|---|------|-----------|
| **Databricks 신기능** | 즉시 지원 | Provider 릴리즈 대기 |
| **클라우드 인프라** | 불가 (Databricks 전용) | AWS/Azure/GCP 모두 관리 |
| **상태 파일** | 불필요 (플랫폼 관리) | `.tfstate` 직접 관리, 협업 시 원격 상태 설정 필요 |
| **드리프트 감지** | 제한적 | 강력한 `terraform plan` |
| **생태계** | Databricks 내부 | 광범위한 커뮤니티 모듈 |

### 학습 곡선

```
초급: databricks.yml 기본 구조 이해 (1-2일)
  → Job/Pipeline 단순 배포

중급: 멀티 환경(targets), 변수 관리, 권한 설정 (1주)
  → dev/staging/prod 분리 운영

고급: CI/CD 연동, 모노레포, 커스텀 템플릿 (2-4주)
  → 팀 전체 배포 플랫폼 구축
```

---

## 9. 베스트 프랙티스와 흔한 실수

### 베스트 프랙티스

**1. 항상 validate 먼저 실행**

```bash
# 배포 전 반드시 검증
databricks bundle validate -t prod
databricks bundle deploy -t prod
```

**2. Service Principal로 CI/CD 인증**

Personal Access Token은 사용자와 연결되어 있어 퇴직 시 파이프라인이 중단됩니다. CI/CD에서는 반드시 Service Principal을 사용합니다.

**3. run_as를 prod에 명시**

```yaml
targets:
  prod:
    mode: production
    run_as:
      service_principal_name: "sp-prod-pipeline"  # 사용자 개인 계정이 아닌 SP 사용
```

**4. 변수 기본값은 dev 기준으로 설정**

개발자가 `-t` 옵션 없이 명령을 실행하면 기본 타겟(dev)이 사용됩니다. 실수로 prod에 영향을 주는 것을 방지합니다.

**5. 리소스 이름에 `${bundle.target}` 포함**

```yaml
name: "daily-etl-${bundle.target}"   # daily-etl-dev, daily-etl-prod 구분
```

같은 Workspace를 여러 환경이 공유할 때 이름 충돌을 방지합니다.

### 흔한 실수

| 실수 | 원인 | 해결 방법 |
|------|------|---------|
| 배포 후 리소스가 사라짐 | `mode: development`에서 다른 사용자가 deploy 실행 — 이름에 사용자명이 붙어 별도 리소스로 생성됨 | dev는 개인별 독립 공간임을 이해, 공유 dev는 mode 생략 |
| prod에 dev 카탈로그 데이터가 들어감 | 변수 기본값이 dev 카탈로그인 상태로 prod 배포 | `-t prod` 지정 시 변수 재확인, validate에서 변수 출력 확인 |
| CI/CD 인증 실패 | Token 만료 또는 Service Principal 권한 부족 | SP에 Workspace 접근 + 번들 리소스 CAN_MANAGE 권한 부여 |
| YAML 들여쓰기 오류 | 탭/스페이스 혼용 | 에디터에서 YAML 린터 활성화, `bundle validate`로 사전 확인 |
| 리소스 삭제 안 됨 | `bundle destroy`를 실행했지만 외부에서 수동으로 생성된 리소스는 삭제 불가 | DABs 외부에서 생성한 리소스는 DABs가 관리하지 않음을 이해 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **databricks.yml** | 번들의 모든 것을 정의하는 메인 설정 파일입니다 |
| **resources** | Jobs, Pipelines, Dashboards 등 Databricks 리소스를 YAML로 선언합니다 |
| **targets** | dev, staging, prod 등 환경별 설정을 분리합니다 |
| **variables** | 환경별로 다른 값(카탈로그, 엔드포인트 등)을 `${var.이름}` 으로 참조합니다 |
| **bundle validate** | 실제 배포 없이 설정 오류를 사전에 확인합니다 |
| **bundle deploy** | 정의된 리소스를 대상 환경에 생성/업데이트합니다 |
| **CI/CD** | GitHub Actions 등과 연동하여 PR 기반 자동 배포 파이프라인을 구축합니다 |

---

## 참고 링크

- [Databricks: Declarative Automation Bundles 공식 문서](https://docs.databricks.com/aws/en/dev-tools/bundles/)
- [Databricks: Bundle configuration reference (전체 YAML 스키마)](https://docs.databricks.com/aws/en/dev-tools/bundles/settings.html)
- [Databricks: Bundle resources (리소스 유형별 설정)](https://docs.databricks.com/aws/en/dev-tools/bundles/resources.html)
- [Databricks: Bundle templates](https://docs.databricks.com/aws/en/dev-tools/bundles/templates.html)
- [Databricks: CI/CD with DABs (GitHub Actions)](https://docs.databricks.com/aws/en/dev-tools/bundles/ci-cd.html)
- [Azure Databricks: Declarative Automation Bundles](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/bundles/)
- [Databricks: DABs vs Terraform — When to use each](https://docs.databricks.com/aws/en/dev-tools/index.html)
