# 클러스터 설정

## 클러스터를 설정할 때 고려해야 할 항목들

All-Purpose 또는 Job 클러스터를 생성할 때, 워크로드의 특성에 맞게 다양한 설정을 조정할 수 있습니다. 올바른 설정은 성능을 최적화하고 불필요한 비용을 절감하는 데 직결되므로, 각 설정 항목의 의미를 정확히 이해하는 것이 중요합니다. 이 문서에서는 주요 설정 항목과 선택 가이드를 안내해 드리겠습니다.

---

## Databricks Runtime 버전

> 💡 **Databricks Runtime**은 클러스터에서 실행되는 소프트웨어 패키지입니다. Apache Spark와 함께 다양한 라이브러리, 최적화, 보안 기능이 포함되어 있습니다.

Runtime 버전은 클러스터의 기반 소프트웨어 스택을 결정합니다. 워크로드 유형에 맞는 Runtime을 선택하면 별도 라이브러리 설치 없이 바로 작업을 시작할 수 있습니다.

| Runtime 유형 | 포함 내용 | 적합한 워크로드 |
|-------------|----------|---------------|
| **Standard** | Spark + Delta Lake + 기본 라이브러리 | 일반 데이터 엔지니어링 |
| **ML Runtime** | Standard + PyTorch, TensorFlow, XGBoost, scikit-learn 등 | 머신러닝 모델 학습 |
| **Photon Runtime** | Standard + Photon 엔진 (C++ 기반 고속 처리) | SQL 분석, ETL 성능 최적화 |
| **GPU Runtime** | ML Runtime + GPU 드라이버 (CUDA, cuDNN) | 딥러닝, LLM 파인튜닝 |

### LTS vs 최신 버전

Databricks Runtime은 **LTS(Long Term Support)** 와 일반(최신) 버전으로 나뉩니다. 프로덕션 환경에서는 안정성이 검증된 LTS 버전을 사용하고, 개발/테스트에서는 최신 기능이 포함된 최신 버전을 활용하는 것이 일반적입니다.

| 버전 유형 | 지원 기간 | 적합한 환경 |
|----------|----------|-----------|
| **LTS** | 약 2년간 보안 패치 및 버그 수정 | 프로덕션, 안정성 우선 |
| **최신** | 다음 버전 출시까지 | 개발, 테스트, 신기능 활용 |

> 🆕 **최신 버전**: Databricks Runtime **18.1**이 최신 GA 버전이며, Apache Spark 4.1.0을 기반으로 합니다. 특별한 이유가 없다면 최신 LTS(Long Term Support) 버전을 선택하시는 것을 권장합니다.

---

## 노드 타입 (Node Type)

### Driver 노드와 Worker 노드

Spark 클러스터는 하나의 Driver 노드와 여러 Worker 노드로 구성됩니다. Driver는 작업을 계획하고 결과를 수집하는 역할을 하며, Worker는 실제 데이터를 처리합니다. 각 노드의 역할과 크기를 적절히 설정하는 것이 성능 최적화의 핵심입니다.

| 항목 | Driver 노드 | Worker 노드 |
|------|-------------|-------------|
| 역할 | 작업 계획, 결과 수집, Notebook 실행 | 실제 데이터 처리 (병렬) |
| 개수 | 항상 1대 | 0대 이상 (설정 가능) |
| 크기 선택 | 중간 크기 권장 (결과 수집에 충분한 메모리) | 워크로드에 따라 선택 |
| 안정성 | 반드시 On-Demand (장애 시 전체 작업 실패) | Spot 사용 가능 |

> 💡 **Single Node 클러스터**: Worker 없이 Driver만으로 구성된 클러스터입니다. 소규모 데이터 탐색, 라이브러리 테스트, 단일 머신 ML 학습 등에 적합합니다. Worker가 없으므로 분산 처리는 불가능합니다.

### 인스턴스 패밀리 선택 가이드

