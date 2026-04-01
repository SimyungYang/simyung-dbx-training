# Databricks ML 개요

## 왜 Databricks에서 ML을 하나요?

전통적으로 ML 워크플로우는 **도구가 파편화** 되어 있었습니다. 데이터 준비는 Spark, 모델 학습은 Jupyter + GPU 서버, 실험 추적은 별도 서버, 모델 배포는 또 다른 도구... 이런 분산된 환경은 관리가 복잡하고, 데이터를 여러 시스템 간에 복사해야 하며, 거버넌스 적용이 어렵습니다.

Databricks는 이 모든 과정을 **하나의 플랫폼**에서 수행할 수 있게 합니다. 데이터 엔지니어가 준비한 데이터를 별도로 내보내지 않고 바로 ML 학습에 사용할 수 있으며, 학습된 모델은 클릭 한 번으로 REST API로 배포됩니다. 이러한 통합 환경은 **ML 프로젝트의 생산성을 극적으로 향상** 시키고, 실험에서 프로덕션까지의 시간(Time to Production)을 크게 단축합니다.

### 외부 ML 플랫폼과의 비교

ML을 수행할 수 있는 플랫폼은 여러 가지가 있습니다. 각각의 장단점을 이해하면 Databricks의 차별점을 더 명확하게 파악할 수 있습니다.

| 비교 항목 | Databricks | AWS SageMaker | Google Vertex AI | Azure ML Studio |
|-----------|-----------|---------------|-----------------|-----------------|
| **데이터 통합**| Lakehouse 데이터 직접 사용 | S3에서 데이터 로드 필요 | BigQuery/GCS 연동 | Azure Storage 연동 |
| **실험 추적**| MLflow 기본 내장 | SageMaker Experiments | Vertex Experiments | MLflow/Azure ML |
| **모델 레지스트리**| Unity Catalog 통합 | SageMaker Registry | Vertex Model Registry | Azure ML Registry |
| **거버넌스**| UC로 데이터~모델 통합 거버넌스 | IAM 기반 | IAM 기반 | RBAC 기반 |
| **GenAI 지원**| Agent Framework, Tracing | JumpStart, Bedrock 연동 | Model Garden, Gemini | Azure OpenAI 연동 |
| **오픈소스**| MLflow (오픈소스) 기반 | 자체 SDK | 자체 SDK | MLflow 지원 |
| **멀티클라우드**| AWS, Azure, GCP 모두 지원 | AWS 전용 | GCP 전용 | Azure 전용 |

> 💡 **핵심 차별점**: Databricks는 **데이터 레이크하우스와 ML이 하나의 플랫폼에 통합** 되어 있다는 점이 가장 큰 강점입니다. 다른 플랫폼에서는 데이터 준비와 ML이 별도 서비스이므로 데이터 이동과 권한 관리의 복잡성이 증가합니다.

---

## ML 워크플로우 전체 그림

Databricks에서의 ML 워크플로우는 크게 5단계로 구성됩니다. 각 단계가 유기적으로 연결되어 있어, 한 플랫폼 안에서 전체 ML 라이프사이클을 관리할 수 있습니다.

| 단계 | 작업 | 설명 |
|------|------|------|
| 1 | 데이터 준비 | Feature Engineering으로 학습 데이터를 준비합니다 |
| 2 | 실험/학습 | MLflow Tracking으로 실험을 추적하며 모델을 학습합니다 |
| 3 | 모델 등록 | UC Model Registry에 모델을 등록하여 버전 관리합니다 |
| 4 | 모델 배포 | Model Serving으로 REST API 엔드포인트에 배포합니다 |
| 5 | 모니터링 | Inference Tables로 성능을 모니터링합니다 |
| ↩ | 재학습/개선 | 모니터링 결과를 바탕으로 실험/학습 단계로 돌아가 반복합니다 |

