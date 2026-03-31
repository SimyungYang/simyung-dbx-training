# AI 에이전트란?

## 개념

> 💡 **AI 에이전트(AI Agent)** 란 LLM(대규모 언어 모델)을 두뇌로 사용하여, **스스로 판단하고 도구를 호출하며 작업을 수행**하는 자율적인 AI 시스템입니다. 단순한 챗봇과 달리, 에이전트는 목표를 달성하기 위해 여러 단계를 계획하고 실행합니다.

### 챗봇 vs AI 에이전트

| 비교 | 전통적 챗봇 | AI 에이전트 |
|------|-----------|------------|
| **응답 방식** | 미리 정해진 규칙/패턴 매칭 | LLM이 맥락을 이해하여 자율 응답 |
| **도구 사용** | ❌ 텍스트 응답만 가능 | ✅ DB 조회, API 호출, 파일 검색 등 도구 사용 |
| **멀티스텝** | ❌ 단일 응답 | ✅ 여러 단계를 순차적으로 계획·실행 |
| **학습** | 규칙을 수동으로 추가 | 프롬프트/컨텍스트로 행동 조정 |
| **비유** | 자동응답 안내 전화 | 문제를 직접 해결하는 전문 상담원 |

---

## 에이전트의 구성 요소

| 구성 요소 | 역할 | 상호작용 |
|-----------|------|----------|
| **사용자** | 질문/요청을 입력합니다 | AI 에이전트에 요청을 전달합니다 |
| **LLM (두뇌)** | 이해, 판단, 계획을 수행합니다 | Tools, Retriever, Memory, Guardrails와 양방향 통신합니다 |
| **Tools (도구)** | DB 조회, API 호출, 문서 검색, 코드 실행 | LLM의 지시에 따라 실행됩니다 |
| **Retriever (검색기)** | Vector Search로 관련 문서 검색 | LLM에 맥락 정보를 제공합니다 |
| **Memory (메모리)** | 대화 이력, 상태 저장 | LLM이 맥락을 유지하도록 지원합니다 |
| **Guardrails** | 안전 장치, 제약 조건 적용 | LLM의 입출력을 필터링합니다 |
| **응답** | 응답 + 수행 결과 | 에이전트가 최종 결과를 사용자에게 반환합니다 |

| 구성 요소 | 역할 | Databricks 도구 |
|-----------|------|----------------|
| **LLM (두뇌)** | 사용자 요청을 이해하고, 어떤 도구를 사용할지 판단합니다 | Foundation Model API, 외부 모델 엔드포인트 |
| **Tools (도구)** | 에이전트가 실행할 수 있는 구체적인 액션입니다 | Unity Catalog Functions, MCP Servers |
| **Retriever (검색기)** | 관련 문서/데이터를 검색하여 LLM에 맥락을 제공합니다 | Vector Search |
| **Memory (메모리)** | 대화 이력을 저장하여 맥락을 유지합니다 | 대화 메시지 배열, Lakebase |
| **Guardrails (안전 장치)** | 부적절한 응답이나 위험한 행동을 방지합니다 | 시스템 프롬프트, Expectations |

---

## 에이전트의 동작 흐름: ReAct 패턴

대부분의 AI 에이전트는 **ReAct(Reasoning + Acting)** 패턴으로 동작합니다.

> 💡 **ReAct 패턴**이란 LLM이 (1) **생각(Reasoning)**하여 다음 행동을 결정하고, (2) **행동(Acting)**하여 도구를 호출하고, (3) 결과를 **관찰(Observation)**하여 다음 생각에 반영하는 반복 루프입니다.

| 단계 | 유형 | 설명 |
|------|------|------|
| 1 | 사용자 질문 | "지난 달 매출이 가장 높은 지역은 어디인가요?" |
| 2 | 생각 (Reasoning) | "매출 데이터를 조회해야 합니다. get_sales_data 도구를 사용하겠습니다." |
| 3 | 행동 (Acting) | `get_sales_data(period='2025-02')` 호출 |
| 4 | 관찰 (Observation) | 서울 12억, 부산 8억, 대구 5억 |
| 5 | 생각 (Reasoning) | "서울이 12억으로 가장 높습니다. 답변을 생성하겠습니다." |
| 6 | 최종 응답 | "2025년 2월 매출이 가장 높은 지역은 서울(12억 원)입니다." |

