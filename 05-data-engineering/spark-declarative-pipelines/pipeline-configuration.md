# SDP 파이프라인 설정 상세

## 왜 파이프라인 설정이 중요한가?

SDP(Spark Declarative Pipelines, 구 Delta Live Tables)는 파이프라인의 **실행 모드, 클러스터 구성, 타겟 카탈로그, 채널** 등을 설정을 통해 제어합니다. 올바른 설정은 개발 생산성, 프로덕션 안정성, 비용 효율성에 직접적인 영향을 미칩니다.

---

## Development vs Production 모드

SDP는 두 가지 실행 모드를 제공하며, 각 모드는 파이프라인의 동작 방식이 크게 다릅니다.

| 비교 항목 | Development 모드 | Production 모드 |
|-----------|-----------------|----------------|
| **데이터 재처리** | 전체 데이터가 아닌 **2개 파일만 샘플** 처리합니다 | **전체 데이터**를 처리합니다 |
| **클러스터 재사용** | 실행 사이에 클러스터를 유지합니다 (빠른 반복) | 매 실행마다 클러스터를 새로 생성합니다 |
| **오류 처리** | 오류 발생 시 파이프라인을 즉시 중단합니다 | 재시도 정책에 따라 자동 복구를 시도합니다 |
| **비용** | 클러스터 유지 비용 발생 | 실행 시에만 비용 발생 |
| **적합한 용도** | 개발, 디버깅, 테스트 | 프로덕션 운영 |

| 모드 | 특징 | 설명 |
|------|------|------|
| **Development 모드** | 샘플 데이터, 클러스터 유지, 빠른 피드백 | 개발 및 테스트에 적합합니다 |
| **Production 모드** | 전체 데이터, 클러스터 자동 관리, 재시도 정책 | 프로덕션 운영에 적합합니다 |

> Development 모드에서 검증을 완료한 후 Production 모드로 전환합니다.

> 🔥 **현업 사례: Development 모드와 Production 모드를 혼동해서 데이터를 날린 경험**
>
> 한 고객사의 데이터 엔지니어가 프로덕션 파이프라인을 수정한 후, **Development 모드인 줄 모르고 "Start" 버튼**을 눌렀습니다. Development 모드에서는 기존 테이블을 삭제하고 샘플 데이터만으로 재생성하기 때문에, **프로덕션 Gold 테이블 3개의 데이터가 샘플 2개 파일 분량으로 대체**되었습니다. 마침 BI 대시보드가 이 테이블을 참조하고 있었고, 경영진이 "매출이 갑자기 99% 줄었다"고 보고받는 사태가 벌어졌습니다.
>
> **복구 과정**: Delta의 Time Travel 기능(`RESTORE TABLE ... TO VERSION AS OF`)으로 데이터를 복구했지만, 원인 파악과 복구에 약 2시간이 소요되었습니다.
>
> **재발 방지책**:
> 1. **프로덕션 파이프라인은 반드시 `"development": false`로 고정** (UI에서 변경 불가하도록)
> 2. **개발용과 프로덕션용 파이프라인을 물리적으로 분리** (같은 파이프라인에서 모드 전환 금지)
> 3. **프로덕션 파이프라인은 Asset Bundles로만 배포** (UI 직접 수정 금지)

### 안전한 개발-프로덕션 분리 패턴

현업에서 권장하는 패턴은 **같은 코드, 다른 파이프라인**입니다.

```yaml
# Asset Bundles로 환경별 파이프라인 자동 분리
# bundle.yml
targets:
  dev:
    default: true
    resources:
      pipelines:
        etl:
          name: "etl-pipeline-DEV"
          catalog: "dev_catalog"
          target: "ecommerce"
          development: true     # ← 개발 모드

  prod:
    resources:
      pipelines:
        etl:
          name: "etl-pipeline-PROD"
          catalog: "production"
          target: "ecommerce"
          development: false    # ← 프로덕션 모드
```

이렇게 하면 `databricks bundle deploy -t dev`와 `databricks bundle deploy -t prod`로 완전히 분리된 파이프라인이 생성되어, 모드 혼동 사고를 원천 차단할 수 있습니다.

---

## 클러스터 설정

### 서버리스 컴퓨트

서버리스를 선택하면 클러스터 구성을 직접 관리할 필요가 없습니다. Databricks가 자동으로 인프라를 프로비저닝합니다.

