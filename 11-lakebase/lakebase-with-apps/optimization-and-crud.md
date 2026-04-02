# 성능 최적화와 CRUD 패턴

## 1. 왜 Lakebase에서 최적화가 중요한가

### OLTP vs OLAP 특성 차이

Databricks는 본래 대규모 분석(OLAP, Online Analytical Processing) 워크로드에 최적화된 플랫폼입니다.
Lakebase는 그 위에 **OLTP (Online Transaction Processing)** 특성을 더한 관리형 PostgreSQL로, 두 세계를 연결하는 역할을 합니다.

| 특성 | OLAP (Delta / Spark) | OLTP (Lakebase / PostgreSQL) |
|------|----------------------|------------------------------|
| 주요 작업 | 대규모 집계, 스캔 | 단건 삽입/조회/수정/삭제 |
| 트랜잭션 | 제한적 (ACID on Delta) | 완전한 ACID 트랜잭션 |
| 동시 접속 | 수십~수백 Spark job | 수십~수천 클라이언트 연결 |
| 지연 시간 | 초~분 단위 허용 | 밀리초 단위 요구 |
| 인덱스 | 파일 통계 기반 | B-tree, Hash, GIN, GiST 등 |

### Databricks 관리형이라는 특징

Lakebase는 단순한 PostgreSQL이 아니라 **Databricks Unity Catalog** 아래에서 동작하는 관리형 서비스입니다.
이 때문에 다음과 같은 제약과 이점이 동시에 존재합니다.

- **이점**: 자동 백업, Unity Catalog 거버넌스, Delta 테이블과의 양방향 동기화 (Synced Tables)
- **제약**: 직접 OS/스토리지 레벨 접근 불가, `pg_hba.conf` 등 일부 서버 파라미터 변경 제한
- **네트워크**: Databricks Apps와 동일 VPC 내에서 Private Link로 저지연 통신

따라서 최적화 전략은 "PostgreSQL 표준 방법"을 기반으로 하되, **관리형 환경의 제약을 인식하고 적용**해야 합니다.

