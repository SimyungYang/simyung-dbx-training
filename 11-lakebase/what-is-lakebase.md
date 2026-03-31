# Lakebase란?

## 개념

> 💡 **Lakebase**는 Databricks가 제공하는 **관리형 PostgreSQL 호환 OLTP 데이터베이스**입니다. 레이크하우스 플랫폼 안에서 OLTP(트랜잭션 처리)와 OLAP(분석)를 하나로 통합하여, 데이터 복사 없이 운영 데이터를 바로 분석에 활용할 수 있게 합니다.

> 💡 **OLTP (Online Transaction Processing)**: 주문 접수, 결제 처리, 재고 차감, 사용자 로그인 등 **건 단위의 빠른 읽기/쓰기**에 최적화된 시스템입니다. 웹 애플리케이션의 백엔드 DB로 사용됩니다. MySQL, PostgreSQL, Oracle이 대표적입니다.

---

## 왜 Lakebase가 필요한가요?

### 기존의 문제: OLTP-OLAP 분리

기존에는 OLTP(MySQL, PostgreSQL)와 OLAP(Databricks)를 **별도로 운영**하고, ETL로 데이터를 복사해야 했습니다. 이로 인해 여러 문제가 발생했습니다.

**기존 vs Lakebase 비교**

| 방식 | 흐름 | 특징 |
|------|------|------|
| **기존 (별도 운영)** | 웹 앱 → PostgreSQL (OLTP) → ETL 복사 (시간 지연) → Delta Lake (OLAP) → 대시보드 | 별도 데이터베이스 운영, ETL로 인한 지연 |
| **Lakebase (통합)** | 웹 앱 → Lakebase (OLTP) → 자동 실시간 동기화 (Data Sync) → Delta Lake (OLAP) → 대시보드 | 통합 관리, 실시간 동기화 |

| 기존 문제 | Lakebase의 해결 |
|-----------|---------------|
| ETL 파이프라인 구축·운영 비용 | Data Sync로 자동 동기화. ETL 불필요 |
| 데이터 지연 (시간~일 단위) | 거의 실시간 동기화 |
| OLTP DB 별도 관리 (패치, 백업, 장애 대응) | Databricks가 완전 관리 |
| 거버넌스 분리 (OLTP/OLAP 별도 권한) | Unity Catalog로 통합 거버넌스 |

---

## 핵심 특징

### PostgreSQL 호환성

Lakebase는 **PostgreSQL 프로토콜과 호환**됩니다. 기존 PostgreSQL 애플리케이션, 드라이버, ORM(SQLAlchemy, Django ORM 등)을 **코드 변경 없이** 그대로 사용할 수 있습니다.

```python
# 기존 PostgreSQL 코드가 그대로 동작합니다
import psycopg2

conn = psycopg2.connect(
    host="<lakebase-host>",
    port=5432,
    dbname="mydb",
    user="<user>",
    password="<token>"
)
```

### 오토스케일링

> 🆕 **Lakebase Autoscaling (GA)**: 워크로드에 따라 컴퓨팅 리소스를 **자동으로 확장/축소**합니다. 트래픽이 적은 시간에는 자동으로 축소되고, 급증 시 자동으로 확장됩니다. 최대 **8TB**까지 스토리지가 자동 확장됩니다.

### Instant Branching

> 💡 **브랜칭(Branching)** 이란 현재 데이터베이스의 **즉시 복제본**을 생성하는 기능입니다. Git의 브랜치처럼, 원본에 영향을 주지 않고 별도의 환경에서 작업할 수 있습니다.

| 활용 | 설명 |
|------|------|
| **개발/테스트** | 프로덕션 데이터의 복제본에서 안전하게 테스트합니다 |
| **CI/CD** | 배포 전 스키마 변경을 브랜치에서 검증합니다 |
| **데이터 분석** | 특정 시점의 스냅샷을 분석합니다 |

### 자동 백업 & 복구

| 기능 | 설명 |
|------|------|
| **자동 백업** | 지속적으로 자동 백업됩니다 |
| **Point-in-Time Recovery** | 특정 시점으로 데이터를 복원할 수 있습니다 |
| **보존 기간** | 최대 35일 |

