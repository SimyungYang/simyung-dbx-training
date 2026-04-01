# Databricks CLI

## 개념

> 💡 **Databricks CLI** 는 터미널(명령줄)에서 Databricks를 관리하고 조작할 수 있는 커맨드라인 도구입니다. Workspace, 클러스터, Job, SQL, Asset Bundles 등 대부분의 Databricks 기능을 CLI로 사용할 수 있습니다.

GUI(웹 UI)만으로도 Databricks를 사용할 수 있지만, CLI를 활용하면 ** 반복 작업 자동화, CI/CD 파이프라인 구축, 스크립트 기반 관리** 등 더 효율적인 운영이 가능합니다. 특히 여러 Workspace를 관리하거나 배포 파이프라인을 구축할 때 CLI는 필수적인 도구입니다.

---

## 설치

Databricks CLI는 OS에 따라 다양한 방법으로 설치할 수 있습니다. 설치 후 `databricks --version`으로 설치를 확인합니다.

```bash
# macOS (Homebrew) — 가장 권장
brew install databricks/tap/databricks

# Linux / Windows (curl)
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

# 버전 확인
databricks --version
# Databricks CLI v0.237.0

# CLI 업데이트
brew upgrade databricks/tap/databricks    # macOS
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh  # Linux
```

> ⚠️ ** 주의**: Databricks CLI v2 (현재 버전)와 레거시 CLI (Python 패키지 `databricks-cli`)는 완전히 다른 도구입니다. 레거시 CLI는 더 이상 권장되지 않습니다. `pip install databricks-cli`로 설치한 것이 있다면 제거하고 새 CLI를 사용하세요.

---

## 인증 설정

Databricks CLI를 사용하려면 먼저 인증을 설정해야 합니다. OAuth 기반 인증이 권장되며, 여러 Workspace를 프로필로 관리할 수 있습니다.

### OAuth 인증 (권장)

```bash
# 대화형 인증 설정 — 브라우저가 열리며 OAuth 로그인을 수행합니다
databricks configure

# 입력 항목:
# Databricks Host: https://dbc-abc123.cloud.databricks.com
# (브라우저에서 OAuth 로그인 완료)
```

### Personal Access Token (PAT) 인증

```bash
# PAT 토큰 기반 인증 — OAuth가 어려운 환경(CI/CD 등)에서 사용합니다
databricks configure --token

# 입력 항목:
# Databricks Host: https://dbc-abc123.cloud.databricks.com
# Personal Access Token: dapi...
```

### 환경 변수 인증 (CI/CD)

CI/CD 파이프라인에서는 환경 변수로 인증 정보를 전달합니다. 별도의 설정 파일이 필요 없어 자동화에 적합합니다.

```bash
# 환경 변수로 인증 (CI/CD에서 사용)
export DATABRICKS_HOST="https://dbc-abc123.cloud.databricks.com"
export DATABRICKS_TOKEN="dapi..."

# 환경 변수가 설정되면 별도 프로필 없이 바로 명령 실행 가능
databricks clusters list
```

---

## 프로필 관리

여러 Workspace를 사용하는 경우, ** 프로필** 을 통해 각 Workspace의 인증 정보를 관리합니다. 프로필은 `~/.databrickscfg` 파일에 저장됩니다.

```bash
# 프로필 목록 확인
databricks auth profiles

# 출력 예시:
# Name          Host                                        Valid
# DEFAULT       https://dbc-abc123.cloud.databricks.com     YES
# staging       https://dbc-def456.cloud.databricks.com     YES
# production    https://dbc-ghi789.cloud.databricks.com     YES

# 특정 프로필로 명령 실행
databricks clusters list --profile production

# 인증 상태 확인
databricks auth token --profile production

# 새 프로필 추가
databricks configure --profile staging
```

### 프로필 설정 파일 구조

```ini
# ~/.databrickscfg
[DEFAULT]
host = https://dbc-abc123.cloud.databricks.com
auth_type = databricks-cli

[staging]
host = https://dbc-def456.cloud.databricks.com
auth_type = databricks-cli

[production]
host = https://dbc-ghi789.cloud.databricks.com
token = dapi...
```

---

## 주요 명령어

### Workspace 관리

Workspace의 파일과 폴더를 관리합니다. 노트북, 파일, 디렉토리의 목록 조회, 내보내기, 가져오기가 가능합니다.

