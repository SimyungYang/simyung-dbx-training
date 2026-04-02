# Lakeflow Connect란?

## 왜 Lakeflow Connect가 필요한가요?

기업의 데이터는 운영 데이터베이스(MySQL, PostgreSQL, Oracle)와 SaaS 애플리케이션(Salesforce, SAP, HubSpot)에 흩어져 있습니다. 이 데이터를 분석용 레이크하우스로 가져오려면 전통적으로 다음과 같은 과정이 필요했습니다:

1. 소스 시스템에 연결하는 코드 작성
2. 초기 전체 데이터 복사(Full Load)
3. 이후 변경된 데이터만 추적하는 CDC 로직 구현
4. 스키마 변경 감지 및 대응
5. 오류 처리, 재시도, 모니터링 구현

이 과정은 수주~수개월이 걸리며, 지속적인 유지보수도 필요합니다. **Lakeflow Connect는 이 모든 과정을 자동화하여 코드 없이 몇 번의 클릭만으로 데이터 수집 파이프라인을 구성** 할 수 있게 해줍니다.

> 💡 **Lakeflow Connect** 는 외부 데이터 소스(데이터베이스, SaaS 애플리케이션)에서 Databricks 레이크하우스로 데이터를 **자동으로 수집하는 관리형 커넥터** 서비스입니다. Fivetran, Airbyte 같은 데이터 통합(ELT) 도구와 유사한 역할을 하지만, Databricks 플랫폼에 네이티브로 통합되어 있습니다.

> 💡 **CDC(Change Data Capture)** 란 소스 데이터베이스에서 발생하는 INSERT, UPDATE, DELETE 등의 변경 사항을 실시간으로 캡처하여 대상 시스템에 반영하는 기술입니다. 전체 데이터를 매번 다시 복사하는 것보다 훨씬 효율적이며, 소스 시스템에 대한 부하도 최소화합니다.

---

## 지원 소스 시스템 상세

Lakeflow Connect는 지속적으로 새로운 커넥터를 추가하고 있습니다. 아래는 주요 지원 소스 목록입니다.

### 데이터베이스 커넥터

| 소스 | 지원 상태 | CDC 방식 | 비고 |
|------|:--------:|---------|------|
| **MySQL** | GA | Binlog 기반 | MySQL 5.7+, MariaDB 지원 |
| **PostgreSQL** | GA | Logical Replication | PostgreSQL 10+ |
| **SQL Server** | GA | CT(Change Tracking) / CDC | SQL Server 2016+ |
| **Oracle** | Public Preview | LogMiner | Oracle 12c+ |
| **MongoDB** | Public Preview | Change Streams | MongoDB 4.0+ |

### SaaS 커넥터

| 소스 | 지원 상태 | 수집 방식 | 비고 |
|------|:--------:|---------|------|
| **Salesforce** | GA | API 폴링 (Bulk API) | Sales Cloud, Service Cloud |
| **ServiceNow** | GA | Table API | ITSM 데이터 수집 |
| **Google Analytics** | Public Preview | Reporting API | GA4 지원 |
| **Workday HCM** | Beta | RAAS API | 인사 데이터 |
| **HubSpot** | Beta | REST API | 마케팅/영업 데이터 |
| **TikTok Ads** | Beta | Reporting API | 광고 성과 데이터 |
| **Google Ads** | Beta | Google Ads API | 광고 캠페인 데이터 |
| **Zendesk** | Beta | REST API | 고객 지원 데이터 |
| **NetSuite** | Beta | SuiteQL | ERP 데이터 |
| **Dynamics 365** | Beta | OData API | CRM/ERP 데이터 |

### 파일/기타 커넥터

| 소스 | 지원 상태 | 수집 방식 | 비고 |
|------|:--------:|---------|------|
| **SFTP** | Public Preview | 파일 폴링 | CSV, JSON 등 파일 수집 |
| **Google Sheets** | GA | Sheets API | 스프레드시트 동기화 |
| **SharePoint** | Beta | Graph API | 문서/리스트 데이터 |

