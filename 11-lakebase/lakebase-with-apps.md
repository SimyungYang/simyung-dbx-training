# Databricks Apps와 Lakebase 연동

## 왜 Databricks Apps + Lakebase인가요?

웹 애플리케이션을 만들 때 가장 큰 고민 중 하나는 **"데이터를 어디에 저장하고, 어떻게 분석까지 연결할 것인가?"**입니다. 기존에는 앱의 백엔드 DB(PostgreSQL 등)와 분석 플랫폼(Databricks)을 별도로 운영하고, 그 사이에 복잡한 ETL 파이프라인을 구축해야 했습니다.

**Databricks Apps + Lakebase** 조합을 사용하면, 앱 개발부터 데이터 분석까지 **하나의 플랫폼**에서 처리할 수 있습니다.

| 기존 방식 | Databricks Apps + Lakebase |
|-----------|---------------------------|
| 앱 서버 별도 호스팅 (AWS EC2, Heroku 등) | Databricks Apps에서 호스팅 |
| DB 별도 운영 (RDS, Cloud SQL 등) | Lakebase로 통합 |
| ETL 파이프라인 직접 구축 | Data Sync로 자동 동기화 |
| 인증 체계 분리 (앱 인증 + DB 인증 + 분석 인증) | Databricks OAuth 하나로 통합 |
| 거버넌스 분리 | Unity Catalog 통합 거버넌스 |

> 💡 **Databricks Apps**: Databricks 워크스페이스 내에서 **Streamlit, Dash, Flask, FastAPI, Gradio** 등 Python 기반 웹 앱을 직접 배포하고 호스팅할 수 있는 서비스입니다. 별도의 서버 관리 없이 앱을 운영할 수 있습니다.

---

## 통합 아키텍처

| 계층 | 구성 요소 | 역할 |
|------|-----------|------|
| **사용자** | 웹 브라우저 | HTTPS로 앱에 접속합니다 |
| **Databricks Apps** | Python 앱 (Streamlit/FastAPI) | 웹 애플리케이션을 실행합니다 |
|  | Databricks OAuth | 자동 인증을 처리합니다 |
| **Lakebase** | PostgreSQL 호환 DB | 인증된 연결로 데이터를 읽고 씁니다 |
| **분석 환경** | Delta Lake | Data Sync로 동기화된 데이터를 분석합니다 |

이 아키텍처의 핵심 장점은 **인증이 자동으로 처리된다는 것**입니다. Databricks Apps에서 실행되는 앱은 워크스페이스의 OAuth 인증을 자동으로 상속받아, 별도의 비밀번호 관리가 필요 없습니다.

---

## 프레임워크별 연결 예제

### Streamlit 앱

Streamlit은 데이터 앱을 가장 빠르게 만들 수 있는 프레임워크입니다. Lakebase와의 연결이 매우 간단합니다.

```python
# app.py — Streamlit + Lakebase 기본 연결
import streamlit as st
import psycopg2
import pandas as pd

st.set_page_config(page_title="주문 관리", layout="wide")
st.title("📦 주문 관리 시스템")

# 커넥션 풀링 (앱 전체에서 재사용)
@st.cache_resource
def get_connection():
    """Lakebase 연결을 생성하고 캐싱합니다."""
    return psycopg2.connect(
        host=st.secrets["lakebase"]["host"],
        port=5432,
        dbname=st.secrets["lakebase"]["dbname"],
        user=st.secrets["lakebase"]["user"],
        password=st.secrets["lakebase"]["password"],
        sslmode="require"
    )

conn = get_connection()

# --- 데이터 조회 ---
st.header("최근 주문")
df = pd.read_sql("""
    SELECT o.id, c.name AS customer, o.product, 
           o.amount, o.status, o.created_at
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    ORDER BY o.created_at DESC
    LIMIT 50
""", conn)
st.dataframe(df, use_container_width=True)

# --- 주문 등록 폼 ---
st.header("새 주문 등록")
with st.form("order_form"):
    col1, col2 = st.columns(2)
    with col1:
        customer_id = st.number_input("고객 ID", min_value=1, step=1)
        product = st.text_input("상품명")
    with col2:
        amount = st.number_input("금액 (원)", min_value=0, step=1000)
        status = st.selectbox("상태", ["pending", "confirmed", "shipped"])
    
    submitted = st.form_submit_button("주문 등록", type="primary")
    if submitted and product:
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO orders (customer_id, product, amount, status) "
            "VALUES (%s, %s, %s, %s)",
            (customer_id, product, amount, status)
        )
        conn.commit()
        st.success(f"✅ 주문이 등록되었습니다: {product} (₩{amount:,.0f})")
        st.rerun()  # 페이지 새로고침으로 최신 데이터 표시

# --- 매출 요약 ---
st.header("📊 매출 요약")
summary = pd.read_sql("""
    SELECT 
        status,
        COUNT(*) AS order_count,
        SUM(amount) AS total_amount
    FROM orders
    GROUP BY status
""", conn)
st.bar_chart(summary.set_index("status")["total_amount"])
```

