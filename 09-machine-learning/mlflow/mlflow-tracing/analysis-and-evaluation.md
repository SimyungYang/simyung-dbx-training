# Trace 분석과 평가 (Analysis & Evaluation)

## 1. 왜 Trace 분석과 평가가 필요한가

LLM (Large Language Model) 기반 애플리케이션과 에이전트 (Agent) 는 **블랙박스** 특성을 갖습니다. 동일한 입력에도 확률적으로 다른 출력이 나오며, 내부 추론 과정이 불투명합니다. 이런 특성은 다음 세 가지 문제를 만들어냅니다.

### 1-1. 블랙박스 문제

전통적인 소프트웨어는 단위 테스트 (Unit Test) 로 품질을 보장할 수 있습니다. 하지만 LLM 응답은 결정론적이지 않기 때문에, **실제 실행 흔적 (Trace) 을 기록하고 분석** 해야만 문제의 원인을 추적할 수 있습니다.

| 일반 소프트웨어 | LLM 애플리케이션 |
|----------------|-----------------|
| 입력 → 결정론적 출력 | 입력 → 확률적 출력 |
| 단위 테스트로 검증 가능 | Trace 기반 분석 필요 |
| 코드 디버깅으로 원인 파악 | Span 단위 분석으로 원인 파악 |
| 배포 후 변경 없음 | 프롬프트/모델/데이터 드리프트 발생 |

### 1-2. 프로덕션 품질 보장

개발 환경에서 잘 동작하던 앱이 프로덕션에서 품질이 저하되는 이유는 다양합니다.

- **데이터 드리프트 (Data Drift)**: 사용자가 예상치 못한 질문 패턴 사용
- **모델 드리프트 (Model Drift)**: LLM 공급자가 모델을 조용히 업데이트
- **컨텍스트 오염 (Context Pollution)**: 검색된 문서의 품질 저하
- **프롬프트 엣지 케이스 (Prompt Edge Case)**: 특정 입력에서만 실패하는 케이스

Trace 분석을 통해 이런 문제를 **조기에 감지** 하고 대응할 수 있습니다.

### 1-3. 비용 최적화

LLM API 호출 비용은 입력/출력 토큰 수에 비례합니다. Trace 데이터를 분석하면 다음을 최적화할 수 있습니다.

- 불필요하게 긴 시스템 프롬프트 식별
- 중복 LLM 호출 (캐싱으로 대체 가능한 구간) 발견
- 응답 생성에 불필요한 토큰이 포함된 패턴 탐지
- 고비용 모델 호출을 저비용 모델로 대체 가능한 케이스 분류

---

## 2. Trace 검색과 필터링

### 2-1. mlflow.search_traces() API

`mlflow.search_traces()` 는 기록된 Trace 를 Python 코드로 검색하는 핵심 API 입니다. 반환값은 `pandas.DataFrame` 입니다.

```python
import mlflow
import pandas as pd

# 기본 사용법: 특정 실험의 최근 Trace 조회
traces_df = mlflow.search_traces(
    experiment_ids=["123456789"],  # 실험 ID 목록
    max_results=100,               # 최대 반환 건수 (기본값: 100)
)

print(traces_df.columns.tolist())
# ['trace_id', 'experiment_id', 'timestamp_ms', 'execution_time_ms',
#  'status', 'request', 'response', 'tags', 'spans']
```

### 2-2. 태그 기반 필터링

```python
# 특정 태그 조건으로 필터링
# filter_string은 SQL WHERE 절과 유사한 DSL을 사용합니다
traces_df = mlflow.search_traces(
    experiment_ids=["123456789"],
    filter_string="tag.user_score >= '4' AND tag.environment = 'production'",
    max_results=500,
)
```

### 2-3. 상태 및 시간 기반 필터링

```python
# 에러 상태의 Trace만 조회
error_traces = mlflow.search_traces(
    experiment_ids=["123456789"],
    filter_string="status = 'ERROR'",
    max_results=200,
)

# 특정 기간의 Trace 조회 (timestamp_ms는 Unix 밀리초)
import time

seven_days_ago_ms = int((time.time() - 7 * 86400) * 1000)

recent_traces = mlflow.search_traces(
    experiment_ids=["123456789"],
    filter_string=f"timestamp_ms >= {seven_days_ago_ms}",
    order_by=["timestamp_ms DESC"],
    max_results=1000,
)

print(f"최근 7일 Trace 수: {len(recent_traces)}")
```

### 2-4. UI에서의 Trace 탐색

