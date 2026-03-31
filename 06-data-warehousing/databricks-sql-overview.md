# Databricks SQL 개요

## Databricks SQL이란?

> 💡 **Databricks SQL (DBSQL)** 은 Delta Lake 위의 데이터를 **SQL로 직접 조회하고 분석**할 수 있는 Databricks의 SQL 분석 환경입니다. 별도의 데이터 복사 없이, 레이크하우스 데이터에 대해 **웨어하우스급 SQL 성능**을 제공합니다.

전통적인 데이터 웨어하우스(Snowflake, Redshift 등)에서는 원본 데이터를 **독자 형식으로 복사**해야 했습니다. Databricks SQL은 이런 복사 과정 없이, **오픈 포맷(Delta Lake)** 데이터를 직접 쿼리합니다. 이로써 데이터 사일로(Data Silo)와 ETL 복잡성을 제거하고, 동일한 데이터에 대해 데이터 엔지니어링과 SQL 분석을 하나의 플랫폼에서 수행할 수 있습니다.

**전통적 방식 vs Databricks SQL 방식**

| 방식 | 흐름 | 설명 |
|------|------|------|
| **전통적 방식** | Data Lake → (ETL 복사) → Data Warehouse → BI 도구 | 데이터를 별도 웨어하우스로 복사해야 합니다 |
| **Databricks SQL 방식** | Delta Lake (레이크하우스) → SQL Warehouse (Photon 엔진) → SQL Editor / BI 도구 / AI/BI Dashboard | 복사 없이 레이크하우스에서 직접 분석합니다 |

---

## DBSQL의 핵심 가치

| 가치 | 설명 |
|------|------|
| **데이터 복사 불필요** | 레이크하우스의 Delta 테이블을 직접 조회합니다. ETL로 별도 웨어하우스에 복사할 필요가 없습니다 |
| **웨어하우스급 성능** | Photon 엔진이 기본 활성화되어 기존 웨어하우스(Redshift, Snowflake) 수준의 SQL 성능을 제공합니다 |
| **통합 거버넌스** | Unity Catalog의 권한이 SQL 쿼리에도 동일하게 적용됩니다 |
| **AI 함수 내장** | SQL 안에서 직접 LLM을 호출하여 분류, 추출, 요약을 수행할 수 있습니다 |
| **Serverless** | SQL Warehouse가 수 초 만에 시작되고, 유휴 시 자동 중지됩니다 |

---

## 주요 구성 요소

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **SQL Editor** | SQL 작성/실행 | 자동완성, 구문 강조, 실행 계획 분석을 지원하는 웹 기반 편집기입니다 |
| **SQL Warehouse** | 쿼리 실행 엔진 | SQL 쿼리를 실행하는 전용 서버리스 컴퓨팅 리소스입니다 |
| **AI/BI Dashboard** | 시각화 | SQL 쿼리 결과를 차트, 표, KPI 카운터로 시각화합니다 |
| **Query History** | 이력 관리 | 실행한 쿼리의 이력, 성능, 비용을 확인합니다 |
| **Alerts** | 자동 알림 | 쿼리 결과가 조건을 만족하면 알림을 발송합니다 |

---

## SQL Warehouse 아키텍처

SQL Warehouse는 DBSQL의 **컴퓨팅 엔진**입니다. Photon이라는 C++ 네이티브 실행 엔진을 탑재하여 기존 Spark SQL 대비 **최대 12배** 빠른 성능을 제공합니다.

**SQL Warehouse 아키텍처**

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **사용자 쿼리** | 입력 | Query Router로 전달됩니다 |
| **Query Router** | 라우팅 | 쿼리를 적절한 처리 경로로 분배합니다 (결과 캐시 활용 가능) |
| **Query Optimizer** | 최적화 | 비용 기반 최적화를 수행합니다 |
| **Photon Engine** | 실행 엔진 | C++ 네이티브 벡터화 처리 (자동 스케일링 + 디스크 캐시 지원) |
| **클라우드 스토리지** | 데이터 소스 | Delta Lake에서 데이터를 읽습니다 |

### Photon 엔진

