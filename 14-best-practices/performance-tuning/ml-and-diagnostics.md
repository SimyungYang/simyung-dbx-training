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

## 6. GPU 클러스터 사이징

### 6.1 GPU 인스턴스 비교

| GPU 인스턴스 | GPU | GPU 메모리 | 적합한 용도 | 시간당 비용 (USD, 대략) |
|------------|-----|---------|-----------|---------------------|
| **g5.xlarge** | A10G × 1 | 24GB | 소규모 파인튜닝, 추론 | $1.01 |
| **g5.2xlarge** | A10G × 1 | 24GB | 파인튜닝 + 큰 CPU 메모리 | $1.21 |
| **g5.12xlarge** | A10G × 4 | 96GB | 중규모 분산 학습 | $5.67 |
| **g5.48xlarge** | A10G × 8 | 192GB | 대규모 분산 학습 | $16.29 |
| **p4d.24xlarge** | A100 × 8 | 320GB | LLM 파인튜닝, 대규모 모델 | $32.77 |
| **p5.48xlarge** | H100 × 8 | 640GB | LLM 사전 학습, 초대규모 | $98.32 |

### 6.2 모델 크기별 GPU 선택 가이드

| 모델 규모 | 파라미터 수 | 최소 GPU 메모리 | 권장 인스턴스 | 비고 |
|---------|-----------|-------------|-----------|------|
| **소형** | < 1B | 8~16GB | g5.xlarge (A10G × 1) | scikit-learn, XGBoost도 포함 |
| **중형** | 1~7B | 24~48GB | g5.xlarge ~ g5.2xlarge | Llama 3 8B, Mistral 7B |
| **대형** | 7~13B | 48~96GB | g5.12xlarge (A10G × 4) | Llama 3 13B |
| **초대형** | 13~70B | 160~320GB | p4d.24xlarge (A100 × 8) | Llama 3 70B, DBRX |
| **거대** | 70B+ | 640GB+ | p5.48xlarge (H100 × 8) | 사전 학습용 |

{% hint style="info" %}
**GPU 메모리 계산 공식**: 모델 파라미터를 float16(2바이트)으로 로드할 때 필요한 GPU 메모리 = `파라미터 수 × 2바이트`. 예: 7B 모델 = 7 × 10^9 × 2 = 14GB. 학습 시에는 옵티마이저 상태와 그래디언트로 인해 **3~4배** 추가 메모리가 필요합니다.
{% endhint %}

### 6.3 GPU 활용률 모니터링

```python
# GPU 사용률 확인 (노트북에서)
import subprocess
result = subprocess.run(['nvidia-smi', '--query-gpu=utilization.gpu,memory.used,memory.total',
                        '--format=csv,noheader,nounits'], capture_output=True, text=True)
print(result.stdout)

# GPU 사용률 지속 모니터링 (학습 중)
# nvidia-smi dmon -s u -d 5  # 5초 간격으로 사용률 출력
```

| GPU 활용률 | 진단 | 조치 |
|----------|------|------|
| **< 30%** | GPU 낭비 (데이터 로딩 병목) | 데이터 로더 최적화, num_workers 증가 |
| **30~70%** | 적정 (약간의 여유) | 배치 사이즈 증가 시도 |
| **70~95%** | 최적 활용 | 현재 설정 유지 |
| **> 95%** | OOM 위험 | 배치 사이즈 축소 또는 큰 GPU |

---

## 7. 분산 학습 성능 튜닝

### 7.1 TorchDistributor 활용

```python
from pyspark.ml.torch.distributor import TorchDistributor

def train_fn():
    import torch
    import torch.distributed as dist
    from torch.nn.parallel import DistributedDataParallel as DDP
    
    # 분산 환경 초기화 (TorchDistributor가 자동 설정)
    local_rank = int(os.environ.get("LOCAL_RANK", 0))
    device = torch.device(f"cuda:{local_rank}")
    
    model = MyModel().to(device)
    model = DDP(model, device_ids=[local_rank])
    
    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)
    
    for epoch in range(10):
        for batch in train_loader:
            loss = model(batch)
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
    
    return model

# Databricks에서 분산 학습 실행
distributor = TorchDistributor(
    num_processes=4,     # 총 GPU 수
    local_mode=False,    # 멀티 노드
    use_gpu=True
)
model = distributor.run(train_fn)
```

