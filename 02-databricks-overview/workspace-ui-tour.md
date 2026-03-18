# Workspace UI 둘러보기

## Workspace에 접속하기

Databricks Workspace는 **웹 브라우저**를 통해 접속합니다. 별도의 소프트웨어 설치가 필요 없으며, Chrome, Edge, Firefox 등 최신 브라우저를 사용하시면 됩니다.

접속 URL은 클라우드 환경에 따라 다음과 같은 형태입니다.

| 클라우드 | URL 형태 | 예시 |
|----------|----------|------|
| AWS | `https://<workspace-id>.cloud.databricks.com` | `https://dbc-abc123.cloud.databricks.com` |
| Azure | `https://adb-<workspace-id>.azuredatabricks.net` | `https://adb-123456.azuredatabricks.net` |
| GCP | `https://<workspace-id>.gcp.databricks.com` | `https://dbc-xyz789.gcp.databricks.com` |

---

## 메인 화면 구성

Workspace에 로그인하면 아래와 같은 구조의 화면이 나타납니다.

```
┌─────────────────────────────────────────────────────────┐
│  🔍 검색 바 (Search)                        👤 사용자   │
├──────────┬──────────────────────────────────────────────┤
│          │                                              │
│  📁 Home │          메인 콘텐츠 영역                      │
│          │                                              │
│  💻 Workspace│     (선택한 메뉴에 따라 변경)                │
│          │                                              │
│  📊 SQL  │                                              │
│          │                                              │
│  🖥️ Compute│                                            │
│          │                                              │
│  ⏰ Jobs │                                              │
│          │                                              │
│  📈 Dashboards│                                         │
│          │                                              │
│  🛡️ Catalog│                                            │
│          │                                              │
│  ⚙️ Settings│                                           │
│          │                                              │
├──────────┴──────────────────────────────────────────────┤
│  하단 상태 바                                             │
└─────────────────────────────────────────────────────────┘
```

---

## 좌측 사이드바 메뉴 상세

### 🏠 Home

Workspace에 로그인했을 때 가장 먼저 보이는 화면입니다. 최근 작업한 노트북, 실행한 쿼리, 자주 사용하는 항목 등이 표시됩니다. 새로운 노트북이나 쿼리를 빠르게 만들 수 있는 버튼도 여기에 있습니다.

### 💻 Workspace

노트북, 파일, 폴더를 관리하는 공간입니다. 파일 탐색기처럼 폴더 구조를 사용하여 노트북과 파일을 정리할 수 있습니다.

| 주요 폴더 | 설명 |
|-----------|------|
| **Users/** | 각 사용자의 개인 폴더입니다. 다른 사용자가 접근할 수 없습니다 |
| **Shared/** | 모든 사용자가 접근할 수 있는 공유 폴더입니다 |
| **Repos/** | Git 저장소와 연동된 프로젝트 폴더입니다 |

### 📊 SQL Editor

SQL 쿼리를 작성하고 실행하는 전용 에디터입니다. 자동 완성, 구문 하이라이팅, 실행 결과 시각화 기능을 제공합니다.

### 🖥️ Compute

클러스터와 SQL Warehouse를 생성하고 관리하는 화면입니다.

| 리소스 유형 | 설명 |
|------------|------|
| **All-Purpose Clusters** | 노트북에서 대화형으로 코드를 실행하기 위한 클러스터입니다 |
| **Job Clusters** | 스케줄된 작업 실행 시 자동으로 생성되는 클러스터입니다 |
| **SQL Warehouses** | SQL 쿼리 실행에 특화된 컴퓨팅 리소스입니다 |

### ⏰ Jobs (Workflows)

데이터 파이프라인과 작업을 스케줄링하고 모니터링하는 화면입니다. 작업의 실행 이력, 성공/실패 현황, 실행 시간 등을 확인할 수 있습니다.

### 📈 Dashboards

AI/BI 대시보드를 생성하고 관리하는 공간입니다. SQL 쿼리 결과를 차트, 표, 텍스트 등으로 시각화할 수 있습니다.

### 🛡️ Catalog

Unity Catalog를 통해 데이터 자산을 탐색하고 관리하는 화면입니다.

```
Catalog (카탈로그)
  └── Schema (스키마)
       ├── Tables (테이블)
       ├── Views (뷰)
       ├── Volumes (볼륨 - 파일 저장)
       ├── Functions (함수)
       └── Models (ML 모델)
```

여기에서 테이블의 스키마(컬럼 정보), 샘플 데이터, 리니지(데이터 흐름), 권한 설정 등을 확인할 수 있습니다.

### ⚙️ Settings

Workspace 설정을 관리하는 영역입니다. 관리자 권한이 필요한 설정도 여기에서 접근합니다.

---

## 자주 사용하는 주요 기능

### 새 노트북 만들기

1. 좌측 사이드바에서 **+ New** 또는 **Home** 화면에서 **New Notebook** 클릭
2. 노트북 이름 입력
3. 기본 언어 선택 (Python, SQL, Scala, R)
4. 클러스터 연결 (Serverless 또는 기존 클러스터)
5. 코드 작성 및 실행 (Shift + Enter 또는 실행 버튼)

### SQL 쿼리 실행

1. 좌측 사이드바에서 **SQL Editor** 클릭
2. SQL Warehouse 선택 (없으면 새로 생성)
3. 쿼리 작성 후 실행 (Ctrl/Cmd + Enter)
4. 결과를 테이블, 차트 등으로 확인

### 클러스터 생성

1. **Compute** 메뉴 클릭
2. **Create Compute** 또는 **Create SQL Warehouse** 클릭
3. 클러스터 이름, 크기, 자동 종료 시간 등 설정
4. **Create** 클릭 → 클러스터가 시작될 때까지 수 분 대기 (Serverless는 수 초)

---

## 핵심 단축키

| 단축키 | 기능 | 사용 위치 |
|--------|------|----------|
| `Shift + Enter` | 현재 셀 실행 후 다음 셀로 이동 | Notebook |
| `Ctrl/Cmd + Enter` | 현재 셀/쿼리 실행 | Notebook, SQL Editor |
| `Ctrl/Cmd + Shift + -` | 셀 분할 | Notebook |
| `Ctrl/Cmd + /` | 주석 토글 | Notebook, SQL Editor |
| `Tab` | 자동 완성 | Notebook, SQL Editor |

---

## 정리

| 핵심 메뉴 | 역할 |
|-----------|------|
| **Home** | 최근 작업, 빠른 시작 |
| **Workspace** | 노트북, 파일, 폴더 관리 |
| **SQL Editor** | SQL 쿼리 작성·실행 |
| **Compute** | 클러스터, SQL Warehouse 관리 |
| **Jobs** | 작업 스케줄링·모니터링 |
| **Dashboards** | 데이터 시각화 |
| **Catalog** | Unity Catalog를 통한 데이터 탐색 |

다음 문서에서는 코드를 작성하고 실행하는 핵심 도구인 **Notebook**의 사용법을 자세히 알아보겠습니다.

---

## 참고 링크

- [Databricks: Navigate the workspace](https://docs.databricks.com/aws/en/workspace/)
- [Azure Databricks: Workspace UI](https://learn.microsoft.com/en-us/azure/databricks/workspace/)