### High Availability

> 🆕 **고가용성(HA)**: 가용 영역(Availability Zone) 간 **자동 장애 조치(Failover)** 를 지원합니다. 하나의 AZ에 장애가 발생해도 다른 AZ에서 자동으로 서비스가 계속됩니다.

---

## Delta Lake와의 자동 동기화 (Data Sync)

Lakebase의 가장 차별화된 기능은 **Data Sync**입니다.

> 💡 **Data Sync**는 Lakebase의 데이터를 **자동으로 Delta Lake 테이블에 동기화**하는 기능입니다. Lakebase에서 INSERT/UPDATE/DELETE된 데이터가 거의 실시간으로 Delta 테이블에 반영됩니다.

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **앱 (OLTP 쿼리)** | INSERT/UPDATE 수행 | Lakebase에 데이터를 쓰고 읽습니다 |
| **Lakebase (PostgreSQL)** | OLTP 데이터 저장 | PostgreSQL 호환 데이터베이스입니다 |
| **Delta Lake (분석용)** | 자동 Data Sync로 거의 실시간 동기화 | 분석용 데이터를 제공합니다 |
| **DBSQL 분석** | SQL 분석 | Delta Lake 데이터를 SQL로 분석합니다 |
| **MLflow 학습** | ML 모델 학습 | Delta Lake 데이터로 모델을 학습합니다 |
| **AI/BI 대시보드** | 시각화 | 대시보드로 데이터를 시각화합니다 |

| 장점 | 설명 |
|------|------|
| **ETL 불필요** | 별도의 ETL 파이프라인을 구축할 필요가 없습니다 |
| **실시간 반영** | OLTP 변경이 거의 실시간으로 분석 테이블에 반영됩니다 |
| **통합 거버넌스** | Unity Catalog에서 OLTP와 OLAP 데이터를 함께 관리합니다 |
| **리니지 추적** | Lakebase → Delta Lake의 데이터 흐름이 리니지로 추적됩니다 |

---

## 사용 시나리오

| 시나리오 | 설명 |
|----------|------|
| **웹 앱 백엔드** | Databricks Apps(Streamlit, FastAPI 등)의 데이터 저장소로 사용합니다 |
| **AI 에이전트 상태 저장** | 에이전트의 대화 이력, 사용자 설정 등을 저장합니다 |
| **실시간 대시보드** | OLTP 데이터가 Data Sync로 즉시 대시보드에 반영됩니다 |
| **ML 피처 저장소** | 실시간 피처를 Lakebase에 저장하고, 배치 분석은 Delta Lake에서 수행합니다 |
| **마이크로서비스 DB** | 각 서비스의 독립적인 DB로 사용합니다 |

---

## 기존 PostgreSQL 서비스와의 비교

| 비교 | Lakebase | Amazon RDS PostgreSQL | Azure Database for PostgreSQL |
|------|----------|----------------------|------------------------------|
| **PostgreSQL 호환** | ✅ | ✅ | ✅ |
| **관리형** | ✅ 완전 관리 | ✅ 완전 관리 | ✅ 완전 관리 |
| **오토스케일링** | ✅ 자동 | ❌ 수동 크기 변경 | ❌ 수동 크기 변경 |
| **Instant Branching** | ✅ | ❌ | ❌ |
| **Delta Lake 동기화** | ✅ 자동 | ❌ ETL 필요 | ❌ ETL 필요 |
| **Unity Catalog 통합** | ✅ | ❌ | ❌ |
| **Databricks 생태계** | ✅ 네이티브 | 별도 연동 필요 | 별도 연동 필요 |
| **HIPAA/규정 준수** | ✅ | ✅ | ✅ |

> 💡 **핵심 차별점**: Lakebase의 가장 큰 가치는 "PostgreSQL + Delta Lake 자동 동기화 + Unity Catalog 통합"입니다. 기존 관리형 PostgreSQL 서비스는 OLTP만 제공하지만, Lakebase는 **OLTP-OLAP 통합**을 추가 비용과 노력 없이 제공합니다.

---

## 심화: Principal SA 레벨 아키텍처 및 운영 가이드

