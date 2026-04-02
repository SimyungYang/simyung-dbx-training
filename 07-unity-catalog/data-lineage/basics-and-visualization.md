# 리니지 기본 개념과 시각화

## 리니지란?

> 💡 **데이터 리니지(Data Lineage)** 란 데이터가 **어디서 왔고(upstream), 어디로 가는지(downstream)** 를 추적하는 기능입니다. "이 테이블의 데이터는 어떤 소스에서 왔지?", "이 컬럼을 변경하면 어디에 영향이 가지?"라는 질문에 답할 수 있습니다.

---

## 왜 데이터 리니지가 필요한가?

### 데이터 신뢰성 문제

현대 데이터 파이프라인은 수십 개의 테이블을 거쳐 데이터가 흐릅니다. 소스 시스템에서 변경이 발생하면 그 영향이 어디까지 전파되는지 파악하기 어렵습니다. **데이터 리니지** 가 없으면 다음과 같은 문제가 발생합니다.

- Gold 테이블 값이 갑자기 달라졌을 때 원인 파악에 수 시간 소요
- 소스 스키마 변경이 어떤 downstream 리포트에 영향을 주는지 알 수 없음
- 특정 데이터가 신뢰할 수 있는지 확인할 방법 부재

### 규제 준수 (GDPR, SOX)

| 규제 | 요구 사항 | 리니지 활용 |
|------|-----------|-------------|
| **GDPR** | 개인정보(PII)가 어디에 저장·전달되는지 파악 | `email`, `phone` 컬럼의 전파 경로 추적 |
| **SOX** | 재무 데이터의 출처와 변환 과정 감사 | 재무 지표 컬럼의 upstream 추적 |
| **개인정보보호법** | 데이터 이동 경로 문서화 | 국내외 시스템 간 PII 흐름 확인 |

규제 감사(Audit) 시 "이 수치가 어디서 왔는지"를 즉시 제시할 수 있어야 합니다. 수동 문서화는 현실적으로 불가능하며, **자동 리니지 캡처** 가 유일한 실용적 대안입니다.

### 영향도 분석 (Impact Analysis)

스키마 변경 전 영향 범위를 사전에 파악합니다.

```
silver_orders.total_amount 컬럼 타입 변경 시 영향 범위:
  → gold_daily_revenue.total_revenue       (SUM 집계)
  → gold_customer_360.lifetime_revenue     (고객별 SUM)
  → revenue_dashboard                      (AI/BI Dashboard)
  → churn_prediction_model                 (MLflow 모델 Feature)
```

### 디버깅 (Debugging)

**데이터 품질 이슈** 발생 시 역방향으로 소스를 추적합니다. Gold 테이블 이상 → Silver 테이블 확인 → Bronze 테이블 확인 순서로 빠르게 문제를 격리할 수 있습니다.

---

## 어떻게 작동하는가?

### Unity Catalog의 자동 리니지 캡처 메커니즘

Unity Catalog는 **Spark 실행 계획(Execution Plan)** 을 분석하여 리니지를 자동으로 캡처합니다. 별도의 설정 없이 Unity Catalog가 활성화된 환경에서 쿼리를 실행하면 자동으로 수집됩니다.

```
쿼리 실행
  → Spark QueryExecution Plan 분석
  → 입력 테이블/컬럼 및 출력 테이블/컬럼 추출
  → Unity Catalog Metastore에 리니지 메타데이터 저장
  → system.access.table_lineage / column_lineage 테이블에서 조회 가능
```

### 캡처 범위

| 작업 유형 | 추적 여부 | 설명 |
|-----------|----------|------|
| **SQL 쿼리** (CTAS, INSERT, MERGE) | ✅ | SQL Warehouse, Notebook에서 실행한 쿼리 |
| **DLT Pipeline** | ✅ | Streaming Table, Materialized View의 소스 추적 |
| **Spark DataFrame** | ✅ | `df.write.saveAsTable()` 등 Unity Catalog 테이블 기록 |
| **MLflow 모델** | ✅ | 모델이 어떤 Feature Table에서 학습되었는지 |
| **AI/BI Dashboard** | ✅ | 대시보드가 어떤 테이블을 조회하는지 |
| **외부 도구 (JDBC)** | ⚠️ 부분적 | SQL Warehouse를 통한 외부 BI 도구 접근 |
| **외부 ETL 도구** | ❌ | Databricks 외부에서 직접 S3/Delta 접근 시 미추적 |
| **동적 SQL** | ⚠️ 제한적 | `EXECUTE IMMEDIATE` 등 런타임 생성 쿼리는 캡처 불완전 |

---

## 리니지 시각화

