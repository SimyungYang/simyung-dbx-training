# AI/BI 기술 아키텍처 심화

---

## 1. 왜 Compound AI System인가 — 단일 LLM의 한계

Databricks AI/BI가 단일 LLM이 아니라 **Compound AI System(복합 AI 시스템)** 으로 설계된 데에는 명확한 이유가 있습니다.

### 단일 LLM만으로는 부족한 이유

| 한계 | 설명 | 결과 |
|------|------|------|
| **환각 (Hallucination)** | LLM은 존재하지 않는 컬럼명이나 테이블명을 만들어낼 수 있습니다 | 실행 불가한 SQL 생성 |
| **SQL 정확도 부족** | 일반 LLM은 기업 데이터 스키마를 알지 못합니다 | 의미상 틀린 집계 결과 |
| **컨텍스트 한계** | 수백 개의 테이블, 수천 개의 컬럼을 하나의 프롬프트에 담을 수 없습니다 | 스키마 매칭 실패 |
| **보안 미적용** | LLM 자체는 행(row)/열(column) 수준의 접근 제어를 인식하지 못합니다 | 권한 없는 데이터 노출 위험 |
| **비결정성** | 동일 질문에 매번 다른 SQL이 생성될 수 있습니다 | 대시보드 결과 불일치 |

이러한 한계를 극복하기 위해, AI/BI는 각각의 역할에 특화된 여러 컴포넌트가 협력하는 파이프라인 구조를 채택합니다. 각 단계에서 검증과 보정이 이루어지기 때문에, 단일 LLM 대비 훨씬 높은 신뢰성과 정확도를 달성할 수 있습니다.

---

## 2. Compound AI System 아키텍처 상세

Databricks AI/BI는 아래 컴포넌트들이 순차적으로 협력하는 파이프라인입니다.

| 컴포넌트 | 역할 | 동작 원리 |
|----------|------|-----------|
| **NLU (Natural Language Understanding)** | 질문 의도 파악 | 사용자의 자연어에서 의도(intent), 엔터티(entity), 시간 필터, 집계 조건을 추출합니다. "지난 달 매출"처럼 상대적 시간 표현도 절대 날짜로 변환합니다 |
| **Schema Understanding** | 테이블/컬럼 매핑 | Unity Catalog의 메타데이터(테이블명, 컬럼명, COMMENT, 데이터 타입)를 벡터 임베딩으로 색인하여 의미적으로 가장 가까운 테이블과 컬럼을 선택합니다. Metric View가 정의된 경우 해당 정의를 우선 적용합니다 |
| **SQL Generator** | SQL 생성 | 추출된 의도와 매핑된 스키마를 결합하여 Databricks SQL 방언에 최적화된 SELECT 문을 생성합니다. INSERT/UPDATE/DELETE는 생성하지 않습니다 |
| **SQL Validator** | SQL 검증 | 생성된 SQL의 문법 오류, 존재하지 않는 오브젝트 참조, Unity Catalog 권한 문제를 사전에 검출합니다. 오류 발견 시 Self-correction을 시도합니다 |
| **Result Formatter** | 결과 포맷팅 | 쿼리 결과의 데이터 타입과 카디널리티를 분석하여 자연어 요약, 표, 적절한 차트 유형을 자동으로 선택합니다 |
| **Feedback Loop** | 품질 개선 | 사용자 피드백(좋아요/싫어요), 인증된 답변(Certified Answers), 관리자 설정 지침을 반영하여 다음 응답의 품질을 지속적으로 개선합니다 |

---

## 3. NL→SQL 변환 파이프라인 상세

### 정상 처리 흐름

Genie가 사용자의 질문을 SQL로 변환하는 과정을 단계별로 살펴보겠습니다.

**사용자**: "지난 달 서울 매출 상위 5개 제품을 알려줘"

| 단계 | 처리 | 상세 |
|------|------|------|
| **1단계: NLU (의도 분석)** | 자연어 → 구조화 | 시간 필터: "지난 달" → `DATE_TRUNC('month', CURRENT_DATE - INTERVAL 1 MONTH)`, 공간 필터: "서울" → `region = '서울'`, 측정값: "매출" → `SUM(order_amount)`, 차원: "제품" → `product_name`, 수량 제한: "상위 5개" → `ORDER BY … DESC LIMIT 5` |
| **2단계: Schema Matching** | 테이블 매핑 | 테이블: `gold_orders`, 컬럼 매핑: 매출 → `order_amount`, 제품 → `product_name`. Metric View 존재 시 정의를 우선 사용 |
| **3단계: SQL Generation** | SQL 생성 | 안전한 SELECT 문만 생성 (INSERT/UPDATE/DELETE 불가) |
| **4단계: Validation** | SQL 검증 | 권한 확인, 존재하는 컬럼 확인, 문법 검사. 통과 시 다음 단계로 진행 |
| **5단계: Execution** | SQL 실행 | Serverless SQL Warehouse에서 실행, Photon 엔진 활용. 결과를 자연어 요약 + 차트로 변환 |

