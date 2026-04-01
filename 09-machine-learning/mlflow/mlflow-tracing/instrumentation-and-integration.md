# 계측 방식과 프레임워크 통합

## 자동 계측 vs 수동 계측 — 상세 비교

### 자동 계측 (Autolog)의 동작 원리

Autolog는 대상 프레임워크의 핵심 메서드를 **몽키패치(Monkey Patch)**하여 호출 전후에 Span 생성/종료 코드를 주입합니다.

```python
# 내부 동작 의사 코드 (실제 구현은 더 복잡)
original_create = openai.ChatCompletion.create

def patched_create(*args, **kwargs):
    with mlflow.start_span(name="ChatCompletion", span_type="LLM") as span:
        span.set_inputs({"messages": kwargs.get("messages")})
        result = original_create(*args, **kwargs)
        span.set_outputs({"response": result.choices[0].message.content})
        span.set_attributes({
            "token_usage.input": result.usage.prompt_tokens,
            "token_usage.output": result.usage.completion_tokens,
            "model": kwargs.get("model")
        })
        return result

openai.ChatCompletion.create = patched_create
```

### 자동 vs 수동 계측 선택 가이드

| 기준 | 자동 계측 | 수동 계측 |
|------|----------|----------|
| **설정 난이도** | 한 줄 (`mlflow.xxx.autolog()`) | 각 함수에 데코레이터/컨텍스트 추가 |
| **커버리지** | 프레임워크 내부 호출만 | 비즈니스 로직 포함 전체 |
| **커스텀 속성** | 제한적 (프레임워크가 제공하는 정보만) | 자유롭게 추가 가능 |
| **오버헤드** | 매우 낮음 | 매우 낮음 |
| **권장 사용** | 빠른 프로토타이핑, 표준 파이프라인 | 프로덕션, 세밀한 제어 필요 시 |

> 💡 **실무 패턴**: 자동 계측과 수동 계측을 **함께 사용**하는 것이 가장 효과적입니다. 프레임워크 호출은 Autolog로 자동 추적하고, 비즈니스 로직(전처리, 후처리, 캐시 확인 등)은 수동으로 Span을 추가합니다.

```python
# 자동 + 수동 혼합 패턴
mlflow.langchain.autolog()  # LangChain 호출은 자동 추적

@mlflow.trace(name="rag_with_cache", span_type="CHAIN")
def rag_pipeline(question: str) -> str:
    # 수동: 캐시 확인 (자동 추적 대상 아님)
    with mlflow.start_span(name="cache_check") as span:
        cached = check_cache(question)
        span.set_attributes({"cache_hit": cached is not None})
        if cached:
            return cached

    # 자동: LangChain 호출은 autolog가 추적
    result = chain.invoke({"question": question})

    # 수동: 응답 품질 검증 (자동 추적 대상 아님)
    with mlflow.start_span(name="quality_check") as span:
        quality = validate_response(result)
        span.set_attributes({"quality_score": quality})

    return result
```

---

## LangChain/LlamaIndex 통합 상세

### LangChain 통합

LangChain의 모든 주요 컴포넌트가 자동으로 추적됩니다.

| 컴포넌트 | Span Type | 추적 내용 |
|----------|-----------|----------|
| `ChatModel` (ChatOpenAI 등) | LLM | 프롬프트, 응답, 토큰 수, 모델명, 온도 |
| `Retriever` (VectorStoreRetriever) | RETRIEVER | 쿼리, 검색 결과, 문서 수, 점수 |
| `Tool` | TOOL | 도구명, 입력, 출력, 실행 시간 |
| `Chain` (RunnableSequence) | CHAIN | 체인 구성, 각 단계 입출력 |
| `Agent` (AgentExecutor) | AGENT | 추론 과정, 도구 선택 이유, 반복 횟수 |

```python
# LangChain LCEL 파이프라인의 자동 추적
mlflow.langchain.autolog()

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (
    ChatPromptTemplate.from_template("Translate to Korean: {text}")
    | ChatOpenAI(model="gpt-4o")
    | StrOutputParser()
)

# 호출 시 자동으로 3개 Span 생성:
# 1. ChatPromptTemplate (CHAIN)
# 2. ChatOpenAI (LLM) - 토큰, 지연시간 포함
# 3. StrOutputParser (PARSER)
result = chain.invoke({"text": "Hello, world!"})
```

### LlamaIndex 통합

```python
mlflow.llama_index.autolog()

from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 인덱스 생성 시 임베딩 호출이 자동 추적됨
documents = SimpleDirectoryReader("data").load_data()
index = VectorStoreIndex.from_documents(documents)

# 쿼리 시 전체 파이프라인 자동 추적
# Retrieval → Reranking → LLM Generation
query_engine = index.as_query_engine()
response = query_engine.query("데이터 레이크하우스란 무엇인가요?")
```

---
