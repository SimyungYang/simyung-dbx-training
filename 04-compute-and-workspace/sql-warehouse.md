# SQL Warehouse

## SQL Warehouse란?

> 💡 **SQL Warehouse**는 SQL 쿼리를 실행하기 위한 **전용 컴퓨팅 리소스**입니다. 일반 Spark 클러스터와 달리 SQL 분석에 최적화되어 있으며, BI 도구 연결, 대시보드 조회 등에 적합합니다.

SQL Warehouse는 Databricks에서 **데이터 분석과 BI 워크로드를 위해 설계된 전용 SQL 엔진**입니다. 내부적으로 Apache Spark 기반이지만, SQL 실행에 특화된 최적화가 적용되어 있으며, **Photon**엔진이 기본 포함되어 빠른 쿼리 성능을 제공합니다.

---

## 왜 SQL Warehouse가 필요한가?

일반 All-Purpose Cluster로도 SQL을 실행할 수 있지만, SQL Warehouse를 사용하면 다음과 같은 이점이 있습니다.

| 비교 항목 | All-Purpose Cluster | SQL Warehouse |
|-----------|-------------------|---------------|
| ** 지원 언어**| Python, SQL, Scala, R | SQL 전용 |
| ** 주요 용도**| 개발, ETL, ML | SQL 분석, BI, 대시보드 |
| ** 최적화**| 범용 Spark | SQL 쿼리에 특화 (Photon 기본 포함) |
| ** 연결**| 노트북 | SQL Editor, BI 도구, JDBC/ODBC |
| ** 비용**| All-Purpose DBU 단가 | SQL DBU 단가 (더 저렴) |
| ** 동시성**| 제한적 | ** 멀티 클러스터 스케일링**으로 높은 동시성 지원 |
| **관리** | 사용자가 설정/관리 | 자동 관리 (특히 Serverless) |
| **보안** | Workspace 수준 | Unity Catalog 통합 (행/열 수준 보안) |

> 💡 **언제 SQL Warehouse를 사용하나요?** BI 도구 연결, 대시보드 조회, Ad-hoc SQL 분석, 정기 SQL 보고서 등 **SQL 중심 워크로드**에는 항상 SQL Warehouse를 사용하는 것이 권장됩니다. Python/Scala 코드가 필요한 개발이나 ML 학습에는 All-Purpose Cluster를 사용합니다.

---

## Photon 엔진

> 💡 **Photon**은 Databricks가 C++로 개발한 **차세대 벡터화 쿼리 엔진**입니다. Apache Spark의 SQL 실행 엔진을 대체하며, 동일한 쿼리를 최대 12배 빠르게 실행할 수 있습니다.

### Photon의 핵심 특징

| 특징 | 설명 |
|------|------|
| **네이티브 벡터화**| C++로 작성되어 CPU 캐시와 SIMD 명령어를 활용합니다 |
| ** 코드 변경 없음**| 기존 SQL 쿼리를 수정할 필요 없이 자동으로 Photon이 적용됩니다 |
| **Delta Lake 최적화**| Delta 파일 읽기/쓰기에 특화된 최적화가 적용되어 있습니다 |
| ** 자동 폴백**| Photon이 지원하지 않는 연산은 자동으로 Spark 엔진으로 전환됩니다 |

### Photon 성능 비교

| 항목 | 기존 Spark SQL | Photon 엔진 |
|------|---------------|-------------|
| 구현 | JVM 기반 | C++ 네이티브 |
| 처리 방식 | 행(Row) 단위 | 벡터화(Vectorized) 처리 |
| 메모리 | 가비지 컬렉션 오버헤드 | 직접 메모리 관리 |
| 성능 | 기준 | 최대 12배 빠름 |

> 💡 ** 벡터화(Vectorized) 처리란?**데이터를 한 행씩 처리하는 대신 ** 열(Column) 단위로 수천 개의 값을 한 번에 처리**하는 방식입니다. 현대 CPU의 SIMD(Single Instruction, Multiple Data) 명령어를 활용하여 대폭적인 성능 향상을 얻을 수 있습니다.

---

## SQL Warehouse 유형

Databricks는 세 가지 유형의 SQL Warehouse를 제공합니다.

| 유형 | 설명 | Photon | 고급 기능 | 가격 |
|------|------|--------|-----------|------|
| **Serverless**| Databricks가 완전 자동 관리. 수 초 내 시작합니다 | ✅ | ✅ | DBU에 VM 비용 포함 |
| **Pro**| 고급 기능 포함. 고객이 VM을 관리합니다 | ✅ | ✅ | DBU + VM 비용 별도 |
| **Classic**| 기본 SQL 기능만 제공합니다 | ✅ | ❌ | DBU + VM 비용 별도 |

### 유형별 기능 비교

