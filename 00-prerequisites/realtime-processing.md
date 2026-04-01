# 실시간 처리 기술 — Kafka, Flink, Spark Streaming

## 왜 실시간 처리가 필요한가요?

비즈니스 환경이 점점 빨라지면서, "어제의 데이터"가 아닌 "지금의 데이터"로 의사결정해야 하는 상황이 많아졌습니다.

| 사례 | 필요한 응답 시간 | 설명 |
|------|----------------|------|
| 신용카드 사기 감지 | 밀리초 | 결제 승인 전에 사기 여부를 판단해야 합니다 |
| 실시간 추천 | 초 | 사용자의 최근 행동에 기반한 상품 추천입니다 |
| IoT 설비 모니터링 | 초 | 온도, 진동 이상 시 즉시 알림을 보내야 합니다 |
| 실시간 대시보드 | 분 | 현재 매출, 트래픽을 실시간으로 표시합니다 |
| 로그 모니터링 | 초~분 | 서버 에러 급증 시 즉시 대응해야 합니다 |

---

## 실시간 처리의 핵심 구성 요소

실시간 데이터 처리 시스템은 일반적으로 세 가지 구성 요소로 이루어집니다.

| 구성 요소 | 역할 | 예시 |
|-----------|------|------|
| **Producer (데이터 생산자)**| 이벤트를 발행합니다 | 앱, 센서, 서버 |
| **Message Queue (메시지 큐)**| 이벤트를 전달합니다 | Kafka, Kinesis |
| **Consumer (데이터 소비자)**| 이벤트를 처리합니다 | Spark, Flink |
| **Storage (저장소)**| 처리 결과를 저장합니다 | Delta Lake |

| 구성 요소 | 역할 | 비유 |
|-----------|------|------|
| **Producer (생산자)**| 이벤트/데이터를 생성하여 발행합니다 | 편지를 보내는 사람 |
| **Message Queue (메시지 큐)**| 이벤트를 임시 저장하고 순서대로 전달합니다 | 우체국 (편지를 모아서 배달) |
| **Consumer (소비자)**| 이벤트를 받아서 처리합니다 | 편지를 받아서 읽는 사람 |
| **Storage (저장소)**| 처리된 결과를 영구 저장합니다 | 파일 캐비닛 |

---

## Apache Kafka

### 개념

> 💡 **Apache Kafka**는 LinkedIn에서 개발한 **분산 이벤트 스트리밍 플랫폼** 입니다. 초당 수백만 건의 이벤트를 안정적으로 수집하고, 여러 소비자에게 전달할 수 있습니다. 현재 전 세계 Fortune 500 기업의 80% 이상이 사용하고 있습니다.

### 핵심 개념

**Apache Kafka 구조**

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **Topic: "orders"**| 메시지 채널 | 주문 관련 이벤트를 저장합니다 |
| Partition 0 | 병렬 처리 단위 | msg1, msg4, msg7... |
| Partition 1 | 병렬 처리 단위 | msg2, msg5, msg8... |
| Partition 2 | 병렬 처리 단위 | msg3, msg6, msg9... |
| **Producer**| 데이터 생산자 | 주문 서비스, 모바일 앱 등이 Topic에 메시지를 전송합니다 |
| **Consumer Group**| 데이터 소비자 | 각 Partition을 분담하여 병렬로 메시지를 처리합니다 |

| 개념 | 설명 |
|------|------|
| **Topic (토픽)**| 이벤트가 발행되는 카테고리입니다. "orders", "clicks", "logs" 같은 이름을 가집니다 |
| **Partition (파티션)**| 토픽을 물리적으로 나눈 단위입니다. 파티션이 여러 개이면 병렬로 처리할 수 있습니다 |
| **Producer (프로듀서)**| 토픽에 이벤트를 발행하는 애플리케이션입니다 |
| **Consumer (컨슈머)**| 토픽에서 이벤트를 읽어가는 애플리케이션입니다 |
| **Consumer Group**| 같은 토픽을 읽는 컨슈머들의 그룹입니다. 파티션이 그룹 내 컨슈머에게 분배됩니다 |
| **Offset**| 각 파티션에서 컨슈머가 어디까지 읽었는지를 나타내는 위치 정보입니다 |
| **Broker**| Kafka 서버 인스턴스입니다. 여러 Broker가 클러스터를 구성합니다 |