```json
{
    "name": "my-serverless-pipeline",
    "serverless": true,
    "catalog": "catalog",
    "target": "schema"
}
```

| 장점 | 고려사항 |
|------|---------|
| 클러스터 시작 대기 시간 최소화 | 커스텀 라이브러리 설치 제한 |
| 인프라 관리 불필요 | 특정 인스턴스 타입 지정 불가 |
| 자동 스케일링 | 일부 워크로드에서 비용이 다를 수 있음 |

### Job Cluster (클래식)

세밀한 클러스터 제어가 필요한 경우 직접 클러스터를 구성합니다.

```json
{
    "name": "my-classic-pipeline",
    "clusters": [
        {
            "label": "default",
            "num_workers": 4,
            "node_type_id": "i3.xlarge",
            "spark_conf": {
                "spark.databricks.delta.optimizeWrite.enabled": "true"
            },
            "aws_attributes": {
                "availability": "SPOT_WITH_FALLBACK",
                "first_on_demand": 1
            }
        },
        {
            "label": "maintenance",
            "num_workers": 2,
            "node_type_id": "m5.xlarge"
        }
    ]
}
```

| 클러스터 레이블 | 용도 |
|---------------|------|
| `default` | 파이프라인의 기본 처리에 사용됩니다 |
| `maintenance` | OPTIMIZE, VACUUM 같은 유지보수 작업에 사용됩니다 |

### Auto Scaling 설정

```json
{
    "clusters": [
        {
            "label": "default",
            "autoscale": {
                "min_workers": 2,
                "max_workers": 10,
                "mode": "ENHANCED"
            }
        }
    ]
}
```

### Enhanced Autoscaling의 실전 효과

SDP의 **Enhanced Autoscaling**은 일반 클러스터의 Autoscaling과 다릅니다. 파이프라인의 데이터 흐름 단계를 이해하고, **각 단계에서 필요한 만큼만 노드를 할당**합니다.

| 비교 항목 | 일반 Autoscaling | Enhanced Autoscaling (SDP) |
|-----------|:----------------:|:-------------------------:|
| **스케일 기준** | CPU/메모리 사용률 | 데이터 흐름 단계별 워크로드 |
| **스케일 속도** | 느림 (메트릭 수집 후 판단) | 빠름 (파이프라인 그래프 분석) |
| **과잉 프로비저닝** | 빈번함 | 최소화 |
| **실전 비용 절감** | 10~20% | **30~50%** |

> 💡 **현업 팁**: Enhanced Autoscaling의 효과가 가장 극적인 경우는 **"Bronze에서 대량 수집 → Silver에서 소량 변환 → Gold에서 집계"** 패턴입니다. Bronze 단계에서 10개 노드가 필요해도, Gold 단계에서는 2개면 충분합니다. Enhanced Autoscaling은 이 차이를 자동으로 감지하여 노드를 줄입니다.

```
[Bronze 수집]     ████████████████████  10 nodes
[Silver 변환]     ████████████          6 nodes
[Gold 집계]       ████                  2 nodes

→ 일반 Autoscaling: 전 구간 10 nodes 유지 → 비용 낭비
→ Enhanced: 단계별 최적화 → 약 40% 절감
```

---

## Channel (채널) 설정

채널은 SDP 런타임의 **버전**을 결정합니다. 신기능 테스트와 프로덕션 안정성 사이의 균형을 맞출 수 있습니다.

| 채널 | 설명 | 권장 용도 |
|------|------|----------|
| **Current** | 최신 안정 버전의 SDP 런타임입니다 | 프로덕션 환경 |
| **Preview** | 차기 버전의 기능을 미리 사용할 수 있습니다 | 개발, 신기능 테스트 |

```json
{
    "name": "my-pipeline",
    "channel": "CURRENT"
}
```

---

## 타겟 카탈로그/스키마 설정

### Unity Catalog 통합

SDP 파이프라인의 결과 테이블을 Unity Catalog에 저장합니다.

```json
{
    "name": "sales-pipeline",
    "catalog": "production",
    "target": "sales_data"
}
```

| 설정 | 설명 |
|------|------|
| `catalog` | 결과 테이블이 저장될 Unity Catalog 카탈로그입니다 |
| `target` | 결과 테이블이 저장될 스키마(데이터베이스)입니다 |

