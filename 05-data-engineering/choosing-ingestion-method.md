# 수집 방법 선택 가이드

## 왜 수집 방법 선택이 중요한가요?

데이터 파이프라인을 구축할 때 가장 먼저 마주치는 질문은 "**어떤 방법으로 데이터를 가져올 것인가?**" 입니다. 잘못된 수집 방법을 선택하면 다음과 같은 문제가 발생할 수 있습니다:

- **과도한 비용**: 불필요하게 복잡한 도구를 사용하여 인프라 비용이 증가합니다
- **높은 지연시간**: 실시간이 필요한 곳에 배치 방식을 적용하면 비즈니스 의사결정이 늦어집니다
- **운영 부담**: 커스텀 코드로 직접 구현하면 유지보수에 많은 시간이 소요됩니다
- **데이터 품질 문제**: 스키마 변경이나 중복 처리를 고려하지 않으면 데이터 신뢰성이 떨어집니다

> 💡 **핵심 원칙**: Databricks가 제공하는 **관리형 서비스를 최대한 활용**하는 것이 비용과 운영 부담을 줄이는 가장 효과적인 방법입니다. 직접 코드를 작성하는 것은 관리형 서비스가 지원하지 않는 경우에만 선택해야 합니다.

---

## Databricks의 데이터 수집 방법 5가지

Databricks에서 데이터를 수집하는 주요 방법은 다음과 같습니다:

| 수집 방법 | 한줄 설명 | 주요 용도 |
|-----------|----------|----------|
| **Auto Loader** | 클라우드 스토리지의 새 파일을 자동 감지하여 수집 | S3, ADLS, GCS의 파일 수집 |
| **Lakeflow Connect** | 외부 DB/SaaS에서 관리형 CDC 수집 | MySQL, Salesforce 등 |
| **SDP (Spark Declarative Pipelines)** | 선언적 파이프라인으로 수집+변환 | ETL 파이프라인 전체 구성 |
| **Lakeflow Jobs** | 오케스트레이션 + 커스텀 수집 | 스케줄링, API 호출, 복잡한 워크플로 |
| **COPY INTO** | 일회성/간헐적 파일 적재 | 마이그레이션, 초기 적재 |

---

## 각 수집 방법 상세 비교

### Auto Loader

Auto Loader는 클라우드 스토리지(S3, ADLS, GCS)에 새로 도착하는 파일을 **자동으로 감지하고 증분 수집**하는 Databricks의 핵심 수집 도구입니다.

**특징:**
- 파일 알림(Notification) 또는 디렉토리 리스팅(Directory Listing) 방식으로 새 파일 감지
- 스키마 추론 및 스키마 진화(Schema Evolution) 자동 처리
- Exactly-once 처리 보장 (체크포인트 기반)
- CSV, JSON, Parquet, Avro, ORC, XML, 텍스트 등 다양한 포맷 지원

**적합한 경우:**
- 클라우드 스토리지에 파일이 지속적으로 도착하는 경우
- IoT 센서 데이터, 로그 파일, 이벤트 데이터 수집
- 스트리밍과 배치 모두 지원이 필요한 경우

```python
# Auto Loader 기본 사용 예시
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/checkpoints/schema")
    .load("s3://my-bucket/raw-data/events/")
)

df.writeStream \
    .option("checkpointLocation", "/checkpoints/events") \
    .trigger(availableNow=True) \
    .toTable("catalog.bronze.events")
```

### Lakeflow Connect

Lakeflow Connect는 외부 데이터베이스와 SaaS 애플리케이션에서 **코드 없이 관리형 커넥터로 데이터를 수집**하는 서비스입니다.

**특징:**
- 초기 스냅샷 + CDC(Change Data Capture) 증분 수집 자동화
- 스키마 진화 자동 처리
- Unity Catalog Connection 기반 보안 연결
- 서버리스 컴퓨트로 실행 (인프라 관리 불필요)

**적합한 경우:**
- 운영 데이터베이스(MySQL, PostgreSQL, Oracle 등)에서 CDC 수집
- SaaS 애플리케이션(Salesforce, Workday, HubSpot 등) 데이터 동기화
- 코드 작성 없이 빠르게 수집 파이프라인을 구성해야 하는 경우

### SDP (Spark Declarative Pipelines)

SDP는 **"무엇을 만들지"만 선언하면 실행 계획을 자동으로 관리**하는 선언적 파이프라인 프레임워크입니다. 수집과 변환을 하나의 파이프라인에서 처리할 수 있습니다.

