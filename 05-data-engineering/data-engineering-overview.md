# 데이터 엔지니어링 전체 그림

## 왜 데이터 엔지니어링이 중요한가요?

데이터 엔지니어링은 **원천 데이터를 비즈니스에서 활용할 수 있는 형태로 변환하는 전체 과정**을 의미합니다. 아무리 좋은 분석 도구나 AI 모델이 있어도, 데이터가 정확하고 시의적절하게 준비되지 않으면 아무 소용이 없습니다.

데이터 엔지니어링 파이프라인은 다음과 같은 역할을 수행합니다:

1. **수집(Ingestion)**: 다양한 소스에서 데이터를 가져옵니다
2. **변환(Transformation)**: 원시 데이터를 정제하고 비즈니스 로직을 적용합니다
3. **오케스트레이션(Orchestration)**: 수집과 변환 작업을 자동으로 스케줄링하고 관리합니다
4. **품질 관리(Quality)**: 데이터가 정확하고 완전한지 검증합니다
5. **모니터링(Monitoring)**: 파이프라인이 정상적으로 동작하는지 관찰합니다

> 💡 **비유**: 데이터 엔지니어링은 "정수 시설"과 비슷합니다. 강이나 호수에서 원수(Raw Data)를 가져와서, 여러 단계의 정화 과정(변환)을 거쳐, 가정(분석/AI)에서 바로 마실 수 있는 깨끗한 물(정제된 데이터)로 만드는 것입니다.

---

## Databricks의 데이터 엔지니어링 도구 전체 맵

Databricks에서 데이터 엔지니어링은 **Lakeflow**라는 통합 제품 브랜드 아래에서 제공됩니다. 수집부터 변환, 오케스트레이션까지 데이터 파이프라인의 전체 라이프사이클을 지원합니다.

