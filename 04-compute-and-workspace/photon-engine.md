# Photon 엔진 상세

## Photon이란?

**Photon** 은 Databricks가 개발한 **C++ 기반 벡터화(Vectorized) 실행 엔진** 입니다. 기존 Apache Spark의 JVM(Java Virtual Machine) 기반 실행 엔진을 대체하여, SQL 및 DataFrame 워크로드의 성능을 **2배에서 최대 8배까지 향상** 시킵니다.

> 💡 ** 핵심 아이디어**: Spark의 기존 엔진은 Java/Scala로 구현되어 JVM의 오버헤드(GC, 메모리 관리)가 있었습니다. Photon은 C++로 작성되어 JVM 없이 ** 네이티브 코드 수준** 의 성능을 제공합니다.

---

## 기존 Spark SQL 엔진과의 차이

### 아키텍처 비교

| 비교 항목 | Spark SQL 엔진 (기존) | Photon 엔진 |
|-----------|---------------------|-------------|
| ** 구현 언어** | Java/Scala (JVM 위에서 실행) | C++ (네이티브 실행) |
| ** 데이터 처리 방식** | 행 단위(Row-at-a-time) + Whole-Stage CodeGen | 벡터화(Vectorized, 배치 단위) |
| ** 메모리 관리** | JVM Garbage Collector | 직접 메모리 관리 (off-heap) |
| **CPU 활용**| JIT 컴파일 | SIMD 명령어 활용 |
| ** 호환성**| 모든 Spark 연산 지원 | 지원 범위가 점진적으로 확장 중 |

### 성능 이점

| 성능 영역 | 설명 |
|----------|------|
| **SIMD(Single Instruction, Multiple Data)**| 하나의 CPU 명령으로 여러 데이터를 동시에 처리합니다. 벡터 연산이 수십 배 빨라집니다 |
| ** 메모리 관리**| JVM GC(Garbage Collection) 없이 직접 메모리를 관리하므로, GC 일시 정지가 발생하지 않습니다 |
| ** 캐시 효율**| CPU 캐시에 최적화된 데이터 레이아웃으로, 캐시 미스를 줄입니다 |
| ** 네이티브 실행**| C++ 네이티브 코드로 JVM 오버헤드(바이트코드 해석, JIT 워밍업)가 없습니다 |

> 💡 **SIMD란?**CPU의 벡터 레지스터를 활용하여, 하나의 명령어로 여러 데이터 항목에 동시에 같은 연산을 수행하는 기술입니다. 예를 들어 4개의 정수 덧셈을 한 번에 처리할 수 있습니다.

---

## 성능 벤치마크

Databricks의 공식 벤치마크에 따르면, Photon은 다양한 워크로드에서 다음과 같은 성능 향상을 보여줍니다.

| 워크로드 유형 | 성능 향상 (배수) | 설명 |
|-------------|----------------|------|
| **TPC-DS**| 약 2~3배 | 표준 데이터 웨어하우스 벤치마크 |
| ** 집계 (GROUP BY)**| 약 3~5배 | SUM, COUNT, AVG 등의 집계 연산 |
| ** 조인 (JOIN)**| 약 2~4배 | 해시 조인, 소트 머지 조인 |
| ** 필터링 (WHERE)**| 약 2~3배 | 조건부 필터링 |
| ** 문자열 처리**| 약 3~8배 | LIKE, SUBSTR, CONCAT 등 |
| ** 파일 쓰기**| 약 2~3배 | Parquet/Delta 파일 쓰기 |

---

## 활성화 방법

### All-Purpose Cluster에서 활성화

클러스터 생성 시 **Photon 가속** 을 체크하거나, Runtime 이름에 `photon`이 포함된 버전을 선택합니다.

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

SQL Warehouse는 ** 기본적으로 Photon이 활성화** 되어 있습니다. 별도 설정이 필요 없습니다.

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
| ** 스캔**| Delta Lake, Parquet 파일 읽기 |
| ** 필터**| WHERE, HAVING 조건 필터링 |
| ** 집계**| GROUP BY, SUM, COUNT, AVG, MIN, MAX, DISTINCT |
| ** 조인**| INNER, LEFT, RIGHT, FULL OUTER, CROSS, SEMI, ANTI JOIN |
| ** 정렬**| ORDER BY, SORT |
| ** 윈도우 함수**| ROW_NUMBER, RANK, LAG, LEAD, SUM OVER 등 |
| ** 문자열**| LIKE, SUBSTR, CONCAT, UPPER, LOWER, TRIM 등 |
| ** 날짜/시간**| DATE_TRUNC, DATE_ADD, DATEDIFF 등 |
| ** 쓰기**| Delta Lake INSERT, MERGE, UPDATE, DELETE |
| ** 파일 쓰기**| Parquet, Delta 형식 쓰기 |

