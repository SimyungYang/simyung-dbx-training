# 모델 평가

## 왜 체계적인 평가가 필요한가요?

AI 에이전트와 LLM 애플리케이션은 **비결정적**(같은 입력에도 매번 다른 출력)이므로, "잘 동작하는 것 같다"라는 주관적 판단만으로는 품질을 보장할 수 없습니다. MLflow Evaluate는 **자동화된 체계적 평가**를 제공합니다.

---

## MLflow Evaluate 개요

```python
import mlflow

results = mlflow.genai.evaluate(
    data=eval_dataset,              # 평가 데이터셋
    predict_fn=my_agent.predict,    # 평가 대상 (에이전트/모델)
    scorers=[...]                   # 평가 기준 (Scorer)
)
```

---

## 평가 데이터셋

평가 데이터셋은 **질문(입력)** 과 **기대 답변(선택)** 으로 구성됩니다.

```python
eval_data = [
    {
        "inputs": {"question": "반품 정책이 뭔가요?"},
        "expectations": {"expected_response": "구매 후 30일 이내 무료 반품 가능합니다."}
    },
    {
        "inputs": {"question": "배송 기간은 얼마나 걸리나요?"},
        "expectations": {"expected_response": "일반 배송 2~3 영업일, 특급 배송 당일~익일입니다."}
    },
    {
        "inputs": {"question": "회원 등급 기준이 어떻게 되나요?"},
        "expectations": {"expected_response": "연간 구매 금액 기준: Silver 50만원, Gold 200만원, Platinum 500만원 이상입니다."}
    }
]
```

### 데이터셋 소스

| 소스 | 설명 |
|------|------|
| **수동 작성** | 핵심 질문-답변 쌍을 전문가가 직접 작성합니다 |
| **프로덕션 트레이스** | MLflow Tracing에서 실제 사용자 질문을 추출합니다 |
| **Review App 피드백** | 팀원들의 평가 결과에서 생성합니다 |
| **Delta 테이블** | 기존 Q&A 데이터를 테이블에서 로드합니다 |

---

## 내장 Scorer (평가 기준)

### LLM Judge Scorer

LLM을 **심판(Judge)** 으로 사용하여 답변 품질을 자동 평가합니다.

| Scorer | 평가 내용 | 기대 답변 필요 |
|--------|----------|-------------|
| **Correctness** | 답변이 기대 답변과 **의미적으로 일치**하는지 | ✅ 필요 |
| **Safety** | 답변에 **유해하거나 부적절한 내용**이 없는지 | ❌ 불필요 |
| **RetrievalGroundedness** | 답변이 **검색된 문서에 근거**하는지 (환각 여부) | ❌ 불필요 |
| **RetrievalRelevance** | 검색된 문서가 **질문과 관련**있는지 | ❌ 불필요 |
| **Guidelines** | 사용자 정의 **가이드라인**을 준수하는지 | ❌ 불필요 |

### 사용 예시

```python
import mlflow

results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=my_agent.predict,
    scorers=[
        # 정확도: 기대 답변과 비교
        mlflow.genai.scorers.Correctness(),

        # 안전성: 유해 콘텐츠 검사
        mlflow.genai.scorers.Safety(),

        # 근거성: 검색 결과에 기반한 답변인지
        mlflow.genai.scorers.RetrievalGroundedness(),

        # 커스텀 가이드라인
        mlflow.genai.scorers.Guidelines(
            guidelines=[
                "답변은 반드시 한국어로 작성되어야 합니다",
                "답변에 출처를 명시해야 합니다",
                "모르는 질문에는 '확인 후 답변드리겠습니다'라고 답변해야 합니다",
                "경쟁사 제품을 추천하면 안 됩니다"
            ]
        )
    ]
)
```

### 커스텀 Scorer

내장 Scorer로 부족한 경우, **직접 Scorer를 작성**할 수 있습니다.

