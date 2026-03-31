# Table of contents

## Databricks 플랫폼

* [Databricks란?](02-databricks-overview/what-is-databricks.md)
* [Databricks 아키텍처](02-databricks-overview/databricks-architecture.md)
* [Workspace UI 둘러보기](02-databricks-overview/workspace-ui-tour.md)
* [Notebook 사용법](02-databricks-overview/notebooks-basics.md)
  * [Notebook 고급 기능](02-databricks-overview/notebooks-advanced.md)
* [무료 체험 시작하기](02-databricks-overview/getting-started-trial.md)

## 레이크하우스 아키텍처

* [레이크하우스란?](03-lakehouse-architecture/what-is-lakehouse.md)
* [Delta Lake 핵심](03-lakehouse-architecture/delta-lake-fundamentals.md)
  * [관리형·외부·외래 테이블](03-lakehouse-architecture/managed-vs-external-tables.md)
  * [타임 트래블](03-lakehouse-architecture/time-travel.md)
  * [스키마 진화](03-lakehouse-architecture/schema-evolution.md)
* [Medallion 아키텍처](03-lakehouse-architecture/medallion-architecture.md)
* [Delta Lake 실전 운영](03-lakehouse-architecture/delta-lake-operations.md)
  * [Liquid Clustering](03-lakehouse-architecture/liquid-clustering.md)
  * [VACUUM과 OPTIMIZE](03-lakehouse-architecture/vacuum-and-optimize.md)
  * [Deletion Vectors](03-lakehouse-architecture/deletion-vectors.md)
  * [Predictive Optimization](03-lakehouse-architecture/predictive-optimization.md)
* [Delta Lake & Iceberg](03-lakehouse-architecture/delta-and-iceberg.md)
  * [UniForm (External Iceberg Reads)](03-lakehouse-architecture/uniform.md)

## 컴퓨트

* [Apache Spark 기초](04-compute-and-workspace/spark-basics.md)
* [클러스터 종류](04-compute-and-workspace/cluster-types.md)
* [클러스터 설정](04-compute-and-workspace/cluster-configuration.md)
  * [클러스터 정책](04-compute-and-workspace/cluster-policies.md)
  * [Instance Pools](04-compute-and-workspace/instance-pools.md)
  * [Photon 엔진](04-compute-and-workspace/photon-engine.md)
  * [Spot 인스턴스](04-compute-and-workspace/spot-instances.md)
* [SQL Warehouse](04-compute-and-workspace/sql-warehouse.md)
* [Serverless 컴퓨트](04-compute-and-workspace/serverless-compute.md)

## 데이터 엔지니어링

* [데이터 엔지니어링 개요](05-data-engineering/data-engineering-overview.md)
* [수집 방법 선택 가이드](05-data-engineering/choosing-ingestion-method.md)
* [Auto Loader](05-data-engineering/auto-loader/README.md)
  * [Auto Loader란?](05-data-engineering/auto-loader/what-is-auto-loader.md)
  * [File Notification vs Directory Listing](05-data-engineering/auto-loader/file-notification-vs-directory-listing.md)
  * [스키마 추론과 진화](05-data-engineering/auto-loader/schema-inference.md)
  * [주요 옵션](05-data-engineering/auto-loader/auto-loader-options.md)
  * [Auto Loader 실습](05-data-engineering/auto-loader/auto-loader-hands-on.md)
* [Structured Streaming](05-data-engineering/structured-streaming.md)
* [Spark Declarative Pipelines (SDP)](05-data-engineering/spark-declarative-pipelines/README.md)
  * [SDP란?](05-data-engineering/spark-declarative-pipelines/what-is-sdp.md)
  * [Streaming Tables & Materialized Views](05-data-engineering/spark-declarative-pipelines/streaming-tables-and-mvs.md)
  * [Expectations (데이터 품질)](05-data-engineering/spark-declarative-pipelines/expectations.md)
  * [파이프라인 설정](05-data-engineering/spark-declarative-pipelines/pipeline-configuration.md)
  * [이벤트 로그](05-data-engineering/spark-declarative-pipelines/event-log.md)
  * [CDC와 SCD 처리](05-data-engineering/spark-declarative-pipelines/cdc-and-scd.md)
  * [SDP 실습](05-data-engineering/spark-declarative-pipelines/sdp-hands-on.md)
