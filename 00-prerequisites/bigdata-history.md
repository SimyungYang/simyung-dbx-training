# 빅데이터의 역사 — Google 논문에서 레이크하우스까지

## 빅데이터는 어떻게 시작되었나요?

현재 우리가 사용하는 Databricks, Spark, Delta Lake 같은 기술들은 어느 날 갑자기 등장한 것이 아닙니다. 20년이 넘는 기술 진화의 결과물입니다. 이 역사를 이해하면, 각 기술이 **왜** 만들어졌고, **어떤 문제**를 해결하는지 더 깊이 이해하실 수 있습니다.

---

## 타임라인으로 보는 빅데이터 역사

**빅데이터 기술의 진화 타임라인**

| 연도 | 주요 이벤트 |
|------|-----------|
| 2003 | Google GFS 논문 |
| 2004 | Google MapReduce 논문 |
| 2006 | Apache Hadoop 탄생, AWS S3 및 EC2 출시 |
| 2008 | Hive (SQL on Hadoop) |
| 2009 | Spark 연구 시작 (UC Berkeley) |
| 2010 | Apache Kafka 개발 (LinkedIn) |
| 2012 | AWS Redshift 출시 |
| 2013 | Databricks 설립 |
| 2014 | Apache Spark 1.0 출시, Apache Flink 탄생 |
| 2015 | Spark 2.0 (DataFrame API) |
| 2017 | Snowflake 성장, 클라우드 DW 시대 |
| 2019 | Delta Lake 오픈소스 공개 |
| 2020 | Lakehouse 개념 발표, Unity Catalog |
| 2023 | GenAI 통합, LLM Serving |
| 2024 | AI Agent Framework, Lakebase |

---

## 1단계: Google의 혁신 (2003~2004)

모든 것의 시작은 **Google**의 두 편의 논문입니다.

### Google File System (GFS) — 2003

Google은 전 세계 웹 페이지를 크롤링하여 저장해야 했습니다. 기존의 파일 시스템으로는 이 규모를 감당할 수 없었습니다.

> 💡 **GFS(Google File System)** 는 수천 대의 저렴한 서버에 데이터를 분산 저장하는 파일 시스템입니다. 하나의 파일을 여러 조각으로 나누어 여러 서버에 복제본을 저장하므로, 일부 서버가 고장 나도 데이터가 유실되지 않습니다.

### MapReduce — 2004

데이터를 분산 저장했다면, 분산된 데이터를 **병렬로 처리**하는 방법도 필요했습니다.

> 💡 **MapReduce**는 대용량 데이터를 **Map(분배)** 단계와 **Reduce(집계)** 단계로 나누어 여러 서버에서 동시에 처리하는 프레임워크입니다.

```
예시: 도서관의 모든 책에서 단어 빈도 세기

Map (분배):   각 서버가 맡은 책에서 단어별 횟수를 셈
             서버1: {"data":5, "science":3}
             서버2: {"data":8, "lake":2}

Reduce (집계): 모든 서버의 결과를 합침
             {"data":13, "science":3, "lake":2}
```

---

## 2단계: Hadoop의 시대 (2006~2012)

### Apache Hadoop의 탄생

Google의 논문을 읽은 **Doug Cutting**(Apache Lucene의 개발자)이 이를 오픈소스로 구현한 것이 **Apache Hadoop**입니다.

> 💡 **Apache Hadoop**은 GFS를 구현한 **HDFS(Hadoop Distributed File System)** 와 MapReduce를 구현한 **Hadoop MapReduce**를 핵심으로 하는 오픈소스 빅데이터 프레임워크입니다.

### Hadoop 생태계의 성장

Hadoop을 중심으로 다양한 도구들이 생겨나며 하나의 거대한 생태계가 형성되었습니다.