### Kafka의 특징

| 특징 | 설명 |
|------|------|
| **높은 처리량**| 초당 수백만 건의 이벤트를 처리할 수 있습니다 |
| **내구성**| 이벤트를 디스크에 저장하므로, 컨슈머가 느려도 데이터가 유실되지 않습니다 |
| **재처리 가능**| Offset을 되돌려서 과거 이벤트를 다시 읽을 수 있습니다 |
| **다중 소비**| 하나의 이벤트를 여러 Consumer Group이 독립적으로 읽을 수 있습니다 |

### 관련 솔루션

| 솔루션 | 설명 |
|--------|------|
| **Confluent Cloud**| Kafka의 관리형 클라우드 서비스입니다 |
| **Amazon MSK**| AWS에서 관리형 Kafka를 제공합니다 |
| **Azure Event Hubs**| Kafka API와 호환되는 Azure의 스트리밍 서비스입니다 |
| **Amazon Kinesis**| AWS의 자체 실시간 스트리밍 서비스입니다 |

---

## 스트림 처리 엔진

메시지 큐에서 이벤트를 받아 실시간으로 처리하는 엔진들을 살펴보겠습니다.

### Apache Spark Structured Streaming

> 💡 **Spark Structured Streaming**은 Apache Spark의 스트리밍 모듈로, **마이크로 배치(Micro-Batch)** 방식으로 스트리밍 데이터를 처리합니다. 배치 코드와 동일한 DataFrame API를 사용할 수 있어서, 배치→스트리밍 전환이 매우 쉽습니다.

```python
# Kafka에서 주문 이벤트를 실시간으로 읽어서 Delta 테이블에 저장
df = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "orders")
    .load()
)

# JSON 파싱 및 변환
from pyspark.sql.functions import from_json, col
schema = "order_id LONG, customer_id LONG, amount DOUBLE, ts TIMESTAMP"

orders = (df
    .select(from_json(col("value").cast("string"), schema).alias("data"))
    .select("data.*")
)

# Delta Lake에 스트리밍으로 저장
(orders.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/checkpoints/orders")
    .toTable("catalog.schema.streaming_orders")
)
```

### Apache Flink

> 💡 **Apache Flink**는 진정한 **이벤트 단위(Event-at-a-time)** 스트리밍 처리 엔진입니다. 이벤트가 도착하는 즉시 하나씩 처리하므로, Spark의 마이크로 배치보다 더 낮은 지연 시간(밀리초 수준)을 달성할 수 있습니다.

### Spark Streaming vs Flink 비교

| 비교 항목 | Spark Structured Streaming | Apache Flink |
|-----------|--------------------------|-------------|
| **처리 모델**| 마이크로 배치 (기본) | 이벤트 단위 (True Streaming) |
| **지연 시간**| 수백 밀리초 ~ 초 | 밀리초 ~ 수십 밀리초 |
| **배치/스트리밍 통합**| 동일 API (DataFrame) | 별도 API (DataStream vs Table) |
| **상태 관리**| 체크포인트 기반 | Savepoint + 체크포인트 |
| **생태계**| Spark 생태계 (MLlib, SQL 등) | 독자 생태계 |
| **Databricks 지원**| ✅ 네이티브 지원 | ❌ 별도 운영 필요 |

> 💡 **체크포인트(Checkpoint)란?** 스트리밍 처리의 현재 진행 상태(어디까지 읽었고, 어떤 집계 값을 가지고 있는지)를 주기적으로 저장하는 것입니다. 장애가 발생하면 마지막 체크포인트부터 재시작하여 데이터 유실 없이 처리를 이어갈 수 있습니다.

---

