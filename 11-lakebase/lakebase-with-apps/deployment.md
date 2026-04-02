# 배포 및 운영

## 1. 왜 Lakebase + Apps 배포가 중요한가

기존 OLTP 웹 애플리케이션은 RDS, Cloud SQL 등 외부 데이터베이스를 사용하고, 분석이 필요할 때마다 별도의 ETL 파이프라인으로 데이터를 Lakehouse로 이동시켜야 했습니다. 이 방식은 다음과 같은 문제를 야기합니다.

- **데이터 사일로 (Data Silo)**: 운영 DB와 분석 플랫폼이 분리되어 실시간 분석이 어렵습니다.
- **ETL 복잡도**: 데이터 이동 파이프라인 유지·관리 비용이 발생합니다.
- **보안 경계 분산**: 별도 DB의 IAM/시크릿 관리가 추가됩니다.
- **거버넌스 공백**: Unity Catalog 혜택(데이터 계보, 접근 제어)을 운영 DB 데이터에 적용하기 어렵습니다.

**Databricks Apps + Lakebase** 조합은 이 문제를 근본적으로 해결합니다.

| 이점 | 설명 |
|------|------|
| **단일 플랫폼** | 앱 서빙, OLTP 스토리지, 분석이 하나의 Databricks 워크스페이스에서 처리됩니다 |
| **자동 동기화 (Data Sync)** | Lakebase 트랜잭션 데이터가 Delta Lake로 자동 반영됩니다 |
| **통합 거버넌스** | Unity Catalog로 운영 데이터에도 행/열 수준 접근 제어가 적용됩니다 |
| **OAuth 통합 인증** | 별도 인증 서버 없이 Databricks OAuth 2.0으로 앱과 DB를 함께 보호합니다 |
| **비용 단순화** | 외부 DB 인스턴스 비용 없이 Databricks DBU 기반으로 과금이 통합됩니다 |

---

## 2. 배포 아키텍처

### 전체 구성도

```
┌─────────────────────────────────────────────────────┐
│              Databricks Workspace                    │
│                                                     │
│  ┌──────────────┐    PostgreSQL    ┌─────────────┐  │
│  │ Databricks   │◄───────────────►│  Lakebase   │  │
│  │ Apps         │  psycopg2/       │  (OLTP DB)  │  │
│  │ (Streamlit / │  SQLAlchemy      └──────┬──────┘  │
│  │  FastAPI)    │                         │ Data    │
│  └──────┬───────┘                         │ Sync    │
│         │ OAuth 2.0                       ▼         │
│         │                       ┌─────────────────┐ │
│  ┌──────▼───────┐               │  Delta Lake /   │ │
│  │  Databricks  │               │  Unity Catalog  │ │
│  │  Identity    │               │  (분석 레이어)   │ │
│  │  (SSO)       │               └─────────────────┘ │
│  └──────────────┘                                   │
└─────────────────────────────────────────────────────┘
           ▲
           │ HTTPS (Databricks 관리형 도메인)
           │
     End User / Browser
```

### OAuth 인증 흐름 (Authentication Flow)

1. 사용자가 앱 URL(`https://<workspace>.azuredatabricks.net/apps/<app-name>`)에 접근합니다.
2. Databricks Apps 런타임이 Databricks OAuth 2.0으로 사용자를 인증합니다.
3. 앱 컨테이너 내부에서 `DATABRICKS_TOKEN` 환경 변수가 자동 주입됩니다.
4. 앱 코드가 해당 토큰을 사용해 Lakebase에 연결합니다 (`generate_lakebase_credential()` API 활용).
5. 모든 DB 접근이 사용자 신원과 연결된 단기 자격증명 (Short-lived Credential)으로 이루어집니다.

> **핵심**: 앱 코드에 장기 비밀번호(Long-lived Password)를 하드코딩할 필요가 없습니다. Databricks가 자격증명 생명주기를 전적으로 관리합니다.

---

## 3. app.yaml 설정

`app.yaml`은 Databricks Apps의 배포 명세 파일입니다. 앱 실행 명령, 환경 변수, 필요 리소스를 선언합니다.

### 기본 구조 예시 (Streamlit + Lakebase)