### 복잡한 멀티스텝 예시

```
사용자: "VIP 고객 중 최근 3개월 미구매자에게 쿠폰을 발송해 주세요"

에이전트:
  1. [생각] VIP 고객 목록이 필요합니다 → get_customer_list(tier='VIP') 호출
  2. [관찰] VIP 고객 150명 확인
  3. [생각] 3개월 미구매자를 필터링해야 합니다 → query_orders(days=90) 호출
  4. [관찰] 42명이 3개월 내 구매 없음
  5. [생각] 쿠폰을 발송해야 합니다 → send_coupon(customers=42명, type='10%할인') 호출
  6. [관찰] 42명에게 쿠폰 발송 완료
  7. [응답] "VIP 고객 중 최근 3개월 미구매자 42명에게 10% 할인 쿠폰을 발송했습니다."
```

---

## RAG (Retrieval-Augmented Generation)

### 왜 RAG가 필요한가요?

LLM은 학습 데이터에 포함된 일반적인 지식에 대해서는 잘 답변하지만, 다음과 같은 한계가 있습니다.

| 한계 | 설명 |
|------|------|
| **지식 단절** | 학습 이후의 최신 정보를 알지 못합니다 |
| **내부 정보 부재** | 기업의 내부 문서, 정책, 데이터를 알지 못합니다 |
| **환각(Hallucination)** | 모르는 것을 그럴듯하게 지어낼 수 있습니다 |

> 💡 **RAG(Retrieval-Augmented Generation, 검색 증강 생성)** 은 LLM이 답변하기 전에, 먼저 **관련 문서를 검색(Retrieve)**하여 맥락으로 제공한 후, 그 맥락을 기반으로 **답변을 생성(Generate)**하는 패턴입니다.

| 단계 | 구성 요소 | 설명 |
|------|-----------|------|
| 1 | 사용자 질문 | "반품 정책이 어떻게 되나요?" |
| 2 | Vector Search | 관련 문서를 검색합니다 |
| 3 | 검색 결과 | 관련 문서 3건: 반품 규정 v3.2, FAQ #45, 교환 안내문 |
| 4 | LLM | 질문 + 검색된 문서를 결합하여 답변을 생성합니다 |
| 5 | 최종 응답 | "구매 후 30일 이내 무료 반품 가능합니다. (출처: 반품 규정 v3.2)" |

### RAG vs Fine-tuning 비교

| 비교 | RAG | Fine-tuning |
|------|-----|-------------|
| **데이터 업데이트** | 문서 추가만으로 즉시 반영 | 모델 재학습 필요 (시간/비용) |
| **근거 제시** | 출처 문서를 명시 가능 | 근거 제시 어려움 |
| **비용** | 검색 인프라 비용 | GPU 학습 비용 |
| **정확도** | 검색 품질에 의존 | 도메인 전문성 높음 |
| **환각 방지** | ✅ 문서 기반 답변으로 감소 | 개선되지만 완전하지 않음 |
| **적합한 경우** | 내부 문서 Q&A, 고객 지원 | 특수 도메인 언어/스타일 |

> 💡 **실무 권장**: 대부분의 기업용 AI 에이전트는 **RAG**로 시작하는 것을 권장합니다. Fine-tuning은 RAG로 충분하지 않은 특수한 경우(의료 용어, 법률 문서 등)에 추가로 고려합니다.

---

## Databricks의 에이전트 개발 도구

