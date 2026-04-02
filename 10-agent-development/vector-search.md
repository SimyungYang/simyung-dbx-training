# Vector Search — 벡터 유사도 검색

## 왜 Vector Search가 필요한가요?

전통적인 키워드 검색(SQL의 `LIKE` 또는 `CONTAINS`)은 **정확히 일치하는 단어** 만 찾을 수 있습니다. "환불 규정"을 검색하면 "반품 정책"은 찾지 못합니다. 그러나 이 둘은 의미적으로 매우 유사합니다.

**Vector Search** 는 텍스트의 **의미(Semantic)** 를 이해하여, 키워드가 다르더라도 의미가 유사한 항목을 찾아줍니다. RAG(검색 증강 생성) 기반 AI 에이전트의 핵심 구성 요소입니다.

| 검색 유형 | 방식 | 장점 | 한계 |
|-----------|------|------|------|
| **키워드 검색** | 정확한 단어 매칭 | 빠르고 예측 가능 | 동의어, 의미 유사성 처리 불가 |
| **Vector Search** | 임베딩 벡터 유사도 | 의미적 유사성 검색 가능 | 임베딩 모델에 의존 |
| **하이브리드 검색** | 키워드 + 벡터 결합 | 두 방식의 장점 결합 | 구성이 복잡 |

---

## 핵심 개념: 임베딩 (Embedding)

> 💡 **임베딩(Embedding)** 이란 텍스트를 수백~수천 차원의 **숫자 벡터(배열)** 로 변환하는 것입니다. 의미가 유사한 텍스트는 벡터 공간에서 **가까운 위치** 에 놓이게 됩니다.

```
"강아지"  → [0.23, -0.15, 0.87, 0.42, ...]  (1024차원)
"개"      → [0.25, -0.13, 0.85, 0.40, ...]  ← 가까움! (유사함)
"자동차"  → [-0.71, 0.34, -0.28, 0.55, ...] ← 멂! (다름)
```

> 💡 **코사인 유사도(Cosine Similarity)란?** 두 벡터 간의 각도를 기반으로 유사도를 측정하는 방법입니다. 값은 -1에서 1 사이이며, 1에 가까울수록 유사합니다.

---

## Databricks Vector Search 아키텍처

