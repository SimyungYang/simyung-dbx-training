# Review App 상세

## Review App이란?

**Review App** 은 배포된 AI 에이전트를 **웹 기반 채팅 UI** 로 테스트하고, 테스터들이 각 응답에 대해 **피드백을 남길 수 있는** 도구입니다. 에이전트를 프로덕션에 배포하기 전에 품질을 검증하고, 수집된 피드백을 **평가 데이터셋으로 변환** 하여 에이전트를 지속적으로 개선할 수 있습니다.

> 💡 **비유**: Review App은 "소프트웨어 베타 테스트 프로그램"과 같습니다. 제한된 인원이 실제로 사용해 보고, 버그나 개선 사항을 보고하는 과정입니다.

---

## agents.deploy()로 배포

Review App은 `agents.deploy()`로 에이전트를 배포할 때 **자동으로 생성** 됩니다.

```python
from databricks import agents

# 에이전트 배포 — Review App이 자동 생성됩니다
deployment = agents.deploy(
    model_name="catalog.schema.customer_support_agent",
    model_version=3
)

# 배포 결과 확인
print(f"Endpoint: {deployment.endpoint_name}")
print(f"Review App URL: {deployment.review_app_url}")
```

### 배포 시 자동 생성되는 리소스

| 리소스 | 설명 |
|--------|------|
| **Model Serving Endpoint** | 에이전트를 REST API로 호스팅합니다 |
| **Review App** | 웹 기반 채팅 UI로 테스트할 수 있습니다 |
| **Inference Tables** | 모든 요청/응답을 Delta Table에 자동 기록합니다 |
| **Feedback Tables** | 테스터 피드백이 별도 테이블에 저장됩니다 |

---

## Review App 설정

### 테스터 추가

```python
from databricks import agents

# 특정 사용자에게 Review App 접근 권한 부여
agents.set_permissions(
    model_name="catalog.schema.customer_support_agent",
    users=[
        "tester1@company.com",
        "tester2@company.com",
        "product-manager@company.com"
    ],
    permission="CAN_QUERY"
)
```

### 테스트 안내문 설정

테스터에게 어떤 시나리오를 테스트해야 하는지 안내합니다.

```python
agents.set_review_instructions(
    model_name="catalog.schema.customer_support_agent",
    instructions="""
    ## 테스트 안내

    다음 시나리오를 테스트해 주세요:

    ### 기본 기능
    1. "최근 주문 상태를 알려주세요" (정상 주문 조회)
    2. "주문번호 99999 상태" (존재하지 않는 주문)
    3. "반품 절차가 어떻게 되나요?" (FAQ 질문)

    ### 경계 사례
    4. 한국어/영어 혼용 질문
    5. 매우 긴 질문 (500자 이상)
    6. 답변 범위 밖 질문 (예: "오늘 날씨 어때요?")

    ### 안전성
    7. 개인정보 요청 ("고객 전화번호 알려줘")
    8. 부적절한 요청

    ### 피드백 방법
    - 각 응답에 👍 또는 👎를 클릭하세요
    - 문제가 있는 응답에는 반드시 코멘트를 남겨주세요
    - "Expected response"에 올바른 답변을 입력하면 평가 데이터로 활용됩니다
    """
)
```

---

## 피드백 수집

### 피드백 유형

Review App에서 테스터는 다음과 같은 피드백을 남길 수 있습니다.

| 피드백 유형 | 설명 | 활용 |
|-----------|------|------|
| **Thumbs Up/Down** | 응답의 전반적 품질 평가입니다 | 에이전트 성능 추이 모니터링 |
| **코멘트** | 문제점이나 개선 사항을 텍스트로 기술합니다 | 구체적인 문제 파악 |
| **Expected Response** | 올바른 답변을 직접 입력합니다 | 평가 데이터셋 구축에 활용 |
| **Retrieval Quality** | 검색된 문서의 관련성을 평가합니다 | RAG 파이프라인 개선 |

### 피드백 데이터 조회

피드백은 Unity Catalog의 테이블에 자동 저장됩니다.

```sql
-- 피드백 테이블 조회
SELECT
    request_id,
    timestamp,
    user_input,
    agent_response,
    feedback_rating,    -- 'positive' 또는 'negative'
    feedback_comment,
    expected_response,
    reviewer_email
FROM catalog.schema.agent_feedback
ORDER BY timestamp DESC;
```

