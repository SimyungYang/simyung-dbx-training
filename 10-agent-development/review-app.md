# Review App 상세

## Review App이란?

**Review App**은 배포된 AI 에이전트를 **웹 기반 채팅 UI**로 테스트하고, 테스터들이 각 응답에 대해 **피드백을 남길 수 있는** 도구입니다. 에이전트를 프로덕션에 배포하기 전에 품질을 검증하고, 수집된 피드백을 **평가 데이터셋으로 변환**하여 에이전트를 지속적으로 개선할 수 있습니다.

> 💡 **비유**: Review App은 "소프트웨어 베타 테스트 프로그램"과 같습니다. 제한된 인원이 실제로 사용해 보고, 버그나 개선 사항을 보고하는 과정입니다.

---

## agents.deploy()로 배포

Review App은 `agents.deploy()`로 에이전트를 배포할 때 **자동으로 생성**됩니다.

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

Review App의 피드백을 **Agent Evaluation의 평가 데이터셋**으로 변환할 수 있습니다. 이를 통해 에이전트 개선 사이클을 구축합니다.

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
