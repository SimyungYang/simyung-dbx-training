# Lakebase 설정

## 왜 설정 가이드가 필요한가요?

Lakebase는 Databricks가 완전 관리하는 PostgreSQL 호환 데이터베이스이지만, **인스턴스 생성, 연결 설정, 사용자 관리, 모니터링** 등 초기 설정을 올바르게 해야 안정적으로 운영할 수 있습니다. 이 문서에서는 Lakebase를 처음부터 끝까지 설정하는 전체 과정을 단계별로 안내합니다.

> 💡 ** 관리형 데이터베이스(Managed Database)**: 서버 프로비저닝, 패치, 백업, 장애 복구 등 인프라 운영을 클라우드 제공자가 대신 처리하는 데이터베이스 서비스입니다. 사용자는 데이터와 쿼리에만 집중할 수 있습니다.

---

## 전체 설정 흐름

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 인스턴스 생성 | Lakebase 인스턴스를 생성합니다 |
| 2 | 연결 설정 | 연결 정보를 구성합니다 |
| 3 | 테이블 생성 | 필요한 테이블을 생성합니다 |
| 4 | 사용자/역할 관리 | 접근 권한을 설정합니다 |
| 5 | 모니터링 설정 | 모니터링을 구성합니다 |
| 6 | 백업/복구 확인 | 백업 및 복구 절차를 확인합니다 |
| 7 | 운영 준비 완료 | 프로덕션 운영이 가능합니다 |

---

## 1단계: 인스턴스 생성

Lakebase 데이터베이스는 세 가지 방법으로 생성할 수 있습니다.

### UI를 통한 생성

1. Databricks 워크스페이스에서 **Catalog Explorer** 를 엽니다
2. 좌측 메뉴에서 원하는 **카탈로그 > 스키마** 를 선택합니다
3. **Create** → **Lakebase Database** 를 클릭합니다
4. 데이터베이스 이름을 입력하고 **Create** 를 클릭합니다

### SQL을 통한 생성

```sql
-- 기본 Lakebase 데이터베이스 생성
CREATE DATABASE my_catalog.my_schema.app_db
TYPE LAKEBASE;

-- 생성 확인
DESCRIBE DATABASE my_catalog.my_schema.app_db;

-- 결과 예시:
-- name: app_db
-- type: LAKEBASE
-- state: RUNNING
-- host: app-db-xxxx.lakebase.databricks.com
-- port: 5432
```

### Python SDK를 통한 생성

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Lakebase 데이터베이스 생성
db = w.lakebase.create_database(
    catalog_name="my_catalog",
    schema_name="my_schema",
    name="app_db"
)

print(f"데이터베이스 생성 완료: {db.name}")
print(f"호스트: {db.host}")
print(f"포트: {db.port}")
print(f"상태: {db.state}")
```

> 💡 **생성 시간**: Lakebase 인스턴스는 일반적으로 **1~3분** 내에 생성됩니다. 상태가 `PROVISIONING`에서 `RUNNING`으로 바뀌면 사용 가능합니다.

---

## 2단계: 연결 설정

Lakebase는 표준 **PostgreSQL 프로토콜** 을 지원하므로, 기존 PostgreSQL 클라이언트와 드라이버를 그대로 사용할 수 있습니다.

### 연결 정보 확인

| 항목 | 값 |
|------|-----|
| **호스트** | `<database-name>-xxxx.lakebase.databricks.com` |
| ** 포트** | `5432` (PostgreSQL 표준 포트) |
| ** 데이터베이스명** | 생성 시 지정한 이름 (예: `app_db`) |
| ** 인증** | Databricks Personal Access Token 또는 OAuth |

### psql (CLI) 연결

```bash
# Personal Access Token으로 연결
psql "host=app-db-xxxx.lakebase.databricks.com \
      port=5432 \
      dbname=app_db \
      user=token \
      password=dapi_your_personal_access_token \
      sslmode=require"

