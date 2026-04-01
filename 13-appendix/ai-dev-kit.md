# Databricks AI Dev Kit

## AI Dev Kit란?

Databricks AI Dev Kit는 **AI 코딩 어시스턴트**(Claude Code, VS Code Copilot, Cursor 등)와 **Databricks 플랫폼**을 연결하는 통합 도구입니다. MCP(Model Context Protocol) 서버를 통해 AI 어시스턴트가 Databricks의 리소스에 직접 접근할 수 있게 해주며, 자연어 명령만으로 SQL 실행, 클러스터 관리, 파이프라인 생성, 대시보드 구축 등의 작업을 수행할 수 있습니다.

> 💡 **MCP(Model Context Protocol)**란?
> Anthropic이 제안한 개방형 프로토콜로, AI 모델이 외부 도구·데이터 소스와 표준화된 방식으로 소통할 수 있게 합니다. AI Dev Kit는 이 프로토콜을 활용하여 Databricks API를 AI 어시스턴트의 "도구(tool)"로 노출합니다.

### 왜 AI Dev Kit가 필요한가?

기존에는 Databricks 작업을 수행하려면 UI에 접속하거나 CLI/SDK를 직접 호출해야 했습니다. AI Dev Kit를 사용하면 개발자가 **코딩 환경을 떠나지 않고**자연어로 Databricks 플랫폼과 상호작용할 수 있습니다.

| 기존 방식 | AI Dev Kit 방식 |
|---|---|
| Databricks UI에서 SQL 에디터 열기 | "매출 테이블에서 월별 합계를 조회해줘" |
| CLI로 클러스터 시작 명령 실행 | "개발용 클러스터를 시작해줘" |
| 노트북에서 파이프라인 코드 작성 | "bronze→silver 변환 파이프라인을 만들어줘" |
| 대시보드 UI에서 차트 수동 구성 | "매출 트렌드 대시보드를 생성해줘" |

---

## 지원하는 AI 코딩 도구

AI Dev Kit는 MCP를 지원하는 다양한 AI 코딩 어시스턴트와 통합됩니다.

| 도구 | 설정 방식 | 비고 |
|---|---|---|
| **Claude Code**| `claude mcp add` 명령 또는 `.mcp.json` | Anthropic CLI 기반 |
| **VS Code (GitHub Copilot)**| `.vscode/mcp.json` 설정 | Copilot Chat Agent 모드 |
| **Cursor**| `.cursor/mcp.json` 설정 | Cursor Composer에서 활용 |
| **Windsurf**| MCP 설정 파일 | Cascade에서 활용 |
| **Claude Desktop**| `claude_desktop_config.json` | 데스크톱 앱 |

---

## 설치 및 설정

### 1단계: 사전 요구사항

```bash
# Python 3.10 이상 필요
python --version

# Databricks CLI 설치 (인증 설정용)
brew install databricks/tap/databricks

# Databricks 인증 프로파일 설정
databricks auth login --host https://<workspace-url>
```

### 2단계: AI Dev Kit 설치

```bash
# pip로 설치
pip install databricks-ai-dev-kit

# 또는 uvx로 직접 실행 (설치 없이)
uvx databricks-ai-dev-kit
```

### 3단계: AI 코딩 도구별 설정

#### Claude Code 설정

```bash
# Claude Code에 Databricks MCP 서버 추가
claude mcp add databricks \
  -- uvx databricks-ai-dev-kit@latest
```

또는 프로젝트 루트에 `.mcp.json` 파일을 생성합니다.

```json
{
  "mcpServers": {
    "databricks": {
      "command": "uvx",
      "args": ["databricks-ai-dev-kit@latest"],
      "env": {
        "DATABRICKS_HOST": "https://<workspace-url>",
        "DATABRICKS_PROFILE": "DEFAULT"
      }
    }
  }
}
```

#### VS Code (GitHub Copilot) 설정

프로젝트 루트에 `.vscode/mcp.json` 파일을 생성합니다.

```json
{
  "servers": {
    "databricks": {
      "command": "uvx",
      "args": ["databricks-ai-dev-kit@latest"],
      "env": {
        "DATABRICKS_HOST": "https://<workspace-url>"
      }
    }
  }
}
```

#### Cursor 설정

프로젝트 루트에 `.cursor/mcp.json` 파일을 생성합니다.