```yaml
# app.yaml — Databricks Apps 배포 설정
command:
  - "streamlit"
  - "run"
  - "app.py"
  - "--server.port=8501"
  - "--server.address=0.0.0.0"

# 리소스 선언 — Databricks가 자동으로 자격증명을 주입합니다
resources:
  - name: "lakebase-instance"
    description: "운영 Lakebase PostgreSQL 인스턴스"
    databricks_resource_type: "databricks-lakebase-database"
    # Lakebase 인스턴스 이름 (Databricks Console에서 확인)
    instance_id: "${var.lakebase_instance_id}"

  - name: "sql-warehouse"
    description: "분석용 SQL Warehouse"
    databricks_resource_type: "databricks-sql-warehouse"
    warehouse_id: "${var.warehouse_id}"

# 환경 변수 설정
env:
  - name: LAKEBASE_HOST
    valueFrom:
      resourceRef:
        resourceName: "lakebase-instance"
        fieldPath: "host"
  - name: LAKEBASE_PORT
    value: "5432"
  - name: LAKEBASE_DBNAME
    value: "shop_db"
  - name: APP_ENV
    value: "production"
  - name: LOG_LEVEL
    value: "INFO"
  # 시크릿은 Databricks Secrets에서 참조합니다
  - name: APP_SECRET_KEY
    valueFrom:
      secretRef:
        secretName: "app-secrets"
        key: "secret_key"
```

### FastAPI를 사용하는 경우

```yaml
# app.yaml — FastAPI 배포
command:
  - "uvicorn"
  - "main:app"
  - "--host=0.0.0.0"
  - "--port=8000"
  - "--workers=2"

resources:
  - name: "lakebase-instance"
    databricks_resource_type: "databricks-lakebase-database"
    instance_id: "${var.lakebase_instance_id}"

env:
  - name: DATABASE_URL
    value: "postgresql://${LAKEBASE_USER}:${LAKEBASE_PASSWORD}@${LAKEBASE_HOST}:5432/app_db"
  - name: POOL_SIZE
    value: "10"
  - name: MAX_OVERFLOW
    value: "20"
```

---

## 4. 배포 방법

### 4-1. Databricks CLI를 사용한 직접 배포

```bash
# Databricks CLI 설치
pip install databricks-cli

# 워크스페이스 인증 설정
databricks configure --token

# 앱 생성 (최초 1회)
databricks apps create my-lakebase-app \
  --description "Lakebase 기반 쇼핑몰 앱"

# 앱 배포 (소스 디렉터리 기준)
databricks apps deploy my-lakebase-app \
  --source-code-path ./my-app

# 배포 상태 확인
databricks apps get my-lakebase-app

# 앱 로그 실시간 확인
databricks apps logs my-lakebase-app --follow
```

### 4-2. Databricks Asset Bundles (DAB) 를 사용한 배포

Asset Bundles는 앱, Job, Pipeline 등 Databricks 리소스를 코드로 관리(IaC)하는 방식입니다.

```yaml
# databricks.yml — Asset Bundle 설정
bundle:
  name: lakebase-app-bundle

variables:
  lakebase_instance_id:
    description: "Lakebase 인스턴스 ID"
  warehouse_id:
    description: "SQL Warehouse ID"

targets:
  dev:
    mode: development
    workspace:
      host: https://adb-xxxx.azuredatabricks.net
    variables:
      lakebase_instance_id: "dev-instance-id"
      warehouse_id: "dev-warehouse-id"

  prod:
    mode: production
    workspace:
      host: https://adb-yyyy.azuredatabricks.net
    variables:
      lakebase_instance_id: "prod-instance-id"
      warehouse_id: "prod-warehouse-id"

resources:
  apps:
    lakebase_app:
      name: "my-lakebase-app"
      description: "Lakebase 기반 운영 앱"
      source_code_path: ./app
```

```bash
# 개발 환경 배포
databricks bundle deploy --target dev

# 프로덕션 환경 배포
databricks bundle deploy --target prod
```

### 4-3. GitHub Actions CI/CD 연동

```yaml
# .github/workflows/deploy.yml
name: Deploy Databricks App

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Databricks CLI
        run: pip install databricks-cli

      - name: Deploy to Production
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: |
          databricks bundle deploy --target prod
```

---

## 5. 환경 분리 (Environment Isolation)

프로덕션 데이터 오염을 방지하기 위해 환경별로 독립된 Lakebase 인스턴스를 사용하는 것이 원칙입니다.