```python
from mlflow.genai.scorers import scorer

@scorer
def korean_check(request, response):
    """답변이 한국어인지 확인하는 커스텀 Scorer"""
    import re
    korean_ratio = len(re.findall('[가-힣]', response.content)) / max(len(response.content), 1)
    return {
        "score": 1.0 if korean_ratio > 0.5 else 0.0,
        "justification": f"한국어 비율: {korean_ratio:.1%}"
    }

@scorer
def length_check(request, response):
    """답변 길이가 적절한지 확인하는 커스텀 Scorer"""
    length = len(response.content)
    return {
        "score": 1.0 if 50 <= length <= 500 else 0.0,
        "justification": f"답변 길이: {length}자 ({'적절' if 50 <= length <= 500 else '부적절'})"
    }

# 커스텀 Scorer 사용
results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=my_agent.predict,
    scorers=[korean_check, length_check, mlflow.genai.scorers.Safety()]
)
```

---

## 평가 결과 분석

```python
# 전체 메트릭 요약
print(results.metrics)
# {'correctness/mean': 0.85, 'safety/mean': 0.98, 'guidelines/mean': 0.72}

# 개별 평가 결과 테이블
display(results.tables["eval_results"])
# 각 질문별로 점수, 판단 근거(justification)가 표시됩니다
```

---

## 평가 워크플로우

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 에이전트 구축 | 에이전트를 개발합니다 |
| 2 | 평가 실행 | 자동 평가를 수행합니다 |
| 3 | 품질 기준 판단 | 통과 시 배포, 미통과 시 개선으로 이동합니다 |
| 4a | 배포 | 품질 기준 통과 시 프로덕션에 배포합니다 |
| 4b | 개선 | 프롬프트, RAG, Tool을 수정하고 다시 평가합니다 |
| 5 | 프로덕션 모니터링 | 배포 후 성능을 모니터링합니다 |
| ↩ | 품질 저하 감지 | 모니터링에서 문제 발견 시 개선 단계로 돌아갑니다 |

---

## MLflow 3.0 Scorer 아키텍처 심화

### Scorer 유형 분류

MLflow 3.0에서 Scorer는 크게 세 가지 유형으로 나뉩니다.

| 유형 | 설명 | 예시 |
|------|------|------|
| **Built-in LLM Judge** | Databricks가 제공하는 사전 정의된 LLM 기반 평가 | Correctness, Safety, RetrievalGroundedness |
| **Custom LLM Judge** | 사용자 정의 프롬프트로 LLM 판단을 수행하는 Scorer | Guidelines(커스텀 규칙) |
| **Programmatic Scorer** | 코드 로직으로 평가하는 Scorer | 정규식 체크, 길이 체크, 응답 시간 체크 |

### Built-in Scorer 동작 원리

각 Built-in Scorer는 내부적으로 **전문화된 프롬프트 템플릿**과 **판단 기준 루브릭(Rubric)** 을 사용합니다.

#### Correctness (정확도)

| 항목 | 설명 |
|------|------|
| **입력** | 질문, 모델 응답, 기대 답변 (expected_response) |
| **Judge 프롬프트** | "기대 답변과 의미적으로 동일한 내용을 포함하는가?" |
| **점수** | yes/no (이진 판단) |
| **판단 근거** | 어떤 부분이 일치/불일치하는지 자연어로 설명 |

```python
# Correctness Scorer는 의미적 유사성을 평가합니다
# "30일 이내 무료 반품" vs "한 달 내 반품 가능" → ✅ 의미적으로 일치
mlflow.genai.scorers.Correctness()
```

#### RetrievalGroundedness (근거성)

| 항목 | 설명 |
|------|------|
| **입력** | 질문, 모델 응답, 검색된 문서(context/retrieved_docs) |
| **Judge 프롬프트** | "응답의 각 주장이 검색된 문서에 근거하는가?" |
| **점수** | yes/no |
| **핵심 목적** | 환각(Hallucination) 감지 — 문서에 없는 내용을 답변에 포함했는지 |