| 단계 | Databricks 도구 | 역할 | 전통적 도구 (비교) |
|------|----------------|------|------------------|
| **데이터 준비**| Feature Engineering, Spark | 피처 테이블 생성, 학습 데이터 준비 | pandas (로컬), 별도 Feature Store |
| **실험/학습**| MLflow Tracking, Notebooks | 하이퍼파라미터 튜닝, 모델 학습, 실험 비교 | Jupyter + W&B / TensorBoard |
| **모델 등록**| Unity Catalog Models | 모델 버전 관리, 승격(Alias) 관리 | 수동 파일 관리 또는 별도 Registry |
| **모델 배포**| Model Serving | REST API 엔드포인트로 실시간 추론 | Flask + Docker + K8s |
| **모니터링**| Inference Tables, Lakehouse Monitoring | 성능 추적, 드리프트 감지, 비용 추적 | 별도 모니터링 도구 |

### 단계별 상세 설명

#### 1단계: 데이터 준비 (Feature Engineering)

ML 모델의 성능은 데이터 품질에 크게 좌우됩니다. Databricks의 Feature Engineering 기능을 사용하면, Lakehouse에 저장된 원본 데이터로부터 **피처(Feature)**를 추출하고 재사용 가능한 피처 테이블로 관리할 수 있습니다. 피처 테이블은 학습과 서빙에서 동일한 피처를 일관되게 사용하도록 보장하여, **학습-서빙 불일치(Training-Serving Skew)** 문제를 방지합니다.

#### 2단계: 실험 및 학습 (MLflow Tracking)

MLflow Tracking은 모든 실험의 **하이퍼파라미터, 메트릭, 모델 아티팩트** 를 자동으로 기록합니다. `mlflow.autolog()` 한 줄이면 대부분의 ML 프레임워크에서 자동 로깅이 활성화되며, 여러 실험을 UI에서 시각적으로 비교하여 최적의 모델을 선택할 수 있습니다. Databricks Notebook에서 Spark의 분산 컴퓨팅 파워를 활용하여 대규모 데이터셋으로 학습하는 것도 가능합니다.

#### 3단계: 모델 등록 (Unity Catalog Model Registry)

학습이 완료된 모델은 Unity Catalog에 등록하여 **버전 관리, 접근 권한, 리니지 추적** 을 적용합니다. Alias 기능(`champion`, `challenger`)을 통해 프로덕션 모델과 테스트 모델을 명확하게 구분하며, 데이터 테이블과 동일한 거버넌스 체계 아래에서 모델을 관리합니다.

#### 4단계: 모델 배포 (Model Serving)

등록된 모델을 **서버리스 REST API 엔드포인트** 로 배포합니다. 인프라 관리 없이 오토스케일링이 적용되며, A/B 테스트를 위한 트래픽 분할도 지원합니다. 전통 ML 모델뿐 아니라 LLM, RAG 체인, AI 에이전트까지 동일한 방식으로 배포할 수 있습니다.

#### 5단계: 모니터링 (Inference Tables + Lakehouse Monitoring)

배포된 모델의 모든 입력/출력이 **Inference Table** 에 자동 기록됩니다. 이 데이터를 기반으로 응답 지연시간, 오류율, 데이터 드리프트를 모니터링하며, 품질이 저하되면 알림을 받아 재학습 파이프라인을 트리거할 수 있습니다.

---

## 전통 ML vs GenAI 워크플로우

Databricks는 전통적인 ML(테이블 데이터 기반 예측)과 GenAI(LLM 기반 에이전트) **모두** 를 지원합니다. 두 워크플로우는 데이터 유형, 모델 종류, 평가 방식에서 근본적인 차이가 있지만, Databricks에서는 동일한 플랫폼 위에서 두 가지 모두를 수행할 수 있습니다.

| 비교 | 전통 ML | GenAI / LLM |
|------|---------|------------|
| **데이터**| 정형 데이터 (테이블) | 텍스트, 문서, 이미지 |
| **모델**| scikit-learn, XGBoost, PyTorch | Llama, GPT, Claude + RAG/Agent |
| **학습**| 커스텀 모델 직접 학습 | 사전학습 모델 사용 + Fine-tuning (선택) |
| **입력**| 피처 벡터 (수치) | 자연어 텍스트, 프롬프트 |
| **출력**| 예측값 (수치, 클래스) | 생성된 텍스트, 도구 호출 |
| **평가**| accuracy, F1, AUC | Correctness, Safety, Groundedness |
| **Databricks 도구**| Feature Store, Tracking, Serving | Vector Search, Tracing, Agent Framework |