**Streamlit secrets 설정** (`secrets.toml`):

```toml
[lakebase]
host = "app-db-xxxx.lakebase.databricks.com"
dbname = "shop_db"
user = "token"
password = "dapi_your_token"
```

### FastAPI 앱

REST API 백엔드가 필요한 경우 FastAPI를 사용합니다. 비동기 처리와 자동 API 문서 생성이 장점입니다.

```python
# app.py — FastAPI + Lakebase REST API
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from contextlib import asynccontextmanager
import psycopg2
from psycopg2 import pool
from typing import Optional
import os

# 커넥션 풀 초기화
db_pool = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    """앱 시작 시 커넥션 풀 생성, 종료 시 정리"""
    global db_pool
    db_pool = pool.ThreadedConnectionPool(
        minconn=2,
        maxconn=10,
        host=os.environ["LAKEBASE_HOST"],
        port=5432,
        dbname=os.environ["LAKEBASE_DBNAME"],
        user="token",
        password=os.environ["LAKEBASE_TOKEN"],
        sslmode="require"
    )
    yield
    db_pool.closeall()

app = FastAPI(title="주문 API", lifespan=lifespan)

# --- 모델 정의 ---
class OrderCreate(BaseModel):
    customer_id: int
    product: str
    amount: float
    status: Optional[str] = "pending"

class OrderResponse(BaseModel):
    id: int
    customer_id: int
    product: str
    amount: float
    status: str

# --- API 엔드포인트 ---
@app.get("/orders", response_model=list[OrderResponse])
def list_orders(limit: int = 50, status: Optional[str] = None):
    """주문 목록을 조회합니다."""
    conn = db_pool.getconn()
    try:
        cursor = conn.cursor()
        if status:
            cursor.execute(
                "SELECT id, customer_id, product, amount, status "
                "FROM orders WHERE status = %s "
                "ORDER BY created_at DESC LIMIT %s",
                (status, limit)
            )
        else:
            cursor.execute(
                "SELECT id, customer_id, product, amount, status "
                "FROM orders ORDER BY created_at DESC LIMIT %s",
                (limit,)
            )
        rows = cursor.fetchall()
        return [
            OrderResponse(id=r[0], customer_id=r[1], product=r[2],
                         amount=r[3], status=r[4])
            for r in rows
        ]
    finally:
        db_pool.putconn(conn)

@app.post("/orders", response_model=OrderResponse, status_code=201)
def create_order(order: OrderCreate):
    """새 주문을 생성합니다."""
    conn = db_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO orders (customer_id, product, amount, status) "
            "VALUES (%s, %s, %s, %s) RETURNING id",
            (order.customer_id, order.product, order.amount, order.status)
        )
        order_id = cursor.fetchone()[0]
        conn.commit()
        return OrderResponse(
            id=order_id, customer_id=order.customer_id,
            product=order.product, amount=order.amount,
            status=order.status
        )
    except Exception as e:
        conn.rollback()
        raise HTTPException(status_code=400, detail=str(e))
    finally:
        db_pool.putconn(conn)

@app.patch("/orders/{order_id}/status")
def update_order_status(order_id: int, status: str):
    """주문 상태를 업데이트합니다."""
    conn = db_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE orders SET status = %s, updated_at = CURRENT_TIMESTAMP "
            "WHERE id = %s RETURNING id",
            (status, order_id)
        )
        if cursor.fetchone() is None:
            raise HTTPException(status_code=404, detail="주문을 찾을 수 없습니다")
        conn.commit()
        return {"message": f"주문 {order_id} 상태가 '{status}'로 변경되었습니다"}
    finally:
        db_pool.putconn(conn)
```

### Dash 앱

인터랙티브 차트와 대시보드가 필요한 경우 Dash를 사용합니다.

