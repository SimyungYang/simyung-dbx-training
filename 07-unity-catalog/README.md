# 07. Unity Catalog (데이터 거버넌스)

> Databricks의 통합 거버넌스 솔루션인 Unity Catalog를 학습합니다.

## 학습 목표
- Unity Catalog의 3-Level 네임스페이스(Catalog → Schema → Table) 이해
- 데이터 접근 권한(Grants) 관리 방법
- 데이터 리니지(Lineage) 추적
- Delta Sharing을 통한 외부 데이터 공유

## 문서 목록

| 순서 | 파일명 | 내용 |
|------|--------|------|
| 1 | `what-is-unity-catalog.md` | Unity Catalog란? — 통합 거버넌스의 필요성과 핵심 개념 |
| 2 | `namespace-and-objects.md` | 네임스페이스 구조 — Catalog, Schema, Table, Volume, Function |
| 3 | `access-control.md` | 권한 관리 — GRANT/REVOKE, 소유권, 행/열 수준 보안 |
| 4 | `data-lineage.md` | 데이터 리니지 — 테이블·컬럼 수준 데이터 흐름 추적 |
| 5 | `delta-sharing.md` | Delta Sharing — 조직 간 안전한 데이터 공유 |
| 6 | `volumes.md` | Volumes — 비테이블 데이터(파일, 이미지, 모델) 관리 |

## 참고 문서
- [Databricks: Unity Catalog](https://docs.databricks.com/aws/en/data-governance/)
- [Azure Databricks: Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/)