**특징:**
- Auto Loader와 통합하여 수집부터 변환까지 하나의 파이프라인으로 구성
- Expectations(데이터 품질 규칙) 내장
- 자동 의존성 관리 및 증분 처리
- Streaming Table, Materialized View 등 다양한 출력 형태

**적합한 경우:**
- 수집과 변환을 하나의 선언적 파이프라인으로 관리하고 싶은 경우
- 데이터 품질 규칙을 파이프라인에 내장하고 싶은 경우
- Medallion 아키텍처(Bronze → Silver → Gold)를 체계적으로 구현하는 경우

### Lakeflow Jobs

Lakeflow Jobs는 Databricks의 **워크플로 오케스트레이션 도구**입니다. 다양한 태스크를 조합하여 복잡한 수집 워크플로를 구성할 수 있습니다.

**특징:**
- Python, SQL, Notebook, JAR 등 다양한 태스크 타입 지원
- 스케줄(크론), 파일 도착 트리거, 연속 실행 등 유연한 트리거
- 태스크 간 의존성 관리 (DAG 구성)
- 재시도, 알림, 조건부 실행 등 운영 기능

**적합한 경우:**
- REST API에서 데이터를 주기적으로 호출해야 하는 경우
- 여러 수집 작업을 순서대로 또는 병렬로 조합해야 하는 경우
- 수집 후 검증, 알림 등 복잡한 후처리가 필요한 경우

### COPY INTO

COPY INTO는 클라우드 스토리지의 파일을 Delta 테이블에 **일회성 또는 간헐적으로 적재**하는 SQL 명령입니다.

**특징:**
- 단순한 SQL 문법으로 즉시 실행 가능
- 멱등성(idempotent) 보장 (같은 파일을 중복 적재하지 않음)
- 스트리밍이 아닌 배치 방식으로 동작
- 스키마 진화 지원이 제한적

**적합한 경우:**
- 일회성 데이터 마이그레이션 또는 초기 적재
- 파일이 간헐적으로(하루 1-2회) 도착하고, 실시간 처리가 필요 없는 경우
- 빠르게 프로토타이핑하거나 테스트할 때

```sql
-- COPY INTO 기본 사용 예시
COPY INTO catalog.bronze.sales_data
FROM 's3://my-bucket/exports/sales/'
FILEFORMAT = CSV
FORMAT_OPTIONS ('header' = 'true', 'inferSchema' = 'true')
COPY_OPTIONS ('mergeSchema' = 'true');
```

---

## 의사결정 트리

어떤 수집 방법을 선택해야 할지 아래 플로차트를 따라가 보시기 바랍니다.

| 데이터 소스 유형 | 추천 수집 방법 |
|-----------------|---------------|
| 외부 DB (MySQL, PostgreSQL, Oracle) — Lakeflow Connect 커넥터 있음 | **Lakeflow Connect** |
| 외부 DB — 커넥터 없음, CDC 필요 | **JDBC + Spark Structured Streaming** |
| 외부 DB — 커넥터 없음, 스냅샷 | **JDBC + COPY INTO** |
| 클라우드 스토리지 파일 — 지속적 신규 파일 | **Auto Loader** |
| 클라우드 스토리지 파일 — 일회성/비정기 | **COPY INTO / read_files** |
| SaaS API (Salesforce 등) — 커넥터 있음 | **Lakeflow Connect** |
| SaaS API — 커넥터 없음 | **Custom Python + Lakeflow Jobs** |
| 스트리밍 (Kafka 등) | **Spark Structured Streaming** |

---

## 데이터 소스 유형별 추천

### 파일 기반 소스

| 시나리오 | 추천 방법 | 이유 |
|----------|----------|------|
| S3에 JSON 로그가 실시간으로 도착 | **Auto Loader** | 새 파일 자동 감지, 스키마 추론 지원 |
| Azure Blob에 일일 CSV 리포트 업로드 | **Auto Loader** 또는 **COPY INTO** | 빈도에 따라 선택 |
| 1회성 Parquet 파일 초기 적재 | **COPY INTO** | 단순하고 빠른 실행 |
| GCS에 IoT 센서 데이터 스트리밍 | **Auto Loader** (SDP 내부) | 연속 수집 + 품질 검증 통합 |

### 데이터베이스 소스

