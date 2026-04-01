# 프레임워크별 연결 예제

## 왜 Databricks Apps + Lakebase인가요?

웹 애플리케이션을 만들 때 가장 큰 고민 중 하나는 "**데이터를 어디에 저장하고, 어떻게 분석까지 연결할 것인가?**"입니다. 기존에는 앱의 백엔드 DB(PostgreSQL 등)와 분석 플랫폼(Databricks)을 별도로 운영하고, 그 사이에 복잡한 ETL 파이프라인을 구축해야 했습니다.

**Databricks Apps + Lakebase**조합을 사용하면, 앱 개발부터 데이터 분석까지 ** 하나의 플랫폼**에서 처리할 수 있습니다.

| 기존 방식 | Databricks Apps + Lakebase |
|-----------|---------------------------|
| 앱 서버 별도 호스팅 (AWS EC2, Heroku 등) | Databricks Apps에서 호스팅 |
| DB 별도 운영 (RDS, Cloud SQL 등) | Lakebase로 통합 |
| ETL 파이프라인 직접 구축 | Data Sync로 자동 동기화 |
| 인증 체계 분리 (앱 인증 + DB 인증 + 분석 인증) | Databricks OAuth 하나로 통합 |
| 거버넌스 분리 | Unity Catalog 통합 거버넌스 |

> 💡 **Databricks Apps**: Databricks 워크스페이스 내에서 **Streamlit, Dash, Flask, FastAPI, Gradio**등 Python 기반 웹 앱을 직접 배포하고 호스팅할 수 있는 서비스입니다. 별도의 서버 관리 없이 앱을 운영할 수 있습니다.

---

## 통합 아키텍처

| 계층 | 구성 요소 | 역할 |
|------|-----------|------|
| ** 사용자**| 웹 브라우저 | HTTPS로 앱에 접속합니다 |
| **Databricks Apps**| Python 앱 (Streamlit/FastAPI) | 웹 애플리케이션을 실행합니다 |
|  | Databricks OAuth | 자동 인증을 처리합니다 |
| **Lakebase**| PostgreSQL 호환 DB | 인증된 연결로 데이터를 읽고 씁니다 |
| ** 분석 환경**| Delta Lake | Data Sync로 동기화된 데이터를 분석합니다 |

이 아키텍처의 핵심 장점은 ** 인증이 자동으로 처리된다는 것**입니다. Databricks Apps에서 실행되는 앱은 워크스페이스의 OAuth 인증을 자동으로 상속받아, 별도의 비밀번호 관리가 필요 없습니다.

---

## 프레임워크별 연결 예제

### Streamlit 앱

Streamlit은 데이터 앱을 가장 빠르게 만들 수 있는 프레임워크입니다. Lakebase와의 연결이 매우 간단합니다.

**프로젝트 구조**:

| 파일 | 설명 |
|------|------|
| `app.py` | 메인 앱 코드 |
| `requirements.txt` | Python 의존성 |
| `app.yaml` | Databricks Apps 설정 |
| `.streamlit/secrets.toml` | 시크릿 (로컬 개발용) |

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