> 🆕 **최신 동향**: Databricks는 매 릴리즈마다 새로운 커넥터를 추가하고 있습니다. 최신 지원 소스 목록은 [공식 문서](https://docs.databricks.com/aws/en/lakeflow-connect/)에서 확인하실 수 있습니다.

---

## 아키텍처: 전체 동작 흐름

Lakeflow Connect의 전체 아키텍처는 다음과 같이 구성됩니다.

| 계층 | 구성 요소 | 역할 |
|------|-----------|------|
| **소스 시스템** | 운영 DB (MySQL, PostgreSQL) | 원본 데이터 소스 |
|  | SaaS 앱 (Salesforce, HubSpot) | 원본 데이터 소스 |
| **Lakeflow Connect** | Unity Catalog Connection | 자격증명을 관리합니다 |
|  | Ingestion Gateway | 연결 및 변환을 수행합니다 |
|  | Ingestion Pipeline | 실행 엔진입니다 |
| **Databricks 레이크하우스** | Bronze Delta Table | 원시 데이터를 저장합니다 |
|  | Unity Catalog | 메타데이터와 리니지를 관리합니다 |

### 주요 구성 요소

| 구성 요소 | 역할 | 설명 |
|----------|------|------|
| **Unity Catalog Connection** | 자격증명 관리 | 소스 시스템 접속 정보를 안전하게 저장하고, Secret으로 비밀번호를 관리합니다 |
| **Ingestion Gateway** | 연결 및 변환 | 소스에서 데이터를 읽어 Delta 포맷으로 변환합니다. 네트워크 연결 및 인증을 처리합니다 |
| **Ingestion Pipeline** | 실행 엔진 | 실제 데이터 수집을 실행하는 서버리스 컴퓨트 엔진입니다. 스케줄링과 상태 관리를 담당합니다 |

---

## 초기 스냅샷 vs CDC 증분 동기화

Lakeflow Connect의 수집은 크게 두 단계로 이루어집니다.

### 1단계: 초기 스냅샷 (Initial Snapshot)

처음 파이프라인을 시작하면, 소스 테이블의 **전체 데이터를 한 번 복사** 합니다.

**Snapshot (전체 복사)**: 소스 테이블 (MySQL) →→→ 대상 테이블 (Delta)

| id | name | city |
|----|------|------|
| 1 | Kim | Seoul |
| 2 | Lee | Busan |
| 3 | Park | Daegu |

### 2단계: CDC 증분 수집 (Incremental CDC)

초기 스냅샷 이후에는 소스에서 **변경된 데이터(INSERT, UPDATE, DELETE)만 실시간으로 반영** 합니다.

```
소스에서 변경 발생:
  - INSERT: id=4, Park, Incheon (신규)
  - UPDATE: id=2, Lee, Seoul (도시 변경)
  - DELETE: id=3 (삭제)

CDC 스트림으로 변경분만 전달:

| op | id | name | city | ts |
|----|----|------|------|----|
| INSERT | 4 | Park | Incheon | ... |
| UPDATE | 2 | Lee | Seoul | ... |
| DELETE | 3 | | | ... |

대상 테이블에 반영:

| id | name | city | 상태 |
|----|------|------|------|
| 1 | Kim | Seoul | 변경 없음 |
| 2 | Lee | Seoul | UPDATE 반영 |
| 4 | Park | Incheon | INSERT 반영 |

> id=3은 DELETE 반영으로 제거됨
```

> 💡 **CDC의 장점**: 전체 데이터를 매번 다시 복사하는 Full Load 방식 대비, CDC는 변경분만 전달하므로 **소스 시스템의 부하가 최소화** 되고, **네트워크 트래픽이 크게 줄어들며**, **수집 지연시간이 초 단위로 단축** 됩니다.

---

## 스키마 진화(Schema Evolution) 처리

운영 데이터베이스에서는 애플리케이션 업데이트에 따라 스키마가 변경될 수 있습니다. Lakeflow Connect는 이를 자동으로 감지하고 처리합니다.

| 스키마 변경 유형 | Lakeflow Connect 동작 | 대상 테이블 영향 |
|----------------|---------------------|----------------|
| **컬럼 추가**(ADD COLUMN) | 자동으로 대상 테이블에 컬럼 추가 | 기존 행의 새 컬럼은 NULL |
| **컬럼 타입 변경** | 호환 가능한 변경만 자동 처리 | 예: INT → BIGINT 자동 변환 |
| **컬럼 삭제**(DROP COLUMN) | 대상 테이블에서 해당 컬럼 유지 (삭제하지 않음) | 이후 행에서 NULL로 채워짐 |
| **테이블 추가** | 설정에 따라 자동 수집 시작 | 새 Delta 테이블 생성 |

> ⚠️ **비호환 스키마 변경**: 컬럼 타입을 호환되지 않는 형태로 변경하면(예: VARCHAR → INT) 파이프라인이 일시 중지될 수 있습니다. 이 경우 수동으로 대응이 필요합니다.

---

## 에러 처리 및 재시도 메커니즘

Lakeflow Connect는 다양한 장애 상황에 대해 자동으로 대응합니다.

| 장애 유형 | 자동 대응 | 수동 대응 필요 |
|----------|---------|--------------|
| **네트워크 일시 단절** | 자동 재시도 (지수 백오프) | 불필요 |
| **소스 DB 일시 중단** | 재연결 대기 후 자동 재개 | 장시간 중단 시 확인 필요 |
| **자격증명 만료** | 파이프라인 일시 중지 | 자격증명 갱신 후 재시작 |
| **스키마 비호환 변경** | 파이프라인 일시 중지 | 스키마 매핑 수정 후 재시작 |
| **디스크/메모리 부족** | 서버리스 자동 스케일링 | 데이터 볼륨이 극단적인 경우 설정 조정 |
| **중복 데이터** | 체크포인트 기반 중복 방지 | 불필요 (Exactly-once 보장) |

---

## 성능 특성 및 제한사항

### 성능 특성

| 항목 | 수치/특성 |
|------|----------|
| **초기 스냅샷 속도** | 소스 DB 성능과 네트워크에 의존, 일반적으로 수백 GB/시간 |
| **CDC 지연시간** | 일반적으로 수초~수분 |
| **동시 테이블 수** | 파이프라인당 수백 개 테이블 지원 |
| **컴퓨트 타입** | 서버리스 (자동 스케일링) |
| **실행 모드** | 연속(Continuous) 또는 트리거(Triggered) |

### 주요 제한사항

| 제한사항 | 설명 | 대안 |
|---------|------|------|
| 지원되지 않는 소스 | 커넥터 목록에 없는 소스는 사용 불가 | Lakeflow Jobs + 커스텀 코드 |
| 복잡한 변환 | 수집 시 복잡한 변환 로직 불가 | SDP 파이프라인에서 후처리 |
| 소스 DB 버전 요구사항 | 각 DB별 최소 버전 요구 | 소스 DB 업그레이드 |
| 네트워크 연결 | 소스와 Databricks 간 네트워크 연결 필요 | Private Link / VPN 설정 |
| 커스텀 CDC 쿼리 | 특정 행/컬럼 필터링 제한적 | SDP에서 Silver 변환 시 필터 |

---

## 실습 예제: MySQL CDC 파이프라인 설정

### 사전 준비

1. **소스 MySQL 설정**: Binlog가 활성화되어 있어야 합니다
2. **네트워크**: Databricks에서 MySQL에 접속 가능해야 합니다
3. **계정**: `SELECT`, `REPLICATION SLAVE`, `REPLICATION CLIENT` 권한 필요

### Step 1: Unity Catalog Connection 생성

```sql
-- MySQL Connection 생성
CREATE CONNECTION mysql_erp
TYPE MYSQL
OPTIONS (
    host = 'mysql-prod.example.com',
    port = '3306',
    user = 'databricks_cdc',
    password = secret('my-scope', 'mysql-erp-password')
);

-- 연결 테스트 (권한 확인)
-- Catalog Explorer > External Data > Connections에서 "Test Connection" 클릭
```

### Step 2: Ingestion Pipeline 생성 (UI)

1. **Pipelines** 메뉴 → **Create Pipeline** 클릭
2. **Ingestion (Lakeflow Connect)** 선택
3. 소스 유형: **MySQL** 선택
4. Connection: `mysql_erp` 선택
5. 수집할 스키마/테이블 선택
6. 대상 카탈로그/스키마 지정 (예: `analytics.bronze`)
7. 실행 모드: **Continuous**(실시간) 또는 **Triggered**(주기적)
8. **Create** 클릭

### Step 3: 수집 결과 확인

```sql
-- 수집된 데이터 확인
SELECT * FROM analytics.bronze.customers LIMIT 10;

-- 수집 메타데이터 확인 (CDC 타임스탬프)
SELECT _ingested_at, _change_type, *
FROM analytics.bronze.orders
ORDER BY _ingested_at DESC
LIMIT 20;

-- 소스와 대상 건수 비교
SELECT 'source' AS origin, COUNT(*) AS cnt
FROM mysql_connection.ecommerce.customers
UNION ALL
SELECT 'target', COUNT(*)
FROM analytics.bronze.customers;
```

---

## 현업 사례: Fivetran에서 Lakeflow Connect로 전환한 기업의 비용 비교

> 🔥 **실전 경험담**
>
> 한 중견 이커머스 기업에서 Fivetran으로 MySQL(주문, 고객, 상품 등 15개 테이블)과 Salesforce(10개 오브젝트)를 수집하고 있었습니다. Fivetran의 월 비용은 행(Row) 기반 과금으로 **월 약 $3,500** 이었습니다.
>
> Lakeflow Connect GA 이후, MySQL 커넥터를 Lakeflow Connect로 전환한 결과는 다음과 같았습니다:
>
> | 항목 | Fivetran | Lakeflow Connect | 절감 |
> |------|:--------:|:----------------:|:----:|
> | **MySQL (15 테이블)** | $2,100/월 | $800/월 (DBU) | **62% 절감** |
> | **Salesforce (10 오브젝트)** | $1,400/월 | $1,400/월 (Fivetran 유지) | 0% |
> | **합계** | $3,500/월 | $2,200/월 | **37% 절감** |
>
> Salesforce는 Lakeflow Connect로 아직 전환하지 않았습니다 (당시 GA가 아니었기 때문). **핵심 절감 요인은 Fivetran의 행 기반 과금 vs Lakeflow Connect의 DBU 과금 차이** 였습니다. CDC로 변경분만 수집하면 실제 처리량이 적어 DBU 비용이 낮게 유지됩니다.

> 💡 **현업 팁**: 비용 절감 외에도, Lakeflow Connect 전환의 숨은 장점은 **관리 포인트 감소** 입니다. Fivetran은 별도 SaaS 콘솔에서 관리해야 하지만, Lakeflow Connect는 Databricks 내에서 모든 것을 관리합니다. Unity Catalog 리니지도 자동으로 추적되어 "이 데이터가 어디에서 온 것인지" 한눈에 파악할 수 있습니다.

---

## CDC 지연시간의 현실

공식 문서에서는 "수초~수분"이라고 안내하지만, 현업에서 체감하는 지연시간은 여러 요인에 따라 달라집니다.

| 영향 요인 | 최적 조건 | 현실적 조건 | 최악의 경우 |
|----------|:--------:|:---------:|:---------:|
| **소스 DB 위치** | 같은 리전 | 다른 리전 | 다른 대륙 |
| **네트워크** | Private Link | VPN | 퍼블릭 인터넷 |
| **트랜잭션 빈도** | 초당 10건 | 초당 1,000건 | 초당 10,000건+ |
| **행 크기** | 1KB 미만 | 1~10KB | 100KB+ (LOB 포함) |
| **체감 지연** | **5~15초** | **30초~2분** | **5~10분** |

> 🔥 **이것을 안 하면**: "CDC니까 실시간이겠지"라고 기대하면 안 됩니다. 한 고객사에서 "주문 후 1초 이내에 대시보드에 반영되어야 한다"는 요구사항을 가지고 Lakeflow Connect를 도입했다가, 실제 지연이 30초~1분이라는 것을 알고 실망한 적이 있습니다. **CDC의 지연시간은 "분 단위 준실시간"이지, "초 단위 실시간"이 아닙니다.** 초 단위 실시간이 필요하면 Kafka + Structured Streaming 조합을 고려해야 합니다.

### 지연시간을 최소화하는 팁

1. **소스 DB와 같은 리전에 Databricks 배포**: 네트워크 지연 최소화
2. **Private Link 사용**: 퍼블릭 인터넷 대비 지연 50% 이상 감소
3. **대형 LOB 컬럼 제외**: `TEXT`, `BLOB` 같은 대용량 컬럼은 별도 수집
4. **Continuous 모드 사용**: Triggered 모드는 폴링 간격만큼 추가 지연 발생

---

## 커넥터가 없는 소스 대응 전략

Lakeflow Connect의 커넥터 수는 아직 Fivetran(300+)이나 Airbyte(350+)에 비해 적습니다. 커넥터가 없는 소스를 처리해야 할 때의 현실적인 대응 전략입니다.

| 대안 | 적합한 경우 | 구현 난이도 | 비용 |
|------|-----------|:---------:|:----:|
| **Fivetran/Airbyte 병행** | 커넥터가 없는 소스가 5개 이상 | 낮음 | 높음 (별도 SaaS 비용) |
| **Lakeflow Jobs + JDBC** | RDBMS 소스, 배치 OK | 중간 | 낮음 (DBU만) |
| **Lakeflow Jobs + REST API** | SaaS 소스, API 제공 | 중간~높음 | 낮음 |
| **Auto Loader + 파일** | 소스가 파일을 제공(SFTP, S3) | 낮음 | 매우 낮음 |
| **Kafka + Structured Streaming** | 초 단위 실시간 필요 | 높음 | 중간 |

### 커스텀 JDBC 수집 예시

```python
# Lakeflow Jobs에서 JDBC로 직접 수집하는 패턴
# Lakeflow Connect에 없는 DB2 같은 소스에 활용

def incremental_jdbc_load(spark, source_table, target_table, watermark_col):
    """증분 수집 패턴: 마지막 수집 시점 이후 변경분만 가져오기"""

    # 마지막 수집 시점 확인
    last_watermark = spark.sql(f"""
        SELECT COALESCE(MAX({watermark_col}), '1970-01-01')
        FROM {target_table}
    """).collect()[0][0]

    # 변경분만 수집
    jdbc_url = dbutils.secrets.get("prod-secrets", "db2-jdbc-url")
    new_data = (spark.read.format("jdbc")
        .option("url", jdbc_url)
        .option("user", dbutils.secrets.get("prod-secrets", "db2-user"))
        .option("password", dbutils.secrets.get("prod-secrets", "db2-pass"))
        .option("query", f"""
            SELECT * FROM {source_table}
            WHERE {watermark_col} > '{last_watermark}'
        """)
        .load())

    # Delta 테이블에 MERGE
    new_data.createOrReplaceTempView("new_data")
    spark.sql(f"""
        MERGE INTO {target_table} AS t
        USING new_data AS s
        ON t.id = s.id
        WHEN MATCHED THEN UPDATE SET *
        WHEN NOT MATCHED THEN INSERT *
    """)
```

> 💡 **현업 팁**: 커넥터가 없는 소스가 1~2개라면 커스텀 JDBC로 충분합니다. 하지만 5개 이상이면 **Fivetran을 병행하는 것이 총 비용(개발 인건비 포함)에서 유리** 합니다. 커스텀 코드는 유지보수 비용이 숨어 있기 때문입니다 (스키마 변경 대응, 에러 처리, 모니터링 등).

---

## 다른 수집 도구와의 비교

Lakeflow Connect와 유사한 역할을 하는 외부 도구들과의 비교입니다.

| 비교 항목 | Lakeflow Connect | Fivetran | Airbyte | 커스텀 코드 (JDBC) |
|----------|:----------------:|:--------:|:-------:|:-----------------:|
| **설정 난이도** | 매우 쉬움 (No-code) | 쉬움 | 중간 | 어려움 |
| **Databricks 통합** | 네이티브 | 커넥터 필요 | 커넥터 필요 | 직접 구현 |
| **Unity Catalog 연동** | 자동 | 설정 필요 | 설정 필요 | 수동 |
| **데이터 리니지** | 자동 추적 | 제한적 | 제한적 | 없음 |
| **CDC 지원** | 내장 | 내장 | 내장 | 직접 구현 |
| **스키마 진화** | 자동 | 자동 | 자동 | 직접 구현 |
| **비용 모델** | DBU 과금 | 행 기반 과금 | 행 기반/오픈소스 | 컴퓨트 비용만 |
| **커넥터 수** | 증가 중 (20+) | 300+ | 350+ | 무제한 (직접 구현) |
| **서버리스** | 기본값 | SaaS | 자체 호스팅/Cloud | 클러스터 필요 |

> 💡 **선택 기준**: Lakeflow Connect가 지원하는 소스라면 **네이티브 통합의 이점**(Unity Catalog 자동 연동, 리니지, 서버리스)이 크므로 Lakeflow Connect를 우선 사용합니다. 지원되지 않는 소스가 많다면 Fivetran이나 Airbyte를 보조적으로 활용할 수 있습니다.

---

## 정리

- **Lakeflow Connect** 는 외부 DB/SaaS에서 코드 없이 관리형으로 데이터를 수집하는 서비스입니다
- **초기 스냅샷 + CDC 증분 수집** 을 자동으로 처리하여 소스 시스템 부하를 최소화합니다
- **스키마 진화를 자동으로 감지** 하고 대상 테이블에 반영합니다
- Unity Catalog와 네이티브로 통합되어 **자격증명 관리, 메타데이터, 리니지** 가 자동화됩니다
- 지원 커넥터가 없는 소스는 **Lakeflow Jobs + 커스텀 코드** 로 대응합니다

---

## 참고 링크

- [Databricks: Lakeflow Connect](https://docs.databricks.com/aws/en/lakeflow-connect/)
- [Databricks: Create a Connection](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-connection.html)
- [Databricks: Lakeflow Connect Supported Sources](https://docs.databricks.com/aws/en/lakeflow-connect/)
- [Databricks Blog: Introducing Lakeflow Connect](https://www.databricks.com/blog/introducing-lakeflow-connect)
- [Databricks: CDC with Lakeflow Connect](https://docs.databricks.com/aws/en/lakeflow-connect/)
