# ML/AI 최적화와 성능 진단

## 5. ML/AI 성능 최적화

### 5.1 Model Serving 지연시간 최적화

| 최적화 영역 | 전략 | 효과 |
|-----------|------|------|
| **콜드 스타트** | Minimum Instances >= 1 | 첫 요청 지연 제거 |
| **모델 크기** | ONNX 변환, 양자화 | 로드 시간 50~80% 감소 |
| **GPU 선택** | 모델 크기에 맞는 GPU | 과도한 GPU는 비용 낭비 |
| **배치 요청** | Batch Inference 활용 | 개별 요청 대비 10~100x 처리량 |
| **Feature Lookup** | Online Table 활용 | Feature 조회 < 10ms |

```python
# Model Serving 엔드포인트 설정 (최적화 버전)
import mlflow

# 모델 등록 시 최적화 힌트
mlflow.pyfunc.log_model(
    artifact_path="model",
    python_model=my_model,
    pip_requirements=["scikit-learn==1.3.0"],  # 최소 의존성
    metadata={"serving_optimization": "latency"}
)
```

### 5.2 배치 추론 병렬화

```python
import mlflow
from pyspark.sql.functions import struct

# Spark UDF로 배치 추론 병렬화
model_uri = "models:/fraud_detection/production"
predict_udf = mlflow.pyfunc.spark_udf(spark, model_uri)

# 대용량 데이터셋에 병렬 추론 적용
predictions = (
    spark.table("catalog.schema.transactions")
    .withColumn("prediction", predict_udf(struct("feature1", "feature2", "feature3")))
)

# 최적화 팁: 파티션 수를 코어 수의 2~3배로 설정
predictions.repartition(spark.sparkContext.defaultParallelism * 2)
```

### 5.3 Vector Search 검색 속도 최적화

| 최적화 항목 | 권장 설정 | 이유 |
|-----------|----------|------|
| **임베딩 차원** | 768~1536 (모델에 따라) | 불필요하게 큰 차원은 속도 저하 |
| **인덱스 유형** | Delta Sync Index | 자동 동기화, 운영 부담 최소 |
| **엔드포인트 크기** | 데이터 크기에 비례 | 1M 문서 미만: Small, 이상: Medium+ |
| **필터링** | 메타데이터 필터 활용 | 검색 범위 축소로 속도/정확도 향상 |

```python
# Vector Search 쿼리 최적화
results = index.similarity_search(
    query_text="고객 이탈 예측 방법",
    columns=["content", "source", "page"],  # 필요한 컬럼만 반환
    filters={"category": "ml", "year": 2025},  # 메타데이터 필터로 범위 축소
    num_results=5  # 필요한 만큼만 요청
)
```

---

## 6. 성능 진단 체크리스트

단계별로 성능 문제를 진단하고 해결하는 가이드입니다.

### Step 1: 쿼리가 느린가?

```text
□ Query Profile에서 가장 시간이 오래 걸리는 단계는?
  → Scan이 오래 걸림 → Step 2로
  → Join이 오래 걸림 → Step 3으로
  → Aggregate가 오래 걸림 → Step 4로
```

### Step 2: 스캔 최적화

```text
□ 읽은 데이터량 vs 필요한 데이터량 비교
  → 과도한 스캔 → Liquid Clustering 적용했는가?
  → 클러스터링 적용됨 → OPTIMIZE를 최근에 실행했는가?
  → OPTIMIZE 실행됨 → 클러스터링 키가 쿼리 패턴과 맞는가?
□ ANALYZE TABLE로 통계를 수집했는가?
□ 파일 수가 과도하지 않은가? (Small File 문제)
```

### Step 3: 조인 최적화

```text
□ 조인 유형 확인 (Broadcast vs SortMerge)
  → 한쪽 테이블이 작은데 SortMerge? → Broadcast 힌트 추가
  → 양쪽 모두 대용량? → Shuffle Partition 수 조정
□ 데이터 편향(Skew) 존재하는가?
  → 특정 태스크만 오래 걸림 → AQE Skew Join 설정 확인
  → 극심한 Skew → Salting 기법 적용
□ 조인 키에 NULL이 많은가?
  → NULL 키는 사전 필터링
```

### Step 4: 집계/셔플 최적화

```text
□ Shuffle 데이터량이 과도한가?
  → Pre-aggregate 가능한가? (서브쿼리에서 먼저 집계)
  → Shuffle Partition 수 조정 (기본 200, 데이터에 따라 조절)
□ Spill to Disk 발생하는가?
  → 메모리 확대 (Memory-optimized 인스턴스)
  → Partition 수 증가로 단위 데이터 축소
□ Photon이 활성화되어 있는가?
  → SQL 집계 워크로드에 Photon 적용 시 2~8x 향상
```

### Step 5: 리소스 최적화

```text
□ 클러스터/Warehouse 사이즈가 적절한가?
  → CPU 사용률 > 80% 지속 → 스케일 업
  → 메모리 사용률 > 80% → Memory-optimized 인스턴스
  → I/O 대기 높음 → Storage-optimized 인스턴스
□ Autoscaling이 적절히 동작하는가?
  → Min/Max 워커 수 검토
□ 네트워크 병목이 있는가?
  → 리전 간 데이터 전송 최소화
  → 같은 리전에 컴퓨트와 스토리지 배치
```

### 빠른 참조: 증상별 해결 매트릭스

| 증상 | 1순위 확인 | 2순위 확인 | 3순위 확인 |
|------|----------|----------|----------|
| 쿼리 시작이 느림 | Warehouse Auto Stop 설정 | Instance Pool 사용 | Serverless 전환 |
| 스캔이 느림 | Liquid Clustering | 통계 수집 | 파일 크기 최적화 |
| 조인이 느림 | Broadcast 가능 여부 | Skew 확인 | 클러스터 사이즈 |
| 집계가 느림 | Photon 활성화 | MV 사전 계산 | Partition 수 조정 |
| Spill 발생 | 메모리 확대 | Partition 수 증가 | 쿼리 분리 |
| 스트리밍 지연 | 트리거 간격 | Checkpoint 위치 | 파일 수 제한 |
| ML 추론 느림 | 모델 최적화 | GPU 선택 | 배치 크기 조정 |

---

## 참고 링크

- [Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering)
- [Data Skipping](https://docs.databricks.com/aws/en/delta/data-skipping)
- [Photon 엔진](https://docs.databricks.com/aws/en/compute/photon)
- [클러스터 구성 모범 사례](https://docs.databricks.com/aws/en/compute/cluster-config-best-practices)
- [SQL Warehouse 사이징 가이드](https://docs.databricks.com/aws/en/compute/sql-warehouse/warehouse-behavior)
- [AQE(Adaptive Query Execution)](https://docs.databricks.com/aws/en/optimizations/aqe)
- [Predictive Optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
- [Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/index)
