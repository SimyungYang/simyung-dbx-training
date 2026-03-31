# Volumes — 파일 관리

## Volumes란?

> 💡 **Volume**은 Unity Catalog에서 **비테이블 파일**(이미지, PDF, CSV, 모델 파일, 로그 등)을 관리하는 저장 공간입니다. 테이블이 행/열 구조의 데이터를 관리한다면, Volume은 **파일 자체**를 관리합니다.

테이블과 동일한 `카탈로그.스키마` 체계에 속하므로, 동일한 권한 관리(GRANT/REVOKE)와 감사(Audit)가 적용됩니다.

---

## Volume의 위치

```
catalog
  └── schema
       ├── tables (테이블 — 행/열 데이터)
       ├── views (뷰)
       └── volumes (볼륨 — 파일 데이터)
            └── my_volume
                 ├── reports/report_2025.pdf
                 ├── images/product_001.jpg
                 └── raw_data/orders_20250315.csv
```

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
| **Managed Volume** | Databricks가 스토리지 위치를 자동 관리합니다 | Databricks 관리 경로 |
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
| **RAG 문서 저장** | PDF, DOCX, 웹 페이지 | `ai_parse_document`로 파싱 → 청킹 → Vector Search |
| **원본 데이터 수집** | CSV, JSON, Parquet | Auto Loader로 읽어서 Delta 테이블에 적재 |
| **ML 모델 아티팩트** | 모델 파일, 설정 파일 | MLflow 모델과 함께 관리 |
| **이미지/미디어** | 상품 이미지, 의료 영상 | Spark로 배치 처리, ML 모델 입력 |
| **보고서/내보내기** | PDF 리포트, Excel 파일 | 대시보드 결과를 파일로 내보내기 |

---

## Managed vs External Volume 심화

두 Volume 유형의 선택은 데이터 관리 전략에 중요한 영향을 미칩니다.

### 상세 비교

| 비교 항목 | Managed Volume | External Volume |
|-----------|---------------|-----------------|
| **스토리지 위치** | UC Metastore의 기본 경로 하위에 자동 생성 | 사용자가 지정한 클라우드 스토리지 경로 |
| **스토리지 비용** | Databricks 관리 버킷에 포함 | 고객 클라우드 스토리지 비용 별도 발생 |
| **데이터 삭제 시** | Volume 삭제 시 데이터도 함께 삭제 | Volume 삭제해도 원본 데이터 유지 |
| **외부 접근** | Databricks를 통해서만 접근 | 클라우드 네이티브 도구(AWS CLI, azcopy)로도 접근 가능 |
| **기존 데이터 연결** | 불가 (새로 업로드 필요) | 기존 스토리지의 데이터를 그대로 연결 가능 |
| **권한 모델** | UC 권한만 적용 | UC 권한 + 클라우드 IAM 이중 체크 |
| **Storage Credential** | 불필요 | External Location + Storage Credential 필요 |
| **백업/복제** | Databricks가 관리하는 내구성 정책 | 고객이 클라우드 복제(Cross-Region 등) 직접 관리 |

### External Volume 설정 과정

```sql
-- 1. Storage Credential 생성 (관리자)
CREATE STORAGE CREDENTIAL my_s3_credential
WITH (
    AWS_IAM_ROLE.ROLE_ARN = 'arn:aws:iam::123456789012:role/databricks-external-volume-role'
);

-- 2. External Location 생성 (관리자)
CREATE EXTERNAL LOCATION my_external_location
URL 's3://my-company-data-lake/volumes/'
WITH (STORAGE CREDENTIAL my_s3_credential);

-- 3. External Volume 생성
CREATE EXTERNAL VOLUME production.ecommerce.external_data
LOCATION 's3://my-company-data-lake/volumes/ecommerce/'
COMMENT '기존 S3 데이터를 UC로 거버넌스';
```

### 선택 가이드