# 연결 테스트
SELECT version();
-- 결과: PostgreSQL 16.x (Lakebase)
```

### Python (psycopg2) 연결

```python
import psycopg2

# 기본 연결
conn = psycopg2.connect(
    host="app-db-xxxx.lakebase.databricks.com",
    port=5432,
    dbname="app_db",
    user="token",
    password="dapi_your_personal_access_token",
    sslmode="require"
)

# 연결 테스트
cursor = conn.cursor()
cursor.execute("SELECT version()")
print(cursor.fetchone())
# 출력: ('PostgreSQL 16.x (Lakebase)',)

# 테이블 생성 예제
cursor.execute("""
    CREATE TABLE IF NOT EXISTS products (
        product_id  SERIAL PRIMARY KEY,
        name        VARCHAR(200) NOT NULL,
        category    VARCHAR(100),
        price       DECIMAL(10,2) NOT NULL,
        stock       INTEGER DEFAULT 0,
        created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
""")
conn.commit()
print("테이블 생성 완료!")
```

### Python (SQLAlchemy) 연결

```python
from sqlalchemy import create_engine

# SQLAlchemy 엔진 생성
engine = create_engine(
    "postgresql+psycopg2://token:dapi_your_token@"
    "app-db-xxxx.lakebase.databricks.com:5432/app_db",
    connect_args={"sslmode": "require"},
    pool_size=5,        # 커넥션 풀 크기
    max_overflow=10,    # 최대 추가 연결 수
    pool_timeout=30     # 풀 대기 타임아웃 (초)
)

# pandas와 함께 사용
import pandas as pd
df = pd.read_sql("SELECT * FROM products LIMIT 10", engine)
print(df)
```

### JDBC 연결 (Java/Scala)

```
jdbc:postgresql://app-db-xxxx.lakebase.databricks.com:5432/app_db?
  user=token&
  password=dapi_your_token&
  sslmode=require
```

> ⚠️ **SSL 필수**: Lakebase는 보안을 위해 **SSL 연결만 허용** 합니다. 연결 시 `sslmode=require`를 반드시 설정하시기 바랍니다.

---

## 3단계: 인스턴스 사이징 가이드

Lakebase는 **오토스케일링** 을 지원하지만, 초기 워크로드 특성에 맞는 적절한 시작점을 선택하는 것이 좋습니다.

| 워크로드 규모 | 동시 연결 수 | 데이터 크기 | 권장 사항 |
|---------------|-------------|-------------|-----------|
| **소규모** (PoC/개발) | 1~10 | 1GB 미만 | 기본 설정으로 시작합니다 |
| ** 중규모** (소규모 프로덕션) | 10~100 | 1~50GB | 커넥션 풀링을 설정합니다 |
| ** 대규모** (프로덕션) | 100~1,000 | 50GB~1TB | 커넥션 풀링 + 읽기 전용 복제본을 검토합니다 |
| ** 초대규모** | 1,000+ | 1TB+ | Databricks 팀과 사이징 상담을 권장합니다 |

> 🆕 ** 오토스케일링 (GA)**: Lakebase는 워크로드에 따라 자동으로 컴퓨팅 리소스를 확장/축소합니다. 스토리지는 최대 **8TB** 까지 자동 확장됩니다. 별도의 스케일링 작업이 필요 없습니다.

---

## 4단계: 사용자 및 역할 관리

Lakebase는 **Unity Catalog** 와 통합되어, Databricks의 사용자 및 그룹 체계를 그대로 사용합니다.

### Unity Catalog 기반 권한 관리

```sql
-- 특정 사용자에게 데이터베이스 사용 권한 부여
GRANT USAGE ON DATABASE my_catalog.my_schema.app_db
TO `user@company.com`;

-- 특정 그룹에게 테이블 읽기 권한 부여
GRANT SELECT ON TABLE my_catalog.my_schema.app_db.products
TO `data_analysts`;

-- 특정 그룹에게 테이블 쓰기 권한 부여
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE my_catalog.my_schema.app_db.orders
TO `app_developers`;