### Catalog Explorer UI에서 확인하는 방법

1. 왼쪽 메뉴에서 **Catalog** 선택
2. 조회할 테이블 클릭
3. **Lineage** 탭 클릭
4. **Upstream** (이 테이블의 소스)과 **Downstream** (이 테이블을 참조하는 객체)을 그래프로 확인

그래프에서 노드를 클릭하면 해당 테이블/노트북/Job의 상세 정보로 이동할 수 있습니다.

![Unity Catalog 데이터 리니지 그래프 개요](https://docs.databricks.com/aws/en/assets/images/uc-lineage-overview-d654e26bc9af96d30f971da157eb1497.png)

### 컬럼 레벨 리니지 (Column-Level Lineage)

테이블 수준 리니지에서 더 나아가, 특정 **컬럼** 이 어떤 컬럼에서 파생되었는지까지 추적합니다.

- 테이블 Lineage 탭 → 컬럼 이름 클릭
- 해당 컬럼의 upstream/downstream 컬럼이 하이라이트됩니다

![컬럼 수준 리니지 예시](https://docs.databricks.com/aws/en/assets/images/uc-lineage-column-lineage-89a091244240d677fe7a6e0503076089.png)

*출처: [Databricks 공식 문서 — Data lineage](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-lineage.html)*

---

## 리니지의 범위 예시

### 테이블 수준 리니지

| 단계 | 소스/대상 | 설명 |
|------|-----------|------|
| 소스 | MySQL (외부 소스) | 원본 데이터 |
| Bronze | `bronze_orders` | 원본 데이터를 수집 |
| Silver | `silver_orders` | 정제된 데이터 |
| Gold | `gold_daily_revenue` | 일별 매출 집계 → 매출 대시보드에 활용 |
| Gold | `gold_customer_360` | 고객 통합 뷰 → 고객 분석 대시보드 및 이탈 예측 모델에 활용 |

### 컬럼 수준 리니지

```
silver_orders.total_amount
  ← bronze_orders.raw_data:amount  (JSON에서 추출)

gold_daily_revenue.total_revenue
  ← silver_orders.total_amount  (SUM 집계)

gold_customer_360.lifetime_revenue
  ← silver_orders.total_amount  (고객별 SUM)
```

---

## SQL로 리니지 조회하기

Unity Catalog의 **시스템 테이블(System Tables)** 을 통해 리니지 데이터를 프로그래밍 방식으로 조회할 수 있습니다.

> 참고: 시스템 테이블은 `system.access` 스키마에 위치합니다. 과거 일부 문서에서 사용되던 `system.lineage` 스키마는 현재 `system.access.table_lineage` / `system.access.column_lineage`로 통합되었습니다.

### system.access.table_lineage 조회

```sql
-- 특정 테이블의 downstream: 이 테이블을 참조하는 모든 객체
SELECT
    target_table_full_name   AS downstream_table,
    target_type,
    event_time,
    created_by
FROM system.access.table_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
ORDER BY event_time DESC;

-- 특정 테이블의 upstream: 이 테이블이 참조하는 소스
SELECT
    source_table_full_name   AS upstream_table,
    source_type,
    event_time,
    created_by
FROM system.access.table_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue'
ORDER BY event_time DESC;
```

### system.access.column_lineage 조회

```sql
-- 특정 컬럼의 upstream: total_revenue가 어디서 왔는지
SELECT
    source_table_full_name,
    source_column_name,
    target_column_name,
    event_time
FROM system.access.column_lineage
WHERE target_table_full_name = 'production.ecommerce.gold_daily_revenue'
  AND target_column_name = 'total_revenue'
ORDER BY event_time DESC;

-- PII 컬럼의 전파 경로 추적 (GDPR 감사 대응)
SELECT
    source_table_full_name,
    source_column_name,
    target_table_full_name,
    target_column_name,
    event_time
FROM system.access.column_lineage
WHERE source_column_name IN ('email', 'phone', 'ssn', 'name')
ORDER BY source_table_full_name, target_table_full_name;

-- 미사용 테이블 후보 식별 (비용 최적화)
-- downstream이 한 번도 기록되지 않은 테이블 확인
SELECT DISTINCT source_table_full_name
FROM system.access.table_lineage
WHERE source_table_full_name NOT IN (
    SELECT DISTINCT target_table_full_name
    FROM system.access.table_lineage
)
ORDER BY 1;
```

---

## 실전 활용 시나리오

### 1. 스키마 변경 영향도 분석

`silver_orders.total_amount` 컬럼의 타입을 `DECIMAL(10,2)`에서 `DOUBLE`로 변경하려는 경우:

```sql
-- 변경 전, 영향 받는 downstream 테이블 전체 목록 조회
SELECT DISTINCT
    target_table_full_name,
    target_type
FROM system.access.column_lineage
WHERE source_table_full_name = 'production.ecommerce.silver_orders'
  AND source_column_name = 'total_amount';
```

결과를 확인 후 영향 받는 팀에 사전 공지하고 변경을 진행합니다.

### 2. 데이터 품질 이슈 역추적

Gold 테이블의 `total_revenue` 값이 어제 대비 30% 하락한 경우:

1. `system.access.column_lineage`로 `total_revenue`의 upstream 컬럼 확인
2. 각 upstream 테이블에서 동일 시점의 데이터 이상 여부 확인
3. 문제 발생 단계(Bronze/Silver) 격리 후 파이프라인 수정

### 3. 규제 감사 대응 (GDPR 삭제 요청)

고객이 데이터 삭제를 요청한 경우, PII가 복사된 모든 테이블을 파악해야 합니다.

```sql
-- email 컬럼이 전파된 모든 테이블 확인
SELECT DISTINCT target_table_full_name
FROM system.access.column_lineage
WHERE source_column_name = 'email'
ORDER BY 1;
```

### 4. 비용 최적화 — 미사용 테이블 식별

리니지 데이터와 접근 로그를 결합하여, 생성은 되었지만 어떤 downstream에서도 참조되지 않고 조회도 없는 테이블을 식별합니다. 해당 테이블은 삭제 혹은 아카이빙 대상이 됩니다.

---

## 장단점과 한계

### 장점

- **설정 불필요**: Unity Catalog 활성화만으로 자동 수집
- **컬럼 레벨 세분화**: 단순 테이블 의존성을 넘어 컬럼까지 추적
- **UI + SQL 이중 접근**: 시각적 탐색과 프로그래밍 방식 모두 지원
- **다양한 객체 지원**: 테이블, 노트북, Job, 대시보드, MLflow 모델 모두 연결

### 한계 및 주의사항

| 한계 | 설명 |
|------|------|
| **외부 도구 미추적** | Spark/SQL Warehouse를 거치지 않고 S3/Delta에 직접 쓰는 경우 캡처 안 됨 |
| **동적 SQL 제한** | `EXECUTE IMMEDIATE`나 런타임 생성 쿼리는 캡처 불완전 |
| **보존 기간** | 기본 1년(365일). 장기 감사가 필요하면 별도 아카이빙 필요 |
| **Unity Catalog 전용** | Hive Metastore 기반 레거시 테이블은 리니지 미지원 |
| **반영 지연** | 리니지 데이터는 수 분~수십 분 지연 후 system 테이블에 반영 |

---

## 베스트 프랙티스

### 리니지 품질 향상

1. **네이밍 규칙 일관화**: `bronze_`, `silver_`, `gold_` 접두사 사용으로 계층 파악이 용이합니다
2. **노트북 대신 Job 사용**: 노트북은 실행 주체가 불명확할 수 있으므로, 정기 파이프라인은 **Databricks Job** 으로 등록해 리니지에 Job 이름이 기록되도록 합니다
3. **명시적 CTAS 사용**: `CREATE TABLE ... AS SELECT`를 사용하면 컬럼 수준 리니지가 더 정확하게 캡처됩니다
4. **PII 태그와 리니지 결합**: Unity Catalog **태그(Tag)** 로 PII 컬럼을 표시하고, 리니지 쿼리 시 태그 기반 필터링을 활용합니다

### 정기적 리니지 검토

```sql
-- 월별 리니지 검토: 최근 30일간 변경된 테이블 의존 관계 확인
SELECT
    source_table_full_name,
    target_table_full_name,
    COUNT(*)        AS lineage_event_count,
    MAX(event_time) AS last_seen
FROM system.access.table_lineage
WHERE event_time >= CURRENT_DATE - INTERVAL 30 DAYS
GROUP BY 1, 2
ORDER BY last_seen DESC;
```

- 분기별로 downstream이 없는 테이블 목록을 생성하여 테이블 수명주기를 관리합니다
- 신규 파이프라인 배포 후 리니지 그래프를 검토해 의도하지 않은 의존성이 생기지 않았는지 확인합니다

---

## 참고 링크

- [Databricks 공식 문서 — Data lineage with Unity Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-lineage.html)
- [System tables reference — table_lineage / column_lineage](https://docs.databricks.com/aws/en/administration-guide/system-tables/lineage.html)
- [Unity Catalog 시스템 테이블 개요](https://docs.databricks.com/aws/en/administration-guide/system-tables/index.html)
- [Column-level lineage 상세](https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-lineage.html#column-level-lineage)