| 시나리오 | 추천 방법 | 이유 |
|----------|----------|------|
| MySQL 운영 DB에서 실시간 CDC 수집 | **Lakeflow Connect** | 관리형 CDC, 코드 불필요 |
| PostgreSQL에서 일일 전체 덤프 | **Lakeflow Jobs** + JDBC | 스케줄 배치 수집 |
| Oracle에서 특정 테이블만 CDC | **Lakeflow Connect** | 테이블 선택 수집 지원 |
| 레거시 DB (커넥터 미지원) | **Lakeflow Jobs** + Python | 커스텀 JDBC 연결 |

### SaaS 소스

| 시나리오 | 추천 방법 | 이유 |
|----------|----------|------|
| Salesforce 고객/영업 데이터 동기화 | **Lakeflow Connect** | 네이티브 커넥터 GA |
| HubSpot 마케팅 데이터 수집 | **Lakeflow Connect** | 베타 커넥터 사용 가능 |
| 커넥터 미지원 SaaS | **Lakeflow Jobs** + REST API | Python으로 API 호출 |
| Google Sheets 데이터 동기화 | **Lakeflow Connect** | Google Sheets 커넥터 GA |

### 스트리밍 소스

| 시나리오 | 추천 방법 | 이유 |
|----------|----------|------|
| Kafka 토픽에서 이벤트 수집 | **Structured Streaming** (SDP 내부) | 네이티브 Kafka 커넥터 |
| Amazon Kinesis 스트림 | **Structured Streaming** | Kinesis 커넥터 지원 |
| Event Hubs (Azure) | **Structured Streaming** | Kafka 호환 프로토콜 사용 |

---

## 배치 vs 스트리밍 관점에서의 선택

| 수집 유형 | 도구 | 특징 |
|-----------|------|------|
| 배치 수집 | COPY INTO, Lakeflow Jobs (스케줄), Auto Loader (배치 모드) | 정해진 시간에 일괄 처리 |
| 스트리밍 수집 | Auto Loader (스트리밍 모드), Spark Structured Streaming, Lakeflow Connect (CDC) | 실시간/준실시간 처리 |

| 관점 | 배치 수집 | 스트리밍 수집 | 하이브리드 |
|------|----------|-------------|-----------|
| **데이터 지연시간** | 분~시간 | 초~분 | 분 단위 (트리거 배치) |
| **비용** | 낮음 (사용 시에만) | 높음 (항상 실행) | 중간 |
| **복잡도** | 낮음 | 중간~높음 | 중간 |
| **적합한 경우** | 일일 리포트, 배치 ETL | 실시간 대시보드, 알림 | 준실시간 분석 |
| **대표 도구** | COPY INTO, Jobs | Structured Streaming | Auto Loader (trigger) |

> 💡 **Trigger AvailableNow 패턴**: Auto Loader를 `trigger(availableNow=True)`로 실행하면 스트리밍의 장점(증분 처리, 체크포인트)을 유지하면서 배치처럼 실행할 수 있습니다. 비용 효율적이면서도 안정적인 증분 수집이 가능하여 가장 많이 권장되는 패턴입니다.

---

## 비용 / 복잡도 / 지연시간 종합 비교

| 수집 방법 | 초기 설정 복잡도 | 운영 복잡도 | 비용 수준 | 데이터 지연 | 스키마 진화 | 코드 필요 여부 |
|-----------|:---------------:|:-----------:|:---------:|:----------:|:----------:|:-------------:|
| **Auto Loader** | 낮음 | 낮음 | 낮음~중간 | 초~분 | 자동 | Python/SQL |
| **Lakeflow Connect** | 매우 낮음 | 매우 낮음 | 중간 | 초~분 | 자동 | 불필요 (No-code) |
| **SDP** | 중간 | 낮음 | 중간 | 초~분 | 자동 | SQL/Python |
| **Lakeflow Jobs** | 중간~높음 | 중간 | 가변적 | 분~시간 | 수동 | Python/SQL |
| **COPY INTO** | 매우 낮음 | 낮음 | 낮음 | 시간~일 | 제한적 | SQL |

---

## 실제 시나리오별 추천 구성

### 시나리오 1: 이커머스 데이터 파이프라인

> 운영 MySQL DB(주문, 고객, 상품) + S3 클릭스트림 로그 + Salesforce CRM