### 전통 ML 예시: 이탈 예측

고객의 과거 행동 데이터를 바탕으로 이탈 가능성을 예측하는 전형적인 분류(Classification) 문제입니다. `mlflow.autolog()`를 사용하면 모델의 파라미터, 메트릭, 학습된 모델 자체가 모두 자동으로 MLflow에 기록됩니다.

```python
import mlflow
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import train_test_split

mlflow.autolog()  # 자동 로깅 활성화

# 데이터 준비 — Lakehouse의 Gold 테이블에서 직접 로드
train_df = spark.table("gold.customer_features").toPandas()
X = train_df.drop("churned", axis=1)
y = train_df["churned"]

# 학습/테스트 분할
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# 모델 학습 → 자동으로 파라미터/메트릭/모델이 MLflow에 기록됩니다
with mlflow.start_run(run_name="churn_prediction_v1"):
    model = GradientBoostingClassifier(n_estimators=200, max_depth=5, learning_rate=0.1)
    model.fit(X_train, y_train)

    # 테스트 셋 평가
    accuracy = model.score(X_test, y_test)
    print(f"테스트 정확도: {accuracy:.4f}")
```

### GenAI 예시: 고객 지원 에이전트

사전학습된 LLM에 사내 문서를 RAG로 연결하고, 주문 조회 같은 도구(Tool)를 부여하여 고객 지원 에이전트를 구축합니다. MLflow에 에이전트를 로깅하면 동일한 Model Serving 인프라로 배포할 수 있습니다.

```python
import mlflow

# RAG 기반 에이전트 구축
agent = CustomerSupportAgent(
    llm="databricks-meta-llama-3-3-70b-instruct",
    vector_index="catalog.schema.docs_index",
    tools=["get_order_status", "create_ticket"]
)

# MLflow에 에이전트 로깅
with mlflow.start_run():
    mlflow.pyfunc.log_model("agent", python_model=agent)

# 에이전트를 Unity Catalog에 등록
mlflow.register_model(
    "runs:/<run_id>/agent",
    "production.ml_models.customer_support_agent"
)
```

---

## 실무 워크플로우 예시: 엔드투엔드 ML 파이프라인

아래는 실제 프로덕션 환경에서 Databricks ML을 활용하는 전체 흐름을 보여주는 예시입니다. 이탈 예측 모델을 데이터 준비부터 배포, 모니터링까지 자동화하는 시나리오입니다.

| 태스크 | 작업 | 상세 |
|--------|------|------|
| **트리거**| 매일 자정 Lakeflow Jobs | - |
| Task 1 | Feature Engineering | Spark로 customer_features 테이블 갱신 |
| Task 2 | Model Training | 최신 피처로 모델 재학습 + MLflow 자동 로깅 |
| Task 3 | Model Evaluation | 테스트 셋으로 성능 검증 (정확도 > 0.85?) |
| Task 4 (성공 시) | Model Registration | UC에 새 버전 등록 + champion Alias 부여 |
| Task 5 | Model Serving 갱신 | 엔드포인트가 자동으로 새 모델 버전을 서빙 |

이처럼 Databricks에서는 데이터 파이프라인과 ML 파이프라인이 동일한 플랫폼에서 동작하므로, Lakeflow Jobs로 전체 흐름을 하나의 워크플로우로 관리할 수 있습니다.

---

## Databricks ML 핵심 컴포넌트 맵

아래 표는 Databricks에서 제공하는 ML 관련 컴포넌트를 카테고리별로 정리한 것입니다. 각 컴포넌트의 역할을 파악하면, 본인의 ML 프로젝트에 어떤 도구를 사용해야 하는지 빠르게 판단할 수 있습니다.

