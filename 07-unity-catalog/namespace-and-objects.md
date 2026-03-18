# 네임스페이스 구조

## Unity Catalog의 3-Level 네임스페이스

> 💡 Unity Catalog는 **Catalog → Schema → Object** 3단계 계층 구조로 모든 데이터 자산을 관리합니다. 이 구조는 ANSI SQL 표준의 3-Level 네임스페이스를 따릅니다.

```
metastore (최상위 - 리전당 1개)
  ├── catalog_1 (카탈로그)
  │     ├── schema_a (스키마)
  │     │     ├── table_1 (테이블)
  │     │     ├── view_1 (뷰)
  │     │     ├── volume_1 (볼륨 - 파일)
  │     │     ├── function_1 (함수)
  │     │     └── model_1 (ML 모델)
  │     └── schema_b
  └── catalog_2
```

### 완전한 이름 (Fully Qualified Name)

```sql
-- 3-Level 이름: catalog.schema.object
SELECT * FROM production.ecommerce.orders;
--               ^^^^^^^^^^  ^^^^^^^^^  ^^^^^^
--               카탈로그      스키마     테이블
```

---

## 각 레벨의 역할

| 레벨 | 역할 | 일반적인 구성 전략 | 예시 |
|------|------|-----------------|------|
| **Metastore** | 최상위 컨테이너. **리전별 하나**만 존재하며, 여러 Workspace에서 공유됩니다 | 계정 수준 관리 | US-East Metastore |
| **Catalog** | 데이터 자산의 최상위 논리 그룹입니다. **환경별** 또는 **팀별**로 분리합니다 | `production`, `staging`, `dev`, `sandbox` | 환경별 분리 |
| **Schema** | 관련 객체를 논리적으로 그룹화합니다. **도메인별** 또는 **Medallion 계층별**로 분리합니다 | `ecommerce`, `hr`, `finance` 또는 `bronze`, `silver`, `gold` | 도메인별 분리 |
| **Object** | 실제 데이터 자산입니다 | 테이블, 뷰, 볼륨, 함수, 모델 | `orders`, `customers` |

---

## 객체 유형 상세

| 객체 유형 | 설명 | 생성 예시 |
|-----------|------|----------|
| **Table** | 행/열 구조의 데이터입니다. Managed(관리형)와 External(외부) 두 유형이 있습니다 | `CREATE TABLE catalog.schema.orders (...)` |
| **View** | SQL 쿼리의 별칭입니다. 데이터를 물리적으로 저장하지 않습니다 | `CREATE VIEW catalog.schema.v_active_customers AS SELECT ...` |
| **Materialized View** | 쿼리 결과를 물리적으로 저장하고, 자동으로 갱신합니다 | `CREATE MATERIALIZED VIEW catalog.schema.mv_daily_kpi AS SELECT ...` |
| **Streaming Table** | 스트리밍 데이터를 증분 처리하는 테이블입니다 | `CREATE STREAMING TABLE catalog.schema.bronze_orders AS ...` |
| **Volume** | 비테이블 파일(PDF, 이미지, CSV 등)을 저장하는 공간입니다 | `CREATE VOLUME catalog.schema.raw_files` |
| **Function** | SQL 또는 Python 사용자 정의 함수입니다 | `CREATE FUNCTION catalog.schema.calculate_tax(...)` |
| **Model** | MLflow로 등록한 ML 모델입니다 | `mlflow.register_model(..., "catalog.schema.model_name")` |
| **Connection** | 외부 시스템(DB, SaaS)의 연결 정보입니다 | `CREATE CONNECTION mysql_prod TYPE MYSQL OPTIONS (...)` |
| **External Location** | 외부 클라우드 스토리지 경로입니다 | `CREATE EXTERNAL LOCATION my_s3 URL 's3://bucket/path'` |
| **Storage Credential** | 클라우드 스토리지 접근 자격 증명입니다 | `CREATE STORAGE CREDENTIAL my_cred ...` |

---

## 네임스페이스 구성 전략

### 전략 1: 환경별 카탈로그 + 도메인별 스키마 (가장 일반적)

```
production
  ├── ecommerce (주문, 고객, 상품)
  ├── hr (직원, 급여)
  └── finance (매출, 비용)

staging
  ├── ecommerce
  └── hr

dev
  ├── ecommerce
  └── sandbox (개인 실험용)
```

### 전략 2: 환경별 카탈로그 + Medallion별 스키마

```
production
  ├── bronze (원본 데이터)
  ├── silver (정제된 데이터)
  └── gold (비즈니스 집계)

dev
  ├── bronze
  ├── silver
  └── gold
```

### SQL로 네임스페이스 생성

```sql
-- 카탈로그 생성
CREATE CATALOG IF NOT EXISTS production
COMMENT '프로덕션 데이터';

-- 스키마 생성
CREATE SCHEMA IF NOT EXISTS production.ecommerce
COMMENT '이커머스 도메인';

-- 테이블 생성
CREATE TABLE production.ecommerce.orders (...) USING DELTA;

-- 볼륨 생성
CREATE VOLUME production.ecommerce.raw_files
COMMENT '원본 파일 저장소';

-- 기본 카탈로그/스키마 설정 (매번 3-Level 이름을 쓰지 않아도 됨)
USE CATALOG production;
USE SCHEMA ecommerce;
SELECT * FROM orders;  -- = production.ecommerce.orders
```

---

## Managed vs External

| 비교 | Managed (관리형) | External (외부) |
|------|-----------------|----------------|
| **데이터 위치** | Databricks가 관리하는 위치에 자동 저장 | 사용자가 지정한 외부 경로 (S3, ADLS) |
| **DROP 시** | 테이블 + 데이터 **모두 삭제** | 테이블 정의만 삭제, **데이터는 유지** |
| **Predictive Opt** | ✅ 지원 | ❌ 미지원 |
| **권장** | ✅ 신규 테이블에 권장 | 기존 데이터 위치를 유지해야 할 때 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **3-Level 네임스페이스** | Catalog → Schema → Object 계층 구조입니다 |
| **Catalog** | 최상위 그룹. 환경별(prod/dev) 또는 팀별로 분리합니다 |
| **Schema** | 관련 객체를 묶는 논리 그룹. 도메인별로 분리합니다 |
| **10가지 객체 유형** | Table, View, MV, Streaming Table, Volume, Function, Model 등입니다 |
| **Managed vs External** | 데이터 관리 방식이 다릅니다. Managed를 권장합니다 |

---

## 참고 링크

- [Databricks: Unity Catalog objects](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)
- [Databricks: Create catalogs and schemas](https://docs.databricks.com/aws/en/data-governance/unity-catalog/create-catalogs.html)