```
MySQL (주문/고객/상품)  → Lakeflow Connect (CDC)     → Bronze 테이블
S3 클릭스트림 JSON       → Auto Loader (연속 수집)     → Bronze 테이블
Salesforce CRM           → Lakeflow Connect (커넥터)   → Bronze 테이블
                                                         ↓
                                              SDP 파이프라인 (변환)
                                                         ↓
                                              Silver/Gold 테이블
```

### 시나리오 2: IoT 센서 데이터 수집

> 수만 대 센서 → Kafka → Databricks → 실시간 대시보드

```
IoT 센서 → Kafka 토픽 → Structured Streaming (SDP 내부) → Bronze
                                                            ↓
                                                    SDP 변환 → Silver → Gold
                                                                          ↓
                                                              실시간 대시보드 (Lakeview)
```

### 시나리오 3: 일일 배치 리포트 수집

> ERP 시스템에서 매일 CSV 파일을 S3에 업로드

```
ERP → SFTP → S3 버킷 → Auto Loader (trigger=availableNow, 일 1회)
                         → Bronze 테이블
                         → Lakeflow Jobs로 스케줄링
```

---

## COPY INTO에서 Auto Loader로 마이그레이션

기존에 COPY INTO를 사용하고 있다면, 데이터 볼륨이 증가함에 따라 Auto Loader로 마이그레이션하는 것을 권장합니다.

### 왜 마이그레이션해야 하나요?

| 비교 항목 | COPY INTO | Auto Loader |
|-----------|-----------|-------------|
| 새 파일 감지 방식 | 매번 디렉토리 스캔 | 알림 또는 증분 리스팅 |
| 대규모 파일 처리 | 파일 수 증가 시 성능 저하 | 수백만 파일도 효율적 처리 |
| 스키마 진화 | 제한적 (mergeSchema 옵션) | 자동 감지 및 진화 |
| 체크포인트 | 없음 (멱등성만 보장) | 있음 (정확한 재시작 지점) |
| 스트리밍 지원 | 불가 (배치 전용) | 가능 (연속/트리거 배치) |

### 마이그레이션 예시

**기존 COPY INTO 코드:**
```sql
-- 매일 스케줄로 실행
COPY INTO catalog.bronze.events
FROM 's3://my-bucket/events/'
FILEFORMAT = JSON
COPY_OPTIONS ('mergeSchema' = 'true');
```

**Auto Loader로 변환:**
```python
# 같은 소스를 Auto Loader로 수집 (SDP 파이프라인 내)
import dlt

@dlt.table(
    comment="Auto Loader로 수집한 이벤트 데이터"
)
def bronze_events():
    return (
        spark.readStream
            .format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.schemaLocation", "/checkpoints/events/schema")
            .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
            .load("s3://my-bucket/events/")
    )
```

> ⚠️ **마이그레이션 주의사항**: Auto Loader로 전환할 때 기존에 COPY INTO로 이미 적재한 파일이 중복 수집되지 않도록, 초기 체크포인트 설정에 주의해야 합니다. 새 경로에서 시작하거나, `cloudFiles.backfillInterval` 옵션을 활용하여 기존 파일을 처리할 수 있습니다.

---

## 정리

- **관리형 커넥터가 있으면 Lakeflow Connect를 우선 사용**합니다 (가장 적은 코드, 가장 낮은 운영 부담)
- **클라우드 스토리지 파일은 Auto Loader**가 최선의 선택입니다 (COPY INTO보다 확장성과 안정성이 우수)
- **스트리밍 소스(Kafka/Kinesis)는 Structured Streaming**을 SDP 파이프라인 내에서 사용합니다
- **커넥터가 없는 소스는 Lakeflow Jobs + 커스텀 코드**로 처리합니다
- **수집과 변환을 통합하려면 SDP 파이프라인**을 활용합니다
- **COPY INTO는 일회성 적재에만 사용**하고, 반복적인 수집에는 Auto Loader를 선택합니다

---

## 참고 링크

- [Databricks: Ingestion Overview](https://docs.databricks.com/aws/en/ingestion/)
- [Databricks: Lakeflow Connect Connectors](https://docs.databricks.com/aws/en/lakeflow-connect/)
- [Databricks: Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)
- [Databricks: COPY INTO](https://docs.databricks.com/aws/en/sql/language-manual/delta-copy-into.html)
- [Databricks: Structured Streaming](https://docs.databricks.com/aws/en/structured-streaming/)
- [Databricks Blog: Choosing the Right Ingestion Method](https://www.databricks.com/blog)