| 카테고리 | 컴포넌트 | 설명 |
|----------|----------|------|
| **컴퓨트**| ML Runtime | ML 라이브러리(PyTorch, TensorFlow, scikit-learn, XGBoost 등)가 사전 설치된 런타임입니다. 별도 패키지 설치 없이 바로 학습을 시작할 수 있습니다 |
| | GPU Clusters | NVIDIA GPU가 장착된 클러스터로, 딥러닝 모델 학습과 LLM 파인튜닝에 사용합니다. 단일 노드부터 멀티노드 분산 학습까지 지원합니다 |
| **실험 관리**| MLflow Tracking | 파라미터, 메트릭, 아티팩트를 체계적으로 추적하고, 여러 실험을 UI에서 시각적으로 비교합니다 |
| | MLflow Autolog | `mlflow.autolog()` 한 줄로 주요 ML 프레임워크의 자동 로깅을 활성화합니다. 수동 코드 작성이 최소화됩니다 |
| | Notebooks | 대화형 개발 환경으로, 코드 실행, 시각화, 문서화를 한 곳에서 수행합니다. Python, R, Scala, SQL을 모두 지원합니다 |
| **피처**| Feature Engineering | 피처 테이블을 생성하고 관리합니다. 학습과 서빙에서 동일한 피처를 보장하여 Training-Serving Skew를 방지합니다 |
| | Online Tables | 실시간 추론을 위해 피처를 밀리초 단위로 서빙합니다. Feature Table의 온라인 버전입니다 |
| **모델 관리**| UC Model Registry | Unity Catalog에서 모델 버전과 Alias를 관리합니다. 데이터 테이블과 동일한 거버넌스 체계가 적용됩니다 |
| **배포**| Model Serving | 서버리스 REST API 엔드포인트로 모델을 배포합니다. 오토스케일링, A/B 테스트, GPU 서빙을 지원합니다 |
| | Foundation Model API | Databricks가 호스팅하는 LLM(Llama, DBRX 등)을 즉시 사용할 수 있는 API입니다. Pay-per-token과 Provisioned Throughput 과금을 지원합니다 |
| **GenAI**| Vector Search | 임베딩 벡터 기반 유사도 검색 서비스입니다. RAG 파이프라인에서 관련 문서를 검색하는 데 사용합니다 |
| | Agent Framework | AI 에이전트를 구축하기 위한 프레임워크입니다. Tool 연결, 멀티턴 대화, 멀티 에이전트 오케스트레이션을 지원합니다 |
| | MLflow Tracing | GenAI 앱의 실행 흐름(LLM 호출, Tool 실행, 검색 등)을 단계별로 추적하여 디버깅과 최적화에 활용합니다 |
| **평가**| MLflow Evaluate | 모델/에이전트의 품질을 자동으로 평가합니다. LLM Judge 기반의 Correctness, Safety, Groundedness 등의 Scorer를 제공합니다 |
| **모니터링**| Inference Tables | 서빙 엔드포인트의 모든 추론 입력/출력을 Delta 테이블에 자동 기록합니다. 품질 분석과 감사의 기반 데이터입니다 |
| | Lakehouse Monitoring | 데이터와 모델의 드리프트를 자동 감지합니다. 통계적 변화가 감지되면 알림을 보내어 선제적 대응이 가능합니다 |

---

## ML Runtime 상세

ML 워크로드를 실행할 때는 **ML Runtime** 이 포함된 클러스터를 사용하는 것이 좋습니다. ML Runtime은 Standard Runtime에 머신러닝에 필요한 라이브러리를 사전 설치하여 제공하므로, 환경 설정에 들이는 시간을 절약할 수 있습니다.

### ML Runtime에 포함된 주요 라이브러리