## 이벤트 드리븐 아키텍처 (EDA)

### 개념

> 💡 **이벤트 드리븐 아키텍처(Event-Driven Architecture, EDA)**란 시스템의 구성 요소들이 **이벤트** 를 중심으로 통신하는 아키텍처 패턴입니다. A 서비스에서 발생한 이벤트를 Kafka 같은 메시지 큐에 발행하면, 관심 있는 다른 서비스들이 이를 구독하여 처리합니다.

**이벤트 드리븐 아키텍처 (마이크로서비스)**

| 서비스 | 역할 | Kafka 연동 |
|--------|------|-----------|
| **주문 서비스**| 주문 생성 이벤트 발행 | Producer |
| **Kafka (이벤트 버스)**| 이벤트를 중앙에서 관리 | 메시지 브로커 |
| **결제 서비스**| 주문 이벤트 구독 → 결제 처리 | Consumer |
| **재고 서비스**| 주문 이벤트 구독 → 재고 차감 | Consumer |
| **알림 서비스**| 주문 이벤트 구독 → 알림 발송 | Consumer |
| **분석 서비스**| 주문 이벤트 구독 → 데이터 분석 | Consumer |

### EDA의 장점

| 장점 | 설명 |
|------|------|
| **느슨한 결합**| 서비스 간 직접 호출 없이 이벤트로 통신하므로, 서비스를 독립적으로 변경할 수 있습니다 |
| **확장성**| 이벤트를 처리하는 서비스(Consumer)를 쉽게 추가할 수 있습니다 |
| **실시간 반응**| 이벤트 발생 즉시 관련 서비스가 반응합니다 |
| **이력 보존**| Kafka에 이벤트가 보존되므로, 나중에 새로운 서비스가 과거 이벤트를 재처리할 수 있습니다 |

---

## Databricks에서의 실시간 처리

Databricks는 **Spark Structured Streaming** 을 기반으로 실시간 처리를 지원하며, 이를 Medallion 아키텍처와 결합하여 사용합니다.

| 계층 | 구성 요소 | 역할 |
|------|-----------|------|
| **소스**| Kafka / Kinesis | 스트리밍 데이터를 수집합니다 |
| **Bronze**| Streaming Table | 원본 데이터를 수집합니다 |
| **Silver**| Streaming Table | 실시간 변환을 수행합니다 |
| **Gold**| Materialized View | 집계 결과를 제공합니다 |
| **출력**| 실시간 대시보드 | Gold 데이터를 시각화합니다 |

| Databricks 기능 | 역할 |
|----------------|------|
| **Auto Loader**| 클라우드 스토리지의 새 파일을 실시간으로 감지하여 수집합니다 |
| **Structured Streaming**| Kafka, Kinesis 등에서 이벤트를 실시간으로 읽고 처리합니다 |
| **Streaming Tables (SDP)**| 선언적으로 스트리밍 파이프라인을 정의합니다 |
| **Materialized Views (SDP)**| 스트리밍 데이터를 실시간으로 집계합니다 |
| **Delta Lake**| 스트리밍으로 적재된 데이터에 ACID 트랜잭션을 보장합니다 |

---

## Kafka를 직접 운영해본 경험 — ZooKeeper의 고통

Kafka를 Confluent Cloud나 Amazon MSK 같은 관리형 서비스로 사용하면 편리하지만, 직접 운영한다면 이야기가 완전히 달라집니다.

### ZooKeeper 관리의 고통

Kafka는 오랫동안 **ZooKeeper** 라는 분산 코디네이션 서비스에 의존해왔습니다. ZooKeeper가 죽으면 Kafka도 죽습니다.