워크로드의 특성에 따라 적절한 인스턴스 패밀리를 선택해야 합니다. 잘못된 인스턴스 선택은 성능 저하와 비용 낭비의 원인이 됩니다.

| 인스턴스 유형 | 특징 | 적합한 워크로드 | AWS 예시 | Azure 예시 |
|-------------|------|---------------|----------|-----------|
| **범용 (General Purpose)** | CPU와 메모리 균형 | 일반 ETL, SQL 분석 | m5, m6i | Standard_D |
| **메모리 최적화 (Memory)** | 메모리 용량이 큼 | 대규모 조인, 캐싱, ML | r5, r6i | Standard_E |
| **컴퓨트 최적화 (Compute)** | CPU 성능이 높음 | CPU 집약적 변환 | c5, c6i | Standard_F |
| **스토리지 최적화 (Storage)** | 로컬 SSD가 큼 | 대용량 셔플, 스필 | i3, i4i | Standard_L |
| **GPU** | GPU 장착 | 딥러닝, LLM | p3, g4dn, p4d | Standard_NC |

> 💡 **셔플(Shuffle)이란?** `groupBy()`, `join()` 같은 연산을 할 때, 데이터가 Executor 간에 재분배되는 과정입니다. 대량의 셔플이 발생하면 네트워크와 디스크 I/O가 증가합니다. 스토리지 최적화 인스턴스는 로컬 SSD를 활용하여 셔플 성능을 향상시킵니다.

### 인스턴스 선택 의사결정 가이드

```
질문 1: "GPU가 필요한가요?" (딥러닝, LLM)
  ├─ Yes → GPU 인스턴스
  └─ No → 질문 2

질문 2: "대규모 조인이나 캐싱이 많은가요?"
  ├─ Yes → 메모리 최적화
  └─ No → 질문 3

질문 3: "셔플이 많은 복잡한 ETL인가요?"
  ├─ Yes → 스토리지 최적화
  └─ No → 범용 (기본 선택)
```

> 🆕 **Flexible Node Types**: Databricks가 최근 GA로 출시한 기능으로, 요청한 인스턴스 타입을 사용할 수 없는 경우 **호환되는 다른 인스턴스 타입으로 자동 대체**합니다. 클라우드 용량 부족으로 클러스터 시작이 실패하는 문제를 방지합니다.

---

## 오토스케일링 (Autoscaling)

> 💡 **오토스케일링(Autoscaling)** 이란 워크로드에 따라 Worker 노드의 수를 **자동으로 늘리거나 줄이는** 기능입니다.

오토스케일링은 워크로드 변동이 큰 환경에서 비용 효율성을 크게 향상시킵니다. 유휴 시에는 최소 Worker만 유지하고, 부하가 증가하면 자동으로 Worker를 추가하여 처리 속도를 높입니다.

| 상태 | Workers 수 | 설명 |
|------|-----------|------|
| 유휴 상태 | 2 Workers (최소) | 작업 없음 |
| 작업 증가 | → 8 Workers | 자동 스케일 업 |
| 피크 | → 10 Workers (최대) | 최대 부하 |
| 작업 감소 | → 2 Workers | 자동 스케일 다운 |

### 설정 방법

```
최소 Worker 수: 2
최대 Worker 수: 10
```

- **최소(Min)**: 클러스터가 유휴 상태일 때 유지하는 최소 Worker 수입니다. 0으로 설정하면 유휴 시 Worker가 모두 해제됩니다
- **최대(Max)**: 부하가 높을 때 확장할 수 있는 최대 Worker 수입니다. 비용 상한을 제어하는 역할을 합니다

### 워크로드별 권장 설정

| 워크로드 | 최소 | 최대 | 설명 |
|----------|------|------|------|
| 대화형 개발 | 1~2 | 4~8 | 탐색 시 적은 노드, 무거운 작업 시 확장합니다 |
| 정기 ETL (배치) | 고정 | 고정 | 예측 가능한 워크로드는 고정 크기가 효율적입니다 |
| 스트리밍 | 2 | 워크로드 피크에 맞춤 | 트래픽 변동에 대응합니다 |
| ML 학습 | 고정 | 고정 | 분산 학습은 노드 수 변경에 민감하므로 고정을 권장합니다 |

