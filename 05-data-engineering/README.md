# 05. 데이터 엔지니어링 (Data Engineering)

> 실제 데이터 파이프라인을 구축하는 핵심 기술을 학습합니다.
> Databricks의 Lakeflow 제품군을 중심으로 데이터 수집부터 변환, 오케스트레이션까지 다룹니다.

## 학습 목표
- Auto Loader를 사용한 파일 기반 데이터 수집 방법 이해
- Spark Declarative Pipelines(SDP)를 활용한 선언적 파이프라인 구축
- Lakeflow Connect를 통한 외부 소스 데이터 수집(CDC 포함)
- Lakeflow Jobs를 활용한 워크플로우 오케스트레이션

## 하위 섹션

### 📂 [auto-loader/](./auto-loader/)
파일 기반 데이터 수집의 기본 — 클라우드 스토리지에서 자동으로 새 파일 감지 및 수집

### 📂 [spark-declarative-pipelines/](./spark-declarative-pipelines/)
선언적 파이프라인(SDP, 구 DLT) — Streaming Tables, Materialized Views, CDC 처리

### 📂 [lakeflow-connect/](./lakeflow-connect/)
외부 데이터 소스 연결 — SaaS, 데이터베이스, 메시지큐 등에서 데이터 수집

### 📂 [lakeflow-jobs/](./lakeflow-jobs/)
워크플로우 오케스트레이션 — 작업 스케줄링, 의존성 관리, 모니터링

## 문서 목록 (이 폴더)

| 순서 | 파일명 | 내용 |
|------|--------|------|
| 1 | `data-engineering-overview.md` | 데이터 엔지니어링 전체 그림 — Lakeflow 제품군 소개와 각 역할 |
| 2 | `choosing-ingestion-method.md` | 수집 방법 선택 가이드 — Auto Loader vs Lakeflow Connect vs 커스텀 |

## 참고 문서
- [Databricks: Data Engineering](https://docs.databricks.com/aws/en/data-engineering)
- [Azure Databricks: Data Engineering](https://learn.microsoft.com/en-us/azure/databricks/data-engineering/)