| 도구 | 역할 | 설명 |
|------|------|------|
| **Mosaic AI Agent Framework** | 에이전트 구축 | ChatAgent 인터페이스, Tool 연동 |
| **Vector Search** | 문서 검색 (RAG) | 임베딩 기반 유사도 검색, Reranker |
| **Foundation Model API** | LLM 제공 | Llama, DBRX, 외부 모델 프록시 |
| **Unity Catalog Functions** | Tool 정의 | SQL/Python 함수를 에이전트의 도구로 사용 |
| **MLflow Tracing** | 관찰 가능성 | 에이전트의 실행 흐름을 추적 |
| **MLflow Evaluate** | 에이전트 평가 | 자동화된 품질 평가 |
| **Model Serving** | 에이전트 배포 | REST API 엔드포인트로 배포 |
| **Review App** | 인간 피드백 | 팀원이 에이전트를 테스트하고 평가 |
| **Agent Bricks** | 사전 구축 에이전트 | Knowledge Assistant, Genie, Supervisor |

---

## 에이전트 개발 워크플로우

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 데이터 준비 | 문서 파싱, 청킹, Vector Search 인덱스 생성 |
| 2 | 에이전트 구축 | ChatAgent 구현, Tool 연결 |
| 3 | 평가 | MLflow Evaluate, Review App |
| 4 | 배포 | Model Serving 엔드포인트 |
| 5 | 모니터링 | MLflow Tracing, Inference Tables |
| ↩ | 개선 | 모니터링 결과를 바탕으로 에이전트 구축 단계로 돌아가 반복 개선 |

---

## 에이전트 아키텍처 패턴

프로덕션 에이전트를 설계할 때는 작업의 복잡도와 도메인 범위에 따라 적절한 아키텍처 패턴을 선택해야 합니다.

### Single Agent (단일 에이전트)

가장 기본적인 패턴으로, 하나의 LLM이 모든 도구를 직접 호출하며 작업을 수행합니다.

| 항목 | 설명 |
|------|------|
| **구조** | 1개 LLM + N개 도구 |
| **장점** | 구현이 단순하고, 디버깅이 쉽습니다 |
| **단점** | 도구가 많아지면(15개+) LLM의 도구 선택 정확도가 떨어집니다 |
| **적합한 경우** | 명확한 단일 도메인(예: HR 문의 봇, IT 헬프데스크) |

```python
# Databricks에서 Single Agent 구현 예시
from databricks.agents import ChatAgent, Tool

class HRAgent(ChatAgent):
    def __init__(self):
        self.tools = [
            Tool(func=lookup_employee, description="직원 정보 조회"),
            Tool(func=check_leave_balance, description="연차 잔여일 조회"),
            Tool(func=submit_leave_request, description="연차 신청"),
        ]
```

### Multi-Agent (멀티 에이전트)

여러 전문 에이전트가 각자의 도메인을 담당하고, 라우터가 사용자 요청을 적절한 에이전트로 전달합니다.

| 항목 | 설명 |
|------|------|
| **구조** | 라우터 + N개 전문 에이전트 |
| **장점** | 각 에이전트가 적은 수의 도구에 집중하여 정확도가 높습니다 |
| **단점** | 라우팅 정확도가 전체 성능을 좌우합니다. 에이전트 간 컨텍스트 공유가 복잡합니다 |
| **적합한 경우** | 여러 도메인을 커버해야 하는 통합 어시스턴트(예: 사내 포털 봇) |

```python
# 라우터 기반 Multi-Agent 패턴
agents = {
    "hr": HRAgent(),       # HR 관련 질문 전담
    "it": ITAgent(),       # IT 지원 전담
    "finance": FinanceAgent(),  # 재무 관련 전담
}

def route(user_query: str) -> str:
    # LLM이 사용자 질문을 분류하여 적절한 에이전트로 라우팅
    category = classify_intent(user_query)
    return agents[category].chat(user_query)
```

### Supervisor (감독자 패턴)

Databricks Agent Bricks에서 제공하는 패턴으로, **감독자 에이전트**가 하위 에이전트들에게 작업을 위임하고, 결과를 종합합니다.

| 항목 | 설명 |
|------|------|
| **구조** | Supervisor Agent → Worker Agent 1, 2, ... N |
| **장점** | 복잡한 멀티스텝 작업을 병렬로 처리할 수 있습니다. 감독자가 품질을 관리합니다 |
| **단점** | LLM 호출 횟수가 많아 비용과 지연시간이 증가합니다 |
| **적합한 경우** | 여러 시스템에 걸친 복합 작업(예: "마케팅 보고서 작성" → 데이터 조회 + 분석 + 시각화 + 문서 생성) |

