# Databricks REST API 활용

## 개념

> 💡 **Databricks REST API**는 Databricks 플랫폼의 거의 모든 기능을 **HTTP 요청으로 제어**할 수 있는 프로그래밍 인터페이스입니다. 웹 UI에서 할 수 있는 대부분의 작업을 API로 자동화할 수 있습니다.

### 왜 REST API가 필요한가요?

| 상황 | UI 방식 | REST API 방식 |
|------|---------|--------------|
| **클러스터 100개 설정 변경** | 하나씩 수동으로 변경 | 스크립트로 일괄 변경 |
| **매일 Job 상태 모니터링** | 매번 UI 접속해서 확인 | 자동 알림 시스템 구축 |
| **CI/CD 파이프라인** | 수동 배포 | API로 자동 배포 |
| **외부 시스템 연동** | 불가능 | Slack, JIRA 등과 통합 |
| **Workspace 초기 설정** | 반복 수작업 | IaC(Infrastructure as Code)로 자동화 |

---

## 인증 방법

REST API를 호출하려면 먼저 인증이 필요합니다. 용도에 따라 적절한 인증 방식을 선택합니다.

### Personal Access Token (PAT)

가장 간단한 인증 방식으로, 개발/테스트에 적합합니다.

```bash
# PAT 생성: Workspace UI > User Settings > Developer > Access Tokens

# 환경 변수로 설정 (권장)
export DATABRICKS_HOST="https://dbc-abc123.cloud.databricks.com"
export DATABRICKS_TOKEN="dapi1234567890abcdef"

# curl에서 PAT 사용
curl -s -X GET \
  "${DATABRICKS_HOST}/api/2.0/clusters/list" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json"
```

> ⚠️ **주의**: PAT는 발급한 사용자의 모든 권한을 가지므로, 코드에 직접 하드코딩하지 마세요. 환경 변수나 시크릿 관리 도구를 사용하세요.

### OAuth Machine-to-Machine (M2M)

프로덕션 자동화에 권장되는 방식입니다. Service Principal을 사용하여 사람의 개입 없이 인증합니다.

```bash
# 1단계: Service Principal 생성 (Account Console에서)
# 2단계: Client ID와 Client Secret 발급

# OAuth 토큰 발급
curl -s -X POST \
  "https://dbc-abc123.cloud.databricks.com/oidc/v1/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=<CLIENT_ID>" \
  -d "client_secret=<CLIENT_SECRET>" \
  -d "scope=all-apis"

# 응답에서 access_token 추출 후 사용
# {
#   "access_token": "eyJhbG...",
#   "token_type": "Bearer",
#   "expires_in": 3600
# }
```

### 인증 방식 비교

| 방식 | 보안 수준 | 만료 | 적합한 용도 |
|------|-----------|------|------------|
| **PAT** | 중간 | 수동 설정 (최대 365일) | 개발, 테스트, 개인 스크립트 |
| **OAuth M2M** | 높음 | 자동 (1시간) | CI/CD, 프로덕션 자동화 |
| **OAuth U2M** | 높음 | 자동 | 대화형 애플리케이션 |

---

## 주요 API 영역

### API URL 구조

```
https://<workspace-url>/api/<version>/<service>/<action>
                              │          │         │
                              │          │         └── 세부 작업
                              │          └── 서비스 영역
                              └── API 버전 (2.0 또는 2.1)
```

### 주요 서비스 목록

| API 영역 | 엔드포인트 | 주요 기능 |
|----------|-----------|-----------|
| **Clusters** | `/api/2.0/clusters/` | 클러스터 생성, 시작, 종료, 목록 조회 |
| **Jobs** | `/api/2.1/jobs/` | Job 생성, 실행, 상태 조회, 스케줄 관리 |
| **SQL** | `/api/2.0/sql/` | SQL Warehouse 관리, 쿼리 실행 |
| **Unity Catalog** | `/api/2.1/unity-catalog/` | 카탈로그, 스키마, 테이블, 권한 관리 |
| **Workspace** | `/api/2.0/workspace/` | 노트북, 폴더 관리 |
| **Model Serving** | `/api/2.0/serving-endpoints/` | 모델 엔드포인트 생성, 관리, 호출 |
| **DBFS** | `/api/2.0/dbfs/` | 파일 업로드, 다운로드, 삭제 |
| **Repos** | `/api/2.0/repos/` | Git 저장소 연동 관리 |
| **Secrets** | `/api/2.0/secrets/` | 시크릿 스코프, 키 관리 |

---

## curl 예제로 주요 작업 수행

### 클러스터 관리