### 실패 케이스 처리와 Self-correction 메커니즘

모든 질문이 깔끔하게 처리되는 것은 아닙니다. AI/BI는 다음과 같은 실패 상황에서 자동 복구를 시도합니다.

| 실패 유형 | 발생 원인 | 처리 방식 |
|-----------|-----------|-----------|
| **모호한 질문** | "매출이 좋은 제품" — '좋은'의 기준이 불명확 | Genie가 사용자에게 명확화 질문을 역으로 던집니다 ("기간을 지정해 주시겠어요?") |
| **스키마 매칭 실패** | "리텐션율"이라는 컬럼이 실제로 없는 경우 | 유사 컬럼을 후보로 제시하거나, 관리자가 등록한 지침(Instruction)을 참조합니다 |
| **SQL 실행 오류** | 생성된 SQL이 실행 시점에 오류 발생 | **Self-correction**: 오류 메시지를 컨텍스트에 포함하여 SQL을 최대 N회 재생성 시도합니다 |
| **권한 오류** | 사용자가 해당 테이블에 접근 권한 없음 | Unity Catalog 오류를 포착하여 "접근 권한이 없습니다" 메시지를 안전하게 반환합니다 |
| **결과 없음** | 조건에 맞는 데이터가 없음 | "해당 조건에 맞는 데이터가 없습니다"라고 명확히 안내하며, 조건 완화를 제안합니다 |

> **Self-correction** 이란 SQL Validator가 오류를 감지하면, 오류 메시지와 원래 질문을 다시 SQL Generator에 피드백하여 수정된 SQL을 재생성하는 루프입니다. 이 과정이 내부적으로 자동화되어 있어 사용자는 대부분 오류 없이 결과를 받게 됩니다.

---

## 4. Certified Answers(인증된 답변)

### 개념과 필요성

**Certified Answers(인증된 답변)** 는 관리자(또는 데이터 큐레이터)가 특정 질문에 대한 정답 SQL을 미리 등록해 두는 기능입니다.

기업 환경에서는 "이번 달 KPI는?" 또는 "월간 활성 사용자(MAU)가 몇 명이야?"처럼 반복적으로 물어보는 핵심 질문이 있습니다. 이런 질문에 대해 AI가 매번 새로 SQL을 생성하면 다음과 같은 문제가 발생합니다:

- 정의가 다른 컬럼을 선택해 결과가 달라질 수 있습니다
- 복잡한 비즈니스 로직(예: MAU 산정 기준)을 LLM이 정확히 추론하지 못할 수 있습니다
- 결과에 대한 신뢰도가 낮아집니다

### 동작 방식

1. 관리자가 Genie Space의 **Instructions** 탭에서 자주 묻는 질문과 정답 SQL을 등록합니다.
2. 사용자가 유사한 질문을 입력하면, NLU가 해당 질문을 인증된 답변 목록과 매칭합니다.
3. 매칭 성공 시, SQL Generator를 거치지 않고 등록된 검증 SQL을 직접 실행합니다.
4. 결과에 "인증된 답변" 배지가 표시되어 사용자가 신뢰할 수 있음을 확인할 수 있습니다.

| 구분 | 일반 응답 | 인증된 답변 |
|------|-----------|------------|
| SQL 출처 | AI가 실시간 생성 | 관리자가 사전 등록 |
| 정확도 | 메타데이터 품질에 의존 | 보장됨 |
| 표시 방식 | 일반 텍스트 | "Certified" 배지 표시 |
| 적합한 용도 | 탐색적 질문 | KPI, 핵심 지표 |

---

## 5. 메타데이터가 품질을 결정한다

AI/BI의 SQL 생성 정확도는 Unity Catalog에 등록된 **메타데이터(metadata) 품질** 에 직접적으로 의존합니다. 좋은 메타데이터 없이는 좋은 AI 응답도 없습니다.

### COMMENT의 역할

Unity Catalog에서 테이블과 컬럼에 작성하는 `COMMENT`는 Schema Understanding 컴포넌트가 의미적 매칭을 수행할 때 핵심 입력으로 사용됩니다.

```sql
-- 컬럼 COMMENT 예시
ALTER TABLE gold.sales.orders
  ALTER COLUMN order_amount COMMENT '주문 금액 (VAT 제외, 단위: 원). 반품 처리된 주문은 제외됨.';

ALTER TABLE gold.sales.orders
  ALTER COLUMN region COMMENT '배송지 기준 광역시도 (예: 서울, 경기, 부산)';
```

