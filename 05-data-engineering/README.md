# 05. 데이터 엔지니어링 (Data Engineering)

> 실제 데이터 파이프라인을 구축하는 핵심 기술을 학습합니다.
> Databricks의 Lakeflow 제품군을 중심으로 데이터 수집부터 변환, 오케스트레이션까지 다룹니다.

## 학습 목표
- Auto Loader를 사용한 파일 기반 데이터 수집 방법 이해
- Spark Declarative Pipelines(SDP)를 활용한 선언적 파이프라인 구축
- Lakeflow Connect를 통한 외부 소스 데이터 수집(CDC 포함)
- Lakeflow Jobs를 활용한 워크플로우 오케스트레이션

## 문서 목록 (이 폴더)

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [데이터 엔지니어링 전체 그림](data-engineering-overview.md) | Lakeflow 제품군 소개와 각 역할을 설명합니다 |
| 2 | [수집 방법 선택 가이드](choosing-ingestion-method.md) | Auto Loader vs Lakeflow Connect vs 커스텀 비교를 안내합니다 |

## 하위 섹션

### 📂 [Auto Loader](auto-loader/README.md)
파일 기반 데이터 수집의 기본 — 클라우드 스토리지에서 자동으로 새 파일 감지 및 수집

| 문서 | 내용 |
|------|------|
| [Auto Loader란?](auto-loader/what-is-auto-loader.md) | 개념, 동작 방식, 사용 시나리오를 설명합니다 |
| [주요 옵션](auto-loader/auto-loader-options.md) | 파일 포맷별 설정, 에러 처리, 메타데이터 컬럼을 다룹니다 |
| [Auto Loader 실습](auto-loader/auto-loader-hands-on.md) | SDP 기반 수집 파이프라인을 구축합니다 |

### 📂 [Spark Declarative Pipelines](spark-declarative-pipelines/README.md)
선언적 파이프라인(SDP, 구 DLT) — Streaming Tables, Materialized Views, CDC 처리

| 문서 | 내용 |
|------|------|
| [SDP란?](spark-declarative-pipelines/what-is-sdp.md) | 선언적 파이프라인 개념, 구성 요소, 시스템 제약사항을 설명합니다 |
| [Streaming Tables & Materialized Views](spark-declarative-pipelines/streaming-tables-and-mvs.md) | 두 테이블 유형의 차이점과 선택 기준을 안내합니다 |
| [Expectations](spark-declarative-pipelines/expectations.md) | 데이터 품질 관리 — 위반 처리 방식을 다룹니다 |
| [CDC 처리](spark-declarative-pipelines/cdc-and-scd.md) | APPLY CHANGES, SCD Type 1/2 구현을 설명합니다 |
| [SDP 실습](spark-declarative-pipelines/sdp-hands-on.md) | Medallion 아키텍처 기반 파이프라인을 구축합니다 |

### 📂 [Lakeflow Connect](lakeflow-connect/README.md)
외부 데이터 소스 연결 — SaaS, 데이터베이스, 메시지큐 등에서 데이터 수집

| 문서 | 내용 |
|------|------|
| [Lakeflow Connect란?](lakeflow-connect/what-is-lakeflow-connect.md) | 개념, 지원 소스, 동작 방식을 설명합니다 |
| [수집 파이프라인 구성](lakeflow-connect/ingestion-pipeline-setup.md) | 소스 연결, 테이블 매핑, 스케줄 설정을 안내합니다 |
| [Lakeflow Connect 실습](lakeflow-connect/lakeflow-connect-hands-on.md) | 외부 DB에서 CDC 기반 수집을 구성합니다 |

### 📂 [Lakeflow Jobs](lakeflow-jobs/README.md)
워크플로우 오케스트레이션 — 작업 스케줄링, 의존성 관리, 모니터링

| 문서 | 내용 |
|------|------|
| [Lakeflow Jobs란?](lakeflow-jobs/what-is-lakeflow-jobs.md) | 워크플로우 오케스트레이션 개념과 구성 요소를 설명합니다 |
| [작업 구성](lakeflow-jobs/job-configuration.md) | 태스크, 의존성, 클러스터 정책, 파라미터를 다룹니다 |
| [스케줄링과 트리거](lakeflow-jobs/scheduling-and-triggers.md) | Cron, 파일 도착 트리거, Continuous 모드를 안내합니다 |
| [모니터링과 알림](lakeflow-jobs/monitoring-and-alerting.md) | 실행 이력, 알림, 비용 추적을 설명합니다 |

## 참고 문서
- [Databricks: Data Engineering](https://docs.databricks.com/aws/en/data-engineering)
- [Azure Databricks: Data Engineering](https://learn.microsoft.com/en-us/azure/databricks/data-engineering/)
