# 05-1. Auto Loader

> 클라우드 스토리지(S3, ADLS, GCS)에 도착하는 새 파일을 자동으로 감지하고 수집하는 방법을 학습합니다.

## 학습 목표
- Auto Loader의 동작 원리(Directory Listing vs File Notification) 이해
- 다양한 파일 포맷(CSV, JSON, Parquet, Avro) 수집 방법
- 스키마 추론(Schema Inference)과 스키마 진화(Schema Evolution) 처리
- Rescue Data Column을 활용한 불량 데이터 처리

## 문서 목록

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [Auto Loader란?](what-is-auto-loader.md) | 개념, 동작 방식, 파일 감지 모드, 스키마 관리를 설명합니다 |
| 2 | [주요 옵션](auto-loader-options.md) | 파일 포맷별 설정, 에러 처리, 메타데이터 컬럼을 다룹니다 |
| 3 | [Auto Loader 실습](auto-loader-hands-on.md) | SDP 기반 수집 파이프라인을 구축합니다 |

## 참고 문서
- [Databricks: Auto Loader](https://docs.databricks.com/aws/en/ingestion/auto-loader/)