* [Lakeflow Connect](05-data-engineering/lakeflow-connect/README.md)
  * [Lakeflow Connect란?](05-data-engineering/lakeflow-connect/what-is-lakeflow-connect.md)
  * [수집 파이프라인 구성](05-data-engineering/lakeflow-connect/ingestion-pipeline-setup.md)
  * [Lakeflow Connect 실습](05-data-engineering/lakeflow-connect/lakeflow-connect-hands-on.md)
* [Lakeflow Jobs](05-data-engineering/lakeflow-jobs/README.md)
  * [Lakeflow Jobs란?](05-data-engineering/lakeflow-jobs/what-is-lakeflow-jobs.md)
  * [작업 구성](05-data-engineering/lakeflow-jobs/job-configuration.md)
    * [태스크 유형](05-data-engineering/lakeflow-jobs/task-types.md)
    * [조건부 태스크와 흐름 제어](05-data-engineering/lakeflow-jobs/conditional-tasks.md)
  * [스케줄링과 트리거](05-data-engineering/lakeflow-jobs/scheduling-and-triggers.md)
  * [모니터링과 알림](05-data-engineering/lakeflow-jobs/monitoring-and-alerting.md)

## 데이터 웨어하우징

* [Databricks SQL 개요](06-data-warehousing/databricks-sql-overview.md)
* [SQL 주요 기능](06-data-warehousing/sql-language-features.md)
  * [윈도우 함수](06-data-warehousing/window-functions.md)
  * [SQL 스크립팅](06-data-warehousing/sql-scripting.md)
  * [PIVOT과 UNPIVOT](06-data-warehousing/pivot-unpivot.md)
* [테이블과 뷰](06-data-warehousing/tables-and-views.md)
  * [구체화된 뷰 (Materialized View)](06-data-warehousing/materialized-views.md)
* [AI 함수](06-data-warehousing/ai-functions.md)
* [쿼리 최적화](06-data-warehousing/query-optimization.md)

## Unity Catalog

* [Unity Catalog란?](07-unity-catalog/what-is-unity-catalog.md)
* [네임스페이스와 객체](07-unity-catalog/namespace-and-objects.md)
  * [외부 로케이션](07-unity-catalog/external-locations.md)
  * [스토리지 자격 증명](07-unity-catalog/storage-credentials.md)
  * [UC 함수](07-unity-catalog/functions.md)
* [권한 관리](07-unity-catalog/access-control.md)
  * [행 필터 (Row Filter)](07-unity-catalog/row-filters.md)
  * [컬럼 마스킹 (Column Mask)](07-unity-catalog/column-masks.md)
  * [태그와 ABAC](07-unity-catalog/tags.md)
* [데이터 리니지](07-unity-catalog/data-lineage.md)
* [Delta Sharing](07-unity-catalog/delta-sharing.md)
  * [Databricks Marketplace](07-unity-catalog/marketplace.md)
  * [Clean Rooms](07-unity-catalog/clean-rooms.md)
* [Volumes](07-unity-catalog/volumes.md)
* [Metric Views (비즈니스 시맨틱)](08-ai-bi/metric-views.md)

## AI/BI

* [AI/BI 개요](08-ai-bi/aibi-overview.md)
* [AI/BI 대시보드](08-ai-bi/lakeview-dashboards.md)
* [Genie](08-ai-bi/genie.md)
* [알림과 스케줄링](08-ai-bi/alerts-and-scheduling.md)
* [AI/BI 아키텍처 심화](08-ai-bi/aibi-architecture.md)
* [하이브리드 BI 전략](08-ai-bi/hybrid-bi-strategy.md)

## 머신러닝

* [Databricks ML 개요](09-machine-learning/ml-on-databricks-overview.md)
* [ML Runtime](09-machine-learning/ml-runtime.md)
  * [AutoML](09-machine-learning/automl.md)
  * [분산 학습](09-machine-learning/distributed-training.md)
  * [하이퍼파라미터 튜닝](09-machine-learning/hyperparameter-tuning.md)
* [MLflow](09-machine-learning/mlflow/README.md)
  * [MLflow란?](09-machine-learning/mlflow/what-is-mlflow.md)
  * [실험 추적](09-machine-learning/mlflow/experiment-tracking.md)
  * [모델 레지스트리](09-machine-learning/mlflow/model-registry.md)
  * [MLflow Tracing](09-machine-learning/mlflow/mlflow-tracing.md)
  * [모델 평가](09-machine-learning/mlflow/mlflow-evaluation.md)