| 환경 | Lakebase 인스턴스 | 용도 |
|------|-------------------|------|
| **dev** | `lakebase-dev` | 개발자 로컬 테스트, 스키마 변경 실험 |
| **staging** | `lakebase-staging` | QA 테스트, 성능 검증, 마이그레이션 리허설 |
| **prod** | `lakebase-prod` | 실제 운영, 고가용성, 백업 필수 |

### 환경별 연결 설정 예시 (Python)

```python
# config.py — 환경별 DB 연결 설정
import os

APP_ENV = os.getenv("APP_ENV", "dev")

DB_CONFIG = {
    "dev": {
        "host": os.getenv("LAKEBASE_DEV_HOST"),
        "dbname": "app_dev",
        "pool_size": 3,
        "max_overflow": 5,
    },
    "staging": {
        "host": os.getenv("LAKEBASE_STAGING_HOST"),
        "dbname": "app_staging",
        "pool_size": 5,
        "max_overflow": 10,
    },
    "production": {
        "host": os.getenv("LAKEBASE_PROD_HOST"),
        "dbname": "app_prod",
        "pool_size": 20,
        "max_overflow": 40,
    },
}

current_config = DB_CONFIG[APP_ENV]
```

> **주의**: 스테이징 환경에 프로덕션 데이터를 복사할 때는 반드시 개인정보(PII)를 마스킹(Masking)하세요.

---

## 6. 스케일링과 성능 (Scaling & Performance)

### Lakebase 인스턴스 사이징 가이드

| 규모 | vCPU | RAM | 적합한 동시 접속 수 |
|------|------|-----|---------------------|
| 소규모 (Small) | 2 | 8 GB | ~50 |
| 중간 (Medium) | 4 | 16 GB | ~200 |
| 대규모 (Large) | 8 | 32 GB | ~500 |
| 초대규모 (XLarge) | 16 | 64 GB | ~1,000+ |

### 연결 풀링 (Connection Pooling) 설정

```python
# database.py — SQLAlchemy 연결 풀 설정
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool
import os

def get_engine():
    db_url = (
        f"postgresql+psycopg2://{os.getenv('LAKEBASE_USER')}:"
        f"{os.getenv('LAKEBASE_PASSWORD')}@"
        f"{os.getenv('LAKEBASE_HOST')}:5432/"
        f"{os.getenv('LAKEBASE_DBNAME')}"
    )
    return create_engine(
        db_url,
        poolclass=QueuePool,
        pool_size=int(os.getenv("POOL_SIZE", 10)),      # 기본 유지 연결 수
        max_overflow=int(os.getenv("MAX_OVERFLOW", 20)), # 최대 초과 허용 연결 수
        pool_timeout=30,       # 풀에서 연결을 기다리는 최대 시간 (초)
        pool_recycle=1800,     # 연결 재활용 주기 (초) — 장시간 유휴 연결 방지
        pool_pre_ping=True,    # 연결 유효성 사전 확인 (Dead Connection 방지)
        connect_args={
            "connect_timeout": 10,
            "application_name": "databricks-app",
            "sslmode": "require",  # SSL 필수 (보안)
        },
    )

# 앱 시작 시 엔진 1회 초기화
engine = get_engine()
```

### Auto-scaling 고려 사항

- Databricks Apps는 자체적으로 **요청 기반 스케일링** 을 지원합니다.
- 앱 인스턴스가 늘어날수록 DB 연결 수도 증가합니다. `pool_size × 앱 인스턴스 수`가 Lakebase 최대 연결 수를 초과하지 않도록 설계하세요.
- 고트래픽 환경에서는 **PgBouncer** 또는 Lakebase 내장 연결 관리자를 활용하는 것을 권장합니다.

---

## 7. 모니터링과 로깅 (Monitoring & Logging)

### 앱 로그 확인

```bash
# CLI에서 실시간 로그 스트리밍
databricks apps logs my-lakebase-app --follow

# 특정 기간 로그 조회
databricks apps logs my-lakebase-app \
  --start-time "2026-01-01T00:00:00Z" \
  --end-time "2026-01-02T00:00:00Z"
```

### Python 로깅 설정 예시