### Photon으로 실행되지 않는 연산

Photon이 지원하지 않는 연산은 자동으로 기존 Spark 엔진으로 ** 폴백(fallback)** 됩니다. 사용자는 이를 인지할 필요 없이, 쿼리가 정상적으로 실행됩니다.

| 카테고리 | 비지원 연산 |
|---------|-----------|
| **UDF**| Python UDF, Scala UDF (Pandas UDF는 일부 지원) |
| ** 스트리밍**| 일부 스트리밍 연산 |
| ** 비 Delta 포맷**| CSV, JSON, ORC 읽기 (Delta/Parquet만 가속) |

> 💡 **Photon fallback**: 쿼리의 일부만 Photon으로 실행되고 나머지는 Spark 엔진으로 실행되는 혼합 실행도 가능합니다. Spark UI의 **Photon 실행 비율** 에서 확인할 수 있습니다.

---

## 비용 고려사항

### Photon DBU 단가

Photon을 사용하면 DBU(Databricks Unit) 소비 단가가 달라집니다.

| 컴퓨트 유형 | 일반 DBU 단가 | Photon DBU 단가 | 비고 |
|------------|-------------|----------------|------|
| **All-Purpose**| 기본 | 약 1.5~2배 | 개발/탐색용 |
| **Jobs**| 기본 | 약 1.5~2배 | 배치 Job용 |
| **SQL Warehouse**| Photon 기본 포함 | 기본 요금에 포함 | 추가 비용 없음 |

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

클러스터의 Spark UI → SQL/DataFrame 탭에서 각 쿼리의 **Photon 실행 비율** 을 확인할 수 있습니다. 비율이 높을수록 Photon의 이점을 더 많이 활용하고 있다는 의미입니다.

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

## Photon 내부 아키텍처 심화

Photon의 성능 이점을 정확히 이해하려면, 내부 동작 원리를 더 깊이 알아야 합니다.

### Vectorized Execution (벡터화 실행) 상세

기존 Spark의 Whole-Stage CodeGen은 행(row) 단위로 파이프라인을 생성합니다. 반면 Photon은 **열(column) 단위 배치** 로 처리합니다.

| 항목 | Spark Whole-Stage CodeGen | Photon Vectorized |
|------|--------------------------|-------------------|
| ** 처리 단위**| 1행씩 연산자를 거침 | 1,024~4,096행을 한 번에 처리 |
| ** 데이터 레이아웃**| Row-oriented (Tungsten) | Column-oriented (Arrow 호환) |
| ** 분기 예측**| 행마다 조건 분기 | 배치 전체에 동일 연산 적용(분기 최소화) |
| **CPU 파이프라인 활용**| 분기 미스로 파이프라인 스톨 빈번 | 분기 없는 벡터 연산으로 파이프라인 효율 극대화 |

### SIMD 활용 상세

Photon은 CPU의 **AVX2/AVX-512** 명령어 세트를 적극 활용합니다.

```
# SIMD 없이 (스칼라 연산): 4개 값 덧셈 → 4번 연산
a[0] + b[0] → c[0]
a[1] + b[1] → c[1]
a[2] + b[2] → c[2]
a[3] + b[3] → c[3]

# AVX2 SIMD: 4개 값 덧셈 → 1번 연산 (256-bit 레지스터)
VADDPD ymm0, ymm1, ymm2    # 4개의 64-bit double을 한 번에 덧셈

# AVX-512 SIMD: 8개 값 덧셈 → 1번 연산 (512-bit 레지스터)
VADDPD zmm0, zmm1, zmm2    # 8개의 64-bit double을 한 번에 덧셈
```

| SIMD 세대 | 레지스터 폭 | 동시 처리 (64-bit) | 주요 인스턴스 |
|-----------|-----------|-------------------|-------------|
| **SSE**| 128-bit | 2개 | 구형 인스턴스 |
| **AVX2**| 256-bit | 4개 | m5, r5 시리즈 |
| **AVX-512**| 512-bit | 8개 | m5n, r5n, i3en 시리즈 |