![Vector Search - 임베딩 자동 계산 방식 아키텍처](https://docs.databricks.com/aws/en/assets/images/calculate-embeddings-c6fa28b679c1cf21b1f93434a0ed927d.png)

*출처: [Databricks 공식 문서](https://docs.databricks.com/aws/en/generative-ai/vector-search/index.html)*
| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| **1. 데이터 준비** | Delta 테이블 | 원본 텍스트가 저장된 소스 테이블입니다 |
| **2. 인덱스 생성** | Vector Search Endpoint | 인덱스를 호스팅하는 컴퓨팅 리소스입니다 |
|  | 임베딩 모델 | 텍스트를 벡터로 자동 변환합니다 |
|  | Vector Search Index | Delta Sync로 소스 테이블과 동기화되어 임베딩 벡터를 저장합니다 |
| **3. 검색** | 질문 텍스트 | 질문이 임베딩으로 변환되어 유사도 검색이 수행됩니다 |
|  | 유사 문서 결과 | 인덱스에서 가장 유사한 문서가 반환됩니다 |

| 구성 요소 | 역할 |
|-----------|------|
| **Vector Search Endpoint** | 인덱스를 호스팅하고 검색 쿼리를 처리하는 컴퓨팅 리소스입니다 |
| **Vector Search Index** | 임베딩 벡터가 저장된 검색 인덱스입니다 |
| **Delta Sync** | 소스 Delta 테이블이 변경되면 인덱스를 자동으로 갱신합니다 |

---

## Endpoint 생성

```python
from databricks.vector_search.client import VectorSearchClient

vsc = VectorSearchClient()

# Endpoint 생성
vsc.create_endpoint(
    name="vs-endpoint-prod",
    endpoint_type="STANDARD"  # STANDARD 또는 STORAGE_OPTIMIZED
)
```

| 유형 | 설명 | 적합한 사용 |
|------|------|-----------|
| **STANDARD** | 범용. 빠른 검색 성능 | 대부분의 RAG 애플리케이션 |
| **STORAGE_OPTIMIZED** | 대용량에 최적화, 비용 효율적 | 수천만 건 이상의 대규모 인덱스 |

---

## Index 유형 (3가지)

### 1. Delta Sync + Managed Embeddings (권장)

가장 간편한 방식입니다. 소스 Delta 테이블의 **텍스트 컬럼** 을 지정하면, Databricks가 **임베딩 변환부터 인덱스 동기화까지 모든 과정을 자동으로** 관리합니다. 사용자는 임베딩 모델을 선택하기만 하면 됩니다.

**동작 원리:**
1. 소스 Delta 테이블에서 `embedding_source_column`(텍스트 컬럼)을 읽습니다
2. 지정된 임베딩 모델(`embedding_model_endpoint_name`)로 텍스트를 벡터로 변환합니다
3. 변환된 벡터를 Vector Search 인덱스에 저장합니다
4. `TRIGGERED` 모드는 수동/스케줄 갱신, `CONTINUOUS` 모드는 소스 변경 시 실시간 반영합니다

**적합한 경우:** 대부분의 RAG 애플리케이션에 권장됩니다. 임베딩 파이프라인을 직접 구축할 필요가 없으므로 빠르게 시작할 수 있습니다.

```python
index = vsc.create_delta_sync_index(
    endpoint_name="vs-endpoint-prod",
    index_name="catalog.schema.docs_index",
    source_table_name="catalog.schema.documents",
    primary_key="doc_id",
    embedding_source_column="content",           # 임베딩할 텍스트 컬럼
    embedding_model_endpoint_name="databricks-gte-large-en",  # 임베딩 모델
    pipeline_type="TRIGGERED",  # TRIGGERED: 수동/스케줄 | CONTINUOUS: 실시간
    columns_to_sync=["doc_id", "content", "title", "source", "updated_at"]
)
```

| 파라미터 | 설명 |
|---------|------|
| `embedding_source_column` | 임베딩으로 변환할 텍스트 컬럼입니다 |
| `embedding_model_endpoint_name` | 사용할 임베딩 모델의 서빙 엔드포인트입니다 |
| `pipeline_type` | `TRIGGERED`: 수동 트리거 시 동기화 / `CONTINUOUS`: 변경 즉시 동기화 |
| `columns_to_sync` | 검색 결과에 포함할 추가 컬럼들입니다 (메타데이터) |

---

### 2. Delta Sync + Self-Managed Embeddings

임베딩을 **직접 사전 계산** 하여 Delta 테이블에 벡터 컬럼으로 저장한 경우 사용합니다. 커스텀 임베딩 모델(예: 한국어 전용 모델, 도메인 특화 모델)을 사용하거나, 임베딩 생성 과정을 세밀하게 제어하고 싶을 때 적합합니다.

**동작 원리:**
1. 사용자가 직접 임베딩을 계산하여 Delta 테이블의 `embedding` 컬럼에 저장합니다
2. Vector Search는 이미 계산된 벡터를 그대로 인덱싱합니다
3. Delta 테이블이 업데이트되면 인덱스도 자동 동기화됩니다

**적합한 경우:**
- 한국어 전용 임베딩 모델(KoSimCSE, multilingual-e5 등)을 사용할 때
- 이미지 임베딩 등 텍스트가 아닌 벡터를 인덱싱할 때
- 임베딩 전처리(청킹, 필터링)를 세밀하게 제어할 때

```python
# 사전 준비: 임베딩 벡터가 포함된 Delta 테이블
# | doc_id | content         | embedding (array<float>) |
# |--------|-----------------|--------------------------|
# | 1      | "환불 절차..."   | [0.23, -0.15, 0.87, ...] |

index = vsc.create_delta_sync_index(
    endpoint_name="vs-endpoint-prod",
    index_name="catalog.schema.docs_index_self",
    source_table_name="catalog.schema.documents_with_embeddings",
    primary_key="doc_id",
    embedding_vector_column="embedding",   # 이미 계산된 벡터 컬럼
    embedding_dimension=1024,              # 벡터 차원 수
    pipeline_type="TRIGGERED"
)
| 단계 | 처리 | 결과 |
|------|------|------|
| 1 | 벡터 검색 | 의미적으로 유사한 Top-20 |
| 2 | 키워드 검색 (BM25) | 키워드 매칭 Top-20 |
| 3 | 결과 병합 (RRF) | 중복 제거, 점수 융합 → Top-20 |
| 4 | Reranker | LLM 기반 재순위화 → Top-5 |
| 5 | LLM 답변 생성 | 최종 응답 |python
# Direct Access 인덱스 생성
index = vsc.create_direct_access_index(
    endpoint_name="vs-endpoint-prod",
    index_name="catalog.schema.docs_direct",
    primary_key="doc_id",
    embedding_dimension=1024,
    schema={
        "doc_id": "string",
        "content": "string",
        "category": "string",
        "embedding": "array<float>"
    }
)

# 벡터 삽입 (upsert)
index.upsert([
    {
        "doc_id": "doc-001",
        "content": "환불은 구매 후 14일 이내에 가능합니다.",
        "category": "정책",
        "embedding": [0.23, -0.15, 0.87, ...]  # 직접 계산한 벡터
    },
    {
        "doc_id": "doc-002",
        "content": "배송은 영업일 기준 2-3일 소요됩니다.",
        "category": "배송",
        "embedding": [0.11, 0.42, -0.33, ...]
    }
])

# 벡터 삭제
index.delete(["doc-001"])
```

> ⚠️ **Direct Access의 한계**: Delta 테이블과 동기화되지 않으므로, 데이터 일관성 관리를 직접 해야 합니다. 대부분의 경우 **Delta Sync 방식을 권장** 하며, Direct Access는 특수한 실시간 요구사항이 있을 때만 사용하세요.

---

### 인덱스 유형 비교

| 비교 항목 | Delta Sync (Managed) | Delta Sync (Self-Managed) | Direct Access |
|----------|---------------------|--------------------------|---------------|
| **임베딩 생성** | Databricks 자동 | 사용자 직접 계산 | 사용자 직접 계산 |
| **Delta 동기화** | ✅ 자동 | ✅ 자동 | ❌ 수동 (API) |
| **설정 복잡도** | 낮음 (가장 간편) | 중간 | 높음 |
| **임베딩 모델 선택** | Databricks 내장 모델 | 자유 선택 | 자유 선택 |
| **적합한 경우** | 대부분의 RAG ✅ | 커스텀/다국어 모델 | 실시간 삽입, 외부 소스 |
| **권장 여부** | ⭐ 1순위 권장 | 커스텀 모델 필요 시 | 특수 요구 시에만 |

---

## 유사도 검색

### 텍스트 검색

```python
results = vsc.get_index(
    endpoint_name="vs-endpoint-prod",
    index_name="catalog.schema.docs_index"
).similarity_search(
    query_text="반품 및 환불 절차가 어떻게 되나요?",
    columns=["doc_id", "title", "content", "source"],
    num_results=5
)

for doc in results['result']['data_array']:
    print(f"Score: {doc[-1]:.4f} | Title: {doc[1]}")
```

### 필터 결합 검색

```python
results = index.similarity_search(
    query_text="데이터 보안 정책",
    columns=["doc_id", "title", "content", "category"],
    filters={"category": "보안"},
    num_results=3
)
```

---

## Vector Search Reranker

> 🆕 **Reranker(GA)**: 초기 검색 결과를 LLM 기반으로 **재순위화** 하여 정확도를 높이는 기능입니다.

```python
results = index.similarity_search(
    query_text="반품 시 배송비는 누가 부담하나요?",
    columns=["doc_id", "content"],
    num_results=20,  # 넓게 검색
    query_options={
        "reranker": {
            "model_name": "databricks-reranker",
            "columns": ["content"],
            "top_k": 5  # 상위 5개 선별
        }
    }
)
```

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | 질문 | 사용자의 검색 질문입니다 |
| 2 | Vector Search | 20개 후보 문서를 넓게 검색합니다 |
| 3 | Reranker | LLM 기반으로 재순위화하여 상위 5개를 선별합니다 |
| 4 | 최종 결과 | 가장 관련성 높은 5개 문서가 반환됩니다 |

---

## 임베딩 모델 선택

| 모델 | 차원 | 언어 | 비고 |
|------|------|------|------|
| `databricks-gte-large-en` | 1024 | 영어 | Databricks 내장 |
| `databricks-bge-large-en` | 1024 | 영어 | Databricks 내장 |
| multilingual-e5-large | 1024 | 다국어 | 한국어 성능 우수 (커스텀 배포 필요) |
| BGE-M3 | 1024 | 다국어 | 한국어 포함 100+ 언어 지원 |

> 💡 **한국어 임베딩**: 한국어 검색 품질을 높이려면, 다국어 임베딩 모델(multilingual-e5-large, BGE-M3)을 Model Serving에 배포하여 사용하는 것을 권장합니다.

---

## 심화: Principal SA 레벨 성능 튜닝 및 운영 가이드

### 인덱스 성능 튜닝

Databricks Vector Search는 내부적으로 **HNSW (Hierarchical Navigable Small World)** 알고리즘을 사용합니다. ANN(Approximate Nearest Neighbor) 검색의 성능은 인덱스 파라미터에 크게 의존합니다.

> 💡 **HNSW란?** 그래프 기반의 근사 최근접 이웃(ANN) 검색 알고리즘입니다. 데이터를 계층적 그래프로 구성하여, 정확한 전수 검색(brute-force) 대비 수백 배 빠르면서도 95%+ 재현율(recall)을 달성합니다.

#### 임베딩 차원 수와 성능 트레이드오프

| 임베딩 차원 | 검색 속도 | 메모리 사용 | 정확도 | 대표 모델 |
|-----------|----------|-----------|--------|----------|
| **384** | 매우 빠름 | 낮음 | 보통 | all-MiniLM-L6-v2, gte-small |
| **768** | 빠름 | 중간 | 좋음 | gte-base, bge-base |
| **1024** | 보통 | 높음 | 매우 좋음 | gte-large, bge-large, multilingual-e5-large |
| **1536** | 느림 | 매우 높음 | 최고 | OpenAI text-embedding-3-large |
| **3072** | 매우 느림 | 매우 높음 | 최고+ | OpenAI text-embedding-3-large (full) |

> 💡 **실무 가이드**: 대부분의 한국어 RAG 사용 사례에서 **1024차원** 이 최적의 균형점입니다. 384차원은 프로토타이핑에, 1536+ 차원은 법률/의료 등 고정밀 도메인에 적합합니다. 차원을 2배로 늘리면 메모리 사용량도 2배, 검색 지연시간은 1.3~1.5배 증가합니다.

#### ANN 검색 정확도 vs 속도

| 파라미터 | 역할 | 높이면 | 낮추면 |
|---------|------|--------|--------|
| **ef_search**(검색 시) | 탐색할 후보 수 | 정확도 향상, 속도 저하 | 속도 향상, 정확도 저하 |
| **ef_construction**(인덱스 구축 시) | 그래프 구축 품질 | 인덱스 품질 향상, 구축 시간 증가 | 구축 빠름, 품질 저하 |
| **M**(그래프 연결 수) | 노드당 이웃 수 | 정확도 향상, 메모리 증가 | 메모리 절약, 정확도 저하 |

> ⚠️ **Databricks Vector Search에서는 HNSW 파라미터를 직접 제어할 수 없습니다.** 시스템이 데이터 규모에 따라 자동 최적화합니다. 다만, `num_results` 파라미터를 늘려 후보군을 확대하고 Reranker로 정밀도를 높이는 전략이 가능합니다.

---

### 대규모 인덱스 운영 (수천만~수억 건)

| 고려사항 | 상세 | 권장 대응 |
|---------|------|----------|
| **인덱스 빌드 시간** | 1억 건 x 1024차원 기준 수 시간~하루 소요 가능 | `TRIGGERED` 모드로 업무 시간 외 동기화 스케줄링 |
| **동기화 지연** | `CONTINUOUS` 모드에서 대규모 배치 업데이트 시 수 분~수십 분 지연 | 대량 업데이트는 배치로 묶어 한 번에 수행 |
| **메모리 요구량** | 1억 건 x 1024차원 x 4바이트 = 약 400GB (인덱스 메타 포함 시 2~3배) | `STORAGE_OPTIMIZED` 엔드포인트 사용 |
| **엔드포인트 확장** | 단일 엔드포인트에 복수 인덱스 호스팅 가능, 그러나 리소스 경합 발생 | 대규모 인덱스는 전용 엔드포인트에 격리 |
| **인덱스 재구축** | 임베딩 모델 변경 시 전체 재인덱싱 필요 | 모델 변경 전에 소규모 A/B 테스트로 품질 검증 |

```python
# 대규모 인덱스 상태 모니터링
index = vsc.get_index(
    endpoint_name="vs-endpoint-prod",
    index_name="catalog.schema.docs_index"
)

# 인덱스 상태 확인
status = index.describe()
print(f"상태: {status.get('status')}")
print(f"인덱싱된 문서 수: {status.get('num_docs')}")
print(f"마지막 동기화: {status.get('last_sync_time')}")
```

> 💡 **운영 팁**: 인덱스가 5,000만 건을 넘으면 `STORAGE_OPTIMIZED` 엔드포인트로 전환하세요. 검색 지연시간이 약간 증가(+10~30ms)하지만 비용이 40~60% 절감됩니다.

---

### 하이브리드 검색 구현

벡터 검색만으로는 정확한 키워드(제품 코드, 법률 조항 번호 등)를 놓칠 수 있습니다. 키워드 검색(BM25)과 벡터 검색을 결합한 하이브리드 검색이 프로덕션 RAG의 표준입니다.

#### 하이브리드 검색 파이프라인

```
[사용자 질문]
| 단계 | 처리 | 결과 |
|------|------|------|
| 1 | 벡터 검색 | 의미적 유사 Top-20 |
| 2 | 키워드 검색 (BM25) | 키워드 매칭 Top-20 |
| 3 | 결과 병합 (RRF) | 중복 제거, 점수 융합 → Top-20 |
| 4 | Reranker | LLM 기반 재순위화 → Top-5 |
| 5 | LLM 답변 생성 | 최종 응답 |
```

```python
# Reciprocal Rank Fusion (RRF) 구현 예시
def reciprocal_rank_fusion(vector_results, keyword_results, k=60):
    """두 검색 결과를 RRF 알고리즘으로 융합합니다."""
    scores = {}

    for rank, doc in enumerate(vector_results):
        doc_id = doc['doc_id']
        scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)

    for rank, doc in enumerate(keyword_results):
        doc_id = doc['doc_id']
        scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)

    # 점수 내림차순 정렬
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)

# 1단계: 벡터 검색
vector_results = index.similarity_search(
    query_text="산업안전보건법 제44조 위반 시 벌칙",
    columns=["doc_id", "content"],
    num_results=20
)

# 2단계: 키워드 검색 (Delta 테이블에서 SQL로)
keyword_results = spark.sql("""
    SELECT doc_id, content
    FROM catalog.schema.documents
    WHERE content LIKE '%제44조%' AND content LIKE '%벌칙%'
    LIMIT 20
""").collect()

# 3단계: RRF 융합
fused = reciprocal_rank_fusion(vector_results, keyword_results)

# 4단계: Reranker로 최종 정렬 (상위 5개 선별)
final_results = index.similarity_search(
    query_text="산업안전보건법 제44조 위반 시 벌칙",
    columns=["doc_id", "content"],
    num_results=20,
    query_options={
        "reranker": {
            "model_name": "databricks-reranker",
            "columns": ["content"],
            "top_k": 5
        }
    }
)
```

> 💡 **RRF(Reciprocal Rank Fusion)**: 서로 다른 검색 시스템의 결과를 공정하게 합치는 알고리즘입니다. 각 결과의 순위 역수를 합산하여 최종 점수를 계산합니다. k=60이 학술 연구에서 가장 성능이 좋다고 알려져 있습니다.

---

### 한국어 임베딩 전략

한국어는 교착어 특성(조사, 어미 변화)으로 인해 영어 대비 임베딩 품질이 크게 달라질 수 있습니다.

#### 모델별 한국어 성능 비교

| 모델 | 차원 | 한국어 성능 (MTEB 기준) | 다국어 지원 | 배포 용이성 | 권장 상황 |
|------|------|----------------------|-----------|-----------|----------|
| **multilingual-e5-large** | 1024 | ⭐⭐⭐⭐ (우수) | 100+ 언어 | Model Serving 배포 | 한국어 RAG 1순위 권장 |
| **BGE-M3** | 1024 | ⭐⭐⭐⭐⭐ (최우수) | 100+ 언어 | 모델 크기 큼 (2.3GB) | 최고 품질 필요 시 |
| **KoSimCSE-roberta** | 768 | ⭐⭐⭐ (양호) | 한국어 전용 | 가벼움 | 한국어 단일 언어 환경 |
| **databricks-gte-large-en** | 1024 | ⭐⭐ (보통) | 영어 중심 | 내장 (즉시 사용) | 영어 문서 위주 |
| **OpenAI text-embedding-3-large** | 3072 | ⭐⭐⭐⭐ (우수) | 다국어 | 외부 API 호출 | OpenAI 생태계 활용 시 |

#### 한국어 청킹(Chunking) 전략

한국어 문서를 청킹할 때 형태소 경계를 무시하면 검색 품질이 크게 떨어집니다.

| 청킹 방법 | 설명 | 한국어 적합도 |
|----------|------|-------------|
| **고정 길이 (Fixed-size)** | 500자/1000자 단위로 자름 | ❌ 문장 중간에서 잘릴 수 있음 |
| **문장 기반 (Sentence)** | 문장 부호(. ! ?)로 분리 | ⚠️ 한국어 문장 끝 감지가 부정확할 수 있음 |
| **문단 기반 (Paragraph)** | 빈 줄(\n\n)로 분리 | ✅ 자연스러운 의미 단위 |
| **의미 기반 (Semantic)** | 임베딩 유사도가 급변하는 지점에서 분리 | ✅ 최고 품질, 구현 복잡 |
| **재귀적 (Recursive)** | 문단 → 문장 → 단어 순으로 재귀적 분리 | ✅ LangChain 기본값, 범용 |

```python
# 한국어 최적화 청킹 예시 (재귀적 분리 + 형태소 인식)
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 한국어에 최적화된 구분자 순서
korean_splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", ".", "!", "?", "。", " ", ""],
    chunk_size=500,        # 한국어는 영어 대비 정보 밀도가 높으므로 500자 권장
    chunk_overlap=50,      # 문맥 유지를 위한 오버랩
    length_function=len,
    is_separator_regex=False
)

# 청크 생성
chunks = korean_splitter.split_text(document_text)
```

> 💡 **청킹 최적화 팁**: (1) 한국어는 영어 대비 글자당 정보 밀도가 높으므로, 영어 1000자 기준이면 한국어는 500자가 적절합니다. (2) `chunk_overlap`을 50~100자로 설정하여 청크 경계에서의 문맥 손실을 방지하세요. (3) 표, 코드 블록은 별도 청크로 분리하여 구조를 보존하세요.

---

### 비용 최적화

| 엔드포인트 유형 | CU 소비 | 검색 지연시간 | 적합한 규모 | 월 비용 (예상) |
|--------------|--------|------------|-----------|-------------|
| **STANDARD** | 높음 | 10~50ms | 100만 건 이하 | $200~800/월 |
| **STORAGE_OPTIMIZED** | 낮음 | 30~100ms | 1,000만 건 이상 | $100~400/월 |

#### 비용 절감 전략

| 전략 | 절감 효과 | 상세 |
|------|----------|------|
| **STORAGE_OPTIMIZED 전환** | 40~60% | 5,000만 건 이상이면 반드시 전환 |
| **임베딩 차원 축소** | 20~30% | 1024 → 768로 줄이면 메모리/비용 절감 (품질 소폭 저하) |
| **TRIGGERED 동기화** | 30~50% | CONTINUOUS 대비 비용 절감, 실시간성이 불필요한 경우 |
| **불필요 컬럼 제거** | 10~20% | `columns_to_sync`에서 검색에 불필요한 대용량 컬럼 제외 |
| **인덱스 통합** | 20~30% | 유사한 용도의 소규모 인덱스를 하나로 통합하고 필터로 분리 |
| **엔드포인트 공유** | 30~40% | 개발/스테이징 환경은 하나의 엔드포인트에 복수 인덱스 배치 |

```python
# 비용 효율적인 인덱스 설정 예시
index = vsc.create_delta_sync_index(
    endpoint_name="vs-endpoint-prod",
    index_name="catalog.schema.docs_index",
    source_table_name="catalog.schema.documents",
    primary_key="doc_id",
    embedding_source_column="content",
    embedding_model_endpoint_name="databricks-gte-large-en",
    pipeline_type="TRIGGERED",    # 비용 절감: 수동 트리거
    columns_to_sync=[             # 최소한의 컬럼만 동기화
        "doc_id", "content", "title", "category"
        # ❌ "full_html", "raw_pdf" 등 대용량 컬럼 제외
    ]
)
```

> 💡 **CU 사이징 공식**: `필요 CU ≈ (인덱스 문서 수 / 100만) x (임베딩 차원 / 1024) x (동시 QPS / 10)`. 예를 들어 500만 건, 1024차원, 50 QPS이면 약 25 CU가 필요합니다. 실제 운영에서는 20% 여유분을 추가하세요.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **임베딩** | 텍스트를 숫자 벡터로 변환하여 의미적 유사도를 계산합니다 |
| **Delta Sync Index** | Delta 테이블과 자동 동기화되는 인덱스입니다. 대부분의 경우 권장됩니다 |
| **Managed Embeddings** | Databricks가 임베딩 변환까지 자동 처리합니다 |
| **Reranker** | 초기 검색 결과를 LLM으로 재순위화하여 정확도를 높입니다 |

---

## 참고 링크

- [Databricks: Vector Search](https://docs.databricks.com/aws/en/generative-ai/vector-search.html)
- [Databricks: Create vector search index](https://docs.databricks.com/aws/en/generative-ai/create-query-vector-search.html)
- [Databricks: Reranker](https://docs.databricks.com/aws/en/generative-ai/vector-search-reranker.html)