```python
# app.py — Dash + Lakebase 실시간 대시보드
from dash import Dash, html, dcc, callback, Output, Input
import plotly.express as px
import pandas as pd
import psycopg2

app = Dash(__name__)

def get_connection():
    return psycopg2.connect(
        host="app-db-xxxx.lakebase.databricks.com",
        port=5432, dbname="shop_db",
        user="token", password="dapi_your_token",
        sslmode="require"
    )

app.layout = html.Div([
    html.H1("📊 실시간 매출 대시보드"),
    dcc.Interval(id='interval', interval=30*1000),  # 30초마다 갱신
    dcc.Graph(id='revenue-chart'),
    dcc.Graph(id='status-chart')
])

@callback(
    Output('revenue-chart', 'figure'),
    Output('status-chart', 'figure'),
    Input('interval', 'n_intervals')
)
def update_charts(n):
    conn = get_connection()
    
    # 시간대별 매출
    revenue_df = pd.read_sql("""
        SELECT DATE_TRUNC('hour', created_at) AS hour,
               SUM(amount) AS revenue
        FROM orders
        WHERE created_at >= CURRENT_DATE
        GROUP BY 1 ORDER BY 1
    """, conn)
    
    # 상태별 주문 수
    status_df = pd.read_sql("""
        SELECT status, COUNT(*) AS count
        FROM orders GROUP BY status
    """, conn)
    
    conn.close()
    
    fig1 = px.line(revenue_df, x='hour', y='revenue',
                   title='시간대별 매출')
    fig2 = px.pie(status_df, values='count', names='status',
                  title='주문 상태 분포')
    return fig1, fig2

if __name__ == '__main__':
    app.run(debug=False, host='0.0.0.0', port=8050)
```

---

## OAuth 인증 패턴

Databricks Apps에서 Lakebase에 접근할 때, 두 가지 인증 패턴을 사용할 수 있습니다.

### 앱 인증 (Service Principal)

앱 자체가 하나의 서비스 계정으로 인증합니다. 모든 사용자의 요청이 동일한 권한으로 처리됩니다.

```python
import os
from databricks.sdk import WorkspaceClient

# Databricks Apps 환경에서는 자동으로 서비스 프린시펄 인증이 설정됩니다
w = WorkspaceClient()

# Lakebase 크레덴셜 생성
credential = w.lakebase.generate_credential(
    database_name="my_catalog.my_schema.shop_db"
)

# 생성된 크레덴셜로 연결
conn = psycopg2.connect(
    host=credential.host,
    port=credential.port,
    dbname=credential.database,
    user=credential.username,
    password=credential.password,
    sslmode="require"
)
```

### 사용자 인증 (User Passthrough)

각 사용자의 Databricks 인증을 그대로 Lakebase에 전달합니다. 사용자별로 다른 권한을 적용할 수 있습니다.

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | 사용자 → Databricks App | Databricks OAuth로 인증합니다 |
| 2 | Databricks App → Lakebase | 사용자 토큰을 전달합니다 |
| 3 | Lakebase → Unity Catalog | Unity Catalog 권한을 확인합니다 |

| 인증 패턴 | 장점 | 단점 | 적합한 상황 |
|-----------|------|------|-------------|
| **앱 인증** | 설정 간단, 커넥션 풀 공유 | 사용자별 권한 구분 불가 | 내부 도구, 모든 사용자 동일 권한 |
| **사용자 인증** | 세밀한 권한 제어, 감사 로그 | 커넥션 풀 공유 어려움 | 멀티 테넌트 앱, 규정 준수 필수 |

---

## 커넥션 풀링 설정

프로덕션 앱에서는 **커넥션 풀링(Connection Pooling)** 이 필수입니다. 매 요청마다 새 연결을 생성하면 성능이 크게 저하됩니다.

> 💡 **커넥션 풀링(Connection Pooling)**: 데이터베이스 연결을 미리 여러 개 만들어 놓고, 요청이 올 때마다 재사용하는 기법입니다. 연결 생성/종료의 오버헤드를 줄여 응답 속도를 크게 향상시킵니다.

### psycopg2 ThreadedConnectionPool