| 카테고리 | 라이브러리 |
|----------|-----------|
| **전통 ML**| scikit-learn, XGBoost, LightGBM, CatBoost |
| **딥러닝**| PyTorch, TensorFlow, Keras |
| **데이터 처리**| pandas, NumPy, SciPy |
| **시각화**| matplotlib, seaborn, plotly |
| **NLP**| Hugging Face Transformers, tokenizers |
| **ML 관리**| MLflow (자동 내장) |
| **분산 학습**| Horovod, DeepSpeed, PyTorch Distributed |
| **하이퍼파라미터 튜닝**| Hyperopt, Optuna |

> 💡 **GPU Runtime** 은 ML Runtime에 CUDA, cuDNN 등 GPU 드라이버가 추가된 버전입니다. 딥러닝이나 LLM 파인튜닝 시에는 반드시 GPU Runtime을 선택해야 합니다.

---

## 현장에서 배운 것들: ML 프로젝트의 현실

### ML 프로젝트의 80%는 데이터 준비에 쓰인다 — 실화

이건 과장이 아니라 수치로 증명된 현실입니다. 제가 참여한 30개 이상의 ML 프로젝트에서 시간 분배를 추적해봤습니다.

| 단계 | 교과서 예상 | 현실 (평균) | 실제 하는 일 |
|------|-----------|-----------|------------|
| **데이터 수집/탐색**| 10% | 25% | "이 컬럼이 뭔지 아는 사람?" 회의, 데이터 사전 만들기, 원본 DB 접근 권한 받기 |
| **데이터 정제/피처 엔지니어링**| 20% | 55% | NULL 처리, 이상치 제거, 피처 생성, 레이블 정의 논의, 데이터 불균형 처리 |
| **모델 학습/실험**| 40% | 10% | AutoML 또는 XGBoost 한두 번 돌리면 의외로 빠름 |
| **평가/배포/모니터링**| 30% | 10% | 모델은 좋은데 배포하는 데 3개월 걸림 (인프라팀과 협의...) |

> 💡 **현업 진실**: "멋진 알고리즘"보다 "**깨끗한 데이터**"가 모델 성능에 10배 더 큰 영향을 미칩니다. XGBoost + 잘 정제된 피처가 최신 딥러닝 모델 + 지저분한 데이터를 이기는 경우를 수없이 봤습니다. Databricks의 가치는 이 데이터 준비 과정을 **같은 플랫폼 안에서** 할 수 있다는 것입니다.

### Jupyter 노트북에서 시작한 모델이 프로덕션까지 가는 현실적 여정

데이터 과학자의 로컬 Jupyter에서 시작된 모델이 실제 서비스에 배포되기까지의 현실적인 여정을 그려보겠습니다.

#### 전통적 방식 (Databricks 없이)

```
[1주차] 데이터 과학자: Jupyter에서 모델 개발
    → "내 노트북에서는 잘 되는데요!"

[2주차] 데이터 과학자 → DE: "이 데이터를 S3에서 가져와주세요"
    → DE: "어떤 포맷으로? 스키마 정의해주세요" → 2주 소요

[4주차] 모델 완성 → "배포해주세요"
    → 인프라팀: "Docker 이미지 만들어주세요"
    → 데이터 과학자: "Docker가 뭔가요?"

[6주차] Docker 이미지 완성
    → 인프라팀: "K8s에 올렸는데 메모리 부족이요"
    → 사이즈 조정, 오토스케일링 설정...

[8주차] 배포 완료!
    → 2주 뒤: "모델 성능이 떨어지는데 왜 그런지 모르겠어요"
    → 모니터링 없음, 입력 데이터 드리프트 감지 불가

[12주차] "모델 재학습 하고 싶은데, 처음에 어떤 데이터로 학습했는지 기억이..."
    → 재현 불가능 → 처음부터 다시 시작

총 소요: 3개월, 재학습 주기: "할 수 있으면..."
```

#### Databricks 방식

