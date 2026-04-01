# Managed vs External Volume 심화

## Managed vs External Volume 심화

두 Volume 유형의 선택은 데이터 관리 전략에 중요한 영향을 미칩니다.

### 상세 비교

| 비교 항목 | Managed Volume | External Volume |
|-----------|---------------|-----------------|
| **스토리지 위치**| UC Metastore의 기본 경로 하위에 자동 생성 | 사용자가 지정한 클라우드 스토리지 경로 |
| ** 스토리지 비용**| Databricks 관리 버킷에 포함 | 고객 클라우드 스토리지 비용 별도 발생 |
| ** 데이터 삭제 시**| Volume 삭제 시 데이터도 함께 삭제 | Volume 삭제해도 원본 데이터 유지 |
| ** 외부 접근**| Databricks를 통해서만 접근 | 클라우드 네이티브 도구(AWS CLI, azcopy)로도 접근 가능 |
| ** 기존 데이터 연결**| 불가 (새로 업로드 필요) | 기존 스토리지의 데이터를 그대로 연결 가능 |
| ** 권한 모델**| UC 권한만 적용 | UC 권한 + 클라우드 IAM 이중 체크 |
| **Storage Credential**| 불필요 | External Location + Storage Credential 필요 |
| ** 백업/복제**| Databricks가 관리하는 내구성 정책 | 고객이 클라우드 복제(Cross-Region 등) 직접 관리 |

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
| ** 새 프로젝트**| Managed | 설정이 간단하고 관리 부담이 없습니다 |
| ** 기존 S3/ADLS 데이터**| External | 데이터 이동 없이 거버넌스만 추가합니다 |
| ** 외부 시스템과 파일 교환**| External | 외부 도구에서도 직접 파일에 접근 가능합니다 |
| **ETL 랜딩 존**| External | Auto Loader 등으로 외부에서 적재한 파일을 처리합니다 |
| ** 노트북 작업 파일**| Managed | 임시 파일, 실험 데이터 관리에 적합합니다 |
| ** 규제 데이터** | External | 데이터 위치를 고객이 명시적으로 제어해야 할 때 |

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
