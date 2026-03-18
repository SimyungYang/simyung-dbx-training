# 03. 레이크하우스 아키텍처 (Lakehouse Architecture)

> 데이터 웨어하우스와 데이터 레이크의 장점을 결합한 레이크하우스 패러다임을 학습합니다.

## 학습 목표
- 레이크하우스(Lakehouse)가 등장한 이유와 기존 방식의 한계 이해
- Delta Lake의 핵심 기능(ACID 트랜잭션, 타임 트래블, 스키마 관리) 학습
- Medallion 아키텍처(Bronze → Silver → Gold) 설계 패턴 이해
- Delta Lake와 Apache Iceberg의 관계 파악

## 문서 목록

| 순서 | 파일명 | 내용 |
|------|--------|------|
| 1 | `what-is-lakehouse.md` | 레이크하우스란? — 데이터 웨어하우스 + 데이터 레이크의 통합 |
| 2 | `delta-lake-fundamentals.md` | Delta Lake 핵심 — ACID 트랜잭션, 타임 트래블, 스키마 진화 |
| 3 | `medallion-architecture.md` | Medallion 아키텍처 — Bronze/Silver/Gold 계층 설계 패턴 |
| 4 | `delta-lake-operations.md` | Delta Lake 실전 — MERGE, UPDATE, DELETE, OPTIMIZE, VACUUM |
| 5 | `delta-and-iceberg.md` | Delta Lake & Iceberg — UniForm, 외부 엔진 연동, 상호 운용성 |

## 참고 문서
- [Databricks: What is a data lakehouse?](https://docs.databricks.com/aws/en/lakehouse/)
- [Databricks: Delta Lake](https://docs.databricks.com/aws/en/delta/)
- [Azure Databricks: What is Delta Lake?](https://learn.microsoft.com/en-us/azure/databricks/delta/)
