# Databricks Python SDK

## Databricks SDK란?

Databricks Python SDK는 Databricks 플랫폼의 모든 기능을 **프로그래밍 방식으로 제어** 할 수 있게 해주는 공식 Python 라이브러리입니다. 클러스터 생성, Job 실행, SQL 쿼리, Unity Catalog 관리 등 UI에서 할 수 있는 거의 모든 작업을 Python 코드로 자동화할 수 있습니다.

> 💡 **왜 SDK가 필요한가?**
> UI에서 수동으로 작업하는 것은 소규모 환경에서는 문제가 없지만, 수십 개의 클러스터를 관리하거나 수백 개의 Job을 배포할 때는 자동화가 필수적입니다. SDK를 사용하면 Infrastructure as Code(IaC) 방식으로 Databricks 리소스를 관리할 수 있습니다.

---

## 설치 및 인증

### 설치

```bash
pip install databricks-sdk
```

### 인증 방법

Databricks SDK는 **통합 인증(Unified Auth)** 을 지원합니다. 여러 인증 방법 중 환경에 맞는 것을 선택할 수 있습니다.

| 인증 방법 | 적합한 환경 | 보안 수준 |
|---|---|---|
| **OAuth (User-to-Machine)** | 개인 개발 환경 | 높음 (권장) |
| **PAT (개인 접근 토큰)** | 간단한 스크립트, 테스트 | 보통 |
| **OAuth (Machine-to-Machine)** | CI/CD, 자동화 파이프라인 | 높음 |
| **Service Principal** | 프로덕션 서비스 | 높음 (권장) |
| **Azure Managed Identity** | Azure 환경 | 높음 |

#### OAuth 인증 (권장)

```python
from databricks.sdk import WorkspaceClient

# Databricks CLI 로그인이 되어 있으면 자동으로 인증
w = WorkspaceClient()

# 현재 사용자 확인
me = w.current_user.me()
print(f"인증된 사용자: {me.user_name}")
```

#### PAT 인증

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient(
    host="https://<workspace-url>",
    token="dapi..."
)
```

#### Service Principal (M2M OAuth)

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient(
    host="https://<workspace-url>",
    client_id="<client-id>",
    client_secret="<client-secret>"
)
```

#### 환경 변수 사용

코드에 인증 정보를 직접 넣지 않고 환경 변수를 활용하는 것을 권장합니다.

```bash
export DATABRICKS_HOST="https://<workspace-url>"
export DATABRICKS_TOKEN="dapi..."
```

```python
# 환경 변수가 설정되어 있으면 자동으로 인식
w = WorkspaceClient()
```

---

## 주요 클라이언트

SDK는 두 가지 주요 클라이언트를 제공합니다.

| 클라이언트 | 용도 | 주요 API |
|---|---|---|
| **WorkspaceClient** | 워크스페이스 수준 작업 | 클러스터, Job, SQL, 노트북 등 |
| **AccountClient** | 계정 수준 작업 | 워크스페이스 관리, 사용자/그룹, 메타스토어 |

```python
from databricks.sdk import WorkspaceClient, AccountClient

# 워크스페이스 클라이언트 (가장 많이 사용)
w = WorkspaceClient()

# 계정 클라이언트 (관리자용)
a = AccountClient(
    host="https://accounts.cloud.databricks.com",
    account_id="<account-id>"
)
```

---

## 핵심 API 영역

### 클러스터 관리 (w.clusters)

```python
# 클러스터 목록 조회
for cluster in w.clusters.list():
    print(f"{cluster.cluster_name}: {cluster.state}")

# 클러스터 시작 (완료까지 대기)
w.clusters.start(cluster_id="0123-456789-abcdef").result()
print("클러스터가 시작되었습니다.")
```

### Job 관리 (w.jobs)

```python
# Job 목록 조회
for job in w.jobs.list():
    print(f"[{job.job_id}] {job.settings.name}")

# Job 즉시 실행 및 완료 대기
run = w.jobs.run_now(job_id=12345)
result = run.result()
print(f"실행 완료: {result.state.result_state}")
```

### SQL 실행 (w.statement_execution)

```python
# SQL 쿼리 실행
from databricks.sdk.service.sql import StatementState

response = w.statement_execution.execute_statement(
    warehouse_id="abc123def456",
    statement="SELECT region, SUM(revenue) AS total FROM sales GROUP BY region ORDER BY total DESC LIMIT 10",
    wait_timeout="30s"
)

if response.status.state == StatementState.SUCCEEDED:
    # 컬럼명 출력
    columns = [col.name for col in response.manifest.schema.columns]
    print("\t".join(columns))

    # 결과 행 출력
    for row in response.result.data_array:
        print("\t".join(str(v) for v in row))
```

### Unity Catalog 관리

```python
# 3-level 네임스페이스 탐색: 카탈로그 → 스키마 → 테이블
for catalog in w.catalogs.list():
    print(f"카탈로그: {catalog.name}")

for table in w.tables.list(catalog_name="my_catalog", schema_name="my_schema"):
    print(f"  테이블: {table.name} ({table.table_type})")

# 권한 부여
from databricks.sdk.service.catalog import SecurableType, PermissionsChange, Privilege

w.grants.update(
    securable_type=SecurableType.TABLE,
    full_name="my_catalog.my_schema.customers",
    changes=[PermissionsChange(add=[Privilege.SELECT], principal="data-analysts")]
)
```