> 💡 ** 인스턴스 선택 팁**: AVX-512를 지원하는 인스턴스(예: Intel Ice Lake 이상)에서 Photon의 성능 이점이 가장 극대화됩니다. AWS의 경우 `m6i`, `r6i`, `i4i` 시리즈가 해당됩니다.

### Columnar Processing 내부 동작

Photon은 Delta Lake의 Parquet 파일을 읽을 때, ** 컬럼 단위로 메모리에 적재** 합니다.

```
# 기존 Spark: Row 기반으로 디코딩 후 처리
Parquet Column → Decode → Row 변환 → Row 단위 처리

# Photon: Column 그대로 메모리에 적재하여 벡터 연산
Parquet Column → Direct Column Buffer → SIMD 벡터 연산
```

이 방식의 장점:
- **CPU 캐시 라인 활용률 극대화**: 같은 컬럼의 연속된 값이 메모리에 인접하므로, L1/L2 캐시 히트율이 높아집니다
- ** 데이터 압축 활용**: 컬럼 단위 인코딩(Dictionary, RLE 등)을 디코딩 없이 직접 연산할 수 있는 경우가 있습니다
- ** 불필요한 컬럼 스킵**: 쿼리에 필요한 컬럼만 읽으므로 I/O가 줄어듭니다

---

## 지원/미지원 연산 상세

### Photon에서 특히 빠른 연산

| 워크로드 유형 | 왜 빠른가 | 성능 향상 |
|-------------|---------|----------|
| **Full Table Scan**| SIMD로 대량 데이터를 한꺼번에 필터링합니다 | 3~5x |
| **GROUP BY 집계**| 벡터화 해시 테이블로 배치 단위 집계합니다 | 3~5x |
| **Hash Join**| 벡터화 해시 프로브로 조인 성능이 향상됩니다 | 2~4x |
| ** 문자열 처리**| C++ 네이티브 문자열 라이브러리 사용. JVM String 오버헤드 없음 | 3~8x |
| **Delta MERGE**| Scan + Join + Write가 모두 Photon으로 가속됩니다 | 2~4x |
| **Parquet/Delta 쓰기**| 벡터화 인코딩 + 네이티브 I/O | 2~3x |
| **Window 함수**| 파티션 내 정렬과 연산이 벡터화됩니다 | 2~3x |

### Photon이 느린 또는 효과 없는 경우

| 워크로드 유형 | 이유 | 대안 |
|-------------|------|------|
| **Python UDF**| UDF는 JVM/Python 인터프리터에서 실행되므로 Photon 우회 | Pandas UDF(Arrow 기반)로 전환하거나 SQL 내장 함수 사용 |
| **Scala/Java UDF**| UDF 코드가 JVM에서 실행됩니다 | SQL 내장 함수로 대체 |
| **CSV/JSON 직접 읽기**| Photon은 Parquet/Delta만 가속합니다 | Auto Loader로 Delta로 먼저 변환 |
| ** 소량 데이터 처리**| 벡터화 초기화 오버헤드가 데이터 처리 시간보다 클 수 있습니다 | 데이터가 수 MB 이하면 Photon 효과 미미 |
| **ML 학습 (MLlib)**| ML 알고리즘은 별도의 실행 경로를 사용합니다 | GPU 클러스터가 더 효과적 |
| ** 스트리밍 마이크로배치**| 배치 크기가 작으면 벡터화 이점이 줄어듭니다 | 마이크로배치 간격을 충분히 크게(10초+) 설정 |

### Photon 실행 비율 확인 및 개선

Spark UI에서 Photon 비율이 낮다면 다음을 점검하세요.

| 점검 항목 | 확인 방법 | 개선 방법 |
|-----------|----------|----------|
| **UDF 사용 여부**| Spark Plan에서 `PythonUDF` 노드 확인 | SQL 내장 함수로 대체 |
| ** 파일 포맷**| 소스가 CSV/JSON인지 확인 | Delta로 변환 후 읽기 |
| ** 복잡한 타입**| Map, Array, Struct 컬럼 연산 | 단순 타입으로 평탄화(flatten) |
| **Spark 버전**| 구버전 Runtime 사용 | 최신 Runtime으로 업그레이드(매 버전 Photon 커버리지 확대) |

---

## DBU 단가 차이와 ROI 계산

