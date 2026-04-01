# OAuth 인증과 커넥션 풀링

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
| **앱 인증**| 설정 간단, 커넥션 풀 공유 | 사용자별 권한 구분 불가 | 내부 도구, 모든 사용자 동일 권한 |
| **사용자 인증**| 세밀한 권한 제어, 감사 로그 | 커넥션 풀 공유 어려움 | 멀티 테넌트 앱, 규정 준수 필수 |

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