```bash
# 클러스터 목록 조회
curl -s -X GET \
  "${DATABRICKS_HOST}/api/2.0/clusters/list" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" | jq '.clusters[] | {id: .cluster_id, name: .cluster_name, state: .state}'

# 클러스터 생성
curl -s -X POST \
  "${DATABRICKS_HOST}/api/2.0/clusters/create" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "cluster_name": "my-analytics-cluster",
    "spark_version": "16.2.x-scala2.12",
    "node_type_id": "i3.xlarge",
    "autoscale": {
      "min_workers": 1,
      "max_workers": 4
    },
    "autotermination_minutes": 30,
    "custom_tags": {
      "team": "analytics",
      "env": "dev"
    }
  }'

# 클러스터 시작
curl -s -X POST \
  "${DATABRICKS_HOST}/api/2.0/clusters/start" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"cluster_id": "0123-456789-abcdefgh"}'

# 클러스터 종료
curl -s -X POST \
  "${DATABRICKS_HOST}/api/2.0/clusters/delete" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"cluster_id": "0123-456789-abcdefgh"}'
```

### Job 관리

```bash
# Job 목록 조회
curl -s -X GET \
  "${DATABRICKS_HOST}/api/2.1/jobs/list?limit=10" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" | jq '.jobs[] | {id: .job_id, name: .settings.name}'

# Job 생성
curl -s -X POST \
  "${DATABRICKS_HOST}/api/2.1/jobs/create" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "daily-etl-pipeline",
    "tasks": [
      {
        "task_key": "extract",
        "notebook_task": {
          "notebook_path": "/Repos/team/project/notebooks/extract"
        },
        "new_cluster": {
          "spark_version": "16.2.x-scala2.12",
          "node_type_id": "i3.xlarge",
          "num_workers": 2
        }
      }
    ],
    "schedule": {
      "quartz_cron_expression": "0 0 6 * * ?",
      "timezone_id": "Asia/Seoul"
    }
  }'

# Job 즉시 실행 (Run Now)
curl -s -X POST \
  "${DATABRICKS_HOST}/api/2.1/jobs/run-now" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"job_id": 12345}'

# Job 실행 상태 조회
curl -s -X GET \
  "${DATABRICKS_HOST}/api/2.1/jobs/runs/get?run_id=67890" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" | jq '{state: .state, start_time: .start_time}'
```

### SQL Warehouse 관리

```bash
# SQL Warehouse 목록 조회
curl -s -X GET \
  "${DATABRICKS_HOST}/api/2.0/sql/warehouses" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" | jq '.warehouses[] | {id: .id, name: .name, state: .state}'

# SQL 문 실행 (Statement Execution API)
curl -s -X POST \
  "${DATABRICKS_HOST}/api/2.0/sql/statements" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "warehouse_id": "abcdef1234567890",
    "statement": "SELECT region, COUNT(*) AS cnt FROM catalog.gold.orders GROUP BY region",
    "wait_timeout": "30s"
  }'
```

### Unity Catalog 관리

```bash
# 카탈로그 목록 조회
curl -s -X GET \
  "${DATABRICKS_HOST}/api/2.1/unity-catalog/catalogs" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" | jq '.catalogs[] | {name: .name, owner: .owner}'

# 테이블 정보 조회
curl -s -X GET \
  "${DATABRICKS_HOST}/api/2.1/unity-catalog/tables/catalog.schema.my_table" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" | jq '{name: .name, columns: [.columns[] | {name: .name, type: .type_name}]}'

# 권한 부여
curl -s -X PATCH \
  "${DATABRICKS_HOST}/api/2.1/unity-catalog/permissions/table/catalog.schema.my_table" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "changes": [
      {
        "principal": "data_analysts",
        "add": ["SELECT"]
      }
    ]
  }'
```

### Model Serving 엔드포인트 호출

```bash
# 모델 엔드포인트에 추론 요청
curl -s -X POST \
  "${DATABRICKS_HOST}/serving-endpoints/my-model/invocations" \
  -H "Authorization: Bearer ${DATABRICKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "dataframe_records": [
      {"feature1": 1.0, "feature2": "A", "feature3": 100},
      {"feature1": 2.5, "feature2": "B", "feature3": 200}
    ]
  }'
```

---

## Python requests 라이브러리 활용

실무에서는 curl보다 Python 스크립트로 API를 호출하는 경우가 더 많습니다.

### 기본 설정

```python
import os
import requests

# 환경 변수에서 인증 정보 읽기
HOST = os.environ["DATABRICKS_HOST"]
TOKEN = os.environ["DATABRICKS_TOKEN"]

HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}

def api_get(endpoint, params=None):
    """GET 요청 헬퍼"""
    resp = requests.get(f"{HOST}{endpoint}", headers=HEADERS, params=params)
    resp.raise_for_status()
    return resp.json()

def api_post(endpoint, payload):
    """POST 요청 헬퍼"""
    resp = requests.post(f"{HOST}{endpoint}", headers=HEADERS, json=payload)
    resp.raise_for_status()
    return resp.json()
```

### 실전 활용 예시: 유휴 클러스터 자동 종료