### 정확한 비용 비교 프레임워크

Photon의 비용 효율성을 판단하려면 ** 총 비용(TCO)** 을 비교해야 합니다.

```
# 총 비용 공식
총 비용 = 실행 시간 x (인스턴스 비용/hr + DBU 수 x DBU 단가/hr)

# Photon ROI 공식
ROI = (Photon 없이 총 비용 - Photon 총 비용) / Photon 없이 총 비용 x 100%
```

### 워크로드별 ROI 시뮬레이션

| 시나리오 | 클러스터 | Spark 실행시간 | Photon 실행시간 | Spark 비용 | Photon 비용 | ROI |
|---------|---------|-------------|---------------|-----------|------------|-----|
| ** 일일 ETL (집계 중심)** | 4 Workers, r5.2xlarge | 2시간 | 40분 | $48 | $28 | **+42%**|
| **DBSQL 대시보드 쿼리**| SQL Warehouse (Medium) | 30초/쿼리 | 10초/쿼리 | 기본 요금 | 기본 요금 | ** 추가 비용 없음**|
| ** 문자열 ETL (LIKE, REGEX)**| 8 Workers, m5.xlarge | 3시간 | 30분 | $72 | $21 | **+71%**|
| **Python UDF 파이프라인**| 4 Workers, m5.xlarge | 1시간 | 55분 | $24 | $24 | **-8%**(DBU 단가만 증가) |
| ** 소규모 탐색 쿼리**| 2 Workers, m5.large | 10초 | 8초 | $0.04 | $0.05 | **-20%**|

> 💡 **ROI 판단 기준**: Photon이 실행 시간을 **1.5배 이상** 단축하는 워크로드에서 비용 이점이 있습니다. UDF가 전체 실행 시간의 50% 이상을 차지하면 Photon 효과가 미미합니다.

### 하이브리드 전략

모든 워크로드에 Photon을 적용하기보다, 워크로드 특성에 따라 선택적으로 적용하는 것이 비용 효율적입니다.

| 클러스터/파이프라인 | Photon 권장 | 이유 |
|-------------------|-----------|------|
| **SQL Warehouse**| 항상 ON (기본) | 추가 비용 없이 SQL 성능 향상 |
| **ETL Job (SQL/DataFrame 중심)**| ON | 집계/조인/쓰기 가속으로 ROI 양호 |
| **ETL Job (UDF 중심)**| OFF | Photon 효과 미미, DBU 단가만 상승 |
| **SDP 파이프라인**| ON | 파이프라인 전체가 SQL 기반이므로 효과 큼 |
| **ML Feature Engineering**| ON | SQL 기반 피처 변환에 효과적 |
| **ML Training**| OFF | GPU가 더 효과적, Photon 효과 없음 |
| **Interactive Notebook (탐색)**| 선택적 | 대용량 데이터 탐색 시에만 ON |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Photon**| C++ 기반 벡터화 실행 엔진으로, SQL/DataFrame 성능을 2~8배 향상시킵니다 |
| ** 벡터화 처리**| 행 단위가 아닌 배치(열) 단위로 데이터를 처리하여 SIMD를 활용합니다 |
| **JVM 오버헤드 제거**| GC 일시 정지가 없고, 메모리를 직접 관리합니다 |
| ** 자동 폴백**| 미지원 연산은 자동으로 기존 Spark 엔진으로 실행됩니다 |
| ** 비용 효과**| 집계/조인 중심 워크로드에서 총 비용이 감소합니다 |
| **SIMD 활용**| AVX2/AVX-512로 한 번의 CPU 명령으로 여러 데이터를 동시 처리합니다 |
| **ROI 판단**| 실행 시간 1.5배+ 단축 시 비용 이점. UDF 비중이 높으면 효과 미미합니다 |
| ** 하이브리드 전략** | SQL 중심 워크로드에만 선택적으로 Photon을 적용하는 것이 최적입니다 |

---

## 참고 링크

- [Databricks: Photon runtime](https://docs.databricks.com/aws/en/compute/photon.html)
- [Databricks: Photon FAQ](https://docs.databricks.com/aws/en/compute/photon.html#photon-faq)
- [Databricks Blog: Photon Technical Deep Dive](https://www.databricks.com/blog/2022/01/28/photon-technical-deep-dive.html)
- [Azure Databricks: Photon](https://learn.microsoft.com/en-us/azure/databricks/compute/photon)