```bash
# 파일/폴더 목록
databricks workspace list /Users/user@company.com

# 재귀적 목록 (하위 폴더 포함)
databricks workspace list /Users/user@company.com --recursive

# 노트북 내보내기
databricks workspace export /Users/user@company.com/notebook.py ./local/

# 폴더 전체 내보내기
databricks workspace export-dir /Users/user@company.com/project ./local/project

# 노트북 가져오기
databricks workspace import ./local/notebook.py /Users/user@company.com/notebook.py

# 폴더 전체 가져오기
databricks workspace import-dir ./local/project /Users/user@company.com/project

# 파일/폴더 삭제
databricks workspace delete /Users/user@company.com/old-notebook --recursive
```

### 클러스터 관리

클러스터의 생성, 시작, 정지, 상태 확인 등을 수행합니다.

```bash
# 클러스터 목록
databricks clusters list

# 클러스터 상세 정보
databricks clusters get --cluster-id 0123-456789-abc

# 클러스터 시작
databricks clusters start --cluster-id 0123-456789-abc

# 클러스터 정지 (delete는 terminate을 의미)
databricks clusters delete --cluster-id 0123-456789-abc

# 클러스터 생성 (JSON 설정 파일 사용)
databricks clusters create --json '{
    "cluster_name": "my-dev-cluster",
    "spark_version": "15.4.x-scala2.12",
    "node_type_id": "m5.xlarge",
    "num_workers": 2,
    "autotermination_minutes": 30
}'

# 클러스터 이벤트 조회
databricks clusters events --cluster-id 0123-456789-abc
```

### Job 관리

Lakeflow Jobs의 생성, 실행, 모니터링을 수행합니다.

```bash
# Job 목록
databricks jobs list

# Job 상세 정보
databricks jobs get --job-id 12345

# Job 즉시 실행
databricks jobs run-now --job-id 12345

# 파라미터와 함께 실행
databricks jobs run-now --job-id 12345 --json '{
    "notebook_params": {
        "date": "2025-03-01",
        "mode": "full_refresh"
    }
}'

# Job 실행 상태 확인
databricks runs get --run-id 67890

# 최근 실행 목록
databricks runs list --job-id 12345 --limit 10

# 실행 취소
databricks runs cancel --run-id 67890

# Job 삭제
databricks jobs delete --job-id 12345
```

### SQL 실행

SQL Warehouse를 통해 SQL 쿼리를 실행합니다.

```bash
# SQL 쿼리 실행
databricks sql execute --warehouse-id abc123 \
    --statement "SELECT COUNT(*) FROM catalog.schema.orders"

# 결과를 JSON 형식으로 출력
databricks sql execute --warehouse-id abc123 \
    --statement "SELECT * FROM catalog.schema.orders LIMIT 5" \
    --output json

# SQL Warehouse 목록
databricks warehouses list

# SQL Warehouse 시작/정지
databricks warehouses start --id abc123
databricks warehouses stop --id abc123
```

### Volume 파일 관리

Unity Catalog Volume의 파일을 관리합니다. 로컬 파일을 업로드하거나, Volume에서 파일을 다운로드할 수 있습니다.

```bash
# Volume 파일 목록
databricks fs ls /Volumes/catalog/schema/volume/

# 파일 업로드
databricks fs cp ./local_file.csv /Volumes/catalog/schema/volume/

# 디렉토리 전체 업로드 (재귀)
databricks fs cp ./local_dir/ /Volumes/catalog/schema/volume/data/ --recursive

# 파일 다운로드
databricks fs cp /Volumes/catalog/schema/volume/file.csv ./local/

# 디렉토리 전체 다운로드
databricks fs cp /Volumes/catalog/schema/volume/data/ ./local/data/ --recursive

# 파일 삭제
databricks fs rm /Volumes/catalog/schema/volume/old_file.csv

# 파일 내용 확인
databricks fs cat /Volumes/catalog/schema/volume/config.json
```

### Secrets 관리

비밀번호, API 키 등 민감한 정보를 안전하게 저장하고 관리합니다. Secrets는 노트북이나 Job에서 참조할 수 있지만, 평문으로 노출되지 않습니다.

```bash
# Secret Scope 생성
databricks secrets create-scope my-secrets

# Secret 저장 (대화형 입력)
databricks secrets put-secret my-secrets db-password

# Secret 저장 (파일에서 읽기)
databricks secrets put-secret my-secrets api-key --string-value "sk-abc123..."

# Secret 목록 확인 (값은 표시되지 않음)
databricks secrets list-secrets my-secrets

# Secret Scope 목록
databricks secrets list-scopes

# Secret 삭제
databricks secrets delete-secret my-secrets old-key

# Secret Scope 삭제
databricks secrets delete-scope old-scope
```

```python
# 노트북에서 Secret 사용
password = dbutils.secrets.get(scope="my-secrets", key="db-password")
```

### Asset Bundles