```python
from psycopg2 import pool

# 앱 시작 시 풀 초기화
connection_pool = pool.ThreadedConnectionPool(
    minconn=2,      # 최소 연결 수
    maxconn=20,     # 최대 연결 수
    host="app-db-xxxx.lakebase.databricks.com",
    port=5432,
    dbname="shop_db",
    user="token",
    password="dapi_your_token",
    sslmode="require"
)

# 요청 처리 시
def handle_request():
    conn = connection_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM orders LIMIT 10")
        results = cursor.fetchall()
        return results
    finally:
        connection_pool.putconn(conn)  # 풀에 연결 반환
```

### 풀 크기 가이드라인

| 동시 사용자 수 | minconn | maxconn | 설명 |
|----------------|---------|---------|------|
| ~10명 | 2 | 5 | 개발/테스트 환경입니다 |
| 10~50명 | 5 | 15 | 소규모 프로덕션입니다 |
| 50~200명 | 10 | 30 | 중규모 프로덕션입니다 |
| 200명+ | 20 | 50+ | 대규모 프로덕션, PgBouncer 검토가 필요합니다 |

> ⚠️ **maxconn 과다 설정 주의**: 커넥션 풀의 최대 크기를 Lakebase 인스턴스의 최대 연결 수보다 크게 설정하면 연결 실패가 발생할 수 있습니다. 여러 앱이 동일한 Lakebase에 접속하는 경우, 전체 연결 수를 고려하여 설정하시기 바랍니다.

---

## 트랜잭션 관리

OLTP 앱에서 **트랜잭션(Transaction)** 관리는 데이터 정합성의 핵심입니다.

```python
def transfer_order(conn, order_id, new_status):
    """주문 상태 변경과 재고 업데이트를 트랜잭션으로 처리합니다."""
    try:
        cursor = conn.cursor()
        
        # 트랜잭션 시작 (autocommit=False가 기본)
        cursor.execute(
            "UPDATE orders SET status = %s WHERE id = %s RETURNING product",
            (new_status, order_id)
        )
        result = cursor.fetchone()
        if result is None:
            raise ValueError(f"주문 {order_id}을 찾을 수 없습니다")
        
        product = result[0]
        
        # 상태가 'shipped'이면 재고 차감
        if new_status == 'shipped':
            cursor.execute(
                "UPDATE inventory SET stock = stock - 1 "
                "WHERE product = %s AND stock > 0 RETURNING stock",
                (product,)
            )
            if cursor.fetchone() is None:
                raise ValueError(f"재고 부족: {product}")
        
        conn.commit()  # 모든 변경사항 확정
        return {"success": True, "message": f"주문 {order_id} → {new_status}"}
    
    except Exception as e:
        conn.rollback()  # 오류 시 모든 변경사항 취소
        return {"success": False, "error": str(e)}
```

> 💡 **ACID 트랜잭션**: Lakebase는 PostgreSQL과 동일한 ACID(원자성, 일관성, 격리성, 지속성) 트랜잭션을 지원합니다. `commit()`이 호출될 때까지 변경사항은 다른 연결에서 보이지 않으며, `rollback()`으로 모든 변경을 취소할 수 있습니다.

---

## 에러 처리 패턴

프로덕션 앱에서는 체계적인 에러 처리가 필수입니다.

```python
import psycopg2
from psycopg2 import errors
import logging

logger = logging.getLogger(__name__)

def safe_db_operation(pool, query, params=None):
    """안전한 데이터베이스 작업 패턴"""
    conn = None
    try:
        conn = pool.getconn()
        cursor = conn.cursor()
        cursor.execute(query, params)
        
        if query.strip().upper().startswith("SELECT"):
            result = cursor.fetchall()
        else:
            conn.commit()
            result = cursor.rowcount
        
        return {"success": True, "data": result}
    
    except errors.UniqueViolation as e:
        # 중복 키 오류 (예: 이메일 중복)
        if conn: conn.rollback()
        logger.warning(f"중복 데이터: {e}")
        return {"success": False, "error": "이미 존재하는 데이터입니다"}
    
    except errors.ForeignKeyViolation as e:
        # 외래 키 참조 오류
        if conn: conn.rollback()
        logger.warning(f"참조 무결성 위반: {e}")
        return {"success": False, "error": "참조하는 데이터가 존재하지 않습니다"}
    
    except errors.OperationalError as e:
        # 연결 오류 (네트워크, 타임아웃 등)
        logger.error(f"DB 연결 오류: {e}")
        if conn:
            pool.putconn(conn, close=True)  # 불량 연결 폐기
            conn = None
        return {"success": False, "error": "데이터베이스 연결 오류가 발생했습니다"}
    
    except Exception as e:
        if conn: conn.rollback()
        logger.error(f"예상치 못한 오류: {e}")
        return {"success": False, "error": "처리 중 오류가 발생했습니다"}
    
    finally:
        if conn:
            pool.putconn(conn)
```