> 💡 **배치 작업에서 고정 크기를 권장하는 이유**: 정기 배치 작업은 처리량이 예측 가능하므로, 스케일 업/다운에 소요되는 시간 오버헤드가 오히려 비효율적일 수 있습니다. 고정 크기로 설정하면 클러스터 시작부터 최대 성능으로 처리를 시작합니다.

---

## Spot 인스턴스

> 💡 **Spot 인스턴스(Spot Instance)** 란 클라우드 제공자의 여유 용량을 활용하여 **정가의 60~90% 할인된 가격**으로 사용할 수 있는 인스턴스입니다. 단, 클라우드 제공자가 용량이 필요하면 **갑자기 회수(preemption)**할 수 있습니다.

| 비교 항목 | On-Demand (정가) | Spot (할인) |
|-----------|-----------------|------------|
| 비용 | 100% | 10~40% |
| 안정성 | 보장됨 | 회수될 수 있음 |
| 적합한 용도 | Driver, 프로덕션 | Worker, 비핵심 작업 |

### 모범 사례

- **Driver 노드**는 항상 On-Demand를 사용합니다 (중간에 회수되면 전체 작업이 실패합니다)
- **Worker 노드**는 Spot과 On-Demand를 혼합합니다 (Spot을 우선 사용하되, 일부 On-Demand를 혼합하여 안정성을 확보합니다)
- Spot이 회수되어도 Spark는 자동으로 작업을 재시도합니다
- **Spot Fallback**: Spot 인스턴스를 확보하지 못하면 자동으로 On-Demand로 대체하는 옵션을 활성화합니다

---

## Photon 엔진

> 💡 **Photon**은 Databricks가 C++로 개발한 차세대 쿼리 실행 엔진입니다. 기존 Spark SQL 엔진을 대체하여 SQL 쿼리와 DataFrame 연산을 **최대 수 배 빠르게** 실행합니다.

### Photon이 빠른 이유

| 기술 | 설명 |
|------|------|
| **네이티브 벡터 실행** | JVM 대신 C++로 CPU 명령어를 직접 실행하여 오버헤드를 줄입니다 |
| **SIMD 최적화** | CPU의 벡터 연산 명령어(AVX 등)를 활용하여 여러 데이터를 동시에 처리합니다 |
| **메모리 관리** | JVM의 가비지 컬렉션 없이 효율적으로 메모리를 관리합니다 |
| **자동 적용** | Photon이 처리할 수 있는 연산은 자동으로 Photon으로 실행되고, 그 외는 기존 Spark 엔진이 처리합니다 |

### 활성화 방법

클러스터 생성 시 **Photon 가속**을 체크하거나, Photon이 포함된 Runtime을 선택하면 됩니다. Photon은 추가 DBU가 부과되지만, 실행 시간 단축으로 총 비용이 줄어드는 경우가 많습니다.

### Photon이 효과적인 경우

| 워크로드 | 효과 |
|----------|------|
| SQL 분석 (조인, 집계, 필터) | 매우 효과적 (2~8배 성능 향상) |
| ETL (Delta Lake 읽기/쓰기) | 효과적 (1.5~3배 성능 향상) |
| Python UDF 중심 | 제한적 (Photon은 SQL/DataFrame에 최적화) |
| ML 모델 학습 | 미적용 (ML은 별도 프레임워크 사용) |

---

## Init Script (초기화 스크립트)

> 💡 **Init Script**는 클러스터가 시작될 때 **각 노드에서 자동으로 실행되는 셸 스크립트**입니다. 시스템 패키지 설치, 환경 변수 설정, 보안 구성 등에 사용합니다.

### Init Script 유형

