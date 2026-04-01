# 데이터 검증과 트러블슈팅

## 데이터 검증

각 실습 후 데이터가 올바르게 수집되었는지 검증하는 단계입니다.

### 검증 체크리스트

| 검증 항목 | 확인 방법 | 예상 결과 |
|-----------|-----------|-----------|
| **행 수 정합** | `COUNT(*)`로 원본 파일과 테이블 비교 | 파일의 행 수와 테이블 행 수 일치 |
| **중복 없음**| `GROUP BY`로 기본 키 중복 확인 | 중복 레코드 없음 |
| **NULL 처리**| 필수 컬럼 NULL 비율 확인 | 허용 범위 내 |
| **_rescued_data**| 파싱 실패 건 확인 | 0 또는 허용 범위 내 |
| **스키마 일치** | `DESCRIBE`로 컬럼 타입 확인 | 예상 스키마와 일치 |

```sql
-- 종합 데이터 품질 리포트
SELECT
    'bronze_orders' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT order_id) AS unique_keys,
    SUM(CASE WHEN _rescued_data IS NOT NULL THEN 1 ELSE 0 END) AS rescued_rows,
    SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_key_rows,
    MIN(_ingested_at) AS first_ingestion,
    MAX(_ingested_at) AS last_ingestion
FROM training.auto_loader_lab.bronze_orders

UNION ALL

SELECT
    'bronze_customers' AS table_name,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT customer_id) AS unique_keys,
    SUM(CASE WHEN _rescued_data IS NOT NULL THEN 1 ELSE 0 END) AS rescued_rows,
    SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_key_rows,
    MIN(_ingested_at) AS first_ingestion,
    MAX(_ingested_at) AS last_ingestion
FROM training.auto_loader_lab.bronze_customers;
```

---

## 트러블슈팅

### 체크포인트 리셋

체크포인트가 손상되었거나, 처음부터 다시 처리해야 할 때 사용합니다.

```python
# 체크포인트 삭제 (주의: 모든 파일을 처음부터 다시 처리합니다)
dbutils.fs.rm("/Volumes/training/auto_loader_lab/checkpoints/csv_orders/", recurse=True)

# 스키마 위치도 리셋 (필요한 경우)
dbutils.fs.rm("/Volumes/training/auto_loader_lab/schema/csv_orders/", recurse=True)

print("체크포인트 및 스키마 리셋 완료. 스트림을 다시 시작하면 모든 파일을 처리합니다.")
```

> ⚠️ **체크포인트 리셋 시 주의**: 체크포인트를 삭제하면 모든 파일을 처음부터 다시 처리합니다. 대상 테이블에 `MERGE` 대신 `APPEND`를 사용하고 있다면 데이터 중복이 발생합니다. 리셋 전에 대상 테이블도 함께 초기화하세요.

```python
# 테이블과 체크포인트를 함께 초기화하는 안전한 방법
spark.sql("DROP TABLE IF EXISTS training.auto_loader_lab.bronze_orders")
dbutils.fs.rm("/Volumes/training/auto_loader_lab/checkpoints/csv_orders/", recurse=True)
dbutils.fs.rm("/Volumes/training/auto_loader_lab/schema/csv_orders/", recurse=True)
print("테이블, 체크포인트, 스키마 모두 초기화 완료.")
```

### 스키마 충돌 해결

스키마 진화 과정에서 충돌이 발생할 수 있습니다.

| 문제 | 원인 | 해결 방법 |
|------|------|-----------|
| **UnknownFieldException**| `schemaEvolutionMode=failOnNewColumns`에서 새 컬럼 감지 | 스키마를 수동으로 업데이트하거나, `addNewColumns` 모드로 변경 |
| **타입 불일치**| 같은 컬럼이 다른 타입으로 도착 (예: STRING → INT) | `schemaHints`로 타입을 명시하거나, `rescuedDataColumn`으로 처리 |
| **스키마 덮어쓰기 필요**| 스키마 위치의 기존 스키마가 잘못된 경우 | `cloudFiles.allowOverwrites=true` 설정 후 스키마 위치 리셋 |

```python
# 스키마 충돌 해결 예제: 타입 힌트로 강제 지정
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", schema_path)
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    # 문제가 되는 컬럼의 타입을 명시적으로 지정
    .option("cloudFiles.schemaHints",
            "order_id BIGINT, amount DOUBLE, zip_code STRING")
    .option("rescuedDataColumn", "_rescued_data")
    .load(source_path)
)
```

### 일반적인 에러와 해결 방법

| 에러 메시지 | 원인 | 해결 방법 |
|-------------|------|-----------|
| `PERMISSION_DENIED` | 볼륨/S3 경로 접근 권한 부족 | Unity Catalog 권한 또는 IAM 역할 확인 |
| `AnalysisException: path does not exist` | 소스 경로가 존재하지 않음 | 경로 오타 확인, 볼륨이 생성되었는지 확인 |
| `StreamingQueryException: checkpoint` | 체크포인트 손상 또는 호환 불가 | 체크포인트 리셋 후 재시작 |
| `DELTA_SCHEMA_CHANGE_SINCE_ANALYSIS` | 스트리밍 중 스키마 변경 | `mergeSchema=true` 옵션 추가 |
| `MALFORMED_RECORD_IN_PARSING` | 파싱할 수 없는 레코드 존재 | `rescuedDataColumn` 또는 `badRecordsPath` 설정 |

---

## 실습 정리 (리소스 삭제)

실습이 끝나면 생성한 리소스를 정리합니다.

```sql
-- 테이블 삭제
DROP TABLE IF EXISTS training.auto_loader_lab.bronze_orders;
DROP TABLE IF EXISTS training.auto_loader_lab.bronze_customers;
DROP TABLE IF EXISTS training.auto_loader_lab.silver_orders;
DROP TABLE IF EXISTS training.auto_loader_lab.silver_customers;
DROP VIEW IF EXISTS training.auto_loader_lab.gold_daily_sales;
DROP VIEW IF EXISTS training.auto_loader_lab.gold_customer_by_city;

-- 볼륨 삭제 (데이터 포함)
DROP VOLUME IF EXISTS training.auto_loader_lab.raw_data;

-- 스키마 삭제
DROP SCHEMA IF EXISTS training.auto_loader_lab CASCADE;
```

```python
# 체크포인트 정리
dbutils.fs.rm("/Volumes/training/auto_loader_lab/", recurse=True)
print("모든 실습 리소스 정리 완료!")
```

---

## 정리

| 실습 | 학습 포인트 |
|------|-------------|
| **실습 1: CSV 수집**| Auto Loader 기본 사용법, 체크포인트 기반 증분 처리 |
| **실습 2: JSON + 스키마 진화**| `schemaEvolutionMode=addNewColumns`로 자동 스키마 확장 |
| **실습 3: SDP 통합**| `read_files()` 함수로 Medallion 파이프라인 구축 |
| **데이터 검증**| `_rescued_data`, 행 수 정합, NULL 비율 확인 |
| **트러블슈팅** | 체크포인트 리셋, 스키마 충돌 해결, 일반 에러 대응 |

---

## 참고 링크

- [Databricks: Auto Loader tutorial](https://docs.databricks.com/aws/en/ingestion/auto-loader/tutorial.html)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html)
- [Databricks: Auto Loader schema inference](https://docs.databricks.com/aws/en/ingestion/auto-loader/schema.html)
- [Databricks: read_files in SDP](https://docs.databricks.com/aws/en/ingestion/auto-loader/unity-catalog.html)
- [Databricks: Streaming tables](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-streaming-table.html)