### 피드백 분석

```sql
-- 일별 피드백 점수 추이
SELECT
    DATE(timestamp) AS date,
    COUNT(*) AS total_feedback,
    COUNT(CASE WHEN feedback_rating = 'positive' THEN 1 END) AS positive,
    COUNT(CASE WHEN feedback_rating = 'negative' THEN 1 END) AS negative,
    ROUND(
        COUNT(CASE WHEN feedback_rating = 'positive' THEN 1 END) * 100.0 / COUNT(*), 1
    ) AS satisfaction_rate_pct
FROM catalog.schema.agent_feedback
GROUP BY DATE(timestamp)
ORDER BY date DESC;

-- 부정 피드백이 많은 질문 패턴 분석
SELECT
    user_input,
    agent_response,
    feedback_comment,
    expected_response
FROM catalog.schema.agent_feedback
WHERE feedback_rating = 'negative'
ORDER BY timestamp DESC
LIMIT 50;
```

---

## 평가 데이터셋 자동 생성

Review App의 피드백을 **Agent Evaluation의 평가 데이터셋** 으로 변환할 수 있습니다. 이를 통해 에이전트 개선 사이클을 구축합니다.

### 피드백에서 평가 데이터셋 생성

```python
import pandas as pd

# 피드백 데이터를 평가 데이터셋 형식으로 변환
feedback_df = spark.sql("""
    SELECT
        user_input AS request,
        expected_response AS expected_response,
        agent_response AS response
    FROM catalog.schema.agent_feedback
    WHERE expected_response IS NOT NULL
        AND feedback_rating = 'negative'
""").toPandas()

# 평가 데이터셋으로 저장
eval_dataset = feedback_df.rename(columns={
    "request": "request",
    "expected_response": "expected_response",
    "response": "response"
})

# Delta 테이블로 저장
spark.createDataFrame(eval_dataset).write.mode("overwrite").saveAsTable(
    "catalog.schema.eval_dataset"
)
```

### 에이전트 개선 사이클

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 에이전트 배포 | `agents.deploy()`로 에이전트를 배포합니다 |
| 2 | 피드백 수집 | Review App에서 테스터가 피드백을 남깁니다 |
| 3 | 부정 피드백 분석 | 낮은 평가를 받은 응답 패턴을 분석합니다 |
| 4 | 평가 데이터셋 업데이트 | 피드백을 평가 데이터셋에 반영합니다 |
| 5 | 에이전트 개선 | 프롬프트, 도구, RAG를 수정합니다 |
| 6 | 성능 측정 | Agent Evaluation으로 성능을 측정합니다 |
| 7 | 품질 기준 충족 여부 | 미충족 시 5단계로 돌아갑니다 |
| 8 | 새 버전 배포 | 품질 기준 충족 시 새 버전을 배포하고, 2단계로 돌아가 반복합니다 |

---

## MLflow Evaluate와의 연동

수집된 평가 데이터셋을 `mlflow.evaluate()`에 활용하여 에이전트의 성능을 정량적으로 측정합니다.

```python
import mlflow

# 평가 데이터셋 로드
eval_data = spark.table("catalog.schema.eval_dataset").toPandas()

# 에이전트 평가 실행
results = mlflow.evaluate(
    model="models:/catalog.schema.customer_support_agent/3",
    data=eval_data,
    model_type="databricks-agent",
    evaluator_config={
        "databricks-agent": {
            "metrics": [
                "answer_correctness",     # 정답과의 일치도
                "groundedness",           # 근거 기반 답변 여부
                "relevance",              # 질문과의 관련성
                "safety"                  # 안전성
            ]
        }
    }
)

# 결과 확인
print(f"Answer Correctness: {results.metrics['answer_correctness/mean']:.2f}")
print(f"Groundedness: {results.metrics['groundedness/mean']:.2f}")
print(f"Relevance: {results.metrics['relevance/mean']:.2f}")
print(f"Safety: {results.metrics['safety/mean']:.2f}")

# 개별 평가 결과
results.tables["eval_results_table"].display()
```

---

## 현업 사례: Review App 피드백으로 정확도를 60%에서 85%로 올린 과정

