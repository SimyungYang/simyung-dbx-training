# Databricks Asset Bundles (DABs)

## 개념

> 💡 **Databricks Asset Bundles**는 Databricks 리소스(Jobs, Pipelines, 대시보드 등)를 **코드로 정의하고 배포**하는 IaC(Infrastructure as Code) 도구입니다. YAML 파일로 리소스를 선언하고, CLI로 배포합니다.

> 🆕 최근 **Declarative Automation Bundles**로 명칭이 변경되었습니다.

---

## 프로젝트 구조

```
my-project/
├── databricks.yml          # 메인 설정 파일
├── resources/
│   ├── jobs.yml            # Job 정의
│   └── pipelines.yml       # Pipeline 정의
├── src/
│   ├── etl_notebook.py     # 소스 코드
│   └── transform.sql
└── environments/
    ├── dev.yml             # 개발 환경 설정
    └── prod.yml            # 프로덕션 환경 설정
```

---

## 기본 사용법

```bash
# 프로젝트 초기화
databricks bundle init

# 개발 환경에 배포
databricks bundle deploy -t dev

# 프로덕션에 배포
databricks bundle deploy -t prod

# Job 실행
databricks bundle run my_job -t dev
```

---

## 참고 링크

- [Databricks: Asset Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)