### 7.2 분산 학습 성능 최적화

| 최적화 항목 | 전략 | 효과 |
|-----------|------|------|
| **데이터 로딩** | num_workers=4, pin_memory=True | GPU 대기 시간 감소 |
| **배치 사이즈** | GPU 메모리의 70~80% 활용 | GPU 활용률 극대화 |
| **통신 백엔드** | NCCL (GPU 간), Gloo (CPU) | 그래디언트 동기화 속도 |
| **Mixed Precision** | torch.cuda.amp | 메모리 50% 절감, 속도 2x |
| **Gradient Accumulation** | GPU 메모리 부족 시 | 실효 배치 사이즈 증가 |

```python
# Mixed Precision Training (메모리 절감 + 속도 향상)
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()

for batch in train_loader:
    optimizer.zero_grad()
    
    with autocast():  # FP16으로 연산
        output = model(batch)
        loss = criterion(output, labels)
    
    scaler.scale(loss).backward()  # 그래디언트 스케일링
    scaler.step(optimizer)
    scaler.update()

# Gradient Accumulation (큰 배치 효과를 작은 GPU로)
accumulation_steps = 4
for i, batch in enumerate(train_loader):
    loss = model(batch) / accumulation_steps
    loss.backward()
    
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

{% hint style="warning" %}
**분산 학습 주의사항**: 노드 간 네트워크 대역폭이 중요합니다. Databricks에서 멀티 노드 학습 시 **EFA(Elastic Fabric Adapter)** 또는 고대역폭 네트워크가 지원되는 인스턴스(p4d, p5)를 사용하세요. 네트워크 병목이 있으면 GPU가 유휴 상태로 대기합니다.
{% endhint %}

---

## 8. Spark ML vs 단일 노드 성능 비교

### 8.1 선택 가이드

| 기준 | 단일 노드 (scikit-learn, XGBoost) | Spark ML |
|------|-------------------------------|----------|
| **데이터 크기** | < 100GB (메모리 내) | 100GB+ (분산 처리) |
| **모델 복잡도** | 높음 (딥러닝, 복잡한 앙상블) | 낮~중 (선형, 트리, 기본 앙상블) |
| **Feature 수** | 1000+ | 100~1000 |
| **학습 속도** | 빠름 (단일 노드 최적화) | 데이터 분산 오버헤드 |
| **하이퍼파라미터 튜닝** | Optuna, Hyperopt | Spark ML CrossValidator |
| **MLflow 통합** | 네이티브 | 네이티브 |

```python
# 단일 노드: Pandas + scikit-learn (데이터 < 100GB)
import pandas as pd
from sklearn.ensemble import GradientBoostingClassifier
import mlflow

# Spark 테이블을 Pandas로 변환 (데이터가 메모리에 들어갈 때)
df = spark.table("catalog.schema.features").toPandas()

with mlflow.start_run():
    model = GradientBoostingClassifier(n_estimators=100)
    model.fit(df[features], df[target])
    mlflow.sklearn.log_model(model, "model")

# 분산: Spark ML (데이터 > 100GB)
from pyspark.ml.classification import GBTClassifier
from pyspark.ml.feature import VectorAssembler

assembler = VectorAssembler(inputCols=features, outputCol="features")
gbt = GBTClassifier(featuresCol="features", labelCol="target", maxIter=100)

pipeline = Pipeline(stages=[assembler, gbt])
model = pipeline.fit(spark.table("catalog.schema.features"))
```

### 8.2 하이브리드 패턴

```python
# 대용량 데이터에서 Feature Engineering은 Spark로,
# 학습은 단일 노드로 하는 하이브리드 패턴