| 기능 | Classic | Pro | Serverless |
|------|---------|-----|------------|
| SQL 쿼리 실행 | ✅ | ✅ | ✅ |
| JDBC/ODBC 연결 | ✅ | ✅ | ✅ |
| Photon 엔진 | ✅ | ✅ | ✅ |
| AI 함수 (ai_query 등) | ❌ | ✅ | ✅ |
| 예측 함수 | ❌ | ✅ | ✅ |
| 쿼리 태그 | ❌ | ✅ | ✅ |
| 인텔리전트 워크로드 관리 | ❌ | ✅ | ✅ |
| 시작 시간 | 3~5분 | 3~5분 | ** 수 초**|
| 인프라 관리 | 고객 | 고객 | **Databricks**|

> 🆕 **Serverless SQL Warehouse**가 현재 권장되는 기본 선택입니다. 수 초 만에 시작되고, 사용하지 않을 때는 자동으로 리소스를 해제하여 비용을 절약합니다.

---

## Warehouse 사이징 가이드

SQL Warehouse는 **T-Shirt 사이즈**로 크기를 선택합니다. 사이즈가 클수록 더 복잡한 쿼리를 빠르게 처리할 수 있습니다.

| 사이즈 | 클러스터 워커 수 | DBU/시간 (참고) | 적합한 워크로드 |
|--------|-----------------|----------------|---------------|
| **2X-Small**| 1 | ~2 | 가벼운 개발, 테스트, Ad-hoc 쿼리 |
| **X-Small**| 1 | ~4 | 소규모 쿼리, 소수 사용자 |
| **Small**| 2 | ~8 | 소규모 팀 분석 (5~10명) |
| **Medium**| 4 | ~16 | 중규모 워크로드, 대시보드 |
| **Large**| 8 | ~32 | 대규모 동시 쿼리, 복잡한 조인 |
| **X-Large**| 16 | ~64 | 고성능 분석, 대규모 집계 |
| **2X-Large ~ 4X-Large**| 32~64 | ~128~256 | 엔터프라이즈급 대규모 분석 |

> 💡 ** 사이징 팁**: 처음에는 **Small**또는 **Medium**으로 시작하고, 쿼리 프로필에서 병목을 분석한 후 필요하면 사이즈를 올리는 것이 좋습니다. 사이즈를 올리는 것보다 클러스터 수를 늘리는 것이 동시성에는 더 효과적입니다.

### 워크로드 유형별 추천

| 워크로드 | 추천 사이즈 | 클러스터 수 | 이유 |
|----------|-----------|------------|------|
| **개발/테스트**| 2X-Small | 1 | 비용 절감이 중요하며 동시 사용자가 적습니다 |
| ** 팀 대시보드**| Small~Medium | 2~4 | 여러 사용자가 동시에 대시보드를 조회합니다 |
| **BI 도구 연결**| Medium~Large | 3~5 | Tableau/Power BI가 다수의 병렬 쿼리를 실행합니다 |
| ** 대규모 ETL (SQL)**| Large~X-Large | 1~2 | 복잡한 조인과 대용량 집계가 필요합니다 |
| ** 전사 분석 플랫폼**| Medium | 5~10+ | 동시 사용자가 많으므로 클러스터 수로 확장합니다 |

---

## 스케일링 설정

### Auto-Stop (자동 중지)

사용하지 않을 때 자동으로 중지하여 비용을 절약합니다.

| 설정 | Serverless | Pro/Classic |
|------|------------|-------------|
| ** 기본값**| 10분 | 45분 |
| ** 최소값**| 5분 | 10분 |
| ** 권장값**| 10분 | 10~15분 |

> ⚠️ **Serverless는 Auto-Stop이 빠릅니다**: Serverless는 수 초 만에 재시작되므로, 짧은 Auto-Stop 시간을 설정해도 사용자 경험에 영향이 없습니다. Pro/Classic은 재시작에 3~5분이 소요되므로 너무 짧으면 불편할 수 있습니다.

### Scaling (멀티 클러스터 스케일링)

동시 사용자가 많을 때 자동으로 클러스터 수를 늘려 병렬 처리 능력을 확장합니다.

```
최소 클러스터: 1    ← 유휴 시 최소 유지 수 (비용 절감 시 1)
최대 클러스터: 5    ← 동시 부하 시 최대 확장 수
```

| 구성 요소 | 역할 |
|-----------|------|
| 쿼리 요청 | 사용자/BI 도구에서 요청 |
| 로드 밸런서 | 쿼리를 클러스터에 분배 |
| 클러스터 1 (항상 활성) | 기본 처리 |
| 클러스터 2 (자동 확장) | 부하 증가 시 자동 추가 |
| 클러스터 3 (자동 확장) | 추가 부하 처리 |
| 캐시 레이어 | 디스크 기반 결과 캐싱 |
| Delta Lake | 실제 데이터 저장소 |