MLflow UI 에서 Trace 를 탐색할 때는 다음 경로를 사용합니다.

1. **Experiment 페이지** → `Traces` 탭 클릭
2. 좌측 필터 패널에서 Status, 날짜 범위, 태그 필터 적용
3. 특정 Trace 클릭 → Span 트리 (Span Tree) 뷰에서 단계별 실행 확인
4. Span 클릭 시 입출력 (Input/Output), 속성 (Attributes), 이벤트 (Events) 상세 확인

> Databricks 환경에서는 MLflow UI 가 Workspace UI 에 통합되어 있습니다. Experiment 페이지 접근 경로: **Experiments → (실험명) → Traces 탭**

---

## 3. Trace 분석 패턴

### 3-1. Latency 병목 Span 찾기

```python
import mlflow
import pandas as pd

traces_df = mlflow.search_traces(
    experiment_ids=["123456789"],
    max_results=1000,
)

# Span 목록을 플랫하게 펼치기
span_records = []
for _, row in traces_df.iterrows():
    for span in row["spans"]:
        span_records.append({
            "trace_id": row["trace_id"],
            "span_name": span["name"],
            "span_type": span.get("span_type", "UNKNOWN"),
            "latency_ms": (span["end_time_ns"] - span["start_time_ns"]) / 1e6,
            "status": span["status"]["status_code"],
        })

spans_df = pd.DataFrame(span_records)

# Span 유형별 Latency 통계
latency_stats = (
    spans_df.groupby("span_type")["latency_ms"]
    .agg(["mean", "median", lambda x: x.quantile(0.95), "count"])
    .rename(columns={"mean": "avg_ms", "median": "p50_ms", "<lambda_0>": "p95_ms", "count": "n"})
    .sort_values("p95_ms", ascending=False)
)

print(latency_stats)
# span_type     avg_ms   p50_ms   p95_ms     n
# LLM           1200.0   1100.0   2500.0   890
# RETRIEVER       80.0     65.0    200.0   890
# EMBEDDING       25.0     20.0     50.0   890
# → LLM이 P95 기준 전체 지연의 90% 이상을 차지
```

SQL (Unity Catalog 시스템 테이블) 로도 동일한 분석이 가능합니다.

```sql
-- 가장 느린 Span 유형 식별 (Unity Catalog 시스템 테이블 기준)
SELECT
    span_type,
    AVG(latency_ms)                       AS avg_latency,
    PERCENTILE(latency_ms, 0.50)          AS p50_latency,
    PERCENTILE(latency_ms, 0.95)          AS p95_latency,
    PERCENTILE(latency_ms, 0.99)          AS p99_latency,
    COUNT(*)                              AS span_count
FROM system.mlflow.trace_spans
WHERE timestamp >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY span_type
ORDER BY p95_latency DESC;
```

### 3-2. Token 사용량 분석

```python
# Trace 에서 LLM Span의 토큰 사용량 집계
token_records = []
for _, row in traces_df.iterrows():
    for span in row["spans"]:
        if span.get("span_type") == "LLM":
            attrs = span.get("attributes", {})
            token_records.append({
                "trace_id": row["trace_id"],
                "input_tokens": attrs.get("llm.token_count.prompt", 0),
                "output_tokens": attrs.get("llm.token_count.completion", 0),
                "model": attrs.get("llm.model_name", "unknown"),
            })

tokens_df = pd.DataFrame(token_records)
tokens_df["cost_usd"] = (
    tokens_df["input_tokens"] * 0.000003 +   # GPT-4o 기준 예시
    tokens_df["output_tokens"] * 0.000015
)

print("모델별 일평균 비용 추정:")
print(tokens_df.groupby("model")["cost_usd"].sum())
print(f"\n총 예상 비용 (수집 기간): ${tokens_df['cost_usd'].sum():.2f}")
```

### 3-3. 에러 패턴 분석

```python
# 에러가 발생한 Trace 비율 확인
total = len(traces_df)
error_count = (traces_df["status"] == "ERROR").sum()
print(f"에러율: {error_count}/{total} ({error_count/total*100:.1f}%)")

# 에러 Span 의 원인 분류
error_spans = []
for _, row in traces_df[traces_df["status"] == "ERROR"].iterrows():
    for span in row["spans"]:
        if span["status"]["status_code"] == "ERROR":
            events = span.get("events", [])
            for event in events:
                if event.get("name") == "exception":
                    error_spans.append({
                        "span_name": span["name"],
                        "error_type": event["attributes"].get("exception.type", "Unknown"),
                        "error_msg": event["attributes"].get("exception.message", "")[:80],
                    })

error_df = pd.DataFrame(error_spans)
print("\n에러 유형별 발생 횟수:")
print(error_df["error_type"].value_counts().head(10))
```