| 상황 | 권장 유형 | 이유 |
|------|----------|------|
| **새 프로젝트** | Managed | 설정이 간단하고 관리 부담이 없습니다 |
| **기존 S3/ADLS 데이터** | External | 데이터 이동 없이 거버넌스만 추가합니다 |
| **외부 시스템과 파일 교환** | External | 외부 도구에서도 직접 파일에 접근 가능합니다 |
| **ETL 랜딩 존** | External | Auto Loader 등으로 외부에서 적재한 파일을 처리합니다 |
| **노트북 작업 파일** | Managed | 임시 파일, 실험 데이터 관리에 적합합니다 |
| **규제 데이터** | External | 데이터 위치를 고객이 명시적으로 제어해야 할 때 |

---

## Volume 접근 패턴 심화

### SQL 접근 고급 패턴

```sql
-- 디렉토리 내 파일 수와 총 크기 확인
LIST '/Volumes/production/ecommerce/raw_files/orders/2025/';

-- 와일드카드로 여러 파일 읽기
SELECT * FROM read_files(
    '/Volumes/production/ecommerce/raw_files/orders/2025/03/',
    format => 'json',
    schemaHints => 'order_id BIGINT, amount DOUBLE, order_date DATE'
);

-- 특정 파일 패턴만 읽기
SELECT * FROM read_files(
    '/Volumes/production/ecommerce/raw_files/',
    format => 'csv',
    pathGlobFilter => '*.csv',
    recursiveFileLookup => true,
    header => true
);

-- 바이너리 파일 읽기 (이미지 등)
SELECT
    path,
    length(content) AS file_size_bytes
FROM read_files(
    '/Volumes/production/ecommerce/images/',
    format => 'binaryFile'
);
```

### Python 접근 고급 패턴

```python
import os

# 표준 Python 파일 I/O로 접근 (가장 직관적)
volume_path = "/Volumes/production/ecommerce/raw_files"

# 파일 목록 조회
for f in os.listdir(volume_path):
    filepath = os.path.join(volume_path, f)
    size = os.path.getsize(filepath)
    print(f"{f}: {size:,} bytes")

# 파일 읽기/쓰기
with open(f"{volume_path}/config.json", "r") as f:
    config = json.load(f)

with open(f"{volume_path}/output/result.json", "w") as f:
    json.dump(result_data, f, ensure_ascii=False)

# Pandas로 CSV 직접 읽기
import pandas as pd
df = pd.read_csv(f"{volume_path}/data.csv")

# PIL로 이미지 처리
from PIL import Image
img = Image.open(f"{volume_path}/images/product_001.jpg")
```

### Databricks CLI 접근

```bash
# 파일 업로드
databricks fs cp ./local_data/ /Volumes/production/ecommerce/raw_files/ --recursive

# 파일 다운로드
databricks fs cp /Volumes/production/ecommerce/reports/monthly.pdf ./local/

# 디렉토리 생성
databricks fs mkdirs /Volumes/production/ecommerce/raw_files/2025/04/

# 파일 삭제
databricks fs rm /Volumes/production/ecommerce/raw_files/temp_file.csv

# 파일 이동 (복사 + 삭제)
databricks fs cp /Volumes/production/ecommerce/raw_files/old/ \
                 /Volumes/production/ecommerce/archive/old/ --recursive
databricks fs rm /Volumes/production/ecommerce/raw_files/old/ --recursive
```

### REST API 접근

```python
import requests

# REST API로 파일 업로드
workspace_url = "https://<workspace-url>"
token = "<access-token>"

# 파일 업로드
with open("local_file.csv", "rb") as f:
    response = requests.put(
        f"{workspace_url}/api/2.0/fs/files/Volumes/production/ecommerce/raw_files/data.csv",
        headers={"Authorization": f"Bearer {token}"},
        data=f
    )

# 파일 다운로드
response = requests.get(
    f"{workspace_url}/api/2.0/fs/files/Volumes/production/ecommerce/raw_files/data.csv",
    headers={"Authorization": f"Bearer {token}"}
)
with open("downloaded.csv", "wb") as f:
    f.write(response.content)
```

---

## 대용량 파일 처리 전략

### 파일 크기별 최적 접근 방법