-- 권한 확인
SHOW GRANTS ON TABLE my_catalog.my_schema.app_db.products;
```

### 역할 기반 접근 제어 패턴

| 역할 | 권한 | 대상 사용자 |
|------|------|-------------|
| **읽기 전용** | SELECT | 데이터 분석가, BI 사용자 |
| ** 읽기/쓰기** | SELECT, INSERT, UPDATE, DELETE | 앱 개발자, 백엔드 서비스 |
| ** 관리자** | ALL PRIVILEGES | DBA, 플랫폼 관리자 |

> 💡 **Unity Catalog 통합의 장점**: Lakebase의 권한은 Unity Catalog에서 중앙 관리됩니다. OLTP 데이터와 Delta Lake 데이터의 권한을 하나의 체계로 관리할 수 있어, 거버넌스 복잡도가 크게 줄어듭니다.

---

## 5단계: OAuth 인증 설정

프로덕션 환경에서는 Personal Access Token 대신 **OAuth** 를 사용하는 것을 권장합니다.

### 서비스 프린시펄(Service Principal) 인증

```python
import psycopg2
from databricks.sdk import WorkspaceClient

# Databricks SDK를 통해 OAuth 토큰 획득
w = WorkspaceClient()
token = w.tokens.create(
    comment="Lakebase app connection",
    lifetime_seconds=3600  # 1시간
)

# OAuth 토큰으로 Lakebase 연결
conn = psycopg2.connect(
    host="app-db-xxxx.lakebase.databricks.com",
    port=5432,
    dbname="app_db",
    user="token",
    password=token.token_value,
    sslmode="require"
)
```

| 인증 방식 | 적합한 환경 | 보안 수준 |
|-----------|-------------|-----------|
| **Personal Access Token** | 개발, 테스트 | 중간 |
| **OAuth (Service Principal)** | 프로덕션 앱 | 높음 |
| **OAuth (User-to-Machine)** | 대화형 앱 | 높음 |

> ⚠️ **Personal Access Token 주의**: PAT는 만료일을 설정하더라도 유출 위험이 있습니다. 프로덕션 환경에서는 반드시 **OAuth 인증** 또는 ** 서비스 프린시펄**을 사용하시기 바랍니다.

---

## 6단계: 백업 및 복구

### 자동 백업

Lakebase는 ** 자동으로 지속적인 백업**을 수행합니다. 별도의 백업 설정이 필요 없습니다.

| 기능 | 설명 |
|------|------|
| ** 자동 백업** | 지속적으로 자동 수행됩니다. 별도 설정 불필요합니다 |
| ** 보존 기간** | 최대 **35일** 동안 백업이 유지됩니다 |
| **Point-in-Time Recovery** | 보존 기간 내의 ** 임의 시점**으로 복원할 수 있습니다 |
| ** 복원 소요 시간** | 데이터 크기에 따라 수 분~수십 분이 소요됩니다 |

### Point-in-Time Recovery

```python
from databricks.sdk import WorkspaceClient
from datetime import datetime, timedelta

w = WorkspaceClient()

# 2시간 전 시점으로 복원
restore_time = datetime.utcnow() - timedelta(hours=2)

restored_db = w.lakebase.restore_database(
    catalog_name="my_catalog",
    schema_name="my_schema",
    name="app_db_restored",
    source_database="app_db",
    restore_point=restore_time.isoformat()
)