```python
# logging_config.py
import logging
import time
from functools import wraps

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger(__name__)

def log_slow_query(threshold_ms: int = 500):
    """느린 쿼리를 자동으로 경고 로그로 기록하는 데코레이터"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = time.time()
            result = func(*args, **kwargs)
            elapsed_ms = (time.time() - start) * 1000
            if elapsed_ms > threshold_ms:
                logger.warning(
                    f"SLOW QUERY detected: {func.__name__} took {elapsed_ms:.1f}ms"
                )
            return result
        return wrapper
    return decorator
```

### Lakebase 성능 메트릭 쿼리

```sql
-- 현재 활성 연결 수 확인
SELECT count(*) AS active_connections
FROM pg_stat_activity
WHERE state = 'active';

-- 느린 쿼리 Top 10 조회
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 테이블별 블로트(Bloat) 확인
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup, 0) * 100, 2) AS dead_ratio
FROM pg_stat_user_tables
ORDER BY dead_ratio DESC NULLS LAST;
```

### 헬스체크 엔드포인트 구현

```python
# health.py — FastAPI 헬스체크
from fastapi import APIRouter
from sqlalchemy import text
from database import engine

router = APIRouter()

@router.get("/health")
def health_check():
    try:
        with engine.connect() as conn:
            conn.execute(text("SELECT 1"))
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return {"status": "unhealthy", "database": str(e)}, 503
```

---

## 8. 장단점과 트레이드오프 (Trade-offs)

### Databricks Apps vs 외부 호스팅 비교

| 항목 | Databricks Apps + Lakebase | 외부 호스팅 (Vercel + RDS / AWS ECS) |
|------|---------------------------|--------------------------------------|
| **초기 설정** | 워크스페이스 내 몇 번의 클릭 | VPC, IAM, 보안 그룹 별도 구성 필요 |
| **인증 통합** | Databricks OAuth 자동 처리 | 별도 인증 서버(Auth0, Cognito) 구성 필요 |
| **데이터 거버넌스** | Unity Catalog 즉시 적용 | 별도 거버넌스 레이어 구축 필요 |
| **분석 연계** | Data Sync로 자동 반영 | ETL 파이프라인 별도 구축 필요 |
| **커스터마이징** | Databricks 지원 프레임워크로 제한 | 자유로운 언어/런타임 선택 가능 |
| **Cold Start** | 첫 요청 시 약간의 지연 발생 가능 | 상시 실행 인스턴스는 지연 없음 |
| **비용** | DBU 기반 통합 과금 | 서비스별 분산 과금 (예측 어려울 수 있음) |
| **글로벌 CDN** | 현재 미지원 | Vercel/CloudFront CDN 활용 가능 |

### 언제 Databricks Apps + Lakebase를 선택해야 하는가

- 데이터 팀이 이미 Databricks를 사용하고 있는 경우
- 운영 데이터를 실시간으로 분석에 활용해야 하는 경우
- 내부 도구(Internal Tool) 또는 B2B SaaS 형태의 앱인 경우
- 단일 거버넌스 정책으로 모든 데이터를 통제해야 하는 경우

### 외부 호스팅이 더 적합한 경우

- 글로벌 CDN이 반드시 필요한 퍼블릭 웹사이트
- 매우 낮은 Cold Start 레이턴시가 요구되는 고빈도 서비스
- Databricks가 지원하지 않는 특정 런타임(Go, Rust, Node.js 등)이 필요한 경우

---

## 9. 베스트 프랙티스와 흔한 실수

### 베스트 프랙티스 (Best Practices)

| 항목 | 권장 방법 |
|------|-----------|
| **자격증명 관리** | 비밀번호 하드코딩 금지 — Databricks Secrets 또는 자동 주입 토큰 사용 |
| **연결 풀링** | 프로덕션에서는 반드시 `pool_size`, `max_overflow`, `pool_recycle` 설정 |
| **SQL 인젝션 방지** | 파라미터 바인딩(`%s`, `?`) 사용 — f-string SQL은 절대 금지 |
| **트랜잭션 관리** | 커밋/롤백을 명시적으로 처리하고 커넥션을 즉시 반환 |
| **마이그레이션 관리** | Alembic 등 마이그레이션 도구로 스키마 변경을 버전 관리 |
| **헬스체크** | `/health` 엔드포인트에서 DB 연결 상태를 항상 노출 |
| **인덱스 설계** | 자주 조회되는 컬럼에 인덱스 추가, EXPLAIN ANALYZE로 쿼리 계획 확인 |

### 흔한 실수 (Common Mistakes)