> 💡 여기서의 "클러스터"는 SQL Warehouse 내부의 컴퓨팅 단위입니다. 하나의 클러스터가 하나의 무거운 쿼리를 처리하므로, 클러스터 수 = 동시에 처리할 수 있는 무거운 쿼리 수로 이해하시면 됩니다. (가벼운 쿼리는 하나의 클러스터에서 여러 개를 동시에 처리할 수 있습니다.)

### 쿼리 큐잉 동작 원리

모든 클러스터가 사용 중일 때, 새로운 쿼리는 **큐(Queue)** 에 대기합니다.

| 상황 | 동작 |
|------|------|
| 클러스터에 여유가 있음 | 즉시 실행됩니다 |
| 클러스터가 모두 사용 중 | 큐에 대기하며, 스케일 아웃이 가능하면 새 클러스터를 추가합니다 |
| 최대 클러스터에 도달 | 기존 쿼리가 완료될 때까지 큐에서 대기합니다 |
| 큐 대기 시간 초과 | 쿼리가 취소됩니다 (기본 타임아웃은 설정 가능) |

---

## 비용 관리

### DBU 비용 구조

| 유형 | DBU 단가 (참고) | VM 비용 | 총 비용 |
|------|----------------|---------|---------|
| **Classic**| 낮음 | 별도 과금 | DBU + VM |
| **Pro**| 중간 | 별도 과금 | DBU + VM |
| **Serverless**| 높음 | **DBU에 포함**| DBU만 (올인원) |

> 💡 **Serverless가 더 저렴할 수 있습니다**: DBU 단가는 Serverless가 높지만, VM 비용이 포함되어 있고, 유휴 시간 비용이 없으며, 관리 인건비가 절감되므로 **총 소유 비용(TCO)** 은 Serverless가 유리한 경우가 많습니다.

### 비용 모니터링

```sql
-- 시스템 테이블을 통한 SQL Warehouse 비용 모니터링
SELECT
    warehouse_id,
    usage_date,
    SUM(usage_quantity) AS total_dbus,
    ROUND(SUM(usage_quantity) * 0.55, 2) AS estimated_cost_usd  -- 예시 단가
FROM system.billing.usage
WHERE usage_metadata.warehouse_id IS NOT NULL
  AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAY
GROUP BY warehouse_id, usage_date
ORDER BY usage_date DESC;

-- 쿼리별 비용 추적
SELECT
    query_id,
    user_name,
    warehouse_id,
    total_duration_ms,
    rows_produced,
    ROUND(total_task_duration_ms / 3600000.0 * 2, 4) AS estimated_dbus  -- 근사치
FROM system.query.history
WHERE start_time >= CURRENT_DATE() - INTERVAL 7 DAY
ORDER BY total_duration_ms DESC
LIMIT 20;
```

---

## BI 도구 연결

SQL Warehouse는 **JDBC/ODBC**표준 프로토콜을 지원하여 다양한 BI 도구와 연결할 수 있습니다.

| BI 도구 | 연결 방식 | 특징 |
|---------|-----------|------|
| **Tableau**| Databricks 전용 커넥터 또는 ODBC | 가장 많이 사용되는 BI 연결입니다 |
| **Power BI**| Databricks 전용 커넥터 | DirectQuery 모드 지원으로 실시간 조회가 가능합니다 |
| **Looker**| JDBC | LookML 모델에서 Databricks 테이블을 직접 참조합니다 |
| **dbt**| Databricks 어댑터 | SQL 기반 데이터 변환 파이프라인을 구축합니다 |
| **Excel**| ODBC 또는 Databricks Excel Add-in | 스프레드시트에서 직접 SQL을 실행합니다 |
| **Sigma**| Databricks 커넥터 | 클라우드 네이티브 BI 도구입니다 |

> 🆕 **Databricks Excel Add-in**: Public Preview로 출시되어, Excel에서 직접 SQL을 실행하고 데이터를 가져올 수 있습니다.

### 연결 정보 확인

SQL Warehouse 상세 페이지에서 **Connection details**탭을 클릭하면 연결에 필요한 정보를 확인할 수 있습니다.

| 정보 | 설명 | 예시 |
|------|------|------|
| **Server Hostname**| 워크스페이스의 호스트명입니다 | `adb-1234567890.12.azuredatabricks.net` |
| **HTTP Path**| SQL Warehouse의 고유 경로입니다 | `/sql/1.0/warehouses/abc123def456` |
| **Port**| HTTPS 포트입니다 | `443` |

### Partner Connect

**Partner Connect**를 사용하면 BI 도구 연결을 몇 번의 클릭만으로 설정할 수 있습니다.