# Step 1: Spark로 대규모 Feature Engineering (분산)
features_df = (
    spark.table("catalog.schema.raw_data")  # 1TB
    .withColumn("feature_1", ...)
    .withColumn("feature_2", ...)
    .groupBy("entity_id")
    .agg(...)
)
features_df.write.saveAsTable("catalog.schema.features")  # 50GB

# Step 2: 학습용 샘플링 (50GB → 5GB)
sample_df = features_df.sample(fraction=0.1).toPandas()

# Step 3: 단일 노드에서 학습 (최적화된 라이브러리 활용)
import xgboost as xgb
model = xgb.XGBClassifier(tree_method="gpu_hist", gpu_id=0)
model.fit(sample_df[feature_cols], sample_df[target_col])

# Step 4: 전체 데이터에 Spark UDF로 추론 (분산)
predict_udf = mlflow.pyfunc.spark_udf(spark, "models:/my_model/production")
predictions = features_df.withColumn("prediction", predict_udf(struct(*feature_cols)))
```

---

## 9. OOM 디버깅

### 9.1 OOM 유형별 진단

| OOM 유형 | 에러 메시지 | 원인 | 해결 |
|---------|-----------|------|------|
| **Driver OOM** | `java.lang.OutOfMemoryError: Java heap space` (Driver) | collect(), toPandas(), 큰 Broadcast | Driver 메모리 증가 또는 collect 제거 |
| **Executor OOM** | `java.lang.OutOfMemoryError: Java heap space` (Executor) | 데이터 편향, 큰 파티션 | 파티션 수 증가, 메모리 증가 |
| **Container OOM** | `Container killed by YARN for exceeding memory limits` | Off-heap 메모리 초과 | `spark.memory.offHeap.size` 증가 |
| **GPU OOM** | `CUDA out of memory` | 배치 사이즈 과대, 모델 과대 | 배치 사이즈 축소, Mixed Precision |

### 9.2 OOM 해결 체크리스트

```text
[Driver OOM 체크리스트]
□ collect(), toPandas(), show(매우 큰 수) 사용하고 있는가?
  → 필터/집계 후 소량만 수집
□ Broadcast 변수가 너무 큰가? (> 1GB)
  → Broadcast 임계값 낮추기
□ 대시보드/노트북에서 대량 결과를 표시하는가?
  → LIMIT 추가

[Executor OOM 체크리스트]
□ Spill to Disk가 발생하는가?
  → 파티션 수 증가: spark.sql.shuffle.partitions = 2000
□ 데이터 Skew가 있는가?
  → AQE Skew Join 활성화, Salting 적용
□ 인스턴스 메모리가 부족한가?
  → Memory-optimized 인스턴스 (r5, r6i)

[GPU OOM 체크리스트]
□ 배치 사이즈가 너무 큰가?
  → 배치 사이즈 절반으로 줄이기
□ Mixed Precision을 사용하고 있는가?
  → torch.cuda.amp 적용 (메모리 50% 절감)
□ Gradient Checkpointing을 사용하는가?
  → model.gradient_checkpointing_enable() (메모리 절감)
```

```python
# Driver OOM 방지: 대량 데이터 수집 대신 파일로 저장
# ❌ 위험
large_df = spark.table("catalog.schema.big_table")
pandas_df = large_df.toPandas()  # OOM!

# ✅ 안전
large_df.write.format("delta").save("/tmp/result")
# 필요한 부분만 읽기
small_df = spark.read.format("delta").load("/tmp/result").filter("date = '2026-03-01'").toPandas()
```

---

## 10. GC 튜닝

### 10.1 GC 문제 진단

```text
[GC 문제 증상]
- Spark UI의 Executor 탭에서 GC Time이 전체 시간의 10% 이상
- 태스크 실행 시간이 불규칙 (GC Pause로 인한 간헐적 지연)
- "GC overhead limit exceeded" 에러
```

| GC 설정 | 기본값 | 권장값 | 효과 |
|--------|-------|-------|------|
| **`-XX:+UseG1GC`** | G1GC (기본) | G1GC 유지 | Spark에 최적화 |
| **`-XX:G1HeapRegionSize`** | 자동 | 16m | 대용량 힙에서 효율적 |
| **`-XX:InitiatingHeapOccupancyPercent`** | 45 | 35 | GC 일찍 시작하여 긴 Pause 방지 |
| **`-Xmx`** | 클러스터 설정 | 인스턴스 메모리의 60~70% | 나머지는 Off-heap/OS 용 |

```python
# GC 튜닝 설정 (클러스터 Spark 설정)
spark.conf.set("spark.executor.extraJavaOptions",
    "-XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:InitiatingHeapOccupancyPercent=35")