> 🔥 **실전 경험담**
>
> 한 금융 고객사에서 "내부 규정 검색 에이전트"를 구축했습니다. 첫 배포 후 Review App에서 테스터 8명이 2주간 테스트한 결과, **만족도(Thumbs Up 비율)가 60%에 불과** 했습니다. 주요 불만 사항은 다음과 같았습니다:
>
> 1. **검색은 되지만 답변이 너무 일반적**(35% 불만) — "규정 3.2.1항에 따르면..."이 아니라 "관련 규정이 있습니다"식의 답변
> 2. **관련 없는 문서를 인용**(25% 불만) — "대출 규정"을 물었는데 "예금 규정"을 인용
> 3. **최신 개정 내용 미반영**(20% 불만) — 2024년 개정된 내용이 아닌 2023년 버전으로 답변
> 4. **한국어 답변 품질**(20% 불만) — 영어 번역투 문장이 많음
>
> **개선 과정 (4주간 3라운드 반복)**:
>
> | 라운드 | 조치 | 만족도 변화 |
> |--------|------|:-----------:|
> | **1라운드** | 시스템 프롬프트에 "반드시 조항 번호를 인용하라" 추가, 검색 문서 수 3→7개로 증가 | 60% → 70% |
> | **2라운드** | 문서를 조항 단위로 재분할(Chunking 전략 변경), 구버전 문서 제거 | 70% → 78% |
> | **3라운드** | 부정 피드백에서 Expected Response를 수집하여 Few-shot 예시로 활용, LLM을 GPT-4o로 교체 | 78% → 85% |
>
> **핵심 교훈**: 에이전트 품질은 한 번에 올라가지 않습니다. **Review App → 피드백 분석 → 개선 → 재배포의 반복 사이클** 이 핵심이며, 보통 3~5라운드를 거쳐야 프로덕션 수준(80%+ 만족도)에 도달합니다.

---

## 효과적인 테스터 선정 방법

현업에서 Review App의 성공은 **누가 테스트하느냐** 에 크게 좌우됩니다.

### 이상적인 테스터 구성

| 역할 | 인원 | 선정 이유 | 기대 기여 |
|------|:----:|----------|----------|
| **실제 사용자 (현업 담당자)** | 3~5명 | 실제 업무에서 묻는 질문을 테스트합니다 | 현실적인 질문 패턴 확보 |
| **도메인 전문가** | 1~2명 | 답변의 정확성을 검증합니다 | Expected Response 작성 |
| **QA/테스트 전문가** | 1명 | 엣지 케이스, 보안 시나리오를 테스트합니다 | 경계 사례 발견 |
| **비전문가 (신입사원 등)** | 1~2명 | 초보자 관점에서 이해도를 평가합니다 | UX 개선점 발견 |

> 🔥 **이것을 안 하면**: 개발자만으로 테스터를 구성하면 **"개발자가 기대하는 질문"만 테스트** 하게 됩니다. 실제 사용자는 "그 규정 있잖아, 그거 뭐였더라?"처럼 모호한 질문을 하는데, 개발자는 이런 패턴을 테스트하지 않습니다. **실제 사용자를 반드시 테스터에 포함하세요.**

### 테스터 온보딩 체크리스트

```markdown
## Review App 테스터 가이드

### 시작 전 확인사항
- [ ] Review App URL에 접속되는지 확인
- [ ] 테스트 시나리오 문서를 읽었는지 확인

### 테스트 방법
1. **자연스럽게 질문하세요**— 실제 업무에서 하는 것처럼 질문합니다
2. **모든 답변에 피드백을 남기세요**— 👍 또는 👎를 반드시 클릭합니다
3. **👎를 클릭할 때는 반드시 이유를 적어주세요**— "틀림", "불충분" 등 한 줄이라도
4. **Expected Response를 적극 작성하세요**— "이렇게 답변했어야 해"를 적으면 평가 데이터로 활용됩니다
5. **다양한 표현으로 같은 질문을 해보세요**— "연차 규정", "휴가 정책", "연차 쓰려면?" 등

### 테스트 목표 (1인당)
- 최소 30개 질문
- 부정 피드백 시 100% 코멘트 작성
- Expected Response 10개 이상 작성
```

---

## 피드백 분석 프로세스

수집된 피드백을 효과적으로 분석하려면 체계적인 프로세스가 필요합니다.

### Step 1: 부정 피드백 분류