프로젝트를 코드로 정의하고, 환경별로 배포합니다. CI/CD 파이프라인의 핵심 도구입니다.

```bash
# 프로젝트 초기화 (템플릿 선택)
databricks bundle init

# 기본 Python 템플릿으로 초기화
databricks bundle init default-python

# 설정 파일 검증
databricks bundle validate

# 로컬 변경사항 동기화 (개발 중)
databricks bundle deploy -t dev

# Job 실행
databricks bundle run my_job -t dev

# 특정 환경에 배포
databricks bundle deploy -t staging
databricks bundle deploy -t production

# 리소스 정리 (삭제)
databricks bundle destroy -t dev

# 현재 번들 상태 요약
databricks bundle summary -t dev
```

---

## 유용한 옵션

| 옵션 | 설명 | 사용 예시 |
|------|------|----------|
| `--profile <name>` | 특정 인증 프로필 사용 | `databricks clusters list --profile prod` |
| `--output json` | 결과를 JSON 형식으로 출력 | `databricks jobs list --output json` |
| `--output text` | 결과를 텍스트 테이블로 출력 | `databricks clusters list --output text` |
| `--debug` | 디버그 로그 출력 (문제 진단 시) | `databricks clusters list --debug` |
| `-h` / `--help` | 도움말 표시 | `databricks jobs --help` |
| `--log-level` | 로그 수준 설정 | `databricks bundle deploy --log-level debug` |

---

## 자주 사용하는 워크플로우

### 1. 개발 환경 빠른 설정

```bash
# 새 프로젝트 시작
databricks bundle init default-python
cd my-project

# 개발 환경에 배포
databricks bundle deploy -t dev

# 코드 수정 후 재배포 + 실행
databricks bundle deploy -t dev && databricks bundle run my_job -t dev
```

### 2. 프로덕션 배포 (CI/CD)

```bash
# CI/CD 파이프라인에서 실행
export DATABRICKS_HOST="https://dbc-prod.cloud.databricks.com"
export DATABRICKS_TOKEN="${PROD_TOKEN}"

# 검증 → 배포
databricks bundle validate -t production
databricks bundle deploy -t production
```

### 3. 대량 데이터 업로드

```bash
# 로컬 디렉토리의 모든 파일을 Volume에 업로드
databricks fs cp ./data/ /Volumes/catalog/schema/raw_data/ --recursive --overwrite
```

### 4. 클러스터 비용 관리

```bash
# 실행 중인 클러스터 확인
databricks clusters list --output json | python3 -c "
import json, sys
clusters = json.load(sys.stdin).get('clusters', [])
running = [c for c in clusters if c.get('state') == 'RUNNING']
for c in running:
    print(f\"{c['cluster_name']}: {c['cluster_id']} ({c.get('state')})\")
print(f'실행 중: {len(running)}개')
"
```

### 5. Job 실행 결과 모니터링

```bash
# 최근 실패한 Job Run 확인
databricks runs list --job-id 12345 --limit 5 --output json | python3 -c "
import json, sys
runs = json.load(sys.stdin).get('runs', [])
for r in runs:
    state = r.get('state', {}).get('result_state', 'N/A')
    print(f\"Run {r['run_id']}: {state}\")
"
```

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **설치**| Homebrew(macOS) 또는 curl(Linux/Windows)로 설치합니다. 레거시 CLI와 혼동하지 마세요 |
| ** 인증**| OAuth(권장) 또는 PAT 기반으로 설정합니다. CI/CD에서는 환경 변수를 사용합니다 |
| ** 프로필**| 여러 Workspace의 인증을 `~/.databrickscfg`에서 프로필로 관리합니다 |
| **Workspace**| 노트북과 파일의 내보내기/가져오기를 수행합니다 |
| **Clusters**| 클러스터의 생성, 시작, 정지, 상태 조회를 수행합니다 |
| **Jobs**| Job의 생성, 실행, 모니터링, 파라미터 전달을 수행합니다 |
| **Secrets**| 민감한 정보를 암호화하여 안전하게 저장하고 관리합니다 |
| **Asset Bundles** | YAML로 프로젝트를 정의하고, 환경별로 배포합니다. CI/CD의 핵심입니다 |

---

## 참고 링크

- [Databricks: CLI](https://docs.databricks.com/aws/en/dev-tools/cli/)
- [Databricks: CLI commands](https://docs.databricks.com/aws/en/dev-tools/cli/commands.html)
- [Databricks: CLI authentication](https://docs.databricks.com/aws/en/dev-tools/cli/authentication.html)
- [Databricks: Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)
- [Databricks: Secrets CLI](https://docs.databricks.com/aws/en/dev-tools/cli/secrets-cli.html)