| 패턴 비교 | Single Agent | Multi-Agent (Router) | Supervisor |
|-----------|-------------|---------------------|------------|
| **도구 수** | 1~15개 | 에이전트당 5~10개 | 에이전트당 3~8개 |
| **지연시간** | 낮음 | 중간 | 높음 |
| **비용** | 낮음 | 중간 | 높음 |
| **정확도** | 도구 수 적을 때 높음 | 라우팅 정확도에 의존 | 감독자 품질에 의존 |
| **디버깅** | 쉬움 | 중간 | 어려움 |

> 💡 **실무 권장**: 처음에는 Single Agent로 시작하고, 도구가 15개를 넘거나 도메인이 명확히 분리되면 Multi-Agent로 전환하세요. Supervisor 패턴은 복합 작업이 반드시 필요한 경우에만 도입합니다.

---

## 에이전트 메모리 관리

에이전트가 대화 맥락을 유지하고 과거 상호작용으로부터 학습하려면 **메모리 관리**가 필수적입니다.

### Short-term Memory (단기 메모리)

현재 대화 세션 내에서 맥락을 유지하는 메모리입니다.

| 전략 | 설명 | 장단점 |
|------|------|--------|
| **전체 대화 이력 전달** | 모든 메시지를 LLM에 전달합니다 | 정확하지만 토큰 비용이 급증합니다. 긴 대화에 부적합합니다 |
| **슬라이딩 윈도우** | 최근 N개 메시지만 전달합니다 | 비용을 제어하지만 초기 맥락을 잃습니다 |
| **요약 기반** | 오래된 대화를 LLM이 요약하여 압축합니다 | 비용과 맥락의 균형이 좋지만 요약 시 정보 손실이 있습니다 |
| **토큰 예산 기반** | 총 토큰 수를 제한하고 오래된 메시지를 제거합니다 | 비용을 엄격히 제어합니다 |

```python
# 슬라이딩 윈도우 + 요약 하이브리드 전략
def manage_memory(messages: list, max_recent: int = 10, max_tokens: int = 4000):
    if len(messages) <= max_recent:
        return messages

    # 오래된 메시지를 요약
    old_messages = messages[:-max_recent]
    summary = llm.summarize(old_messages)

    # 요약 + 최근 메시지를 결합
    return [{"role": "system", "content": f"이전 대화 요약: {summary}"}] + messages[-max_recent:]
```

### Long-term Memory (장기 메모리)

세션을 넘어 지속되는 메모리로, 사용자 선호도, 과거 질문 패턴, 해결된 이슈 등을 저장합니다.

| 저장소 | 적합한 데이터 | Databricks 도구 |
|--------|-------------|----------------|
| **Vector Store** | 과거 대화 중 유용한 Q&A 쌍, 해결 사례 | Databricks Vector Search |
| **Key-Value Store** | 사용자 선호도, 설정, 프로필 | Lakebase (PostgreSQL) |
| **Delta Table** | 구조화된 상호작용 로그, 피드백 이력 | Unity Catalog Delta 테이블 |

```python
# Lakebase를 활용한 장기 메모리 저장 예시
import psycopg2

def save_user_preference(user_id: str, key: str, value: str):
    """사용자 선호도를 Lakebase에 저장"""
    conn = psycopg2.connect(host="lakebase-endpoint", dbname="agent_memory")
    with conn.cursor() as cur:
        cur.execute("""
            INSERT INTO user_preferences (user_id, pref_key, pref_value, updated_at)
            VALUES (%s, %s, %s, NOW())
            ON CONFLICT (user_id, pref_key) DO UPDATE SET pref_value = %s, updated_at = NOW()
        """, (user_id, key, value, value))
    conn.commit()
```