"서울 매출"이라는 질문이 들어왔을 때, `region` 컬럼의 COMMENT에 "광역시도"라는 설명이 있으면 AI가 올바르게 `region = '서울'`로 매핑할 수 있습니다.

### Metric View (메트릭 뷰)

**Metric View** 는 비즈니스 지표를 SQL로 공식 정의해 두는 기능입니다. "MAU", "리텐션율" 같은 복잡한 개념을 정확하게 계산하는 SQL 스니펫을 사전 등록하면, AI/BI가 해당 지표를 질문받을 때 이 정의를 우선 사용합니다.

### 테이블/컬럼 명명 규칙

| 나쁜 예 | 좋은 예 | 이유 |
|---------|---------|------|
| `tbl_ord_v2` | `orders` | 자연어와의 매핑이 명확합니다 |
| `amt` | `order_amount` | 약어 없이 풀네임을 사용해야 AI가 인식합니다 |
| `flg_del` | `is_deleted` | Boolean 컬럼은 `is_` 접두어가 의미를 명확히 합니다 |
| `dt` | `order_date` | 무슨 날짜인지 컨텍스트가 필요합니다 |

> 실제 운영 환경에서 AI/BI 도입 시 **메타데이터 정비** 를 선행 작업으로 반드시 수행해야 합니다. COMMENT 작성률이 낮은 테이블은 AI 응답 품질이 현저히 떨어집니다.

---

## 6. Photon 엔진 — JVM vs Native 성능 차이

### 개요

AI/BI의 쿼리는 **Serverless SQL Warehouse** 에서 실행되며, 내부적으로 **Photon 엔진** 이 핵심 역할을 합니다.

> **Photon** 은 Databricks가 C++로 작성한 **네이티브 벡터화 쿼리 엔진** 입니다. JVM 기반 Spark SQL 엔진을 대체하여 BI 쿼리에서 2~8배의 성능 향상을 제공합니다.

### JVM 기반 엔진 vs Photon 비교

| 비교 항목 | JVM 기반 Spark SQL | Photon (C++ Native) |
|-----------|-------------------|---------------------|
| **메모리 관리** | JVM Heap 사용, GC(Garbage Collection) 발생 | Off-heap 메모리 직접 관리, GC 없음 |
| **SIMD 활용** | JVM JIT 컴파일러에 의존, 제한적 | CPU의 AVX-512 등 SIMD 명령어 직접 활용 |
| **벡터화 실행** | 행(row) 단위 처리 위주 | 컬럼(column) 단위 벡터 배치 처리 |
| **코드 생성** | JVM 바이트코드로 컴파일 | 네이티브 머신 코드로 직접 컴파일 |
| **콜드 스타트** | JVM 초기화 오버헤드 존재 | 빠른 초기화 |

### AI/BI에서 Photon 최적화의 효과

| Photon 최적화 | 설명 | AI/BI 연관성 |
|-------------|------|-------------|
| **벡터화 실행 (Vectorized Execution)** | SIMD 명령어로 한 번에 여러 행을 처리합니다 | 대시보드의 집계 쿼리(SUM, COUNT, AVG)가 빠르게 실행됩니다 |
| **코드 생성 (Codegen)** | 쿼리를 네이티브 코드로 컴파일하여 실행합니다 | 반복 실행되는 대시보드 쿼리에 효과적입니다 |
| **메모리 관리 (Off-heap)** | GC 오버헤드를 제거합니다 | 동시 다수 쿼리(대시보드 동시 로딩)에 안정적입니다 |
| **인텔리전트 캐싱** | Disk I/O 결과를 메모리에 캐시합니다 | 대시보드 새로고침 시 빠른 응답을 제공합니다 |

---

## 7. 보안과 거버넌스 — Unity Catalog 기반 권한 제어

### AI/BI에서 보안이 적용되는 방식

AI/BI는 독자적인 권한 시스템을 갖지 않습니다. 모든 데이터 접근은 **Unity Catalog** 의 권한 정책을 그대로 따릅니다. 이는 AI를 통한 우회 접근이 구조적으로 불가능함을 의미합니다.

```
사용자 질문
    ↓
Genie NL→SQL 생성
    ↓
SQL Warehouse 실행
    ↓ (이 시점에서 Unity Catalog 권한 검사 적용)
Unity Catalog
    ├── Table-level ACL: 테이블 접근 권한
    ├── Column-level Security: 특정 컬럼 마스킹/숨김
    └── Row-level Security: 특정 행만 보이도록 필터링
```

### Row-level Security (행 수준 보안)

Unity Catalog의 **Row Filter** 를 설정하면, 사용자 속성(예: 소속 부서, 국가)에 따라 자동으로 WHERE 조건이 추가됩니다.