### CAP 정리와 일관성 모델

분산 시스템에서 **CAP 정리**(Consistency, Availability, Partition Tolerance)는 세 가지를 동시에 만족할 수 없다는 원칙입니다. Lakebase의 일관성 모델을 정확히 이해해야 합니다.

> 💡 **CAP 정리**: 네트워크 파티션(장애)이 발생했을 때, 분산 시스템은 **일관성(Consistency)**과 **가용성(Availability)** 중 하나를 선택해야 합니다. 은행 시스템은 일관성을, SNS 피드는 가용성을 우선하는 것이 일반적입니다.

| 구성 요소 | 일관성 모델 | 상세 |
|----------|-----------|------|
| **Lakebase (OLTP)** | **Strong Consistency** | PostgreSQL의 MVCC 기반. 쓰기 직후 읽기에서 최신 값을 보장합니다. SERIALIZABLE 격리 수준까지 지원합니다 |
| **Data Sync (OLTP→OLAP)** | **Eventual Consistency** | CDC 기반 비동기 복제. 수 초~수십 초의 지연(lag)이 발생할 수 있습니다. 정확한 지연시간은 트랜잭션 볼륨에 비례합니다 |
| **Delta Lake (OLAP)** | **Snapshot Isolation** | Data Sync로 반영된 시점의 일관된 스냅샷을 제공합니다. 동일 쿼리 내에서는 일관성이 보장됩니다 |

```
[Lakebase OLTP]  ──CDC 스트림──▶  [Data Sync]  ──Delta commit──▶  [Delta Lake OLAP]
  Strong Consistency              수 초 지연 (Eventual)            Snapshot Isolation
  (즉시 읽기 일관성)               (비동기 복제)                    (쿼리 내 일관성)
```

> ⚠️ **실무 주의**: OLTP에서 방금 INSERT한 데이터가 OLAP 대시보드에 즉시 나타나지 않을 수 있습니다. 고객에게 "거의 실시간(near real-time)"이라는 표현을 사용하고, **SLA로 정확한 지연시간을 약속하지 마세요**. 일반적으로 정상 부하에서 5~30초, 고부하 시 수 분의 지연이 발생할 수 있습니다.

---

### 성능 특성

| 성능 지표 | 수치 | 비고 |
|----------|------|------|
| **읽기 지연시간** | 1~5ms (단순 PK 조회) | PostgreSQL과 동등한 수준. 인덱스 적용 시 |
| **쓰기 지연시간** | 2~10ms (단건 INSERT) | 네트워크 지연 포함. 배치 INSERT는 throughput 우선 |
| **동시 연결 수** | 최대 수백~수천 (인스턴스 크기에 따라) | PostgreSQL의 `max_connections`와 유사한 제한 |
| **TPS (Transactions/sec)** | 수천~수만 TPS | 단순 쿼리 기준. 복잡한 조인은 성능 저하 |
| **스토리지** | 최대 8TB 자동 확장 | 8TB 초과 시 수평 분할(sharding) 또는 아키텍처 재검토 필요 |

#### 커넥션 풀링 전략

PostgreSQL은 프로세스 기반이므로 동시 연결이 증가하면 메모리와 CPU 사용량이 급증합니다. 프로덕션 환경에서는 반드시 커넥션 풀링을 사용해야 합니다.

```python
# PgBouncer 또는 애플리케이션 레벨 풀링 권장
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://user:token@lakebase-host:5432/mydb",
    pool_size=20,          # 풀의 기본 연결 수
    max_overflow=10,       # 초과 허용 연결 수
    pool_timeout=30,       # 연결 대기 타임아웃 (초)
    pool_recycle=1800,     # 연결 재활용 주기 (30분)
    pool_pre_ping=True     # 사용 전 연결 유효성 검사
)
```

| 풀링 전략 | 권장 설정 | 이유 |
|----------|----------|------|
| **pool_size** | 앱 인스턴스당 10~30 | Lakebase 인스턴스의 `max_connections` / 앱 인스턴스 수 |
| **max_overflow** | pool_size의 50% | 트래픽 스파이크 대응 |
| **pool_recycle** | 1800초 (30분) | 장시간 유휴 연결의 TCP 타임아웃 방지 |
| **idle 연결 정리** | pool_pre_ping=True | 끊어진 연결을 자동 감지하여 재연결 |

