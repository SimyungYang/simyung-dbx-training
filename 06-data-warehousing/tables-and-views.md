# 테이블과 뷰

## 왜 테이블과 뷰의 구분이 필요한가?

데이터 웨어하우스에서 데이터를 저장하고 활용하는 방식은 크게 **테이블(Table)** 과 **뷰(View)** 로 나뉩니다. 테이블은 데이터를 **물리적으로 저장** 하는 객체이고, 뷰는 데이터를 **논리적으로 참조** 하는 객체입니다.

> 💡 **비유**: 테이블은 도서관의 **실제 책** 이고, 뷰는 **목차나 색인** 과 같습니다. 책 자체를 복사하지 않고도, 목차를 통해 원하는 정보에 빠르게 접근할 수 있습니다.

이 구분이 중요한 이유는 **비용, 성능, 보안, 유지보수** 측면에서 각각 최적의 선택이 달라지기 때문입니다. 잘못된 선택은 불필요한 스토리지 비용 증가나 쿼리 성능 저하로 이어질 수 있습니다.

**데이터 객체 분류**

| 대분류 | 유형 | 설명 |
|--------|------|------|
| **테이블 (Table)** | 관리형 테이블 (Managed Table) | Databricks가 데이터와 메타데이터를 모두 관리합니다 |
|  | 외부 테이블 (External Table) | 외부 스토리지의 데이터를 참조합니다 |
|  | 스트리밍 테이블 (Streaming Table) | 스트리밍 데이터를 수집합니다 |
| **뷰 (View)** | 일반 뷰 (View) | SQL 쿼리를 저장하며, 실행 시 계산됩니다 |
|  | 임시 뷰 (Temporary View) | 세션 내에서만 유효한 뷰입니다 |
|  | 구체화된 뷰 (Materialized View) | 결과를 미리 계산하여 저장하는 뷰입니다 |

---

## 테이블 유형

### 관리형 테이블(Managed Table) vs 외부 테이블(External Table)

Databricks에서 생성하는 테이블은 크게 **관리형 테이블** 과 **외부 테이블** 두 가지로 나뉩니다.

| 비교 항목 | 관리형 테이블 (Managed) | 외부 테이블 (External) |
|-----------|----------------------|---------------------|
| **데이터 위치** | Unity Catalog가 관리하는 기본 스토리지 | 사용자가 지정한 외부 경로 (S3, ADLS, GCS) |
| **메타데이터** | Unity Catalog가 관리 | Unity Catalog가 관리 |
| **삭제 시 동작** | 메타데이터 + 데이터 **모두 삭제** | 메타데이터만 삭제, **데이터는 유지** |
| **LOCATION 절** | 지정하지 않음 | 반드시 `LOCATION` 지정 |
| **권장 사용** | 대부분의 일반적인 경우 (권장) | 외부 시스템과 데이터 공유, 기존 데이터 활용 |
| **거버넌스** | Unity Catalog가 완전 관리 | Unity Catalog가 메타데이터만 관리 |
| **스토리지 비용** | Databricks 관리 스토리지 | 사용자 관리 스토리지 |

> 💡 **관리형 테이블을 권장하는 이유**: Unity Catalog가 데이터 라이프사이클을 완전히 관리하므로 거버넌스, 리니지 추적, 보안 측면에서 유리합니다. 외부 테이블은 다른 시스템과 동일한 데이터를 공유해야 하는 경우에만 사용하는 것이 좋습니다.

```sql
-- ✅ 관리형 테이블 (권장) — LOCATION 절 없음
CREATE TABLE catalog.schema.orders (
    order_id    BIGINT       COMMENT '주문 고유 ID',
    customer_id BIGINT       COMMENT '고객 ID',
    order_date  DATE         COMMENT '주문일',
    amount      DECIMAL(10,2) COMMENT '주문 금액',
    status      STRING       COMMENT '주문 상태'
) USING DELTA
COMMENT '주문 관리 테이블';

-- 외부 테이블 — LOCATION 지정 필수
CREATE TABLE catalog.schema.external_orders (
    order_id    BIGINT,
    customer_id BIGINT,
    order_date  DATE,
    amount      DECIMAL(10,2),
    status      STRING
) USING DELTA
LOCATION 's3://my-bucket/external/orders/'
COMMENT '외부 스토리지에 저장된 주문 테이블';
```

