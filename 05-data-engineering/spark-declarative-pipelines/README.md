# 05-2. Spark Declarative Pipelines (SDP)

> "무엇을(What)" 정의하면 Databricks가 "어떻게(How)"를 처리해주는 선언적 파이프라인을 학습합니다.
> (이전 명칭: Delta Live Tables / DLT)

## 학습 목표
- SDP의 개념과 기존 Spark 작업 대비 장점 이해
- Streaming Tables과 Materialized Views의 차이와 사용 시나리오
- 데이터 품질 제약조건(Expectations)을 활용한 데이터 검증
- CDC(Change Data Capture) 처리와 SCD Type 2 구현

## 문서 목록

| 순서 | 파일명 | 내용 |
|------|--------|------|
| 1 | `what-is-sdp.md` | SDP란? — 선언적 파이프라인 개념, 명령형 vs 선언형 비교 |
| 2 | `streaming-tables-and-mvs.md` | Streaming Tables & Materialized Views — 차이점, 선택 기준 |
| 3 | `expectations.md` | 데이터 품질 관리 — Expectations를 활용한 데이터 검증 규칙 |
| 4 | `cdc-and-scd.md` | CDC 처리 — APPLY CHANGES, SCD Type 1/2 구현 |
| 5 | `sdp-hands-on.md` | 실습 — Medallion 아키텍처 기반 SDP 파이프라인 구축 |

## 참고 문서
- [Databricks: Spark Declarative Pipelines](https://docs.databricks.com/aws/en/sdp/)
- [Azure Databricks: Delta Live Tables](https://learn.microsoft.com/en-us/azure/databricks/delta-live-tables/)
