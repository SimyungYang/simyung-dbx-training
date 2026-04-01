# 데이터 품질 관리와 CI/CD

## 5. 데이터 품질 관리

### 5.1 SDP Expectations로 파이프라인 품질 보장

```python
import dlt

@dlt.table(comment="품질 검증이 적용된 실버 주문 테이블")
@dlt.expect("valid_order_id", "order_id IS NOT NULL")
@dlt.expect("valid_amount", "amount > 0 AND amount < 1000000")
@dlt.expect_or_drop("valid_customer", "customer_id IS NOT NULL")
@dlt.expect_or_fail("valid_date", "order_date <= current_date()")
def silver_orders():
    return dlt.read_stream("bronze_orders")
```

| Expectation 유형 | 위반 시 동작 | 사용 시나리오 |
|-----------------|-----------|-------------|
| `@dlt.expect` | 경고 기록, 데이터 유지 | 데이터 품질 모니터링 (소프트 체크) |
| `@dlt.expect_or_drop` | 위반 행 제거 | 잘못된 데이터 필터링 (프로덕션 권장) |
| `@dlt.expect_or_fail` | 파이프라인 중단 | 치명적 오류 (날짜 미래값 등) |

### 5.2 Lakehouse Monitoring으로 드리프트 감지

```sql
-- Lakehouse Monitor 생성 (시계열 프로파일)
CREATE OR REPLACE MONITOR prod_sales.gold.daily_revenue
USING TIME SERIES
  TIMESTAMP_COL event_date
  GRANULARITIES ("1 day", "1 week")
WITH (
  LINKED_ENTITIES = ("prod_sales.gold.daily_revenue_monitor")
);
```

| 모니터링 유형 | 감지 대상 | 적용 테이블 |
|-------------|----------|-----------|
| **Snapshot** | 시점 기준 통계 분포 | 디멘션 테이블, 룩업 테이블 |
| **Time Series** | 시간에 따른 분포 변화 | 팩트 테이블, 이벤트 테이블 |
| **Inference** | ML 모델 입력/출력 드리프트 | ML 추론 결과 테이블 |

모니터링이 생성하는 핵심 지표:

| 지표 | 설명 | 활용 |
|------|------|------|
| **Null 비율** | 컬럼별 NULL 값 비율 | 급증 시 데이터 소스 문제 의심 |
| ** 고유값 수** | 컬럼별 Distinct 값 수 | 급변 시 데이터 스키마 변경 의심 |
| ** 분포 변화** | KL Divergence, JS Distance | 데이터 드리프트 감지 |
| ** 이상치** | Z-score 기반 탐지 | 비정상 데이터 유입 감지 |

### 5.3 데이터 품질 SLA 정의

| SLA 항목 | 골드 등급 | 실버 등급 | 브론즈 등급 |
|---------|---------|---------|----------|
| ** 완전성 (Completeness)** | NULL < 0.1% | NULL < 1% | NULL < 5% |
| ** 정확성 (Accuracy)** | 오류율 < 0.01% | 오류율 < 0.1% | 오류율 < 1% |
| ** 적시성 (Timeliness)** | 15분 이내 갱신 | 1시간 이내 | 24시간 이내 |
| ** 일관성 (Consistency)** | 참조 무결성 100% | 참조 무결성 99.9% | 참조 무결성 99% |
| ** 고유성 (Uniqueness)** | 중복 0% | 중복 < 0.01% | 중복 < 0.1% |

---

## 6. CI/CD 및 자동화

### 6.1 Databricks Asset Bundles + GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy Databricks Assets

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main

      - name: Validate bundle
        run: databricks bundle validate
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_SP_TOKEN }}

  deploy-staging:
    needs: validate
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main

      - name: Deploy to staging
        run: databricks bundle deploy --target staging
        env:
          DATABRICKS_HOST: ${{ secrets.STAGING_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.STAGING_SP_TOKEN }}

  deploy-production:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main

      - name: Deploy to production
        run: databricks bundle deploy --target production
        env:
          DATABRICKS_HOST: ${{ secrets.PROD_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.PROD_SP_TOKEN }}
```

### 6.2 환경별 프로모션 (Dev → Staging → Prod)

```yaml
# databricks.yml (Databricks Asset Bundle 설정)
bundle:
  name: sales-data-pipeline

targets:
  dev:
    workspace:
      host: https://dev-workspace.cloud.databricks.com
    default: true
    variables:
      catalog: dev_sales
      warehouse_size: Small

  staging:
    workspace:
      host: https://staging-workspace.cloud.databricks.com
    variables:
      catalog: stg_sales
      warehouse_size: Medium

  production:
    workspace:
      host: https://prod-workspace.cloud.databricks.com
    run_as:
      service_principal_name: svc-prod-etl
    variables:
      catalog: prod_sales
      warehouse_size: Large
    permissions:
      - level: CAN_MANAGE
        group_name: platform-admins
      - level: CAN_VIEW
        group_name: data-engineers