Photon은 C++로 작성된 **벡터화 쿼리 실행 엔진(Vectorized Query Engine)**입니다.

| 특성 | 설명 |
|------|------|
| **네이티브 C++** | JVM 오버헤드 없이 CPU 캐시를 최대한 활용합니다 |
| **벡터화 처리** | 한 번에 수천 행을 배치(Batch)로 처리하여 CPU 효율을 극대화합니다 |
| **적응형 쿼리 실행** | 런타임에 데이터 분포를 파악하여 실행 계획을 자동 조정합니다 |
| **기본 활성화** | SQL Warehouse에서는 Photon이 기본으로 활성화되어 있습니다 |

### SQL Warehouse 타입 비교

| 비교 항목 | Classic | Pro | Serverless |
|-----------|---------|-----|------------|
| **인프라 관리** | 고객 클라우드 | 고객 클라우드 | **Databricks 관리** |
| **시작 시간** | 5~10분 | 5~10분 | **수 초** |
| **Photon** | ✅ | ✅ | ✅ |
| **AI 함수** | ❌ | ✅ | ✅ |
| **Query Federation** | ❌ | ✅ | ✅ |
| **Predictive I/O** | ❌ | ❌ | ✅ |
| **Auto Stop** | ✅ | ✅ | ✅ (자동) |
| **비용 모델** | DBU + 인프라 | DBU + 인프라 | **DBU만** (인프라 포함) |
| **권장 사용** | 비용 최적화 | 일반 프로덕션 | **신규 프로젝트 (권장)** |

> 💡 **Serverless SQL Warehouse를 권장하는 이유**: 수 초 만에 시작되어 유휴 비용을 최소화하고, Databricks가 인프라를 관리하므로 운영 부담이 없습니다. Predictive I/O 등 최신 최적화 기능도 Serverless에서만 지원됩니다.

---

## 기존 데이터 웨어하우스와의 비교

| 비교 항목 | Databricks SQL | Snowflake | Amazon Redshift |
|-----------|---------------|-----------|-----------------|
| **아키텍처** | 레이크하우스 (오픈 포맷) | 독자 포맷 | 독자 포맷 |
| **데이터 저장** | 고객 클라우드 (S3/ADLS) | Snowflake 관리 | Redshift 관리 |
| **파일 포맷** | Delta Lake (Parquet 기반, 오픈) | 독자 포맷 | 독자 포맷 |
| **ML/AI 통합** | ✅ 네이티브 (MLflow, Agent) | 제한적 | 제한적 (SageMaker 연동) |
| **AI 함수** | ✅ ai_query, ai_classify 등 | Cortex (유사) | 없음 |
| **스트리밍** | ✅ Structured Streaming 통합 | Snowpipe | Kinesis 연동 |
| **거버넌스** | Unity Catalog (통합) | 내장 거버넌스 | IAM + Lake Formation |
| **Serverless** | ✅ 수 초 시작 | ✅ | Serverless 옵션 있음 |
| **Photon 엔진** | ✅ C++ 네이티브 | Snowflake 엔진 | - |

---

## SQL Editor 주요 기능

| 기능 | 설명 |
|------|------|
| **자동완성** | 테이블명, 컬럼명, SQL 키워드를 자동 완성합니다 |
| **구문 강조** | SQL 구문을 색상으로 구분하여 가독성을 높입니다 |
| **실행 계획** | `EXPLAIN` 없이도 쿼리 실행 계획을 시각적으로 분석합니다 |
| **매개변수** | `{{parameter_name}}` 형태로 동적 매개변수를 사용합니다 |
| **스니펫** | 자주 사용하는 SQL 패턴을 저장하고 재사용합니다 |
| **버전 이력** | 쿼리의 변경 이력이 자동으로 저장됩니다 |
| **공유** | 쿼리를 다른 사용자와 공유할 수 있습니다 |

### 매개변수 활용 예제

SQL Editor에서 `{{parameter}}` 구문을 사용하면 동적으로 값을 바꿔가며 쿼리를 실행할 수 있습니다.