---

## 성능 최적화

### 쿼리 캐싱

자주 조회하는 데이터는 캐싱하여 Lakebase 부하를 줄입니다.

```python
import streamlit as st
import pandas as pd

# Streamlit의 TTL 캐싱 활용 (5분간 캐시)
@st.cache_data(ttl=300)
def get_product_categories():
    """상품 카테고리 목록을 캐싱합니다 (5분)."""
    conn = get_connection()
    return pd.read_sql("SELECT DISTINCT category FROM products ORDER BY 1", conn)

# 사용자별로 캐시가 필요 없는 데이터
@st.cache_data(ttl=30)
def get_recent_orders(limit=50):
    """최근 주문 목록을 캐싱합니다 (30초)."""
    conn = get_connection()
    return pd.read_sql(
        f"SELECT * FROM orders ORDER BY created_at DESC LIMIT {limit}",
        conn
    )
```

### 인덱스 설계

앱의 쿼리 패턴에 맞는 인덱스를 설계합니다.

```sql
-- 자주 사용하는 필터링 조건에 인덱스 생성
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at DESC);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- 텍스트 검색이 필요한 경우
CREATE INDEX idx_products_name_trgm ON products 
USING gin(name gin_trgm_ops);

-- 인덱스 사용 상태 확인
SELECT
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

| 최적화 항목 | 방법 | 효과 |
|-------------|------|------|
| **쿼리 캐싱** | Streamlit `@cache_data`, Redis | DB 부하 50~90% 감소 |
| **인덱스** | 쿼리 패턴에 맞는 B-tree/GIN 인덱스 | 조회 속도 10~100배 향상 |
| **페이지네이션** | LIMIT/OFFSET 또는 커서 기반 | 대용량 데이터 안정적 조회 |
| **커넥션 풀링** | psycopg2 pool, SQLAlchemy pool | 연결 오버헤드 제거 |
| **배치 INSERT** | executemany, COPY | 대량 삽입 속도 10배 향상 |

---

## CRUD 앱 예제: 고객 관리 시스템

다음은 Streamlit으로 만든 완전한 CRUD(생성, 조회, 수정, 삭제) 앱 예제입니다.

```python
# customer_app.py — 고객 관리 CRUD 앱
import streamlit as st
import psycopg2
import pandas as pd

st.set_page_config(page_title="고객 관리", page_icon="👥", layout="wide")

@st.cache_resource
def get_conn():
    return psycopg2.connect(
        host=st.secrets["lakebase"]["host"],
        port=5432,
        dbname=st.secrets["lakebase"]["dbname"],
        user=st.secrets["lakebase"]["user"],
        password=st.secrets["lakebase"]["password"],
        sslmode="require"
    )

conn = get_conn()

st.title("👥 고객 관리 시스템")

# 탭 구성
tab_list, tab_create, tab_edit = st.tabs(["📋 고객 목록", "➕ 신규 등록", "✏️ 정보 수정"])

# --- 조회 (Read) ---
with tab_list:
    search = st.text_input("🔍 고객 검색 (이름 또는 이메일)")
    if search:
        df = pd.read_sql(
            "SELECT * FROM customers WHERE name ILIKE %s OR email ILIKE %s "
            "ORDER BY created_at DESC",
            conn, params=(f"%{search}%", f"%{search}%")
        )
    else:
        df = pd.read_sql(
            "SELECT * FROM customers ORDER BY created_at DESC LIMIT 100",
            conn
        )
    st.dataframe(df, use_container_width=True)
    st.caption(f"총 {len(df)}명 표시 중")

# --- 생성 (Create) ---
with tab_create:
    with st.form("create_customer"):
        name = st.text_input("이름")
        email = st.text_input("이메일")
        tier = st.selectbox("등급", ["standard", "premium", "vip"])
        
        if st.form_submit_button("등록", type="primary"):
            try:
                cursor = conn.cursor()
                cursor.execute(
                    "INSERT INTO customers (name, email, tier) "
                    "VALUES (%s, %s, %s)",
                    (name, email, tier)
                )
                conn.commit()
                st.success(f"✅ {name}님이 등록되었습니다!")
            except psycopg2.errors.UniqueViolation:
                conn.rollback()
                st.error("❌ 이미 등록된 이메일입니다.")