| 도구 | 연도 | 역할 | 설명 |
|------|------|------|------|
| **HDFS** | 2006 | 분산 저장 | 대용량 파일을 여러 서버에 분산 저장합니다 |
| **MapReduce** | 2006 | 분산 처리 | Map-Reduce 패턴으로 데이터를 병렬 처리합니다 |
| **Hive** | 2008 | SQL on Hadoop | Facebook이 개발. SQL로 Hadoop 데이터를 조회합니다 |
| **Pig** | 2008 | 스크립트 처리 | Yahoo가 개발. 스크립트 언어로 데이터를 처리합니다 |
| **HBase** | 2008 | NoSQL DB | Hadoop 위의 분산 데이터베이스입니다 |
| **ZooKeeper** | 2008 | 코디네이션 | 분산 시스템의 설정 관리와 동기화를 담당합니다 |
| **YARN** | 2012 | 리소스 관리 | 클러스터의 리소스(CPU, 메모리)를 관리합니다 |

### Hadoop의 한계

Hadoop은 혁명적이었지만, 심각한 한계가 있었습니다.

| 한계 | 설명 |
|------|------|
| **느린 처리 속도** | MapReduce는 각 단계의 중간 결과를 디스크에 씁니다. I/O가 매우 많습니다 |
| **복잡한 프로그래밍** | Map과 Reduce 함수를 Java로 직접 작성해야 하여, 간단한 집계도 수십 줄의 코드가 필요합니다 |
| **반복 처리에 비효율** | ML처럼 같은 데이터를 반복 처리하는 작업에 매우 비효율적입니다 |
| **운영 복잡성** | Hadoop 클러스터를 직접 구축하고 운영하는 것이 매우 어렵습니다 |
| **Small File 문제** | 작은 파일이 많으면 NameNode 메모리가 폭발합니다. 파일 1개당 약 150바이트의 메타데이터가 NameNode 힙에 올라가므로, 파일 1억 개면 NameNode에 약 15GB 메모리가 필요합니다 |
| **데이터 형식 난립** | CSV, JSON, Avro, Parquet, ORC 등 다양한 형식이 혼재하여, 어떤 형식으로 저장해야 최적인지 매번 고민해야 했습니다 |

### 실전 인사이트: Hadoop 클러스터를 직접 운영해본 사람만 아는 고통

Hadoop 시대를 직접 경험한 사람으로서, 그 시절의 고통을 구체적으로 전달하고 싶습니다. 오늘날 Spark와 레이크하우스가 왜 혁신인지를 이해하려면, Hadoop이 얼마나 힘들었는지를 알아야 합니다.

**HDFS NameNode 장애 — 새벽 호출의 공포**

HDFS의 **NameNode**는 파일 시스템의 메타데이터(어떤 파일이 어떤 DataNode에 저장되어 있는지)를 관리하는 단일 마스터 노드입니다. 이 NameNode가 죽으면 **클러스터 전체가 멈춥니다**. 데이터는 DataNode에 살아있지만, 메타데이터가 없으니 어디에 무엇이 있는지 아무도 모르는 상태가 됩니다.

```
[실제 인시던트 타임라인]
02:15 AM - NameNode 프로세스 OOM(Out of Memory)으로 크래시
02:17 AM - 모니터링 알림 → 당직 엔지니어 호출
02:35 AM - 엔지니어 접속, NameNode 재시작 시도
02:45 AM - NameNode가 fsimage + edit log를 읽어들이는 중... (파일 수백만 개)
03:20 AM - NameNode 복구 완료 (메타데이터 로딩에 35분 소요)
03:25 AM - DataNode들이 block report를 보내는 중...
03:50 AM - 클러스터 완전 복구 (총 다운타임: 약 1시간 35분)
04:00 AM - 밀린 MapReduce 잡들이 한꺼번에 실행되며 리소스 경쟁 발생
```

HA(High Availability)가 도입되기 전에는 이런 일이 월 1~2회 발생했습니다. HA를 구성해도 Standby NameNode로의 페일오버가 항상 깔끔하지는 않았고, ZooKeeper와의 연동 문제로 "brain split"(양쪽 NameNode가 모두 Active라고 주장하는 상황)이 발생하기도 했습니다.

**MapReduce로 WordCount는 쉽지만 조인은 지옥이었습니다**

MapReduce의 대표 예제인 WordCount는 20줄이면 되지만, **두 개의 데이터셋을 조인**하려면 상황이 완전히 달라집니다.