---

## 4. MLflow Evaluation — mlflow.genai.evaluate()

### 4-1. 기본 개념

`mlflow.genai.evaluate()` 는 LLM 애플리케이션의 출력을 **Scorer (평가자)** 를 사용하여 자동으로 채점합니다. 평가 결과는 MLflow Experiment 에 기록되어 버전 간 비교가 가능합니다.

```
평가 데이터셋 (inputs + expected_outputs)
        ↓
    앱/모델 실행 → 실제 출력 생성
        ↓
    Scorer 실행 → 각 항목 채점 (0.0 ~ 1.0 또는 True/False)
        ↓
    MLflow Experiment 에 평가 Run 기록
```

### 4-2. 기본 Scorer 종류

| Scorer | 설명 | 주요 판정 기준 |
|--------|------|---------------|
| **Guidelines** | 사용자 정의 지침 준수 여부 | 프롬프트 기반 LLM-as-a-judge |
| **Correctness** | 예상 정답과의 일치도 | 의미적 동등성 (Semantic Equivalence) |
| **Safety** | 유해 콘텐츠 포함 여부 | 폭력, 혐오, 개인정보 등 |
| **RetrievalGroundedness** | 검색 결과에 근거한 답변인지 | 환각 (Hallucination) 감지 |

### 4-3. mlflow.genai.evaluate() 사용 예시

```python
import mlflow
import mlflow.genai
from mlflow.genai.scorers import Guidelines, Correctness, Safety, RetrievalGroundedness
import pandas as pd

# 평가 데이터셋 준비
eval_dataset = pd.DataFrame([
    {
        "inputs": {"question": "Databricks의 Unity Catalog란 무엇인가요?"},
        "expected_outputs": "Unity Catalog는 Databricks의 통합 거버넌스 솔루션입니다.",
    },
    {
        "inputs": {"question": "Delta Lake의 ACID 트랜잭션을 설명해주세요."},
        "expected_outputs": "Delta Lake는 ACID 트랜잭션을 지원하여 데이터 일관성을 보장합니다.",
    },
])

# 평가할 앱 함수 정의 (이미 배포된 모델이나 로컬 함수 모두 가능)
def my_rag_app(inputs):
    question = inputs["question"]
    # 실제 앱 호출 로직
    answer = call_rag_pipeline(question)
    return {"answer": answer}

# 평가 실행
with mlflow.start_run(run_name="rag_eval_v1"):
    results = mlflow.genai.evaluate(
        data=eval_dataset,
        predict_fn=my_rag_app,
        scorers=[
            Guidelines(guidelines="답변은 한국어로 작성되어야 합니다."),
            Correctness(),
            Safety(),
            RetrievalGroundedness(),
        ],
    )

print(results.metrics)
# {'correctness/v1/mean': 0.85, 'safety/v1/mean': 1.0,
#  'retrieval_groundedness/v1/mean': 0.78, ...}
```

> **참고**: `mlflow.genai.evaluate()` 는 MLflow 2.14 이상에서 지원합니다. 기존 `mlflow.evaluate()` 와 API 구조가 다릅니다.

---

## 5. 커스텀 Scorer 작성

기본 Scorer 로 커버되지 않는 도메인 특화 평가 기준은 `@scorer` 데코레이터를 사용하여 직접 정의합니다.

### 5-1. @scorer 데코레이터 기본 구조

```python
from mlflow.genai.scorers import scorer
from mlflow.entities.assessment import Assessment

@scorer
def korean_language_check(inputs, outputs) -> Assessment:
    """응답이 한국어로만 작성되었는지 검사하는 커스텀 Scorer"""
    answer = outputs.get("answer", "")

    # 한글 문자 비율 계산
    korean_chars = sum(1 for c in answer if '\uAC00' <= c <= '\uD7A3')
    total_chars = len([c for c in answer if c.strip()])

    if total_chars == 0:
        return Assessment(name="korean_language_check", value=False, rationale="빈 응답")

    korean_ratio = korean_chars / total_chars
    passed = korean_ratio >= 0.5  # 50% 이상 한글이면 통과

    return Assessment(
        name="korean_language_check",
        value=passed,
        rationale=f"한글 비율: {korean_ratio:.1%}",
    )
```

