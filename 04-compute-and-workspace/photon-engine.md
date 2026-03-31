# Photon 엔진 상세

## Photon이란?

**Photon**은 Databricks가 개발한 **C++ 기반 벡터화(Vectorized) 실행 엔진**입니다. 기존 Apache Spark의 JVM(Java Virtual Machine) 기반 실행 엔진을 대체하여, SQL 및 DataFrame 워크로드의 성능을 **2배에서 최대 8배까지 향상**시킵니다.

> 💡 **핵심 아이디어**: Spark의 기존 엔진은 Java/Scala로 구현되어 JVM의 오버헤드(GC, 메모리 관리)가 있었습니다. Photon은 C++로 작성되어 JVM 없이 **네이티브 코드 수준**의 성능을 제공합니다.

---

## 기존 Spark SQL 엔진과의 차이

### 아키텍처 비교

| 비교 항목 | Spark SQL 엔진 (기존) | Photon 엔진 |
|-----------|---------------------|-------------|
| **구현 언어** | Java/Scala (JVM 위에서 실행) | C++ (네이티브 실행) |
| **데이터 처리 방식** | 행 단위(Row-at-a-time) + Whole-Stage CodeGen | 벡터화(Vectorized, 배치 단위) |
| **메모리 관리** | JVM Garbage Collector | 직접 메모리 관리 (off-heap) |
| **CPU 활용** | JIT 컴파일 | SIMD 명령어 활용 |
| **호환성** | 모든 Spark 연산 지원 | 지원 범위가 점진적으로 확장 중 |

### 성능 이점

| 성능 영역 | 설명 |
|----------|------|
| **SIMD(Single Instruction, Multiple Data)** | 하나의 CPU 명령으로 여러 데이터를 동시에 처리합니다. 벡터 연산이 수십 배 빨라집니다 |
| **메모리 관리** | JVM GC(Garbage Collection) 없이 직접 메모리를 관리하므로, GC 일시 정지가 발생하지 않습니다 |
| **캐시 효율** | CPU 캐시에 최적화된 데이터 레이아웃으로, 캐시 미스를 줄입니다 |
| **네이티브 실행** | C++ 네이티브 코드로 JVM 오버헤드(바이트코드 해석, JIT 워밍업)가 없습니다 |

> 💡 **SIMD란?** CPU의 벡터 레지스터를 활용하여, 하나의 명령어로 여러 데이터 항목에 동시에 같은 연산을 수행하는 기술입니다. 예를 들어 4개의 정수 덧셈을 한 번에 처리할 수 있습니다.

---

## 성능 벤치마크

Databricks의 공식 벤치마크에 따르면, Photon은 다양한 워크로드에서 다음과 같은 성능 향상을 보여줍니다.

| 워크로드 유형 | 성능 향상 (배수) | 설명 |
|-------------|----------------|------|
| **TPC-DS** | 약 2~3배 | 표준 데이터 웨어하우스 벤치마크 |
| **집계 (GROUP BY)** | 약 3~5배 | SUM, COUNT, AVG 등의 집계 연산 |
| **조인 (JOIN)** | 약 2~4배 | 해시 조인, 소트 머지 조인 |
| **필터링 (WHERE)** | 약 2~3배 | 조건부 필터링 |
| **문자열 처리** | 약 3~8배 | LIKE, SUBSTR, CONCAT 등 |
| **파일 쓰기** | 약 2~3배 | Parquet/Delta 파일 쓰기 |

---

## 활성화 방법

### All-Purpose Cluster에서 활성화

클러스터 생성 시 **Photon 가속**을 체크하거나, Runtime 이름에 `photon`이 포함된 버전을 선택합니다.

```
Runtime: 15.4.x-photon-scala2.12
```

### Job Cluster에서 활성화

```yaml
# Asset Bundles YAML 설정
new_cluster:
  spark_version: "15.4.x-photon-scala2.12"
  runtime_engine: "PHOTON"
  num_workers: 4
  node_type_id: "m5.2xlarge"
```

### SQL Warehouse

SQL Warehouse는 **기본적으로 Photon이 활성화**되어 있습니다. 별도 설정이 필요 없습니다.

### SDP 파이프라인에서 활성화

```json
{
    "name": "my-pipeline",
    "photon": true,
    "clusters": [
        {
            "label": "default",
            "num_workers": 4
        }
    ]
}
```

---

## 지원되는 연산

Photon은 대부분의 SQL 및 DataFrame 연산을 지원하며, 지원 범위가 지속적으로 확장되고 있습니다.

### 지원되는 주요 연산