```java
// MapReduce로 두 테이블 조인하는 의사 코드 (실제로는 200줄 이상)
// 1. Mapper: 양쪽 데이터에 "출처 태그"를 붙여서 emit
// 2. Partitioner: 조인 키로 파티셔닝
// 3. Reducer: 같은 키의 레코드를 모아서 조인
// 4. 만약 한쪽 데이터가 메모리에 안 올라가면? → Map-Side Join 고려
// 5. 데이터 스큐가 있으면? → Salting + Secondary Sort 구현
// 6. NULL 키 처리? → Custom Comparator 작성
// → 간단한 LEFT JOIN 하나에 Java 클래스가 5개 이상 필요
```

오늘날 Spark에서 `df1.join(df2, "key", "left")` 한 줄이면 되는 것을, Hadoop MapReduce에서는 **수백 줄의 Java 코드와 3~4시간의 디버깅**이 필요했습니다. 게다가 중간 결과가 모두 HDFS에 쓰여야 하므로, 3단계 조인(A-B-C)을 하면 중간 파일이 2번 생성되어 I/O 비용이 기하급수적으로 증가했습니다.

**Hadoop 클러스터 운영의 현실**

| 운영 업무 | 빈도 | 소요 시간 | 고통 지수 |
|-----------|------|----------|----------|
| DataNode 디스크 교체 | 주 1~2회 (노드 수십 대면) | 2~4시간 | 중간 |
| NameNode 메모리 튜닝 | 월 1회 | 반나절 | 높음 |
| YARN 큐 설정 조정 | 수시 (팀 간 리소스 분쟁) | 1~2시간 | 매우 높음 |
| Hive 메타스토어 장애 | 월 1~2회 | 1~3시간 | 높음 |
| 버전 업그레이드 | 연 1~2회 | 1~2주 | 극도로 높음 |
| 보안 설정 (Kerberos) | 초기 1회 + 수시 | 수일~수주 | 지옥 |

> 💡 **Kerberos 설정의 악몽**: Hadoop 클러스터에 보안을 적용하려면 Kerberos 인증을 구성해야 했습니다. keytab 파일 관리, 티켓 갱신, 서비스 프린시펄 설정... 설정 하나 틀리면 클러스터 전체가 인증 실패로 멈춥니다. "Kerberos 없이 쓰면 안 되나요?"라고 물으면, 보안팀이 "안 됩니다"라고 답하는 것이 일상이었습니다.

---

## 3단계: Apache Spark의 등장 (2009~2015)

### Spark의 탄생 — 왜 정말 혁신이었는지

UC Berkeley AMPLab의 **Matei Zaharia**가 Hadoop MapReduce의 한계를 극복하기 위해 개발한 것이 **Apache Spark**입니다.

| 비교 항목 | Hadoop MapReduce | Apache Spark |
|-----------|-----------------|--------------|
| 중간 결과 저장 | 디스크 | **메모리** |
| 처리 속도 | 느림 | 최대 **100배** 빠름 |
| 프로그래밍 | Java (복잡) | Python, SQL, Scala (간결) |
| 반복 처리 | 매우 비효율 | 효율적 (메모리 캐싱) |
| 스트리밍 | 별도 도구 필요 | 내장 (Structured Streaming) |
| ML | 별도 도구 (Mahout) | 내장 (MLlib) |

Spark가 정말 혁신이었던 이유를 구체적으로 설명하겠습니다.

**혁신 1: 코드량의 극적 감소**

```python
# Spark에서의 조인 — 1줄
result = orders.join(customers, "customer_id", "left")

# Spark에서의 집계 — 3줄
daily_revenue = (
    orders
    .groupBy("order_date")
    .agg(sum("amount").alias("total_revenue"))
)
```

MapReduce에서 200줄이 필요했던 조인이 **1줄**로, 50줄이 필요했던 집계가 **3줄**로 줄었습니다. 이것만으로도 개발 생산성이 10배 이상 향상되었습니다.

**혁신 2: 인터랙티브 분석의 가능성**

