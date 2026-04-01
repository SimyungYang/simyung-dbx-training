# Declarative Automation Bundles

## 개념

> 💡 **Databricks Declarative Automation Bundles**(구 Databricks Declarative Automation Bundles / DABs)는 Databricks 리소스(Jobs, Pipelines, 대시보드 등)를 ** 코드로 정의하고 배포** 하는 IaC(Infrastructure as Code) 도구입니다. YAML 파일로 리소스를 선언하고, `databricks bundle` CLI 명령어로 배포합니다.

> 🆕 ** 명칭 변경**: 2025년부터 **Databricks Declarative Automation Bundles (DABs)** 가 **Declarative Automation Bundles** 로 리브랜딩되었습니다. CLI 명령어(`databricks bundle`)와 설정 파일 구조는 동일합니다. 기존 문서나 블로그에서 "DABs"나 "Declarative Automation Bundles"로 언급되는 것은 같은 기능을 가리킵니다.

> 💡 **IaC(Infrastructure as Code)란?** 인프라(서버, 데이터베이스, 파이프라인 등)를 수동으로 설정하는 대신, ** 코드(설정 파일)로 정의** 하여 자동으로 배포하는 방식입니다. 코드이므로 Git으로 버전 관리할 수 있고, 코드 리뷰를 통해 변경 사항을 검토할 수 있으며, 여러 환경(dev, staging, prod)에 동일한 구성을 일관되게 배포할 수 있습니다.

---

## 왜 Declarative Automation Bundles가 필요한가요?

### UI에서 직접 생성하는 방식의 문제

| 문제 | 설명 |
|------|------|
| ** 재현 불가** | "누가, 언제, 어떤 설정으로 만들었지?" 추적이 어렵습니다 |
| ** 환경 불일치** | 개발 환경과 프로덕션 환경의 설정이 다를 수 있습니다 |
| ** 수동 배포** | 매번 UI에서 클릭하여 수동으로 배포해야 합니다 |
| ** 코드 리뷰 불가** | 인프라 변경에 대한 리뷰가 어렵습니다 |

### Declarative Automation Bundles의 해결

| 장점 | 설명 |
|------|------|
| ** 코드로 관리** | 모든 리소스 정의가 YAML 파일로 관리됩니다 |
| **Git 버전 관리**| 변경 이력을 추적하고, 이전 버전으로 롤백할 수 있습니다 |
| ** 멀티 환경 배포**| dev, staging, prod에 일관된 구성을 배포합니다 |
| **CI/CD 통합**| GitHub Actions, Azure DevOps 등과 연동 가능합니다 |
| ** 코드 리뷰** | PR(Pull Request)을 통해 변경 사항을 리뷰합니다 |

---

## 프로젝트 구조

| 파일/디렉토리 | 설명 |
|-------------|------|
| `databricks.yml` | 메인 설정 파일 (프로젝트 루트) |
| `resources/jobs.yml` | Job 정의 |
| `resources/pipelines.yml` | Pipeline 정의 |
| `resources/dashboards.yml` | 대시보드 정의 |
| `src/` | 소스 코드 (노트북, Python, SQL) |
| `tests/` | 테스트 코드 |
| `fixtures/` | 테스트 데이터 |

---

## 주요 설정 파일

### databricks.yml (메인 설정)

```yaml
bundle:
  name: "ecommerce-data-pipeline"

# 변수 정의 (환경별로 다른 값 사용)
variables:
  catalog:
    default: "dev"
  warehouse_id:
    default: "abc123"

# 리소스 정의 파일 포함
include:
  - "resources/*.yml"

# 환경별 설정
targets:
  dev:
    mode: development
    default: true
    workspace:
      host: "https://dbc-dev.cloud.databricks.com"
    variables:
      catalog: "dev"

  staging:
    workspace:
      host: "https://dbc-staging.cloud.databricks.com"
    variables:
      catalog: "staging"

  prod:
    mode: production
    workspace:
      host: "https://dbc-prod.cloud.databricks.com"
    variables:
      catalog: "production"
    run_as:
      service_principal_name: "sp-prod-pipeline"
```

### resources/jobs.yml (Job 정의)

```yaml
resources:
  jobs:
    daily_etl:
      name: "daily-sales-etl-${bundle.target}"
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"  # 매일 새벽 2시
        timezone_id: "Asia/Seoul"
      email_notifications:
        on_failure:
          - "team@company.com"
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
```

### resources/pipelines.yml (Pipeline 정의)

```yaml
resources:
  pipelines:
    medallion:
      name: "medallion-pipeline-${bundle.target}"
      catalog: "${var.catalog}"
      schema: "ecommerce"
      serverless: true
      continuous: false
      libraries:
        - notebook:
            path: "src/pipelines/medallion_pipeline.sql"
      notifications:
        - on_failure:
            - "team@company.com"
```

---

## CLI 명령어

### 프로젝트 초기화

```bash
# 새 프로젝트 생성 (템플릿 선택)
databricks bundle init

# 기존 디렉토리에서 초기화
databricks bundle init --existing-project
```

### 배포

```bash
# 개발 환경에 배포 (기본 target)
databricks bundle deploy

# 특정 환경에 배포
databricks bundle deploy -t staging
databricks bundle deploy -t prod

# 배포 전 검증만 수행 (Dry Run)
databricks bundle validate
```

### 실행

```bash
# Job 실행
databricks bundle run daily_etl

# 특정 환경의 Job 실행
databricks bundle run daily_etl -t prod

# Pipeline 실행
databricks bundle run medallion
```

### 정리

```bash
# 배포된 리소스 삭제
databricks bundle destroy -t dev
```

---

## CI/CD 통합 예시

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy Data Pipeline

on:
  push:
    branches: [main]

jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Databricks CLI
        uses: databricks/setup-cli@main

      - name: Validate Bundle
        run: databricks bundle validate -t prod
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}

      - name: Deploy to Production
        run: databricks bundle deploy -t prod
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
```

---

## Declarative Automation Bundles vs Terraform 비교

| 비교 | Declarative Automation Bundles | Terraform |
|------|--------------|-----------|
| **대상**| Databricks 리소스 전용 | 모든 클라우드 인프라 |
| ** 언어**| YAML | HCL |
| ** 학습 곡선**| 낮음 | 중간~높음 |
| ** 상태 관리**| Databricks가 관리 | Terraform State 파일 관리 필요 |
| **Databricks 통합**| 네이티브 (최적화) | Provider 필요 |
| ** 적합한 경우**| Databricks 리소스만 관리 | 클라우드 인프라 + Databricks 통합 관리 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Declarative Automation Bundles**| YAML로 Databricks 리소스를 정의하고 CLI로 배포하는 IaC 도구입니다 |
| **targets**| dev, staging, prod 등 환경별 설정을 분리합니다 |
| **variables**| 환경별로 다른 값(카탈로그, 엔드포인트 등)을 변수로 관리합니다 |
| **databricks bundle deploy**| 정의된 리소스를 대상 환경에 배포합니다 |
| **CI/CD** | GitHub Actions 등과 연동하여 자동 배포 파이프라인을 구축합니다 |

---

## 참고 링크

- [Databricks: Declarative Automation Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)
- [Databricks: Bundle configuration reference](https://docs.databricks.com/aws/en/dev-tools/bundles/settings.html)
- [Azure Databricks: Declarative Automation Bundles](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/bundles/)