* [Model Serving](09-machine-learning/model-serving/README.md)
  * [Model Serving 개요](09-machine-learning/model-serving/model-serving-overview.md)
  * [Foundation Model API](09-machine-learning/model-serving/foundation-model-api.md)
  * [커스텀 모델 배포](09-machine-learning/model-serving/custom-model-deployment.md)
  * [AI Gateway](09-machine-learning/model-serving/ai-gateway.md)
  * [엔드포인트 모니터링](09-machine-learning/model-serving/endpoint-monitoring.md)
* [Feature Engineering](09-machine-learning/feature-engineering/README.md)
  * [Feature Engineering 개요](09-machine-learning/feature-engineering/feature-engineering-overview.md)
  * [피처 테이블 관리](09-machine-learning/feature-engineering/feature-table-management.md)
  * [실시간 피처 서빙](09-machine-learning/feature-engineering/online-serving.md)
* [Lakehouse Monitoring](09-machine-learning/lakehouse-monitoring.md)

## AI 에이전트

* [AI 에이전트란?](10-agent-development/what-is-ai-agent.md)
* [Vector Search](10-agent-development/vector-search.md)
* [RAG 파이프라인](10-agent-development/rag-pipeline.md)
* [에이전트 구축](10-agent-development/building-agents.md)
  * [도구와 UC 함수](10-agent-development/tools-and-functions.md)
* [에이전트 평가](10-agent-development/agent-evaluation.md)
* [에이전트 배포](10-agent-development/agent-deployment.md)
  * [Review App](10-agent-development/review-app.md)
  * [가드레일 (Guardrails)](10-agent-development/guardrails.md)
* [Agent Bricks](10-agent-development/agent-bricks.md)

## Lakebase

* [Lakebase란?](11-lakebase/what-is-lakebase.md)
* [Lakebase 설정](11-lakebase/lakebase-setup.md)
* [Data Sync](11-lakebase/data-sync.md)
* [Apps 연동](11-lakebase/lakebase-with-apps.md)

## 보안과 거버넌스

* [보안 개요](12-security-and-governance/security-overview.md)
  * [CMK 암호화](12-security-and-governance/cmk-encryption.md)
  * [시크릿 관리](12-security-and-governance/secret-management.md)
  * [감사 로그](12-security-and-governance/audit-logs.md)
* [인증과 접근 제어](12-security-and-governance/identity-and-access.md)
  * [SSO 설정](12-security-and-governance/sso-setup.md)
  * [SCIM 프로비저닝](12-security-and-governance/scim-provisioning.md)
  * [서비스 프린시펄](12-security-and-governance/service-principals.md)
* [네트워크 보안](12-security-and-governance/network-security.md)
* [시스템 테이블](12-security-and-governance/system-tables.md)
* [워크스페이스 관리](12-security-and-governance/workspace-admin.md)

## 모범 사례

* [비용 최적화](14-best-practices/cost-optimization.md)
* [성능 최적화](14-best-practices/performance-tuning.md)
* [엔터프라이즈 거버넌스](14-best-practices/enterprise-governance.md)

## 개발 도구

* [Databricks CLI](13-appendix/databricks-cli.md)
* [Declarative Automation Bundles](13-appendix/databricks-asset-bundles.md)
* [Databricks Apps](13-appendix/databricks-apps.md)
* [Databricks SDK](13-appendix/databricks-sdk.md)
* [Databricks Connect](13-appendix/databricks-connect.md)
* [REST API 활용](13-appendix/rest-api.md)
* [AI Dev Kit](13-appendix/ai-dev-kit.md)
* [Genie Code](13-appendix/genie-code.md)

## 부록 — 선행 지식

* [관계형 데이터베이스 기초](00-prerequisites/rdb-fundamentals.md)
* [데이터 모델링 — Star/Snowflake 스키마](00-prerequisites/schema-design-patterns.md)
* [빅데이터의 역사](00-prerequisites/bigdata-history.md)
* [빅데이터 생태계](00-prerequisites/bigdata-ecosystem.md)
* [실시간 처리 기술](00-prerequisites/realtime-processing.md)

## 부록 — 데이터 기초

* [데이터 엔지니어링이란?](01-data-fundamentals/what-is-data-engineering.md)
* [데이터 웨어하우스 vs 데이터 레이크](01-data-fundamentals/data-warehouse-vs-data-lake.md)
* [ETL과 ELT](01-data-fundamentals/etl-elt-basics.md)
* [배치 처리 vs 스트리밍 처리](01-data-fundamentals/batch-vs-streaming.md)
* [정형·반정형·비정형 데이터](01-data-fundamentals/structured-semi-unstructured.md)

## 부록 — 참고

* [용어 사전](13-appendix/glossary.md)
* [학습 로드맵](13-appendix/learning-path.md)
