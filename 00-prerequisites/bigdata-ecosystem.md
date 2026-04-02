# 빅데이터 생태계 — 주요 솔루션과 빅 플레이어

## 현재 빅데이터 생태계의 전체 그림

빅데이터 생태계는 매우 넓고 다양한 도구들이 존재합니다. 이 문서에서는 각 영역별로 어떤 솔루션이 있고, 주요 벤더들이 어떤 포지션을 차지하고 있는지 정리해 드리겠습니다.

> 💡 **왜 생태계를 알아야 하나요?** Databricks를 사용하더라도 주변 도구들과 연동하는 경우가 많습니다. "우리 팀은 이미 Kafka를 쓰고 있는데, Databricks와 어떻게 연결하나요?" "기존 Tableau 대시보드를 Databricks SQL과 연결할 수 있나요?" 같은 질문에 답하려면, 전체 생태계를 알아야 합니다. 또한 기술 선택 회의에서 "왜 Databricks인가?"를 설명하려면, 경쟁 도구들의 강점과 약점을 이해하고 있어야 합니다.

---

## 빅데이터의 등장 배경

### 전통 RDBMS의 한계

관계형 데이터베이스(RDBMS, Relational Database Management System)는 수십 년간 기업 데이터를 관리해온 핵심 기술입니다. Oracle, MySQL, PostgreSQL 같은 시스템은 엄격한 스키마(Schema), ACID 트랜잭션, SQL 기반 조회를 제공하며 OLTP(Online Transaction Processing) 워크로드에 최적화되어 있습니다.

그러나 인터넷이 확산되고 모바일 기기가 폭발적으로 증가하면서, 전통 RDBMS가 처리하기 어려운 새로운 데이터 특성이 등장했습니다.

| 한계 | 설명 |
|------|------|
| **수직 확장(Scale-Up) 의존** | CPU와 RAM을 더 강력한 서버로 교체하는 방식은 비용이 기하급수적으로 증가하고, 물리적 한계가 명확합니다 |
| **정형 데이터만 처리** | 로그 파일, 이미지, JSON, XML 같은 반정형/비정형 데이터를 저장하고 분석하기 어렵습니다 |
| **배치 분석에 부적합** | 수 TB의 데이터를 전체 스캔하는 분석 쿼리는 OLTP 데이터베이스에 큰 부하를 줍니다 |
| **고비용** | Oracle 등 상용 RDBMS의 라이선스 비용은 매우 높습니다 |

### 3V: 빅데이터를 정의하는 세 가지 특성

2001년 Gartner의 애널리스트 Doug Laney가 제안한 **3V** 프레임워크는 빅데이터를 정의하는 가장 널리 사용되는 모델입니다.

| V | 의미 | 예시 |
|---|------|------|
| **Volume (규모)** | 데이터의 절대적인 크기가 전통적인 방법으로 처리하기 어려운 수준 | 하루 수십 TB의 클릭 로그, 연간 수 PB의 센서 데이터 |
| **Velocity (속도)** | 데이터가 생성되고 처리되는 속도가 매우 빠름 | 초당 수백만 건의 IoT 센서 이벤트, 실시간 금융 거래 |
| **Variety (다양성)** | 정형, 반정형, 비정형 등 다양한 형태의 데이터 | 구조화된 거래 내역 + 비구조화된 SNS 텍스트 + 이미지/동영상 |

이후 **Veracity (정확성)**, **Value (가치)** 를 추가한 5V 모델로 발전했지만, 핵심은 동일합니다. 기존 RDBMS로는 이 특성들을 모두 만족시키기 어려웠고, 이를 해결하기 위한 새로운 기술들이 등장했습니다.

> 💡 **실무에서의 관점**: "우리 데이터가 빅데이터인가요?"라는 질문을 자주 받습니다. 절대적인 크기보다 **"현재 도구로 처리하기 어려운가?"** 가 더 실용적인 기준입니다. 수십 GB라도 초당 처리해야 한다면 빅데이터 기술이 필요하고, 수 TB라도 월 1회 배치 처리라면 기존 도구로 충분할 수 있습니다.

---

## Hadoop 생태계

### Hadoop의 등장

2004년 Google이 발표한 두 편의 논문이 분산 처리의 패러다임을 바꿨습니다.

- **GFS (Google File System)**: 수천 대의 범용 서버에 데이터를 분산 저장하는 방법
- **MapReduce**: 분산된 데이터를 병렬로 처리하는 프로그래밍 모델

Yahoo!의 Doug Cutting과 Mike Cafarella는 이 논문을 기반으로 오픈소스 구현체인 **Apache Hadoop** 을 만들었습니다(2006년). 이후 Facebook, LinkedIn, Twitter 등 주요 인터넷 기업들이 Hadoop을 채택하면서 빅데이터 시대가 열렸습니다.

### Hadoop의 핵심 컴포넌트

```
Hadoop 클러스터 구조

┌────────────────────────────────────────────┐
│                YARN                        │
│       (리소스 관리 / 작업 스케줄링)           │
├───────────────┬────────────────────────────┤
│  MapReduce    │  Hive / Pig / Spark 등      │
│  (배치 처리)   │  (상위 레이어 처리 도구)     │
├───────────────┴────────────────────────────┤
│                 HDFS                       │
│          (분산 파일 시스템)                  │
└────────────────────────────────────────────┘
```

**HDFS (Hadoop Distributed File System)**

HDFS는 데이터를 128MB 단위의 블록(Block)으로 나누어 여러 DataNode에 분산 저장합니다. 기본적으로 각 블록을 3개 노드에 복제(Replication Factor=3)하여 내결함성(Fault Tolerance)을 확보합니다.

- **NameNode**: 파일 시스템의 메타데이터(어떤 블록이 어느 노드에 있는지)를 관리하는 마스터 노드
- **DataNode**: 실제 데이터 블록을 저장하는 워커 노드
- **한계**: NameNode가 단일 장애점(SPOF, Single Point of Failure)이며, 소규모 파일을 많이 저장하면 메타데이터 과부하가 발생합니다

**MapReduce**

MapReduce는 두 단계로 데이터를 처리합니다.

| 단계 | 역할 | 예시 (단어 세기) |
|------|------|----------------|
| **Map** | 입력 데이터를 키-값 쌍으로 변환 | "hello world" → (hello,1), (world,1) |
| **Shuffle & Sort** | 같은 키를 가진 데이터를 모아 정렬 | (hello,[1,1,1]), (world,[1,1]) |
| **Reduce** | 같은 키의 값들을 집계 | (hello,3), (world,2) |

**YARN (Yet Another Resource Negotiator)**

YARN은 Hadoop 2.0에서 도입된 리소스 관리 레이어입니다. 클러스터의 CPU와 메모리 자원을 관리하고, MapReduce 외의 다른 처리 프레임워크(Spark, Tez 등)도 Hadoop 위에서 실행할 수 있게 합니다.

