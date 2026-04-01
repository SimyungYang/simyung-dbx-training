# 플랫폼 설계

## 이 문서에서 다루는 내용

대규모 조직에서 Databricks 플랫폼을 안전하고 체계적으로 운영하기 위한 거버넌스 전략을 다룹니다. 플랫폼 아키텍처 설계부터 접근 제어, 데이터 분류, 감사/컴플라이언스, 데이터 품질 관리, CI/CD 자동화까지 엔터프라이즈 수준의 모범 사례를 제공합니다.

---

## 1. 플랫폼 설계

### 1.1 Workspace 분리 전략

Workspace를 어떻게 분리하느냐에 따라 보안, 비용 관리, 운영 효율성이 결정됩니다.

| 전략 | 구조 | 장점 | 단점 | 적합한 조직 |
|------|------|------|------|-----------|
| **환경별 분리** | Dev / Staging / Prod | 환경 간 완전 격리, 규제 충족 용이 | Workspace 수 증가 | 규제 산업 (금융, 의료) |
| ** 팀별 분리** | DE / DS / Analytics | 팀별 독립 운영, 비용 추적 용이 | 크로스팀 협업 어려움 | 팀 간 독립성 높은 조직 |
| ** 하이브리드** | 환경 × 팀 매트릭스 | 최대 유연성 | 관리 복잡도 높음 | 대기업 (1000+ 사용자) |
| ** 단일 Workspace** | 하나의 Workspace | 관리 단순, Unity Catalog로 격리 | 규모 확장 시 한계 | 스타트업, 소규모 팀 |

> 💡 ** 권장 패턴**: 대부분의 엔터프라이즈는 ** 환경별 분리 (Dev/Staging/Prod)** 를 기본으로 하되, Unity Catalog의 카탈로그 수준에서 팀/프로젝트를 분리하는 방식이 효과적입니다.

| 수준 | 구성 |
|------|------|
| **Unity Catalog Metastore** | Account 수준 - 모든 Workspace에서 공유 |
| Dev Workspace | 개발팀 자유 실험, 느슨한 정책 |
| Staging Workspace | 통합 테스트, 프로덕션과 유사한 정책 |
| Prod Workspace | 엄격한 접근 제어, 감사 로그 필수 |

### 1.2 Unity Catalog 네임스페이스 설계

**3단계 네임스페이스**: `catalog.schema.table`

| 수준 | 명명 규칙 | 예시 | 용도 |
|------|----------|------|------|
| **Catalog** | `{환경}_{도메인}` 또는 `{환경}` | `prod_sales`, `dev_hr` | 환경 + 비즈니스 도메인 격리 |
| **Schema** | `{데이터 계층}` 또는 `{팀}` | `bronze`, `silver`, `gold` | Medallion 계층 또는 팀 분리 |
| **Table** | `{엔티티}_{설명}` | `orders_daily`, `customers_dim` | 비즈니스 엔티티 |

** 명명 규칙 표준안**

```sql
-- 카탈로그 명명 패턴
-- 패턴 1: 환경 중심 (소규모 조직)
-- prod / dev / staging

-- 패턴 2: 환경 + 도메인 (중대형 조직, 권장)
-- prod_sales / prod_marketing / dev_sales / dev_marketing

-- 패턴 3: 도메인 중심 (데이터 메시 구조)
-- sales / marketing / finance (환경은 Workspace로 분리)

-- 스키마 명명: Medallion 계층
CREATE SCHEMA prod_sales.bronze;    -- 원본 데이터
CREATE SCHEMA prod_sales.silver;    -- 정제된 데이터
CREATE SCHEMA prod_sales.gold;      -- 비즈니스 집계
CREATE SCHEMA prod_sales.sandbox;   -- 개발/실험용

-- 테이블 명명 규칙
-- {엔티티}_{유형}: orders_fact, customers_dim, daily_revenue_agg
-- 접미사: _fact (팩트), _dim (디멘션), _agg (집계), _raw (원본)
```

### 1.3 멀티 클라우드 / 멀티 리전 거버넌스

| 고려사항 | 전략 | 구현 방법 |
|---------|------|----------|
| ** 데이터 주권** | 리전별 Metastore 분리 | 리전별 Unity Catalog Metastore 생성 |
| ** 크로스 리전 공유** | Delta Sharing | 리전 간 데이터를 복제 없이 공유 |
| ** 통합 거버넌스** | Account 수준 관리 | Account Console에서 전체 Workspace 관리 |
| ** 재해 복구** | 멀티 리전 복제 | 핵심 데이터를 다른 리전에 비동기 복제 |

> ⚠️ ** 주의**: Unity Catalog의 리니지(Lineage)와 접근 제어는 ** 동일 Metastore 내에서만** 작동합니다. 크로스 Metastore 데이터 공유 시에는 Delta Sharing을 사용하며, 리니지 추적이 단절될 수 있습니다.

---
