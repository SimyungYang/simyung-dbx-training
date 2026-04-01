# 배포 및 운영

## 배포 및 운영 팁

### Databricks Apps 배포

```bash
# 프로젝트 구조
my-app/
| 파일 | 설명 |
|------|------|
| `app.py` | 메인 앱 코드 |
| `requirements.txt` | Python 의존성 |
| `app.yaml` | Databricks Apps 설정 |
| `.streamlit/secrets.toml` | 시크릿 (로컬 개발용) |
```

**app.yaml 설정**:

```yaml
# app.yaml — Databricks Apps 배포 설정
command:
  - "streamlit"
  - "run"
  - "app.py"
  - "--server.port=8501"
  - "--server.address=0.0.0.0"

env:
  - name: LAKEBASE_HOST
    value: "app-db-xxxx.lakebase.databricks.com"
  - name: LAKEBASE_DBNAME
    value: "shop_db"
```

### 운영 체크리스트

| 항목 | 확인 사항 |
|------|-----------|
| **커넥션 풀**| 적절한 크기로 설정되어 있는지 확인합니다 |
| **에러 처리**| 모든 DB 작업에 try/except가 있는지 확인합니다 |
| **SQL 인젝션 방지**| 파라미터 바인딩(`%s`)을 사용하는지 확인합니다 |
| **타임아웃 설정**| 쿼리 타임아웃과 연결 타임아웃이 설정되어 있는지 확인합니다 |
| **로깅**| 에러와 느린 쿼리가 로깅되는지 확인합니다 |
| **헬스체크**| `/health` 엔드포인트에서 DB 연결 상태를 확인합니다 |

> ⚠️ **SQL 인젝션 방지**: 사용자 입력을 SQL 쿼리에 직접 넣으면 보안 취약점이 됩니다. 반드시 **파라미터 바인딩**(`%s` 플레이스홀더)을 사용하시기 바랍니다. `f"SELECT * FROM users WHERE id={user_input}"` 형태는 절대 사용하지 마세요.

---

## 정리

| 핵심 포인트 | 설명 |
|-------------|------|
| **통합 플랫폼**| Databricks Apps + Lakebase로 앱 개발~분석까지 하나의 플랫폼에서 처리합니다 |
| **다양한 프레임워크**| Streamlit, FastAPI, Dash, Flask 등 Python 프레임워크를 자유롭게 선택합니다 |
| **OAuth 통합 인증**| 별도의 인증 시스템 없이 Databricks OAuth를 활용합니다 |
| **커넥션 풀링**| 프로덕션에서는 반드시 커넥션 풀을 설정합니다 |
| **자동 동기화**| 앱 데이터가 Data Sync로 자동으로 분석 환경에 반영됩니다 |
| **보안** | SQL 인젝션 방지, SSL 필수, 파라미터 바인딩을 준수합니다 |

---

## 참고 링크

- [Databricks: Databricks Apps](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/)
- [Databricks: Lakebase](https://docs.databricks.com/aws/en/lakebase/)
- [Azure Databricks: Databricks Apps](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/databricks-apps/)
- [Streamlit Documentation](https://docs.streamlit.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [psycopg2 Documentation](https://www.psycopg.org/docs/)