### Hadoop의 한계 — 왜 Spark로 넘어갔는가

Hadoop은 혁신적이었지만, 실무에서는 심각한 단점이 있었습니다.

| 한계 | 구체적인 문제 |
|------|-------------|
| **디스크 I/O 집중** | MapReduce의 모든 중간 결과를 HDFS 디스크에 기록합니다. Map 결과 → 디스크 저장 → Shuffle → 디스크 저장 → Reduce 순서로, I/O가 매우 많습니다 |
| **반복 처리 비효율** | ML 알고리즘처럼 같은 데이터를 여러 번 반복해서 처리할 때, 매번 디스크에서 읽어야 합니다. 100번 반복하면 100번 디스크 I/O가 발생합니다 |
| **실시간 처리 불가** | MapReduce는 배치 처리 전용입니다. 분~시간 단위의 지연이 기본입니다 |
| **복잡한 프로그래밍** | 간단한 집계도 Map 함수와 Reduce 함수를 별도로 작성해야 합니다. 코드가 길고 직관적이지 않습니다 |
| **운영 복잡성** | NameNode, DataNode, ResourceManager, NodeManager, ZooKeeper, Hive Metastore 등 수십 개의 컴포넌트를 개별 설정해야 합니다 |

> 💡 **역사적 맥락**: 2010년대 초 Hadoop을 운영한 엔지니어라면 "Container killed by YARN for exceeding memory limits" 에러를 수도 없이 보았을 것입니다. 또한 Hive에서 간단한 JOIN 쿼리를 실행하면 수십 분이 걸리는 것이 당연했습니다. 이런 고통이 Spark의 등장을 환영받게 만들었습니다.

---

## Apache Spark — 왜 100배 빠른가

### Spark의 탄생

Apache Spark는 2009년 UC Berkeley AMPLab에서 Matei Zaharia가 박사 연구 프로젝트로 시작했습니다. "MapReduce의 반복 처리 문제를 인메모리(In-Memory) 처리로 해결하자"는 아이디어가 출발점이었습니다.

2014년 Apache Top-Level 프로젝트로 승격되었고, 같은 해 Matei Zaharia를 포함한 창업자들이 **Databricks** 를 설립하여 Spark의 상용 플랫폼을 만들었습니다.

### MapReduce 대비 100배 빠른 이유

**1. 인메모리 처리 (In-Memory Processing)**

Spark의 가장 큰 혁신은 중간 결과를 메모리(RAM)에 저장한다는 점입니다. MapReduce는 모든 중간 결과를 디스크에 기록하지만, Spark는 가능한 한 메모리에 유지합니다.

```
MapReduce 처리 흐름:
HDFS → Map → [디스크 저장] → Shuffle → [디스크 저장] → Reduce → [디스크 저장]
(각 단계마다 디스크 I/O 발생)

Spark 처리 흐름:
HDFS → [메모리 로드] → 변환1 → 변환2 → 변환3 → [최종 결과만 저장]
(메모리에서 연속 처리, 필요시에만 디스크 사용)
```

메모리 접근 속도는 디스크 대비 수십~수백 배 빠릅니다. ML처럼 같은 데이터를 100번 반복하는 작업에서는 이 차이가 더욱 극명하게 나타납니다.

**2. DAG (Directed Acyclic Graph) 실행 계획**

Spark는 작업 전체를 **DAG** 로 표현하고, Catalyst Optimizer가 최적의 실행 계획을 수립한 뒤 한 번에 실행합니다.

```
예시: filter → groupBy → sort → limit

Spark의 최적화:
- 불필요한 컬럼 제거 (Predicate Pushdown)
- filter를 최대한 앞으로 당겨 처리할 데이터 최소화
- 여러 변환을 하나의 Stage로 묶어 Shuffle 최소화 (Pipelining)
```

반면 MapReduce는 작업을 독립된 Map-Reduce 쌍으로 표현하기 때문에, 복잡한 쿼리를 여러 개의 MapReduce 잡으로 나눠야 하고 각각 디스크를 경유해야 합니다.

**3. Lazy Evaluation (지연 실행)**

Spark는 변환(Transformation) 명령을 즉시 실행하지 않고, 실제 결과가 필요한 액션(Action)이 호출될 때 한 번에 처리합니다. 이를 통해 불필요한 연산을 최소화하고 최적화 기회를 극대화합니다.

```python
# 아래 코드는 즉시 실행되지 않음 (Transformation)
df_filtered = df.filter(col("age") > 25)
df_grouped = df_filtered.groupBy("city").count()

# collect()를 호출하는 순간 전체 계획을 최적화하여 실행 (Action)
result = df_grouped.collect()
```

### RDD → DataFrame → Dataset 진화

Spark의 API는 세 단계를 거쳐 발전했습니다.

**RDD (Resilient Distributed Dataset) — Spark 1.0**

RDD는 Spark의 가장 기본적인 데이터 구조입니다. 분산된 불변(Immutable) 데이터 컬렉션으로, 파티션 단위로 클러스터 노드에 분산됩니다.

- **장점**: 완전한 제어권, 타입 안전성(Type Safety), 복잡한 커스텀 로직 구현 가능
- **단점**: 최적화를 개발자가 수동으로 해야 함, verbose한 코드, SQL과 연동 어려움

```python
# RDD API 예시
rdd = sc.textFile("data.txt")
counts = rdd.flatMap(lambda line: line.split(" ")) \
            .map(lambda word: (word, 1)) \
            .reduceByKey(lambda a, b: a + b)
```

**DataFrame — Spark 1.3 (2015년)**

DataFrame은 이름이 있는 컬럼으로 구성된 분산 데이터셋입니다. pandas DataFrame과 유사한 개념이지만, 클러스터에서 분산 처리됩니다. **Catalyst Optimizer** 가 자동으로 실행 계획을 최적화하여 RDD보다 빠른 경우도 많습니다.

- **장점**: SQL과 완벽하게 통합, 자동 최적화, 직관적인 API
- **단점**: 컴파일 타임 타입 체크 없음 (Python/R에서는 런타임에 오류 발견)

```python
# DataFrame API 예시
df = spark.read.parquet("data.parquet")
result = df.filter(col("age") > 25) \
           .groupBy("city") \
           .agg(avg("salary").alias("avg_salary"))
```

**Dataset — Spark 1.6 (Scala/Java 전용)**

Dataset은 DataFrame의 타입 안전성 버전입니다. Scala/Java에서 사용 가능하며, 컴파일 타임에 타입 오류를 잡을 수 있습니다. Python에서는 DataFrame이 사실상 Dataset과 동일하게 동작합니다.

| API | 타입 안전성 | 언어 | 최적화 |
|-----|-----------|------|--------|
| **RDD** | 있음 | 모든 언어 | 수동 |
| **DataFrame** | 없음 | 모든 언어 | 자동 (Catalyst) |
| **Dataset** | 있음 | Scala/Java | 자동 (Catalyst) |

