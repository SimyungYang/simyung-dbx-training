# 레이크하우스란?

## 왜 레이크하우스가 등장했나요?

[이전 섹션](../01-data-fundamentals/data-warehouse-vs-data-lake.md)에서 데이터 웨어하우스와 데이터 레이크의 장단점을 살펴보았습니다. 이 두 가지 접근법에는 각각 분명한 한계가 있었습니다.

### 기존 방식의 문제점

```mermaid
graph TB
    subgraph Problem["❌ 기존의 2-Platform 아키텍처"]
        direction LR
        Source["📦 소스<br/>데이터"]

        subgraph Lake["데이터 레이크"]
            L1["원본 데이터 저장"]
            L2["ML 학습용"]
        end

        subgraph WH["데이터 웨어하우스"]
            W1["정제된 데이터"]
            W2["BI/SQL 분석용"]
        end

        Source --> Lake
        Lake -->|"데이터 복사<br/>(ETL)"| WH
    end
```

| 문제점 | 설명 |
|--------|------|
| **데이터 중복** | 동일한 데이터가 레이크와 웨어하우스에 이중으로 저장되어 스토리지 비용이 증가합니다 |
| **데이터 불일치** | 복사 과정에서 시간 차이가 발생하여 두 시스템의 데이터가 달라질 수 있습니다 |
| **복잡한 관리** | 두 개의 시스템을 각각 관리하고, 사이의 ETL 파이프라인도 관리해야 합니다 |
| **거버넌스 분산** | 레이크와 웨어하우스에서 각각 별도로 권한을 관리해야 합니다 |
| **높은 비용** | 스토리지 중복 + 컴퓨팅 중복 + 관리 인력 중복 |

### 레이크하우스의 해결 방법

> 💡 **레이크하우스(Lakehouse)**란 **데이터 레이크의 유연성과 저비용**에 **데이터 웨어하우스의 성능과 안정성**을 결합한 차세대 데이터 아키텍처입니다.

```mermaid
graph TB
    subgraph Solution["✅ 레이크하우스 아키텍처"]
        Source["📦 소스<br/>데이터"]

        subgraph LH["💎 레이크하우스"]
            L1["저렴한 클라우드 스토리지<br/>(S3, ADLS, GCS)"]
            L2["+ Delta Lake<br/>(ACID 트랜잭션)"]
            L3["+ 쿼리 엔진<br/>(빠른 SQL 분석)"]
            L4["+ 거버넌스<br/>(Unity Catalog)"]
        end

        C1["📊 BI/SQL 분석"]
        C2["🤖 ML/AI"]
        C3["📈 리포트"]

        Source --> LH
        LH --> C1
        LH --> C2
        LH --> C3
    end
```

쉽게 말하면, 레이크하우스는 **"데이터 레이크 위에 웨어하우스의 기능을 얹은 것"**입니다.

---

## 레이크하우스의 핵심 원리

레이크하우스가 "최고의 장점만 결합"할 수 있는 비결은 다음 세 가지 기술적 혁신에 있습니다.

### 1. 오픈 포맷 기반 저장

데이터를 Parquet 같은 **오픈 파일 포맷**으로 클라우드 오브젝트 스토리지에 저장합니다. 특정 벤더의 독점 포맷이 아니기 때문에 어떤 도구로도 읽을 수 있습니다.

> 💡 **Parquet(파케이)란?** Apache에서 개발한 오픈소스 **컬럼 기반 파일 포맷**입니다. 행 단위가 아니라 열 단위로 데이터를 저장하기 때문에, 분석 쿼리에서 필요한 컬럼만 효율적으로 읽을 수 있어 매우 빠르고 저장 공간도 적게 차지합니다. 현재 빅데이터 생태계에서 가장 널리 사용되는 파일 포맷 중 하나입니다.

### 2. 트랜잭션 레이어 (Delta Lake)

오픈 파일 포맷 위에 **트랜잭션 관리 레이어**를 추가합니다. 이것이 바로 **Delta Lake**의 역할입니다. 데이터 레이크에 없던 ACID 트랜잭션, 스키마 관리, 타임 트래블 등의 기능을 제공합니다. (다음 문서에서 자세히 다룹니다.)

### 3. 고성능 쿼리 엔진

데이터 레이크 위에서도 데이터 웨어하우스 수준의 SQL 쿼리 성능을 제공하는 최적화된 엔진을 사용합니다. Databricks에서는 **Photon** 엔진이 이 역할을 수행합니다.

> 💡 **Photon이란?** Databricks가 개발한 **차세대 쿼리 엔진**입니다. C++로 작성되어 기존 Spark SQL 엔진보다 최대 수 배 빠른 성능을 제공합니다. 특별한 설정 없이 Photon이 활성화된 클러스터를 사용하면 자동으로 적용됩니다.

---

## 레이크하우스 vs 기존 방식 비교

| 비교 항목 | 데이터 레이크 | 데이터 웨어하우스 | 레이크하우스 |
|-----------|-------------|-----------------|------------|
| **데이터 유형** | 모든 유형 | 정형만 | 모든 유형 ✅ |
| **ACID 트랜잭션** | ❌ | ✅ | ✅ |
| **SQL 성능** | 느림 | 빠름 | 빠름 ✅ |
| **ML/AI 지원** | ✅ | 제한적 | ✅ |
| **스키마 관리** | 없음 | 강력 | 강력 ✅ |
| **저장 비용** | 저렴 | 비쌈 | 저렴 ✅ |
| **거버넌스** | 약함 | 강력 | 강력 ✅ |
| **오픈 포맷** | ✅ | ❌ (보통 독점) | ✅ |

