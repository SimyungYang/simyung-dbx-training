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

---

## 3단계: Apache Spark의 등장 (2009~2015)

### Spark의 탄생

UC Berkeley AMPLab의 **Matei Zaharia**가 Hadoop MapReduce의 한계를 극복하기 위해 개발한 것이 **Apache Spark**입니다.

| 비교 항목 | Hadoop MapReduce | Apache Spark |
|-----------|-----------------|--------------|
| 중간 결과 저장 | 디스크 | **메모리** |
| 처리 속도 | 느림 | 최대 **100배** 빠름 |
| 프로그래밍 | Java (복잡) | Python, SQL, Scala (간결) |
| 반복 처리 | 매우 비효율 | 효율적 (메모리 캐싱) |
| 스트리밍 | 별도 도구 필요 | 내장 (Structured Streaming) |
| ML | 별도 도구 (Mahout) | 내장 (MLlib) |

### Databricks의 설립 (2013)

Spark를 만든 연구팀이 **Databricks**를 설립하여, Spark를 기업이 쉽게 사용할 수 있는 관리형 플랫폼으로 제공하기 시작했습니다.

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

---

## 5단계: 레이크하우스의 시대 (2019~현재)

### Delta Lake와 레이크하우스 (2019~2020)

Databricks는 2-Platform 문제를 해결하기 위해 **Delta Lake**를 오픈소스로 공개하고(2019), **레이크하우스 아키텍처** 논문을 발표했습니다(2020).

> "데이터 레이크의 유연성과 저비용 + 웨어하우스의 성능과 안정성 = 레이크하우스"

### 현재의 모습 (2023~2025)

| 트렌드 | 설명 |
|--------|------|
| **GenAI 통합** | 대규모 언어 모델(LLM)을 데이터 플랫폼에 통합. Agent 개발 지원 |
| **서버리스 전환** | 인프라 관리 부담 제거. 사용한 만큼만 과금 |
| **오픈 포맷 수렴** | Delta Lake와 Iceberg의 상호 운용성 강화 (UniForm) |
| **통합 거버넌스** | Unity Catalog로 데이터, ML 모델, AI 에이전트까지 통합 관리 |
| **Lakebase** | 레이크하우스에 OLTP(PostgreSQL) 기능까지 통합 |

| 시대 | 기간 | 핵심 기술 | 전환 계기 |
|------|------|----------|----------|
| **Hadoop 시대** | 2006~2012 | Hadoop (분산 저장/처리) | - |
| **Spark + Cloud DW 시대** | 2013~2018 | Spark + Cloud DW (2-Platform) | Hadoop의 처리 속도 한계 |
| **Lakehouse 시대** | 2019~현재 | Lakehouse (통합 플랫폼) | 2-Platform 간 데이터 복사 문제 |

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

---

## 정리

| 시대 | 핵심 기술 | 해결한 문제 | 남은 한계 |
|------|-----------|-----------|-----------|
| **RDBMS** (1970~) | Oracle, MySQL | 구조화된 데이터의 안전한 관리 | 대용량 데이터 처리 불가 |
| **Hadoop** (2006~) | HDFS, MapReduce | 대용량 데이터의 분산 저장/처리 | 느린 속도, 복잡한 프로그래밍 |
| **Spark** (2014~) | Apache Spark | 빠른 인메모리 처리, 간결한 API | 레이크의 거버넌스/신뢰성 부재 |
| **Cloud DW** (2012~) | Redshift, Snowflake | 빠른 SQL 분석, 관리 편의성 | 2-Platform 문제, 데이터 복사 |
| **Lakehouse** (2020~) | Delta Lake, Unity Catalog | 통합 플랫폼, 오픈 포맷, 거버넌스 | 현재 진행형으로 발전 중 |

다음 문서에서는 현재의 **빅데이터 생태계**에 어떤 솔루션들이 있고, 주요 벤더들의 위치를 살펴보겠습니다.

---

## 참고 링크

- [Databricks Blog: What is a Data Lakehouse?](https://www.databricks.com/blog/2020/01/30/what-is-a-data-lakehouse.html)
- [Original Lakehouse Paper (CIDR 2021)](https://www.cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf)
- [Apache Spark History](https://spark.apache.org/history.html)
- [Databricks: Release Notes](https://docs.databricks.com/aws/en/release-notes/)