# GC 로그 활성화 (디버깅용)
spark.conf.set("spark.executor.extraJavaOptions",
    "-XX:+UseG1GC -Xlog:gc*:file=/tmp/gc.log:time,uptime:filecount=5,filesize=10m")
```

### 10.2 GC 부담 줄이는 코딩 패턴

```python
# ❌ GC 부담 높은 패턴: 불필요한 중간 DataFrame 생성
df1 = spark.table("table1")
df2 = df1.filter("date > '2026-01-01'")
df3 = df2.withColumn("new_col", ...)
df4 = df3.groupBy("region").agg(...)
df5 = df4.orderBy("total")

# ✅ GC 친화적 패턴: 체이닝으로 중간 참조 제거
result = (
    spark.table("table1")
    .filter("date > '2026-01-01'")
    .withColumn("new_col", ...)
    .groupBy("region").agg(...)
    .orderBy("total")
)

# ✅ 사용 완료한 DataFrame 캐시 해제
cached_df.unpersist()
```

---

## 11. 메모리 프로파일링

### 11.1 Spark 메모리 구조

```text
[Executor 메모리 구조]

전체 Executor 메모리 (spark.executor.memory)
├── Spark Memory (60%)
│   ├── Storage Memory: 캐시된 데이터, Broadcast
│   └── Execution Memory: Shuffle, Sort, Join, 집계
│
├── User Memory (40%)
│   ├── Python UDF 데이터
│   └── 사용자 데이터 구조
│
└── Reserved Memory (300MB 고정)

Off-Heap Memory (spark.memory.offHeap.size)
├── Photon 엔진 메모리
└── Arrow 직렬화 버퍼
```

### 11.2 메모리 사용 진단

```python
# 현재 메모리 설정 확인
print(f"Executor Memory: {spark.conf.get('spark.executor.memory')}")
print(f"Driver Memory: {spark.conf.get('spark.driver.memory')}")
print(f"Memory Fraction: {spark.conf.get('spark.memory.fraction')}")
print(f"Off-Heap Size: {spark.conf.get('spark.memory.offHeap.size', 'not set')}")

# 캐시된 데이터 크기 확인
for table_name in ["table1", "table2"]:
    if spark.catalog.isCached(table_name):
        print(f"{table_name}: cached")
```

| 메모리 설정 | 조정 방향 | 시나리오 |
|-----------|---------|---------|
| `spark.executor.memory` | ↑ | Spill 발생, OOM |
| `spark.memory.fraction` | ↑ (0.6→0.7) | 캐시 많이 사용, 큰 집계 |
| `spark.memory.offHeap.size` | ↑ | Photon 사용, Container OOM |
| `spark.sql.shuffle.partitions` | ↑ | 파티션당 메모리 초과 |
| `spark.driver.memory` | ↑ | Driver OOM, 큰 Broadcast |

{% hint style="info" %}
**메모리 최적화 우선순위**: 1) 불필요한 컬럼 제거 (SELECT *  금지) → 2) 파티션 수 조정 → 3) Broadcast 임계값 조정 → 4) 인스턴스 메모리 증가. 인스턴스 업그레이드는 마지막 수단으로, 코드 최적화를 먼저 시도하세요.
{% endhint %}

---

## 12. 성능 진단 체크리스트

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
- [TorchDistributor](https://docs.databricks.com/aws/en/machine-learning/train-model/distributed-training/spark-pytorch-distributor)
- [GPU Cluster Configuration](https://docs.databricks.com/aws/en/compute/gpu)