MapReduce는 잡을 제출하고 30분~1시간을 기다려야 결과를 볼 수 있었습니다. Spark는 데이터를 메모리에 캐싱하고 **즉시** 쿼리 결과를 확인할 수 있어, 데이터 탐색과 분석이 비로소 "대화형"으로 가능해졌습니다. 노트북 환경(Databricks Notebook)과 결합되면서, 데이터 분석의 패러다임이 완전히 바뀌었습니다.

**혁신 3: 통합 엔진**

Hadoop 시절에는 배치는 MapReduce, SQL은 Hive, 스트리밍은 Storm, ML은 Mahout으로 각각 별도의 엔진을 배워야 했습니다. Spark는 **하나의 엔진으로 배치, SQL, 스트리밍, ML을 모두 처리**할 수 있었습니다. 기술 스택이 단순해지면서, 팀에 필요한 인력과 학습 비용이 대폭 줄었습니다.

**혁신 4: Lazy Evaluation과 Query Optimization**

Spark의 또 다른 혁신은 **지연 평가(Lazy Evaluation)** 입니다. 변환(transformation)을 호출해도 즉시 실행하지 않고, 액션(action)이 호출될 때 전체 실행 계획을 최적화한 뒤 한 번에 실행합니다. 이는 MapReduce에서는 불가능했던 **쿼리 최적화**를 가능하게 했습니다. Catalyst 옵티마이저와 Tungsten 엔진이 자동으로 실행 계획을 최적화해주므로, 개발자가 직접 실행 순서를 고민하지 않아도 됩니다. 이 아키텍처는 이후 Databricks의 Photon 엔진으로 발전하여, SQL 워크로드에서 기존 Spark 대비 최대 12배 빠른 성능을 제공하게 됩니다.

> 💡 MapReduce에서 "join + filter + aggregate" 3단계를 실행하면 중간 결과가 HDFS에 2번 쓰여야 했습니다. Spark에서는 Catalyst가 filter를 join 앞으로 이동(predicate pushdown)시키고, 불필요한 컬럼을 제거(column pruning)하여 전체 실행 시간을 수십 배 줄입니다.

### Databricks의 설립 (2013)

Spark를 만든 연구팀이 **Databricks**를 설립하여, Spark를 기업이 쉽게 사용할 수 있는 관리형 플랫폼으로 제공하기 시작했습니다.

> 💡 **Databricks가 해결한 핵심 문제**: Spark 자체는 오픈소스로 무료이지만, 이것을 기업 환경에서 안정적으로 운영하려면 클러스터 프로비저닝, 모니터링, 보안 설정, 버전 관리 등 막대한 운영 비용이 필요했습니다. Databricks는 **"Spark를 설치하지 않고도 5분 만에 사용할 수 있는 플랫폼"**을 제공하여, 데이터 팀이 인프라가 아닌 데이터 자체에 집중할 수 있게 만들었습니다. 이것이 Hadoop 시절 클러스터 운영에 팀 리소스의 40%를 쓰던 것에서, 0%로 줄이는 혁명적 전환이었습니다.

직접 경험으로 비교하면, Hadoop 클러스터를 온프레미스에서 세팅하는 데 **2~4주**가 걸렸고, Spark on YARN 환경 구성에 **추가 1주**가 필요했습니다. Databricks에서는 워크스페이스 생성 후 **10분 이내**에 첫 번째 Spark 잡을 실행할 수 있었습니다.

---

## 4단계: 클라우드 데이터 웨어하우스 (2012~2018)

Hadoop과 Spark가 빅데이터 처리를 해결하는 동안, **SQL 분석**을 위한 클라우드 데이터 웨어하우스도 등장했습니다.

| 서비스 | 출시 연도 | 제공사 | 특징 |
|--------|----------|--------|------|
| **Amazon Redshift** | 2012 | AWS | 최초의 클라우드 데이터 웨어하우스 |
| **Google BigQuery** | 2012 | Google | 서버리스, 자동 확장 |
| **Snowflake** | 2014 | Snowflake Inc. | 스토리지/컴퓨트 분리, 멀티 클라우드 |
| **Azure Synapse** | 2019 | Microsoft | Spark + SQL 통합 |

### 2-Platform 문제

