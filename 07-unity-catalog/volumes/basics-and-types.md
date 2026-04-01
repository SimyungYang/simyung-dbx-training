# 기본 개념과 Volume 유형

## Volumes란?

> 💡 **Volume** 은 Unity Catalog에서 ** 비테이블 파일**(이미지, PDF, CSV, 모델 파일, 로그 등)을 관리하는 저장 공간입니다. 테이블이 행/열 구조의 데이터를 관리한다면, Volume은 ** 파일 자체** 를 관리합니다.

테이블과 동일한 `카탈로그.스키마` 체계에 속하므로, 동일한 권한 관리(GRANT/REVOKE)와 감사(Audit)가 적용됩니다.

---

## Volume의 위치

| 수준 | 오브젝트 | 설명 |
|------|---------|------|
| catalog > schema | tables | 행/열 데이터 |
| | views | 뷰 |
| | volumes | 파일 데이터 |
|   └ my_volume | `reports/report_2025.pdf` | PDF 파일 |
| | `images/product_001.jpg` | 이미지 파일 |
| | `raw_data/orders_20250315.csv` | CSV 원시 데이터 |

### 파일 경로 형식

```
/Volumes/<catalog>/<schema>/<volume>/<path>

예시:
/Volumes/production/ecommerce/raw_files/orders/2025/03/data.csv
```

---

## Volume 유형

| 유형 | 설명 | 데이터 위치 |
|------|------|-----------|
| **Managed Volume**| Databricks가 스토리지 위치를 자동 관리합니다 | Databricks 관리 경로 |
| **External Volume** | 사용자가 지정한 외부 스토리지 경로를 사용합니다 | 고객 지정 경로 (S3, ADLS) |

```sql
-- Managed Volume 생성 (권장)
CREATE VOLUME production.ecommerce.raw_files
COMMENT '원본 파일 저장소';

-- External Volume 생성
CREATE EXTERNAL VOLUME production.ecommerce.external_data
LOCATION 's3://my-bucket/external-data/'
COMMENT '외부 스토리지 연결';
```

---

## 파일 조작

### SQL로 파일 다루기

```sql
-- 파일 목록 조회
LIST '/Volumes/production/ecommerce/raw_files/';

-- 파일을 테이블로 읽기
SELECT * FROM read_files(
    '/Volumes/production/ecommerce/raw_files/orders/',
    format => 'csv',
    header => true
);

-- 파일에서 텍스트 추출 (AI)
SELECT ai_parse_document(
    '/Volumes/production/ecommerce/raw_files/contracts/agreement.pdf',
    'markdown'
) AS content;
```

### Python으로 파일 다루기

```python
# 파일 업로드
dbutils.fs.cp(
    "file:/tmp/report.pdf",
    "/Volumes/production/ecommerce/reports/report_2025.pdf"
)

# 파일 목록 조회
files = dbutils.fs.ls("/Volumes/production/ecommerce/raw_files/")
for f in files:
    print(f"{f.name}  {f.size} bytes")

# 파일 다운로드
dbutils.fs.cp(
    "/Volumes/production/ecommerce/reports/report.pdf",
    "file:/tmp/downloaded_report.pdf"
)

# pandas로 CSV 읽기
import pandas as pd
df = pd.read_csv("/Volumes/production/ecommerce/raw_files/data.csv")
```

### REST API로 파일 업로드

```bash
# CLI로 파일 업로드
databricks fs cp ./local_file.csv /Volumes/production/ecommerce/raw_files/

# Volume의 파일 목록
databricks fs ls /Volumes/production/ecommerce/raw_files/
```

---

## 권한 관리

```sql
-- Volume 읽기 권한 부여
GRANT READ VOLUME ON VOLUME production.ecommerce.raw_files TO `analysts`;

-- Volume 쓰기 권한 부여
GRANT WRITE VOLUME ON VOLUME production.ecommerce.raw_files TO `data_engineers`;

-- 권한 확인
SHOW GRANTS ON VOLUME production.ecommerce.raw_files;
```

---

## 활용 시나리오

| 시나리오 | 저장 파일 | 처리 방법 |
|----------|----------|----------|
| **RAG 문서 저장**| PDF, DOCX, 웹 페이지 | `ai_parse_document`로 파싱 → 청킹 → Vector Search |
| ** 원본 데이터 수집**| CSV, JSON, Parquet | Auto Loader로 읽어서 Delta 테이블에 적재 |
| **ML 모델 아티팩트**| 모델 파일, 설정 파일 | MLflow 모델과 함께 관리 |
| ** 이미지/미디어**| 상품 이미지, 의료 영상 | Spark로 배치 처리, ML 모델 입력 |
| ** 보고서/내보내기** | PDF 리포트, Excel 파일 | 대시보드 결과를 파일로 내보내기 |

---