```sql
-- 매개변수를 사용한 쿼리 (SQL Editor에서 실행)
SELECT
    order_date,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue
FROM catalog.schema.orders
WHERE order_date BETWEEN '{{start_date}}' AND '{{end_date}}'
  AND status = '{{order_status}}'
GROUP BY order_date
ORDER BY order_date;
```

> 💡 **매개변수 타입**: Text, Number, Date, Date Range, Dropdown 등 다양한 타입을 지원합니다. Dropdown은 미리 정의한 값 목록 또는 쿼리 결과를 선택지로 사용할 수 있습니다.

---

## 쿼리 히스토리 및 쿼리 프로필

### 쿼리 히스토리 (Query History)

실행한 모든 쿼리의 이력, 상태, 실행 시간, 비용(DBU)을 확인할 수 있습니다. 관리자는 워크스페이스 전체의 쿼리 이력을 볼 수 있어 **비용 분석과 성능 모니터링**에 유용합니다.

| 항목 | 설명 |
|------|------|
| **상태** | 성공, 실패, 취소, 실행 중 등 쿼리 상태를 확인합니다 |
| **실행 시간** | 쿼리 실행에 소요된 시간을 밀리초 단위로 표시합니다 |
| **스캔 데이터** | 쿼리가 읽은 데이터 양을 확인합니다 |
| **사용자** | 누가 쿼리를 실행했는지 확인합니다 (관리자 기능) |
| **비용** | 쿼리 실행에 사용된 DBU를 확인합니다 |

### 쿼리 프로필 (Query Profile)

개별 쿼리의 **실행 계획을 시각적으로 분석**할 수 있습니다. 병목 지점을 찾아 성능을 최적화하는 데 핵심적인 도구입니다.

| 분석 항목 | 설명 |
|-----------|------|
| **연산자 트리** | 쿼리가 어떤 순서로 실행되었는지 트리 형태로 보여줍니다 |
| **시간 분포** | 각 연산자가 소비한 시간 비율을 확인합니다 |
| **데이터 흐름** | 각 단계에서 처리된 행(Row) 수를 확인합니다 |
| **Spill 여부** | 메모리 부족으로 디스크에 데이터가 넘친 구간을 표시합니다 |
| **Photon 활용** | 어떤 연산이 Photon으로 실행되었는지 확인합니다 |

> 🆕 **Query Tags**: SQL Warehouse에서 실행되는 쿼리에 태그를 달아 비즈니스 맥락별로 비용을 추적할 수 있습니다.

> 🆕 **Warehouse Activity Details**: SQL Warehouse의 쿼리 활동, 세션, 유휴 상태를 색상 코딩된 시각화로 확인할 수 있습니다.

---

## 외부 BI 도구 연동

Databricks SQL은 표준 **JDBC/ODBC** 프로토콜을 지원하여 대부분의 BI 도구와 연동할 수 있습니다.

### 지원 BI 도구

| BI 도구 | 연결 방식 | 특징 |
|---------|----------|------|
| **Tableau** | Databricks 전용 커넥터 | Tableau Desktop/Server 모두 지원, 라이브 연결 권장 |
| **Power BI** | Databricks 전용 커넥터 | DirectQuery 모드로 실시간 데이터 조회 가능 |
| **dbt** | dbt-databricks 어댑터 | SQL 기반 변환 파이프라인을 Databricks에서 실행 |
| **Looker** | JDBC | Looker Modeling Language와 통합 |
| **기타** | JDBC/ODBC | 표준 프로토콜을 지원하는 모든 도구 연결 가능 |

### JDBC/ODBC 연결 설정

BI 도구에서 SQL Warehouse에 연결하려면 다음 정보가 필요합니다.

```
Server Hostname: <workspace-url>.databricks.com
Port:           443
HTTP Path:      /sql/1.0/warehouses/<warehouse-id>
Auth:           OAuth (M2M) 또는 Personal Access Token
```

> 💡 **연결 정보 확인 방법**: SQL Warehouse의 상세 페이지 → **Connection Details** 탭에서 모든 연결 정보를 확인할 수 있습니다. 각 BI 도구별 설정 가이드 링크도 제공됩니다.