이 시기에 많은 기업들은 **데이터 레이크**(Hadoop/Spark)와 **데이터 웨어하우스**(Redshift/Snowflake)를 **동시에 운영**해야 했습니다. 이로 인한 데이터 복사, 비용 증가, 거버넌스 분산 문제가 심각해졌습니다.

### 실전 인사이트: 2-Platform 시대의 고통

이 시기를 직접 경험한 입장에서, 2-Platform 운영이 얼마나 비효율적이었는지 구체적으로 설명하겠습니다.

**전형적인 2-Platform 아키텍처**:

| 단계 | 흐름 |
|------|------|
| 원본 데이터 | → S3 (데이터 레이크) |
| ETL 경로 1 | Spark로 ETL → S3 (가공된 데이터) → COPY INTO → Redshift/Snowflake (DW) → BI 대시보드 |
| ETL 경로 2 | Spark MLlib로 ML 모델 학습 |

> **문제**: S3의 데이터와 DW의 데이터가 "다른 버전"이 되는 순간이 발생합니다

| 2-Platform의 고통 | 구체적 상황 |
|-------------------|-----------|
| **데이터 불일치** | S3에서 ETL을 다시 돌렸는데 DW에는 반영이 안 되어, 대시보드와 ML 모델이 서로 다른 데이터를 보고 있음 |
| **비용 2배** | 같은 데이터를 S3에도, Redshift에도 저장. 스토리지 비용이 사실상 2배 |
| **거버넌스 분산** | S3의 접근 권한은 IAM으로, Redshift의 접근 권한은 SQL GRANT로 별도 관리. 정합성 보장 불가 |
| **ETL 지옥** | 데이터 레이크 → DW로 데이터를 복사하는 ETL 파이프라인이 별도로 필요. 이 파이프라인이 실패하면 분석 지연 |
| **스키마 관리** | 데이터 레이크(Schema-on-Read)와 DW(Schema-on-Write)의 스키마가 달라서 매핑 로직 유지 필요 |

> 💡 이 고통을 한 문장으로 요약하면: **"같은 데이터를 두 곳에 두고, 두 곳을 동기화하는 데 팀의 30% 이상의 시간을 쓰고 있었다"**입니다. 레이크하우스는 바로 이 문제를 해결하기 위해 탄생했습니다.

---

## 5단계: 레이크하우스의 시대 (2019~현재)

### Delta Lake와 레이크하우스 (2019~2020)

Databricks는 2-Platform 문제를 해결하기 위해 **Delta Lake**를 오픈소스로 공개하고(2019), **레이크하우스 아키텍처** 논문을 발표했습니다(2020).

> "데이터 레이크의 유연성과 저비용 + 웨어하우스의 성능과 안정성 = 레이크하우스"

레이크하우스가 이전 시대의 문제를 어떻게 해결했는지 구체적으로 살펴보겠습니다.

| 이전 시대의 고통 | 레이크하우스의 해결책 |
|----------------|-------------------|
| HDFS NameNode 단일 장애점 | 클라우드 오브젝트 스토리지(S3/ADLS)는 99.999% 내구성. 인프라 운영 불필요 |
| MapReduce의 느린 처리 | Spark 엔진 + Photon(C++ 네이티브 엔진)으로 초고속 처리 |
| 2-Platform 데이터 복사 | Delta Lake 하나로 ETL, SQL 분석, ML을 모두 처리. 복사 불필요 |
| 거버넌스 분산 | Unity Catalog로 데이터, ML 모델, AI 에이전트까지 단일 거버넌스 |
| 클러스터 운영 부담 | Serverless 컴퓨트로 인프라 관리 완전 제거 |
| 스키마 관리 복잡성 | Delta Lake의 스키마 강제(Schema Enforcement)로 데이터 품질 보장 |

> 💡 레이크하우스의 핵심 혁신은 **"데이터를 한 곳에만 저장하고, 그 위에서 모든 워크로드(ETL, SQL, ML, AI)를 실행한다"**는 것입니다. 2-Platform 시대에 데이터를 복사하느라 허비했던 시간과 비용이 사라집니다.

### 현재의 모습 (2023~2026)