---

### 비용 모델: 기존 서비스 대비 TCO 비교

Lakebase의 비용은 **Lakebase 자체 비용 + Data Sync 비용**으로 구성됩니다.

| 비용 항목 | Lakebase | AWS RDS PostgreSQL | Azure DB for PostgreSQL |
|----------|---------|-------------------|----------------------|
| **컴퓨트 (월, 중형 기준)** | DBU 기반 (Serverless 과금) | db.r6g.xlarge: ~$350/월 | GP_Gen5_4: ~$400/월 |
| **스토리지 (1TB/월)** | 포함 (자동 확장) | gp3: ~$80/TB/월 | ~$115/TB/월 |
| **Data Sync 추가 비용** | Delta Lake 동기화 DBU 소비 | ❌ (기능 없음) | ❌ (기능 없음) |
| **ETL 파이프라인 비용** | ❌ (불필요) | 별도 구축 필요 ($500~2000/월) | 별도 구축 필요 |
| **거버넌스 도구 비용** | ❌ (Unity Catalog 포함) | 별도 도구 필요 | 별도 도구 필요 |
| **운영 인력** | 최소 (완전 관리형) | DBA 0.5~1명 필요 | DBA 0.5~1명 필요 |

> 💡 **TCO 관점**: Lakebase 자체 비용만 보면 RDS와 유사하거나 약간 높을 수 있습니다. 그러나 **ETL 파이프라인 제거, 거버넌스 통합, 운영 부담 감소**를 포함하면 전체 TCO는 30~50% 절감됩니다. 특히 DBA 인건비(연 1억 원+)를 고려하면 경제성이 큽니다.

---

### 스키마 진화(Schema Evolution) 복잡성

Lakebase에서 OLTP 스키마를 변경하면 Data Sync를 통해 Delta Lake에도 반영되어야 합니다. 이 과정에서 주의해야 할 사항이 있습니다.

| 스키마 변경 유형 | OLTP 영향 | OLAP(Delta Lake) 반영 | 주의사항 |
|---------------|----------|---------------------|---------|
| **ADD COLUMN** | 즉시 반영 | Data Sync가 자동 반영 (Schema Evolution 지원) | 하위 호환. 안전합니다 |
| **DROP COLUMN** | 즉시 반영 | Delta Lake에서 해당 컬럼이 NULL로 유지될 수 있음 | 하위 소비자 쿼리 확인 필요 |
| **ALTER COLUMN TYPE** | 즉시 반영 | 타입 호환성에 따라 Data Sync 실패 가능 | INT→BIGINT는 안전, VARCHAR→INT는 위험 |
| **RENAME COLUMN** | 즉시 반영 | Delta Lake에서 새 컬럼으로 인식될 수 있음 | 이름 변경 대신 새 컬럼 추가 + 기존 컬럼 deprecated 권장 |
| **RENAME TABLE** | 즉시 반영 | Data Sync 재설정 필요할 수 있음 | 사전에 Data Sync 비활성화 → 변경 → 재활성화 |

> ⚠️ **스키마 변경 프로토콜**: 프로덕션에서 스키마를 변경할 때는 (1) Data Sync 상태를 확인하고, (2) 하위 호환되는 변경만 수행하며, (3) OLAP 소비자(대시보드, ML 파이프라인)에 미리 공지하는 절차를 따르세요.

---

### 프로덕션 운영 패턴

#### 패턴 1: 멀티 테넌트 격리

SaaS 애플리케이션에서 테넌트별 데이터를 격리하는 전략입니다.

| 격리 전략 | 구현 | 장점 | 단점 |
|----------|------|------|------|
| **DB 레벨 격리** | 테넌트별 Lakebase 인스턴스 | 완전 격리, 독립 스케일링 | 비용 높음, 관리 복잡 |
| **스키마 레벨 격리** | 하나의 Lakebase 내 테넌트별 스키마 | 중간 격리, 비용 효율 | 스키마 수 증가 시 관리 부담 |
| **Row 레벨 격리** | `tenant_id` 컬럼 + RLS(Row Level Security) | 비용 최소, 단순 | 쿼리에 항상 tenant_id 필터 필요, 실수 위험 |