```python
# ❌ 잘못된 예시 1: f-string으로 SQL 작성 (SQL 인젝션 취약점)
query = f"SELECT * FROM orders WHERE user_id = {user_id}"

# ✅ 올바른 예시: 파라미터 바인딩 사용
query = "SELECT * FROM orders WHERE user_id = %s"
cursor.execute(query, (user_id,))

# ❌ 잘못된 예시 2: 앱 시작마다 새 연결 생성 (연결 누수)
def get_data():
    conn = psycopg2.connect(...)  # 매 호출마다 새 연결 — 메모리 누수
    ...

# ✅ 올바른 예시: 연결 풀에서 연결을 빌려 사용 후 반환
def get_data():
    with engine.connect() as conn:  # 풀에서 빌려서 자동 반환
        result = conn.execute(text("SELECT ..."))
        return result.fetchall()

# ❌ 잘못된 예시 3: 환경 변수 없이 하드코딩
LAKEBASE_HOST = "my-db.lakebase.databricks.com"  # 코드에 호스트 하드코딩

# ✅ 올바른 예시: 환경 변수에서 읽기
LAKEBASE_HOST = os.getenv("LAKEBASE_HOST")
if not LAKEBASE_HOST:
    raise ValueError("LAKEBASE_HOST 환경 변수가 설정되지 않았습니다.")
```

### 운영 체크리스트

| 항목 | 확인 사항 |
|------|-----------|
| **커넥션 풀** | `pool_size`, `max_overflow`, `pool_recycle` 이 설정되어 있습니다 |
| **에러 처리** | 모든 DB 작업에 `try/except` 및 롤백 처리가 있습니다 |
| **SQL 인젝션 방지** | 파라미터 바인딩(`%s`)을 사용하고 f-string SQL이 없습니다 |
| **타임아웃 설정** | 쿼리 타임아웃과 연결 타임아웃이 명시되어 있습니다 |
| **로깅** | 에러와 500ms 이상 느린 쿼리가 로깅됩니다 |
| **헬스체크** | `/health` 엔드포인트에서 DB 연결 상태를 확인합니다 |
| **SSL 강제** | `sslmode=require`로 암호화 통신이 적용되어 있습니다 |
| **환경 분리** | dev/staging/prod 각각 독립된 Lakebase 인스턴스를 사용합니다 |

---

## 정리

| 핵심 포인트 | 설명 |
|-------------|------|
| **통합 플랫폼** | Databricks Apps + Lakebase로 앱 개발~분석까지 하나의 플랫폼에서 처리합니다 |
| **다양한 프레임워크** | Streamlit, FastAPI, Dash, Flask 등 Python 프레임워크를 자유롭게 선택합니다 |
| **OAuth 통합 인증** | 별도의 인증 시스템 없이 Databricks OAuth를 활용합니다 |
| **커넥션 풀링** | 프로덕션에서는 반드시 커넥션 풀을 설정합니다 |
| **자동 동기화** | 앱 데이터가 Data Sync로 자동으로 분석 환경에 반영됩니다 |
| **환경 분리** | dev/staging/prod 별로 독립된 Lakebase 인스턴스를 운영합니다 |
| **보안** | SQL 인젝션 방지, SSL 필수, 파라미터 바인딩을 준수합니다 |
| **IaC 배포** | Asset Bundles와 CI/CD로 배포를 코드로 관리합니다 |

---

## 참고 링크

- [Databricks: Databricks Apps 개요](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/)
- [Databricks: Databricks Apps — app.yaml 설정 레퍼런스](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/app-yaml.html)
- [Databricks: Lakebase 개요](https://docs.databricks.com/aws/en/lakebase/)
- [Databricks: Asset Bundles (DAB)](https://docs.databricks.com/aws/en/dev-tools/bundles/)
- [Databricks: Databricks CLI — apps 명령어](https://docs.databricks.com/aws/en/dev-tools/cli/apps-commands.html)
- [SQLAlchemy: Connection Pooling](https://docs.sqlalchemy.org/en/20/core/pooling.html)
- [Alembic: Database Migration](https://alembic.sqlalchemy.org/en/latest/)
- [Azure Databricks: Databricks Apps](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/databricks-apps/)
- [Streamlit Documentation](https://docs.streamlit.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [psycopg2 Documentation](https://www.psycopg.org/docs/)