---

## Databricks 레이크하우스의 구성 요소

Databricks의 레이크하우스는 다음 요소들로 구성됩니다.

```mermaid
graph TB
    subgraph Lakehouse["Databricks Lakehouse"]
        direction TB
        UC["🛡️ Unity Catalog<br/>통합 거버넌스"]

        subgraph Storage["💾 스토리지 레이어"]
            CS["클라우드 오브젝트 스토리지<br/>(S3, ADLS, GCS)"]
            DL["Delta Lake<br/>(트랜잭션 + 스키마 관리)"]
            CS --> DL
        end

        subgraph Engine["⚡ 컴퓨팅 레이어"]
            SP["Apache Spark<br/>(분산 처리)"]
            PH["Photon Engine<br/>(고속 SQL)"]
        end

        subgraph Consume["📊 분석 레이어"]
            SQL["Databricks SQL"]
            ML["MLflow + AI"]
            BI["AI/BI Dashboard"]
        end

        UC -.-> Storage
        UC -.-> Engine
        UC -.-> Consume
        Storage --> Engine --> Consume
    end
```

| 레이어 | 구성 요소 | 역할 |
|--------|-----------|------|
| **스토리지** | 클라우드 스토리지 + Delta Lake | 데이터를 오픈 포맷으로 저렴하게 저장하면서도, 트랜잭션과 스키마를 관리합니다 |
| **컴퓨팅** | Spark + Photon | 대규모 데이터를 빠르게 처리하고, SQL 쿼리 성능을 극대화합니다 |
| **분석** | SQL, ML, BI | 하나의 데이터로 SQL 분석, ML 모델 학습, 대시보드를 모두 수행합니다 |
| **거버넌스** | Unity Catalog | 모든 레이어에 걸쳐 통합적인 접근 제어와 감사를 수행합니다 |

---

## 레이크하우스의 실제 장점 사례

### 사례 1: 데이터 복사 제거

**기존**: 데이터 레이크에 원본 저장 → ETL → 웨어하우스에 복사 → BI 도구에서 조회
**레이크하우스**: 데이터 레이크에 Delta Lake로 저장 → SQL로 바로 조회 (복사 불필요!)

### 사례 2: ML과 BI의 통합

**기존**: 분석가는 웨어하우스에서 SQL, 데이터 과학자는 레이크에서 Python → 서로 다른 데이터를 봄
**레이크하우스**: 동일한 Delta 테이블에서 SQL 분석과 ML 학습을 모두 수행 → 일관된 데이터

### 사례 3: 실시간 + 배치 통합

**기존**: 배치 파이프라인과 스트리밍 파이프라인을 별도 시스템에서 운영
**레이크하우스**: Spark Structured Streaming으로 실시간 데이터를 Delta 테이블에 적재 → 같은 테이블에서 배치 분석도 수행

---

## 업계 동향: 레이크하우스의 보편화

레이크하우스 아키텍처는 Databricks가 처음 제안한 개념이지만, 현재는 업계 전반에서 채택되고 있는 추세입니다.

| 벤더 | 레이크하우스 관련 제품 |
|------|---------------------|
| **Databricks** | Delta Lake 기반 Lakehouse Platform (선도) |
| **Snowflake** | Iceberg Tables, Unistore |
| **AWS** | S3 + Glue + Athena (레이크하우스 패턴) |
| **Microsoft** | OneLake (Microsoft Fabric) |
| **Google** | BigLake |

> 💡 **Apache Iceberg란?** Delta Lake와 유사한 목적을 가진 또 다른 오픈소스 테이블 포맷입니다. Netflix가 주도하여 개발했으며, 현재 많은 데이터 플랫폼에서 지원하고 있습니다. Databricks는 Delta Lake를 기본으로 사용하면서도, **UniForm** 기능을 통해 Iceberg 호환성을 제공하고 있어, 외부 엔진에서도 Delta 테이블을 Iceberg 테이블처럼 읽을 수 있습니다.

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **레이크하우스** | 데이터 레이크의 유연성 + 웨어하우스의 안정성을 결합한 아키텍처입니다 |
| **Delta Lake** | 레이크하우스를 가능하게 하는 핵심 기술. 오픈 파일 포맷 위에 트랜잭션을 추가합니다 |
| **Photon** | Databricks의 고성능 쿼리 엔진으로, 웨어하우스 수준의 SQL 성능을 제공합니다 |
| **오픈 포맷** | Parquet 등 독점 포맷이 아닌 오픈 포맷을 사용하여 벤더 종속을 방지합니다 |
| **단일 플랫폼** | 하나의 데이터로 BI, ML, 스트리밍 등 모든 워크로드를 지원합니다 |

다음 문서에서는 레이크하우스의 핵심 기술인 **Delta Lake**를 자세히 살펴보겠습니다.

---

## 참고 링크

- [Databricks: What is a Data Lakehouse?](https://docs.databricks.com/aws/en/lakehouse/)
- [Azure Databricks: What is a data lakehouse?](https://learn.microsoft.com/en-us/azure/databricks/lakehouse/)
- [Databricks Blog: What is a Lakehouse?](https://www.databricks.com/blog/2020/01/30/what-is-a-data-lakehouse.html)
- [Delta Lake Official Site](https://delta.io/)
