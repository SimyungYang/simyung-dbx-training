# 쿼리 최적화

## Liquid Clustering

기존의 파티셔닝과 Z-Order를 대체하는 차세대 최적화 방식입니다.

```sql
-- 테이블 생성 시 Clustering 지정
CREATE TABLE orders CLUSTER BY (order_date, customer_id) AS SELECT ...;

-- 기존 테이블에 추가
ALTER TABLE orders CLUSTER BY (order_date, customer_id);

-- 클러스터링 실행
OPTIMIZE orders;
```

---

## Query Profile 분석

SQL Editor에서 쿼리 실행 후 **Query Profile** 탭을 클릭하면, 쿼리의 실행 계획을 시각적으로 분석할 수 있습니다.

| 확인 항목 | 의미 |
|-----------|------|
| **Scan 크기** | 읽은 데이터량. 불필요하게 많으면 필터/클러스터링을 확인합니다 |
| **Shuffle 크기** | 노드 간 데이터 이동량. JOIN이나 GROUP BY에서 발생합니다 |
| **Spill** | 메모리 초과로 디스크에 쓴 데이터. 클러스터 크기를 늘려야 합니다 |

---

## 최적화 체크리스트

| 항목 | 방법 |
|------|------|
| **필요한 컬럼만 SELECT** | `SELECT *` 대신 필요한 컬럼만 명시합니다 |
| **Liquid Clustering 사용** | 자주 필터링하는 컬럼으로 CLUSTER BY를 설정합니다 |
| **ANALYZE TABLE 실행** | 통계 정보를 갱신하여 쿼리 계획을 최적화합니다 |
| **적절한 Warehouse 크기** | 워크로드에 맞는 T-Shirt 사이즈를 선택합니다 |
| **캐싱 활용** | 동일 쿼리가 반복되면 결과가 자동 캐싱됩니다 |

---

## 참고 링크

- [Databricks: Query optimization](https://docs.databricks.com/aws/en/optimizations/)
- [Databricks: Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering.html)
