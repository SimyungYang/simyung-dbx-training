# 수집 방법 선택 가이드

## 상황별 권장 수집 방법

| 소스 유형 | 수집 방식 | 도구 | 설명 |
|-----------|----------|------|------|
| **RDBMS (MySQL, PostgreSQL, Oracle)** | CDC (증분) | Lakeflow Connect | 초기 스냅샷 후 변경분만 실시간 수집합니다 |
| **SaaS (Salesforce, Workday)** | API 기반 | Lakeflow Connect | 관리형 커넥터로 자동 수집합니다 |
| **클라우드 파일 (CSV, JSON, Parquet)** | 파일 감지 | Auto Loader | 새 파일 도착 시 자동으로 수집합니다 |
| **스트리밍 (Kafka, Kinesis)** | 이벤트 스트림 | Structured Streaming | 실시간 이벤트를 직접 소비합니다 |
| **REST API** | 커스텀 호출 | Python + Jobs | API를 호출하여 데이터를 가져옵니다 |
| **SFTP 서버** | 파일 전송 | SFTP Connector / Auto Loader | SFTP의 파일을 자동 수집합니다 |
| **Google Sheets** | 스케줄 동기화 | Google Sheets Connector | 시트 데이터를 주기적으로 동기화합니다 |

> 💡 **핵심 원칙**: 관리형 커넥터(Lakeflow Connect)가 지원하는 소스라면 커넥터를 우선 사용하세요. 커넥터가 없는 소스는 Auto Loader(파일 기반) 또는 커스텀 코드를 사용합니다.

---

## 참고 링크

- [Databricks: Ingestion](https://docs.databricks.com/aws/en/ingestion/)
- [Databricks: Lakeflow Connect connectors](https://docs.databricks.com/aws/en/lakeflow-connect/)
