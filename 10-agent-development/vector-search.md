# Vector Search

## 개념

> 💡 **Vector Search**는 텍스트, 이미지 등을 **벡터(숫자 배열)**로 변환한 후, 의미적으로 유사한 항목을 빠르게 찾아주는 기능입니다. RAG 파이프라인에서 "질문과 의미가 비슷한 문서"를 찾는 데 핵심적으로 사용됩니다.

> 💡 **임베딩(Embedding)이란?** 텍스트를 수백~수천 차원의 숫자 벡터로 변환하는 것입니다. 의미가 비슷한 텍스트는 벡터 공간에서 가까운 위치에 놓이게 됩니다. 예를 들어, "강아지"와 "개"의 벡터는 매우 가깝고, "강아지"와 "자동차"의 벡터는 멀리 떨어져 있습니다.

---

## Vector Search 인덱스 생성

```python
from databricks.vector_search.client import VectorSearchClient

vsc = VectorSearchClient()

# 엔드포인트 생성
vsc.create_endpoint(name="vs-endpoint", endpoint_type="STANDARD")

# Delta Sync 인덱스 생성 (Delta 테이블과 자동 동기화)
vsc.create_delta_sync_index(
    endpoint_name="vs-endpoint",
    index_name="catalog.schema.docs_index",
    source_table_name="catalog.schema.documents",
    primary_key="doc_id",
    embedding_source_column="content",
    embedding_model_endpoint_name="databricks-gte-large-en",
    pipeline_type="TRIGGERED"
)
```

---

## 유사도 검색

```python
results = vsc.get_index("catalog.schema.docs_index").similarity_search(
    query_text="반품 정책이 궁금합니다",
    columns=["doc_id", "content", "source"],
    num_results=3
)
```

> 🆕 **Vector Search Reranker (GA)**: 검색 결과의 품질을 향상시키는 리랭커(Reranker) 기능이 GA되었습니다. 초기 검색 결과를 더 정교하게 재순위화하여 관련성이 높은 문서를 상위에 배치합니다.

---

## 참고 링크

- [Databricks: Vector Search](https://docs.databricks.com/aws/en/generative-ai/vector-search.html)