| 파일 크기 | 접근 방법 | 설명 |
|-----------|----------|------|
| **< 100MB** | Python open() / pandas | 단일 파일 직접 처리 |
| **100MB ~ 10GB** | Spark read_files() | 분산 처리로 빠르게 읽기 |
| **> 10GB** | Auto Loader + Volume | 스트리밍 방식으로 점진적 처리 |
| **수천 개 소규모 파일** | Auto Loader (파일 알림 모드) | 개별 파일 추적 + 배치 처리 |

### 대용량 업로드 패턴

```python
# 대용량 파일 업로드: 청크 단위 업로드
import os

def upload_large_file(local_path, volume_path, chunk_size=50*1024*1024):
    """50MB 청크 단위로 대용량 파일 업로드"""
    file_size = os.path.getsize(local_path)

    if file_size < chunk_size:
        # 소규모 파일: 직접 복사
        dbutils.fs.cp(f"file:{local_path}", volume_path)
    else:
        # 대용량 파일: dbutils.fs.cp 사용 (내부적으로 멀티파트 업로드)
        dbutils.fs.cp(f"file:{local_path}", volume_path)
        print(f"Uploaded {file_size:,} bytes to {volume_path}")

# 디렉토리 전체 업로드 (병렬)
dbutils.fs.cp("file:/tmp/large_dataset/",
              "/Volumes/production/ml/training_data/",
              recurse=True)
```

### 파일 정리 자동화

```python
from datetime import datetime, timedelta

def cleanup_old_files(volume_path, days_to_keep=90):
    """지정 기간보다 오래된 파일을 자동 삭제"""
    cutoff = datetime.now() - timedelta(days=days_to_keep)

    files = dbutils.fs.ls(volume_path)
    deleted_count = 0

    for f in files:
        # 파일 수정 시간 확인
        mod_time = datetime.fromtimestamp(f.modificationTime / 1000)
        if mod_time < cutoff:
            dbutils.fs.rm(f.path)
            deleted_count += 1

    print(f"Deleted {deleted_count} files older than {days_to_keep} days")

# 매일 실행 (Jobs에서 스케줄)
cleanup_old_files("/Volumes/production/ecommerce/temp_files/", days_to_keep=30)
```

---

## Volume과 Auto Loader 연동

Volume은 **Auto Loader의 소스 경로**로 직접 사용할 수 있어, 파일 수집 파이프라인의 랜딩 존 역할을 합니다.

### Auto Loader + Volume 패턴

```python
# Volume을 소스로 하는 Auto Loader 스트리밍
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", "/Volumes/production/ecommerce/_schemas/orders/")
    .option("header", "true")
    .load("/Volumes/production/ecommerce/raw_files/orders/")
)

# Delta 테이블로 적재
(df.writeStream
    .option("checkpointLocation", "/Volumes/production/ecommerce/_checkpoints/orders/")
    .trigger(availableNow=True)
    .toTable("production.ecommerce.bronze_orders")
)
```

### 파일 수집 → 처리 → 아카이브 파이프라인

```
External Volume (랜딩 존)        Managed Volume (아카이브)
/Volumes/prod/etl/landing/      /Volumes/prod/etl/archive/
  └── orders/                     └── orders/
       ├── 20250301.csv  ──Auto Loader──> bronze_orders (Delta)
       ├── 20250302.csv              │
       └── 20250303.csv              └──> 처리 완료 파일 이동
```

```python
# 처리 완료 파일을 아카이브 Volume으로 이동
import os
from datetime import datetime

landing_path = "/Volumes/production/etl/landing/orders/"
archive_path = "/Volumes/production/etl/archive/orders/"

for f in dbutils.fs.ls(landing_path):
    if f.name.endswith(".csv"):
        # 처리 완료 확인 후 이동
        archive_dest = f"{archive_path}{datetime.now().strftime('%Y/%m/')}{f.name}"
        dbutils.fs.mv(f.path, archive_dest)
```

---

## AI/ML에서의 Volume 활용

### 학습 데이터 관리