---

## 비용 관리 및 최적화

SQL Warehouse 비용은 **사용한 DBU(Databricks Unit)** 를 기준으로 과금됩니다. 다음 전략으로 비용을 효과적으로 관리할 수 있습니다.

### Auto Stop (자동 중지)

유휴 상태가 지속되면 자동으로 Warehouse를 중지하여 비용을 절감합니다.

| 설정 | 설명 | 권장값 |
|------|------|--------|
| **Auto Stop 시간** | 유휴 상태에서 자동 중지까지의 시간 | 10~15분 (Serverless는 자동 관리) |
| **Auto Resume** | 쿼리 요청 시 자동 재시작 | 항상 활성화 |

### 스케일링 설정

| 설정 | 설명 | 권장 |
|------|------|------|
| **Min Clusters** | 최소 클러스터 수 | 1 (항상 응답 가능) |
| **Max Clusters** | 최대 클러스터 수 | 동시 사용자 수에 따라 조정 |
| **Scaling Policy** | 스케일링 정책 | Enhanced Autoscaling (권장) |

### 비용 최적화 팁

```sql
-- 1. 결과 캐시 활용: 동일 쿼리는 캐시에서 즉시 반환 (추가 비용 없음)
-- SQL Warehouse는 결과 캐시를 기본 활성화합니다

-- 2. 필요한 컬럼만 SELECT (SELECT * 지양)
-- ❌ 비효율적
SELECT * FROM catalog.schema.large_table;

-- ✅ 효율적
SELECT order_id, amount, order_date
FROM catalog.schema.large_table
WHERE order_date >= '2025-01-01';

-- 3. EXPLAIN으로 쿼리 계획 확인
EXPLAIN SELECT ... FROM ...;

-- 4. 쿼리 태그로 비용 추적
SET statement_tag = 'marketing_dashboard';
SELECT ... FROM ...;
```

---

## 실습: Warehouse 생성부터 쿼리 실행까지

### Step 1: SQL Warehouse 생성 (UI)

1. 좌측 메뉴에서 **SQL Warehouses** 클릭
2. **Create SQL Warehouse** 클릭
3. 다음 설정 입력:
   - **이름**: `my-analytics-warehouse`
   - **크기**: Small (개발/테스트 용도)
   - **Type**: Serverless
   - **Auto Stop**: 10분
   - **Min/Max Clusters**: 1 / 2
4. **Create** 클릭

### Step 2: SQL Editor에서 쿼리 실행

```sql
-- 카탈로그와 스키마 설정
USE CATALOG my_catalog;
USE SCHEMA my_schema;

-- 샘플 테이블 생성
CREATE TABLE IF NOT EXISTS sales (
    sale_id     BIGINT GENERATED ALWAYS AS IDENTITY,
    product     STRING,
    region      STRING,
    amount      DECIMAL(10,2),
    sale_date   DATE
) USING DELTA;

-- 데이터 삽입
INSERT INTO sales (product, region, amount, sale_date) VALUES
    ('Laptop', 'Seoul', 1500000, '2025-01-15'),
    ('Mouse', 'Busan', 25000, '2025-01-16'),
    ('Keyboard', 'Seoul', 89000, '2025-01-16'),
    ('Monitor', 'Daegu', 450000, '2025-01-17'),
    ('Laptop', 'Seoul', 1800000, '2025-01-17');

-- 지역별 매출 분석
SELECT
    region,
    COUNT(*)    AS sale_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_revenue
FROM sales
GROUP BY region
ORDER BY total_revenue DESC;
```

### Step 3: 쿼리 프로필 확인

1. 쿼리 실행 후 결과 하단의 **Query Profile** 탭 클릭
2. 연산자 트리에서 **가장 시간이 오래 걸린 노드**를 확인
3. Scan 단계에서 읽은 파일 수와 데이터 양을 확인

---

## SQL Warehouse vs All-Purpose Cluster