> 💡 **현업에서의 선택**: 현재 대부분의 Spark 코드는 **DataFrame API** (Python/SQL)를 사용합니다. RDD는 DataFrame으로 표현하기 어려운 복잡한 커스텀 로직에만 제한적으로 사용합니다. Databricks에서는 대부분의 작업을 SQL이나 PySpark DataFrame으로 처리할 수 있습니다.

참고: [Apache Spark 공식 문서](https://spark.apache.org/docs/latest/) | [Databricks: Apache Spark 최적화 가이드](https://docs.databricks.com/en/optimizations/index.html)

---

## 데이터 레이크 vs 데이터 웨어하우스

### 데이터 웨어하우스 (Data Warehouse)

데이터 웨어하우스는 분석을 위해 여러 소스의 정형 데이터를 통합 저장하는 시스템입니다. 1990년대부터 기업 분석(Business Intelligence)의 핵심이었습니다.

| 특성 | 내용 |
|------|------|
| **스키마** | Schema-on-Write: 데이터를 저장하기 전에 스키마를 정의 |
| **데이터 형태** | 정형 데이터 (SQL 테이블) |
| **처리 방식** | OLAP (Online Analytical Processing) |
| **저장 비용** | 높음 (전용 스토리지 + 컴퓨팅) |
| **성능** | SQL 분석에 최적화, 매우 빠름 |
| **유연성** | 낮음 (스키마 변경 어려움) |
| **대표 솔루션** | Teradata, Oracle, Snowflake, BigQuery, Redshift |

**장점**: 높은 쿼리 성능, 강력한 거버넌스, ACID 트랜잭션, BI 도구와 완벽한 통합
**단점**: 비구조화 데이터 처리 불가, 높은 비용, 스키마 변경의 어려움

### 데이터 레이크 (Data Lake)

데이터 레이크는 정형, 반정형, 비정형 데이터를 원본 형태 그대로 저장하는 중앙 저장소입니다. 2010년대 Hadoop의 등장과 함께 대중화되었으며, 이후 클라우드 오브젝트 스토리지(S3, ADLS, GCS)가 그 역할을 대신하게 되었습니다.

| 특성 | 내용 |
|------|------|
| **스키마** | Schema-on-Read: 데이터를 읽을 때 스키마를 적용 |
| **데이터 형태** | 정형 + 반정형 + 비정형 모두 가능 |
| **저장 포맷** | CSV, JSON, Parquet, Avro, 이미지, 동영상 등 |
| **저장 비용** | 낮음 (오브젝트 스토리지는 매우 저렴) |
| **성능** | 정형 데이터 쿼리는 웨어하우스보다 느릴 수 있음 |
| **유연성** | 높음 (어떤 데이터든 원본 그대로 저장) |
| **대표 솔루션** | AWS S3, Azure ADLS, Google GCS + Hadoop/Spark |

**장점**: 유연성, 낮은 저장 비용, ML을 위한 비정형 데이터 지원, 오픈 포맷
**단점**: 데이터 품질 보장 어려움, ACID 없음, "데이터 늪(Data Swamp)" 위험

> 💡 **데이터 늪(Data Swamp)이란?** 레이크에 데이터를 마구 쌓기만 하고 거버넌스와 카탈로그가 부재하면, 어떤 데이터가 어디에 있는지, 신뢰할 수 있는 데이터인지 알 수 없게 됩니다. 많은 기업이 데이터 레이크 구축 후 "데이터는 많은데 쓸 수 있는 데이터는 없다"는 상황에 빠지는 원인입니다.

### 왜 레이크하우스 (Lakehouse)가 등장했는가

데이터 레이크와 데이터 웨어하우스는 각각 상충되는 장단점을 갖고 있습니다. 기업들은 두 가지를 모두 운영하는 경우가 많았는데, 이는 비용 중복, 데이터 복제, 동기화 지연 등의 문제를 야기했습니다.

```
기존 아키텍처의 문제점:

[소스 시스템] → [데이터 레이크 (S3)] → [ETL] → [데이터 웨어하우스 (Redshift)]
                      ↓ ML팀                              ↓ SQL 분석팀
                  (비정형 데이터)                       (정형 데이터)

문제: 같은 데이터가 레이크와 웨어하우스에 이중으로 존재
     → 비용 2배, 동기화 지연, 데이터 불일치 위험
```

**레이크하우스** 는 데이터 레이크의 저비용·유연성과 데이터 웨어하우스의 ACID 트랜잭션·성능·거버넌스를 단일 플랫폼에서 제공하는 개념입니다. Databricks가 2020년 논문과 함께 공식화했으며, Delta Lake가 핵심 기술입니다.

| 특성 | 데이터 웨어하우스 | 데이터 레이크 | 레이크하우스 |
|------|----------------|-------------|------------|
| 저장 비용 | 높음 | 낮음 | 낮음 |
| ACID 트랜잭션 | 있음 | 없음 | 있음 |
| 스키마 강제 | 있음 | 없음 | 있음 (선택적) |
| 비정형 데이터 | 불가 | 가능 | 가능 |
| BI/SQL 성능 | 매우 빠름 | 느림 | 빠름 (Photon/최적화) |
| ML/AI 지원 | 제한적 | 가능 | 가능 |
| 데이터 복제 | 필요 | 원본 저장 | 원본 저장 |

참고: [Databricks: What is a Data Lakehouse?](https://www.databricks.com/glossary/data-lakehouse) | [논문: Lakehouse: A New Generation of Open Platforms](https://www.cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf)

---

## 테이블 포맷 전쟁 — Delta Lake vs Iceberg vs Hudi

### 왜 오픈 테이블 포맷이 필요한가

클라우드 오브젝트 스토리지(S3 등)에 데이터를 저장하면 비용이 낮지만, 기본적으로는 단순한 파일 저장소에 불과합니다. Parquet 파일을 S3에 저장하면:

- **ACID 없음**: 두 작업이 동시에 파일을 수정하면 데이터 손상 가능
- **업데이트/삭제 불가**: Parquet은 불변(Immutable) 파일 포맷
- **스키마 변경 어려움**: 컬럼 추가/변경이 복잡
- **타임 트래블 없음**: 과거 데이터 조회 불가

오픈 테이블 포맷은 이런 문제를 해결하기 위해, Parquet 파일 위에 **메타데이터 레이어** 를 추가합니다. 실제 데이터는 Parquet으로 저장하되, 변경 이력, 스키마, 트랜잭션 로그를 별도로 관리합니다.

### 세 포맷 비교

| 특성 | Delta Lake | Apache Iceberg | Apache Hudi |
|------|-----------|---------------|-------------|
| **개발사** | Databricks (2019년 오픈소스화) | Netflix → Apache (2020년) | Uber → Apache (2020년) |
| **ACID 트랜잭션** | 있음 | 있음 | 있음 |
| **스키마 진화** | 있음 | 있음 (더 유연함) | 있음 |
| **타임 트래블** | 있음 | 있음 | 있음 |
| **파티션 진화** | 제한적 | 매우 유연함 | 있음 |
| **엔진 지원** | Spark 중심, 타 엔진 점진적 지원 | 광범위 (Spark, Flink, Trino, Snowflake 등) | Spark, Flink 중심 |
| **강점** | Spark 생태계, 성숙도, Databricks 통합 | 엔진 독립성, 파티션 유연성 | 증분 처리(Incremental Processing) |
| **채택 현황** | Databricks, Microsoft Fabric | Snowflake, AWS, Google 지원 | Uber, 증분 처리 워크로드 |

**Delta Lake의 핵심 기능**

```
Delta Lake 디렉토리 구조:
/my-table/
  _delta_log/        ← 트랜잭션 로그 (JSON/Parquet)
    00000.json       ← 1번째 트랜잭션
    00001.json       ← 2번째 트랜잭션
    ...
  part-0000.parquet  ← 실제 데이터
  part-0001.parquet
  ...
```

- **Transaction Log**: 모든 변경 사항을 JSON으로 기록하여 ACID 보장
- **Time Travel**: `VERSION AS OF n` 또는 `TIMESTAMP AS OF 'date'` 구문으로 과거 스냅샷 조회
- **MERGE INTO**: Upsert (Insert + Update) 연산을 단일 명령으로 처리
- **Z-Ordering**: 자주 필터링하는 컬럼 기준으로 데이터를 클러스터링하여 쿼리 성능 향상
- **Auto Optimize / VACUUM**: 소규모 파일 자동 병합, 오래된 파일 정리

**Iceberg의 차별점**

Iceberg는 Netflix가 수만 개의 테이블, PB급 데이터를 관리하면서 필요한 기능을 설계했습니다.

- **Hidden Partitioning**: 파티션 컬럼을 쿼리에 명시하지 않아도 자동으로 최적화
- **Partition Evolution**: 테이블 재작성 없이 파티션 전략 변경 가능
- **엔진 독립**: Spark, Flink, Trino, Hive, Snowflake 등 다양한 엔진이 동일한 Iceberg 테이블을 읽고 쓸 수 있음

### UniForm — 테이블 포맷의 수렴

포맷 전쟁의 최종 결론은 "공존"입니다. Databricks는 **UniForm (Universal Format)** 을 통해 Delta Lake 테이블을 Iceberg 또는 Hudi 형식으로도 읽을 수 있게 지원합니다.

```sql
-- UniForm 활성화: Snowflake, Trino 등이 Delta 테이블을 Iceberg로 읽을 수 있음
CREATE TABLE my_table (id INT, name STRING)
TBLPROPERTIES ('delta.universalFormat.enabledFormats' = 'iceberg');
```

이는 벤더 종속(Vendor Lock-in) 우려를 줄이고, 멀티 엔진 환경에서 동일한 데이터를 공유할 수 있게 합니다.

참고: [Delta Lake 공식 문서](https://docs.delta.io/latest/index.html) | [Apache Iceberg 공식 문서](https://iceberg.apache.org/docs/latest/) | [Databricks: UniForm](https://docs.databricks.com/en/delta/uniform.html)

---

## 모던 데이터 스택 (Modern Data Stack)

### 개요

**모던 데이터 스택** 은 2010년대 후반 클라우드 네이티브 환경에서 부상한 데이터 파이프라인 아키텍처입니다. 핵심 원칙은 **전문화** 입니다. 하나의 대형 플랫폼 대신, 각 단계에서 최적의 도구를 조합(Best-of-Breed)합니다.

```
모던 데이터 스택의 전형적인 구성:

[소스] → [수집: Fivetran/Airbyte] → [저장: Snowflake/BigQuery/Databricks]
                                              ↓
                               [변환: dbt] → [데이터 마트]
                                              ↓
                              [BI: Tableau/Looker/Power BI]
```

### ELT vs ETL

전통적인 데이터 파이프라인은 **ETL (Extract → Transform → Load)** 방식이었습니다. 소스에서 데이터를 추출하고, 별도의 변환 서버에서 처리한 뒤, 웨어하우스에 저장했습니다.

모던 데이터 스택에서는 **ELT (Extract → Load → Transform)** 가 표준입니다. 클라우드 웨어하우스의 처리 능력이 강력해지면서, 원본 데이터를 먼저 적재하고 웨어하우스 내부에서 SQL로 변환하는 방식이 더 효율적입니다.

| 비교 | ETL | ELT |
|------|-----|-----|
| **변환 시점** | 적재 전 | 적재 후 |
| **변환 위치** | 별도 서버 | 웨어하우스 내부 |
| **도구** | SSIS, Informatica, Talend | dbt + Snowflake/BigQuery/Databricks |
| **유연성** | 낮음 (스키마 변경 어려움) | 높음 (원본 항상 보존) |
| **비용** | 변환 서버 별도 | 웨어하우스 컴퓨팅 비용 |

### dbt (data build tool)

**dbt** 는 ELT의 T(Transform) 단계를 담당하는 오픈소스 SQL 변환 프레임워크입니다. 2016년 Fishtown Analytics(현 dbt Labs)에서 만들었습니다.

**핵심 개념**:
- SQL SELECT 문으로 데이터 변환 모델을 정의
- 모델 간 의존성을 자동으로 해결하여 올바른 순서로 실행
- 데이터 테스트(Not Null, Unique, Referential Integrity 등) 내장
- 문서화 자동 생성, 데이터 리니지(Lineage) 시각화

```sql
-- dbt 모델 예시: models/marts/orders_summary.sql
{{ config(materialized='table') }}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}   -- 다른 dbt 모델 참조
),
customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
)

SELECT
    o.order_id,
    c.customer_name,
    o.amount,
    o.created_at
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed'
```

> 💡 **왜 dbt가 급성장했나?** 기존에 데이터 변환은 Python이나 Spark를 잘 아는 엔지니어가 담당했습니다. dbt는 SQL을 아는 분석가도 직접 데이터 파이프라인을 관리할 수 있게 했고, Git 기반 버전 관리, 테스트, 문서화를 하나의 도구로 제공했습니다. **"분석가의 소프트웨어 엔지니어링"** 이라는 개념이 대중화된 것입니다.

참고: [dbt 공식 문서](https://docs.getdbt.com/) | [dbt + Databricks 통합 가이드](https://docs.databricks.com/en/partners/prep/dbt.html)

### Fivetran / Airbyte — 관리형 수집 도구

**Fivetran** 은 "EL(Extract-Load) as a Service"입니다. 400개 이상의 소스 커넥터를 제공하며, 스키마 변경 자동 감지, CDC(Change Data Capture), 데이터 정규화를 자동으로 처리합니다.

**Airbyte** 는 Fivetran의 오픈소스 대안입니다. 자체 호스팅이 가능하며, 커스텀 커넥터 개발도 지원합니다.

| 비교 | Fivetran | Airbyte | Lakeflow Connect |
|------|---------|---------|----------------|
| **유형** | SaaS | 오픈소스 | Databricks 관리형 |
| **커넥터 수** | 400+ | 300+ | 주요 소스 집중 |
| **설치/운영** | 불필요 | 자체 관리 | 불필요 |
| **비용** | 행(row) 기준 | 오픈소스 무료 | Databricks에 포함 |
| **최적 용도** | SaaS 소스 수집 | 비용 절감, 커스텀 | Databricks 통합 환경 |

### 오케스트레이션 — Apache Airflow

**Apache Airflow** 는 Airbnb에서 2014년 개발한 오픈소스 워크플로우 오케스트레이션 도구입니다. 데이터 파이프라인을 **DAG (Directed Acyclic Graph)** 으로 정의하고, 스케줄링, 의존성 관리, 재시도, 알림을 처리합니다.

```python
# Airflow DAG 예시
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG('daily_etl', start_date=datetime(2024, 1, 1), schedule_interval='@daily') as dag:
    extract = PythonOperator(task_id='extract', python_callable=extract_data)
    transform = PythonOperator(task_id='transform', python_callable=transform_data)
    load = PythonOperator(task_id='load', python_callable=load_data)

    extract >> transform >> load  # 의존성 정의
```

현재는 **Prefect**, **Dagster** 같은 더 현대적인 대안도 많습니다. Databricks 환경에서는 **Lakeflow Jobs** 가 Airflow를 대체하거나 함께 사용됩니다.

---

## 영역별 기술 지도

| 영역 | 기술 | 설명 |
|------|------|------|
| **데이터 수집** | Apache Kafka | 실시간 스트리밍 수집 |
|  | Apache Flink | 스트림 처리 |
|  | AWS Kinesis | AWS 스트리밍 서비스 |
|  | Fivetran / Airbyte | ELT 수집 도구 |
|  | Lakeflow Connect | Databricks 수집 서비스 |
| **데이터 저장** | Delta Lake | 레이크하우스 테이블 포맷 |
|  | Apache Iceberg | 오픈 테이블 포맷 |
|  | Apache Hudi | 오픈 테이블 포맷 |
|  | AWS S3 / ADLS / GCS | 클라우드 오브젝트 스토리지 |
| **데이터 처리** | Apache Spark | 대규모 분산 처리 |
|  | dbt | SQL 기반 변환 (ELT) |
|  | SQL (Databricks SQL 등) | SQL 기반 분석 |
| **오케스트레이션** | Apache Airflow | 워크플로우 DAG 스케줄링 |
|  | Lakeflow Jobs | Databricks 통합 스케줄러 |
| **데이터 분석/시각화** | Databricks AI/BI | 대시보드 + Genie |
|  | Tableau / Power BI / Looker | 외부 BI 도구 |
| **ML/AI** | MLflow | ML 실험 관리 |
|  | PyTorch / TensorFlow | 딥러닝 프레임워크 |
|  | Agent Framework | AI 에이전트 개발 |

---

## 데이터 수집 (Ingestion) 솔루션

### 스트리밍 수집

| 솔루션 | 개발사 | 설명 |
|--------|--------|------|
| **Apache Kafka** | LinkedIn → Apache | 가장 널리 사용되는 분산 메시지 스트리밍 플랫폼입니다. 초당 수백만 건의 이벤트를 처리할 수 있습니다 |
| **Confluent** | Confluent | Kafka의 상용 관리형 서비스입니다. Kafka 창시자가 설립했습니다 |
| **Amazon Kinesis** | AWS | AWS의 실시간 스트리밍 서비스입니다 |
| **Azure Event Hubs** | Microsoft | Azure의 이벤트 스트리밍 서비스입니다. Kafka API와 호환됩니다 |
| **Google Pub/Sub** | Google | Google Cloud의 메시지 서비스입니다 |

### 배치/CDC 수집

| 솔루션 | 유형 | 설명 |
|--------|------|------|
| **Fivetran** | SaaS | 400+ 소스에서 자동으로 데이터를 수집하는 관리형 ELT 서비스입니다 |
| **Airbyte** | 오픈소스 | Fivetran의 오픈소스 대안입니다. 300+ 커넥터를 제공합니다 |
| **Lakeflow Connect** | Databricks | Databricks의 관리형 수집 서비스입니다. DB, SaaS에서 CDC 기반 수집을 지원합니다 |
| **AWS DMS** | AWS | 데이터베이스 마이그레이션 및 CDC 복제 서비스입니다 |
| **Debezium** | 오픈소스 | CDC(Change Data Capture)에 특화된 오픈소스 커넥터입니다 |
| **Apache NiFi** | 오픈소스 | 시각적 데이터 흐름 관리. 드래그&드롭으로 파이프라인을 구성합니다 |

> 💡 **데이터 수집 도구 선택의 현실**: 현업에서 가장 많이 쓰이는 조합은 **(1) Kafka(실시간 이벤트)** + **(2) Fivetran 또는 Lakeflow Connect(SaaS/DB 배치 수집)** + **(3) Auto Loader(파일 수집)** 입니다. 세 가지를 용도에 맞게 조합하면 대부분의 수집 요구사항을 커버할 수 있습니다.

> 💡 **CDC(Change Data Capture)란?** 데이터베이스에서 발생하는 변경 사항(INSERT, UPDATE, DELETE)을 실시간으로 감지하여 다른 시스템에 전달하는 기술입니다. 전체 데이터를 매번 복사하는 대신, 변경된 부분만 전달하므로 효율적입니다.

---

## 데이터 처리 (Processing) 엔진

| 엔진 | 유형 | 설명 |
|------|------|------|
| **Apache Spark** | 배치 + 스트리밍 | 가장 널리 사용되는 분산 처리 엔진입니다. Databricks의 핵심입니다 |
| **Apache Flink** | 스트리밍 우선 | 진정한 이벤트 단위 스트리밍 처리에 강점이 있습니다 |
| **Trino (구 PrestoSQL)** | 대화형 쿼리 | 빠른 SQL 쿼리 엔진입니다. 여러 데이터 소스를 연합 쿼리할 수 있습니다 |
| **dbt** | SQL 변환 | SQL 기반 데이터 변환 도구입니다. ELT의 T(Transform)에 특화되어 있습니다 |
| **Apache Beam** | 통합 모델 | Google이 개발. 배치/스트리밍 통합 프로그래밍 모델을 제공합니다 |
| **Polars** | 로컬 분석 | Rust 기반의 고성능 DataFrame 라이브러리. 단일 머신에서 Pandas보다 10~100배 빠릅니다 |

> 💡 **Spark vs Polars**: "우리 데이터가 수 GB인데도 Spark를 써야 하나요?"라는 질문을 자주 받습니다. 데이터가 수십 GB 이하이고 단일 머신에서 처리 가능하다면, Polars나 DuckDB가 더 가볍고 빠를 수 있습니다. Spark는 수 TB~PB 규모의 데이터, 또는 여러 사용자가 동시에 접근하는 환경에서 진가를 발휘합니다. Databricks의 Serverless 노트북에서는 소규모 데이터도 빠르게 처리할 수 있으므로, 실무에서는 "일관된 환경"의 이점을 위해 Databricks를 사용하는 경우가 많습니다.

---

## 클라우드 데이터 플랫폼 비교

### 주요 플랫폼 상세 비교

| 플랫폼 | 제공사 | 핵심 강점 | 아키텍처 | 테이블 포맷 |
|--------|--------|-----------|---------|------------|
| **Databricks** | Databricks | 통합 레이크하우스, Spark/ML/AI | 레이크하우스 | Delta Lake |
| **Snowflake** | Snowflake | 사용 편의성, SQL 성능, 멀티 클라우드 | 클라우드 DW | 독자 + Iceberg |
| **BigQuery** | Google | 서버리스, 자동 확장, ML 통합 | 서버리스 DW | 독자 + BigLake |
| **Redshift** | AWS | AWS 생태계 통합, 비용 효율 | 클라우드 DW | 독자 |
| **Synapse Analytics** | Microsoft | Azure 통합, Spark + SQL 결합 | 통합 분석 | 독자 |
| **Microsoft Fabric** | Microsoft | OneLake 기반 통합 플랫폼 | 레이크하우스 | Delta Lake |

**Databricks vs Snowflake** — 가장 자주 받는 비교 질문

| 관점 | Databricks | Snowflake |
|------|-----------|----------|
| **강점** | ML/AI, 스트리밍, 오픈소스, 유연성 | SQL 성능, 사용 편의성, 시장 성숙도 |
| **주요 사용자** | 데이터 엔지니어, ML 엔지니어, 데이터 과학자 | SQL 분석가, 데이터 분석가, BI 팀 |
| **비용 모델** | DBU (Databricks Unit) | 크레딧 기반 |
| **스토리지** | 고객 소유 클라우드 스토리지 | Snowflake 관리 스토리지 |
| **벤더 종속** | 낮음 (오픈 포맷) | 중간 |
| **Spark 지원** | 핵심 엔진 | 없음 (Snowpark로 제한적 지원) |

> 💡 **클라우드 데이터 플랫폼 선택의 현실**: 실무에서 플랫폼을 선택할 때, 순수 기술력보다 "** 우리 팀이 가장 잘 쓸 수 있는 도구**"가 더 중요합니다. SQL 분석가가 대다수인 팀에서는 Snowflake가, ML/AI 엔지니어가 많은 팀에서는 Databricks가, Microsoft 생태계에 깊이 투자한 조직에서는 Fabric이 자연스러운 선택입니다.

---

## AI & ML 플랫폼

| 플랫폼 | 제공사 | 설명 |
|--------|--------|------|
| **MLflow** | Databricks (오픈소스) | ML 실험 추적, 모델 관리, GenAI 트레이싱을 지원합니다 |
| **SageMaker** | AWS | AWS의 관리형 ML 플랫폼입니다 |
| **Vertex AI** | Google | Google Cloud의 ML 플랫폼입니다 |
| **Azure ML** | Microsoft | Azure의 ML 플랫폼입니다 |
| **Hugging Face** | Hugging Face | 오픈소스 ML 모델 허브. LLM 생태계의 중심입니다 |
| **Weights & Biases** | W&B | 실험 추적에 특화된 ML 도구입니다 |

> 💡 **MLflow vs 다른 ML 플랫폼**: MLflow는 Databricks가 만든 오픈소스이지만, Databricks 없이도 사용할 수 있습니다. SageMaker나 Vertex AI는 각 클라우드에 종속되지만, MLflow는 어디서든 실행 가능합니다. 이러한 오픈소스 전략은 Databricks의 핵심 철학 중 하나입니다. 현업에서는 MLflow를 "실험 추적의 표준"으로 사용하는 팀이 매우 많으며, Databricks를 쓰지 않는 회사에서도 MLflow만 독립적으로 사용하는 경우가 있습니다.

---

## 데이터 거버넌스

| 솔루션 | 유형 | 설명 |
|--------|------|------|
| **Unity Catalog** | Databricks (오픈소스) | Databricks의 통합 거버넌스 솔루션입니다 |
| **Collibra** | 상용 | 엔터프라이즈 데이터 거버넌스 플랫폼입니다 |
| **Alation** | 상용 | 데이터 카탈로그 + 거버넌스 플랫폼입니다 |
| **Apache Atlas** | 오픈소스 | Hadoop 생태계의 메타데이터 관리 도구입니다 |
| **Purview** | Microsoft | Azure의 데이터 거버넌스 및 카탈로그 서비스입니다 |
| **AWS Glue Data Catalog** | AWS | AWS의 관리형 메타데이터 카탈로그입니다 |

> 💡 **데이터 거버넌스가 왜 중요한가요?** GDPR(유럽 개인정보보호법), 개인정보보호법 등 규제가 강화되면서, "누가, 어떤 데이터를, 언제, 왜 접근했는지"를 추적할 수 있어야 합니다. 거버넌스 도구 없이 운영하면 감사(Audit) 시 대응이 불가능합니다. Unity Catalog는 이를 Databricks 플랫폼 안에서 자연스럽게 해결합니다.

---

## 빅 플레이어 포지션 맵

**데이터 플랫폼 포지션 비교**

| 플랫폼 | SQL/분석 ↔ 엔지니어링/ML | 온프레미스 ↔ 클라우드 | 특징 |
|--------|----------------------|---------------------|------|
| **Databricks** | 엔지니어링/ML 중심 | 클라우드 네이티브 | Spark 기반 통합 플랫폼 |
| **Snowflake** | SQL/분석 중심 | 클라우드 네이티브 | 클라우드 데이터 웨어하우스 |
| **BigQuery** | SQL/분석 중심 | 클라우드 네이티브 | Google 서버리스 분석 |
| **Redshift** | SQL/분석 중심 | 클라우드 | AWS 데이터 웨어하우스 |
| **Microsoft Fabric** | 중간 | 클라우드 네이티브 | Microsoft 통합 분석 |
| **Hadoop** | 엔지니어링 중심 | 온프레미스 | 레거시 분산 처리 |
| **Teradata** | SQL/분석 중심 | 온프레미스 | 레거시 데이터 웨어하우스 |

---

## 실무 경험으로 본 각 도구 — 써본 사람만 아는 이야기

### Hadoop: 설치에 3일, 운영에 평생

2010년대 초반, Hadoop은 빅데이터의 대명사였습니다. 하지만 실제로 운영해본 사람이라면 이런 기억이 있을 것입니다.

| 항목 | 현실 |
|------|------|
| **설치** | NameNode, DataNode, ResourceManager, NodeManager, ZooKeeper, Hive Metastore... 각각 설정 파일이 수백 줄입니다. 클러스터 한 대를 띄우는 데 3~5일이 걸렸습니다 |
| **운영** | HDFS의 DataNode가 하나 죽으면 리밸런싱이 시작되는데, 대용량 데이터를 복제하느라 네트워크 대역폭을 잡아먹어 다른 작업이 느려집니다 |
| **YARN 튜닝** | 메모리 설정을 잘못하면 "Container killed by YARN" 에러가 끊이질 않습니다. `yarn.nodemanager.resource.memory-mb`, `mapreduce.map.memory.mb` 등 수십 개의 파라미터를 조정해야 합니다 |
| **Hive** | SQL처럼 보이지만, 간단한 SELECT도 MapReduce 잡으로 변환되어 30초~수 분이 걸렸습니다 |

### Spark: 30분이면 시작, 하지만 튜닝은 별도

Spark는 Hadoop의 고통을 크게 줄여주었습니다.

| 항목 | Hadoop 대비 개선 |
|------|-----------------|
| **속도** | 인메모리 처리로 Hive 대비 10~100배 빠릅니다 |
| **API** | DataFrame API가 직관적입니다. SQL도 바로 사용 가능합니다 |
| **설치** | `pip install pyspark` 한 줄로 로컬에서 바로 시작할 수 있습니다 |

하지만 직접 운영하면 다른 문제가 생깁니다.

| 문제 | 설명 |
|------|------|
| **OOM (Out of Memory)** | Shuffle 과정에서 메모리 부족이 가장 흔한 에러입니다. `spark.sql.shuffle.partitions`, `spark.executor.memory` 튜닝이 필수입니다 |
| **데이터 스큐(Skew)** | 특정 키에 데이터가 몰리면 한 executor만 일하고 나머지는 놀게 됩니다. 현업에서 가장 골치아픈 성능 이슈입니다 |
| **클러스터 관리** | EMR이나 직접 구축한 Spark 클러스터는 버전 업그레이드, 라이브러리 관리, 노드 추가/제거를 직접 해야 합니다 |

> 💡 **Databricks와의 차이**: Databricks는 이런 Spark 운영의 고통을 해결하기 위해 만들어졌습니다. 클러스터 자동 확장, Photon 엔진(C++ 기반 고성능 실행 엔진), Adaptive Query Execution(자동 튜닝), 데이터 스큐 자동 감지 등을 플랫폼 수준에서 제공합니다.

### 왜 어떤 도구는 사라지고, 어떤 도구는 살아남았는가?

| 도구 | 상태 | 이유 |
|------|------|------|
| **MapReduce** | 사실상 사망 | Spark가 더 빠르고, 더 쉬운 API를 제공했기 때문입니다 |
| **Apache Pig** | 사실상 사망 | Spark DataFrame/SQL이 Pig Latin보다 더 직관적이었습니다 |
| **Apache Oozie** | 레거시 | Airflow, Databricks Jobs 등 더 나은 워크플로우 도구가 등장했습니다 |
| **Apache Hive** | 축소 | Hive Metastore는 여전히 사용되지만, 쿼리 엔진은 Spark SQL, Trino 등으로 대체되었습니다 |
| **Apache Kafka** | 건재 | 실시간 이벤트 스트리밍이라는 고유한 영역을 확보하고 있습니다 |
| **Apache Spark** | 건재 | 배치 + 스트리밍 + ML을 하나의 엔진으로 처리하는 생태계의 힘입니다 |
| **Apache Flink** | 성장 | True Streaming 분야에서 Spark와 차별화됩니다 |
| **dbt** | 급성장 | SQL 기반 변환에 집중하여 데이터 분석가들에게 큰 호응을 얻었습니다 |

> 💡 **핵심 교훈**: 살아남은 도구들의 공통점은 **(1) 명확한 차별점이 있고, (2) 활발한 커뮤니티가 있으며, (3) 클라우드 환경에 적응했다** 는 것입니다. 반면 사라진 도구들은 "더 나은 대안이 등장했는데, 전환 비용이 높지 않았던" 도구들입니다.

---

## 현재 생태계에서 Databricks의 위치

Databricks는 "데이터 플랫폼의 스위스 아미 나이프"라고 할 수 있습니다. 아래와 같이 다른 도구들을 하나의 플랫폼으로 대체하거나 통합합니다.

| 기존 도구 조합 | Databricks에서는 |
|---------------|-----------------|
| EMR + Spark (직접 관리) | Databricks Runtime (관리형 Spark) |
| Airflow + Cron | Lakeflow Jobs (워크플로우 오케스트레이션) |
| Fivetran + Airbyte | Lakeflow Connect (관리형 커넥터) |
| Redshift / Snowflake (SQL 분석) | Databricks SQL + Photon |
| SageMaker / Vertex AI (ML) | MLflow + Model Serving |
| Apache Atlas + Ranger (거버넌스) | Unity Catalog |
| Tableau / Looker (BI) | AI/BI Dashboard + Genie |

물론 Databricks가 모든 도구를 100% 대체하는 것은 아닙니다. 예를 들어, 복잡한 시각화가 필요하면 Tableau가 여전히 강력하고, 초저지연 스트리밍이 필요하면 Flink가 적합합니다. 하지만 "80%의 워크로드를 하나의 플랫폼에서 처리할 수 있다"는 것은 운영 복잡성을 크게 줄여줍니다.

> 💡 **현업에서는 이렇게 합니다**: 대부분의 기업에서 Databricks를 도입할 때 "모든 기존 도구를 Databricks로 교체"하지는 않습니다. 가장 흔한 패턴은 **(1) 기존 Kafka는 그대로 유지**, **(2) ETL/ML은 Databricks로 통합**, **(3) BI는 기존 Tableau를 Databricks SQL에 연결** 하는 방식입니다. 점진적 마이그레이션이 현실적이며, 한 번에 모든 것을 바꾸려 하면 리스크가 커집니다.

---

## 기술 선택 가이드

### 규모/용도별 적합한 기술 스택

| 상황 | 권장 스택 | 이유 |
|------|---------|------|
| **소규모 (수 GB, 팀 1~3명)** | PostgreSQL + dbt + Metabase | 단순하고 저비용. 과도한 복잡성 없이 SQL로 충분 |
| **중소규모 (수십 GB~수 TB, 팀 3~10명)** | Snowflake + dbt + Tableau 또는 Databricks SQL | SQL 중심이면 Snowflake, ML도 필요하면 Databricks |
| **대규모 (TB~PB, 팀 10명 이상)** | Databricks (레이크하우스) + Kafka + Airflow | 통합 플랫폼으로 운영 복잡성 최소화 |
| **실시간 스트리밍 중심** | Kafka + Flink + Databricks (배치 레이어) | Lambda/Kappa 아키텍처 |
| **ML/AI 중심** | Databricks + MLflow + Hugging Face | Feature Store, 모델 서빙, 실험 추적 통합 |
| **AWS 생태계 올인** | Redshift + Glue + SageMaker | AWS 서비스 간 통합 최적화 |
| **Google Cloud 중심** | BigQuery + Dataflow + Vertex AI | BigQuery ML로 SQL 안에서 ML 가능 |
| **Microsoft 생태계** | Microsoft Fabric 또는 Azure Databricks | Power BI, Azure AD와 자연스러운 통합 |
| **오픈소스 중심 (벤더 독립)** | Spark + Iceberg + Trino + MLflow | 클라우드 종속 최소화, 자체 운영 역량 필요 |

> 💡 **현업에서는 이렇게 합니다**: 기술 선택은 "현재 팀의 역량"과 "데이터 규모"를 기준으로 해야 합니다. 데이터가 수 GB이고 팀이 2~3명이면 PostgreSQL + dbt로 충분합니다. 데이터가 TB~PB 규모이고 팀이 10명 이상이면 Databricks나 Snowflake 같은 클라우드 데이터 플랫폼이 필수적입니다. "나중에 커질 것을 대비해서 처음부터 큰 도구를 도입하자"는 접근은 대부분 과도한 복잡성과 비용만 초래합니다.

---

## 도구 선택 시 피해야 할 함정

현업에서 기술 선택 시 흔히 빠지는 함정들입니다.

### 함정 1: "유행하니까" 도입

| 잘못된 판단 | 현실 |
|------------|------|
| "Netflix가 Iceberg를 쓰니까 우리도" | Netflix는 수만 개의 테이블과 PB급 데이터를 관리합니다. 우리 회사의 데이터 규모와 요구사항이 같은지 먼저 확인해야 합니다 |
| "모두 Kafka를 쓰니까 우리도 도입" | 일일 이벤트가 수만 건이면 Kafka 대신 SQS/Cloud Functions로 충분합니다 |
| "dbt가 대세니까 도입" | 팀에 SQL을 잘 하는 분석가가 있어야 효과가 있습니다 |

### 함정 2: "오픈소스가 무료"라는 착각

| 항목 | 오픈소스 직접 운영 | 관리형 서비스 (Databricks 등) |
|------|------------------|---------------------------|
| 라이선스 비용 | 무료 | 유료 |
| 인프라 비용 | 직접 부담 | 포함 또는 별도 |
| 운영 인력 | 전담 2~3명 필요 | 최소화 |
| 장애 대응 | 직접 해결 | 벤더 지원 |
| 총 소유 비용 (TCO) | 대부분 **더 비쌈** | 예측 가능 |

### 함정 3: "한 도구로 모든 것을"

만능 도구는 없습니다. Databricks가 아무리 통합 플랫폼이라 해도, 초저지연 OLTP(온라인 트랜잭션)나 복잡한 시각화에는 전문 도구가 더 적합합니다. 각 도구의 강점과 약점을 이해하고, 적재적소에 배치하는 것이 핵심입니다.

---

## 데이터 오케스트레이션 도구

데이터 파이프라인을 스케줄링하고 관리하는 도구도 생태계의 중요한 부분입니다.

| 도구 | 유형 | 설명 |
|------|------|------|
| **Apache Airflow** | 오픈소스 | 가장 널리 사용되는 워크플로우 오케스트레이션 도구입니다. Python으로 DAG(Directed Acyclic Graph)를 정의합니다 |
| **Lakeflow Jobs** | Databricks | Databricks 내장 스케줄러입니다. 노트북, SQL, Python, JAR 등을 태스크로 조합할 수 있습니다 |
| **Prefect** | 오픈소스/상용 | Airflow의 현대적 대안입니다. Python 네이티브 경험을 제공합니다 |
| **Dagster** | 오픈소스/상용 | 데이터 자산 중심의 오케스트레이션 도구입니다 |
| **AWS Step Functions** | AWS | AWS 서버리스 워크플로우 서비스입니다 |

> 💡 **현업에서의 선택**: Databricks를 주력으로 사용한다면 **Lakeflow Jobs** 가 가장 자연스럽습니다. 클러스터 관리, 재시도, 알림이 통합되어 있습니다. 만약 Databricks 외의 다른 시스템(AWS Lambda, API 호출 등)도 오케스트레이션해야 한다면 **Airflow** 가 적합합니다. 둘을 함께 사용하는 경우(Airflow가 전체 워크플로우를 관리하고, 개별 Databricks 작업은 Lakeflow Jobs로 실행)도 많습니다.

---

## 데이터 품질 도구

데이터의 정확성과 신뢰성을 보장하기 위한 도구도 점점 중요해지고 있습니다.

| 도구 | 유형 | 설명 |
|------|------|------|
| **Great Expectations** | 오픈소스 | Python 기반 데이터 품질 검증 프레임워크입니다 |
| **Soda** | 오픈소스/상용 | SQL 기반 데이터 품질 체크 도구입니다 |
| **SDP Expectations** | Databricks | SDP 파이프라인에서 선언적으로 데이터 품질 규칙을 정의합니다 |
| **Monte Carlo** | 상용 | 데이터 관측성(Data Observability) 플랫폼입니다 |

---

## 정리

| 영역 | 주요 솔루션 | Databricks의 대응 |
|------|-----------|------------------|
| 데이터 수집 | Kafka, Fivetran, Airbyte | Lakeflow Connect, Auto Loader |
| 데이터 저장 | Delta Lake, Iceberg, Hudi | Delta Lake + UniForm |
| 데이터 처리 | Spark, Flink, dbt | Apache Spark (내장) |
| SQL 분석 | Snowflake, BigQuery, Redshift | Databricks SQL |
| BI | Tableau, Power BI, Looker | AI/BI Dashboard + Genie |
| ML/AI | SageMaker, Vertex AI, MLflow | MLflow + Model Serving + Agent Framework |
| 거버넌스 | Collibra, Alation, Glue Catalog | Unity Catalog |

---

## 참고 링크

- [Databricks Blog: Data + AI Platform](https://www.databricks.com/blog)
- [Databricks: What is a Data Lakehouse?](https://www.databricks.com/glossary/data-lakehouse)
- [Apache Spark 공식 문서](https://spark.apache.org/docs/latest/)
- [Delta Lake 공식 문서](https://docs.delta.io/latest/index.html)
- [Apache Iceberg 공식 문서](https://iceberg.apache.org/docs/latest/)
- [dbt 공식 문서](https://docs.getdbt.com/)
- [dbt + Databricks 통합 가이드](https://docs.databricks.com/en/partners/prep/dbt.html)
- [Databricks: UniForm](https://docs.databricks.com/en/delta/uniform.html)
- [논문: Lakehouse — A New Generation of Open Platforms (CIDR 2021)](https://www.cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf)
- [논문: MapReduce — Simplified Data Processing on Large Clusters (Google, 2004)](https://research.google/pubs/mapreduce-simplified-data-processing-on-large-clusters/)
- [Gartner: Cloud Database Management Systems](https://www.gartner.com/reviews/market/cloud-database-management-systems)
- [DB-Engines Ranking](https://db-engines.com/en/ranking)