print(f"복원 완료: {restored_db.name}")
print(f"복원 시점: {restore_time}")
```

> 💡 ** 복원은 새 인스턴스로 생성됩니다**: Point-in-Time Recovery는 기존 데이터베이스를 덮어쓰지 않고, ** 새로운 데이터베이스 인스턴스**를 생성합니다. 이를 통해 원본 데이터의 안전성을 보장합니다.

---

## 7단계: 모니터링 및 메트릭

### 주요 모니터링 지표

| 메트릭 | 설명 | 주의 기준 |
|--------|------|-----------|
| **active_connections** | 현재 활성 연결 수 | 최대 연결 수의 80% 초과 시 |
| **cpu_utilization** | CPU 사용률 | 80% 이상 지속 시 |
| **storage_used_bytes** | 사용 중인 스토리지 | 7TB 이상 시 (최대 8TB) |
| **query_latency_p99** | 99번째 백분위 쿼리 지연 | 요구사항 대비 초과 시 |
| **replication_lag** | Data Sync 복제 지연 | 목표 SLA 초과 시 |

### SQL로 상태 확인

```sql
-- 현재 활성 연결 확인 (Lakebase에 직접 접속하여 실행)
SELECT count(*) AS active_connections
FROM pg_stat_activity
WHERE state = 'active';

-- 테이블별 크기 확인
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(tablename::regclass) DESC;

-- 느린 쿼리 확인 (5초 이상)
SELECT
    query,
    calls,
    mean_exec_time / 1000 AS avg_seconds,
    total_exec_time / 1000 AS total_seconds
FROM pg_stat_statements
WHERE mean_exec_time > 5000
ORDER BY total_exec_time DESC
LIMIT 10;
```

---

## 8단계: 네트워크 설정

### Private Link (프라이빗 연결)

프로덕션 환경에서는 **Private Link** 를 통해 인터넷을 거치지 않고 Lakebase에 접근하는 것을 권장합니다.

| 연결 방식 | 트래픽 경로 | 보안 수준 | 적합한 환경 |
|-----------|-------------|-----------|-------------|
| **퍼블릭 엔드포인트** | 인터넷 (SSL 암호화) | 중간 | 개발, PoC |
| **Private Link** | 클라우드 내부 네트워크 | 높음 | 프로덕션 |

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| ** 앱 서버 (앱 VPC)** | 클라이언트 | 앱에서 Lakebase에 접근합니다 |
| **Private Link** | 프라이빗 연결 | 인터넷을 경유하지 않는 안전한 연결입니다 |
| **Private Endpoint** | 엔드포인트 | 프라이빗 네트워크 엔드포인트입니다 |
| **Lakebase (Databricks VPC)** | 데이터베이스 | Private Endpoint를 통해 접근합니다 |

> ⚠️ **Private Link 설정**: Private Link 구성에는 클라우드 네트워크 관리자의 협조가 필요합니다. AWS의 경우 VPC Endpoint, Azure의 경우 Private Endpoint를 설정해야 합니다.

---

## 9단계: 비용 관리

### 비용 구성 요소

| 항목 | 과금 기준 | 절약 팁 |
|------|-----------|---------|
| ** 컴퓨팅** | DBU (Databricks Unit) 사용량 | 오토스케일링이 자동으로 유휴 시간 비용을 절감합니다 |
| ** 스토리지** | GB 단위 월별 과금 | 불필요한 데이터를 정기적으로 정리합니다 |
| **Data Sync** | 동기화된 데이터 볼륨 | TRIGGERED 모드로 비용을 줄일 수 있습니다 |
| ** 네트워크** | 리전 간 데이터 전송 | 같은 리전에 앱과 Lakebase를 배치합니다 |

### 비용 절감 체크리스트

1. ** 개발 환경은 TRIGGERED 모드** 사용 — CONTINUOUS는 프로덕션에서만 필요합니다
2. ** 불필요한 인덱스 정리** — 인덱스가 많으면 스토리지와 쓰기 비용이 증가합니다
3. ** 오래된 데이터 아카이빙** — Delta Lake로 동기화된 데이터는 Lakebase에서 삭제할 수 있습니다
4. **Branching 정리** — 사용하지 않는 브랜치는 삭제하여 스토리지를 절약합니다

---

## 실습: 처음부터 끝까지 설정하기

다음은 이커머스 앱의 백엔드로 Lakebase를 설정하는 전체 과정입니다.

```python
from databricks.sdk import WorkspaceClient
import psycopg2

# 1. 워크스페이스 클라이언트 초기화
w = WorkspaceClient()

