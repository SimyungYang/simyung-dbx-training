# Volumes — 파일 관리

## Volumes란?

> 💡 **Volume**은 Unity Catalog에서 **비테이블 파일**(이미지, PDF, CSV, 모델 파일 등)을 관리하는 저장 공간입니다. 테이블과 동일한 카탈로그-스키마 체계에서 권한 관리가 가능합니다.

```sql
-- Volume 생성
CREATE VOLUME production.ecommerce.raw_files;

-- 파일 경로 형식
-- /Volumes/<catalog>/<schema>/<volume>/<path>
```

---

## Volume 유형

| 유형 | 설명 |
|------|------|
| **Managed Volume** | Databricks가 스토리지 위치를 자동 관리합니다 |
| **External Volume** | 사용자가 지정한 외부 스토리지 경로를 사용합니다 |

---

## 파일 조작

```sql
-- Volume에 있는 파일 목록 조회
LIST '/Volumes/production/ecommerce/raw_files/';

-- Volume의 파일을 테이블로 읽기
SELECT * FROM read_files('/Volumes/production/ecommerce/raw_files/orders/', format => 'csv');
```

```python
# Python으로 파일 업로드
dbutils.fs.cp("file:/tmp/report.pdf", "/Volumes/production/ecommerce/reports/report.pdf")
```

---

## 참고 링크

- [Databricks: Volumes](https://docs.databricks.com/aws/en/connect/unity-catalog/volumes.html)
