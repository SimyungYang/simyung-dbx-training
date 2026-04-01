# 성능 최적화와 CRUD 앱 예제

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
| **쿼리 캐싱**| Streamlit `@cache_data`, Redis | DB 부하 50~90% 감소 |
| **인덱스**| 쿼리 패턴에 맞는 B-tree/GIN 인덱스 | 조회 속도 10~100배 향상 |
| **페이지네이션**| LIMIT/OFFSET 또는 커서 기반 | 대용량 데이터 안정적 조회 |
| **커넥션 풀링**| psycopg2 pool, SQLAlchemy pool | 연결 오버헤드 제거 |
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