```python
# 이미지 분류 모델용 학습 데이터 구조
# /Volumes/production/ml/training_images/
#   ├── cats/
#   │    ├── cat_001.jpg
#   │    └── cat_002.jpg
#   └── dogs/
#        ├── dog_001.jpg
#        └── dog_002.jpg

# PyTorch DataLoader에서 Volume 경로 직접 사용
from torchvision import datasets, transforms

train_dataset = datasets.ImageFolder(
    root="/Volumes/production/ml/training_images/",
    transform=transforms.Compose([
        transforms.Resize((224, 224)),
        transforms.ToTensor(),
    ])
)
```

### 모델 아티팩트 관리

```python
# 대형 모델 파일을 Volume에 저장
model_path = "/Volumes/production/ml/model_artifacts/"

# Hugging Face 모델 다운로드 → Volume에 저장
from huggingface_hub import snapshot_download
snapshot_download(
    repo_id="meta-llama/Llama-3-8B",
    local_dir=f"{model_path}llama-3-8b/"
)

# MLflow에서 Volume의 모델 참조
import mlflow
mlflow.pyfunc.log_model(
    artifact_path="model",
    python_model=my_model,
    artifacts={"model_weights": f"{model_path}llama-3-8b/"}
)
```

### RAG용 문서 저장소

```python
# RAG 파이프라인의 문서 소스로 Volume 활용
docs_volume = "/Volumes/production/rag/source_documents/"

# 문서 업로드
dbutils.fs.cp("file:/tmp/company_manual.pdf", f"{docs_volume}manuals/company_manual.pdf")

# 문서 파싱 → 청킹 → Vector Search 인덱스
parsed_docs = spark.sql(f"""
    SELECT
        path,
        ai_parse_document(path, 'markdown') AS content
    FROM read_files('{docs_volume}manuals/', format => 'binaryFile')
""")
```

---

## 비용 및 제한사항

| 항목 | 설명 |
|------|------|
| **스토리지 비용 (Managed)** | Databricks 관리 스토리지 가격 적용 (클라우드 스토리지보다 약간 높음) |
| **스토리지 비용 (External)** | 고객 클라우드 스토리지 비용 (S3: ~$0.023/GB/월, ADLS: ~$0.018/GB/월) |
| **단일 파일 크기 제한** | UI 업로드: 5GB, API: 제한 없음 (멀티파트 업로드) |
| **파일 수 제한** | Volume당 파일 수에 하드 제한은 없지만, LIST 성능은 디렉토리당 10,000개 이하를 권장 |
| **동시 접근** | 여러 클러스터에서 동시에 같은 Volume에 접근 가능 (파일 레벨 잠금은 없음) |
| **트랜잭션** | Volume의 파일 조작은 ACID 트랜잭션을 지원하지 않습니다. Delta 테이블과 달리 원자적 쓰기가 보장되지 않습니다 |

> ⚠️ **Volume vs Delta Table**: 구조화된 데이터는 반드시 **Delta Table**로 관리하세요. Volume은 비정형 파일(이미지, PDF, 모델 파일 등)이나 외부 시스템과의 파일 교환용으로 사용합니다. CSV/JSON 파일도 Volume에 저장 후 Auto Loader로 Delta Table에 적재하는 것이 모범 사례입니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Volume** | Unity Catalog에서 파일을 관리하는 저장 공간입니다 |
| **Managed** | Databricks가 위치를 자동 관리합니다 (권장) |
| **External** | 고객 지정 경로(S3, ADLS)를 사용합니다 |
| **경로 형식** | `/Volumes/<catalog>/<schema>/<volume>/` |
| **권한** | READ VOLUME / WRITE VOLUME으로 접근을 제어합니다 |
| **Auto Loader 연동** | Volume을 랜딩 존으로 사용하여 파일 수집을 자동화합니다 |
| **AI/ML 활용** | 학습 데이터, 모델 아티팩트, RAG 문서를 Volume에서 관리합니다 |

---

## 참고 링크

- [Databricks: Volumes](https://docs.databricks.com/aws/en/connect/unity-catalog/volumes.html)
- [Databricks: Manage files in volumes](https://docs.databricks.com/aws/en/connect/unity-catalog/volumes-manage.html)