```sql
-- Row Filter 함수 예시
CREATE FUNCTION sales.region_filter(region_col STRING)
  RETURN region_col = current_user_region();  -- 현재 사용자의 소속 지역만 허용

-- 테이블에 적용
ALTER TABLE gold.sales.orders
  SET ROW FILTER sales.region_filter ON (region);
```

Genie에서 "전체 매출"을 물어봐도, Row Filter가 적용된 사용자는 자신의 지역 데이터만 조회됩니다.

### Column-level Security (열 수준 보안)

**Column Mask** 를 통해 민감 컬럼(예: 주민번호, 이메일)을 마스킹할 수 있습니다. Genie가 해당 컬럼을 포함한 SQL을 생성하더라도, 마스킹 규칙이 자동 적용됩니다.

| 권한 수준 | 적용 대상 | AI/BI 동작 |
|-----------|-----------|-----------|
| **CATALOG/SCHEMA 권한** | 카탈로그/스키마 전체 | 권한 없는 테이블은 스키마 탐색에서도 보이지 않습니다 |
| **TABLE 권한** | 테이블 단위 | 권한 없는 테이블 SELECT 시 오류 반환 (안전 메시지로 변환) |
| **Column Mask** | 특정 컬럼 | Genie 결과에서 해당 컬럼이 자동 마스킹됩니다 |
| **Row Filter** | 특정 행 | 사용자별로 보이는 데이터 범위가 자동 제한됩니다 |

---

## 8. 한계점과 개선 방향

현재 AI/BI(Genie)가 잘 다루지 못하는 영역을 솔직하게 이해하는 것이 중요합니다.

### 현재 한계

| 한계 영역 | 구체적 내용 | 우회 방법 |
|-----------|------------|-----------|
| **복잡한 다중 JOIN** | 5개 이상의 테이블을 조인하는 복잡한 쿼리는 생성 품질이 떨어집니다 | Metric View나 사전 정의된 Gold 테이블을 활용합니다 |
| **비정형 데이터** | JSON, 이미지, 텍스트 분석은 지원하지 않습니다 | 별도 전처리 후 정형 테이블로 만들어 연결합니다 |
| **복잡한 윈도우 함수** | 중첩 윈도우 함수나 복잡한 순위 계산은 오류가 발생할 수 있습니다 | Certified Answers로 정답 SQL을 사전 등록합니다 |
| **재귀 쿼리 (CTE)** | 계층 구조(조직도, BOM) 탐색용 재귀 쿼리 생성이 어렵습니다 | 전용 뷰(View)를 미리 만들어 단순화합니다 |
| **실시간 데이터** | Streaming 데이터에 대한 실시간 질의는 지원하지 않습니다 | Materialized View로 주기적으로 집계하여 사용합니다 |
| **크로스 카탈로그 쿼리** | 여러 카탈로그에 걸친 쿼리 생성은 제한적입니다 | 단일 카탈로그 내에서 데이터를 통합합니다 |

### 향후 발전 방향

Databricks는 AI/BI의 다음과 같은 개선을 지속적으로 진행하고 있습니다:

- **멀티턴 대화 개선**: 이전 대화 컨텍스트를 더 오래, 더 정확하게 기억하는 능력 강화
- **자동 메타데이터 보강**: AI가 테이블을 분석하여 COMMENT를 자동 추천하는 기능
- **Agent 통합**: Mosaic AI Agent와 연동하여 단순 SQL 쿼리를 넘어 다단계 분석 워크플로우 실행
- **예측 분석**: 과거 데이터 기반 트렌드 예측 및 이상 탐지를 자연어로 요청 가능
- **더 넓은 데이터 소스**: 외부 DB(PostgreSQL, MySQL 등) 연결 시에도 동일한 NL→SQL 품질 제공

---

## 참고 링크

- [Databricks Docs: AI/BI Overview](https://docs.databricks.com/aws/en/aibi/)
- [Databricks Docs: Genie — AI/BI Conversations](https://docs.databricks.com/aws/en/genie/)
- [Databricks Docs: Certified Answers (Instructions)](https://docs.databricks.com/aws/en/genie/instructions.html)
- [Databricks Docs: Photon Engine](https://docs.databricks.com/aws/en/compute/photon.html)
- [Databricks Docs: Unity Catalog Row Filters and Column Masks](https://docs.databricks.com/aws/en/data-governance/unity-catalog/row-and-column-filters.html)
- [Databricks Docs: Metric Views](https://docs.databricks.com/aws/en/dashboards/metric-views.html)
- [Databricks Docs: Table & Column Comments](https://docs.databricks.com/aws/en/data-governance/unity-catalog/best-practices.html)
- [Compound AI Systems — The Shift from Models to Systems (Berkeley Blog)](https://bair.berkeley.edu/blog/2024/02/18/compound-ai-systems/)
