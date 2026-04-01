# Databricks 종합 교육 자료

Databricks Data Intelligence Platform의 핵심 기능과 아키텍처를 체계적으로 다루는 한국어 교육 자료입니다.

## 커리큘럼

| 섹션 | 내용 |
|------|------|
| [00. 선행 지식](00-prerequisites/README.md) | RDB 기초, Star/Snowflake 스키마, 빅데이터 역사, 생태계, 실시간 처리 |
| [01. 데이터 기초](01-data-fundamentals/README.md) | 데이터 엔지니어링, 웨어하우스 vs 레이크, ETL/ELT, 배치 vs 스트리밍 |
| [02. Databricks 개요](02-databricks-overview/README.md) | Databricks란, 아키텍처, Workspace, Notebook |
| [03. 레이크하우스](03-lakehouse-architecture/README.md) | Delta Lake, Medallion 아키텍처, Iceberg 호환 |
| [04. 컴퓨트](04-compute-and-workspace/README.md) | Spark 기초, 클러스터, SQL Warehouse, Serverless |
| [05. 데이터 엔지니어링](05-data-engineering/README.md) | Auto Loader, SDP, Lakeflow Connect, Lakeflow Jobs |
| [06. 데이터 웨어하우징](06-data-warehousing/README.md) | Databricks SQL, AI 함수, 쿼리 최적화 |
| [07. Unity Catalog](07-unity-catalog/README.md) | 거버넌스, 권한, 리니지, Delta Sharing |
| [08. AI/BI](08-ai-bi/README.md) | 대시보드, Genie, 알림 |
| [09. 머신러닝](09-machine-learning/README.md) | MLflow, Model Serving, Feature Engineering |
| [10. AI 에이전트](10-agent-development/README.md) | Vector Search, RAG, Agent Evaluation, Agent Bricks |
| [11. Lakebase](11-lakebase/README.md) | PostgreSQL OLTP, Data Sync |
| [12. 보안](12-security-and-governance/README.md) | 인증, 네트워크, 시스템 테이블 |
| [13. 부록](13-appendix/README.md) | CLI, Asset Bundles, 용어 사전, 학습 로드맵 |

## 역할별 추천 경로

- **데이터 엔지니어**: 00 → 01 → 02 → 03 → 04 → **05(핵심)** → 07
- **데이터 분석가**: 00(RDB) → 01 → 02 → **06(핵심)** → **08(핵심)** → 07
- **데이터 과학자**: 01 → 02 → 04 → **09(핵심)** → 10
- **플랫폼 관리자**: 02 → 04 → **07(핵심)** → **12(핵심)**

## 참고 문서

- [Databricks AWS 공식 문서](https://docs.databricks.com/aws/en/)
- [Azure Databricks 공식 문서](https://learn.microsoft.com/en-us/azure/databricks/)
- [Databricks 릴리즈 노트](https://docs.databricks.com/aws/en/release-notes/)
- [Databricks 공식 블로그](https://www.databricks.com/blog)
