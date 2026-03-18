# Auto Loader 주요 옵션

## 파일 포맷별 옵션

### JSON

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("multiLine", "true")               # 여러 줄에 걸친 JSON
    .option("cloudFiles.inferColumnTypes", "true")  # 타입 자동 추론
    .option("primitivesAsString", "false")      # 숫자를 숫자 타입으로 유지
    .load("s3://bucket/json-data/")
)
```

### CSV

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("header", "true")                   # 첫 줄을 헤더로 사용
    .option("delimiter", ",")                   # 구분자
    .option("encoding", "UTF-8")               # 인코딩
    .option("quote", '"')                       # 따옴표 문자
    .option("escape", "\\")                     # 이스케이프 문자
    .option("nullValue", "NULL")               # NULL 표현 값
    .load("s3://bucket/csv-data/")
)
```

## 에러 처리 옵션

| 옵션 | 설명 |
|------|------|
| `rescuedDataColumn` | 스키마에 맞지 않는 데이터를 별도 컬럼에 보존합니다 |
| `badRecordsPath` | 파싱 실패한 레코드를 별도 경로에 저장합니다 |
| `columnNameOfCorruptRecord` | 손상된 레코드를 담을 컬럼 이름을 지정합니다 |
| `ignoreCorruptFiles` | 손상된 파일을 건너뛰고 계속 처리합니다 |

## 메타데이터 컬럼

Auto Loader는 수집된 파일의 메타데이터를 자동으로 제공합니다.

```sql
SELECT
    *,
    _metadata.file_path,                    -- 소스 파일 경로
    _metadata.file_name,                    -- 파일 이름
    _metadata.file_size,                    -- 파일 크기
    _metadata.file_modification_time,       -- 파일 수정 시간
    _metadata.file_block_start,             -- 블록 시작 위치
    _metadata.file_block_length             -- 블록 길이
FROM STREAM read_files('s3://bucket/data/', format => 'json');
```

---

## 참고 링크

- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/auto-loader/options.html)
