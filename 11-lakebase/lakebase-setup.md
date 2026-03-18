# Lakebase 설정

## 인스턴스 생성

Catalog Explorer에서 **Lakebase** → **Create Database**로 생성합니다.

```sql
-- 또는 SQL로 생성
CREATE DATABASE catalog.schema.my_lakebase
TYPE LAKEBASE;
```

---

## 연결

표준 PostgreSQL 클라이언트로 연결할 수 있습니다.

```python
import psycopg2

conn = psycopg2.connect(
    host="<lakebase-host>",
    port=5432,
    dbname="my_lakebase",
    user="<user>",
    password="<token>"
)

cursor = conn.cursor()
cursor.execute("CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT, email TEXT)")
cursor.execute("INSERT INTO users (name, email) VALUES ('김철수', 'cs@mail.com')")
conn.commit()
```

---

## 참고 링크

- [Databricks: Create Lakebase database](https://docs.databricks.com/aws/en/lakebase/)
