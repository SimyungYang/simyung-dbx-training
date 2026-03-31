# Vector Search — 벡터 유사도 검색

## 왜 Vector Search가 필요한가요?

전통적인 키워드 검색(SQL의 `LIKE` 또는 `CONTAINS`)은 **정확히 일치하는 단어**만 찾을 수 있습니다. "환불 규정"을 검색하면 "반품 정책"은 찾지 못합니다. 그러나 이 둘은 의미적으로 매우 유사합니다.

**Vector Search**는 텍스트의 **의미(Semantic)**를 이해하여, 키워드가 다르더라도 의미가 유사한 항목을 찾아줍니다. RAG(검색 증강 생성) 기반 AI 에이전트의 핵심 구성 요소입니다.

| 검색 유형 | 방식 | 장점 | 한계 |
|-----------|------|------|------|
| **키워드 검색** | 정확한 단어 매칭 | 빠르고 예측 가능 | 동의어, 의미 유사성 처리 불가 |
| **Vector Search** | 임베딩 벡터 유사도 | 의미적 유사성 검색 가능 | 임베딩 모델에 의존 |
| **하이브리드 검색** | 키워드 + 벡터 결합 | 두 방식의 장점 결합 | 구성이 복잡 |

---

## 핵심 개념: 임베딩 (Embedding)

> 💡 **임베딩(Embedding)**이란 텍스트를 수백~수천 차원의 **숫자 벡터(배열)**로 변환하는 것입니다. 의미가 유사한 텍스트는 벡터 공간에서 **가까운 위치**에 놓이게 됩니다.

```
"강아지"  → [0.23, -0.15, 0.87, 0.42, ...]  (1024차원)
"개"      → [0.25, -0.13, 0.85, 0.40, ...]  ← 가까움! (유사함)
"자동차"  → [-0.71, 0.34, -0.28, 0.55, ...] ← 멂! (다름)
```

> 💡 **코사인 유사도(Cosine Similarity)란?** 두 벡터 간의 각도를 기반으로 유사도를 측정하는 방법입니다. 값은 -1에서 1 사이이며, 1에 가까울수록 유사합니다.

---

## Databricks Vector Search 아키텍처

```mermaid
graph TB
    subgraph Prepare["1️⃣ 데이터 준비"]
        DT["Delta 테이블<br/>(원본 텍스트)"]
    end

    subgraph Index["2️⃣ 인덱스 생성"]
        EP["Vector Search Endpoint"]
        IDX["Vector Search Index"]
        EMB["임베딩 모델<br/>(자동 벡터 변환)"]
    end

    subgraph Query["3️⃣ 검색"]
        Q["질문 텍스트"]
        R["유사 문서 결과"]
    end

    DT -->|"Delta Sync"| IDX
    DT --> EMB -->|"벡터 변환"| IDX
    EP --> IDX
    Q -->|"질문 임베딩 → 유사도 검색"| IDX
    IDX --> R
```

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

가장 간편한 방식입니다. 소스 Delta 테이블의 **텍스트 컬럼**을 지정하면, Databricks가 **임베딩 변환부터 인덱스 동기화까지 모든 과정을 자동으로** 관리합니다. 사용자는 임베딩 모델을 선택하기만 하면 됩니다.

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

임베딩을 **직접 사전 계산**하여 Delta 테이블에 벡터 컬럼으로 저장한 경우 사용합니다. 커스텀 임베딩 모델(예: 한국어 전용 모델, 도메인 특화 모델)을 사용하거나, 임베딩 생성 과정을 세밀하게 제어하고 싶을 때 적합합니다.

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
```

> 💡 **Managed vs Self-Managed 선택 기준**: Databricks 내장 임베딩 모델(gte-large-en, bge-large-en)로 충분하면 **Managed**를, 커스텀 모델이 필요하면 **Self-Managed**를 사용하세요. 한국어 RAG의 경우 multilingual-e5-large 같은 다국어 모델을 Self-Managed로 사용하면 검색 품질이 향상됩니다.

---

### 3. Direct Vector Access Index

Delta 테이블과 **동기화하지 않고**, REST API를 통해 벡터를 직접 삽입/수정/삭제하는 방식입니다. 외부 시스템에서 생성된 임베딩을 실시간으로 인덱싱하거나, Delta 테이블 없이 벡터 검색만 필요한 경우에 사용합니다.

**동작 원리:**
1. 인덱스를 스키마와 함께 생성합니다
2. `upsert()` API로 벡터를 직접 삽입/업데이트합니다
3. `delete()` API로 벡터를 삭제합니다
4. Delta 테이블과는 연동되지 않으므로 데이터 관리를 직접 해야 합니다

**적합한 경우:**
- 외부 애플리케이션에서 실시간으로 벡터를 추가/삭제해야 할 때
- Delta 테이블 없이 빠른 프로토타이핑이 필요할 때
- 스트리밍 데이터를 즉시 인덱싱해야 할 때

```python
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

> ⚠️ **Direct Access의 한계**: Delta 테이블과 동기화되지 않으므로, 데이터 일관성 관리를 직접 해야 합니다. 대부분의 경우 **Delta Sync 방식을 권장**하며, Direct Access는 특수한 실시간 요구사항이 있을 때만 사용하세요.

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

> 🆕 **Reranker(GA)**: 초기 검색 결과를 LLM 기반으로 **재순위화**하여 정확도를 높이는 기능입니다.

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

```mermaid
graph LR
    Q["질문"] --> VS["Vector Search<br/>(20개 후보)"]
    VS --> RR["Reranker<br/>(5개 선별)"]
    RR --> R["최종 결과"]
```

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