### 5-2. LLM-as-a-Judge 패턴 커스텀 Scorer

```python
import openai
from mlflow.genai.scorers import scorer
from mlflow.entities.assessment import Assessment

@scorer
def domain_relevance(inputs, outputs) -> Assessment:
    """응답이 Databricks/데이터 엔지니어링 도메인에 관련되는지 판단"""
    question = inputs.get("question", "")
    answer = outputs.get("answer", "")

    prompt = f"""다음 질문과 답변이 데이터 엔지니어링 또는 Databricks 플랫폼과 관련이 있는지 판단하세요.

질문: {question}
답변: {answer}

판단 기준:
- PASS: Databricks, Apache Spark, Delta Lake, MLflow, Unity Catalog 등과 명확히 관련
- FAIL: 위 주제와 무관하거나 일반적인 잡담 수준의 답변

반드시 JSON 형식으로만 응답하세요:
{{"result": "PASS" or "FAIL", "reason": "판단 이유 한 문장"}}"""

    response = openai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"},
    )

    import json
    verdict = json.loads(response.choices[0].message.content)

    return Assessment(
        name="domain_relevance",
        value=(verdict["result"] == "PASS"),
        rationale=verdict["reason"],
    )
```

### 5-3. 커스텀 Scorer 등록 및 실행

```python
# 여러 커스텀 Scorer 를 함께 사용하기
results = mlflow.genai.evaluate(
    data=eval_dataset,
    predict_fn=my_rag_app,
    scorers=[
        Correctness(),           # 기본 Scorer
        korean_language_check,   # 커스텀: 언어 확인
        domain_relevance,        # 커스텀: 도메인 연관성
    ],
)
```

---

## 6. 평가 데이터셋 구축

좋은 평가는 좋은 데이터셋에서 시작합니다. 평가 데이터셋은 크게 세 가지 방법으로 구축합니다.

### 6-1. Trace에서 데이터셋 추출

프로덕션 Trace 에서 고품질 사례를 자동으로 수집합니다.

```python
import mlflow

# 사용자 피드백이 좋은 Trace 를 평가 데이터로 수집
good_traces = mlflow.search_traces(
    experiment_ids=["123456789"],
    filter_string="tag.user_score >= '4' AND status = 'OK'",
    max_results=500,
)

# 평가 데이터셋 형식으로 변환
eval_records = []
for _, row in good_traces.iterrows():
    request = row["request"]
    response = row["response"]
    eval_records.append({
        "inputs": request,
        "expected_outputs": response,  # 좋은 응답을 Gold Answer로 사용
        "source_trace_id": row["trace_id"],
    })

import pandas as pd
eval_df = pd.DataFrame(eval_records)
print(f"수집된 평가 데이터: {len(eval_df)}건")
```

### 6-2. 수동 레이블링 워크플로우

자동 수집만으로는 엣지 케이스를 충분히 커버하기 어렵습니다. 도메인 전문가가 직접 레이블링하는 과정이 필요합니다.

```python
# 검토가 필요한 Trace 에 태그 붙이기
def flag_for_review(trace_id: str, reason: str):
    mlflow.set_trace_tag(trace_id, "review_status", "pending")
    mlflow.set_trace_tag(trace_id, "review_reason", reason)

# 낮은 점수를 받은 Trace 를 검토 대상으로 표시
low_score_traces = mlflow.search_traces(
    experiment_ids=["123456789"],
    filter_string="tag.user_score <= '2'",
)

for _, row in low_score_traces.iterrows():
    flag_for_review(row["trace_id"], "낮은 사용자 평점 - 전문가 검토 필요")
```

### 6-3. Gold Set 관리

**Gold Set** 은 정확성이 검증된 핵심 평가 데이터셋입니다. Spark DataFrame 또는 Delta Table 로 관리하면 버전 관리와 공유가 용이합니다.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

# Gold Set을 Delta Table로 저장
eval_df_spark = spark.createDataFrame(eval_df)
eval_df_spark.write.format("delta").mode("overwrite").saveAsTable(
    "ml_catalog.evaluation.rag_gold_set_v1"
)

