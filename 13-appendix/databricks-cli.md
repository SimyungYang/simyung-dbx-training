# Databricks CLI

## 개념

> 💡 **Databricks CLI**는 터미널(명령줄)에서 Databricks를 관리하고 조작할 수 있는 커맨드라인 도구입니다. Workspace, 클러스터, Job, SQL, Asset Bundles 등 대부분의 Databricks 기능을 CLI로 사용할 수 있습니다.

---

## 설치

```bash
# macOS (Homebrew)
brew install databricks/tap/databricks

# Linux / Windows (curl)
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

# 버전 확인
databricks --version
```

---

## 인증 설정

```bash
# 대화형 인증 설정 (OAuth 기반 — 권장)
databricks configure

# 프로필 확인
databricks auth profiles

# 특정 프로필로 명령 실행
databricks clusters list --profile my-workspace

# 환경 변수로 인증 (CI/CD에서 사용)
export DATABRICKS_HOST="https://dbc-abc123.cloud.databricks.com"
export DATABRICKS_TOKEN="dapi..."
```

---

## 주요 명령어

### Workspace 관리

```bash
# 파일/폴더 목록
databricks workspace list /Users/user@company.com

# 노트북 내보내기
databricks workspace export /Users/user@company.com/notebook.py ./local/

# 노트북 가져오기
databricks workspace import ./local/notebook.py /Users/user@company.com/notebook.py
```

### 클러스터 관리

```bash
# 클러스터 목록
databricks clusters list

# 클러스터 시작/정지
databricks clusters start --cluster-id 0123-456789-abc
databricks clusters delete --cluster-id 0123-456789-abc
```

### Job 관리

```bash
# Job 목록
databricks jobs list

# Job 즉시 실행
databricks jobs run-now --job-id 12345

# Job 실행 상태 확인
databricks runs get --run-id 67890
```

### SQL 실행

```bash
# SQL 쿼리 실행
databricks sql execute --warehouse-id abc123 \
    --statement "SELECT COUNT(*) FROM catalog.schema.orders"
```

### Volume 파일 관리

```bash
# Volume 파일 목록
databricks fs ls /Volumes/catalog/schema/volume/

# 파일 업로드
databricks fs cp ./local_file.csv /Volumes/catalog/schema/volume/

# 파일 다운로드
databricks fs cp /Volumes/catalog/schema/volume/file.csv ./local/
```

### Asset Bundles

```bash
# 프로젝트 초기화
databricks bundle init

# 검증
databricks bundle validate

# 배포
databricks bundle deploy -t dev

# 실행
databricks bundle run my_job -t dev

# 정리
databricks bundle destroy -t dev
```

---

## 유용한 옵션

| 옵션 | 설명 |
|------|------|
| `--profile <name>` | 특정 인증 프로필 사용 |
| `--output json` | 결과를 JSON 형식으로 출력 |
| `--debug` | 디버그 로그 출력 |
| `-h` / `--help` | 도움말 표시 |

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **설치** | Homebrew(macOS) 또는 curl(Linux/Windows)로 설치합니다 |
| **인증** | `databricks configure`로 OAuth 기반 인증을 설정합니다 |
| **주요 명령** | workspace, clusters, jobs, sql, fs, bundle 등 |
| **CI/CD** | 환경 변수로 인증하여 자동화 파이프라인에서 사용합니다 |

---

## 참고 링크

- [Databricks: CLI](https://docs.databricks.com/aws/en/dev-tools/cli/)
- [Databricks: CLI commands](https://docs.databricks.com/aws/en/dev-tools/cli/commands.html)
