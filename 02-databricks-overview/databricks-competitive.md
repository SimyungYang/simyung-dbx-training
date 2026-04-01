# Databricks vs 경쟁사

## 핵심 차이 3가지

데이터 플랫폼 시장에는 Snowflake, AWS EMR, Google BigQuery 등 강력한 경쟁자들이 있습니다. Databricks와의 핵심 차이를 정리하면 다음과 같습니다.

### 차이 1: 오픈소스 기반 vs 독자 포맷

| 플랫폼 | 데이터 포맷 | 벤더 락인 |
|--------|------------|-----------|
| **Databricks** | Delta Lake (오픈소스, Parquet 기반) | 낮음 — 데이터를 S3/ADLS에 직접 소유합니다 |
| **Snowflake** | 독자 포맷 (내부 스토리지) | 높음 — 데이터를 꺼내려면 COPY INTO로 내보내야 합니다 |
| **BigQuery** | 독자 포맷 (Capacitor) | 높음 — 데이터가 Google 내부에 저장됩니다 |

> 💡 ** 이것이 왜 중요한가요?** 3년 후에 플랫폼을 변경해야 할 때, Databricks는 데이터가 이미 S3/ADLS의 Parquet 파일이므로 다른 도구에서 바로 읽을 수 있습니다. 반면 Snowflake나 BigQuery에서 데이터를 옮기려면 대규모 마이그레이션 프로젝트가 필요합니다.

### 차이 2: 엔지니어링 + ML/AI 통합 vs SQL 분석 특화

| 플랫폼 | 강점 | 약점 |
|--------|------|------|
| **Databricks** | ETL, ML, AI Agent, Streaming까지 하나에서 처리 | SQL 편의성이 Snowflake보다 다소 뒤처졌으나, 최근 빠르게 개선 중입니다 |
| **Snowflake** | SQL 분석이 매우 쉽고, 비개발자도 빠르게 적응 | Python/ML 워크로드는 최근에야 지원하기 시작했습니다 (Snowpark) |
| **AWS EMR** | 비용이 저렴하고 세밀한 커스터마이징 가능 | 모든 것을 직접 설정해야 합니다. 관리형 서비스가 아닙니다 |

### 차이 3: Spark 네이티브 성능

Databricks의 Spark 런타임은 오픈소스 Spark 대비 **Photon 엔진** 을 통해 SQL 워크로드에서 2~8배 빠른 성능을 제공합니다. Spark를 만든 회사이니만큼, 내부 최적화 수준이 다릅니다.

| 항목 | 오픈소스 Spark (EMR 등) | Databricks Runtime |
|------|------------------------|-------------------|
| SQL 성능 | 기본 Catalyst 옵티마이저 | Photon (C++로 작성된 벡터화 엔진) |
| I/O 최적화 | 기본 | Predictive I/O, Data Skipping, Z-Order |
| 자동 튜닝 | 수동 설정 | Adaptive Query Execution 자동 적용 |

---

## "우리 회사에 Databricks가 필요한가?" 판단 체크리스트

모든 회사에 Databricks가 최선은 아닙니다. 아래 체크리스트로 판단해 보시기 바랍니다.

### Databricks가 적합한 경우

- [ ] 데이터 파이프라인(ETL)과 SQL 분석을 **하나의 플랫폼** 에서 하고 싶다
- [ ] **머신러닝/AI** 워크로드가 있거나 향후 계획이 있다
- [ ] 현재 ** 여러 도구를 조합**하여 사용하고 있으며, 통합하고 싶다
- [ ] 데이터 ** 거버넌스(누가, 어떤 데이터를, 왜 접근했는지)** 가 중요하다
- [ ] ** 스트리밍**(실시간) 데이터 처리가 필요하다
- [ ] ** 벤더 락인**을 최소화하고 싶다 (오픈소스 기반)
- [ ] 데이터 팀이 **5명 이상** 이며, 다양한 역할(엔지니어, 분석가, 과학자)이 협업한다

### Databricks보다 다른 선택이 나을 수 있는 경우

- [ ] SQL 분석만 필요하고, ML/AI 계획이 없다 → **Snowflake** 또는 **BigQuery** 검토
- [ ] 예산이 매우 제한적이고, 자체 운영 역량이 있다 → 오픈소스(Spark + Iceberg + Trino) 검토
- [ ] AWS 서비스에 이미 깊이 통합되어 있다 → **AWS Glue + Redshift + SageMaker** 검토
- [ ] 데이터 팀이 1~2명이고, 데이터 규모가 작다 → PostgreSQL + dbt로 충분할 수 있습니다

---

## 참고 링크

- [Databricks: Get started with Databricks](https://docs.databricks.com/aws/en/getting-started/)
- [Databricks Blog: What is a Data Intelligence Platform?](https://www.databricks.com/blog)