# CI/CD 파이프라인에서 Gold Set 로드 후 평가 실행
gold_set = spark.table("ml_catalog.evaluation.rag_gold_set_v1").toPandas()
results = mlflow.genai.evaluate(
    data=gold_set,
    predict_fn=my_rag_app,
    scorers=[Correctness(), Safety()],
)
```

---

## 7. 프로덕션 모니터링

### 7-1. Trace 기반 품질 대시보드

Unity Catalog 시스템 테이블 (`system.mlflow.traces`) 을 Databricks SQL 로 쿼리하여 품질 대시보드를 구축합니다.

```sql
-- 일별 핵심 품질 지표 집계
SELECT
    DATE(TIMESTAMP_MILLIS(timestamp_ms))          AS day,
    COUNT(*)                                       AS total_traces,
    SUM(CASE WHEN status = 'ERROR' THEN 1 ELSE 0 END)   AS error_count,
    SUM(CASE WHEN status = 'ERROR' THEN 1 ELSE 0 END)
        * 100.0 / COUNT(*)                         AS error_rate_pct,
    AVG(execution_time_ms)                         AS avg_latency_ms,
    PERCENTILE(execution_time_ms, 0.95)            AS p95_latency_ms,
    AVG(CAST(tags['user_score'] AS DOUBLE))        AS avg_user_score
FROM system.mlflow.traces
WHERE experiment_id = '123456789'
    AND timestamp_ms >= UNIX_MILLIS(CURRENT_DATE() - INTERVAL 30 DAYS)
GROUP BY DATE(TIMESTAMP_MILLIS(timestamp_ms))
ORDER BY day DESC;
```

### 7-2. 드리프트 감지

이동 평균 (Moving Average) 비교로 품질 드리프트를 조기에 감지합니다.

```python
import mlflow
import pandas as pd
import numpy as np

traces_df = mlflow.search_traces(
    experiment_ids=["123456789"],
    max_results=2000,
    order_by=["timestamp_ms ASC"],
)

# 7일 이동 평균 에러율 계산
traces_df["date"] = pd.to_datetime(traces_df["timestamp_ms"], unit="ms").dt.date
daily = traces_df.groupby("date").agg(
    total=("status", "count"),
    errors=("status", lambda x: (x == "ERROR").sum()),
).reset_index()

daily["error_rate"] = daily["errors"] / daily["total"]
daily["rolling_7d_error_rate"] = daily["error_rate"].rolling(7).mean()

# 최근 3일 평균이 기준선의 2배 이상이면 경고
baseline = daily["rolling_7d_error_rate"].iloc[:-3].mean()
recent = daily["rolling_7d_error_rate"].iloc[-3:].mean()

if recent > baseline * 2:
    print(f"[경고] 에러율 급증 감지! 기준선: {baseline:.1%}, 최근 3일: {recent:.1%}")
```

### 7-3. 알림 설정 (Databricks Workflow 활용)

Databricks Workflow (Jobs) 를 사용하여 주기적으로 품질 지표를 체크하고 임계값 초과 시 알림을 전송합니다.

```python
# monitoring_job.py — Databricks Job Task로 등록하여 매시간 실행
import mlflow
import requests
import os

SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]

def send_alert(message: str):
    requests.post(SLACK_WEBHOOK, json={"text": message})

# 최근 1시간 Trace 에러율 확인
import time
one_hour_ago = int((time.time() - 3600) * 1000)

recent = mlflow.search_traces(
    experiment_ids=["123456789"],
    filter_string=f"timestamp_ms >= {one_hour_ago}",
    max_results=500,
)

if len(recent) > 0:
    error_rate = (recent["status"] == "ERROR").mean()
    if error_rate > 0.05:  # 에러율 5% 초과 시 알림
        send_alert(
            f":warning: RAG 앱 에러율 급증\n"
            f"최근 1시간 에러율: {error_rate:.1%}\n"
            f"총 요청 수: {len(recent)}"
        )
```

---

## 8. 베스트 프랙티스

### 8-1. 평가 주기 가이드

| 상황 | 권장 평가 주기 | 비고 |
|------|--------------|------|
| 모델 버전 업데이트 | 배포 전 반드시 실행 | Gold Set 전체 평가 |
| 프롬프트 변경 | 변경 전후 비교 평가 | A/B 테스트와 병행 |
| 프로덕션 정기 평가 | 주 1회 | 샘플링된 Trace 활용 |
| 드리프트 의심 시 | 즉시 | 최근 Trace 기반 |

### 8-2. Scorer 선택 가이드

```
시나리오별 Scorer 조합 추천:

[RAG 챗봇]
  - Correctness: 정답 일치도
  - RetrievalGroundedness: 환각 감지
  - Guidelines: 답변 형식/언어 준수

