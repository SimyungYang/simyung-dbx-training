# 05-2. Spark Declarative Pipelines (SDP)

> "무엇을(What)" 정의하면 Databricks가 "어떻게(How)"를 처리해주는 선언적 파이프라인을 학습합니다.
> (이전 명칭: Delta Live Tables / DLT)

## 학습 목표
- SDP의 개념과 기존 Spark 작업 대비 장점 이해
- Streaming Tables과 Materialized Views의 차이와 사용 시나리오
- 데이터 품질 제약조건(Expectations)을 활용한 데이터 검증
- CDC(Change Data Capture) 처리와 SCD Type 2 구현

## 문서 목록

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [SDP란?](what-is-sdp.md) | 선언적 파이프라인 개념, 구성 요소, 실행 모드, 시스템 제약사항을 설명합니다 |
| 2 | [Streaming Tables & Materialized Views](streaming-tables-and-mvs.md) | 두 테이블 유형의 차이점과 선택 기준을 안내합니다 |
| 3 | [Expectations](expectations.md) | 데이터 품질 관리 — 위반 처리 방식(WARN, DROP, FAIL)을 다룹니다 |
| 4 | [CDC 처리](cdc-and-scd.md) | APPLY CHANGES, SCD Type 1/2 구현, DELETE 동기화를 설명합니다 |
| 5 | [SDP 실습](sdp-hands-on.md) | Medallion 아키텍처 기반 전체 파이프라인을 구축합니다 |

## 참고 문서
- [Databricks: Spark Declarative Pipelines](https://docs.databricks.com/aws/en/sdp/)
- [Azure Databricks: Delta Live Tables](https://learn.microsoft.com/en-us/azure/databricks/delta-live-tables/)