1. Workspace에서 **Partner Connect**메뉴를 엽니다
2. 연결할 BI 도구(예: Tableau, Power BI)를 선택합니다
3. SQL Warehouse와 카탈로그를 지정합니다
4. 자동으로 서비스 프린시펄과 토큰이 생성됩니다
5. BI 도구에서 자동으로 연결이 설정됩니다

---

## 모니터링

### 쿼리 히스토리

SQL Warehouse에서 실행된 모든 쿼리의 이력을 확인할 수 있습니다.

| 확인 항목 | 설명 |
|-----------|------|
| ** 실행 시간**| 쿼리 시작/종료 시간, 소요 시간을 확인합니다 |
| ** 사용자**| 누가 쿼리를 실행했는지 추적합니다 |
| ** 상태**| 성공, 실패, 취소 등의 상태를 확인합니다 |
| ** 리소스 사용**| 스캔한 데이터 크기, 반환된 행 수를 확인합니다 |

### 쿼리 프로필

개별 쿼리의 ** 실행 계획과 병목 지점**을 상세히 분석할 수 있습니다.

```sql
-- 시스템 테이블로 느린 쿼리 모니터링
SELECT
    query_id,
    query_text,
    user_name,
    total_duration_ms / 1000.0 AS duration_seconds,
    rows_produced,
    bytes_read
FROM system.query.history
WHERE warehouse_id = '<your-warehouse-id>'
  AND total_duration_ms > 60000  -- 1분 이상 소요된 쿼리
  AND start_time >= CURRENT_DATE() - INTERVAL 1 DAY
ORDER BY total_duration_ms DESC;
```

---

## 실습: SQL Warehouse 생성 및 최적 설정

### Step 1: Warehouse 생성

1. Databricks Workspace에서 **SQL Warehouses**메뉴로 이동합니다
2. **Create SQL Warehouse** 클릭합니다
3. 다음 설정을 입력합니다:

| 설정 | 값 | 이유 |
|------|-----|------|
| Name | `team-analytics` | 용도를 명확히 하는 이름입니다 |
| Type | Serverless | 빠른 시작, 자동 관리를 위해 선택합니다 |
| Size | Small | 10명 이하의 팀에 적합합니다 |
| Min clusters | 1 | 최소 1개 클러스터를 항상 유지합니다 |
| Max clusters | 3 | 동시 부하 시 최대 3개까지 확장합니다 |
| Auto stop | 10분 | Serverless는 재시작이 빠르므로 짧게 설정합니다 |

### Step 2: 쿼리 실행 테스트

```sql
-- SQL Editor에서 테스트 쿼리 실행
SELECT
    DATE_TRUNC('month', order_date) AS order_month,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value
FROM production.ecommerce.orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY order_month;
```

### Step 3: 쿼리 프로필 분석

1. 쿼리 실행 후 **Query Profile**탭을 클릭합니다
2. **Scan**단계에서 읽은 파일 수와 데이터 크기를 확인합니다
3. **Shuffle**단계에서 네트워크 데이터 이동량을 확인합니다
4. 병목이 있으면 사이즈 업 또는 쿼리 최적화를 고려합니다

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SQL Warehouse**| SQL 분석에 최적화된 전용 컴퓨팅 리소스입니다 |
| **Photon**| C++ 벡터화 엔진으로 SQL 쿼리를 최대 12배 빠르게 실행합니다 |
| **Serverless**| 자동 관리, 수 초 시작, 비용 효율적. 대부분의 경우에 권장됩니다 |
| **T-Shirt 사이즈**| 워크로드 규모에 따라 2X-Small ~ 4X-Large 중 선택합니다 |
| ** 멀티 클러스터**| 동시 사용자 증가 시 클러스터 수를 자동으로 늘려 대응합니다 |
| **Auto-Stop**| 유휴 시 자동 중지하여 비용을 절약합니다 |
| **JDBC/ODBC** | 표준 프로토콜로 Tableau, Power BI 등 BI 도구와 연결됩니다 |

---

## 참고 링크

- [Databricks: SQL Warehouses](https://docs.databricks.com/aws/en/compute/sql-warehouse/)
- [Databricks: SQL Warehouse Types](https://docs.databricks.com/aws/en/compute/sql-warehouse/warehouse-types.html)
- [Databricks: Photon](https://docs.databricks.com/aws/en/compute/photon.html)
- [Databricks: Query Profile](https://docs.databricks.com/aws/en/compute/sql-warehouse/query-profile.html)
- [Azure Databricks: SQL Warehouses](https://learn.microsoft.com/en-us/azure/databricks/compute/sql-warehouse/)
- [Databricks: BI tool connections](https://docs.databricks.com/aws/en/partners/bi/)
- [Databricks Blog: Photon Performance](https://www.databricks.com/blog/photon-technical-deep-dive)
