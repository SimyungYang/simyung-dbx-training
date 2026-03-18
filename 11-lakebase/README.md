# 11. Lakebase (OLTP 데이터베이스)

> Databricks의 관리형 PostgreSQL 호환 OLTP 데이터베이스인 Lakebase를 학습합니다.

## 학습 목표
- Lakebase의 개념과 기존 OLTP 데이터베이스 대비 장점 이해
- Databricks Apps와의 연동 패턴
- Delta Lake와의 자동 동기화(Data Sync) 이해
- Branching을 통한 개발/테스트 워크플로우

## 문서 목록

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [Lakebase란?](what-is-lakebase.md) | 개념, PostgreSQL 호환성, 오토스케일링, 사용 시나리오를 설명합니다 |
| 2 | [Lakebase 설정](lakebase-setup.md) | 인스턴스 생성, 연결, 테이블 관리를 안내합니다 |
| 3 | [Data Sync](data-sync.md) | Lakebase ↔ Delta Lake 자동 동기화를 설명합니다 |
| 4 | [Apps 연동](lakebase-with-apps.md) | Databricks Apps에서 Lakebase를 백엔드로 활용하는 방법을 다룹니다 |

## 참고 문서
- [Databricks: Lakebase](https://docs.databricks.com/aws/en/lakebase/)
- [Azure Databricks: Lakebase](https://learn.microsoft.com/en-us/azure/databricks/lakebase/)