```python
from datetime import datetime, timedelta

def terminate_idle_clusters(max_idle_hours=2):
    """지정 시간 이상 유휴 상태인 클러스터를 종료합니다."""
    clusters = api_get("/api/2.0/clusters/list").get("clusters", [])

    now = datetime.now()
    terminated = []

    for cluster in clusters:
        if cluster["state"] != "RUNNING":
            continue

        # 마지막 활동 시간 확인
        last_activity = cluster.get("last_activity_time", 0)
        if last_activity == 0:
            continue

        last_active = datetime.fromtimestamp(last_activity / 1000)
        idle_duration = now - last_active

        if idle_duration > timedelta(hours=max_idle_hours):
            cluster_id = cluster["cluster_id"]
            cluster_name = cluster["cluster_name"]

            api_post("/api/2.0/clusters/delete", {"cluster_id": cluster_id})
            terminated.append(f"{cluster_name} ({cluster_id})")
            print(f"종료: {cluster_name} (유휴 {idle_duration})")

    return terminated

# 실행
terminated = terminate_idle_clusters(max_idle_hours=2)
print(f"\n총 {len(terminated)}개 클러스터 종료")
```

### 실전 활용 예시: Job 실행 결과 모니터링

```python
import time

def run_and_wait(job_id, timeout_minutes=60):
    """Job을 실행하고 완료될 때까지 대기합니다."""
    # Job 실행
    run = api_post("/api/2.1/jobs/run-now", {"job_id": job_id})
    run_id = run["run_id"]
    print(f"Job {job_id} 실행 시작 (run_id: {run_id})")

    # 완료 대기
    deadline = time.time() + timeout_minutes * 60
    while time.time() < deadline:
        status = api_get(f"/api/2.1/jobs/runs/get?run_id={run_id}")
        life_cycle = status["state"]["life_cycle_state"]
        result = status["state"].get("result_state")

        if life_cycle == "TERMINATED":
            if result == "SUCCESS":
                print(f"Job 완료: SUCCESS (소요 시간: {status.get('run_duration', 0) // 1000}초)")
                return True
            else:
                print(f"Job 실패: {result}")
                print(f"오류: {status['state'].get('state_message', 'N/A')}")
                return False

        print(f"  상태: {life_cycle}...")
        time.sleep(30)

    print("타임아웃 초과")
    return False

# 사용 예시
success = run_and_wait(job_id=12345, timeout_minutes=30)
```

---

## API 버전 관리

Databricks REST API는 여러 버전이 공존합니다.

| API 버전 | 엔드포인트 예시 | 상태 |
|----------|----------------|------|
| **2.0** | `/api/2.0/clusters/list` | 대부분의 서비스에서 사용 중 |
| **2.1** | `/api/2.1/jobs/list` | Jobs, Unity Catalog 등 신규 기능 |

> 💡 **버전 선택 팁**: 공식 문서의 API Reference에서 각 엔드포인트의 최신 버전을 확인하세요. 새로운 기능은 대부분 2.1 버전에서 제공됩니다.

### Databricks SDK (권장)

REST API를 직접 호출하는 대신, 공식 **Databricks SDK**를 사용하면 더 편리합니다.

```python
# Databricks SDK 설치
# pip install databricks-sdk

from databricks.sdk import WorkspaceClient

# 환경 변수에서 자동으로 인증 (DATABRICKS_HOST, DATABRICKS_TOKEN)
w = WorkspaceClient()

# 클러스터 목록 조회
for cluster in w.clusters.list():
    print(f"{cluster.cluster_name}: {cluster.state}")

# Job 실행
run = w.jobs.run_now(job_id=12345)
print(f"Run ID: {run.run_id}")

# SQL 문 실행
result = w.statement_execution.execute_statement(
    warehouse_id="abcdef1234567890",
    statement="SELECT COUNT(*) FROM catalog.gold.orders"
)
print(result.result.data_array)
```

---

## 정리

| 핵심 개념 | 요약 |
|-----------|------|
| **REST API** | HTTP 요청으로 Databricks 플랫폼의 모든 기능을 제어 |
| **인증** | PAT (개발용), OAuth M2M (프로덕션), SDK 자동 인증 |
| **주요 영역** | Clusters, Jobs, SQL, Unity Catalog, Model Serving 등 |
| **Python 활용** | requests 라이브러리 또는 공식 Databricks SDK 사용 |
| **자동화** | 유휴 클러스터 종료, Job 모니터링, CI/CD 파이프라인 등 |

---

## 참고 링크

- [Databricks REST API Reference](https://docs.databricks.com/api/)
- [인증 가이드](https://docs.databricks.com/aws/en/dev-tools/auth/)
- [Databricks SDK for Python](https://docs.databricks.com/aws/en/dev-tools/sdk-python.html)
- [OAuth M2M 인증 설정](https://docs.databricks.com/aws/en/dev-tools/auth/oauth-m2m.html)
- [Service Principal 관리](https://docs.databricks.com/aws/en/admin/users-groups/service-principals.html)
