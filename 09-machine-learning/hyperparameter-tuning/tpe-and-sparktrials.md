# TPE 알고리즘과 SparkTrials 심화

## TPE(Tree-structured Parzen Estimator) 알고리즘 원리

Hyperopt의 기본 알고리즘인 **TPE** 는 Bayesian Optimization의 일종으로, 일반적인 Gaussian Process 기반 방법보다 고차원 공간에서 더 효율적입니다.

### TPE의 핵심 아이디어

일반적인 Bayesian Optimization은 `P(y|x)` (파라미터 x가 주어졌을 때 성능 y의 확률)를 모델링하지만, TPE는 **역방향** 으로 `P(x|y)` (성능 y가 주어졌을 때 파라미터 x의 확률)를 모델링합니다.

```
TPE 동작 과정:

1. 초기 시행 (랜덤): 10~20회 랜덤 샘플링으로 기본 데이터 수집

2. 결과를 두 그룹으로 분리:
   - l(x): "좋은" 결과를 낸 파라미터 분포 (상위 γ%, 기본 γ=25%)
   - g(x): "나쁜" 결과를 낸 파라미터 분포 (나머지 75%)

3. EI(Expected Improvement) 최대화:
   - 다음 시도할 파라미터 = l(x) / g(x) 비율이 높은 지점
   - 즉, "좋은 결과에서는 자주 나타나고, 나쁜 결과에서는 드문" 파라미터를 선택

4. 시행 후 결과를 두 그룹에 반영하고 2~3 반복
```

### 왜 TPE가 효과적인가

| 특성 | Gaussian Process (GP) | TPE |
|------|----------------------|-----|
| **고차원 확장성**| 차원이 높으면 성능 저하 | 차원에 강건 |
| **조건부 파라미터**| 처리 어려움 | 자연스럽게 지원 |
| **범주형 파라미터**| 인코딩 필요 | 직접 지원 (`hp.choice`) |
| **계산 비용**| O(n^3) — 시행 수 증가 시 느림 | O(n log n) — 빠름 |
| **병렬화**| 어려움 | 비교적 용이 |

### TPE의 한계

| 한계 | 설명 |
|------|------|
| **탐색-활용 균형**| γ 값이 너무 작으면 과도한 탐색(exploration), 너무 크면 조기 수렴 |
| **상관관계 무시**| 파라미터 간 상호작용을 직접 모델링하지 않음 |
| **초기 시행 의존**| 초기 랜덤 시행이 편향되면 이후 탐색도 편향될 수 있음 |

---

## SparkTrials 분산 동작 구조

SparkTrials는 Spark 클러스터의 Worker 노드를 활용하여 여러 하이퍼파라미터 조합을 동시에 평가합니다.

### 내부 아키텍처

| 노드 | 역할 | 예시 |
|------|------|------|
| **Driver Node**(Hyperopt 컨트롤러) | TPE 알고리즘 실행, Trial 큐 관리, MLflow 결과 로깅 | 다음 시행할 파라미터 선택 |
| Worker Node 1 | Trial #1 | max_depth=5, lr=0.01 → F1=0.85 |
| Worker Node 2 | Trial #2 | max_depth=7, lr=0.03 → F1=0.87 |
| Worker Node 3 | Trial #3 | max_depth=3, lr=0.1 → F1=0.82 |
| Worker Node 4 | Trial #4 | max_depth=9, lr=0.05 → F1=0.86 |
| Worker Node N | Trial #N | 동시 실행 |

### parallelism과 Bayesian Optimization의 트레이드오프

```
parallelism = 1:  (순차 실행)
  Trial 1 완료 → TPE가 결과 반영 → Trial 2 선택 → 완료 → ...
  장점: 이전 결과를 100% 반영하여 가장 효율적인 탐색
  단점: 매우 느림

parallelism = max_evals/2:  (높은 병렬)
  Trial 1~50 동시 시작 (TPE가 반영할 이전 결과가 거의 없음)
  장점: 빠른 완료
  단점: Random Search에 가까워짐

parallelism = sqrt(max_evals):  (권장 균형점)
  예: max_evals=100 → parallelism=10
  적절한 이전 결과 반영 + 합리적 실행 시간
```

> 💡 **실무 권장**: `parallelism`은 **Worker 노드 수** 와 **max_evals의 제곱근** 중 작은 값을 선택합니다. 예를 들어 Worker 8대, max_evals=100이면 `parallelism=8`이 적절합니다.

### SparkTrials 주의사항

| 주의사항 | 설명 |
|----------|------|
| **메모리**| 각 Worker에서 모델을 학습하므로, Worker 메모리가 충분해야 합니다 |
| **데이터 복제**| 학습 데이터가 각 Worker로 복제됩니다. 대규모 데이터는 브로드캐스트 변수 활용 |
| **GPU**| SparkTrials는 GPU를 Worker당 1개씩 할당합니다. GPU 모델 학습 시 유용 |
| **에러 처리**| 개별 Trial 실패 시 해당 Trial만 건너뛰고 나머지 계속 실행 |
| **체크포인트** | `SparkTrials`는 체크포인트를 지원하지 않으므로, 실행 중 클러스터가 중단되면 처음부터 재실행 |

---
