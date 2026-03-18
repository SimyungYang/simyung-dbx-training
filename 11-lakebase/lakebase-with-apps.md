# Databricks Apps와 Lakebase 연동

## 활용 시나리오

Lakebase는 **Databricks Apps**의 백엔드 데이터베이스로 이상적입니다. Streamlit, Gradio, FastAPI로 만든 앱이 Lakebase에서 데이터를 읽고 쓸 수 있습니다.

```mermaid
graph LR
    U["👤 사용자"] --> APP["🌐 Databricks App<br/>(Streamlit)"]
    APP -->|"CRUD"| LB["Lakebase<br/>(PostgreSQL)"]
    LB -->|"자동 동기화"| DL["Delta Lake"]
    DL --> DASH["📊 대시보드"]
```

---

## 예시: Streamlit 앱에서 Lakebase 사용

```python
import streamlit as st
import psycopg2

@st.cache_resource
def get_connection():
    return psycopg2.connect(
        host=st.secrets["lakebase_host"],
        dbname="mydb",
        user=st.secrets["user"],
        password=st.secrets["token"]
    )

conn = get_connection()

# 데이터 조회
df = pd.read_sql("SELECT * FROM orders ORDER BY created_at DESC LIMIT 100", conn)
st.dataframe(df)

# 데이터 입력
with st.form("new_order"):
    product = st.text_input("상품명")
    amount = st.number_input("금액")
    if st.form_submit_button("주문 등록"):
        cursor = conn.cursor()
        cursor.execute("INSERT INTO orders (product, amount) VALUES (%s, %s)", (product, amount))
        conn.commit()
        st.success("주문이 등록되었습니다!")
```

---

## 참고 링크

- [Databricks: Databricks Apps](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/)
- [Databricks: Lakebase with Apps](https://docs.databricks.com/aws/en/lakebase/)