[에이전트 (Agent)]
  - Safety: 유해 행동 방지
  - Guidelines: 도구 사용 정책 준수
  - 커스텀 Scorer: 도메인 규칙 (예: 금융 규정 준수)

[코드 생성]
  - 커스텀 Scorer: 코드 실행 가능 여부 (subprocess 실행 테스트)
  - Guidelines: 보안 취약 패턴 금지 (eval, exec 등)
```

### 8-3. 비용 vs 품질 트레이드오프

LLM-as-a-Judge 방식의 Scorer 는 판단 품질이 높지만 호출 비용이 발생합니다.

```python
# 비용 절감 전략 1: 샘플링 평가
# 전체 Trace 의 10% 만 평가하여 비용 절감
import random

all_traces = mlflow.search_traces(experiment_ids=["123456789"], max_results=1000)
sample_size = max(50, int(len(all_traces) * 0.1))
sampled = all_traces.sample(n=sample_size, random_state=42)

# 비용 절감 전략 2: 경량 모델로 빠른 1차 필터링
# gpt-4o-mini → 문제 있는 케이스만 gpt-4o 로 심층 평가

# 비용 절감 전략 3: 규칙 기반 Scorer 우선 적용
# 명백한 오류 (에러 상태, 빈 응답) 는 LLM 호출 없이 탈락
@scorer
def basic_sanity_check(inputs, outputs) -> Assessment:
    answer = outputs.get("answer", "")
    if not answer or len(answer.strip()) < 10:
        return Assessment(name="basic_sanity_check", value=False, rationale="빈 응답 또는 너무 짧은 응답")
    return Assessment(name="basic_sanity_check", value=True, rationale="기본 형식 통과")
```

### 8-4. Flywheel (플라이휠) 효과 구축

프로덕션 데이터를 평가에 활용하는 선순환 구조를 만드는 것이 최종 목표입니다.

```
프로덕션 트레이스 수집
        ↓
사용자 피드백 태그 연결 (mlflow.set_trace_tag)
        ↓
고품질 Trace → Gold Set 편입 (Delta Table)
        ↓
새 모델/프롬프트 버전 평가 (mlflow.genai.evaluate)
        ↓
개선된 버전 배포 → 더 나은 트레이스 생성
        ↓ (반복)
```

이 사이클을 자동화하면 시간이 지날수록 GenAI 애플리케이션의 품질이 **지속적으로 향상** 됩니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Trace 분석** | 블랙박스 LLM 앱의 실행 흔적을 기록하고 Latency, 토큰, 에러 패턴을 분석합니다 |
| **search_traces()** | 태그, 상태, 시간 조건으로 Trace 를 Python DataFrame 으로 조회합니다 |
| **genai.evaluate()** | 평가 데이터셋과 Scorer 를 조합하여 LLM 앱을 자동 채점합니다 |
| **기본 Scorer** | Guidelines, Correctness, Safety, RetrievalGroundedness 등을 제공합니다 |
| **커스텀 Scorer** | `@scorer` 데코레이터로 도메인 특화 평가 기준을 직접 정의합니다 |
| **Gold Set** | 검증된 평가 데이터를 Delta Table 로 버전 관리하고 CI/CD 에 통합합니다 |
| **프로덕션 모니터링** | 시스템 테이블 + Workflow Job 으로 드리프트를 자동 감지하고 알림을 전송합니다 |
| **Flywheel 효과** | 프로덕션 트레이스 → 평가 데이터 → 모델 개선의 선순환을 구축합니다 |

---

## 참고 링크

- [Databricks: MLflow Tracing](https://docs.databricks.com/aws/en/mlflow/mlflow-tracing.html)
- [MLflow: Tracing Overview](https://mlflow.org/docs/latest/llms/tracing/index.html)
- [MLflow: search_traces() API Reference](https://mlflow.org/docs/latest/python_api/mlflow.html#mlflow.search_traces)
- [MLflow: GenAI Evaluate](https://mlflow.org/docs/latest/llms/llm-evaluate/index.html)
- [MLflow: Built-in Scorers](https://mlflow.org/docs/latest/llms/llm-evaluate/index.html#built-in-scorers)
- [MLflow: Custom Scorers](https://mlflow.org/docs/latest/llms/llm-evaluate/index.html#custom-scorers)
- [Databricks: Instrument agents](https://docs.databricks.com/aws/en/generative-ai/agent-framework/trace-agents.html)
- [Databricks: Monitor AI quality](https://docs.databricks.com/aws/en/generative-ai/agent-evaluation/index.html)