# 2. Lakebase 데이터베이스 생성
db = w.lakebase.create_database(
    catalog_name="production",
    schema_name="ecommerce",
    name="shop_db"
)
print(f"✅ 데이터베이스 생성: {db.host}:{db.port}")

# 3. 연결
conn = psycopg2.connect(
    host=db.host,
    port=db.port,
    dbname="shop_db",
    user="token",
    password="dapi_your_token",
    sslmode="require"
)
cursor = conn.cursor()

# 4. 테이블 생성
cursor.execute("""
    CREATE TABLE customers (
        id          SERIAL PRIMARY KEY,
        name        VARCHAR(100) NOT NULL,
        email       VARCHAR(200) UNIQUE NOT NULL,
        tier        VARCHAR(20) DEFAULT 'standard',
        created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE orders (
        id          SERIAL PRIMARY KEY,
        customer_id INTEGER REFERENCES customers(id),
        product     VARCHAR(200) NOT NULL,
        amount      DECIMAL(10,2) NOT NULL,
        status      VARCHAR(20) DEFAULT 'pending',
        created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX idx_orders_customer ON orders(customer_id);
    CREATE INDEX idx_orders_status ON orders(status);
""")
conn.commit()
print("✅ 테이블 생성 완료")

# 5. 샘플 데이터 입력
cursor.execute("""
    INSERT INTO customers (name, email, tier) VALUES
    ('김철수', 'cs.kim@example.com', 'premium'),
    ('이영희', 'yh.lee@example.com', 'standard'),
    ('박민수', 'ms.park@example.com', 'premium');
""")

cursor.execute("""
    INSERT INTO orders (customer_id, product, amount, status) VALUES
    (1, '노트북', 1500000, 'completed'),
    (1, '마우스', 35000, 'completed'),
    (2, '키보드', 89000, 'pending'),
    (3, '모니터', 450000, 'shipped');
""")
conn.commit()
print("✅ 샘플 데이터 입력 완료")

# 6. 데이터 확인
cursor.execute("""
    SELECT c.name, o.product, o.amount, o.status
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    ORDER BY o.amount DESC
""")
for row in cursor.fetchall():
    print(f"  {row[0]}: {row[1]} - ₩{row[2]:,.0f} ({row[3]})")

# 출력 예시:
# 김철수: 노트북 - ₩1,500,000 (completed)
# 박민수: 모니터 - ₩450,000 (shipped)
# 이영희: 키보드 - ₩89,000 (pending)
# 김철수: 마우스 - ₩35,000 (completed)

conn.close()
print("✅ 설정 완료!")
```

---

## 정리

| 설정 단계 | 핵심 사항 |
|-----------|-----------|
| ** 인스턴스 생성** | UI, SQL, Python SDK 모두 지원됩니다. 1~3분 내 생성됩니다 |
| ** 연결 설정** | 표준 PostgreSQL 프로토콜, SSL 필수입니다 |
| ** 사이징** | 오토스케일링이 자동으로 처리하므로 기본 설정으로 시작합니다 |
| ** 사용자 관리** | Unity Catalog 통합 권한 관리를 사용합니다 |
| ** 인증** | 프로덕션은 OAuth/서비스 프린시펄을 사용합니다 |
| ** 백업** | 자동 백업 (35일), Point-in-Time Recovery를 지원합니다 |
| ** 모니터링** | pg_stat 뷰와 Databricks 메트릭으로 상태를 확인합니다 |
| ** 네트워크** | 프로덕션은 Private Link를 권장합니다 |

---

## 참고 링크

- [Databricks: Create Lakebase Database](https://docs.databricks.com/aws/en/lakebase/)
- [Azure Databricks: Lakebase](https://learn.microsoft.com/en-us/azure/databricks/lakebase/)
- [Databricks: Unity Catalog Privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/)
- [Databricks: Private Link](https://docs.databricks.com/aws/en/security/network/classic/privatelink.html)
- [PostgreSQL: psycopg2 Documentation](https://www.psycopg.org/docs/)