![Databricks 플랫폼 아키텍처 개요](https://docs.databricks.com/aws/en/assets/images/architecture-c2c83d23e2f7870f30137e97aaadea0b.png)

*출처: [Databricks 공식 문서](https://docs.databricks.com/aws/en/getting-started/overview.html)*
| 단계 | 도구 | 설명 |
|------|------|------|
| 외부 데이터 소스 | DB (MySQL, PostgreSQL, Oracle), SaaS (Salesforce, HubSpot), 클라우드 파일 (S3, ADLS, GCS), 스트리밍 (Kafka, Kinesis) | 원본 데이터 발생지 |
| 수집 (Ingestion) | Lakeflow Connect, Auto Loader, COPY INTO | 데이터를 레이크하우스로 수집 |
| 저장 & 변환 | Delta Lake (Bronze → Silver → Gold), SDP (선언적 파이프라인) | Medallion 아키텍처로 정제 |
| 오케스트레이션 | Lakeflow Jobs | 워크플로우 스케줄링 및 관리 |
| 거버넌스 | Unity Catalog | 전체 데이터 자산 통합 관리 |
| 소비 | Databricks SQL, AI/BI, ML/AI, 외부 BI 도구 | 분석 및 활용 |

---

## Lakeflow 브랜드 체계

Databricks는 2024년부터 데이터 엔지니어링 관련 제품들을 **Lakeflow**라는 하나의 브랜드로 통합하였습니다. 이전에 각각 별도의 이름으로 불리던 도구들이 체계적으로 정리되었습니다.

| Lakeflow 구성 요소 | 이전 이름 / 별칭 | 역할 | 핵심 가치 |
|-------------------|-----------------|------|----------|
| **Lakeflow Connect** | (신규) | 외부 DB/SaaS에서 관리형 수집 | 코드 없이 CDC 수집 자동화 |
| **Lakeflow Declarative Pipelines (SDP)** | Delta Live Tables (DLT) | 선언적 변환 파이프라인 | "무엇을"만 정의하면 "어떻게"는 자동 |
| **Lakeflow Jobs** | Databricks Workflows | 워크플로 오케스트레이션 | 스케줄링, 의존성 관리, 모니터링 |
| **Auto Loader** | Auto Loader (변경 없음) | 클라우드 파일 증분 수집 | 새 파일 자동 감지, 스키마 진화 |

> ⚠️ **용어 참고**: Delta Live Tables(DLT)는 현재 **Spark Declarative Pipelines(SDP)** 또는 **Lakeflow Declarative Pipelines**로 명칭이 변경되었습니다. 기존 문서나 블로그에서 DLT라는 이름이 등장하면 SDP와 동일한 것으로 이해하시면 됩니다.

---

## 각 구성 요소의 역할 상세

### Lakeflow Connect — 관리형 데이터 수집

외부 데이터베이스(MySQL, PostgreSQL, Oracle)와 SaaS 애플리케이션(Salesforce, Workday, HubSpot 등)에서 **코드 작성 없이 데이터를 자동으로 수집**합니다.

- **초기 스냅샷**: 소스 테이블의 전체 데이터를 한 번 복사합니다
- **CDC(Change Data Capture)**: 이후 변경된 데이터만 실시간으로 반영합니다
- **스키마 진화**: 소스에서 컬럼이 추가/변경되면 자동으로 대상 테이블에 반영합니다

### Auto Loader — 파일 기반 증분 수집

클라우드 스토리지(S3, ADLS, GCS)에 도착하는 새 파일을 **자동으로 감지하고 증분 수집**합니다.

- **파일 알림 모드**: 클라우드 이벤트(SNS/SQS, Event Grid)를 활용하여 새 파일을 즉시 감지합니다
- **디렉토리 리스팅 모드**: 주기적으로 디렉토리를 스캔하여 새 파일을 찾습니다
- **스키마 추론/진화**: JSON, CSV 등의 포맷에서 스키마를 자동으로 추론하고 변경을 감지합니다

### SDP (Spark Declarative Pipelines) — 선언적 변환

**"무엇을 만들지"만 선언하면 실행 계획, 의존성 관리, 증분 처리를 자동으로 관리**하는 파이프라인 프레임워크입니다.

- **Streaming Table**: 실시간 증분 처리에 적합한 테이블 형태입니다
- **Materialized View**: 전체 데이터를 주기적으로 재계산하는 뷰입니다
- **Expectations**: 데이터 품질 규칙을 파이프라인에 내장하여 불량 데이터를 자동으로 감지합니다

### Lakeflow Jobs — 워크플로 오케스트레이션

다양한 태스크(노트북, Python, SQL, SDP 파이프라인 등)를 **DAG(Directed Acyclic Graph) 형태로 조합하여 스케줄링하고 모니터링**합니다.

- **다양한 트리거**: 크론 스케줄, 파일 도착, API 호출, 연속 실행 등
- **의존성 관리**: 태스크 간 실행 순서를 그래프로 정의합니다
- **재시도 및 알림**: 실패 시 자동 재시도, 이메일/Slack 알림을 지원합니다

---

## Medallion 아키텍처와의 관계

Databricks의 데이터 엔지니어링 도구들은 **Medallion 아키텍처**(Bronze → Silver → Gold)와 자연스럽게 연결됩니다.

> 💡 **Medallion 아키텍처**는 데이터를 품질과 정제 수준에 따라 3개 계층으로 분류하는 설계 패턴입니다. 자세한 내용은 [03-lakehouse-architecture](../03-lakehouse-architecture/) 섹션을 참고하시기 바랍니다.

| 계층 | 역할 | 담당 도구 | 데이터 특성 |
|------|------|----------|-----------|
| **Bronze** (원시) | 소스에서 있는 그대로 수집 | Auto Loader, Lakeflow Connect | 원본 그대로, 삭제 없음, 감사 목적 |
| **Silver** (정제) | 정제, 중복 제거, 타입 변환 | SDP (Streaming Table, APPLY CHANGES) | 정규화됨, 비즈니스 키 기준 최신화 |
| **Gold** (비즈니스) | 집계, KPI, 비즈니스 뷰 | SDP (Materialized View) | 비즈니스 목적에 최적화, 소비 가능 |

| 레이어 | 테이블 예시 | 수집 도구 | 설명 |
|--------|-----------|----------|------|
| Bronze | raw_customers (Lakeflow Connect), raw_events (Auto Loader), raw_products (COPY INTO) | Lakeflow Connect, Auto Loader, COPY INTO | 원본 데이터 보존 |
| Silver | clean_customers, clean_events, clean_products | SDP (정제 파이프라인) | 데이터 정제 및 표준화 |
| Gold | daily_revenue, customer_360, product_metrics | SDP (집계 파이프라인) | 비즈니스 메트릭 생성 |

---

## 서버리스 컴퓨트와 데이터 엔지니어링

Databricks의 데이터 엔지니어링 도구들은 **서버리스 컴퓨트(Serverless Compute)** 위에서 실행할 수 있습니다. 서버리스를 사용하면 클러스터를 직접 관리할 필요 없이 자동으로 리소스가 할당됩니다.

| 도구 | 서버리스 지원 | 이점 |
|------|:------------:|------|
| **Lakeflow Connect** | ✅ 기본값 | 항상 서버리스로 실행, 인프라 관리 불필요 |
| **SDP** | ✅ 지원 | 파이프라인 실행 시 자동 스케일링 |
| **Lakeflow Jobs** | ✅ 지원 | 잡 실행 시 빠른 시작, 자동 스케일링 |
| **Auto Loader** | ✅ 지원 (SDP/Jobs 내) | SDP 또는 Jobs의 서버리스 컴퓨트 활용 |

> 💡 **서버리스의 장점**: 클러스터 시작 대기 시간이 수초 이내로 줄어들고, 사용한 만큼만 과금되며, 인프라 관리(패치, 스케일링 등)가 완전히 자동화됩니다.

---

## 데이터 품질 관리 개요

데이터 엔지니어링에서 **데이터 품질 관리**는 선택이 아닌 필수입니다. Databricks는 여러 레벨에서 품질을 관리할 수 있는 도구를 제공합니다.

| 품질 관리 방법 | 도구 | 적용 시점 | 설명 |
|--------------|------|---------|------|
| **Expectations** | SDP | 수집/변환 시 | 파이프라인 내에서 선언적 품질 규칙 적용 |
| **Unity Catalog Monitor** | Lakehouse Monitoring | 테이블 레벨 | 데이터 프로파일링, 드리프트 감지, 이상 탐지 |
| **SQL 제약조건** | Delta Lake | 테이블 레벨 | NOT NULL, CHECK 제약조건 |
| **커스텀 검증** | Lakeflow Jobs | 파이프라인 후 | Python/SQL로 비즈니스 규칙 검증 |

```sql
-- SDP Expectations 예시: 데이터 품질 규칙 선언
CREATE OR REFRESH STREAMING TABLE silver_orders (
    CONSTRAINT valid_amount EXPECT (amount > 0) ON VIOLATION DROP ROW,
    CONSTRAINT valid_email EXPECT (email IS NOT NULL) ON VIOLATION FAIL UPDATE
)
AS SELECT * FROM STREAM(bronze_orders);
```

---

## 모니터링 및 관측성(Observability)

파이프라인을 구축한 후에는 지속적인 **모니터링**이 필요합니다. Databricks는 다음과 같은 모니터링 기능을 제공합니다.

| 모니터링 영역 | 도구/기능 | 확인 내용 |
|-------------|----------|----------|
| **파이프라인 실행 상태** | Lakeflow Jobs UI | 성공/실패, 실행 시간, 재시도 횟수 |
| **SDP 파이프라인 메트릭** | SDP Pipeline UI | 처리 건수, 지연 시간, Expectation 위반율 |
| **시스템 테이블** | `system.workflow.job_run_timeline` | 잡 실행 이력, 비용 분석 |
| **알림** | Lakeflow Jobs 알림 | 실패, SLA 위반 시 이메일/Slack/PagerDuty 알림 |
| **데이터 리니지** | Unity Catalog Lineage | 데이터 흐름 추적, 영향 분석 |

> 💡 **시스템 테이블(System Tables)** 은 Databricks 플랫폼의 운영 데이터(잡 실행 이력, 비용, 감사 로그 등)를 Delta 테이블로 제공하는 기능입니다. SQL로 직접 쿼리하여 커스텀 모니터링 대시보드를 만들 수 있습니다.

---

## 비용 최적화 전략

데이터 엔지니어링 파이프라인의 비용을 최적화하기 위한 핵심 전략들입니다.

| 전략 | 설명 | 적용 도구 |
|------|------|----------|
| **서버리스 사용** | 클러스터 대기 비용 제거, 사용한 만큼만 과금 | 모든 Lakeflow 도구 |
| **트리거 배치 모드** | 연속 실행 대신 주기적 트리거로 비용 절감 | Auto Loader, SDP |
| **증분 처리** | 전체 데이터 재처리 대신 변경분만 처리 | Auto Loader, SDP, Lakeflow Connect |
| **적절한 클러스터 크기** | 워크로드에 맞는 클러스터 사이즈 선택 | Lakeflow Jobs |
| **Photon 엔진 활용** | C++ 기반 쿼리 엔진으로 처리 속도 향상 (비용/시간 절감) | SDP, Jobs |
| **파티셔닝/Z-Order** | 불필요한 데이터 스캔 감소 | Delta Lake 테이블 |
| **Liquid Clustering** | 자동 데이터 레이아웃 최적화 | Delta Lake 테이블 |

```sql
-- Liquid Clustering으로 테이블 최적화 (자동 레이아웃)
CREATE TABLE gold_sales
CLUSTER BY (region, sale_date)
AS SELECT * FROM silver_sales_agg;
```

> ⚠️ **비용 모니터링 팁**: `system.billing.usage` 시스템 테이블을 활용하면 파이프라인별 DBU 소비량을 추적하고 비용 이상 징후를 조기에 감지할 수 있습니다.

---

## 학습 로드맵

이 섹션의 문서들을 아래 순서대로 학습하시면 데이터 엔지니어링 전체 흐름을 체계적으로 이해하실 수 있습니다.

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | 데이터 엔지니어링 전체 그림 (본 문서) | 전체 아키텍처 개요 |
| 2 | 수집 방법 선택 가이드 | Auto Loader vs COPY INTO vs Lakeflow Connect |
| 3 | Auto Loader 상세 | 파일 기반 수집 |
| 4 | Lakeflow Connect 상세 | DB/SaaS 수집 |
| 5 | SDP (선언적 파이프라인) | 데이터 변환 |
| 6 | Lakeflow Jobs | 워크플로우 오케스트레이션 |

| 순서 | 문서 | 핵심 학습 내용 |
|:----:|------|-------------|
| 1 | **본 문서** | 전체 그림 파악, 각 도구의 역할 이해 |
| 2 | [수집 방법 선택 가이드](./choosing-ingestion-method.md) | 상황별 최적 수집 방법 결정 |
| 3 | [Auto Loader](./auto-loader/) | 파일 기반 데이터 수집 마스터 |
| 4 | [Lakeflow Connect](./lakeflow-connect/) | DB/SaaS 관리형 수집 이해 |
| 5 | [SDP (선언적 파이프라인)](./spark-declarative-pipelines/) | 선언적 변환, 데이터 품질 관리 |
| 6 | [Lakeflow Jobs](./lakeflow-jobs/) | 워크플로 오케스트레이션, 스케줄링 |

> 💡 **학습 팁**: 각 도구를 개별적으로 이해한 후, 마지막에 **시나리오 기반으로 여러 도구를 조합**하는 연습을 해보시면 실무 적용력이 크게 향상됩니다.

---

## Lakeflow 브랜드 통합 전략 — 왜 이름이 바뀌었는가

Databricks는 2024년 Data+AI Summit에서 데이터 엔지니어링 제품들을 **Lakeflow**라는 단일 브랜드로 통합한다고 발표했습니다. 이 전략적 결정의 배경을 이해하면 Databricks의 제품 방향성을 파악할 수 있습니다.

### 리브랜딩 이전의 문제

| 이전 상태 | 문제점 |
|-----------|--------|
| DLT, Workflows, Auto Loader가 **별도 제품**으로 인식 | 고객이 전체 데이터 파이프라인을 하나로 이해하기 어려움 |
| 각 도구별 별도의 문서, 가격, 지원 | 도입 검토 시 비교 평가가 복잡 |
| 경쟁사(Fivetran+dbt+Airflow 스택)와 **개별 도구 단위**로 비교 | 통합 플랫폼의 장점이 부각되지 않음 |

### Lakeflow 통합의 전략적 의미

```
경쟁사 스택:              Databricks Lakeflow:
┌─────────────┐          ┌─────────────────────────────┐
│  Fivetran   │  수집     │  Lakeflow Connect            │
├─────────────┤          ├─────────────────────────────┤
│  dbt        │  변환     │  Lakeflow Declarative (SDP)  │
├─────────────┤          ├─────────────────────────────┤
│  Airflow    │  오케스트  │  Lakeflow Jobs               │
├─────────────┤          ├─────────────────────────────┤
│  Great Exp. │  품질     │  SDP Expectations (내장)      │
├─────────────┤          ├─────────────────────────────┤
│  (별도)     │  모니터링  │  시스템 테이블 + 이벤트 로그  │
└─────────────┘          └─────────────────────────────┘
```

**핵심 메시지**: 5개 이상의 벤더를 조합하는 대신, 하나의 통합 플랫폼에서 수집-변환-오케스트레이션-품질-모니터링을 모두 처리할 수 있다는 것입니다. 이는 운영 복잡도 감소, 통합 거버넌스(Unity Catalog), 비용 투명성이라는 이점으로 이어집니다.

> 💡 **SA 관점**: 고객 미팅에서 "왜 dbt 대신 SDP를 써야 하나요?" 라는 질문을 자주 받습니다. 핵심 차별점은 **런타임 통합**입니다. dbt는 변환 로직만 정의하고 실행은 별도 컴퓨트에 의존하지만, SDP는 Spark 런타임과 긴밀히 통합되어 자동 증분 처리, Enhanced Autoscaling, 서버리스 실행이 가능합니다.

---

## 파이프라인 아키텍처 패턴 심층 비교

데이터 파이프라인을 설계할 때 선택할 수 있는 대표적인 아키텍처 패턴을 비교합니다.

### Lambda 아키텍처

```
                    ┌── 배치 레이어 (전체 재처리, 정확성) ──┐
원천 데이터 ──┤                                           ├── 서빙 레이어 (통합)
                    └── 스피드 레이어 (실시간, 근사치)  ──┘
```

| 특성 | 설명 |
|------|------|
| **원리** | 동일 데이터를 배치(정확)와 실시간(빠름) 두 경로로 처리합니다 |
| **장점** | 실시간 + 정확성을 모두 제공합니다 |
| **단점** | 두 경로의 로직을 **이중 구현/유지**해야 합니다. 복잡도가 매우 높습니다 |
| **Databricks 적용** | SDP Continuous(스피드) + SDP Triggered(배치)로 구현 가능하지만, **권장하지 않습니다** |

### Kappa 아키텍처

```
원천 데이터 ── 스트리밍 레이어 (단일 경로) ── 서빙 레이어
```

| 특성 | 설명 |
|------|------|
| **원리** | 모든 데이터를 **스트리밍 파이프라인 하나**로 처리합니다 |
| **장점** | 단일 코드베이스, 낮은 복잡도 |
| **단점** | 대규모 재처리(backfill)가 어렵고, 복잡한 집계에 한계가 있습니다 |
| **Databricks 적용** | SDP Streaming Table + Continuous 모드로 자연스럽게 구현 |

### Medallion 아키텍처 (Databricks 권장)

```
원천 데이터 ── Bronze (원시) ── Silver (정제) ── Gold (비즈니스)
```

| 특성 | 설명 |
|------|------|
| **원리** | 데이터를 **품질/정제 수준에 따라 3계층**으로 분리합니다 |
| **장점** | 직관적, 디버깅 용이, 배치/스트리밍 통합 가능 |
| **단점** | 계층이 많아지면 지연 시간 증가, 스토리지 비용 |
| **Databricks 적용** | SDP의 기본 패턴. Streaming Table(Bronze/Silver) + Materialized View(Gold)로 자연스럽게 구현 |

### 아키텍처 선택 가이드

```
Lambda 아키텍처: 레거시 환경에서 마이그레이션 중인 경우에만 고려
Kappa 아키텍처:  모든 데이터가 이벤트 스트림 기반일 때 적합
Medallion:      대부분의 Databricks 프로젝트에서 권장 (배치+스트리밍 통합)
```

> 💡 **실무 팁**: Medallion 아키텍처에서 Bronze → Silver 간 지연이 문제가 되는 경우, Bronze를 건너뛰고 Silver에 직접 수집하는 **"Silver-first" 패턴**도 고려할 수 있습니다. 단, 원본 데이터 보존(감사 목적)이 필요하다면 Bronze는 유지하되 Streaming Table로 구현하여 지연을 최소화합니다.

---

## 비용 최적화 전략 심층 분석

### Serverless vs Classic 컴퓨트 비용 비교

| 비교 항목 | Serverless | Classic (On-Demand) | Classic (Spot/Preemptible) |
|-----------|-----------|-------------------|---------------------------|
| **DBU 단가** | 높음 (~1.5~2x) | 기본 | 기본 |
| **인프라 비용** | 없음 | EC2/VM 비용 추가 | EC2/VM 비용 (60~90% 할인) |
| **시작 시간** | ~10초 | 5~10분 | 5~10분 |
| **유휴 비용** | 없음 | 클러스터 대기 비용 | 클러스터 대기 비용 |
| **관리 부담** | 없음 | 클러스터 구성, 패치, 모니터링 | + Spot 중단 처리 |

### 비용 시뮬레이션: 일일 4회 배치 ETL (회당 30분)

```
Serverless:
  4회 × 0.5시간 × 20 Serverless DBU/시간 = 40 DBU/일
  클라우드 인프라 비용: 0 (Databricks 포함)
  총 비용: ~40 DBU × $0.70 = $28/일

Classic On-Demand (i3.xlarge × 4 Workers):
  클러스터 대기 10분 + 실행 30분 = 40분/회
  4회 × 40분 × 10 DBU/시간 = ~27 DBU/일
  EC2 비용: 4회 × 40분 × $1.25/시간(4노드) ≈ $3.33/일
  총 비용: ~27 DBU × $0.55 + $3.33 = $18.18/일

Classic + Spot (70% 할인):
  DBU 비용: ~$14.85/일
  EC2 비용: ~$1.00/일
  총 비용: ~$15.85/일 (단, Spot 중단 리스크)
```

> 💡 **SA 권장**: 실행 시간이 짧고(< 1시간) 간헐적인 워크로드는 **Serverless**가 유리합니다. 장시간(> 4시간) 지속 실행되는 워크로드나 비용에 매우 민감한 경우는 **Classic + Spot**을 고려합니다. Spot 중단 시 SDP의 자동 체크포인트 복구 기능이 재처리 비용을 최소화합니다.

### Spot 인스턴스 활용 전략

| 전략 | 설명 |
|------|------|
| **Driver는 On-Demand** | Driver 노드가 중단되면 전체 파이프라인이 실패하므로, Driver는 항상 On-Demand로 설정합니다 |
| **Worker는 Spot 70~90%** | Worker가 중단되어도 나머지 Worker가 태스크를 이어받습니다 |
| **Fallback to On-Demand** | Spot 용량이 부족하면 자동으로 On-Demand로 전환합니다 |
| **인스턴스 다양화** | 단일 인스턴스 타입 대신 여러 타입을 지정하여 Spot 가용성을 높입니다 |

---

## 데이터 품질 프레임워크 비교

### SDP Expectations vs Great Expectations vs dbt Tests

| 비교 항목 | SDP Expectations | Great Expectations | dbt Tests |
|-----------|-----------------|-------------------|-----------|
| **통합 수준** | 파이프라인에 **내장** | 별도 프레임워크 | dbt 프로젝트에 내장 |
| **실행 시점** | 데이터 처리와 **동시** | 별도 검증 단계 | 모델 실행 후 |
| **실시간 지원** | 스트리밍 데이터에 적용 가능 | 배치 전용 | 배치 전용 |
| **위반 시 동작** | DROP ROW / FAIL UPDATE / 로깅 | 리포트 생성 | 경고 / 에러 |
| **거버넌스** | Unity Catalog + 이벤트 로그 통합 | 별도 메타데이터 저장소 | dbt Cloud 대시보드 |
| **학습 곡선** | 낮음 (SQL 제약조건과 유사) | 높음 (자체 DSL) | 중간 (YAML/SQL) |
| **커스텀 규칙** | SQL 표현식 | Python 커스텀 Expectation | SQL/Jinja |

### 데이터 품질 관리 성숙도 모델

```
Level 1 — 기본: NOT NULL, 타입 체크 (SDP Expectations)
Level 2 — 규칙 기반: 비즈니스 규칙 검증 (Expectations + CHECK Constraints)
Level 3 — 통계 기반: 드리프트 감지, 이상 탐지 (Lakehouse Monitoring)
Level 4 — 자동화: 이상 시 자동 알림 + 파이프라인 중단 (모니터링 + 알림 통합)
Level 5 — 예측: 품질 저하 예측 및 선제적 대응 (ML 기반 이상 탐지)
```

> 💡 **엔터프라이즈 패턴**: 대부분의 고객은 Level 2(SDP Expectations)에서 시작하여 Level 3(Lakehouse Monitoring)으로 확장합니다. Great Expectations는 이미 도입된 경우에 병행 사용하되, 신규 프로젝트에서는 SDP Expectations + Lakehouse Monitoring 조합을 권장합니다.

---

## 엔터프라이즈 데이터 엔지니어링 패턴

### 환경 분리 (Dev/Staging/Prod)

```
Dev 환경:
  ├── dev_catalog.schema → 개발자별 스키마
  ├── SDP Development Mode (실패 시 클러스터 유지)
  └── 소규모 샘플 데이터

Staging 환경:
  ├── staging_catalog.schema → 통합 테스트
  ├── SDP Triggered Mode (프로덕션 동일 설정)
  └── 프로덕션 유사 데이터 볼륨

Production 환경:
  ├── prod_catalog.schema → 프로덕션 데이터
  ├── SDP Triggered/Continuous (비즈니스 요구에 따라)
  └── 모니터링 + 알림 활성화
```

### CI/CD 파이프라인 통합

```yaml
# Asset Bundles를 활용한 SDP 배포 (databricks.yml)
environments:
  dev:
    workspace:
      host: https://dev-workspace.cloud.databricks.com
    resources:
      pipelines:
        my_pipeline:
          development: true
          catalog: dev_catalog

  prod:
    workspace:
      host: https://prod-workspace.cloud.databricks.com
    resources:
      pipelines:
        my_pipeline:
          development: false
          catalog: prod_catalog
          serverless: true
```

---

## 실전 파이프라인 오케스트레이션 패턴

### 패턴 1: Lakeflow Jobs로 End-to-End 파이프라인 오케스트레이션

Lakeflow Jobs는 **여러 종류의 태스크를 DAG(방향성 비순환 그래프)로 조합** 하여 전체 파이프라인을 하나의 워크플로로 관리합니다. 수집 → 변환 → 품질 검증 → 서빙까지 하나의 Job으로 구성할 수 있습니다.

```
[Job: daily-data-pipeline]
│
├─ Task 1: 수집 (SDP Pipeline — Lakeflow Connect CDC)
│   └─ Bronze 테이블에 원본 데이터 적재
│
├─ Task 2: 변환 (SDP Pipeline — Bronze → Silver → Gold)
│   ├─ Silver: 정제, 중복 제거, SCD Type 1/2
│   └─ Gold: 비즈니스 집계
│
├─ Task 3: 품질 검증 (SQL Task)
│   └─ Gold 테이블의 행 수, NULL 비율, 전일 대비 변화량 체크
│
├─ Task 4-A: 성공 시 → ML 피처 갱신 (Notebook Task)
│   └─ Feature Table 업데이트 → Online Table 동기화
│
├─ Task 4-B: 성공 시 → 대시보드 새로고침 (SQL Task)
│   └─ REFRESH MATERIALIZED VIEW gold_daily_kpi
│
└─ Task 5: 실패 시 → Slack 알림 (Webhook)
```

```python
# Lakeflow Jobs SDK로 위 파이프라인 생성
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import *

w = WorkspaceClient()

w.jobs.create(
    name="daily-data-pipeline",
    tasks=[
        Task(
            task_key="ingest",
            pipeline_task=PipelineTask(pipeline_id="<ingest-pipeline-id>"),
        ),
        Task(
            task_key="transform",
            depends_on=[TaskDependency(task_key="ingest")],
            pipeline_task=PipelineTask(pipeline_id="<sdp-pipeline-id>"),
        ),
        Task(
            task_key="quality_check",
            depends_on=[TaskDependency(task_key="transform")],
            sql_task=SqlTask(
                query=SqlTaskQuery(query_id="<quality-check-query-id>"),
                warehouse_id="<warehouse-id>"
            ),
        ),
        Task(
            task_key="update_features",
            depends_on=[TaskDependency(task_key="quality_check")],
            notebook_task=NotebookTask(
                notebook_path="/Workspace/pipelines/update_features"
            ),
        ),
        Task(
            task_key="refresh_dashboard",
            depends_on=[TaskDependency(task_key="quality_check")],
            sql_task=SqlTask(
                query=SqlTaskQuery(query_id="<refresh-mv-query-id>"),
                warehouse_id="<warehouse-id>"
            ),
        ),
    ],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 6 * * ?",  # 매일 06:00
        timezone_id="Asia/Seoul"
    ),
    email_notifications=JobEmailNotifications(
        on_failure=["data-team@company.com"]
    )
)
```

### 패턴 2: CDC 파이프라인 — SCD Type 1 vs Type 2

CDC(Change Data Capture) 데이터를 처리할 때 가장 중요한 결정은 **SCD(Slowly Changing Dimension) 전략**입니다.

#### SCD Type 1: 최신 값으로 덮어쓰기

변경 이력을 보존하지 않고, **항상 최신 값만 유지**합니다.

```sql
-- SDP에서 SCD Type 1 구현
CREATE OR REFRESH STREAMING TABLE silver_customers;

APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_customers)
KEYS (customer_id)
SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 1;
```

| 항목 | 설명 |
|------|------|
| **결과 테이블** | customer_id당 항상 1개 행 (최신) |
| **이전 값** | 삭제됨 (복구 불가, Delta Time Travel로만 가능) |
| **적합한 경우** | 최신 상태만 필요 (주소, 이메일, 전화번호 등) |
| **스토리지** | 효율적 (행 수 = 고유 키 수) |

#### SCD Type 2: 변경 이력 보존

모든 변경 이력을 **별도 행으로 보존**합니다. `__START_AT`, `__END_AT` 컬럼으로 유효 기간을 관리합니다.

```sql
-- SDP에서 SCD Type 2 구현
CREATE OR REFRESH STREAMING TABLE silver_customers_history;

APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_customers)
KEYS (customer_id)
SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 2;

-- 결과 테이블 구조:
-- | customer_id | name   | city  | __START_AT     | __END_AT       |
-- |-------------|--------|-------|----------------|----------------|
-- | 1001        | 김철수 | 서울  | 2025-01-01     | 2025-03-15     |
-- | 1001        | 김철수 | 부산  | 2025-03-15     | NULL           | ← 현재 값
```

| 항목 | 설명 |
|------|------|
| **결과 테이블** | customer_id당 여러 행 (변경 이력) |
| **현재 값 조회** | `WHERE __END_AT IS NULL` |
| **특정 시점 조회** | `WHERE __START_AT <= '2025-02-01' AND (__END_AT > '2025-02-01' OR __END_AT IS NULL)` |
| **적합한 경우** | 이력 추적 필수 (고객 등급 변화, 가격 변동, 규제 감사) |
| **스토리지** | 변경 빈도에 비례하여 증가 |

#### SCD Type 1 vs Type 2 선택 기준

| 기준 | Type 1 | Type 2 |
|------|--------|--------|
| **"이전 값이 필요한가?"** | 아니오 → Type 1 | 예 → Type 2 |
| **규제 감사 요건** | 없음 → Type 1 | 있음 (변경 이력 보존) → Type 2 |
| **분석 요구** | 현재 상태 분석 → Type 1 | 시간에 따른 변화 분석 → Type 2 |
| **스토리지 비용** | 낮음 | 높음 (이력 누적) |
| **쿼리 복잡도** | 단순 (항상 최신) | 복잡 (`__START_AT`/`__END_AT` 필터) |
| **대표 사용** | 고객 연락처, 제품 정보 | 고객 등급, 가격 이력, 재고 변동 |

#### 실전: 하이브리드 전략 (Type 1 + Type 2 동시 운영)

대부분의 프로덕션 환경에서는 **같은 소스에서 Type 1과 Type 2를 동시에 생성**합니다.

```sql
-- 같은 Bronze 소스에서 두 가지 Silver 테이블 생성

-- Silver 1: 현재 상태 (조인용, 대시보드용) — SCD Type 1
CREATE OR REFRESH STREAMING TABLE silver_customers_current;
APPLY CHANGES INTO silver_customers_current
FROM STREAM(bronze_customers)
KEYS (customer_id) SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 1;

-- Silver 2: 전체 이력 (감사, 시계열 분석용) — SCD Type 2
CREATE OR REFRESH STREAMING TABLE silver_customers_history;
APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_customers)
KEYS (customer_id) SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 2;

-- Gold: 현재 상태 기반 집계 (Type 1에서 읽음 → 빠른 조인)
CREATE OR REFRESH MATERIALIZED VIEW gold_customer_revenue AS
SELECT c.customer_id, c.name, c.city,
       COUNT(o.order_id) AS total_orders,
       SUM(o.amount) AS total_revenue
FROM silver_customers_current c
JOIN silver_orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city;

-- 감사 리포트: 이력 기반 조회 (Type 2에서 읽음)
-- "2025년 1분기에 서울에 거주했던 고객 목록"
SELECT * FROM silver_customers_history
WHERE city = '서울'
  AND __START_AT <= '2025-03-31'
  AND (__END_AT > '2025-01-01' OR __END_AT IS NULL);
```

> 💡 **왜 하이브리드인가?** Type 2만 사용하면 Gold 계층의 JOIN이 복잡해집니다 (`__END_AT IS NULL` 조건이 매번 필요). Type 1의 "현재 상태" 테이블을 별도로 유지하면 **Gold 집계가 단순하고 빠릅니다.**

### 패턴 3: CDC → CDF → 다운스트림 전파

외부 DB의 CDC를 수집하고, Delta CDF(Change Data Feed)로 다운스트림에 전파하는 전체 흐름입니다.

```
[외부 MySQL]
    │ Lakeflow Connect (CDC — binlog)
    ▼
[Bronze: bronze_customers]  ← 원본 CDC 이벤트 보존
    │ SDP: APPLY CHANGES INTO
    ▼
[Silver: silver_customers]  ← SCD Type 1 (CDF 활성화)
    │ Delta CDF (readChangeFeed)
    ├──▶ [Online Table] ← 실시간 Feature Serving
    ├──▶ [Gold: gold_customer_360] ← MV 증분 갱신
    └──▶ [외부 시스템] ← Reverse ETL (foreachBatch)
```

```sql
-- Silver 테이블에 CDF 활성화 (다운스트림 전파용)
ALTER TABLE silver_customers
SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');
```

```python
# Silver의 변경사항을 외부 시스템에 Reverse ETL
def push_to_crm(batch_df, batch_id):
    updates = batch_df.filter("_change_type IN ('insert', 'update_postimage')")
    # CRM API 호출로 고객 정보 동기화
    for row in updates.collect():
        crm_api.update_customer(row.customer_id, row.name, row.email)

(spark.readStream
    .format("delta")
    .option("readChangeFeed", "true")
    .table("silver_customers")
    .writeStream
    .foreachBatch(push_to_crm)
    .option("checkpointLocation", "/checkpoints/reverse-etl")
    .trigger(availableNow=True)
    .start()
)
```

### 파이프라인 오케스트레이션 모범 사례

| # | 원칙 | 설명 |
|---|------|------|
| 1 | **수집과 변환을 분리** | Lakeflow Connect(수집)과 SDP(변환)를 **별도 파이프라인**으로 구성합니다. 수집이 실패해도 변환은 마지막 성공 데이터로 동작합니다 |
| 2 | **Job으로 전체 오케스트레이션** | 수집 파이프라인 → SDP 파이프라인 → 품질 검증 → 후처리를 하나의 Lakeflow Job DAG로 묶습니다 |
| 3 | **품질 게이트** | Silver → Gold 사이에 품질 검증 태스크를 삽입합니다. 실패 시 Gold 갱신을 차단합니다 |
| 4 | **멱등성 보장** | 모든 태스크는 재실행해도 같은 결과를 보장해야 합니다. MERGE, APPLY CHANGES는 기본적으로 멱등적입니다 |
| 5 | **환경별 분리** | Dev/Staging/Prod에 동일한 파이프라인을 Declarative Automation Bundles로 배포합니다 |
| 6 | **SCD 전략은 비즈니스 요구로 결정** | 이력이 필요하면 Type 2, 최신값만 필요하면 Type 1. 대부분 하이브리드 |
| 7 | **CDF로 다운스트림 연결** | Silver 테이블에 CDF를 활성화하면 Online Table, MV, Reverse ETL이 증분으로 동작합니다 |

---

## 정리

Databricks의 데이터 엔지니어링 도구들은 **Lakeflow** 브랜드 아래 체계적으로 구성되어 있습니다:

- **Lakeflow Connect**: 외부 DB/SaaS에서 코드 없이 관리형 수집
- **Auto Loader**: 클라우드 스토리지 파일 자동 증분 수집
- **SDP (Spark Declarative Pipelines)**: 선언적 변환 파이프라인 + 데이터 품질
- **Lakeflow Jobs**: 전체 워크플로 오케스트레이션 및 스케줄링

이 도구들을 Medallion 아키텍처와 결합하면, 원시 데이터부터 비즈니스 인사이트까지 **안정적이고 확장 가능한 데이터 파이프라인**을 구축할 수 있습니다.

---

## 참고 링크

- [Databricks: Data Engineering](https://docs.databricks.com/aws/en/data-engineering)
- [Databricks: Lakeflow Product Page](https://www.databricks.com/product/data-engineering)
- [Databricks: Serverless Compute](https://docs.databricks.com/aws/en/compute/serverless/)
- [Databricks: Lakehouse Monitoring](https://docs.databricks.com/aws/en/lakehouse-monitoring/)
- [Databricks: System Tables](https://docs.databricks.com/aws/en/administration-guide/system-tables/)
- [Databricks Blog: Introducing Lakeflow](https://www.databricks.com/blog/introducing-databricks-lakeflow)