```python
# RAG 파이프라인에서 환각을 감지합니다
mlflow.genai.scorers.RetrievalGroundedness()
# 내부적으로 각 응답 문장을 retrived context와 대조합니다
```

#### RetrievalRelevance (검색 관련성)

| 항목 | 설명 |
|------|------|
| **입력** | 질문, 검색된 문서 |
| **Judge 프롬프트** | "검색된 문서가 질문에 답하는 데 관련 있는 정보를 포함하는가?" |
| **점수** | yes/no |
| **핵심 목적** | 검색 품질 평가 — Vector Search나 Retriever가 적절한 문서를 찾았는지 |

#### Safety (안전성)

| 항목 | 설명 |
|------|------|
| **입력** | 질문, 모델 응답 |
| **평가 범위** | 유해 콘텐츠, 차별적 발언, 개인정보 노출, 폭력적 내용 |
| **점수** | yes(안전)/no(위험) |
| **특이사항** | 기대 답변 불필요, 응답 자체만으로 판단 |

#### Guidelines (가이드라인)

| 항목 | 설명 |
|------|------|
| **입력** | 질문, 모델 응답, 사용자 정의 규칙 목록 |
| **Judge 프롬프트** | "응답이 제시된 각 가이드라인을 준수하는가?" |
| **점수** | yes/no (가이드라인별 개별 판단) |
| **유연성** | 자연어로 어떤 규칙이든 정의 가능 |

```python
# 비즈니스 규칙을 자연어로 정의합니다
mlflow.genai.scorers.Guidelines(
    guidelines=[
        "답변은 200자를 초과하지 않아야 합니다",
        "경쟁사 제품을 비교하거나 추천하면 안 됩니다",
        "가격 정보를 언급할 때는 반드시 '부가세 별도'를 명시해야 합니다",
        "모르는 질문에는 추측하지 말고 '확인 후 답변드리겠습니다'라고 해야 합니다"
    ]
)
```

### LLM-as-Judge 동작 원리

LLM Judge Scorer는 내부적으로 다음과 같은 과정으로 동작합니다.

```
1. 프롬프트 구성
   ├─ System Prompt: Judge 역할 정의 + 루브릭 (판단 기준)
   ├─ User Input: 평가 대상 (질문, 응답, context, expected 등)
   └─ Output Format: JSON 형태의 판단 결과 요청

2. LLM 호출
   └─ Databricks의 Foundation Model (기본: databricks-meta-llama-3-3-70b-instruct)
      또는 사용자 지정 모델

3. 결과 파싱
   ├─ score: yes/no 또는 0.0~1.0
   └─ justification: 판단 근거 (자연어)
```

> 💡 **Judge 모델 커스터마이징**: 기본 Judge 모델 대신 GPT-4o나 자체 파인튜닝된 모델을 사용할 수도 있습니다. 평가의 도메인 특화 정확도를 높이려면 Judge 모델 선택이 중요합니다.

```python
# Judge 모델을 명시적으로 지정
import mlflow

mlflow.genai.scorers.Correctness(
    # 기본값: Databricks의 Foundation Model
    # 필요시 커스텀 엔드포인트를 환경 변수로 설정
)

# 환경 변수로 Judge 모델 변경
import os
os.environ["MLFLOW_DEPLOYMENTS_TARGET"] = "databricks"
```

---

## 평가 데이터셋 설계 전략

### 효과적인 평가 데이터셋 구성

| 카테고리 | 비율 | 목적 | 예시 |
|---------|------|------|------|
| **Happy Path** | 40% | 정상적인 질문에 올바르게 답하는지 | "반품 정책은?" → 정확한 정책 안내 |
| **Edge Case** | 25% | 경계 조건, 모호한 질문 처리 | "반품" (맥락 없는 단어), 오타가 있는 질문 |
| **Out-of-Scope** | 15% | 범위 밖 질문에 적절히 거절하는지 | "오늘 날씨 어때?", "주식 추천해줘" |
| **Adversarial** | 10% | 악의적 질문에 안전하게 대응하는지 | Prompt Injection, Jailbreak 시도 |
| **Multi-turn** | 10% | 대화 맥락을 유지하는지 | "배송 기간은?" → "좀 더 빠른 방법은?" |

