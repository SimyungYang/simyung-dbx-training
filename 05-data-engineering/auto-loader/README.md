# 05-1. Auto Loader

> 클라우드 스토리지(S3, ADLS, GCS)에 도착하는 새 파일을 자동으로 감지하고 수집하는 방법을 학습합니다.

## 학습 목표
- Auto Loader의 동작 원리(Directory Listing vs File Notification) 이해
- 다양한 파일 포맷(CSV, JSON, Parquet, Avro) 수집 방법
- 스키마 추론(Schema Inference)과 스키마 진화(Schema Evolution) 처리
- Rescue Data Column을 활용한 불량 데이터 처리

## 문서 목록

| 순서 | 파일명 | 내용 |
|------|--------|------|
| 1 | `what-is-auto-loader.md` | Auto Loader란? — 개념, 동작 방식, 사용 시나리오 |
| 2 | `auto-loader-options.md` | 주요 옵션 — 파일 포맷별 설정, 스키마 힌트, 에러 처리 |
| 3 | `auto-loader-hands-on.md` | 실습 — S3/ADLS에서 JSON 파일 수집 파이프라인 구축 |

## 참고 문서
- [Databricks: Auto Loader](https://docs.databricks.com/aws/en/ingestion/auto-loader/)
