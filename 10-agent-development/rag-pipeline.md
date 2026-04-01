# RAG 파이프라인 — 검색 증강 생성

## RAG 파이프라인의 전체 구조

RAG 파이프라인은 크게 **오프라인(데이터 준비)**과 ** 온라인(질의 처리)** 두 단계로 나뉩니다.

![RAG 파이프라인 구성 요소 다이어그램](https://docs.databricks.com/aws/en/assets/images/rag-components-f2022fe2c47439e692d1ccc6c14a3238.svg)

*출처: [Databricks 공식 문서](https://docs.databricks.com/aws/en/generative-ai/retrieval-augmented-generation.html)*
**오프라인: 데이터 준비**

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 원본 문서 수집 | PDF, 웹, DB에서 문서를 수집합니다 |
| 2 | 파싱 | `ai_parse_document`으로 텍스트를 추출합니다 |
| 3 | 청킹 | 적절한 크기로 문서를 분할합니다 |
| 4 | Delta 테이블 저장 | doc_id, content, metadata를 저장합니다 |
| 5 | 벡터 인덱싱 | Vector Search Index를 생성합니다 |

**온라인: 질의 처리**

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 사용자 질문 | 사용자가 질문을 입력합니다 |
| 2 | 검색 | Vector Search로 관련 문서를 검색합니다 |
| 3 | 재순위화 | Reranker로 검색 결과를 정렬합니다 |
| 4 | 프롬프트 구성 | 질문 + 검색 결과를 결합합니다 |
| 5 | LLM 호출 | 답변을 생성합니다 |
| 6 | 최종 응답 | 답변 + 출처를 사용자에게 전달합니다 |

---

## Step 1: 문서 수집 및 파싱

### 문서 소스

| 소스 | 방법 |
|------|------|
| **PDF 문서**| `ai_parse_document()` 또는 Python 라이브러리(PyMuPDF, pdfplumber) |
| ** 웹 페이지**| Python requests + BeautifulSoup / Scrapy |
| **Confluence/SharePoint**| 커넥터 또는 API 호출 |
| ** 데이터베이스**| SQL 쿼리로 텍스트 필드 추출 |
| **Google Drive**| Google Drive API |

### Databricks에서 PDF 파싱

```sql
-- ai_parse_document으로 PDF 텍스트 추출
CREATE OR REFRESH MATERIALIZED VIEW parsed_documents AS
SELECT
    file_name,
    ai_parse_document(
        CONCAT('/Volumes/catalog/schema/raw_docs/', file_name),
        'markdown'  -- 구조(제목, 표 등)를 유지하는 markdown 모드
    ) AS content,
    current_timestamp() AS parsed_at
FROM (SELECT file_name FROM LIST('/Volumes/catalog/schema/raw_docs/') WHERE file_name LIKE '%.pdf');
```

```python
# Python으로 PDF 파싱 (더 세밀한 제어)
import fitz  # PyMuPDF

def parse_pdf(path: str) -> str:
    doc = fitz.open(path)
    text = ""
    for page in doc:
        text += page.get_text("text") + "\n\n"
    return text
```

---

## Step 2: 청킹 (Chunking)

> 💡 ** 청킹(Chunking)**이란 긴 문서를 LLM이 처리할 수 있는 ** 적절한 크기의 조각**으로 나누는 것입니다. RAG의 품질에 가장 큰 영향을 미치는 단계 중 하나입니다.

### 청킹 전략

| 전략 | 설명 | 적합한 문서 |
|------|------|-----------|
| **고정 크기** | 일정한 토큰/문자 수로 분할합니다 | 균일한 구조의 문서 |
| **재귀적 분할** | 문단 → 문장 → 단어 순서로 자연스러운 경계에서 분할합니다 | 대부분의 문서 (권장) |
| **의미 기반** | 임베딩 유사도를 기반으로 의미 단위로 분할합니다 | 주제가 자주 바뀌는 문서 |
| **구조 기반** | Markdown 제목, HTML 태그 등 문서 구조를 활용합니다 | 구조화된 문서 |

### 최적의 청크 크기

| 청크 크기 | 장점 | 단점 |
|-----------|------|------|
| **작음 (200~500 토큰)** | 검색 정밀도가 높음 | 맥락이 부족할 수 있음 |
| **중간 (500~1000 토큰)** | 정밀도와 맥락의 균형 | 대부분의 경우 권장 |
| **큼 (1000~2000 토큰)** | 충분한 맥락 제공 | 검색 정밀도 저하 |

### 청킹 구현

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 재귀적 분할 (가장 일반적)
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,         # 최대 문자 수
    chunk_overlap=200,       # 청크 간 겹침 (맥락 연속성 유지)
    separators=["\n\n", "\n", ". ", " ", ""],  # 분할 우선순위
    length_function=len
)

chunks = splitter.split_text(document_text)

# 결과를 Delta 테이블로 저장
from pyspark.sql import Row

chunk_rows = [
    Row(
        doc_id=f"{doc_id}_chunk_{i}",
        parent_doc_id=doc_id,
        chunk_index=i,
        content=chunk,
        title=doc_title,
        source=doc_source
    )
    for i, chunk in enumerate(chunks)
]

chunk_df = spark.createDataFrame(chunk_rows)
chunk_df.write.format("delta").mode("append").saveAsTable("catalog.schema.document_chunks")
```

> 💡 **chunk_overlap**: 청크 간에 일부 텍스트를 겹치게 하면, 문장이 잘리는 것을 방지하고 맥락의 연속성을 유지할 수 있습니다. 일반적으로 chunk_size의 10~20% 정도를 권장합니다.

---

## Step 3: 벡터 인덱스 생성

```python
from databricks.vector_search.client import VectorSearchClient

vsc = VectorSearchClient()

# Managed Embeddings 인덱스 생성 (권장)
vsc.create_delta_sync_index(
    endpoint_name="vs-endpoint",
    index_name="catalog.schema.docs_index",
    source_table_name="catalog.schema.document_chunks",
    primary_key="doc_id",
    embedding_source_column="content",
    embedding_model_endpoint_name="databricks-gte-large-en",
    pipeline_type="TRIGGERED",
    columns_to_sync=["doc_id", "parent_doc_id", "content", "title", "source", "chunk_index"]
)

# 인덱스 동기화 실행
vsc.get_index("vs-endpoint", "catalog.schema.docs_index").sync()
```

---

## Step 4: 검색 및 답변 생성

```python
import mlflow

@mlflow.trace
def rag_answer(question: str) -> dict:
    """RAG 파이프라인: 질문 → 검색 → 답변"""

    # 1. 관련 문서 검색
    docs = search_documents(question)

    # 2. 프롬프트 구성
    context = format_context(docs)
    prompt = build_prompt(question, context)

    # 3. LLM 호출
    answer = call_llm(prompt)

    # 4. 출처 정보 포함
    sources = [{"title": d["title"], "source": d["source"]} for d in docs]

    return {"answer": answer, "sources": sources}


@mlflow.trace(span_type="RETRIEVER")
def search_documents(query: str, num_results: int = 5) -> list:
    """Vector Search + Reranker로 관련 문서 검색"""
    results = index.similarity_search(
        query_text=query,
        columns=["doc_id", "content", "title", "source"],
        num_results=20,  # 초기 검색은 넓게
        query_options={
            "reranker": {
                "model_name": "databricks-reranker",
                "columns": ["content"],
                "top_k": num_results  # Reranker가 최종 선별
            }
        }
    )
    return [
        {"doc_id": r[0], "content": r[1], "title": r[2], "source": r[3]}
        for r in results["result"]["data_array"]
    ]


def format_context(docs: list) -> str:
    """검색된 문서를 LLM에 전달할 컨텍스트로 포맷팅"""
    return "\n\n---\n\n".join([
        f"[출처: {d['title']}]\n{d['content']}"
        for d in docs
    ])


def build_prompt(question: str, context: str) -> str:
    """시스템 프롬프트 + 컨텍스트 + 질문 조합"""
    return f"""당신은 정확하고 도움이 되는 AI 어시스턴트입니다.
아래 제공된 문서만을 근거로 답변해 주세요.
문서에 없는 내용은 "제공된 문서에서 해당 정보를 찾을 수 없습니다"라고 답변해 주세요.
답변 마지막에 참고한 문서의 출처를 명시해 주세요.

### 참고 문서:
{context}

### 질문:
{question}

### 답변:"""


@mlflow.trace(span_type="LLM")
def call_llm(prompt: str) -> str:
    """Foundation Model API로 LLM 호출"""
    response = client.predict(
        endpoint="databricks-meta-llama-3-3-70b-instruct",
        inputs={
            "messages": [{"role": "user", "content": prompt}],
            "temperature": 0.1,
            "max_tokens": 1000
        }
    )
    return response["choices"][0]["message"]["content"]
```

---

## RAG 품질 개선 전략

| 전략 | 설명 | 효과 |
|------|------|------|
| **Reranker 사용**| 초기 검색 후 LLM으로 재순위화 | 검색 정확도 15~30% 향상 |
| ** 청크 크기 튜닝**| 문서 유형에 맞는 최적 크기 탐색 | 답변 품질 개선 |
| ** 하이브리드 검색**| 키워드 + 벡터 검색 결합 | 고유명사, 코드 등에서 효과적 |
| ** 메타데이터 필터**| 카테고리, 날짜 등으로 검색 범위 제한 | 관련성 높은 결과 |
| ** 다국어 임베딩**| 한국어 전용 임베딩 모델 사용 | 한국어 검색 품질 향상 |
| ** 질문 재작성**| LLM으로 질문을 검색에 적합하게 재작성 | 모호한 질문 처리 개선 |
| ** 문서 정기 갱신**| Delta Sync로 최신 문서 자동 반영 | 답변의 최신성 유지 |

---

## 실전 인사이트: RAG 파이프라인의 90%는 청킹에 달려 있습니다

20년간 데이터 파이프라인을 설계해오면서, RAG만큼 "** 데이터 전처리가 결과 품질을 좌우하는**" 시스템은 처음이었습니다. LLM이 아무리 좋아도, 검색되는 청크가 엉망이면 답변도 엉망입니다.

### 청킹 실험에서 배운 것들

실제 프로덕션 RAG를 구축하면서 청킹 전략별로 A/B 테스트를 진행한 경험을 공유합니다.

| 실험 | 설정 | 결과 (정확 답변율) | 교훈 |
|------|------|-------------------|------|
| **고정 500자** | 오버랩 0 | 42% | 문장이 잘리면 의미가 파괴됩니다 |
| **고정 500자** | 오버랩 100자 | 58% | 오버랩만으로 16% 향상. 반드시 넣으세요 |
| **재귀적 1000자** | 오버랩 200자 | 71% | 문단 경계를 존중하니 크게 향상되었습니다 |
| **구조 기반 (제목별)** | Markdown 헤더 분리 | 78% | 문서 구조가 명확한 경우 최고 성능 |
| **구조 + 메타데이터** | 제목+날짜 포함 | 83% | 청크에 "이 문서가 뭔지" 맥락을 넣으니 검색 품질이 급상승 |

> 💡 **핵심 교훈**: 청크에 **제목, 문서명, 날짜** 같은 메타데이터를 함께 포함시키세요. LLM이 "이 청크가 어떤 맥락의 내용인지" 이해하는 데 결정적인 차이를 만듭니다. 청크 본문 앞에 `[제목: XXX | 작성일: 2025-01-15]`를 붙이는 것만으로도 정확도가 5~10% 올라갑니다.

### 청크 경계 문제 — 가장 흔한 실패 원인

```
원본 문서:
"반품 정책: 구매 후 30일 이내 반품 가능합니다.
단, 전자제품은 14일 이내이며, 개봉 시 반품이 불가합니다."

--- 청크 경계에서 잘린 경우 ---
청크 1: "반품 정책: 구매 후 30일 이내 반품 가능합니다."
청크 2: "단, 전자제품은 14일 이내이며, 개봉 시 반품이 불가합니다."

→ 사용자 질문: "전자제품 반품 기한이 어떻게 되나요?"
→ 검색 결과: 청크 2만 검색됨
→ LLM 답변: "14일 이내입니다" (맞지만 "30일"이라는 일반 정책 맥락이 빠짐)
```

이런 문제를 해결하려면 **오버랩을 충분히 설정**하고, 가능하면 **문서의 논리적 단위(섹션, 조항)를 존중하는 청킹**을 사용해야 합니다.

---

## 실전 인사이트: 임베딩 모델과 한국어 품질

### 한국어 임베딩 모델 선택 시 주의사항

임베딩 모델 선택에서 한국어 품질 차이는 극심합니다. 영어로는 잘 되는 모델이 한국어에서는 처참한 성능을 보이는 경우가 많습니다.

| 모델 | 영어 검색 품질 | 한국어 검색 품질 | 비고 |
|------|--------------|----------------|------|
| `databricks-gte-large-en` | 우수 | 보통~낮음 | 영어 전용. 한국어 토크나이저 없음 |
| `databricks-bge-large-en` | 우수 | 낮음 | 영어 최적화 모델 |
| 다국어 모델 (multilingual-e5-large) | 좋음 | 좋음 | 한국어 포함 다국어 지원 |
| 한국어 특화 모델 (KoSimCSE 등) | 보통 | 우수 | 한국어 전용이지만 External Model로 배포 필요 |

> ⚠️ **실전 팁**: 한국어 문서 RAG를 구축한다면, 반드시 **한국어 테스트 쿼리 세트(최소 50개)**를 만들어서 임베딩 모델별 검색 품질을 비교하세요. "databricks-gte-large-en"은 영어에서는 탁월하지만, 한국어에서는 의미적 유사도가 떨어져서 "환불 정책"을 검색했는데 "배송 정보"가 나오는 경우가 있었습니다.

### 한국어 RAG에서의 하이브리드 검색 필수성

한국어는 **조사(은/는/이/가)**와 **어미 변환**이 다양해서, 순수 벡터 검색만으로는 한계가 있습니다. 예를 들어 "클러스터 생성 방법"과 "클러스터를 어떻게 만드나요"는 의미가 같지만, 임베딩 공간에서 거리가 있을 수 있습니다. **키워드 검색(BM25)과 벡터 검색을 결합한 하이브리드 검색**이 한국어 RAG에서는 거의 필수입니다.

---

## 프로덕션 RAG에서 가장 흔한 실패 패턴

### 패턴 1: Hallucination (환각)

프롬프트에 "문서에 없으면 모른다고 답하라"고 써도, LLM은 종종 자체 학습 데이터로 답변을 "지어냅니다". 특히 위험한 상황은 다음과 같습니다.

| 상황 | 위험도 | 대응 |
|------|--------|------|
| 검색 결과가 0건인데 LLM이 답변 생성 | 매우 높음 | 검색 결과 0건이면 LLM 호출 자체를 차단하세요 |
| 검색 결과와 무관한 답변 생성 | 높음 | 답변과 검색 청크 간 **faithfulness 점수**를 측정하세요 |
| 부분적으로만 맞는 답변 | 중간 | 출처 표시를 강제하고, 사용자가 원문을 확인할 수 있게 하세요 |

> 💡 **실전 방어책**: `temperature=0.0`~`0.1`로 설정하고, 시스템 프롬프트에 "**반드시 아래 문서에서 근거를 찾아 인용하며 답변하세요. 근거를 찾을 수 없으면 '해당 정보를 찾을 수 없습니다'라고 답하세요**"를 명시하세요. 그래도 100% 방지는 불가능하므로, Agent Evaluation의 faithfulness 메트릭으로 지속 모니터링해야 합니다.

### 패턴 2: 오래된 문서 문제

RAG에서 가장 교묘한 실패입니다. 사용자가 "현재 반품 정책이 뭔가요?"라고 물었는데, 2년 전 정책 문서가 검색되어 잘못된 답변을 하는 경우입니다.

```python
# 해결 방법: 메타데이터 필터링 + 시간 가중치
results = index.similarity_search(
    query_text=query,
    columns=["doc_id", "content", "title", "source", "updated_at"],
    num_results=20,
    filters={
        "updated_at >=": "2024-01-01"  # 최근 문서만 검색
    }
)
```

> ⚠️ **프로덕션 필수 사항**: 문서에 `updated_at`, `version`, `status(active/archived)` 메타데이터를 반드시 포함시키세요. `status=archived`인 문서는 인덱스에서 아예 제외하거나, 검색 시 필터링해야 합니다. Delta Sync의 `TRIGGERED` 파이프라인을 사용하면, 원본 Delta 테이블에서 문서를 삭제하거나 상태를 변경했을 때 인덱스에 자동 반영됩니다.

### 패턴 3: 너무 많은 컨텍스트 전달

검색 결과를 20개씩 LLM에 전달하면, 오히려 답변 품질이 떨어집니다. LLM이 관련 없는 청크에 "혼란"을 겪기 때문입니다. 실험 결과, **3~5개의 고품질 청크가 10~20개의 저품질 청크보다 항상 좋은 결과**를 보였습니다. Reranker를 사용하여 초기 검색은 넓게(20개), 최종 전달은 좁게(3~5개) 하는 것이 최선의 전략입니다.

### 패턴 4: 평가 없는 배포

RAG 파이프라인을 만들고 "잘 되는 것 같다"는 느낌으로 프로덕션에 배포하는 것은 위험합니다. 반드시 **골든 데이터셋(질문-정답 쌍 최소 100개)**을 만들어서 정량적으로 평가해야 합니다. Databricks의 Agent Evaluation을 활용하면 retrieval precision, answer correctness, faithfulness를 자동으로 측정할 수 있습니다.

```python
import mlflow

# 골든 데이터셋으로 평가
eval_results = mlflow.evaluate(
    model=rag_chain,
    data=golden_dataset,  # 질문 + 기대 답변 + 기대 검색 문서
    model_type="databricks-agent"
)

# 핵심 메트릭 확인
print(f"Retrieval Precision: {eval_results.metrics['retrieval/precision']}")
print(f"Answer Correctness: {eval_results.metrics['answer_correctness/llm_judged/average']}")
print(f"Faithfulness: {eval_results.metrics['faithfulness/llm_judged/average']}")
```

---

## 정리

| 단계 | 핵심 작업 | Databricks 도구 |
|------|----------|----------------|
| **문서 파싱**| PDF/웹 → 텍스트 | ai_parse_document, Python 라이브러리 |
| ** 청킹**| 적절한 크기로 분할 | RecursiveCharacterTextSplitter |
| ** 인덱싱**| 임베딩 + 벡터 인덱스 | Vector Search (Managed Embeddings) |
| ** 검색**| 유사 문서 검색 + 재순위화 | Vector Search + Reranker |
| ** 생성**| LLM으로 답변 생성 | Foundation Model API |
| ** 추적** | 실행 흐름 모니터링 | MLflow Tracing |

---

## 참고 링크

- [Databricks: Build RAG applications](https://docs.databricks.com/aws/en/generative-ai/rag.html)
- [Databricks: Document parsing](https://docs.databricks.com/aws/en/generative-ai/rag-document-parsing.html)
- [Databricks: Vector Search](https://docs.databricks.com/aws/en/generative-ai/vector-search.html)
- [LangChain: Text Splitters](https://python.langchain.com/docs/modules/data_connection/document_transformers/)