위 설정으로 파이프라인 내에서 `CREATE STREAMING TABLE orders`를 실행하면 `production.sales_data.orders`로 생성됩니다.

> 💡 **카탈로그/스키마를 파이프라인 레벨에서 설정하면**, 개별 테이블 정의에서 3단계 네임스페이스를 매번 작성할 필요가 없습니다.

---

## 파이프라인 설정 전체 예제

### JSON 설정

```json
{
    "name": "ecommerce-etl-pipeline",
    "catalog": "production",
    "target": "ecommerce",
    "serverless": true,
    "channel": "CURRENT",
    "development": false,
    "continuous": false,
    "photon": true,
    "edition": "ADVANCED",
    "configuration": {
        "source_path": "s3://raw-data/ecommerce/",
        "quality_threshold": "0.95",
        "environment": "production"
    },
    "libraries": [
        {
            "notebook": {
                "path": "/Workspace/pipelines/ecommerce/bronze"
            }
        },
        {
            "notebook": {
                "path": "/Workspace/pipelines/ecommerce/silver"
            }
        },
        {
            "notebook": {
                "path": "/Workspace/pipelines/ecommerce/gold"
            }
        }
    ],
    "notifications": [
        {
            "email_recipients": ["data-team@company.com"],
            "alerts": {
                "on_update_failure": true,
                "on_update_success": false,
                "on_flow_failure": true
            }
        }
    ]
}
```

### YAML 설정 (Asset Bundles)

```yaml
resources:
  pipelines:
    ecommerce_etl:
      name: "ecommerce-etl-pipeline"
      catalog: "${var.catalog}"
      target: "${var.schema}"
      serverless: true
      channel: "CURRENT"
      development: ${var.is_dev}
      photon: true
      edition: "ADVANCED"

      configuration:
        source_path: "${var.source_path}"
        quality_threshold: "0.95"
        environment: "${bundle.target}"

      libraries:
        - notebook:
            path: "./pipelines/bronze.sql"
        - notebook:
            path: "./pipelines/silver.py"
        - notebook:
            path: "./pipelines/gold.sql"

      notifications:
        - email_recipients:
            - "data-team@company.com"
          alerts:
            on_update_failure: true
            on_flow_failure: true
```

---

## 주요 설정 옵션 총정리

| 설정 | 설명 | 기본값 |
|------|------|--------|
| `name` | 파이프라인 이름입니다 | (필수) |
| `catalog` | Unity Catalog 카탈로그입니다 | — |
| `target` | 타겟 스키마(데이터베이스)입니다 | — |
| `serverless` | 서버리스 컴퓨트 사용 여부입니다 | `false` |
| `channel` | 런타임 채널입니다 (CURRENT/PREVIEW) | `CURRENT` |
| `development` | 개발 모드 활성화 여부입니다 | `true` |
| `continuous` | 연속 실행 모드 여부입니다 | `false` |
| `photon` | Photon 엔진 활성화 여부입니다 | `false` |
| `edition` | 에디션입니다 (CORE/PRO/ADVANCED) | `ADVANCED` |
| `configuration` | 파이프라인에서 사용할 키-값 매개변수입니다 | `{}` |
| `libraries` | 파이프라인에 포함할 노트북/파일 목록입니다 | (필수) |

### configuration 매개변수 활용

`configuration`에 정의한 값은 파이프라인 코드에서 `spark.conf.get()`으로 접근할 수 있습니다.

```python
# Python에서 파이프라인 설정값 읽기
source_path = spark.conf.get("source_path")
quality_threshold = float(spark.conf.get("quality_threshold"))
```

```sql
-- SQL에서 파이프라인 설정값 읽기
SELECT * FROM STREAM read_files(
    '${source_path}',
    format => 'json'
);
```

---

## Continuous 모드 vs Triggered 모드

| 모드 | 동작 | 적합한 시나리오 |
|------|------|--------------|
| **Triggered** (기본값) | 한 번 실행하고 종료합니다. 새 데이터가 있으면 처리하고 멈춥니다 | 배치 ETL, 스케줄 실행 |
| **Continuous** | 항상 실행 상태를 유지하며, 새 데이터가 도착하면 즉시 처리합니다 | 실시간 스트리밍, 저지연 요구 |

> ⚠️ **Continuous 모드는 항상 클러스터가 실행 중이므로 비용이 지속적으로 발생합니다.** 실시간 요구사항이 없다면 Triggered 모드 + 스케줄링을 권장합니다.