```json
{
  "mcpServers": {
    "databricks": {
      "command": "uvx",
      "args": ["databricks-ai-dev-kit@latest"],
      "env": {
        "DATABRICKS_HOST": "https://<workspace-url>"
      }
    }
  }
}
```

### 인증 방법

AI Dev Kit는 Databricks CLI의 통합 인증(Unified Auth)을 사용합니다. 아래 방법 중 하나를 선택할 수 있습니다.

| 인증 방법 | 환경 변수 | 권장 용도 |
|---|---|---|
| **OAuth (U2M)**| `DATABRICKS_HOST` | 개인 개발 환경 (권장) |
| **PAT**| `DATABRICKS_HOST`, `DATABRICKS_TOKEN` | CI/CD, 자동화 |
| **OAuth (M2M)**| `DATABRICKS_HOST`, `DATABRICKS_CLIENT_ID`, `DATABRICKS_CLIENT_SECRET` | 서비스 간 연동 |
| **Profile** | `DATABRICKS_PROFILE` | 여러 워크스페이스 전환 |

```bash
# OAuth 로그인 (권장)
databricks auth login --host https://<workspace-url>

# 또는 환경 변수로 PAT 설정
export DATABRICKS_HOST="https://<workspace-url>"
export DATABRICKS_TOKEN="dapi..."
```

---

## 주요 기능

AI Dev Kit는 MCP 서버를 통해 다양한 Databricks 도구(tool)를 AI 어시스턴트에 노출합니다. 주요 기능 영역은 다음과 같습니다.

### SQL 실행

자연어로 데이터를 조회하고 분석할 수 있습니다.

```
사용자: "지난 30일간 일별 주문 건수와 매출 합계를 조회해줘"

AI Dev Kit → execute_sql 도구 호출:
SELECT
  date_trunc('day', order_date) AS order_day,
  COUNT(*) AS order_count,
  SUM(total_amount) AS total_revenue
FROM catalog.schema.orders
WHERE order_date >= current_date() - INTERVAL 30 DAYS
GROUP BY 1
ORDER BY 1
```

### 클러스터 관리

클러스터 상태 확인, 시작, 중지를 자연어로 수행합니다.

```
사용자: "현재 실행 중인 클러스터 목록을 보여줘"
→ list_clusters 도구 호출

사용자: "개발용 클러스터를 시작해줘"
→ start_cluster 도구 호출
```

### 파이프라인 관리

SDP(선언적 파이프라인)를 생성하고 관리합니다.

```
사용자: "S3의 CSV 파일을 읽어서 bronze 테이블로 적재하는 파이프라인을 만들어줘"
→ create_or_update_pipeline 도구 호출
```

### 대시보드 생성

Lakeview 대시보드를 자연어로 생성할 수 있습니다.

```
사용자: "매출 트렌드와 카테고리별 비중을 보여주는 대시보드를 만들어줘"
→ create_or_update_dashboard 도구 호출
```

### Unity Catalog 관리

카탈로그, 스키마, 테이블 구조를 탐색하고 권한을 관리합니다.

```
사용자: "production 카탈로그에 있는 모든 테이블을 보여줘"
→ manage_uc_objects 도구 호출

사용자: "sales 테이블의 컬럼 정보를 알려줘"
→ get_table_details 도구 호출
```

### 전체 도구 목록

| 카테고리 | 주요 도구 | 설명 |
|---|---|---|
| **SQL**| `execute_sql`, `execute_sql_multi` | SQL 쿼리 실행 및 결과 반환 |
| ** 클러스터**| `list_clusters`, `start_cluster`, `get_cluster_status` | 클러스터 관리 |
| **Job**| `manage_jobs`, `manage_job_runs` | 워크플로 Job 관리 및 실행 |
| ** 파이프라인**| `create_or_update_pipeline`, `start_update` | SDP 파이프라인 관리 |
| ** 대시보드**| `create_or_update_dashboard`, `publish_dashboard` | Lakeview 대시보드 관리 |
| **Unity Catalog**| `manage_uc_objects`, `manage_uc_grants` | 카탈로그/스키마/테이블/권한 관리 |
| **Vector Search**| `create_or_update_vs_index`, `query_vs_index` | 벡터 검색 인덱스 관리 |
| **Genie**| `ask_genie`, `create_or_update_genie` | Genie 스페이스 관리 |
| ** 볼륨**| `upload_to_volume`, `list_volume_files` | Unity Catalog 볼륨 파일 관리 |
| ** 서빙**| `query_serving_endpoint` | 모델 서빙 엔드포인트 호출 |
| **Lakebase**| `create_or_update_lakebase_database` | Lakebase DB 관리 |
| ** 앱**| `create_or_update_app` | Databricks 앱 관리 |