# --- 수정 (Update) ---
with tab_edit:
    customer_id = st.number_input("수정할 고객 ID", min_value=1, step=1)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM customers WHERE id = %s", (customer_id,))
    customer = cursor.fetchone()
    
    if customer:
        with st.form("edit_customer"):
            new_name = st.text_input("이름", value=customer[1])
            new_tier = st.selectbox(
                "등급",
                ["standard", "premium", "vip"],
                index=["standard", "premium", "vip"].index(customer[3])
            )
            col1, col2 = st.columns(2)
            with col1:
                if st.form_submit_button("수정 저장", type="primary"):
                    cursor.execute(
                        "UPDATE customers SET name=%s, tier=%s WHERE id=%s",
                        (new_name, new_tier, customer_id)
                    )
                    conn.commit()
                    st.success("✅ 수정되었습니다!")
            with col2:
                if st.form_submit_button("삭제", type="secondary"):
                    cursor.execute(
                        "DELETE FROM customers WHERE id=%s",
                        (customer_id,)
                    )
                    conn.commit()
                    st.warning("🗑️ 삭제되었습니다.")
    else:
        st.info("고객 ID를 입력하세요.")
```

---

## 배포 및 운영 팁

### Databricks Apps 배포

```bash
# 프로젝트 구조
my-app/
├── app.py              # 메인 앱 코드
├── requirements.txt    # Python 의존성
├── app.yaml           # Databricks Apps 설정
└── .streamlit/
    └── secrets.toml   # 시크릿 (로컬 개발용)
```

**app.yaml 설정**:

```yaml
# app.yaml — Databricks Apps 배포 설정
command:
  - "streamlit"
  - "run"
  - "app.py"
  - "--server.port=8501"
  - "--server.address=0.0.0.0"

env:
  - name: LAKEBASE_HOST
    value: "app-db-xxxx.lakebase.databricks.com"
  - name: LAKEBASE_DBNAME
    value: "shop_db"
```

### 운영 체크리스트

| 항목 | 확인 사항 |
|------|-----------|
| **커넥션 풀** | 적절한 크기로 설정되어 있는지 확인합니다 |
| **에러 처리** | 모든 DB 작업에 try/except가 있는지 확인합니다 |
| **SQL 인젝션 방지** | 파라미터 바인딩(`%s`)을 사용하는지 확인합니다 |
| **타임아웃 설정** | 쿼리 타임아웃과 연결 타임아웃이 설정되어 있는지 확인합니다 |
| **로깅** | 에러와 느린 쿼리가 로깅되는지 확인합니다 |
| **헬스체크** | `/health` 엔드포인트에서 DB 연결 상태를 확인합니다 |

> ⚠️ **SQL 인젝션 방지**: 사용자 입력을 SQL 쿼리에 직접 넣으면 보안 취약점이 됩니다. 반드시 **파라미터 바인딩**(`%s` 플레이스홀더)을 사용하시기 바랍니다. `f"SELECT * FROM users WHERE id={user_input}"` 형태는 절대 사용하지 마세요.

---

## 정리

| 핵심 포인트 | 설명 |
|-------------|------|
| **통합 플랫폼** | Databricks Apps + Lakebase로 앱 개발~분석까지 하나의 플랫폼에서 처리합니다 |
| **다양한 프레임워크** | Streamlit, FastAPI, Dash, Flask 등 Python 프레임워크를 자유롭게 선택합니다 |
| **OAuth 통합 인증** | 별도의 인증 시스템 없이 Databricks OAuth를 활용합니다 |
| **커넥션 풀링** | 프로덕션에서는 반드시 커넥션 풀을 설정합니다 |
| **자동 동기화** | 앱 데이터가 Data Sync로 자동으로 분석 환경에 반영됩니다 |
| **보안** | SQL 인젝션 방지, SSL 필수, 파라미터 바인딩을 준수합니다 |

---

## 참고 링크

- [Databricks: Databricks Apps](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/)
- [Databricks: Lakebase](https://docs.databricks.com/aws/en/lakebase/)
- [Azure Databricks: Databricks Apps](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/databricks-apps/)
- [Streamlit Documentation](https://docs.streamlit.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [psycopg2 Documentation](https://www.psycopg.org/docs/)
