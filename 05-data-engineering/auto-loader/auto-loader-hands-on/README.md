# Auto Loader 실습 (Hands-On)

## 실습 개요

이 실습 시리즈는 Databricks **Auto Loader (자동 로더)** 를 처음부터 프로덕션 수준까지 단계적으로 익히는 4개의 실습으로 구성되어 있습니다.

Auto Loader는 클라우드 스토리지(S3, ADLS, GCS, UC Volume)에 **새로 도착한 파일만 자동으로 감지하고 수집** 하는 Databricks의 증분 파일 수집 기능입니다. Spark Structured Streaming (구조적 스트리밍) 의 `cloudFiles` 소스 포맷으로 구현되어 있습니다.

---

## 실습 목차

| 실습 | 파일 | 핵심 주제 |
|------|------|----------|
| **실습 1** | [사전 준비와 CSV 수집](setup-and-csv.md) | 환경 구성, CSV 증분 수집, Checkpoint 이해 |
| **실습 2** | [JSON 수집과 스키마 진화](json-and-schema-evolution.md) | JSON 수집, 스키마 진화, 중첩 구조 처리 |
| **실습 3** | [SDP와 Auto Loader 통합](sdp-integration.md) | `read_files()`, Medallion 파이프라인, 데이터 품질 |
| **실습 4** | [데이터 검증과 트러블슈팅](validation-and-troubleshooting.md) | 품질 리포트, 체크포인트 리셋, 에러 해결 |

---

## 사전 지식

이 실습을 시작하기 전에 아래 개념을 먼저 확인하세요:

- [Auto Loader란?](../what-is-auto-loader.md) — Auto Loader의 개념, 파일 감지 모드, 스키마 추론/진화
- [Auto Loader 주요 옵션](../auto-loader-options.md) — `cloudFiles.*` 옵션 상세

---

## 실습 환경 요구 사항

| 항목 | 요구 사항 |
|------|----------|
| **Databricks Runtime** | DBR 12.2 LTS 이상 |
| **Unity Catalog** | 활성화된 UC Metastore |
| **필요 권한** | `CREATE CATALOG`, `CREATE SCHEMA`, `CREATE VOLUME`, `CREATE TABLE` |
| **클러스터** | Single Node (실습용으로 충분), Access Mode: Single User 또는 Shared |

---

## 빠른 시작

모든 실습에서 공통으로 사용하는 경로:

```python
# 공통 경로 설정 (모든 실습 노트북 상단에 붙여넣기)
CATALOG     = "training"
SCHEMA      = "auto_loader_lab"
VOLUME_ROOT = f"/Volumes/{CATALOG}/{SCHEMA}/raw_data"

SOURCE_CSV_ORDERS     = f"{VOLUME_ROOT}/csv/orders/"
SOURCE_JSON_CUSTOMERS = f"{VOLUME_ROOT}/json/customers/"

CHECKPOINT_CSV     = f"{VOLUME_ROOT}/../checkpoints/csv_orders/"
CHECKPOINT_JSON    = f"{VOLUME_ROOT}/../checkpoints/json_customers/"
SCHEMA_CSV         = f"{VOLUME_ROOT}/../schema/csv_orders/"
SCHEMA_JSON        = f"{VOLUME_ROOT}/../schema/json_customers/"
```

---

## 실습 완료 후 기대 역량

| 역량 | 설명 |
|------|------|
| Auto Loader 파이프라인 구축 | `cloudFiles` 포맷으로 CSV/JSON을 증분 수집하고 Delta 테이블에 저장할 수 있습니다 |
| 스키마 진화 처리 | `schemaEvolutionMode`와 `mergeSchema`로 소스 스키마 변경에 자동 대응할 수 있습니다 |
| SDP Medallion 파이프라인 | `read_files()` + `CONSTRAINT`로 Bronze-Silver-Gold 파이프라인을 선언적으로 구축할 수 있습니다 |
| 품질 검증 | `_rescued_data`, NULL 비율, 행 수 정합 등으로 수집 품질을 검증할 수 있습니다 |
| 트러블슈팅 | 체크포인트 리셋, 스키마 충돌, 에러 메시지를 진단하고 해결할 수 있습니다 |