---

## 활용 사례

### 사례 1: 자연어로 데이터 파이프라인 구축

```
사용자: "S3 버킷 s3://raw-data/events/ 에서 JSON 파일을 읽어
bronze_events 테이블로 적재하고,
PII 컬럼을 마스킹한 silver_events 테이블을 만들어줘"
```

AI Dev Kit는 이 요청을 분석하여 SDP 파이프라인 코드를 생성하고, `create_or_update_pipeline` 도구로 Databricks에 배포합니다.

### 사례 2: 데이터 탐색 및 분석

```
사용자: "customers 테이블의 구조를 보여주고,
가입월별 고객 수 추이를 차트로 시각화해줘"
```

AI Dev Kit가 테이블 메타데이터를 조회하고, SQL을 실행한 뒤, 대시보드를 생성하는 과정을 자동으로 수행합니다.

### 사례 3: 운영 모니터링

```
사용자: "지금 실행 중인 Job 중 1시간 넘게 걸리는 것이 있어?
있으면 상세 정보를 보여줘"
```

---

## Databricks Asset Bundles와 연동

AI Dev Kit는 `databricks.yml` 파일이 있는 프로젝트에서 자동으로 번들 설정을 인식합니다. 이를 통해 대상 워크스페이스, 카탈로그, 스키마 등의 컨텍스트를 자동으로 파악합니다.

```yaml
# databricks.yml 예시
bundle:
  name: my-data-project

workspace:
  host: https://<workspace-url>

resources:
  pipelines:
    my_pipeline:
      name: "ETL Pipeline"
      target: "my_catalog.my_schema"
```

> 💡 ** 팁**: `databricks.yml`이 프로젝트에 있으면 AI Dev Kit가 자동으로 대상 환경을 인식하므로, "이 프로젝트의 파이프라인을 실행해줘"처럼 간결한 명령이 가능합니다.

---

## 모범 사례

### 효과적인 프롬프트 작성

| 비효율적 | 효과적 |
|---|---|
| "데이터 보여줘" | "catalog.schema.orders 테이블에서 최근 7일 매출 합계를 조회해줘" |
| "파이프라인 만들어" | "S3의 JSON 파일을 Auto Loader로 읽어서 bronze 테이블에 적재하는 SDP 파이프라인을 만들어줘" |
| "대시보드" | "월별 매출 추이(선 그래프)와 카테고리별 비중(파이 차트)을 포함한 대시보드를 만들어줘" |

### 보안 고려사항

- AI Dev Kit는 **사용자의 Databricks 권한**을 그대로 상속합니다. Unity Catalog 권한이 적용되므로 접근 권한이 없는 데이터는 조회할 수 없습니다.
- PAT(개인 접근 토큰)보다 **OAuth 인증**을 권장합니다. 토큰 만료 관리가 자동으로 이루어집니다.
- `.mcp.json` 파일에 토큰을 직접 작성하지 마세요. 환경 변수나 프로파일을 사용하세요.
- `.gitignore`에 민감한 설정 파일이 포함되어 있는지 확인하세요.

---

## 정리

| 항목 | 내용 |
|---|---|
| **핵심 가치**| AI 코딩 어시스턴트에서 자연어로 Databricks 플랫폼 제어 |
| ** 기반 기술**| MCP (Model Context Protocol) |
| ** 지원 도구**| Claude Code, VS Code, Cursor, Windsurf, Claude Desktop |
| ** 인증**| Databricks Unified Auth (OAuth, PAT, Service Principal) |
| ** 권한** | Unity Catalog 권한을 그대로 상속 |

---

## 참고 링크

- [Databricks AI Dev Kit 공식 문서](https://docs.databricks.com/aws/en/dev-tools/ai-dev-kit.html)
- [Databricks AI Dev Kit PyPI](https://pypi.org/project/databricks-ai-dev-kit/)
- [MCP (Model Context Protocol) 소개](https://modelcontextprotocol.io/)
- [Databricks 통합 인증 가이드](https://docs.databricks.com/aws/en/dev-tools/auth/unified-auth-client.html)
- [Databricks Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)
