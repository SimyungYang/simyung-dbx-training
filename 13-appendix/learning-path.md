# 학습 로드맵

## 역할별 추천 학습 경로

### 데이터 엔지니어 (Data Engineer)

```mermaid
graph LR
    A["00 선행 지식"] --> B["01 데이터 기초"]
    B --> C["02 Databricks 개요"]
    C --> D["03 레이크하우스"]
    D --> E["04 컴퓨트"]
    E --> F["05 데이터 엔지니어링<br/>⭐ 핵심"]
    F --> G["06 웨어하우징"]
    G --> H["07 Unity Catalog"]
```

**필수**: 00 → 01 → 02 → 03 → 04 → 05 (전체) → 07
**권장**: 06, 12

---

### 데이터 분석가 (Data Analyst)

```mermaid
graph LR
    A["00 선행 지식<br/>(RDB, 스키마)"] --> B["01 데이터 기초"]
    B --> C["02 Databricks 개요"]
    C --> D["06 웨어하우징<br/>⭐ 핵심"]
    D --> E["08 AI/BI<br/>⭐ 핵심"]
    E --> F["07 Unity Catalog"]
```

**필수**: 00(RDB, 스키마) → 01 → 02 → 06 → 08
**권장**: 03(Medallion), 07

---

### 데이터 과학자 (Data Scientist)

```mermaid
graph LR
    A["01 데이터 기초"] --> B["02 Databricks 개요"]
    B --> C["04 컴퓨트"]
    C --> D["09 머신러닝<br/>⭐ 핵심"]
    D --> E["10 에이전트 개발"]
```

**필수**: 01 → 02 → 04 → 09 (전체)
**권장**: 03, 10

---

### 플랫폼 관리자 (Admin)

```mermaid
graph LR
    A["02 Databricks 개요"] --> B["04 컴퓨트"]
    B --> C["07 Unity Catalog<br/>⭐ 핵심"]
    C --> D["12 보안/거버넌스<br/>⭐ 핵심"]
```

**필수**: 02 → 04 → 07 → 12
**권장**: 03, 06

---

### AI/ML 엔지니어

**필수**: 01 → 02 → 04 → 09 → 10 (전체)
**권장**: 03, 07, 11

---

## 전체 학습 순서 (처음부터 끝까지)

모든 내용을 체계적으로 학습하고 싶다면, **00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12 → 13** 순서대로 진행하시면 됩니다.

---

## 외부 학습 리소스

| 리소스 | URL | 설명 |
|--------|-----|------|
| **Databricks Academy** | academy.databricks.com | 공식 교육 과정 (무료/유료) |
| **Databricks Certifications** | databricks.com/learn/certification | 공식 자격증 |
| **Databricks Community** | community.databricks.com | 커뮤니티 포럼 |
| **Databricks Blog** | databricks.com/blog | 최신 기술 블로그 |
| **Delta Lake Docs** | docs.delta.io | Delta Lake 오픈소스 문서 |
| **MLflow Docs** | mlflow.org/docs | MLflow 오픈소스 문서 |