| 문제 | 현실 |
|------|------|
| **ZooKeeper 클러스터 유지**| 최소 3대(홀수)를 운영해야 합니다. 하나가 죽으면 즉시 복구해야 합니다 |
| **세션 타임아웃**| ZooKeeper와 Kafka Broker 간 세션이 끊기면, Broker가 클러스터에서 제외됩니다. 네트워크 순간 장애에도 민감합니다 |
| **디스크 공간**| ZooKeeper의 트랜잭션 로그가 디스크를 가득 채우면 전체 Kafka 클러스터가 멈춥니다. 새벽 3시에 호출받는 대표적인 원인입니다 |
| **버전 업그레이드**| Kafka와 ZooKeeper 버전 호환성을 맞춰야 합니다. Rolling Upgrade 절차를 잘못하면 서비스 중단이 발생합니다 |

> 🆕 **KRaft 모드 (2024~)**: Kafka 3.3 이후부터 ZooKeeper 없이도 동작하는 **KRaft(Kafka Raft)** 모드가 도입되었습니다. Kafka 4.0부터는 ZooKeeper가 완전히 제거될 예정입니다. 새로 구축하는 경우 KRaft를 사용하시는 것을 권장합니다.

### 파티션 리밸런싱의 악몽

Kafka Broker를 추가하거나 제거할 때, 파티션을 재분배해야 합니다. 이 과정에서 대량의 데이터가 네트워크를 통해 복사되며, 클러스터 전체의 성능이 저하됩니다.

| 상황 | 영향 |
|------|------|
| Broker 1대 추가 | 기존 파티션의 일부를 새 Broker로 이동. 데이터 크기에 따라 수 시간~수일 소요 |
| Broker 1대 장애 | 해당 Broker의 파티션 리더십이 다른 Broker로 넘어감. ISR(In-Sync Replica)이 부족하면 데이터 유실 가능 |
| 토픽 파티션 증가 | 한 번 늘린 파티션은 줄일 수 없습니다. 초기 설계가 매우 중요합니다 |

> 💡 **현업에서는 이렇게 합니다**: 파티션 수는 "현재 필요한 수의 2~3배"로 넉넉하게 설정합니다. 나중에 늘리는 것은 가능하지만, 줄이는 것은 토픽을 재생성해야 하므로 매우 번거롭습니다. 일반적으로 토픽당 파티션 6~12개로 시작하고, Consumer 수에 맞춰 조정합니다.

### Kafka 비용의 현실

| 방식 | 월 비용 (대략) | 운영 부담 |
|------|-------------|-----------|
| **자체 운영 (EC2)**| $500~$2,000+ (3 Broker + 3 ZooKeeper) | 높음 — 모니터링, 패치, 장애 대응 필요 |
| **Amazon MSK**| $1,000~$3,000+ | 중간 — 브로커 관리는 AWS가 담당 |
| **Confluent Cloud**| $500~$5,000+ (사용량 기반) | 낮음 — 완전 관리형 |

> 💡 **이것을 안 하면...**: Kafka 모니터링을 설정하지 않으면, Consumer Lag(소비 지연)이 쌓이고 있는지 알 수 없습니다. 어느 날 갑자기 "실시간 대시보드가 3시간 전 데이터를 보여주고 있다"는 보고를 받게 됩니다. 최소한 Consumer Lag, Broker Disk Usage, ISR Shrink 이 세 가지는 반드시 모니터링하세요.

---

## "실시간이 필요하다고 생각했는데, 마이크로배치로 충분했던 경우"

현업에서 "실시간 처리가 필요합니다"라는 요구사항을 받으면, 먼저 **정말 실시간이 필요한지** 를 따져봐야 합니다.

### 진짜 실시간(밀리초) vs 니어 실시간(초~분) vs 마이크로배치(분~시간)

| 요구사항 | 실제 필요한 수준 | 권장 솔루션 |
|----------|----------------|-----------|
| "신용카드 사기 감지" | **진짜 실시간**(밀리초) | Flink, 또는 별도의 실시간 서빙 레이어 |
| "실시간 대시보드" | **니어 실시간**(1~5분 지연 허용) | Spark Structured Streaming (마이크로배치) |
| "주문 현황 모니터링" | **니어 실시간**(5~15분 지연 허용) | Spark Structured Streaming 또는 Auto Loader |
| "일별 리포트를 오전 9시에" | **배치**(시간 단위) | 스케줄된 배치 잡 |
| "데이터가 들어오면 바로 분석" | 대부분 **마이크로배치** 로 충분 | Auto Loader + SDP |

