# Databricks CLI

## 개념

> 💡 **Databricks CLI**는 터미널(명령줄)에서 Databricks를 관리하고 조작할 수 있는 커맨드라인 도구입니다.

---

## 설치 및 인증

```bash
# 설치 (macOS)
brew install databricks/tap/databricks

# 인증 설정
databricks configure

# 프로필 확인
databricks auth profiles
```

---

## 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `databricks workspace list /` | 워크스페이스 파일 목록 조회 |
| `databricks clusters list` | 클러스터 목록 조회 |
| `databricks jobs list` | Job 목록 조회 |
| `databricks fs ls /Volumes/catalog/schema/vol/` | Volume 파일 목록 |
| `databricks sql execute --statement "SELECT 1"` | SQL 실행 |
| `databricks bundle deploy` | Asset Bundle 배포 |

---

## 참고 링크

- [Databricks: CLI](https://docs.databricks.com/aws/en/dev-tools/cli/)