> 💡 **메모리 비용 고려**: 모든 대화를 저장하면 스토리지와 검색 비용이 증가합니다. 실무에서는 사용자 피드백이 긍정적인 대화, 해결된 이슈, 자주 묻는 질문 등 **고품질 메모리만 선별**하여 저장하는 것이 효율적입니다.

---

## 에이전트 보안 설계

프로덕션 에이전트는 실제 시스템에 접근하므로, 보안이 매우 중요합니다. 잘못 설계하면 데이터 유출이나 의도치 않은 시스템 변경이 발생할 수 있습니다.

### 도구 접근 제어

| 보안 원칙 | 설명 | 구현 방법 |
|-----------|------|----------|
| **최소 권한 원칙** | 에이전트에 필요한 최소한의 도구만 부여합니다 | Unity Catalog Functions에서 GRANT로 도구별 권한 제어 |
| **읽기/쓰기 분리** | 조회 도구와 변경 도구를 분리합니다 | 읽기 전용 에이전트와 쓰기 가능 에이전트를 별도 엔드포인트로 분리 |
| **승인 기반 실행** | 위험한 작업(삭제, 결제 등)은 인간 승인을 필수로 합니다 | Human-in-the-loop 패턴 적용 |
| **도구 호출 제한** | 한 세션에서 도구를 호출할 수 있는 최대 횟수를 제한합니다 | 무한 루프 방지를 위해 max_iterations 설정 |

```python
# Human-in-the-loop 패턴: 위험한 도구 실행 전 승인 요청
DANGEROUS_TOOLS = {"delete_record", "process_payment", "update_user_data"}

def execute_tool(tool_name: str, params: dict, user_session):
    if tool_name in DANGEROUS_TOOLS:
        # 사용자에게 승인 요청
        approval = request_human_approval(
            user_session,
            action=f"{tool_name}({params})",
            message=f"다음 작업을 실행하시겠습니까?"
        )
        if not approval:
            return "사용자가 작업을 취소했습니다."

    return tools[tool_name].execute(**params)
```

### 데이터 유출 방지 (Data Exfiltration Prevention)

| 위험 | 시나리오 | 방어 방법 |
|------|---------|----------|
| **프롬프트 인젝션** | 악의적 입력으로 시스템 프롬프트를 우회합니다 | 입력 필터링, Guardrails, 시스템 프롬프트와 사용자 입력 분리 |
| **민감 데이터 노출** | 에이전트가 답변에 PII(개인정보)를 포함합니다 | 출력 필터링(정규식, NER 기반 마스킹) |
| **도구 체인 공격** | 에이전트가 여러 도구를 조합하여 의도하지 않은 데이터를 추출합니다 | 도구 간 데이터 흐름 모니터링, 조합 제한 |
| **간접 프롬프트 인젝션** | 검색된 문서에 악의적 명령이 포함되어 있습니다 | RAG 소스 데이터 검증, 검색 결과와 명령 분리 |

```python
# Guardrails를 활용한 입출력 필터링
import re

PII_PATTERNS = {
    "주민등록번호": r"\d{6}-[1-4]\d{6}",
    "신용카드": r"\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}",
    "전화번호": r"01[0-9]-\d{3,4}-\d{4}",
}

def sanitize_output(response: str) -> str:
    """에이전트 응답에서 PII를 마스킹"""
    for pii_type, pattern in PII_PATTERNS.items():
        response = re.sub(pattern, f"[{pii_type} 마스킹됨]", response)
    return response
```

### MLflow Tracing 기반 보안 감사

```python
# 모든 도구 호출을 MLflow로 추적하여 보안 감사 로그 생성
import mlflow

@mlflow.trace
def agent_chat(user_input: str):
    with mlflow.start_span("input_validation"):
        validated_input = validate_and_sanitize(user_input)

    with mlflow.start_span("llm_reasoning"):
        response = llm.generate(validated_input)

    with mlflow.start_span("tool_execution"):
        result = execute_tools(response.tool_calls)

    with mlflow.start_span("output_sanitization"):
        safe_output = sanitize_output(result)

    return safe_output
```

---

## 프로덕션 SLA 설계

에이전트를 프로덕션에 배포할 때는 가용성, 지연시간, 비용을 명확히 정의하고 관리해야 합니다.