| 카테고리 | 지원 연산 |
|---------|----------|
| **스캔** | Delta Lake, Parquet 파일 읽기 |
| **필터** | WHERE, HAVING 조건 필터링 |
| **집계** | GROUP BY, SUM, COUNT, AVG, MIN, MAX, DISTINCT |
| **조인** | INNER, LEFT, RIGHT, FULL OUTER, CROSS, SEMI, ANTI JOIN |
| **정렬** | ORDER BY, SORT |
| **윈도우 함수** | ROW_NUMBER, RANK, LAG, LEAD, SUM OVER 등 |
| **문자열** | LIKE, SUBSTR, CONCAT, UPPER, LOWER, TRIM 등 |
| **날짜/시간** | DATE_TRUNC, DATE_ADD, DATEDIFF 등 |
| **쓰기** | Delta Lake INSERT, MERGE, UPDATE, DELETE |
| **파일 쓰기** | Parquet, Delta 형식 쓰기 |

### Photon으로 실행되지 않는 연산

Photon이 지원하지 않는 연산은 자동으로 기존 Spark 엔진으로 **폴백(fallback)** 됩니다. 사용자는 이를 인지할 필요 없이, 쿼리가 정상적으로 실행됩니다.

| 카테고리 | 비지원 연산 |
|---------|-----------|
| **UDF** | Python UDF, Scala UDF (Pandas UDF는 일부 지원) |
| **스트리밍** | 일부 스트리밍 연산 |
| **비 Delta 포맷** | CSV, JSON, ORC 읽기 (Delta/Parquet만 가속) |

> 💡 **Photon fallback**: 쿼리의 일부만 Photon으로 실행되고 나머지는 Spark 엔진으로 실행되는 혼합 실행도 가능합니다. Spark UI의 **Photon 실행 비율**에서 확인할 수 있습니다.

---

## 비용 고려사항

### Photon DBU 단가

Photon을 사용하면 DBU(Databricks Unit) 소비 단가가 달라집니다.

| 컴퓨트 유형 | 일반 DBU 단가 | Photon DBU 단가 | 비고 |
|------------|-------------|----------------|------|
| **All-Purpose** | 기본 | 약 1.5~2배 | 개발/탐색용 |
| **Jobs** | 기본 | 약 1.5~2배 | 배치 Job용 |
| **SQL Warehouse** | Photon 기본 포함 | 기본 요금에 포함 | 추가 비용 없음 |

### 비용 대비 효과 판단

```
성능 향상 배수 > DBU 단가 증가 배수 → Photon이 비용 효율적
```

| 시나리오 | 성능 향상 | DBU 단가 증가 | 총 비용 효과 |
|---------|---------|-------------|-----------|
| 집계 중심 SQL | 3~5배 빠름 | 1.5배 | 비용 절감 (실행 시간이 크게 줄어 총 DBU 감소) |
| 단순 파일 복사 | 1.2배 빠름 | 1.5배 | 비용 증가 (Photon 효과가 적음) |
| 문자열 처리 | 3~8배 빠름 | 1.5배 | 비용 절감 |

### Photon 사용 권장 워크로드

| 권장 | 비권장 |
|------|-------|
| SQL 집계, 조인이 많은 쿼리 | Python UDF가 많은 워크로드 |
| Delta Lake 읽기/쓰기 | CSV/JSON 직접 처리 |
| DBSQL (SQL Warehouse) 워크로드 | 스트리밍 위주의 가벼운 처리 |
| ETL 파이프라인 | ML 학습 (GPU가 더 효과적) |

---

## Photon 성능 확인

### Spark UI에서 Photon 비율 확인

클러스터의 Spark UI → SQL/DataFrame 탭에서 각 쿼리의 **Photon 실행 비율**을 확인할 수 있습니다. 비율이 높을수록 Photon의 이점을 더 많이 활용하고 있다는 의미입니다.

### 쿼리 성능 비교

```sql
-- Photon 활성화 상태에서 실행 시간 측정
SELECT
    region,
    product_category,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value
FROM catalog.schema.large_orders_table  -- 수억 행 테이블
GROUP BY region, product_category
ORDER BY total_revenue DESC;
```

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Photon** | C++ 기반 벡터화 실행 엔진으로, SQL/DataFrame 성능을 2~8배 향상시킵니다 |
| **벡터화 처리** | 행 단위가 아닌 배치(열) 단위로 데이터를 처리하여 SIMD를 활용합니다 |
| **JVM 오버헤드 제거** | GC 일시 정지가 없고, 메모리를 직접 관리합니다 |
| **자동 폴백** | 미지원 연산은 자동으로 기존 Spark 엔진으로 실행됩니다 |
| **비용 효과** | 집계/조인 중심 워크로드에서 총 비용이 감소합니다 |

---

## 참고 링크

- [Databricks: Photon runtime](https://docs.databricks.com/aws/en/compute/photon.html)
- [Databricks: Photon FAQ](https://docs.databricks.com/aws/en/compute/photon.html#photon-faq)
- [Databricks Blog: Photon Technical Deep Dive](https://www.databricks.com/blog/2022/01/28/photon-technical-deep-dive.html)
- [Azure Databricks: Photon](https://learn.microsoft.com/en-us/azure/databricks/compute/photon)