### Model Serving (w.serving_endpoints)

```python
# 엔드포인트 호출
response = w.serving_endpoints.query(
    name="my-model-endpoint",
    dataframe_records=[{"feature1": 1.0, "feature2": "A", "feature3": 100}]
)
print(f"예측 결과: {response.predictions}")
```

### Secret 관리 (w.secrets)

```python
# Secret Scope 생성 및 Secret 저장
w.secrets.create_scope(scope="my-app-secrets")
w.secrets.put_secret(scope="my-app-secrets", key="api-key", string_value="sk-...")
```

---

## Databricks Connect

Databricks Connect는 로컬 IDE(VS Code, PyCharm 등)에서 작성한 PySpark 코드를 **원격 Databricks 클러스터에서 실행** 할 수 있게 해주는 기능입니다. SDK의 일부로 제공됩니다.

### 설치

```bash
pip install databricks-connect==15.4.*
```

> **중요**: `databricks-connect` 버전은 대상 클러스터의 DBR(Databricks Runtime) 버전과 일치해야 합니다.

### 사용 예시

```python
from databricks.connect import DatabricksSession

# 원격 Spark 세션 생성
spark = DatabricksSession.builder.remote(
    host="https://<workspace-url>",
    cluster_id="0123-456789-abcdef"
).getOrCreate()

# 로컬 코드가 원격 클러스터에서 실행됨
df = spark.table("my_catalog.my_schema.orders")
result = df.groupBy("region").sum("revenue").collect()

for row in result:
    print(f"{row['region']}: {row['sum(revenue)']}")
```

### Databricks Connect vs 일반 PySpark 비교

| 항목 | Databricks Connect | 로컬 PySpark |
|---|---|---|
| **실행 위치** | 원격 Databricks 클러스터 | 로컬 머신 |
| **데이터 접근** | Unity Catalog 테이블 직접 접근 | 로컬 파일만 |
| **스케일** | 클러스터 자원 활용 | 로컬 자원 제한 |
| **디버깅** | 로컬 IDE에서 브레이크포인트 지원 | 로컬 IDE에서 디버깅 |
| **적합 용도** | 개발·테스트 | 단위 테스트, 학습 |

---

## REST API vs SDK 비교

Databricks REST API를 직접 호출하는 것 대비 SDK를 사용하면 여러 장점이 있습니다.

| 항목 | REST API | Python SDK |
|---|---|---|
| **타입 안전성** | 없음 (딕셔너리 기반) | 있음 (데이터 클래스) |
| **자동 완성** | 불가 | IDE 자동 완성 지원 |
| **인증 관리** | 수동 헤더 설정 | 통합 인증 자동 처리 |
| **페이지네이션** | 수동 처리 | 자동 이터레이션 |
| **장시간 작업** | 폴링 로직 직접 구현 | `.result()` 메서드로 자동 대기 |
| **에러 처리** | HTTP 상태 코드 파싱 | 구조화된 예외 클래스 |
| **API 버전 관리** | URL 경로에 직접 명시 | SDK 업데이트로 자동 반영 |

REST API를 직접 호출하면 `requests` 라이브러리로 헤더·URL·페이지네이션을 모두 수동 관리해야 합니다. SDK는 이 모든 것을 추상화하므로 코드가 훨씬 간결해집니다.

---

## 실습: 워크스페이스 리소스 인벤토리

아래 스크립트는 SDK를 사용하여 워크스페이스의 주요 리소스 현황을 한눈에 파악하는 예시입니다.

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 현재 사용자 확인
me = w.current_user.me()
print(f"사용자: {me.user_name}")

# 클러스터 현황
clusters = list(w.clusters.list())
running = [c for c in clusters if str(c.state) == "RUNNING"]
print(f"클러스터: 총 {len(clusters)}개, 실행 중 {len(running)}개")

# Job / Warehouse / 서빙 엔드포인트 현황
print(f"Job: 총 {len(list(w.jobs.list()))}개")
print(f"SQL Warehouse: 총 {len(list(w.warehouses.list()))}개")
print(f"서빙 엔드포인트: 총 {len(list(w.serving_endpoints.list()))}개")
```

---

## 정리

| 항목 | 내용 |
|---|---|
| **핵심 가치** | Python으로 Databricks 플랫폼 전체를 프로그래밍 방식으로 제어 |
| **주요 클라이언트** | WorkspaceClient (워크스페이스), AccountClient (계정) |
| **인증** | OAuth, PAT, Service Principal 등 통합 인증 |
| **장점** | 타입 안전성, IDE 자동 완성, 자동 페이지네이션, 장시간 작업 대기 |
| **Databricks Connect** | 로컬 IDE에서 원격 Spark 실행 |

---

## 참고 링크

- [Databricks Python SDK 공식 문서](https://docs.databricks.com/aws/en/dev-tools/sdk-python.html)
- [Databricks SDK API 레퍼런스](https://databricks-sdk-py.readthedocs.io/)
- [Databricks SDK GitHub](https://github.com/databricks/databricks-sdk-py)
- [Databricks Connect 공식 문서](https://docs.databricks.com/aws/en/dev-tools/databricks-connect/)
- [Databricks 통합 인증 가이드](https://docs.databricks.com/aws/en/dev-tools/auth/unified-auth-client.html)
- [REST API 레퍼런스](https://docs.databricks.com/api/)