```
[1일차] Databricks Notebook에서 모델 개발
    → Lakehouse의 Gold 테이블에서 직접 데이터 로드
    → mlflow.autolog()로 자동 실험 추적

[2일차] 최적 모델 선택 → UC Model Registry에 등록
    → 모델 버전, 학습 데이터 리니지, 피처 의존성 모두 기록

[3일차] Model Serving 엔드포인트 생성 (클릭 한 번)
    → 서버리스 오토스케일링 자동 적용
    → Inference Table 자동 활성화 → 모든 입출력 기록

[4일차] Lakeflow Job으로 재학습 파이프라인 자동화
    → 주 1회 자동 재학습 → 성능 검증 → 자동 배포

총 소요: 4일, 재학습 주기: 자동 (주 1회)
```

### MLOps 성숙도 모델: 우리 팀은 어디에 있나요?

제가 고객을 컨설팅할 때 사용하는 MLOps 성숙도 모델입니다. 각 레벨의 현실적인 모습을 그려보겠습니다.

| 레벨 | 이름 | 특징 | 현실 | Databricks 도구 |
|------|------|------|------|----------------|
| **0**| No MLOps | 수동 실험, 모델 pkl 파일 전달 | "모델 파일 슬랙으로 보내줄게" | - |
| **1**| 수동 추적 | MLflow로 실험 추적 시작 | "적어도 어떤 파라미터가 좋은지는 알겠다" | MLflow Tracking |
| **2**| 모델 관리 | Model Registry, 버전 관리 | "champion 모델이 뭔지는 알겠다" | UC Model Registry |
| **3**| 자동 배포 | CI/CD로 모델 자동 배포 | "코드 머지하면 모델이 올라간다" | Model Serving + Jobs |
| **4**| 자동 재학습 | 모니터링 + 자동 재학습 파이프라인 | "모델이 알아서 최신 상태를 유지한다" | Inference Tables + Monitoring + Jobs |
| **5**| 완전 자동화 | Feature Store, A/B 테스트, Shadow 배포 | "새 모델이 기존보다 낫다는 것을 자동으로 검증하고 롤아웃" | Feature Engineering + A/B + 전체 통합 |

> ⚠️ **많은 팀이 이 실수를 합니다**: 레벨 0에서 바로 레벨 5를 목표로 하면 실패합니다. **한 단계씩 성숙도를 올리세요.** 대부분의 팀은 레벨 2~3에서 이미 엄청난 생산성 향상을 경험합니다. 레벨 3에 도달하는 데 보통 3~6개월이 걸립니다.

### ML 프로젝트 실패의 5가지 패턴

성공 사례보다 **실패 패턴** 을 아는 것이 더 가치 있습니다. 20년간 목격한 가장 흔한 실패 패턴입니다.

| 패턴 | 증상 | 원인 | 해결책 |
|------|------|------|--------|
| "**정확도 99%인데 쓸 수 없어요**" | 테스트 정확도는 높지만 실제 사용 시 엉망 | 데이터 누수(Data Leakage), 학습/서빙 스큐 | Feature Store로 학습/서빙 피처 일관성 보장 |
| "**모델이 2주 만에 쓸모없어져요**" | 배포 직후엔 좋았는데 빠르게 성능 저하 | 데이터 드리프트, 계절성 미반영 | Lakehouse Monitoring + 자동 재학습 파이프라인 |
| "**내 노트북에서는 됐는데...**" | 로컬에서 재현 가능하지만 프로덕션에서 실패 | 환경 차이, 라이브러리 버전, 데이터 스냅샷 불일치 | ML Runtime + MLflow 모델 로깅 (환경 패키징) |
| "**6개월 개발했는데 비즈니스 임팩트가 없어요**" | 기술적으로 완벽하지만 비즈니스 가치 없음 | 문제 정의 단계에서 이해관계자 미참여 | MVP부터 빠르게 배포하고 피드백 수집 |
| "**모델 학습에 3일 걸려서 실험을 못 해요**" | 학습 시간이 너무 길어 하이퍼파라미터 튜닝 불가 | 비효율적인 데이터 처리, 단일 머신 학습 | Spark 분산 처리 + GPU 클러스터 + 샘플링 전략 |

### 데이터 과학자 vs ML 엔지니어: 역할의 분리와 협업