| 트렌드 | 설명 |
|--------|------|
| **GenAI 통합** | 대규모 언어 모델(LLM)을 데이터 플랫폼에 통합. Agent 개발 지원 |
| **서버리스 전환** | 인프라 관리 부담 제거. 사용한 만큼만 과금 |
| **오픈 포맷 수렴** | Delta Lake와 Iceberg의 상호 운용성 강화 (UniForm) |
| **통합 거버넌스** | Unity Catalog로 데이터, ML 모델, AI 에이전트까지 통합 관리 |
| **Lakebase** | 레이크하우스에 OLTP(PostgreSQL) 기능까지 통합 |

> 💡 **시대의 흐름을 한 문장으로**: "직접 운영하던 것을 관리형 서비스로, 분산되어 있던 것을 하나로 통합하는 것"이 빅데이터 기술 진화의 일관된 방향입니다. Hadoop 시절에는 HDFS, MapReduce, Hive, YARN을 각각 설치하고 관리했지만, 레이크하우스 시대에는 **하나의 플랫폼에서 모든 것이 통합되어 제공**됩니다. 이 교육 과정에서 배우실 Databricks의 모든 기능은, 바로 이 "통합과 단순화"의 철학 위에 서 있습니다.

Hadoop 시절에는 10명의 팀에 인프라 엔지니어 3명, Spark 개발자 4명, DW 관리자 2명, 거버넌스 담당자 1명이 필요했습니다. 레이크하우스 시대에는 같은 규모의 일을 **데이터 엔지니어 3~4명**으로 처리할 수 있습니다. 남는 인력은 데이터 품질, ML 모델링, AI 에이전트 개발 같은 **비즈니스 가치를 직접 만드는 일**에 투입할 수 있게 되었습니다.

| 시대 | 기간 | 핵심 기술 | 전환 계기 |
|------|------|----------|----------|
| **Hadoop 시대** | 2006~2012 | Hadoop (분산 저장/처리) | - |
| **Spark + Cloud DW 시대** | 2013~2018 | Spark + Cloud DW (2-Platform) | Hadoop의 처리 속도 한계 |
| **Lakehouse 시대** | 2019~현재 | Lakehouse (통합 플랫폼) | 2-Platform 간 데이터 복사 문제 |

---

## 실전 인사이트: 각 시대를 관통하는 교훈

20년간 Hadoop → Spark → Cloud DW → Lakehouse의 전환을 모두 겪으면서 얻은 교훈을 정리합니다.

| 교훈 | 설명 |
|------|------|
| **기술 부채는 복리로 쌓입니다** | Hadoop MapReduce로 작성한 ETL 코드를 Spark로 마이그레이션하는 데 6개월이 걸렸습니다. 새 기술이 나올 때마다 기존 코드를 전환하는 비용이 발생하므로, 처음부터 **오픈 표준(Open Format)**을 사용하는 것이 장기적으로 유리합니다 |
| **인프라 운영은 핵심 역량이 아닙니다** | Hadoop 클러스터 운영에 팀의 40% 시간을 쓰고 있었는데, 관리형 서비스(Databricks)로 전환한 후 그 시간을 데이터 품질과 비즈니스 로직에 투자할 수 있게 되었습니다 |
| **거버넌스를 나중에 하면 10배 어렵습니다** | Hadoop 시절 "일단 데이터부터 넣자"고 S3에 무제한으로 데이터를 적재했다가, 나중에 Unity Catalog로 거버넌스를 적용할 때 "이 데이터가 뭔지", "누가 만들었는지", "PII가 있는지"를 역추적하는 데 수개월이 걸렸습니다 |
| **데이터 엔지니어의 역할이 진화합니다** | Hadoop 시절에는 "인프라 운영자", Spark 시절에는 "ETL 개발자", 레이크하우스 시대에는 "데이터 플랫폼 아키텍트 + AI 파이프라인 설계자"로 역할이 계속 확장되고 있습니다 |

> 💡 **이 역사를 왜 알아야 하나요?** Databricks의 모든 기능(Delta Lake, Unity Catalog, Serverless, Lakehouse)은 앞선 시대의 **구체적인 고통**을 해결하기 위해 만들어졌습니다. 이 맥락을 이해하면, 각 기능이 왜 중요한지, 왜 이렇게 설계되었는지를 훨씬 깊이 이해하실 수 있습니다.

