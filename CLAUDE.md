# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

Databricks Data Intelligence Platform의 핵심 기능과 아키텍처를 체계적으로 다루는 한국어 교육 자료입니다.
GitBook으로 배포: https://simyungyang.gitbook.io/databricks-training/
GitHub: https://github.com/SimyungYang/simyung-dbx-training

## 관련 프로젝트

- **Databricks Enablements**(실전 가이드 & 핸즈온): https://simyungyang.gitbook.io/databricks-enablement-resources/
  - GitHub: https://github.com/SimyungYang/databricks-enablement-blog
  - 역할 분리: Training = 전체 문서(교육) / Enablements = 시나리오별 가이드 & 핸즈온

## 콘텐츠 품질 기준 (최우선 원칙)

모든 문서는 **Databricks Academy 수준** 으로 작성한다. 표면적 소개나 개요 수준에서 멈추지 않는다.

### 각 기술/개념에 반드시 포함할 내용:

1. **왜 등장했는가**— 이전 기술의 한계, 해결하려는 문제
2. **어떻게 작동하는가**— 핵심 메커니즘, 아키텍처, 동작 원리
3. **실전에서 어떻게 활용되는가**— 구체적 사용 사례, 구성 예시, 코드
4. **장단점과 트레이드오프**— 알려진 제약, 대안, 비용, 성능 비교
5. **주의사항과 베스트 프랙티스**— 흔한 실수, 권장 설정, 모범 사례

### 잘못된 예시 vs 올바른 예시:

❌ "Instance Pool은 유휴 인스턴스를 미리 확보하여 클러스터 시작 시간을 단축합니다."
✅ Instance Pool에 대해:
- **왜 필요한가**: 클러스터 시작 시 EC2/VM 프로비저닝에 2-5분 소요 → 인터랙티브 작업에 병목
- **어떻게 작동하는가**: 미리 VM을 확보(warm pool)하여 대기 → 요청 시 즉시 할당 (시작 시간 30초 이내)
- **장점**: 클러스터 시작 시간 80% 단축, 동일 인스턴스 재사용으로 EBS 캐시 유지
- **단점/트레이드오프**: 유휴 인스턴스 비용 발생 (EC2 비용은 계속 과금), Pool 크기 설계 필요
- **언제 사용**: 인터랙티브 개발, Auto-scaling이 빈번한 Job, 여러 클러스터가 같은 사양을 공유할 때
- **언제 사용하지 않을 때**: 서버리스 컴퓨트 사용 시 (Pool 불필요), 가끔 실행하는 배치 작업

## 보강이 필요한 페이지 (TODO)

### Score 1-2 (가장 시급):
- `04-compute-and-workspace/cluster-policies/permissions-and-design.md` (38줄)
- `14-best-practices/cost-optimization/cost-structure.md` (41줄)
- `08-ai-bi/aibi-architecture.md` (53줄)
- `02-databricks-overview/databricks-competitive.md` (64줄)
- `07-unity-catalog/data-lineage/basics-and-visualization.md` (123줄)
- `10-agent-development/agent-deployment/versioning-and-monitoring.md` (128줄)
- `09-machine-learning/mlflow/model-registry/basics-and-registration.md` (153줄)
- `06-data-warehousing/pivot-unpivot/pivot-basics.md` (155줄)
- `05-data-engineering/auto-loader/auto-loader-hands-on/` (4개 파일)
- `05-data-engineering/lakeflow-jobs/job-configuration/tasks-and-dependencies.md` (163줄)
- `11-lakebase/lakebase-with-apps/optimization-and-crud.md` (169줄)
- `09-machine-learning/mlflow/mlflow-tracing/analysis-and-evaluation.md` (169줄)
- `12-security-and-governance/system-tables/overview-and-tables.md` (211줄)
- `13-appendix/glossary.md` (240줄)
- `13-appendix/databricks-asset-bundles.md` (274줄)
- `00-prerequisites/bigdata-ecosystem.md` (327줄)
- `09-machine-learning/model-serving/custom-model-deployment.md` (416줄)
- `09-machine-learning/model-serving/endpoint-monitoring.md` (463줄)

### Score 3 (보강 필요):
- `08-ai-bi/hybrid-bi-strategy.md` (64줄)
- `12-security-and-governance/system-tables/dashboard-and-setup.md` (65줄)
- `02-databricks-overview/databricks-pricing.md` (70줄)
- `11-lakebase/lakebase-with-apps/deployment.md` (72줄)
- `09-machine-learning/mlflow/model-registry/permissions-and-serving.md` (91줄)
- `08-ai-bi/aibi-overview.md` (99줄)

### 전체적으로 부족한 패턴:
- 대부분 문서에서 "장단점/트레이드오프" 섹션이 없음
- "언제 사용하고, 언제 사용하지 않을까" 가이드가 부족
- 비용 관련 정보 (DBU 소모량, 가격 비교)가 부족
- 실무 시나리오 기반 설명이 부족 (이론만 있고 "그래서 어떻게 쓰나?"가 없음)
- 다른 기술/서비스와의 비교가 부족 (예: Instance Pool vs Serverless, Delta Sync vs Direct Access)

## Writing Style

- **언어**: 한국어 (기술 용어는 영문 병기, 예: "데이터 레이크하우스(Data Lakehouse)")
- **어투**: 합니다체 ("~합니다", "~됩니다", "~할 수 있습니다")
- **구조**: 왜 필요한가 → 핵심 개념 → 동작 원리 → 장단점/비교 → 실습/코드 → 베스트 프랙티스 → 정리
- **시각 자료**: Databricks 공식 문서 이미지 우선 활용, 없으면 테이블로 표현 (Mermaid/ASCII art 사용 금지)
- **한국어 볼드 렌더링**: `** 볼드텍스트**` 뒤에 한글이 바로 오면 반드시 스페이스 추가. `**` 바로 안쪽에는 스페이스 넣지 않음.

## File Organization

- 번호 prefix 폴더로 학습 순서 (예: `01-data-fundamentals/`)
- 파일명 영문 kebab-case (예: `medallion-architecture.md`)
- 200줄 이상이면 서브페이지로 분리
- GitBook SUMMARY.md로 네비게이션 관리

## Source References

- **공식 문서**: https://docs.databricks.com/aws/en/
- **Azure**: https://learn.microsoft.com/en-us/azure/databricks/
- **릴리즈 노트**: https://docs.databricks.com/aws/en/release-notes/
- **블로그**: https://www.databricks.com/blog