> 참고: [Databricks Lakebase 공식 문서](https://docs.databricks.com/en/database/index.html)

---

## 2. CRUD 작업 패턴

### INSERT — 삽입 최적화

단건 INSERT는 네트워크 왕복(round-trip)이 발생할 때마다 오버헤드가 누적됩니다.
**배치 처리 (batch insert)** 를 활용하면 성능을 크게 향상시킬 수 있습니다.

```python
import psycopg2
import psycopg2.extras

conn = psycopg2.connect(...)

# ❌ 나쁜 예: 루프 내 단건 INSERT (네트워크 왕복 N번)
for row in data:
    cur.execute("INSERT INTO events (user_id, action) VALUES (%s, %s)", row)

# ✅ 좋은 예: executemany로 배치 INSERT
records = [(row["user_id"], row["action"]) for row in data]
psycopg2.extras.execute_values(
    cur,
    "INSERT INTO events (user_id, action) VALUES %s",
    records,
    page_size=500   # 500건씩 묶어서 전송
)
conn.commit()
```

```sql
-- 대량 데이터 로드 시: COPY 명령이 가장 빠름
-- (Databricks Apps에서 파일 기반으로 사용하는 경우)
COPY events (user_id, action, created_at)
FROM '/tmp/events.csv'
WITH (FORMAT csv, HEADER true);
```

### SELECT — 조회 최적화

```python
# ✅ 파라미터 바인딩으로 SQL Injection 방지 + 실행 계획 재사용
cur.execute(
    "SELECT id, name, tier FROM customers WHERE tier = %s AND created_at > %s",
    ("premium", "2025-01-01")
)

# ✅ 커서 기반 페이지네이션 (OFFSET 대신 keyset pagination 권장)
# OFFSET은 뒤로 갈수록 느려짐 — 대규모 테이블에서 치명적
cur.execute(
    """
    SELECT id, name, created_at FROM orders
    WHERE created_at < %s          -- 마지막 조회 커서
    ORDER BY created_at DESC
    LIMIT 50
    """,
    (last_seen_cursor,)
)

# ❌ 나쁜 예: SELECT * + 대용량 OFFSET
cur.execute("SELECT * FROM orders ORDER BY id OFFSET 10000 LIMIT 50")
```

### UPDATE — 수정 최적화

```sql
-- ✅ 배치 UPDATE: 조건을 명확히 지정하여 인덱스 활용
UPDATE customers
SET tier = 'premium', updated_at = NOW()
WHERE id = ANY(ARRAY[1001, 1002, 1003]);  -- IN 보다 ANY(ARRAY[...])가 플래너 친화적

-- ✅ UPSERT (INSERT ON CONFLICT): 중복 처리를 DB 레벨에서 해결
INSERT INTO customer_stats (customer_id, total_orders, last_order_at)
VALUES (%(cid)s, %(total)s, %(last)s)
ON CONFLICT (customer_id) DO UPDATE
  SET total_orders = EXCLUDED.total_orders,
      last_order_at = EXCLUDED.last_order_at;

-- ❌ 나쁜 예: 인덱스 없는 컬럼으로 UPDATE → Sequential Scan
UPDATE orders SET status = 'shipped' WHERE memo LIKE '%urgent%';
```

### DELETE — 삭제 최적화

```sql
-- ✅ 소프트 삭제 (soft delete) 패턴 권장
-- 실제 DELETE 대신 is_deleted 플래그 설정 → 인덱스 활용, 복구 가능
ALTER TABLE customers ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;
ALTER TABLE customers ADD COLUMN deleted_at TIMESTAMPTZ;

UPDATE customers SET is_deleted = TRUE, deleted_at = NOW() WHERE id = %s;

-- ✅ 정기 하드 삭제가 필요하다면 배치로 처리
DELETE FROM audit_logs
WHERE created_at < NOW() - INTERVAL '90 days'
  AND id IN (SELECT id FROM audit_logs WHERE created_at < NOW() - INTERVAL '90 days' LIMIT 1000);
-- LIMIT으로 한 번에 삭제할 건수를 제한 → 락(lock) 경합 방지
```

---

## 3. 인덱스 전략

### PostgreSQL 인덱스 유형

| 인덱스 유형 | 적합한 사용처 | 예시 |
|-------------|---------------|------|
| **B-tree** | 등호/범위 조건, 정렬 | `WHERE id = ?`, `ORDER BY created_at` |
| **Hash** | 등호 조건만 (범위 불가) | `WHERE email = ?` (등호 전용) |
| **GIN** | 배열, JSONB, 전문 검색 | `WHERE tags @> '{python}'`, `LIKE` with trigram |
| **GiST** | 지리 정보, 범위 타입 | PostGIS 좌표, `tsrange` 겹침 검사 |

```sql
-- B-tree: 복합 인덱스 — 자주 함께 쓰이는 컬럼은 함께 묶기
-- (컬럼 순서가 중요: 카디널리티 높은 것을 앞에)
CREATE INDEX idx_orders_customer_date
    ON orders (customer_id, created_at DESC);

-- Hash: 등호 조건만 있는 고빈도 조회 컬럼
CREATE INDEX idx_sessions_token
    ON user_sessions USING hash (session_token);

-- GIN + trigram: ILIKE 기반 텍스트 검색 가속
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm
    ON products USING gin (name gin_trgm_ops);

-- GIN: JSONB 컬럼 내 키 검색
CREATE INDEX idx_events_payload
    ON events USING gin (payload jsonb_path_ops);

-- 부분 인덱스 (partial index): 조건에 해당하는 행만 인덱싱 → 크기 절감
CREATE INDEX idx_orders_pending
    ON orders (created_at)
    WHERE status = 'pending';
```

### 인덱스 설계 가이드

```sql
-- 인덱스 사용 현황 조회: 사용 안 되는 인덱스는 DROP 검토
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan        AS times_used,
    idx_tup_read    AS tuples_read,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;  -- 사용 횟수 낮은 인덱스 식별

-- 테이블 크기 vs 인덱스 크기 비율 확인
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    pg_size_pretty(pg_indexes_size(oid))        AS indexes_size
FROM pg_class
WHERE relkind = 'r'
ORDER BY pg_total_relation_size(oid) DESC;
```

> 인덱스가 많을수록 INSERT/UPDATE/DELETE 속도가 저하됩니다. 쿼리 패턴을 먼저 분석하고 필요한 인덱스만 생성하십시오.

---

## 4. 연결 관리 (Connection Management)

### Connection Pooling의 필요성

PostgreSQL은 연결(connection)마다 별도 프로세스를 생성합니다. Databricks Apps처럼 다수의 요청이 동시에 들어오는 환경에서는 **커넥션 풀링 (connection pooling)** 이 필수입니다.

```python
# SQLAlchemy Connection Pool 설정 예시
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg2://user:password@host:5432/dbname",
    pool_size=5,          # 유지할 기본 연결 수
    max_overflow=10,      # pool_size 초과 시 최대 추가 연결 수
    pool_timeout=30,      # 연결 대기 타임아웃 (초)
    pool_recycle=1800,    # 30분마다 연결 재생성 (stale connection 방지)
    pool_pre_ping=True,   # 사용 전 연결 유효성 확인
    connect_args={"sslmode": "require"}
)
```

```python
# psycopg2 ThreadedConnectionPool 사용 예시 (멀티스레드 환경)
from psycopg2 import pool as pg_pool
import streamlit as st

@st.cache_resource
def get_pool():
    return pg_pool.ThreadedConnectionPool(
        minconn=2,
        maxconn=10,
        host=st.secrets["lakebase"]["host"],
        port=5432,
        dbname=st.secrets["lakebase"]["dbname"],
        user=st.secrets["lakebase"]["user"],
        password=st.secrets["lakebase"]["password"],
        sslmode="require"
    )

def execute_query(sql, params=None):
    pool = get_pool()
    conn = pool.getconn()
    try:
        with conn.cursor() as cur:
            cur.execute(sql, params)
            conn.commit()
            return cur.fetchall()
    except Exception as e:
        conn.rollback()
        raise e
    finally:
        pool.putconn(conn)  # 반드시 반환
```

### Databricks Apps에서의 연결 패턴

Databricks Apps는 기본적으로 단일 프로세스 내에서 실행됩니다. 아래 원칙을 따르십시오.

- `@st.cache_resource` 로 연결 풀을 **앱 수명 동안 한 번만** 생성
- 요청마다 연결을 생성/소멸하지 않음 (오버헤드 발생)
- **Lakebase 연결 수 상한** 은 인스턴스 크기에 따라 다르며, 기본적으로 수백 개 수준
- 연결이 오래 유휴 상태로 남으면 서버 측에서 끊길 수 있으므로 `pool_pre_ping=True` 권장

> 참고: [PostgreSQL Connection Pooling Best Practices](https://www.postgresql.org/docs/current/runtime-config-connection.html)

---

## 5. 데이터 동기화 — Delta ↔ Lakebase (Synced Tables)

### Synced Tables 개요

**Synced Tables** 는 Delta 테이블의 데이터를 Lakebase PostgreSQL 테이블로 자동 동기화하는 기능입니다.
분석용 데이터(Delta)를 OLTP 앱(Lakebase)에서 빠르게 조회할 수 있도록 연결해 줍니다.

```
Delta Table (Unity Catalog)
        │
        │  Synced Table (자동 동기화)
        ▼
Lakebase PostgreSQL Table
        │
        │  SQL 조회
        ▼
Databricks App (Streamlit 등)
```

### Synced Tables 생성

```sql
-- Lakebase 내에서 Synced Table 생성
-- Unity Catalog의 Delta 테이블을 PostgreSQL 테이블로 동기화
CREATE SYNCED TABLE product_catalog
FROM catalog.schema.products_delta
WITH (sync_mode = 'INCREMENTAL');  -- FULL 또는 INCREMENTAL
```

```python
# Python SDK로 Synced Table 관리
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Synced Table 생성
w.lakebase.create_sync(
    database_name="mydb",
    source_table="catalog.schema.products",
    target_table="product_catalog",
    sync_mode="INCREMENTAL"
)

# 동기화 상태 확인
status = w.lakebase.get_sync(database_name="mydb", sync_name="product_catalog")
print(status.state)  # RUNNING, COMPLETED, FAILED
```

### 실시간 vs 배치 동기화 비교

| 방식 | 동기화 주기 | 사용 사례 | 비고 |
|------|-------------|-----------|------|
| **Incremental Sync** | 변경분만 자동 감지 | 제품 카탈로그, 사용자 프로필 | 권장 기본값 |
| **Full Sync** | 전체 재적재 | 소규모 참조 테이블 | 데이터 양 적을 때만 |
| **수동 트리거** | 필요 시 호출 | 배치 작업 후 즉시 반영 | Job에서 SDK 호출 |

> 참고: [Databricks Synced Tables 문서](https://docs.databricks.com/en/database/synced-tables.html)

---

## 6. 성능 튜닝

### 쿼리 실행 계획 분석 (EXPLAIN)

```sql
-- EXPLAIN: 실행 계획만 확인 (실제 실행 X)
EXPLAIN
SELECT * FROM orders WHERE customer_id = 1001 AND status = 'pending';

-- EXPLAIN ANALYZE: 실제 실행 후 실측값 포함
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, c.name, o.total
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > '2025-01-01'
ORDER BY o.created_at DESC
LIMIT 100;
```

실행 계획에서 주의해야 할 키워드:
- `Seq Scan` — 인덱스 미사용, 풀 스캔 → 인덱스 추가 검토
- `Nested Loop` — 작은 결과셋에 유리, 대용량에는 `Hash Join`이 나을 수 있음
- `cost=...` — 예상 비용 (rows, width 포함)
- `actual time=...` — 실제 소요 시간 (ms)

### 파티셔닝 (Partitioning)

시계열 데이터나 수억 건 이상의 대형 테이블에는 **테이블 파티셔닝 (table partitioning)** 을 적용합니다.

```sql
-- 범위 파티셔닝 (Range Partitioning): 날짜 기반
CREATE TABLE events (
    id          BIGSERIAL,
    user_id     INT NOT NULL,
    action      TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- 월별 파티션 생성
CREATE TABLE events_2025_01
    PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE events_2025_02
    PARTITION OF events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- 파티션 키에 자동으로 인덱스가 적용됨
-- 특정 월 조회 시 해당 파티션만 스캔 (Partition Pruning)
```

### 벌크 로드 최적화

```python
# Delta 테이블 → Lakebase 대량 이관 시
# Spark에서 JDBC bulk write 활용
df = spark.table("catalog.schema.large_table")

df.write \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://host:5432/dbname") \
    .option("dbtable", "target_table") \
    .option("user", "user") \
    .option("password", "password") \
    .option("batchsize", 10000) \
    .option("numPartitions", 8) \
    .mode("append") \
    .save()
```

---

## 7. 장단점과 트레이드오프

### Lakebase vs 직접 RDS 운영

| 항목 | Lakebase | AWS RDS (PostgreSQL) |
|------|----------|----------------------|
| **관리 부담** | 낮음 (완전 관리형) | 중간 (파라미터, 패치 직접) |
| **Delta 연동** | 네이티브 Synced Tables | 별도 ETL 파이프라인 필요 |
| **Unity Catalog 거버넌스** | 자동 적용 | 별도 구성 필요 |
| **커스터마이징** | 제한적 (OS 접근 불가) | 높음 (파라미터 전체 제어) |
| **비용** | Databricks 과금 체계 | EC2/스토리지 직접 비용 |
| **확장성** | 수직 확장 (인스턴스 업그레이드) | 읽기 복제본, Multi-AZ 선택 가능 |

### 비용 고려사항

- Lakebase는 **DBU (Databricks Unit)** 기반으로 과금되므로, 항상 켜두는 OLTP DB로 사용 시 비용이 누적됩니다.
- 트래픽이 낮은 시간대에 **자동 일시 중지 (auto-pause)** 설정을 활용하십시오.
- Synced Tables는 동기화 빈도에 따라 추가 연산 비용이 발생합니다.

---

## 8. 베스트 프랙티스와 흔한 실수

### 베스트 프랙티스

```python
# 1. 항상 컨텍스트 매니저(with)로 트랜잭션 관리
with conn:  # 예외 발생 시 자동 rollback, 정상 시 commit
    with conn.cursor() as cur:
        cur.execute("INSERT INTO orders (...) VALUES (...)", data)

# 2. 파라미터 바인딩 — f-string 직접 삽입 금지
# ❌ SQL Injection 취약
cur.execute(f"SELECT * FROM users WHERE name = '{name}'")

# ✅ 파라미터 바인딩
cur.execute("SELECT * FROM users WHERE name = %s", (name,))

# 3. 쿼리 캐싱: 자주 바뀌지 않는 데이터는 앱 레벨에서 캐싱
@st.cache_data(ttl=300)  # 5분 캐시
def get_product_categories():
    with engine.connect() as conn:
        return pd.read_sql("SELECT DISTINCT category FROM products ORDER BY 1", conn)

# 4. 커서 기반 페이지네이션
@st.cache_data(ttl=30)
def get_recent_orders(last_id=None, page_size=50):
    query = "SELECT * FROM orders"
    params = []
    if last_id:
        query += " WHERE id < %s"
        params.append(last_id)
    query += " ORDER BY id DESC LIMIT %s"
    params.append(page_size)
    with engine.connect() as conn:
        return pd.read_sql(query, conn, params=params)
```

### 흔한 실수

| 실수 | 문제점 | 올바른 방법 |
|------|--------|-------------|
| `SELECT *` 남용 | 불필요한 컬럼 전송, 인덱스 무력화 | 필요한 컬럼만 명시 |
| 루프 내 단건 INSERT | N번 네트워크 왕복 | `execute_values`로 배치 처리 |
| 연결 매번 생성/소멸 | 연결 오버헤드 누적 | Connection Pool 사용 |
| `OFFSET` 기반 페이지네이션 | 뒤로 갈수록 Full Scan | Keyset Pagination 사용 |
| 인덱스 없는 컬럼으로 필터 | Sequential Scan | 쿼리 패턴에 맞는 인덱스 생성 |
| 트랜잭션 미처리 | 중간 실패 시 데이터 불일치 | `with conn` 컨텍스트 매니저 사용 |
| `conn.rollback()` 누락 | 오류 후 커넥션이 비정상 상태 유지 | `try/except/finally` 패턴 준수 |

---

## 전체 CRUD 앱 예제: 고객 관리 시스템

아래는 위 모든 원칙을 적용한 완전한 Streamlit CRUD 앱입니다.

```python
# customer_app.py — 고객 관리 CRUD 앱 (최적화 적용)
import streamlit as st
import psycopg2
import psycopg2.extras
import psycopg2.pool
import pandas as pd

st.set_page_config(page_title="고객 관리", page_icon="👥", layout="wide")

@st.cache_resource
def get_pool():
    """앱 수명 동안 커넥션 풀을 한 번만 생성합니다."""
    return psycopg2.pool.ThreadedConnectionPool(
        minconn=2, maxconn=10,
        host=st.secrets["lakebase"]["host"],
        port=5432,
        dbname=st.secrets["lakebase"]["dbname"],
        user=st.secrets["lakebase"]["user"],
        password=st.secrets["lakebase"]["password"],
        sslmode="require"
    )

def get_conn():
    return get_pool().getconn()

def release_conn(conn):
    get_pool().putconn(conn)

@st.cache_data(ttl=300)
def get_tier_list():
    """등급 목록 — 5분 캐시."""
    conn = get_conn()
    try:
        return pd.read_sql("SELECT DISTINCT tier FROM customers ORDER BY 1", conn)["tier"].tolist()
    finally:
        release_conn(conn)

st.title("👥 고객 관리 시스템")
tab_list, tab_create, tab_edit = st.tabs(["📋 고객 목록", "➕ 신규 등록", "✏️ 정보 수정"])

# --- 조회 (Read) ---
with tab_list:
    search = st.text_input("🔍 고객 검색 (이름 또는 이메일)")
    conn = get_conn()
    try:
        if search:
            df = pd.read_sql(
                "SELECT id, name, email, tier, created_at FROM customers "
                "WHERE name ILIKE %s OR email ILIKE %s ORDER BY created_at DESC",
                conn, params=(f"%{search}%", f"%{search}%")
            )
        else:
            df = pd.read_sql(
                "SELECT id, name, email, tier, created_at FROM customers "
                "ORDER BY created_at DESC LIMIT 100",
                conn
            )
    finally:
        release_conn(conn)
    st.dataframe(df, use_container_width=True)
    st.caption(f"총 {len(df)}명 표시 중")

# --- 생성 (Create) ---
with tab_create:
    with st.form("create_customer"):
        name  = st.text_input("이름")
        email = st.text_input("이메일")
        tier  = st.selectbox("등급", get_tier_list())
        if st.form_submit_button("등록", type="primary"):
            conn = get_conn()
            try:
                with conn:
                    with conn.cursor() as cur:
                        cur.execute(
                            "INSERT INTO customers (name, email, tier) VALUES (%s, %s, %s)",
                            (name, email, tier)
                        )
                st.success(f"✅ {name}님이 등록되었습니다!")
            except psycopg2.errors.UniqueViolation:
                st.error("❌ 이미 등록된 이메일입니다.")
            finally:
                release_conn(conn)

# --- 수정 / 삭제 (Update / Delete) ---
with tab_edit:
    customer_id = st.number_input("고객 ID", min_value=1, step=1)
    conn = get_conn()
    try:
        with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
            cur.execute(
                "SELECT id, name, email, tier FROM customers WHERE id = %s",
                (customer_id,)
            )
            customer = cur.fetchone()
    finally:
        release_conn(conn)

    if customer:
        with st.form("edit_customer"):
            new_name = st.text_input("이름", value=customer["name"])
            tier_opts = get_tier_list()
            new_tier = st.selectbox(
                "등급", tier_opts,
                index=tier_opts.index(customer["tier"]) if customer["tier"] in tier_opts else 0
            )
            col1, col2 = st.columns(2)
            with col1:
                if st.form_submit_button("수정 저장", type="primary"):
                    conn = get_conn()
                    try:
                        with conn:
                            with conn.cursor() as cur:
                                cur.execute(
                                    "UPDATE customers SET name=%s, tier=%s, updated_at=NOW() WHERE id=%s",
                                    (new_name, new_tier, customer_id)
                                )
                        st.success("✅ 수정되었습니다!")
                    finally:
                        release_conn(conn)
            with col2:
                if st.form_submit_button("삭제 (소프트)", type="secondary"):
                    conn = get_conn()
                    try:
                        with conn:
                            with conn.cursor() as cur:
                                cur.execute(
                                    "UPDATE customers SET is_deleted=TRUE, deleted_at=NOW() WHERE id=%s",
                                    (customer_id,)
                                )
                        st.warning("🗑️ 삭제 처리되었습니다.")
                    finally:
                        release_conn(conn)
    else:
        st.info("조회된 고객이 없습니다. ID를 확인하십시오.")
```

---

## 참고 자료

- [Databricks Lakebase 공식 문서](https://docs.databricks.com/en/database/index.html)
- [Databricks Synced Tables](https://docs.databricks.com/en/database/synced-tables.html)
- [PostgreSQL Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- [PostgreSQL Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [psycopg2 Connection Pool](https://www.psycopg.org/docs/pool.html)
- [SQLAlchemy Engine Configuration](https://docs.sqlalchemy.org/en/20/core/engines.html)
