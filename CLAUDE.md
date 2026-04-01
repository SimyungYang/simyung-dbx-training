# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

이 레포지토리는 **데이터 관련 지식이 전혀 없는 사람**을 대상으로 한 Databricks 종합 교육 자료 모음입니다.
고객(비개발자 포함)이 처음부터 단계적으로 학습할 수 있도록 한국어 Markdown 파일로 작성합니다.
최종 산출물은 **GitBook 또는 웹 호스팅(MkDocs, Docusaurus 등)**형태로 배포 가능하도록 구성합니다.

## Target Audience

- 데이터 엔지니어링, 데이터 웨어하우스, ETL 등의 개념을 처음 접하는 사람
- Databricks 플랫폼을 처음 사용하는 고객, SA, 파트너

## Covered Topics

아래 주제들을 초보자도 이해할 수 있도록 자세히 다룹니다:

0. **Prerequisites**— RDB 기초, 스키마 설계(Star/Snowflake), 빅데이터 역사(Hadoop→Spark→Lakehouse), 생태계, 실시간 처리 기술
1. **Data Fundamentals**— 데이터 엔지니어링 기초, 데이터 웨어하우스 vs 데이터 레이크 개념
2. **Databricks Overview**— 플랫폼 소개, 아키텍처, Workspace/Notebook 기초
3. **Lakehouse Architecture**— Delta Lake, Lakehouse 패러다임, Medallion 아키텍처
4. **Compute & Workspace**— Spark 기초, 클러스터, SQL Warehouse, Serverless
5. **Data Engineering**— Auto Loader, SDP(선언적 파이프라인), Lakeflow Connect/Jobs
6. **Data Warehousing**— Databricks SQL, AI 함수, 쿼리 최적화
7. **Unity Catalog**— 데이터 거버넌스, 권한 관리, 리니지, Delta Sharing
8. **AI/BI**— Lakeview 대시보드, Genie, 알림
9. **Machine Learning**— MLflow, Model Serving, Feature Engineering
10. **Agent Development**— AI 에이전트, Vector Search, RAG, Agent Evaluation
11. **Lakebase**— 관리형 PostgreSQL OLTP, Delta Lake 동기화
12. **Security & Governance**— 인증, 네트워크 보안, 시스템 테이블
13. **Appendix**— CLI, Asset Bundles, Apps, 용어 사전, 학습 로드맵

## Content Guidelines

### Writing Style
- ** 언어**: 한국어 (기술 용어는 영문 병기, 예: "데이터 레이크하우스(Data Lakehouse)")
- **어투**: 정중하고 친절한 존댓말(합니다체)을 사용하며, 전문가가 설명하는 듯한 신뢰감 있는 톤 유지
  - ❌ "~이다", "~임", "~함" (단답형/체언종결 금지)
  - ✅ "~합니다", "~됩니다", "~할 수 있습니다", "~살펴보겠습니다"
  - 예시: "이 섹션에서는 Delta Lake의 핵심 개념을 살펴보겠습니다. ACID 트랜잭션이 왜 중요한지 이해하고 나면, 데이터 파이프라인을 훨씬 안정적으로 설계하실 수 있습니다."
- **톤**: 친절하고 쉬운 설명, 비유와 실생활 예시 적극 활용. 독자를 전문가로 성장시키는 멘토의 관점
- **구조**: 각 문서는 "왜 필요한가 → 핵심 개념 → 동작 원리 → 실습 예제 → 정리" 흐름으로 구성
- **시각 자료**:
  - **공식 문서 이미지 우선 활용**: Databricks 공식 문서(docs.databricks.com)에 아키텍처 다이어그램, 스크린샷 등이 있으면 해당 이미지를 직접 활용합니다. 아키텍처를 따로 그리지 않습니다.
  - 이미지 삽입 형식: `![설명](이미지URL)` 으로 공식 문서의 이미지를 임베딩하되, 아래에 원본 페이지 링크를 명시합니다.
  - 공식 이미지가 없는 경우에만 Mermaid 다이어그램으로 보충합니다.
  - 표, 코드 블록도 적극 활용합니다.