> 💡 **현업에서는 이렇게 합니다**: 고객이 "실시간"이라고 말할 때, 실제로 의미하는 것은 대부분 "1시간 전 데이터가 아니라 5분 전 데이터를 보고 싶다"입니다. 이 정도의 요구사항은 Spark Structured Streaming의 마이크로배치(수 초~수 분 간격)로 충분히 충족됩니다. 진짜 밀리초 수준의 실시간이 필요한 경우는 전체 워크로드의 5% 미만입니다.

### Databricks Structured Streaming과의 연결

Databricks에서 실시간 처리를 구현하는 가장 일반적인 패턴은 다음과 같습니다.

```
Kafka → Spark Structured Streaming → Delta Lake (Bronze → Silver → Gold)
                                        ↓
                              SQL Warehouse → AI/BI Dashboard
```

```python
# Databricks에서 Kafka → Delta Lake 실시간 파이프라인 (간단 버전)
(spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "orders")
    .option("startingOffsets", "latest")
    .load()
    .selectExpr("CAST(value AS STRING) as json_str")
    .writeStream
    .format("delta")
    .option("checkpointLocation", "/checkpoints/orders_bronze")
    .outputMode("append")
    .toTable("catalog.schema.orders_bronze")
)
```

**SDP(Spark Declarative Pipelines)를 사용하면 더 간단해집니다**:

```sql
-- SDP에서는 SQL만으로 스트리밍 테이블을 정의할 수 있습니다
CREATE STREAMING TABLE orders_bronze
AS SELECT * FROM STREAM read_kafka(
    bootstrapServers => 'broker:9092',
    subscribe => 'orders'
);

CREATE STREAMING TABLE orders_silver
AS SELECT
    from_json(value, 'order_id LONG, customer_id LONG, amount DOUBLE') AS parsed
FROM STREAM orders_bronze;
```

> 💡 **현업에서의 판단 기준**: "마이크로배치로 충분한 워크로드에 Flink를 도입하면, 시스템 복잡성만 증가하고 ROI가 나오지 않습니다." Databricks를 이미 사용 중이라면, Structured Streaming부터 시작하고, 정말 밀리초 수준의 지연이 필요한 워크로드만 Flink로 분리하는 것이 현실적입니다.

---

## 실시간 처리 아키텍처 패턴

### Lambda 아키텍처

> 💡 **Lambda 아키텍처**는 **배치 레이어**와 **스피드 레이어**(실시간)를 분리하여, 각각의 장점을 활용하는 아키텍처입니다. 배치는 정확성을, 스피드는 속도를 담당합니다.

**단점**: 같은 로직을 배치용과 스트리밍용으로 두 번 구현해야 하는 유지보수 부담이 있습니다.

### Kappa 아키텍처

> 💡 **Kappa 아키텍처**는 모든 데이터를 **스트리밍으로만 처리** 하는 아키텍처입니다. 배치 처리가 필요하면, 과거 이벤트를 Kafka에서 다시 읽어서(재처리) 스트리밍 파이프라인으로 처리합니다.

### Delta 아키텍처 (Databricks 권장)

Databricks는 **Delta Lake의 배치/스트리밍 통합 능력** 을 활용하여, 하나의 코드로 배치와 스트리밍을 모두 처리하는 방식을 권장합니다. Lambda나 Kappa의 장점을 모두 취하면서 복잡성을 줄입니다.

### 아키텍처 선택 가이드

| 아키텍처 | 적합한 경우 | 주의사항 |
|---------|-----------|---------|
| **Lambda**| 레거시 시스템이 이미 배치로 구축되어 있고, 일부만 실시간이 필요한 경우 | 동일 로직을 두 번 구현하는 유지보수 부담이 큽니다 |
| **Kappa**| 모든 데이터가 이벤트 형태이고, Kafka에 충분한 보존 기간이 설정된 경우 | 대규모 재처리 시 Kafka에서 읽는 비용이 큽니다 |
| **Delta**| Databricks를 사용하는 경우 (가장 권장) | 하나의 Delta 테이블에 배치/스트리밍 모두 쓸 수 있어 가장 단순합니다 |