| 비교 항목 | SQL Warehouse | All-Purpose Cluster |
|-----------|---------------|-------------------|
| **주 용도** | SQL 분석, BI 연동 | 개발, ML, 다양한 워크로드 |
| **언어 지원** | SQL 전용 | Python, Scala, R, SQL |
| **엔진** | Photon (기본) | Spark (Photon 선택) |
| **시작 시간** | 수 초 (Serverless) | 3~10분 |
| **비용** | SQL 워크로드에 최적화 | 범용 (상대적으로 고비용) |
| **스케일링** | 자동 멀티 클러스터 | 단일 클러스터 내 스케일링 |
| **BI 도구 연동** | ✅ JDBC/ODBC 최적화 | ⚠️ 가능하나 비권장 |
| **동시 사용자** | 수십~수백 명 동시 쿼리 | 제한적 |
| **권장 사용** | 프로덕션 SQL 분석 | 개발/실험 |

> 💡 **언제 무엇을 쓸까?**: SQL로 데이터를 분석하거나 BI 대시보드를 운영한다면 **SQL Warehouse**를 사용하세요. Python/Scala로 ML 모델을 학습하거나 복잡한 ETL을 개발한다면 **All-Purpose Cluster**를 사용하세요.

---

## 언제 DBSQL을 사용하나요?

| 사용 사례 | 설명 | 대안 |
|-----------|------|------|
| **대화형 SQL 분석** | 데이터를 탐색하고, 집계하고, 패턴을 찾을 때 | 없음 (DBSQL이 최적) |
| **대시보드 생성** | AI/BI Dashboard에서 시각화할 데이터를 쿼리할 때 | Tableau, Power BI |
| **BI 도구 연결** | Tableau, Power BI 등 외부 BI 도구의 백엔드로 사용할 때 | 없음 (JDBC/ODBC 연결) |
| **데이터 검증** | ETL 파이프라인의 결과를 SQL로 검증할 때 | Notebook |
| **Ad-hoc 분석** | 비정기적인 일회성 분석을 수행할 때 | Notebook |
| **AI 분석** | SQL 안에서 LLM으로 텍스트 분류, 요약을 수행할 때 | Python + LLM API |
| **알림 모니터링** | 조건 기반 알림으로 이상 감지를 할 때 | 외부 모니터링 도구 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Databricks SQL** | 레이크하우스 데이터에 대한 SQL 분석 환경입니다 |
| **SQL Warehouse** | SQL 쿼리를 실행하는 서버리스 컴퓨팅입니다. Photon 엔진이 기본 활성화됩니다 |
| **Photon 엔진** | C++ 네이티브 벡터화 실행 엔진으로, 기존 Spark SQL 대비 최대 12배 빠릅니다 |
| **Serverless** | 수 초 만에 시작되며 Databricks가 인프라를 관리합니다. 신규 프로젝트에 권장됩니다 |
| **SQL Editor** | 웹 기반 SQL 작성/실행 도구입니다. 매개변수, 스니펫, 버전 이력을 지원합니다 |
| **쿼리 프로필** | 쿼리 실행 계획을 시각적으로 분석하여 병목을 파악합니다 |
| **BI 연동** | JDBC/ODBC를 통해 Tableau, Power BI 등과 직접 연결됩니다 |
| **오픈 포맷** | Delta Lake(Parquet 기반)로 저장되어 벤더 종속이 없습니다 |

---

## 참고 링크

- [Databricks: Databricks SQL](https://docs.databricks.com/aws/en/sql/)
- [Databricks: SQL Warehouses](https://docs.databricks.com/aws/en/compute/sql-warehouse/)
- [Databricks: SQL Editor](https://docs.databricks.com/aws/en/sql/user/queries/)
- [Databricks: Query History](https://docs.databricks.com/aws/en/sql/admin/query-history.html)
- [Databricks: Query Profile](https://docs.databricks.com/aws/en/sql/user/queries/query-profile.html)
- [Databricks: Photon](https://docs.databricks.com/aws/en/compute/photon.html)
- [Databricks: BI tool connections](https://docs.databricks.com/aws/en/partners/bi/)
- [Azure Databricks: Data Warehousing](https://learn.microsoft.com/en-us/azure/databricks/sql/)