### Industry Terminology (산업 표준 용어)
- Databricks 고유 용어가 아닌 **산업 표준 용어**(예: ACID, CDC, SCD, OLTP, OLAP, MPP, Columnar Storage, Schema-on-Read/Write, Data Mesh, Data Fabric 등)가 등장하면 반드시 별도의 상세 설명을 함께 제공
- 용어 설명 방식:
  - **인라인 설명**: 본문에서 처음 등장할 때 괄호 또는 박스(`> 💡`)로 쉬운 설명 제공
  - **용어 사전 링크**: `13-appendix/glossary.md`의 해당 항목으로 링크하여 더 자세한 설명 참조 가능
- 산업 표준 용어는 "왜 이런 개념이 생겼는지(배경)"까지 설명하여 초보자도 맥락을 이해할 수 있도록 함

### Source References
- **공식 문서(1차 출처)**:
  - AWS: https://docs.databricks.com/aws/en/
  - Azure: https://learn.microsoft.com/en-us/azure/databricks/
- **릴리즈 노트**: https://docs.databricks.com/aws/en/release-notes/
  - 최신 GA/Public Preview/Private Preview 기능 변경사항을 적극 반영
  - 각 주제 문서에서 관련 신규 기능이 있으면 `> 🆕 **[2025.xx 릴리즈]**` 박스로 하이라이트
  - 릴리즈 노트에 언급된 구현 레벨 가이드(마이그레이션 방법, 설정 변경 등)도 실습 예제에 포함
- **공식 블로그**: https://www.databricks.com/blog
  - 신규 피처 소개, 아키텍처 딥다이브, 베스트 프랙티스 글을 참고하여 실무 관점의 내용 보강
  - 블로그에서 제시하는 벤치마크, 성능 비교, 고객 사례 등을 적절히 인용
  - 특히 Data+AI Summit 발표 자료, 기술 블로그의 구현 예제를 교육 자료에 반영
- 각 문서 하단에 **참고 링크 섹션**을 두어 공식 문서, 릴리즈 노트, 블로그 원문을 명시

### Incorporating New Features & Practical Implementation
- 각 주제별 문서 작성 시 해당 영역의 **최신 릴리즈 노트와 블로그 글을 반드시 조회**하여 반영
- 신규 기능은 단순 소개에 그치지 않고, **왜 도입되었는지(배경) → 어떻게 쓰는지(구현) → 기존 방식 대비 장점** 흐름으로 설명
- 코드 예제는 최신 API와 구문을 사용하며, deprecated된 방식은 마이그레이션 가이드와 함께 안내
- Preview 단계(Private/Public)의 기능은 `> ⚠️ **Preview 기능**` 박스로 명시하고, GA 시점 기준으로 안정성 안내 포함

### File Organization
- 각 주제별 번호 prefix 폴더로 학습 순서를 명확히 함 (예: `01-data-fundamentals/`)
- 문서 파일명은 영문 kebab-case 사용 (예: `medallion-architecture.md`)
- 각 섹션 폴더에 `README.md`를 두어 해당 섹션의 학습 목표와 문서 목록을 안내
- 웹 호스팅 도구(GitBook/MkDocs/Docusaurus)의 네비게이션 설정 파일 포함

### Quality Standards
- 전문 용어는 처음 등장 시 반드시 쉬운 말로 풀어서 설명
- 코드 예제는 복사-붙여넣기로 바로 실행 가능하도록 작성
- 각 문서는 독립적으로도 읽을 수 있되, 순서대로 읽으면 체계적으로 학습 가능하도록 구성

## Web Hosting

이 교육 자료는 웹에서 GitBook 스타일로 호스팅할 수 있도록 구성합니다:
- **MkDocs (Material theme)**또는 **Docusaurus** 를 기본 호스팅 도구로 사용
- `mkdocs.yml` 또는 `docusaurus.config.js`에 네비게이션, 테마, 검색 설정 포함
- GitHub Pages 또는 Vercel/Netlify로 배포 가능