---

## Delta 테이블의 내부 구조

Databricks에서 생성되는 모든 테이블은 기본적으로 **Delta Lake** 형식입니다. Delta 테이블은 **Parquet 데이터 파일** 과 **트랜잭션 로그(_delta_log)** 로 구성됩니다.

**Delta Table 내부 구조**

| 경로 | 파일 유형 | 설명 |
|------|----------|------|
| `테이블 디렉토리/` | 루트 | 테이블의 최상위 디렉토리입니다 |
| `_delta_log/` | 트랜잭션 로그 | 모든 변경 이력을 기록합니다 |
| `_delta_log/00000.json` | 커밋 로그 | 개별 트랜잭션의 변경 사항입니다 |
| `_delta_log/00001.json` | 커밋 로그 | 개별 트랜잭션의 변경 사항입니다 |
| `_delta_log/00010.checkpoint.parquet` | 체크포인트 | 로그를 주기적으로 요약한 스냅샷입니다 |
| `part-00000.parquet` | 데이터 파일 | 실제 데이터가 Parquet 형식으로 저장됩니다 |
| `part-00001.parquet` | 데이터 파일 | 실제 데이터가 Parquet 형식으로 저장됩니다 |
| `part-00002.parquet` | 데이터 파일 | 실제 데이터가 Parquet 형식으로 저장됩니다 |

