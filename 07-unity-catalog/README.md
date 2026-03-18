# 07. Unity Catalog (데이터 거버넌스)

> Databricks의 통합 거버넌스 솔루션인 Unity Catalog를 학습합니다.

## 학습 목표
- Unity Catalog의 3-Level 네임스페이스(Catalog → Schema → Table) 이해
- 데이터 접근 권한(Grants) 관리 방법
- 데이터 리니지(Lineage) 추적
- Delta Sharing을 통한 외부 데이터 공유

## 문서 목록

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [Unity Catalog란?](what-is-unity-catalog.md) | 통합 거버넌스의 필요성과 핵심 개념을 설명합니다 |
| 2 | [네임스페이스 구조](namespace-and-objects.md) | Catalog, Schema, Table, Volume, Function, Model 객체를 다룹니다 |
| 3 | [권한 관리](access-control.md) | GRANT/REVOKE, 행/열 수준 보안, ABAC 태그 기반 접근 제어를 상세히 안내합니다 |
| 4 | [데이터 리니지](data-lineage.md) | 테이블·컬럼 수준 데이터 흐름 추적을 설명합니다 |
| 5 | [Delta Sharing](delta-sharing.md) | 조직 간 안전한 데이터 공유 프로토콜을 다룹니다 |
| 6 | [Volumes](volumes.md) | 비테이블 데이터(파일, 이미지, 모델) 관리를 안내합니다 |

## 참고 문서
- [Databricks: Unity Catalog](https://docs.databricks.com/aws/en/data-governance/)
- [Azure Databricks: Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/)