### 데이터셋 크기 가이드

| 단계 | 권장 크기 | 목적 |
|------|----------|------|
| **초기 개발** | 20~50개 | 빠른 반복, 주요 문제 식별 |
| **품질 검증** | 100~300개 | 체계적 평가, 통계적 유의성 확보 |
| **프로덕션 게이트** | 300~1000개 | 배포 전 최종 검증, CI/CD 게이트 |
| **지속 모니터링** | 프로덕션 트레이스 기반 | 실제 사용 패턴 반영, 지속적 품질 관리 |

### Delta 테이블 기반 평가 데이터셋 관리

```python
# 평가 데이터셋을 Delta 테이블로 중앙 관리
eval_df = spark.createDataFrame(eval_data)
eval_df.write.mode("overwrite").saveAsTable("catalog.ml.agent_eval_dataset_v1")

# 버전 관리: 데이터셋 변경 이력 추적
spark.sql("DESCRIBE HISTORY catalog.ml.agent_eval_dataset_v1")

# 이전 버전의 데이터셋으로 평가
eval_data_v0 = spark.read.option("versionAsOf", 0) \
    .table("catalog.ml.agent_eval_dataset_v1")
```

---

## 프로덕션 모니터링과의 연결

### 오프라인 평가 → 온라인 모니터링 연속체

| 단계 | 도구 | 시점 | 목적 |
|------|------|------|------|
| **개발 평가** | `mlflow.genai.evaluate()` | 개발 중 | 프롬프트/RAG 품질 확인 |
| **CI/CD 게이트** | `mlflow.genai.evaluate()` + 임계값 | 배포 전 | 자동 품질 게이트 (통과/실패 판정) |
| **A/B 테스트** | Model Serving + 트래픽 분할 | 배포 직후 | 신규 버전과 기존 버전 비교 |
| **프로덕션 모니터링** | Inference Table + 샘플링 평가 | 상시 | 품질 드리프트 감지 |

### MLflow Tracing → 평가 데이터셋 자동 생성

```python
# 프로덕션 트레이스에서 평가 데이터셋 추출
production_traces = spark.table("catalog.ml.agent_inference_logs")

# 최근 7일의 실제 질문에서 평가 데이터 생성
new_eval_data = production_traces.filter(
    "timestamp >= DATE_SUB(CURRENT_DATE(), 7)"
).select(
    col("request.question").alias("inputs"),
    col("response.answer").alias("actual_response")
).limit(100)

# 전문가 검토 후 expected_response 추가 → 평가 데이터셋에 병합
```

---

## CI/CD 통합 패턴

### 자동 평가 파이프라인

```python
# ci_evaluate.py — CI/CD 파이프라인에서 실행
import mlflow
import sys

# 평가 데이터셋 로드
eval_data = spark.table("catalog.ml.agent_eval_dataset_v1").toPandas().to_dict("records")

# 에이전트 로드
agent = mlflow.pyfunc.load_model("models:/catalog.ml.customer_agent@challenger")

# 평가 실행
results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=agent.predict,
    scorers=[
        mlflow.genai.scorers.Correctness(),
        mlflow.genai.scorers.Safety(),
        mlflow.genai.scorers.RetrievalGroundedness(),
        mlflow.genai.scorers.Guidelines(
            guidelines=["답변은 한국어로 작성되어야 합니다"]
        )
    ]
)

# 품질 게이트: 임계값 확인
thresholds = {
    "correctness/mean": 0.80,
    "safety/mean": 0.95,
    "retrieval_groundedness/mean": 0.85,
}

failed = []
for metric, threshold in thresholds.items():
    actual = results.metrics.get(metric, 0)
    if actual < threshold:
        failed.append(f"{metric}: {actual:.2f} < {threshold:.2f}")

if failed:
    print("❌ Quality gate FAILED:")
    for f in failed:
        print(f"  - {f}")
    sys.exit(1)  # CI/CD 파이프라인 실패
else:
    print("✅ Quality gate PASSED — ready for deployment")
    # champion alias 업데이트
    client = mlflow.MlflowClient()
    challenger_version = client.get_model_version_by_alias(
        "catalog.ml.customer_agent", "challenger"
    ).version
    client.set_registered_model_alias(
        "catalog.ml.customer_agent", "champion", challenger_version
    )
```

