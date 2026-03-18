# 네임스페이스 구조

## Unity Catalog 객체 유형

| 객체 | 설명 | 예시 |
|------|------|------|
| **Metastore** | Unity Catalog의 최상위 컨테이너. 리전별 하나 | 계정 수준 관리 |
| **Catalog** | 데이터 자산의 최상위 논리 그룹 | `prod`, `dev`, `sandbox` |
| **Schema** | Catalog 내의 논리적 그룹 | `ecommerce`, `analytics` |
| **Table** | 행/열 구조의 데이터 | `orders`, `customers` |
| **View** | SQL 쿼리의 별칭 | `v_active_customers` |
| **Volume** | 비테이블 파일 저장소 | 이미지, PDF, CSV 파일 |
| **Function** | SQL/Python 사용자 정의 함수 | `calculate_tax()` |
| **Model** | MLflow 등록 모델 | `fraud_detection_v2` |
| **Connection** | 외부 시스템 연결 정보 | MySQL, Salesforce 연결 |

```sql
-- 카탈로그 생성
CREATE CATALOG IF NOT EXISTS production;

-- 스키마 생성
CREATE SCHEMA IF NOT EXISTS production.ecommerce;

-- 테이블 생성
CREATE TABLE production.ecommerce.orders (...);

-- 볼륨 생성
CREATE VOLUME production.ecommerce.raw_files;
```

---

## 참고 링크

- [Databricks: Unity Catalog objects](https://docs.databricks.com/aws/en/data-governance/unity-catalog/index.html)