```sql
-- 부정 피드백을 카테고리별로 분류
SELECT
    CASE
        WHEN feedback_comment LIKE '%틀%' OR feedback_comment LIKE '%오류%' OR feedback_comment LIKE '%잘못%'
            THEN '오답'
        WHEN feedback_comment LIKE '%부족%' OR feedback_comment LIKE '%부실%' OR feedback_comment LIKE '%짧%'
            THEN '불충분한 답변'
        WHEN feedback_comment LIKE '%관련%없%' OR feedback_comment LIKE '%엉뚱%'
            THEN '관련 없는 답변'
        WHEN feedback_comment LIKE '%느%' OR feedback_comment LIKE '%오래%'
            THEN '응답 속도'
        ELSE '기타'
    END AS issue_category,
    COUNT(*) AS count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct
FROM catalog.schema.agent_feedback
WHERE feedback_rating = 'negative'
GROUP BY 1
ORDER BY count DESC;
```

### Step 2: 우선순위 결정

| 우선순위 | 기준 | 조치 |
|:--------:|------|------|
| **P0 (즉시)** | 오답 + 빈도 높음 (5건 이상) | 프롬프트 수정, 문서 보강 |
| **P1 (1주 내)** | 불충분한 답변 + Expected Response 있음 | 검색 설정 조정, Chunking 개선 |
| **P2 (2주 내)** | 관련 없는 답변 | Vector Search 임계값 조정 |
| **P3 (백로그)** | 기타 개선 요청 | 다음 스프린트에 반영 |

### Step 3: 개선 효과 측정

```python
# 버전별 만족도 비교
version_comparison = spark.sql("""
    SELECT
        CASE
            WHEN timestamp < '2025-03-15' THEN 'v1 (개선 전)'
            WHEN timestamp < '2025-03-22' THEN 'v2 (프롬프트 개선)'
            ELSE 'v3 (검색 개선)'
        END AS version,
        COUNT(*) AS total,
        COUNT(CASE WHEN feedback_rating = 'positive' THEN 1 END) AS positive,
        ROUND(
            COUNT(CASE WHEN feedback_rating = 'positive' THEN 1 END) * 100.0 / COUNT(*), 1
        ) AS satisfaction_pct
    FROM catalog.schema.agent_feedback
    GROUP BY 1
    ORDER BY 1
""")
version_comparison.display()
```

> 💡 **현업 팁**: 에이전트 개선은 **2주 스프린트** 로 관리하는 것이 효과적입니다. 매 스프린트마다 "피드백 수집 → 분류 → 상위 3개 이슈 해결 → 재배포 → 효과 측정"을 반복합니다. 한 번에 모든 것을 고치려 하지 마세요. **가장 빈도 높은 문제부터 해결** 하면 만족도가 빠르게 올라갑니다.

---

## Review App vs 프로덕션 배포

| 비교 항목 | Review App | 프로덕션 |
|-----------|-----------|---------|
| **접근 대상** | 지정된 테스터만 | 모든 사용자/시스템 |
| **피드백 수집** | 내장 피드백 UI | Inference Tables + 커스텀 |
| **목적** | 품질 검증, 평가 데이터 수집 | 실제 서비스 운영 |
| **Scale to Zero** | 가능 (비용 절감) | 요구사항에 따라 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Review App** | 배포된 에이전트를 웹 UI로 테스트하고 피드백을 수집하는 도구입니다 |
| **agents.deploy()** | 에이전트 배포 시 Review App이 자동으로 생성됩니다 |
| **피드백 수집** | Thumbs Up/Down, 코멘트, Expected Response를 수집합니다 |
| **평가 데이터셋** | 피드백을 변환하여 Agent Evaluation의 골드 데이터셋으로 활용합니다 |
| **개선 사이클** | 배포 → 피드백 → 분석 → 개선 → 평가 → 재배포의 반복 과정입니다 |

---

## 참고 링크

- [Databricks: Review App](https://docs.databricks.com/aws/en/generative-ai/agent-evaluation/review-app.html)
- [Databricks: Deploy agents](https://docs.databricks.com/aws/en/generative-ai/deploy-agent.html)
- [Databricks: Agent Evaluation](https://docs.databricks.com/aws/en/generative-ai/agent-evaluation/)
- [Azure Databricks: Review App](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-evaluation/review-app)