### 지연시간(Latency) 최적화

| 최적화 전략 | 설명 | 예상 효과 |
|------------|------|----------|
| **LLM 선택** | 작업 복잡도에 맞는 모델 크기를 선택합니다 | 소형 모델은 2~5x 빠름 |
| **스트리밍 응답** | 토큰 단위로 실시간 스트리밍하여 체감 지연을 줄입니다 | 첫 토큰까지 시간(TTFT) 단축 |
| **도구 병렬 호출** | 독립적인 도구를 동시에 호출합니다 | 도구가 3개면 지연이 1/3로 감소 |
| **검색 결과 캐싱** | 자주 묻는 질문의 Vector Search 결과를 캐싱합니다 | 검색 지연 제거 |
| **프롬프트 최적화** | 시스템 프롬프트를 간결하게 유지합니다 | 토큰 처리 시간 감소 |

### 가용성 및 확장성

| 설계 항목 | 권장 설정 | 이유 |
|-----------|----------|------|
| **Model Serving 엔드포인트** | Provisioned Throughput 사용 | 트래픽 급증 시에도 안정적인 지연시간 보장 |
| **자동 스케일링** | min_instances=2, max_instances=10 | 고가용성 유지 + 트래픽 대응 |
| **헬스 체크** | 30초 간격으로 엔드포인트 상태 확인 | 장애 조기 발견 |
| **타임아웃 설정** | 도구 호출 30초, 전체 응답 120초 | 무한 대기 방지 |
| **재시도 전략** | 최대 3회, 지수 백오프 | 일시적 오류 복구 |

### 비용 예산 관리

에이전트 비용은 주로 LLM 토큰 소비와 Model Serving 인프라에서 발생합니다.

| 비용 요소 | 산정 방법 | 최적화 전략 |
|-----------|----------|------------|
| **LLM 토큰** | 입력 토큰 + 출력 토큰 x 단가 | 프롬프트 압축, 불필요한 컨텍스트 제거 |
| **Model Serving** | 인스턴스 수 x 시간 x DBU 단가 | Provisioned Throughput 적정 사이징 |
| **Vector Search** | 엔드포인트 + 쿼리 수 | 캐싱, 배치 쿼리 |
| **도구 호출** | SQL Warehouse 쿼리 수, API 호출 수 | 불필요한 도구 호출 최소화 |

```
# 월 비용 추정 예시 (일 1,000 대화, 평균 5턴/대화 기준)
- LLM 토큰: 1,000 x 5 x 4,000 토큰 = 20M 토큰/일 ≈ $300~600/월
- Model Serving: Provisioned Throughput 2 인스턴스 ≈ $2,000~4,000/월
- Vector Search: Endpoint 1개 ≈ $200~500/월
- 총 예상: $2,500~5,100/월
```

> 💡 **비용 모니터링**: Inference Tables에 기록되는 토큰 사용량을 SQL로 조회하여 일별/주별 비용 트렌드를 대시보드로 관리하세요. 예상치를 초과하면 즉시 알림을 받도록 설정합니다.

---

## 에이전트 디버깅 전략

에이전트는 비결정적(non-deterministic)이므로, 전통적인 소프트웨어 디버깅과 다른 접근이 필요합니다.

### MLflow Tracing 활용

MLflow Tracing은 에이전트의 실행 흐름을 단계별로 추적할 수 있는 핵심 도구입니다.

| 추적 대상 | 확인 가능한 정보 |
|-----------|-----------------|
| **LLM 호출** | 입력 프롬프트, 출력 텍스트, 토큰 수, 지연시간 |
| **도구 호출** | 호출된 도구 이름, 파라미터, 반환값, 실행 시간 |
| **검색(Retrieval)** | 검색 쿼리, 반환된 문서, 유사도 점수 |
| **전체 체인** | 각 단계의 순서, 소요 시간, 오류 발생 지점 |

### 일반적인 실패 모드와 디버깅 방법