### DABs (Databricks Asset Bundles)와 통합

```yaml
# databricks.yml — 평가 작업 정의
resources:
  jobs:
    agent_evaluation:
      name: "agent-quality-gate"
      tasks:
        - task_key: evaluate
          notebook_task:
            notebook_path: ./notebooks/ci_evaluate
          existing_cluster_id: ${var.cluster_id}

      # 트리거: 모델이 등록되면 자동 실행
      trigger:
        pause_status: UNPAUSED
```

### 평가 결과 대시보드 연동

```python
# 평가 결과를 Delta 테이블에 저장 → 대시보드로 추적
import pandas as pd
from datetime import datetime

eval_record = {
    "timestamp": datetime.now(),
    "model_version": "challenger_v5",
    "correctness_mean": results.metrics["correctness/mean"],
    "safety_mean": results.metrics["safety/mean"],
    "groundedness_mean": results.metrics["retrieval_groundedness/mean"],
    "eval_dataset_size": len(eval_data),
    "passed": len(failed) == 0
}

spark.createDataFrame([eval_record]).write.mode("append") \
    .saveAsTable("catalog.ml.evaluation_history")
```

---

## Edge Case와 주의사항

| 주의사항 | 설명 |
|---------|------|
| **LLM Judge 비용** | 대규모 평가 데이터셋에서 LLM Judge는 상당한 토큰 비용이 발생합니다. 1,000개 평가 시 약 $5~$20 |
| **Judge 모델의 편향** | LLM Judge도 자체적인 편향이 있습니다. 특히 장문의 답변을 더 높게 평가하는 경향이 있으므로, 정량적 Scorer와 병행하세요 |
| **비결정적 평가** | LLM Judge의 점수는 매번 약간 다를 수 있습니다. 동일 데이터셋으로 반복 평가 시 ±5% 편차가 정상입니다 |
| **기대 답변 품질** | 기대 답변(expected_response)이 부정확하면 Correctness 점수가 의미 없어집니다. 전문가 검토가 필수입니다 |
| **평가 데이터 과적합** | 평가 데이터셋에만 최적화하면 실제 성능과 괴리가 생깁니다. 주기적으로 프로덕션 트레이스에서 새 데이터를 추가하세요 |

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **MLflow Evaluate** | 모델/에이전트 품질을 자동으로 평가하는 프레임워크입니다 |
| **Scorer** | 평가 기준. Correctness, Safety, Guidelines 등 내장 + 커스텀 |
| **LLM Judge** | LLM을 심판으로 사용하여 답변 품질을 자동 판단합니다 |
| **평가 데이터셋** | 질문 + 기대 답변으로 구성된 테스트 세트입니다 |
| **CI/CD 통합** | 품질 임계값을 설정하여 배포 전 자동 게이트를 구현합니다 |
| **프로덕션 모니터링** | 트레이스 기반으로 지속적으로 품질을 추적합니다 |
| **반복 개선** | 평가 → 개선 → 재평가 사이클로 품질을 높입니다 |

---

## 참고 링크

- [Databricks: MLflow Evaluation](https://docs.databricks.com/aws/en/mlflow/llm-evaluate.html)
- [Databricks: Agent Evaluation](https://docs.databricks.com/aws/en/generative-ai/agent-evaluation/)
- [MLflow: Evaluate](https://mlflow.org/docs/latest/llms/llm-evaluate/index.html)
