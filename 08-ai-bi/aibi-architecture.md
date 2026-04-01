# AI/BI 기술 아키텍처 심화

## Compound AI System 아키텍처

Databricks AI/BI는 단일 LLM이 아니라, 여러 AI 컴포넌트가 협력하는 **Compound AI System(복합 AI 시스템)** 으로 구축되어 있습니다.

| 컴포넌트 | 역할 | 상세 설명 |
|----------|------|-----------|
| **NLU (Natural Language Understanding)**| 질문 의도 파악 | 사용자의 자연어 질문에서 의도(intent), 엔터티(entity), 필터 조건을 추출합니다 |
| **Schema Understanding**| 테이블 구조 이해 | Unity Catalog의 메타데이터(테이블명, 컬럼명, COMMENT, 데이터 타입)를 활용하여 적절한 테이블/컬럼을 매칭합니다 |
| **SQL Generator**| SQL 생성 | 파악된 의도를 실행 가능한 SQL로 변환합니다. Databricks SQL 방언에 최적화되어 있습니다 |
| **SQL Validator**| SQL 검증 | 생성된 SQL의 문법 오류, 권한 문제, 성능 이슈를 사전 검증합니다 |
| **Result Formatter**| 결과 포맷팅 | 쿼리 결과를 자연어 답변, 표, 차트로 포맷팅합니다 |
| **Feedback Loop**| 학습 개선 | 사용자 피드백(좋아요/싫어요), 인증된 답변을 반영하여 응답 품질을 개선합니다 |

---

## 자연어 → SQL 변환 파이프라인 상세

Genie가 사용자의 질문을 SQL로 변환하는 과정을 단계별로 살펴보겠습니다.

**사용자**: "지난 달 서울 매출 상위 5개 제품을 알려줘"

| 단계 | 처리 | 상세 |
|------|------|------|
| **1단계: NLU (의도 분석)**| 자연어 → 구조화 | 시간 필터: "지난 달" → DATEADD, 공간 필터: "서울" → region = '서울', 측정값: "매출" → SUM(order_amount), 차원: "제품" → product_name, 수량 제한: "상위 5개" → LIMIT 5 |
| **2단계: Schema Matching**| 테이블 매핑 | 테이블: gold_orders, 컬럼 매핑: 매출 → order_amount, 제품 → product_name, Metric View 존재 시 정의를 우선 사용 |
| **3단계: SQL Generation**| SQL 생성 | 안전한 SELECT 문만 생성 (INSERT/UPDATE/DELETE 불가) |
| **4단계: Execution**| SQL 실행 | SQL Warehouse에서 실행, 결과를 자연어와 차트로 변환 |


---

## Photon 엔진의 역할

AI/BI의 쿼리는 **Serverless SQL Warehouse**에서 실행되며, 내부적으로 **Photon 엔진** 이 핵심 역할을 합니다.

> 💡 **Photon**은 Databricks가 C++로 작성한 **네이티브 벡터화 쿼리 엔진** 입니다. JVM 기반 Spark SQL 엔진을 대체하여 BI 쿼리에서 2~8배의 성능 향상을 제공합니다.

| Photon 최적화 | 설명 | AI/BI 연관성 |
|-------------|------|-------------|
| **벡터화 실행**| SIMD 명령어로 한 번에 여러 행을 처리합니다 | 대시보드의 집계 쿼리가 빠르게 실행됩니다 |
| **코드 생성 (Codegen)**| 쿼리를 네이티브 코드로 컴파일하여 실행합니다 | 반복 실행되는 대시보드 쿼리에 효과적입니다 |
| **메모리 관리**| Off-heap 메모리로 GC 오버헤드를 제거합니다 | 동시 다수 쿼리(대시보드 로딩)에 안정적입니다 |
| **인텔리전트 캐싱** | Disk I/O 결과를 메모리에 캐시합니다 | 대시보드 새로고침 시 빠른 응답을 제공합니다 |

---

## 참고 링크

- [Databricks: AI/BI Architecture](https://docs.databricks.com/aws/en/aibi/)
- [Databricks: Photon Engine](https://docs.databricks.com/aws/en/compute/photon.html)
