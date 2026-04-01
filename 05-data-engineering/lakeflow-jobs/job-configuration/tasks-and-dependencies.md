# 태스크 유형과 의존성

## 왜 Job 구성이 중요한가?

데이터 파이프라인은 여러 단계(수집 → 변환 → 적재 → 검증)를 거칩니다. 각 단계를 **태스크(Task)** 로 정의하고, 이들의 실행 순서, 실패 시 대응 방식, 컴퓨트 자원 배분까지 체계적으로 설정해야 안정적인 운영이 가능합니다.

> 💡 **Job이란?**Databricks에서 하나 이상의 태스크를 묶어 실행하는 단위입니다. 단일 노트북 실행부터 수십 개 태스크가 의존 관계로 연결된 복잡한 워크플로까지 모두 Job으로 관리할 수 있습니다.

잘 구성된 Job은 다음과 같은 이점을 제공합니다:

| 이점 | 설명 |
|------|------|
| ** 안정성**| 재시도 정책과 타임아웃으로 일시적 장애를 자동 복구합니다 |
| ** 가시성**| 태스크 DAG를 통해 파이프라인 전체 흐름을 한눈에 파악할 수 있습니다 |
| ** 비용 효율**| Job Cluster와 Spot 인스턴스를 활용하여 비용을 최적화합니다 |
| ** 유지보수**| 파라미터화된 태스크로 코드 변경 없이 다양한 환경에서 실행할 수 있습니다 |

---

## 태스크 유형

Databricks Job은 다양한 유형의 태스크를 지원합니다. 각 태스크 유형은 특정 워크로드에 최적화되어 있습니다.

| 태스크 유형 | 설명 | 대표 사용 사례 |
|-------------|------|----------------|
| **Notebook**| Databricks 노트북을 실행합니다 | ETL 로직, 데이터 변환, ML 학습 |
| **Python Script**| `.py` 파일을 직접 실행합니다 | 범용 Python 스크립트, 배치 처리 |
| **SQL**| SQL 쿼리 또는 SQL 파일을 실행합니다 | 데이터 검증, 집계 테이블 갱신 |
| **dbt**| dbt 프로젝트의 태스크를 실행합니다 | dbt 모델 빌드, 테스트 |
| **Spark Submit**| Spark JAR 또는 Python을 spark-submit으로 실행합니다 | 레거시 Spark 애플리케이션 |
| **Pipeline (SDP)**| SDP(선언적 파이프라인)를 트리거합니다 | 스트리밍 수집, Medallion 파이프라인 |
| **JAR**| Java/Scala JAR 파일을 실행합니다 | Java 기반 ETL, 사내 라이브러리 |
| **Run Job**| 다른 Job을 트리거합니다 | 크로스 팀 워크플로 오케스트레이션 |
| **If/Else 조건**| 조건에 따라 분기 실행합니다 | 데이터 품질 기반 분기 처리 |
| **For Each**| 리스트의 각 항목에 대해 태스크를 반복 실행합니다 | 멀티 테넌트 처리, 파티션별 처리 |

---

## 태스크 의존성 설정

태스크 간 의존성을 설정하면 **DAG(Directed Acyclic Graph)** 형태로 실행 순서가 결정됩니다. Databricks는 세 가지 의존성 패턴을 지원합니다.

### 의존성 패턴

| 패턴 | 설명 | 사용 사례 |
|------|------|-----------|
| ** 선형(Sequential)**| A → B → C 순서대로 실행합니다 | 단계별 ETL 파이프라인 |
| ** 병렬(Fan-out/Fan-in)**| 여러 태스크를 동시에 실행한 후 합류합니다 | 독립적인 데이터 소스 동시 수집 |
| ** 조건부(Conditional)**| 선행 태스크 결과에 따라 분기합니다 | 데이터 검증 결과에 따른 처리 |

### Mermaid 다이어그램: 복잡한 태스크 DAG 예제

| Task | 유형 | 의존성 | 설명 |
|------|------|--------|------|
| ingest_orders | Pipeline Task | - | 주문 데이터를 수집합니다 |
| ingest_products | Pipeline Task | - | 상품 데이터를 수집합니다 |
| transform_orders | Notebook Task | ingest_orders, ingest_products | 데이터를 변환합니다 |
| validate_data | SQL Task | transform_orders | 데이터 품질을 검증합니다 |
| build_gold_tables | dbt Task | validate_data (성공 시) | Gold 테이블을 생성합니다 |
| send_alert | Notebook Task | validate_data (실패 시) | 알림을 발송합니다 |
| update_dashboard | SQL Task | build_gold_tables | 대시보드를 갱신합니다 |
| notify_success | Notebook Task | build_gold_tables | 성공 알림을 발송합니다 |

위 DAG에서:
- `ingest_orders`와 `ingest_products`는 ** 병렬** 로 실행됩니다
- 두 수집 태스크가 모두 완료되면 `transform_orders`가 실행됩니다 (Fan-in)
- `validate_data` 결과에 따라 ** 조건부 분기** 가 발생합니다
- `build_gold_tables` 완료 후 대시보드 갱신과 알림이 ** 병렬** 로 실행됩니다 (Fan-out)

---

## Job 생성 방법

### UI에서 생성

1. 좌측 메뉴 **Workflows**→ **Create Job** 클릭
2. Job 이름 입력
3. Task 추가 (노트북 경로, SQL 쿼리 등 지정)
4. Task 간 의존성 설정 (드래그&드롭 또는 `depends_on` 선택)
5. 컴퓨트 설정 (Serverless, Job Cluster, 또는 기존 All-Purpose Cluster)
6. 스케줄, 알림, 재시도 정책 설정

### Asset Bundles (IaC)

Infrastructure as Code 방식으로 Job을 정의하면 버전 관리와 환경별 배포가 쉬워집니다.

```yaml
resources:
  jobs:
    daily_pipeline:
      name: "daily-sales-pipeline"
      tags:
        team: "data-engineering"
        project: "sales"
        environment: "${bundle.target}"

      # Job 레벨 파라미터 정의
      parameters:
        - name: "target_date"
          default: ""
        - name: "environment"
          default: "production"

      tasks:
        - task_key: "ingest_orders"
          pipeline_task:
            pipeline_id: "${var.pipeline_id}"

        - task_key: "ingest_products"
          pipeline_task:
            pipeline_id: "${var.products_pipeline_id}"

        - task_key: "transform"
          depends_on:
            - task_key: "ingest_orders"
            - task_key: "ingest_products"
          notebook_task:
            notebook_path: "/Workspace/etl/transform"
            base_parameters:
              date: "{{job.parameters.target_date}}"
          new_cluster:
            spark_version: "15.4.x-scala2.12"
            num_workers: 4
            node_type_id: "i3.xlarge"
            aws_attributes:
              availability: "SPOT_WITH_FALLBACK"

        - task_key: "validate"
          depends_on:
            - task_key: "transform"
          sql_task:
            query:
              query_text: >
                SELECT COUNT(*) AS row_count
                FROM gold.daily_revenue
                WHERE sale_date = CURRENT_DATE()
            warehouse_id: "${var.warehouse_id}"

        - task_key: "notify"
          depends_on:
            - task_key: "validate"
          notebook_task:
            notebook_path: "/Workspace/etl/send_notification"

      # 알림 설정
      email_notifications:
        on_failure:
          - "data-team@company.com"
        on_success:
          - "data-team@company.com"

      webhook_notifications:
        on_failure:
          - id: "${var.slack_webhook_id}"

      # 스케줄 설정
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"
        timezone_id: "Asia/Seoul"
```

---