ML 프로젝트에서 가장 흔한 조직 문제는 "**누가 무엇을 책임지는가?**"가 불명확한 것입니다.

| 역할 | 주요 책임 | 핵심 스킬 | Databricks 도구 |
|------|----------|----------|----------------|
| **데이터 과학자**| 문제 정의, EDA, 피처 설계, 모델 실험 | 통계, ML 알고리즘, 도메인 지식 | Notebooks, MLflow Tracking, AutoML |
| **ML 엔지니어**| 모델 프로덕션화, 파이프라인 자동화, 모니터링 | 소프트웨어 엔지니어링, MLOps, 인프라 | Model Serving, Lakeflow Jobs, Inference Tables |
| **데이터 엔지니어**| 피처 파이프라인, 데이터 품질, 인프라 | Spark, SQL, 파이프라인 설계 | Feature Engineering, SDP, Delta Lake |

> 💡 **작은 팀(5명 이하)의 현실**: 한 사람이 여러 역할을 겸합니다. 이때 Databricks의 통합 환경이 빛납니다. 한 사람이 노트북에서 실험하고, 같은 코드를 Job으로 스케줄링하고, 클릭 한 번으로 서빙하는 것이 가능하기 때문입니다.

### 실전 ML 프로젝트 체크리스트

새 ML 프로젝트를 시작할 때 확인해야 할 항목들입니다.

| 단계 | 체크 항목 | 안 하면 벌어지는 일 |
|------|----------|------------------|
| **시작 전**| 비즈니스 성공 지표 정의 ("정확도 85% 이상이면 월 $50K 절감") | 기술적으로 성공해도 프로젝트가 폐기됨 |
| **시작 전**| 충분한 레이블 데이터 존재 확인 (최소 수천 건) | 3개월 개발 후 "데이터가 부족합니다" 발견 |
| **데이터 준비**| Feature Store에 피처 등록 | 다른 팀이 같은 피처를 또 만듬 (중복 작업) |
| **학습**| MLflow autolog 활성화 | 3개월 뒤 "그때 어떤 파라미터였지?" |
| **평가**| 시간 기반 분할 (미래 예측 시뮬레이션) | 데이터 누수로 과대평가된 모델이 프로덕션에 배포됨 |
| **배포**| Shadow 배포 (기존 시스템과 병행) 최소 1주 | 새 모델의 예상치 못한 동작이 고객에게 노출 |
| **모니터링**| Inference Table + 드리프트 알림 설정 | 모델 성능 저하를 고객 불만으로 먼저 알게 됨 |

---

## 정리

| 핵심 메시지 | 설명 |
|------------|------|
| **통합 플랫폼**| 데이터 준비 → 학습 → 배포 → 모니터링을 하나의 플랫폼에서 수행합니다. 데이터 이동과 도구 전환의 오버헤드가 없습니다 |
| **전통 ML + GenAI**| 테이블 데이터 예측부터 LLM 에이전트까지 모두 지원합니다. 동일한 MLflow와 Model Serving 인프라를 공유합니다 |
| **거버넌스**| 모델, 피처, 데이터가 모두 Unity Catalog에서 통합 관리됩니다. 접근 권한, 리니지, 감사가 일원화됩니다 |
| **MLflow 중심**| 실험 추적, 모델 관리, 트레이싱, 평가를 MLflow가 담당합니다. 오픈소스 기반이므로 벤더 락인(Vendor Lock-in) 위험이 낮습니다 |
| **엔드투엔드 자동화** | Lakeflow Jobs로 데이터 파이프라인과 ML 파이프라인을 하나의 워크플로우로 자동화할 수 있습니다 |

---

## 참고 링크

- [Databricks: AI and Machine Learning](https://docs.databricks.com/aws/en/machine-learning/)
- [Azure Databricks: AI and Machine Learning](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/)
- [Databricks: MLflow Guide](https://docs.databricks.com/aws/en/mlflow/)
- [Databricks: Feature Engineering](https://docs.databricks.com/aws/en/machine-learning/feature-store/)
- [Databricks: Model Serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/)