| 실패 모드 | 증상 | 디버깅 방법 |
|-----------|------|------------|
| **도구 선택 오류** | 잘못된 도구를 호출하거나 도구를 호출하지 않습니다 | Tracing에서 LLM의 reasoning 확인 → 도구 설명(description) 개선 |
| **파라미터 추출 오류** | 올바른 도구를 선택하지만 잘못된 파라미터를 전달합니다 | 도구의 파라미터 스키마를 더 명확하게 정의 |
| **무한 루프** | 같은 도구를 반복 호출합니다 | max_iterations 설정, 루프 감지 로직 추가 |
| **환각 응답** | 도구 결과를 무시하고 지어낸 답변을 반환합니다 | 시스템 프롬프트에 "도구 결과만 사용하라"는 지시 강화 |
| **컨텍스트 유실** | 긴 대화에서 초반 맥락을 잊습니다 | 메모리 관리 전략 개선(요약 기반 압축) |

### 체계적 평가 파이프라인

```python
# MLflow Evaluate를 활용한 에이전트 품질 평가
import mlflow

eval_data = [
    {"input": "지난 달 매출 알려줘", "expected_tool": "get_sales_data", "expected_contains": "매출"},
    {"input": "서울 날씨 어때?", "expected_tool": "get_weather", "expected_contains": "온도"},
]

results = mlflow.evaluate(
    model=agent_model_uri,
    data=eval_data,
    model_type="databricks-agent",
    evaluator_config={
        "databricks-agent": {
            "metrics": ["toxicity", "groundedness", "relevance", "tool_accuracy"]
        }
    }
)

# 결과 확인
print(f"Tool 정확도: {results.metrics['tool_accuracy/mean']:.2%}")
print(f"Groundedness: {results.metrics['groundedness/mean']:.2%}")
print(f"Relevance: {results.metrics['relevance/mean']:.2%}")
```

### 프로덕션 모니터링 체크리스트

| 모니터링 항목 | 임계값 예시 | 알림 조건 |
|-------------|-----------|----------|
| **P50 응답 지연시간** | < 5초 | P50 > 8초 시 경고 |
| **P99 응답 지연시간** | < 30초 | P99 > 60초 시 경고 |
| **도구 호출 실패율** | < 1% | > 5% 시 긴급 |
| **사용자 피드백(👍비율)** | > 80% | < 60% 시 경고 |
| **일일 토큰 소비량** | 예산 이내 | 예산 120% 초과 시 경고 |
| **환각 비율** | < 5% | > 10% 시 긴급 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **AI 에이전트** | LLM을 두뇌로, 도구를 수족으로 사용하여 자율적으로 작업을 수행하는 시스템입니다 |
| **ReAct 패턴** | 생각(Reasoning) → 행동(Acting) → 관찰(Observation)의 반복 루프입니다 |
| **RAG** | 문서 검색 → LLM 답변 생성. 내부 데이터 기반 정확한 답변에 필수적입니다 |
| **Tool** | 에이전트가 호출하는 외부 기능. DB 조회, API 호출, 코드 실행 등입니다 |
| **환각(Hallucination)** | LLM이 근거 없이 그럴듯한 답변을 생성하는 현상. RAG로 완화합니다 |
| **아키텍처 패턴** | Single Agent → Multi-Agent → Supervisor 순으로 복잡도가 증가합니다 |
| **메모리 관리** | 단기(세션 내)와 장기(세션 간) 메모리를 적절히 조합해야 합니다 |
| **보안** | 최소 권한, PII 마스킹, 프롬프트 인젝션 방어가 프로덕션 필수 요소입니다 |
| **SLA 설계** | 지연시간, 가용성, 비용을 정량적으로 정의하고 모니터링해야 합니다 |

---

## 참고 링크

- [Databricks: Generative AI](https://docs.databricks.com/aws/en/generative-ai/)
- [Databricks: Mosaic AI Agent Framework](https://docs.databricks.com/aws/en/generative-ai/agent-framework/)
- [Databricks: RAG applications](https://docs.databricks.com/aws/en/generative-ai/rag.html)
- [Databricks Blog: AI Agents](https://www.databricks.com/blog)