```

| 환경 | 배포 트리거 | 승인 필요 | 실행 계정 |
|------|-----------|----------|----------|
| **Dev** | 로컬 `databricks bundle deploy` | 없음 | 개발자 본인 |
| **Staging** | PR 생성 시 자동 | PR 리뷰 승인 | CI/CD Service Principal |
| **Production** | main 브랜치 merge 시 | 2명 이상 승인 | 프로덕션 Service Principal |

### 6.3 코드 리뷰 프로세스

| 리뷰 항목 | 체크 포인트 | 자동화 가능 여부 |
|----------|-----------|---------------|
| **SQL 문법** | 구문 오류, 성능 안티패턴 | ✅ sqlfluff, sqlglot |
| ** 데이터 품질** | Expectation 정의 여부 | ✅ DLT Expectation 검사 |
| ** 보안** | 하드코딩된 비밀번호, PII 노출 | ✅ Secret 스캐너 |
| ** 비용 영향** | 대용량 스캔, 비효율 조인 | ⚠️ 부분 자동화 |
| ** 문서화** | 테이블/컬럼 코멘트 | ✅ 코멘트 존재 여부 검사 |
| ** 테스트** | 단위 테스트 커버리지 | ✅ pytest 연동 |

---

## 7. 거버넌스 성숙도 모델

조직의 데이터 거버넌스 수준을 5단계로 평가하여 개선 로드맵을 수립합니다.

### 5단계 성숙도 평가 프레임워크

| 단계 | 수준 | 접근 제어 | 데이터 품질 | 모니터링 | 자동화 |
|------|------|----------|-----------|---------|-------|
| **1. Initial** | 임의적 | 개인별 직접 권한 | 수동 검증 | 없음 | 없음 |
| **2. Managed** | 기본 | 그룹 기반 권한 | 기본 Expectation | 비용 추적 | 수동 배포 |
| **3. Defined** | 표준화 | Row/Column 필터 | SLA 정의 + 모니터링 | 감사 로그 분석 | CI/CD 구축 |
| **4. Measured** | 정량적 | ABAC 적용 | 자동 드리프트 감지 | 실시간 알림 | 완전 자동화 |
| **5. Optimizing** | 최적화 | Zero Trust | AI 기반 품질 예측 | 예측적 모니터링 | 셀프서비스 |

### 단계별 행동 계획

**1단계 → 2단계 (기초 구축, 1~2개월)**

```
□ Unity Catalog 활성화 및 Metastore 생성
□ IdP 연동 및 SCIM 자동 프로비저닝
□ 역할 기반 그룹 설계 및 생성
□ 프로덕션 워크로드를 Service Principal으로 전환
□ 기본 비용 모니터링 대시보드 구축
```

**2단계 → 3단계 (표준화, 2~3개월)**

```
□ 데이터 분류 체계 수립 및 태깅
□ Row Filter / Column Mask 적용 (PII 테이블)
□ SDP Expectations으로 파이프라인 품질 검증
□ 감사 로그 모니터링 대시보드 구축
□ Databricks Asset Bundles + CI/CD 파이프라인 구축
□ 코드 리뷰 프로세스 정립
```

**3단계 → 4단계 (자동화, 3~6개월)**

```
□ Lakehouse Monitoring으로 자동 드리프트 감지
□ 비용 이상 감지 자동 알림 설정
□ PII 자동 태깅 파이프라인 구축
□ 데이터 품질 SLA 대시보드 운영
□ Predictive Optimization 전면 활성화
□ 규제 준수 자동 보고서 생성
```

**4단계 → 5단계 (최적화, 6개월+)**

```
□ 셀프서비스 데이터 플랫폼 구축
□ 데이터 메시 패턴 적용 (도메인별 소유권)
□ AI 기반 데이터 품질 이상 예측
□ 실시간 컴플라이언스 모니터링
□ 데이터 카탈로그 + Genie 기반 자연어 접근
```

### 성숙도 자가 진단 질문

다음 질문에 답하여 현재 성숙도를 평가해 보세요.

| 질문 | 예 | 아니오 |
|------|---|-------|
| Unity Catalog이 활성화되어 있습니까? | 2단계+ | 1단계 |
| 모든 권한이 그룹을 통해 관리됩니까? | 2단계+ | 1단계 |
| PII 데이터에 Row/Column 필터가 적용되어 있습니까? | 3단계+ | 2단계 |
| 데이터 품질 SLA가 정의되어 있습니까? | 3단계+ | 2단계 |
| CI/CD로 자동 배포되고 있습니까? | 3단계+ | 2단계 |
| 데이터 드리프트가 자동 감지됩니까? | 4단계+ | 3단계 |
| 비용 이상이 자동 알림됩니까? | 4단계+ | 3단계 |
| 셀프서비스 데이터 접근이 가능합니까? | 5단계 | 4단계 |

---

## 참고 링크

- [Unity Catalog 모범 사례](https://docs.databricks.com/aws/en/data-governance/unity-catalog/best-practices)
- [인증 및 접근 제어](https://docs.databricks.com/aws/en/security/auth/index)
- [Row Filter / Column Mask](https://docs.databricks.com/aws/en/tables/row-and-column-filters)
- [감사 로그 시스템 테이블](https://docs.databricks.com/aws/en/admin/system-tables/audit)
- [Lakehouse Monitoring](https://docs.databricks.com/aws/en/lakehouse-monitoring/index)
- [Databricks Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/index)
- [네트워크 보안](https://docs.databricks.com/aws/en/security/network/index)
- [Predictive Optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
