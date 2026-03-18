# SDP란? — 선언적 파이프라인

## 개념

> 💡 **Spark Declarative Pipelines (SDP)**는 데이터 변환 파이프라인을 **"무엇을(What)"** 만들지 선언하면, Databricks가 **"어떻게(How)"** 실행할지를 자동으로 관리해 주는 프레임워크입니다. 이전에는 **Delta Live Tables (DLT)**라는 이름으로 불렸습니다.

### 명령형 vs 선언형 비교

| 방식 | 명령형 (Imperative) | 선언형 (Declarative) |
|------|-------------------|-------------------|
| 비유 | "달걀을 깨고, 팬을 달구고, 기름을 두르고, 달걀을 넣고..." | "스크램블 에그를 만들어 주세요" |
| 코드 | 읽기→변환→쓰기→에러처리→재시도 직접 구현 | 결과 테이블의 정의만 작성 |
| 의존성 | 개발자가 실행 순서를 직접 관리 | 시스템이 자동으로 파악 |
| 에러 처리 | 개발자가 직접 구현 | 시스템이 자동 관리 |

```python
# 명령형: 모든 것을 직접 관리
df = spark.readStream.format("delta").load("/bronze/orders")
cleaned = df.filter(col("order_id").isNotNull()).dropDuplicates(["order_id"])
cleaned.writeStream.format("delta").option("checkpointLocation", "...").toTable("silver_orders")
```

```sql
-- 선언형 (SDP): 결과만 정의
CREATE OR REFRESH STREAMING TABLE silver_orders
AS SELECT * FROM STREAM(bronze_orders) WHERE order_id IS NOT NULL;
```

---

## SDP의 핵심 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **Streaming Table** | 새 데이터만 증분 처리하는 추가 전용(Append-only) 테이블입니다 |
| **Materialized View** | 전체 데이터를 대상으로 결과를 재계산하는 테이블입니다 |
| **Expectations** | 데이터 품질 규칙을 정의하여, 부적합 데이터를 자동 처리합니다 |
| **Pipeline** | 여러 테이블/뷰의 정의를 묶어서 실행하는 단위입니다 |

---

## SDP의 장점

| 장점 | 설명 |
|------|------|
| **자동 의존성 관리** | 테이블 간 참조를 분석하여 실행 순서를 자동으로 결정합니다 |
| **자동 증분 처리** | 변경된 데이터만 처리하여 비용과 시간을 절약합니다 |
| **데이터 품질 모니터링** | Expectations로 데이터 품질을 자동으로 검증하고 리포팅합니다 |
| **자동 에러 복구** | 장애 발생 시 체크포인트에서 자동으로 재시작합니다 |
| **서버리스 지원** | 클러스터 관리 없이 서버리스로 실행할 수 있습니다 |

> 🆕 **TRIGGER ON UPDATE**: 소스 테이블이 변경되면 파이프라인을 자동으로 갱신하는 기능이 추가되었습니다. 수동 스케줄링 없이도 데이터 변경에 즉시 반응하는 파이프라인을 구축할 수 있습니다.

---

## 참고 링크

- [Databricks: Spark Declarative Pipelines](https://docs.databricks.com/aws/en/sdp/)