> 💡 **현업에서는 이렇게 합니다**: "어떤 아키텍처를 선택할지" 고민하기보다, "Databricks + Delta Lake를 쓴다면 Delta 아키텍처가 자동으로 따라온다"고 생각하시면 됩니다. Structured Streaming으로 실시간 데이터를 Delta에 쓰고, 같은 테이블을 배치 잡으로도 읽고 쓸 수 있으므로, 별도의 아키텍처 설계가 크게 필요하지 않습니다.

---

## 실시간 처리 도입 시 체크리스트

실시간 처리를 도입하기 전에 아래 항목들을 점검하세요.

| 항목 | 질문 | 설명 |
|------|------|------|
| **비즈니스 요구**| 정말 실시간이 필요한가? | "5분 지연 허용"이면 마이크로배치로 충분합니다 |
| **데이터 소스**| 소스가 이벤트를 발행할 수 있는가? | CDC, API Webhook, Kafka 연동 가능 여부를 확인합니다 |
| **데이터 양**| 초당 이벤트 수는? | 초당 1,000건 이하면 가벼운 솔루션으로 충분합니다 |
| **Late data**| 늦게 도착하는 데이터를 어떻게 처리할 것인가? | Watermark 설정이 필수입니다 |
| **정확성**| Exactly-once가 필요한가? | 결제, 주문 등은 중복 처리가 허용되지 않습니다 |
| **모니터링**| Consumer Lag, 처리 지연을 어떻게 모니터링할 것인가? | 알림 체계가 없으면 장애를 감지할 수 없습니다 |
| **비용**| 24/7 실행 비용을 감당할 수 있는가? | 스트리밍 클러스터는 항상 실행되므로 비용이 지속적입니다 |

> 💡 **현업에서는 이렇게 합니다**: 실시간 처리를 처음 도입할 때는, 전체 파이프라인을 한 번에 실시간으로 전환하지 마세요. 가장 비즈니스 임팩트가 큰 하나의 파이프라인을 먼저 실시간으로 전환하고, 안정화된 후에 점진적으로 확대하는 것이 현실적입니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **Apache Kafka**| 분산 이벤트 스트리밍 플랫폼. 이벤트를 안정적으로 수집하고 전달합니다 |
| **Structured Streaming**| Spark의 스트리밍 엔진. 마이크로 배치 방식으로 동작합니다 |
| **Apache Flink**| 이벤트 단위 스트리밍 엔진. 밀리초 수준의 낮은 지연 시간을 제공합니다 |
| **EDA**| 이벤트를 중심으로 서비스가 통신하는 아키텍처 패턴입니다 |
| **CDC**| 데이터베이스 변경 사항을 실시간으로 감지하여 전달하는 기술입니다 |
| **체크포인트**| 스트리밍 처리 상태를 저장하여 장애 시 복구를 보장합니다 |
| **마이크로배치**| 대부분의 "실시간" 요구사항에 충분한, 수 초~수 분 간격의 처리 방식입니다 |

이것으로 **선행 지식** 섹션을 마치겠습니다. 이제 [01. 데이터 기초](../01-data-fundamentals/README.md)로 진행하시면, Databricks 중심의 본격적인 학습을 시작하실 수 있습니다.

---

## 참고 링크

- [Apache Kafka Official](https://kafka.apache.org/)
- [Apache Flink Official](https://flink.apache.org/)
- [Databricks: Structured Streaming](https://docs.databricks.com/aws/en/structured-streaming/)
- [Confluent: Kafka 101](https://developer.confluent.io/courses/apache-kafka/events/)
- [Databricks Blog: Streaming](https://www.databricks.com/blog/category/engineering-blog)
