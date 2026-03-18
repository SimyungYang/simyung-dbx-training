# 11. Lakebase (OLTP 데이터베이스)

> Databricks의 관리형 PostgreSQL 호환 OLTP 데이터베이스인 Lakebase를 학습합니다.

## 학습 목표
- Lakebase의 개념과 기존 OLTP 데이터베이스 대비 장점 이해
- Databricks Apps와의 연동 패턴
- Delta Lake와의 자동 동기화(Data Sync) 이해
- Branching을 통한 개발/테스트 워크플로우

## 문서 목록

| 순서 | 파일명 | 내용 |
|------|--------|------|
| 1 | `what-is-lakebase.md` | Lakebase란? — 개념, PostgreSQL 호환성, 사용 시나리오 |
| 2 | `lakebase-setup.md` | Lakebase 설정 — 인스턴스 생성, 연결, 테이블 관리 |
| 3 | `data-sync.md` | Data Sync — Lakebase ↔ Delta Lake 자동 동기화 |
| 4 | `lakebase-with-apps.md` | Apps 연동 — Databricks Apps에서 Lakebase 활용 |

## 참고 문서
- [Databricks: Lakebase](https://docs.databricks.com/aws/en/lakebase/)
- [Azure Databricks: Lakebase](https://learn.microsoft.com/en-us/azure/databricks/lakebase/)