| 구성 요소 | 설명 |
|-----------|------|
| **Parquet 파일** | 실제 데이터가 저장되는 열 기반(Columnar) 파일 형식입니다. 압축 효율과 분석 성능이 뛰어납니다 |
| **_delta_log/** | 모든 변경 이력이 JSON 형식으로 기록되는 트랜잭션 로그 폴더입니다 |
| **커밋 로그 (.json)** | 각 트랜잭션(INSERT, UPDATE, DELETE)마다 하나의 JSON 파일이 생성됩니다 |
| **체크포인트 (.parquet)** | 10번의 커밋마다 자동 생성되며, 로그 읽기 성능을 최적화합니다 |

> 💡 **트랜잭션 로그 덕분에 가능한 기능들**: 타임 트래블(Time Travel), ACID 트랜잭션, 동시 읽기/쓰기, 스키마 변경 이력 추적이 모두 이 로그를 기반으로 동작합니다.

```sql
-- 테이블의 변경 이력 확인
DESCRIBE HISTORY catalog.schema.orders;

-- 타임 트래블: 과거 특정 버전의 데이터 조회
SELECT * FROM catalog.schema.orders VERSION AS OF 3;

-- 타임 트래블: 특정 시점의 데이터 조회
SELECT * FROM catalog.schema.orders TIMESTAMP AS OF '2025-01-15 10:00:00';
```

---

## 뷰(View) 유형

뷰는 데이터를 직접 저장하지 않고 **SQL 쿼리를 이름으로 저장** 하는 객체입니다. 복잡한 쿼리를 단순화하거나, 보안을 위해 특정 컬럼만 노출하는 데 유용합니다.

### 일반 뷰(View)

일반 뷰는 저장된 SQL 쿼리에 이름을 붙인 것입니다. 뷰를 조회할 때마다 **내부 쿼리가 매번 실행** 됩니다.

```sql
-- 활성 고객만 보여주는 뷰
CREATE VIEW catalog.schema.v_active_customers AS
SELECT
    customer_id,
    name,
    email,
    signup_date
FROM catalog.schema.customers
WHERE status = 'ACTIVE';

-- 뷰를 일반 테이블처럼 조회
SELECT * FROM catalog.schema.v_active_customers WHERE signup_date >= '2025-01-01';
```

> 💡 **뷰의 보안 활용**: 민감한 컬럼(주민번호, 비밀번호 등)을 제외한 뷰를 만들면, 원본 테이블 접근 권한 없이도 필요한 데이터만 안전하게 공유할 수 있습니다.

### 임시 뷰(Temporary View)

세션(Session) 동안만 유효한 뷰입니다. 세션이 종료되면 자동으로 삭제됩니다. Notebook이나 SQL Editor에서 중간 결과를 임시로 저장할 때 유용합니다.

```sql
-- 현재 세션에서만 유효한 임시 뷰
CREATE TEMPORARY VIEW temp_high_value_orders AS
SELECT * FROM catalog.schema.orders WHERE amount >= 10000;

-- 임시 뷰 조회
SELECT COUNT(*) FROM temp_high_value_orders;
-- 세션 종료 시 자동 삭제됩니다
```

### 구체화된 뷰(Materialized View)

구체화된 뷰는 쿼리 결과를 **물리적으로 미리 계산하여 저장** 합니다. 원본 데이터가 변경되면 **증분 방식으로 자동 갱신** 됩니다.

```sql
-- 일별 매출 집계를 미리 계산하여 저장
CREATE MATERIALIZED VIEW catalog.schema.mv_daily_revenue AS
SELECT
    DATE(order_date) AS day,
    COUNT(*)         AS order_count,
    SUM(amount)      AS total_revenue,
    AVG(amount)      AS avg_order_value
FROM catalog.schema.orders
GROUP BY DATE(order_date);

-- 구체화된 뷰를 조회하면 미리 계산된 결과를 즉시 반환합니다
SELECT * FROM catalog.schema.mv_daily_revenue WHERE day >= '2025-01-01';

-- 수동으로 새로고침
REFRESH MATERIALIZED VIEW catalog.schema.mv_daily_revenue;
```

> ⚠️ **구체화된 뷰는 SDP(선언적 파이프라인) 또는 SQL Warehouse에서만 생성** 할 수 있습니다. All-Purpose Cluster에서는 생성할 수 없습니다.

---

## 스트리밍 테이블(Streaming Table)

스트리밍 테이블은 **증분 데이터 처리(Incremental Processing)** 에 최적화된 테이블입니다. 새로 추가된 데이터만 처리하여 효율적으로 테이블을 갱신합니다.

```sql
-- 클라우드 스토리지에서 데이터를 증분 수집하는 스트리밍 테이블
CREATE STREAMING TABLE catalog.schema.st_raw_events AS
SELECT * FROM STREAM READ_FILES('s3://my-bucket/raw-events/', format => 'json');

-- 다른 스트리밍 테이블을 소스로 하는 스트리밍 테이블
CREATE STREAMING TABLE catalog.schema.st_clean_events AS
SELECT
    event_id,
    event_type,
    user_id,
    CAST(timestamp AS TIMESTAMP) AS event_time
FROM STREAM catalog.schema.st_raw_events
WHERE event_type IS NOT NULL;
```

---

## Materialized View vs Streaming Table 비교

이 두 가지는 모두 데이터를 물리적으로 저장하지만, 동작 방식이 근본적으로 다릅니다.

| 비교 항목 | Materialized View | Streaming Table |
|-----------|-------------------|-----------------|
| **데이터 처리** | 배치(Batch) 기반 | 증분(Incremental) 기반 |
| **갱신 방식** | 전체 또는 증분 재계산 | 새 데이터만 추가(Append-only) |
| **데이터 수정** | UPDATE, DELETE 반영 가능 | INSERT만 처리 (Append-only) |
| **소스 유형** | 배치 테이블, 뷰 | 스트리밍 소스 (Auto Loader, Kafka 등) |
| **사용 사례** | 집계, 요약 테이블 | 이벤트 수집, 로그 처리, CDC |
| **실행 환경** | SDP, SQL Warehouse | SDP |
| **쿼리 성능** | 미리 계산된 결과로 빠른 조회 | 실시간에 가까운 데이터 반영 |

> 💡 **선택 기준**: "이미 존재하는 데이터를 **요약/집계** 하고 싶다" → Materialized View. "새로 들어오는 데이터를 **계속 수집** 하고 싶다" → Streaming Table.

---

## 파티셔닝 전략

대용량 테이블에서 쿼리 성능을 높이려면, 데이터를 효율적으로 **분할(파티셔닝)** 하는 전략이 필요합니다.

### 기존 파티셔닝 (Hive-style Partitioning)

전통적인 파티셔닝은 특정 컬럼 값에 따라 **물리적으로 디렉토리를 나누는** 방식입니다.

```sql
-- 전통적 파티셔닝 (Hive-style) — 신규 테이블에는 권장하지 않습니다
CREATE TABLE catalog.schema.events_partitioned (
    event_id   BIGINT,
    event_type STRING,
    event_date DATE,
    payload    STRING
) USING DELTA
PARTITIONED BY (event_date);
```

> ⚠️ **기존 파티셔닝의 문제점**: 파티션 키를 잘못 선택하면 **소규모 파일 문제(Small File Problem)** 가 발생합니다. 예를 들어 카디널리티가 너무 높은 컬럼(user_id 등)으로 파티셔닝하면 수백만 개의 작은 파일이 생성되어 오히려 성능이 나빠집니다.

### Liquid Clustering (권장)

Liquid Clustering은 Databricks가 제공하는 **차세대 데이터 레이아웃 최적화** 기능입니다. 파티셔닝과 Z-Order를 대체하며, 데이터 분포에 따라 **자동으로 최적의 레이아웃을 조정** 합니다.

```sql
-- ✅ Liquid Clustering (권장) — 파티셔닝 대신 사용
CREATE TABLE catalog.schema.events_clustered (
    event_id   BIGINT,
    event_type STRING,
    event_date DATE,
    user_id    BIGINT,
    payload    STRING
) USING DELTA
CLUSTER BY (event_date, event_type);

-- 기존 테이블에 Liquid Clustering 적용
ALTER TABLE catalog.schema.events_old
CLUSTER BY (event_date, event_type);

-- Liquid Clustering 제거
ALTER TABLE catalog.schema.events_clustered
CLUSTER BY NONE;
```

| 비교 항목 | Hive-style 파티셔닝 | Liquid Clustering |
|-----------|-------------------|-------------------|
| **키 변경** | 전체 데이터 재작성 필요 | `ALTER TABLE CLUSTER BY`로 즉시 변경 |
| **키 개수** | 1~2개 권장 | 최대 4개 |
| **소규모 파일** | 파티션 설계에 크게 의존 | 자동 최적화 |
| **OPTIMIZE** | 별도 Z-Order 필요 | OPTIMIZE만 실행하면 자동 적용 |
| **신규 테이블** | 비권장 | **권장** |

---

## 테이블 속성 설정 (TBLPROPERTIES)

테이블 속성을 통해 Delta 테이블의 동작을 세밀하게 제어할 수 있습니다.

```sql
-- 테이블 생성 시 속성 지정
CREATE TABLE catalog.schema.audit_logs (
    log_id     BIGINT,
    action     STRING,
    created_at TIMESTAMP
) USING DELTA
TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',     -- 쓰기 시 파일 크기 자동 최적화
    'delta.autoOptimize.autoCompact'   = 'true',     -- 소규모 파일 자동 병합
    'delta.logRetentionDuration'       = '90 days',  -- 트랜잭션 로그 보존 기간
    'delta.deletedFileRetentionDuration' = '30 days' -- 삭제 파일 보존 (타임 트래블)
);

-- 기존 테이블의 속성 변경
ALTER TABLE catalog.schema.audit_logs
SET TBLPROPERTIES ('delta.logRetentionDuration' = '180 days');

-- 테이블 속성 확인
SHOW TBLPROPERTIES catalog.schema.audit_logs;
```

| 주요 속성 | 설명 | 기본값 |
|-----------|------|--------|
| `delta.autoOptimize.optimizeWrite` | 쓰기 시 파일 크기를 자동으로 최적화합니다 | `false` |
| `delta.autoOptimize.autoCompact` | 소규모 파일을 자동으로 병합합니다 | `false` |
| `delta.logRetentionDuration` | 트랜잭션 로그 보존 기간입니다 | `30 days` |
| `delta.deletedFileRetentionDuration` | 삭제된 파일 보존 기간(타임 트래블 범위)입니다 | `7 days` |
| `delta.columnMapping.mode` | 컬럼 이름 변경/삭제를 지원합니다 | `none` |
| `delta.enableDeletionVectors` | 삭제 벡터를 활성화하여 DELETE/UPDATE 성능을 향상시킵니다 | `true` |

---

## 테이블 생명주기 관리

테이블은 **생성 → 수정 → 최적화 → 정리/삭제** 순서로 관리합니다.

```sql
-- 1️⃣ 생성
CREATE TABLE catalog.schema.products (
    product_id   BIGINT GENERATED ALWAYS AS IDENTITY,
    product_name STRING NOT NULL,
    category     STRING,
    price        DECIMAL(10,2),
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
) USING DELTA
CLUSTER BY (category);

-- 2️⃣ 스키마 수정 (컬럼 추가)
ALTER TABLE catalog.schema.products ADD COLUMN discount_pct DECIMAL(5,2);

-- 컬럼 이름 변경 (Column Mapping 필요)
ALTER TABLE catalog.schema.products
SET TBLPROPERTIES ('delta.columnMapping.mode' = 'name');
ALTER TABLE catalog.schema.products RENAME COLUMN discount_pct TO discount_rate;

-- 3️⃣ 데이터 최적화
OPTIMIZE catalog.schema.products;

-- 오래된 파일 정리 (기본 7일 이전)
VACUUM catalog.schema.products;

-- 보존 기간 지정 (주의: 타임 트래블 범위가 줄어듭니다)
VACUUM catalog.schema.products RETAIN 72 HOURS;

-- 4️⃣ 테이블 삭제
DROP TABLE IF EXISTS catalog.schema.products;

-- 외부 테이블의 경우 데이터는 수동 삭제 필요
-- DROP TABLE은 메타데이터만 삭제합니다
```

> ⚠️ **VACUUM 주의사항**: `VACUUM`은 보존 기간보다 오래된 파일을 영구 삭제합니다. 실행 후에는 해당 시점 이전으로 타임 트래블할 수 없으므로, 보존 기간을 신중하게 설정하셔야 합니다.

---

## 선택 가이드: 언제 어떤 타입을 쓸 것인가?

| 시나리오 | 권장 타입 | 이유 |
|----------|----------|------|
| 일반적인 비즈니스 데이터 저장 | **관리형 테이블** | Unity Catalog의 완전한 거버넌스를 활용할 수 있습니다 |
| 외부 시스템과 데이터 공유 | **외부 테이블** | 다른 도구가 같은 스토리지 경로를 참조할 수 있습니다 |
| 복잡한 쿼리를 이름으로 저장 | **일반 뷰** | 스토리지 비용 없이 쿼리를 재사용할 수 있습니다 |
| 보안 목적으로 컬럼 제한 | **일반 뷰** | 민감한 컬럼을 숨기고 필요한 데이터만 노출합니다 |
| Notebook 내 임시 중간 결과 | **임시 뷰** | 세션 종료 시 자동 정리되어 깔끔합니다 |
| 자주 조회되는 집계/요약 | **구체화된 뷰** | 미리 계산된 결과로 쿼리 성능이 비약적으로 향상됩니다 |
| 실시간 이벤트/로그 수집 | **스트리밍 테이블** | 증분 처리로 효율적이며 지연 시간이 짧습니다 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **관리형 테이블** | Unity Catalog가 데이터와 메타데이터를 모두 관리합니다. 대부분의 경우 권장됩니다 |
| **외부 테이블** | 데이터는 사용자 지정 경로에, 메타데이터만 Unity Catalog가 관리합니다 |
| **Delta 테이블 구조** | Parquet 파일 + 트랜잭션 로그(_delta_log)로 ACID, 타임 트래블을 지원합니다 |
| **일반 뷰** | SQL 쿼리의 별칭으로, 매번 실행 시 재계산됩니다 |
| **구체화된 뷰** | 쿼리 결과를 물리적으로 저장하고 증분 갱신합니다 |
| **스트리밍 테이블** | 새 데이터만 증분으로 처리하는 Append-only 테이블입니다 |
| **Liquid Clustering** | 파티셔닝과 Z-Order를 대체하는 차세대 데이터 레이아웃 최적화 기능입니다 |
| **TBLPROPERTIES** | 테이블의 동작(자동 최적화, 보존 기간 등)을 세밀하게 제어합니다 |

---

## 참고 링크

- [Databricks: Tables](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-tables.html)
- [Databricks: Managed and external tables](https://docs.databricks.com/aws/en/data-governance/unity-catalog/create-tables.html)
- [Databricks: Views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-views.html)
- [Databricks: Materialized Views](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-materialized-views.html)
- [Databricks: Streaming Tables](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-streaming-tables.html)
- [Databricks: Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering.html)
- [Databricks: Delta Lake table properties](https://docs.databricks.com/aws/en/delta/table-properties.html)
- [Azure Databricks: Tables and views](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/sql-ref-tables)
