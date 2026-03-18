# Databricks Apps

## 개념

> 💡 **Databricks Apps**는 Databricks 위에서 **웹 애플리케이션을 개발하고 배포**할 수 있는 서비스입니다. Streamlit, Gradio, Flask, FastAPI, Dash 등 다양한 Python 웹 프레임워크를 지원합니다.

---

## 지원 프레임워크

| 프레임워크 | 적합한 용도 |
|-----------|-----------|
| **Streamlit** | 데이터 대시보드, ML 데모 |
| **Gradio** | AI 모델 인터페이스 |
| **Dash** | 대화형 데이터 시각화 |
| **Flask / FastAPI** | REST API, 백엔드 서비스 |

---

## 앱 배포

```yaml
# app.yaml
command:
  - "streamlit"
  - "run"
  - "app.py"
  - "--server.port"
  - "8000"

resources:
  - name: "sql-warehouse"
    sql_warehouse:
      id: "${DATABRICKS_WAREHOUSE_ID}"
      permission: "CAN_USE"
```

```bash
# 배포
databricks apps deploy my-app --source-code-path ./app
```

> 🆕 **컴퓨트 사이징 GA**: Medium(2 vCPU, 6GB)과 Large(4 vCPU, 12GB) 옵션이 GA되어, 앱의 워크로드에 맞는 리소스를 선택할 수 있습니다.

---

## 참고 링크

- [Databricks: Databricks Apps](https://docs.databricks.com/aws/en/dev-tools/databricks-apps/)
