# 대용량 처리와 AI/ML 활용

## 대용량 파일 처리 전략

### 파일 크기별 최적 접근 방법

| 파일 크기 | 접근 방법 | 설명 |
|-----------|----------|------|
| **< 100MB** | Python open() / pandas | 단일 파일 직접 처리 |
| **100MB ~ 10GB** | Spark read_files() | 분산 처리로 빠르게 읽기 |
| **> 10GB** | Auto Loader + Volume | 스트리밍 방식으로 점진적 처리 |
| ** 수천 개 소규모 파일** | Auto Loader (파일 알림 모드) | 개별 파일 추적 + 배치 처리 |

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

Volume은 **Auto Loader의 소스 경로** 로 직접 사용할 수 있어, 파일 수집 파이프라인의 랜딩 존 역할을 합니다.

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

| 볼륨 유형 | 경로 | 내용 | 처리 |
|----------|------|------|------|
| **External Volume** (랜딩 존) | `/Volumes/prod/etl/landing/orders/` | 20250301.csv, 20250302.csv, 20250303.csv | Auto Loader → bronze_orders (Delta) |
| **Managed Volume** (아카이브) | `/Volumes/prod/etl/archive/orders/` | 처리 완료 파일 | 처리 완료 후 이동 |

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
| ** 스토리지 비용 (Managed)** | Databricks 관리 스토리지 가격 적용 (클라우드 스토리지보다 약간 높음) |
| ** 스토리지 비용 (External)** | 고객 클라우드 스토리지 비용 (S3: ~$0.023/GB/월, ADLS: ~$0.018/GB/월) |
| ** 단일 파일 크기 제한** | UI 업로드: 5GB, API: 제한 없음 (멀티파트 업로드) |
| ** 파일 수 제한** | Volume당 파일 수에 하드 제한은 없지만, LIST 성능은 디렉토리당 10,000개 이하를 권장 |
| ** 동시 접근** | 여러 클러스터에서 동시에 같은 Volume에 접근 가능 (파일 레벨 잠금은 없음) |
| ** 트랜잭션** | Volume의 파일 조작은 ACID 트랜잭션을 지원하지 않습니다. Delta 테이블과 달리 원자적 쓰기가 보장되지 않습니다 |

> ⚠️ **Volume vs Delta Table**: 구조화된 데이터는 반드시 **Delta Table** 로 관리하세요. Volume은 비정형 파일(이미지, PDF, 모델 파일 등)이나 외부 시스템과의 파일 교환용으로 사용합니다. CSV/JSON 파일도 Volume에 저장 후 Auto Loader로 Delta Table에 적재하는 것이 모범 사례입니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Volume** | Unity Catalog에서 파일을 관리하는 저장 공간입니다 |
| **Managed** | Databricks가 위치를 자동 관리합니다 (권장) |
| **External** | 고객 지정 경로(S3, ADLS)를 사용합니다 |
| ** 경로 형식** | `/Volumes/<catalog>/<schema>/<volume>/` |
| ** 권한** | READ VOLUME / WRITE VOLUME으로 접근을 제어합니다 |
| **Auto Loader 연동** | Volume을 랜딩 존으로 사용하여 파일 수집을 자동화합니다 |
| **AI/ML 활용** | 학습 데이터, 모델 아티팩트, RAG 문서를 Volume에서 관리합니다 |

---

## 참고 링크

- [Databricks: Volumes](https://docs.databricks.com/aws/en/connect/unity-catalog/volumes.html)
- [Databricks: Manage files in volumes](https://docs.databricks.com/aws/en/connect/unity-catalog/volumes-manage.html)