| 유형 | 실행 시점 | 적합한 용도 |
|------|----------|-----------|
| **Cluster-scoped** | 해당 클러스터 시작 시 | 특정 클러스터에만 필요한 설정 |
| **Global** | 모든 클러스터 시작 시 | 보안 에이전트, 모니터링 도구 |

### 사용 예시

```bash
#!/bin/bash
# Init Script 예시: 시스템 패키지 설치

# OS 패키지 설치
apt-get update && apt-get install -y libmagic-dev

# Python 패키지 설치
pip install opencv-python-headless pdf2image
```

Init Script는 Unity Catalog Volume 또는 Workspace에 저장하여 참조합니다.

```
Init Script 경로: /Volumes/production/config/init-scripts/install-packages.sh
```

> ⚠️ **주의**: Init Script가 실패하면 클러스터 시작이 실패합니다. 스크립트를 배포하기 전에 반드시 테스트하고, 에러 처리 로직을 포함하세요.

---

## 환경 변수 (Environment Variables)

클러스터에 환경 변수를 설정하여 코드에서 참조할 수 있습니다. API 엔드포인트, 환경 구분값 등을 코드와 분리하여 관리할 때 유용합니다.

```
환경 변수 설정 (클러스터 설정 > Advanced > Spark 환경 변수):

ENV=production
API_ENDPOINT=https://api.company.com
REGION=ap-northeast-2
```

```python
# 코드에서 환경 변수 참조
import os
env = os.environ.get("ENV", "dev")
api_url = os.environ.get("API_ENDPOINT")
```

> ⚠️ **보안 주의**: 비밀번호나 API 키 같은 민감한 정보는 환경 변수 대신 **Databricks Secrets**를 사용하세요. 환경 변수는 클러스터 설정에 평문으로 저장되므로 보안에 취약합니다.

---

## 클러스터 태그 (Tags)

클러스터에 태그를 추가하면 **비용 추적, 리소스 분류, 정책 적용**에 활용할 수 있습니다. 특히 비용 분석 시 팀별, 프로젝트별 비용을 구분하는 데 매우 유용합니다.

```
태그 예시:
team = data-engineering
project = sales-pipeline
cost-center = DE-001
environment = production
```

태그는 클라우드 제공자의 인스턴스 태그에도 전파되므로, AWS Cost Explorer나 Azure Cost Management에서 Databricks 비용을 팀별/프로젝트별로 분석할 수 있습니다.

---

## 클러스터 풀 (Instance Pools)

> 💡 **클러스터 풀(Instance Pool)** 은 유휴 인스턴스를 미리 확보해두는 기능입니다. 클러스터가 시작되거나 스케일 아웃될 때 새 인스턴스를 프로비저닝하는 대신 **풀에서 즉시 할당**받아 시작 시간을 크게 단축합니다.

### 동작 원리

```
[풀: 5개 인스턴스 유휴 대기 중]
    │
    ├─ 클러스터 A 시작 → 풀에서 3개 할당 (즉시) → 풀 잔여: 2개
    ├─ 클러스터 B 시작 → 풀에서 2개 할당 (즉시) → 풀 잔여: 0개
    └─ 클러스터 C 시작 → 풀 비어있음 → 새 인스턴스 프로비저닝 (수 분 소요)
        └─ 동시에 풀을 보충 (Min Idle 유지)
```

### 주요 설정

| 설정 | 설명 | 권장값 |
|------|------|--------|
| **Min Idle Instances** | 풀에서 항상 유지할 최소 유휴 인스턴스 수 | 2~5 (즉시 할당 보장) |
| **Max Capacity** | 풀의 최대 인스턴스 수 | 팀 규모에 따라 설정 |
| **Instance Type** | 풀에서 관리할 인스턴스 유형 | 가장 자주 사용하는 유형 |
| **Idle Instance Auto Termination** | 유휴 인스턴스 자동 종료 시간 | 30~60분 |

### 풀 사용 시 이점

| 항목 | 풀 미사용 | 풀 사용 |
|------|----------|---------|
| 클러스터 시작 시간 | 3~7분 | 30초~1분 |
| 스케일 아웃 시간 | 2~5분 | 즉시~30초 |
| 비용 | 인스턴스 사용 시간만 | 유휴 인스턴스 비용 추가 (소량) |