### 실전 비용 비교: Continuous vs Triggered

현업에서 가장 많이 묻는 질문이 "Continuous 모드 비용이 얼마나 되나요?"입니다.

| 모드 | 24시간 기준 비용 (4 Worker, i3.xlarge) | 월 비용 예상 | 데이터 지연 |
|------|:--------------------------------------:|:-----------:|:----------:|
| **Continuous** | 24시간 × $X DBU = ~$50/일 | **~$1,500/월** | **수초~수분** |
| **Triggered (1시간 간격)** | 24회 × 10분 = 4시간 × $X DBU = ~$8/일 | **~$250/월** | **최대 1시간** |
| **Triggered (15분 간격)** | 96회 × 10분 = 16시간 × $X DBU = ~$33/일 | **~$1,000/월** | **최대 15분** |

> 💡 **현업 팁**: 대부분의 내부 분석/BI 용도에서는 **15분~1시간 지연이 허용**됩니다. "실시간이 필요하다"고 요청하는 경우에도, 실제로 확인해 보면 "1시간 이내면 충분"인 경우가 80% 이상입니다. **Continuous 모드는 정말 분 단위 지연이 비즈니스에 직접적인 영향을 미치는 경우(사기 탐지, 실시간 추천 등)에만 사용하세요.**

---

## 서버리스 SDP의 비용 예측 방법

서버리스 SDP는 인프라를 자동 관리하지만, 비용 예측이 어렵다는 현업의 피드백이 있습니다.

### 비용 구성 요소

| 구성 요소 | 과금 기준 | 비중(일반적) |
|----------|----------|:----------:|
| **처리 DBU** | 데이터 처리량(바이트)에 비례 | 70~80% |
| **대기 DBU** | Continuous 모드 시 유휴 대기 | 10~20% |
| **유지보수 DBU** | OPTIMIZE, VACUUM 자동 실행 | 5~10% |

### 비용 예측을 위한 실전 공식

```
월간 서버리스 SDP 비용 ≈
    (일 데이터 처리량 GB × 처리 단가 × 30일)
  + (유지보수 비중 10~15%)
  + (Continuous 시 대기 비용)

예시: 일 100GB 처리, Triggered 모드
  ≈ (100GB × $0.07/GB × 30일) + 15%
  ≈ $210 + $31.5
  ≈ 약 $242/월
```

> 🔥 **현업에서는**: 서버리스 SDP 비용을 정확히 예측하기 어려우므로, **첫 달은 클래식(Job Cluster)과 서버리스를 병렬로 돌려서 비용을 비교**하는 것을 권장합니다. 한 고객사에서는 서버리스 전환 후 비용이 20% 절감된 반면, 다른 고객사에서는 10% 증가한 사례도 있었습니다. **워크로드 특성에 따라 다르므로 반드시 직접 비교 테스트를 하세요.**

### 서버리스 비용 모니터링

```sql
-- 서버리스 SDP의 일별 DBU 사용량 추적
SELECT
    DATE(usage_date) AS date,
    sku_name,
    SUM(usage_quantity) AS total_dbus,
    SUM(usage_quantity * list_price) AS estimated_cost_usd
FROM system.billing.usage
WHERE sku_name LIKE '%SERVERLESS%DLT%'
    AND usage_date > CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY date DESC;
```

---

## 정리

| 핵심 설정 | 권장값 |
|-----------|--------|
| **개발 단계** | Development 모드, Preview 채널 |
| **프로덕션** | Production 모드, Current 채널 |
| **비용 최적화** | 서버리스 또는 Spot 인스턴스 |
| **성능 최적화** | Photon 활성화, Auto Scaling |
| **환경 분리** | Asset Bundles 변수로 카탈로그/스키마 분리 |

---

## 참고 링크

- [Databricks: Configure a SDP pipeline](https://docs.databricks.com/aws/en/delta-live-tables/configure-pipeline.html)
- [Databricks: Pipeline settings](https://docs.databricks.com/aws/en/delta-live-tables/settings.html)
- [Databricks: Serverless SDP](https://docs.databricks.com/aws/en/delta-live-tables/serverless-dlt.html)
- [Azure Databricks: Configure pipelines](https://learn.microsoft.com/en-us/azure/databricks/delta-live-tables/configure-pipeline)