---

## 주요 인물과 조직

| 인물/조직 | 기여 |
|-----------|------|
| **Google** | GFS, MapReduce, BigTable, BigQuery — 분산 시스템의 기초를 놓았습니다 |
| **Doug Cutting** | Apache Hadoop을 창시했습니다. "Hadoop"이라는 이름은 아들의 코끼리 인형에서 따왔습니다 |
| **Matei Zaharia** | Apache Spark를 창시하고 Databricks를 공동 설립했습니다 |
| **Ali Ghodsi** | Databricks CEO. 레이크하우스 비전을 이끌고 있습니다 |
| **Jay Kreps** | Apache Kafka를 창시하고 Confluent를 설립했습니다 |
| **Benoit Dageville** | Snowflake를 공동 설립했습니다 |

> 💡 **흥미로운 사실**: Databricks의 창업 멤버 7명은 모두 UC Berkeley AMPLab 출신입니다. 학계에서 시작된 연구가 산업을 바꾼 대표적인 사례입니다. Spark의 핵심 아이디어인 **RDD(Resilient Distributed Dataset)** 는 Matei Zaharia의 2012년 박사학위 논문에서 발표되었으며, 이것이 오늘날 전 세계 수천 개 기업이 사용하는 데이터 처리 엔진의 기초가 되었습니다.

또한 이 역사에서 주목할 점은, **모든 혁신이 "이전 시스템의 구체적인 고통"에서 출발**했다는 것입니다. Google이 GFS를 만든 것은 기존 파일 시스템이 웹 크롤링 데이터를 감당하지 못했기 때문이고, Spark가 만들어진 것은 MapReduce의 디스크 I/O가 ML 워크로드에 치명적이었기 때문이고, Delta Lake가 만들어진 것은 데이터 레이크에 트랜잭션과 스키마 관리가 없어서 "데이터 늪(Data Swamp)"이 되어가고 있었기 때문입니다.

---

## 정리

| 시대 | 핵심 기술 | 해결한 문제 | 남은 한계 |
|------|-----------|-----------|-----------|
| **RDBMS** (1970~) | Oracle, MySQL | 구조화된 데이터의 안전한 관리 | 대용량 데이터 처리 불가 |
| **Hadoop** (2006~) | HDFS, MapReduce | 대용량 데이터의 분산 저장/처리 | 느린 속도, 복잡한 프로그래밍 |
| **Spark** (2014~) | Apache Spark | 빠른 인메모리 처리, 간결한 API | 레이크의 거버넌스/신뢰성 부재 |
| **Cloud DW** (2012~) | Redshift, Snowflake | 빠른 SQL 분석, 관리 편의성 | 2-Platform 문제, 데이터 복사 |
| **Lakehouse** (2020~) | Delta Lake, Unity Catalog | 통합 플랫폼, 오픈 포맷, 거버넌스 | 현재 진행형으로 발전 중 |

이 표를 보면 하나의 패턴이 보입니다. **각 시대의 "남은 한계"가 다음 시대의 "해결한 문제"가 됩니다.** RDBMS가 대용량을 처리하지 못해서 Hadoop이 나왔고, Hadoop이 느려서 Spark가 나왔고, 2-Platform이 복잡해서 Lakehouse가 나왔습니다. 이 교육 과정을 통해 여러분은 현재 Lakehouse가 해결하고 있는 문제들과, 그 위에서 새롭게 열리고 있는 가능성(GenAI, Agent, Lakebase)을 체계적으로 학습하게 됩니다.

다음 문서에서는 현재의 **빅데이터 생태계**에 어떤 솔루션들이 있고, 주요 벤더들의 위치를 살펴보겠습니다.

---

## 참고 링크

- [Databricks Blog: What is a Data Lakehouse?](https://www.databricks.com/blog/2020/01/30/what-is-a-data-lakehouse.html)
- [Original Lakehouse Paper (CIDR 2021)](https://www.cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf)
- [Apache Spark History](https://spark.apache.org/history.html)
- [Databricks: Release Notes](https://docs.databricks.com/aws/en/release-notes/)