---

## 자동 종료 (Auto Termination)

비용 절약을 위해 **자동 종료** 설정은 매우 중요합니다. 설정하지 않으면, 아무도 사용하지 않아도 클러스터가 계속 실행되어 비용이 발생합니다. 특히 개발/탐색용 All-Purpose 클러스터에서는 반드시 설정해야 합니다.

```
자동 종료 시간: 30분 (유휴 후)
```

- All-Purpose 클러스터에서 설정하는 것을 **강력히 권장**합니다
- 일반적으로 **30분~120분** 사이로 설정합니다
- Job 클러스터는 Job 완료 후 자동으로 종료되므로 별도 설정이 불필요합니다

### 자동 종료 권장 시간

| 용도 | 권장 자동 종료 시간 |
|------|-------------------|
| 개인 탐색/개발 | 30분 |
| 팀 공유 개발 | 60분 |
| 대화형 분석 (Power User) | 90~120분 |
| 프로덕션 All-Purpose | 설정 필수 (60분 권장) |

---

## 클러스터 정책 (Cluster Policies)

> 💡 **클러스터 정책(Cluster Policy)** 은 관리자가 클러스터 설정의 **허용 범위를 제한**하는 규칙입니다. 사용자가 과도한 리소스를 설정하거나 비효율적인 구성을 하는 것을 방지합니다.

```json
{
  "spark_version": {
    "type": "regex",
    "pattern": "15\\.[0-9]+\\.x-scala.*",
    "hidden": false
  },
  "num_workers": {
    "type": "range",
    "minValue": 1,
    "maxValue": 10,
    "defaultValue": 2
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["m5.xlarge", "m5.2xlarge", "r5.xlarge"],
    "defaultValue": "m5.xlarge"
  },
  "autotermination_minutes": {
    "type": "range",
    "minValue": 10,
    "maxValue": 120,
    "defaultValue": 30
  }
}
```

---

## 정리

| 핵심 설정 | 설명 |
|-----------|------|
| **Runtime** | Spark 엔진과 라이브러리가 포함된 소프트웨어 패키지. 최신 LTS 버전을 권장합니다 |
| **노드 타입** | 워크로드 특성(메모리, CPU, GPU)에 맞는 인스턴스를 선택합니다 |
| **오토스케일링** | 부하에 따라 Worker 수를 자동 조절합니다. 대화형 작업에 적합합니다 |
| **Spot 인스턴스** | Worker에 적용하여 비용을 크게 절감할 수 있습니다 |
| **Photon** | C++ 기반 고성능 엔진으로 SQL/DataFrame 성능을 향상시킵니다 |
| **Init Script** | 클러스터 시작 시 자동 실행되는 셸 스크립트입니다 |
| **환경 변수** | 코드와 분리된 설정값을 관리합니다. 민감 정보는 Secrets를 사용합니다 |
| **클러스터 풀** | 유휴 인스턴스를 미리 확보하여 시작 시간을 단축합니다 |
| **자동 종료** | 유휴 시 자동 종료하여 불필요한 비용을 방지합니다 |
| **클러스터 정책** | 관리자가 허용 범위를 제한하여 비용과 보안을 관리합니다 |

---

## 참고 링크

- [Databricks: Cluster configuration](https://docs.databricks.com/aws/en/compute/configure.html)
- [Databricks: Photon](https://docs.databricks.com/aws/en/compute/photon.html)
- [Databricks: Instance Pools](https://docs.databricks.com/aws/en/compute/pool-index.html)
- [Databricks: Cluster Policies](https://docs.databricks.com/aws/en/admin/clusters/policies.html)
- [Databricks: Init Scripts](https://docs.databricks.com/aws/en/init-scripts/index.html)
- [Azure Databricks: Compute configuration](https://learn.microsoft.com/en-us/azure/databricks/compute/configure)
