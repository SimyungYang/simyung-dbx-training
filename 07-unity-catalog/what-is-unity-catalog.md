# Unity Catalog란?

## 개념

> 💡 **Unity Catalog**는 Databricks의 **통합 데이터 거버넌스 솔루션**입니다. 테이블, 파일, ML 모델, AI 에이전트 등 모든 데이터 자산을 하나의 카탈로그에서 관리하며, 접근 권한, 감사, 리니지를 통합적으로 제공합니다.

> 💡 **데이터 거버넌스(Data Governance)란?** 조직의 데이터가 안전하게 관리되고, 적절한 사람만 접근할 수 있으며, 규정을 준수하도록 하는 정책과 프로세스의 총체입니다. "누가, 어떤 데이터를, 언제, 왜 사용했는지"를 추적하고 통제하는 것이 핵심입니다.

---

## 3-Level 네임스페이스

```
Catalog (카탈로그)
  └── Schema (스키마)
       ├── Table (테이블)
       ├── View (뷰)
       ├── Volume (볼륨 - 파일)
       ├── Function (함수)
       └── Model (ML 모델)
```

```sql
-- 완전한 이름: catalog.schema.table
SELECT * FROM production.ecommerce.orders;
```

| 레벨 | 역할 | 예시 |
|------|------|------|
| **Catalog** | 최상위 컨테이너. 환경별, 팀별로 분리합니다 | `production`, `development`, `staging` |
| **Schema** | 관련 객체를 논리적으로 그룹화합니다 | `ecommerce`, `hr`, `finance` |
| **Object** | 실제 데이터 자산 (테이블, 뷰, 볼륨 등) | `orders`, `customers` |

---

## Unity Catalog의 핵심 가치

| 가치 | 설명 |
|------|------|
| **통합 관리** | 테이블, 파일, ML 모델, 함수를 하나의 시스템에서 관리합니다 |
| **세밀한 접근 제어** | 테이블, 행, 열 수준까지 권한을 관리합니다 |
| **감사 (Audit)** | 누가, 언제, 어떤 데이터에 접근했는지 추적합니다 |
| **리니지 (Lineage)** | 데이터가 어디서 왔고, 어디로 가는지 시각적으로 추적합니다 |
| **크로스 워크스페이스** | 여러 Workspace에서 동일한 카탈로그를 공유합니다 |

> 🆕 **ABAC (Attribute-Based Access Control)**: 태그 기반으로 접근 정책을 정의할 수 있는 기능이 Preview로 출시되었습니다. 예를 들어, "PII" 태그가 붙은 모든 컬럼에 마스킹을 자동 적용할 수 있습니다.

---

## 참고 링크

- [Databricks: Unity Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)
