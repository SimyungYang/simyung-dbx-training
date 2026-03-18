# SQL Warehouse

## SQL Warehouse란?

> 💡 **SQL Warehouse**는 SQL 쿼리를 실행하기 위한 **전용 컴퓨팅 리소스**입니다. 일반 Spark 클러스터와 달리 SQL 분석에 최적화되어 있으며, BI 도구 연결, 대시보드 조회 등에 적합합니다.

### 일반 클러스터와의 차이

| 비교 항목 | All-Purpose Cluster | SQL Warehouse |
|-----------|-------------------|---------------|
| **지원 언어** | Python, SQL, Scala, R | SQL 전용 |
| **주요 용도** | 개발, ETL, ML | SQL 분석, BI, 대시보드 |
| **최적화** | 범용 Spark | SQL 쿼리에 특화 (Photon 기본 포함) |
| **연결** | 노트북 | SQL Editor, BI 도구, JDBC/ODBC |
| **비용** | DBU/시간 | DBU/시간 (SQL 전용 요금) |

---

## SQL Warehouse 유형

| 유형 | 설명 | 적합한 사용 |
|------|------|------------|
| **Serverless** | Databricks가 자동 관리. 수 초 내 시작 | 대부분의 SQL 워크로드에 권장 |
| **Pro** | 고급 기능 포함 (ML 예측 함수 등) | 고급 SQL 기능 필요 시 |
| **Classic** | 기본 SQL 기능 | 레거시 호환 |

> 🆕 **Serverless SQL Warehouse**가 현재 권장되는 기본 선택입니다. 수 초 만에 시작되고, 사용하지 않을 때는 자동으로 리소스를 해제하여 비용을 절약합니다.

---

## 사이즈 선택

SQL Warehouse는 **T-Shirt 사이즈**로 크기를 선택합니다.

| 사이즈 | 클러스터 크기 | 적합한 워크로드 |
|--------|-------------|---------------|
| **2X-Small** | 1 클러스터 | 가벼운 개발, 테스트 |
| **X-Small** | 1 클러스터 | 소규모 쿼리 |
| **Small** | 2 클러스터 | 소규모 팀 분석 |
| **Medium** | 4 클러스터 | 중규모 워크로드 |
| **Large** | 8 클러스터 | 대규모 동시 쿼리 |
| **X-Large** | 16 클러스터 | 고성능 요구 |
| **2X-Large ~ 4X-Large** | 32~64 클러스터 | 엔터프라이즈급 |
| **5X-Large** | 128 클러스터 | 초대형 워크로드 |

> 🆕 **5X-Large SQL Warehouse**: 최근 베타로 출시된 가장 큰 사이즈 옵션입니다. 초대규모 동시 쿼리나 극히 복잡한 분석 워크로드에 적합합니다.

---

## 스케일링 설정

### Auto-Stop (자동 중지)

사용하지 않을 때 자동으로 중지하여 비용을 절약합니다.

```
Auto-Stop: 10분 (유휴 후 중지)
```

### Scaling (클러스터 수 조절)

동시 사용자가 많을 때 자동으로 클러스터 수를 늘립니다.

```
최소 클러스터: 1
최대 클러스터: 3
```

> 💡 여기서의 "클러스터"는 SQL Warehouse 내부의 컴퓨팅 단위입니다. 하나의 클러스터가 하나의 쿼리를 처리하므로, 클러스터 수 = 동시에 처리할 수 있는 무거운 쿼리 수로 이해하시면 됩니다. (가벼운 쿼리는 하나의 클러스터에서 여러 개를 동시에 처리할 수 있습니다.)

---

## BI 도구 연결

SQL Warehouse는 **JDBC/ODBC** 표준 프로토콜을 지원하여 다양한 BI 도구와 연결할 수 있습니다.

| BI 도구 | 연결 방식 |
|---------|-----------|
| **Tableau** | Databricks 전용 커넥터 또는 ODBC |
| **Power BI** | Databricks 전용 커넥터 |
| **Looker** | JDBC |
| **dbt** | Databricks 어댑터 |
| **Excel** | ODBC 또는 Databricks Excel Add-in |

> 🆕 **Databricks Excel Add-in**: 최근 Public Preview로 출시되어, Excel에서 직접 SQL을 실행하고 데이터를 가져올 수 있습니다.

### 연결 정보 확인

SQL Warehouse 상세 페이지에서 **Connection details** 탭을 클릭하면 연결에 필요한 정보를 확인할 수 있습니다.

- **Server Hostname**: 워크스페이스의 호스트명
- **HTTP Path**: SQL Warehouse의 고유 경로
- **Port**: 443 (HTTPS)

---

## 정리

| 핵심 개념 | 설명 |
|-----------|------|
| **SQL Warehouse** | SQL 분석에 최적화된 전용 컴퓨팅 리소스입니다 |
| **Serverless** | 자동 관리, 수 초 시작, 비용 효율적. 대부분의 경우에 권장됩니다 |
| **T-Shirt 사이즈** | 워크로드 규모에 따라 2X-Small ~ 5X-Large 중 선택합니다 |
| **Auto-Stop** | 유휴 시 자동 중지하여 비용을 절약합니다 |
| **JDBC/ODBC** | 표준 프로토콜로 Tableau, Power BI 등 BI 도구와 연결됩니다 |

---

## 참고 링크

- [Databricks: SQL Warehouses](https://docs.databricks.com/aws/en/compute/sql-warehouse/)
- [Azure Databricks: SQL Warehouses](https://learn.microsoft.com/en-us/azure/databricks/compute/sql-warehouse/)
- [Databricks: BI tool connections](https://docs.databricks.com/aws/en/partners/bi/)