> 💡 **권장**: 테넌트 수가 100개 미만이고 규제 요건이 높으면 **스키마 레벨 격리**, 테넌트 수가 많고 데이터 규모가 작으면 **Row 레벨 격리**를 선택합니다.

#### 패턴 2: 읽기 복제본 활용

Lakebase의 Data Sync를 읽기 복제본처럼 활용할 수 있습니다.

```
[Lakebase OLTP]
  ├─ 앱 읽기/쓰기 (PostgreSQL 프로토콜)
  ├─ Data Sync → [Delta Lake] → DBSQL 분석 쿼리 (OLAP 읽기)
  └─ Data Sync → [Delta Lake] → ML 학습 (대량 읽기)
```

| 읽기 패턴 | 경로 | 적합한 용도 |
|----------|------|-----------|
| **실시간 단건 조회** | Lakebase 직접 읽기 | 앱 API, 사용자 프로필 조회 |
| **대량 분석 쿼리** | Delta Lake (via DBSQL) | 대시보드, 집계 리포트 |
| **ML 학습 데이터** | Delta Lake (via Spark) | 배치 학습, Feature 생성 |

> 💡 **핵심 이점**: 기존에는 OLTP DB에 분석 쿼리를 날리면 운영 DB가 느려지는 문제가 있었습니다. Lakebase + Data Sync 구조에서는 **분석 쿼리가 OLTP에 전혀 영향을 주지 않습니다**.

#### 패턴 3: 장애 복구 (RTO/RPO)

| 장애 시나리오 | RPO (데이터 손실) | RTO (복구 시간) | 복구 방법 |
|-------------|-----------------|----------------|----------|
| **단일 노드 장애** | 0 (손실 없음) | 수 초~수 분 | HA 자동 Failover |
| **AZ 장애** | 0 (동기 복제 시) | 수 분 | Multi-AZ Failover |
| **리전 장애** | 마지막 백업 시점까지 | 수 시간 | Point-in-Time Recovery (최대 35일) |
| **논리적 오류 (잘못된 DELETE)** | 복구 시점까지 | 수 분~수 시간 | PITR로 특정 시점 복원 |
| **전체 재해** | 마지막 백업 | 수 시간 | 백업에서 새 인스턴스 복원 |

```sql
-- Instant Branching을 활용한 빠른 복구 검증
-- 프로덕션 데이터의 브랜치를 생성하여 복구 절차를 사전 검증합니다
-- 브랜치 생성 → 복구 테스트 → 검증 → 브랜치 삭제
```

> ⚠️ **운영 권장사항**: (1) 정기적으로 PITR 복구를 테스트하세요 (분기 1회 권장). (2) Data Sync lag을 모니터링하여 비정상적 지연을 조기에 감지하세요. (3) Instant Branching을 활용하여 스키마 변경을 사전 검증하세요.

---

## 정리

| 핵심 기능 | 설명 |
|-----------|------|
| **PostgreSQL 호환** | 기존 앱/드라이버를 코드 변경 없이 사용할 수 있습니다 |
| **오토스케일링** | 워크로드에 따라 자동 확장/축소됩니다 (최대 8TB) |
| **Data Sync** | OLTP 데이터가 자동으로 Delta Lake에 동기화됩니다 |
| **Instant Branching** | 프로덕션 데이터의 즉시 복제본을 생성합니다 |
| **High Availability** | AZ 간 자동 장애 조치를 지원합니다 |
| **Unity Catalog 통합** | 통합 거버넌스 하에서 OLTP/OLAP 데이터를 관리합니다 |

---

## 참고 링크

- [Databricks: Lakebase](https://docs.databricks.com/aws/en/lakebase/)
- [Azure Databricks: Lakebase](https://learn.microsoft.com/en-us/azure/databricks/lakebase/)
- [Databricks Blog: Lakebase](https://www.databricks.com/blog)
- [Databricks: Release Notes — Lakebase](https://docs.databricks.com/aws/en/release-notes/)
