# 03. 레이크하우스 아키텍처 (Lakehouse Architecture)

> 데이터 웨어하우스와 데이터 레이크의 장점을 결합한 레이크하우스 패러다임을 학습합니다.

## 학습 목표
- 레이크하우스(Lakehouse)가 등장한 이유와 기존 방식의 한계 이해
- Delta Lake의 핵심 기능(ACID 트랜잭션, 타임 트래블, 스키마 관리) 학습
- Medallion 아키텍처(Bronze → Silver → Gold) 설계 패턴 이해
- Delta Lake와 Apache Iceberg의 관계 파악

## 문서 목록

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [레이크하우스란?](what-is-lakehouse.md) | 데이터 웨어하우스 + 데이터 레이크의 통합 아키텍처를 설명합니다 |
| 2 | [Delta Lake 핵심](delta-lake-fundamentals.md) | ACID 트랜잭션, 타임 트래블, 스키마 진화의 원리를 다룹니다 |
| 3 | [Medallion 아키텍처](medallion-architecture.md) | Bronze/Silver/Gold 계층 설계 패턴과 모범 사례를 안내합니다 |
| 4 | [Delta Lake 실전](delta-lake-operations.md) | MERGE, OPTIMIZE, VACUUM, Liquid Clustering 조작법을 다룹니다 |
| 5 | [Delta Lake & Iceberg](delta-and-iceberg.md) | UniForm, Iceberg REST Catalog, 상호 운용성을 설명합니다 |

## 참고 문서
- [Databricks: What is a data lakehouse?](https://docs.databricks.com/aws/en/lakehouse/)
- [Databricks: Delta Lake](https://docs.databricks.com/aws/en/delta/)
- [Azure Databricks: What is Delta Lake?](https://learn.microsoft.com/en-us/azure/databricks/delta/)
