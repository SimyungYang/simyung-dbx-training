# Feature Engineering 개요

## 피처란?

> 💡 **피처(Feature)**는 ML 모델의 입력으로 사용되는 **개별 데이터 속성**입니다. 예를 들어, 사기 감지 모델에서 "거래 금액", "거래 시간", "최근 7일 거래 횟수" 등이 피처입니다.

---

## Feature Table

Unity Catalog의 테이블을 **Feature Table**로 사용할 수 있습니다. 피처를 중앙에서 관리하고, 여러 모델에서 재사용할 수 있습니다.

```python
from databricks.feature_engineering import FeatureEngineeringClient

fe = FeatureEngineeringClient()

# Feature Table 생성
fe.create_table(
    name="catalog.schema.customer_features",
    primary_keys=["customer_id"],
    df=customer_features_df,
    description="고객 행동 피처"
)
```

---

## Point-in-Time Lookup

> 💡 시간 기반 피처를 결합할 때, 미래 데이터가 섞이는 **데이터 유출(Data Leakage)**을 방지하기 위해, 특정 시점 기준으로 피처를 조회하는 기능입니다.

---

## 참고 링크

- [Databricks: Feature Engineering](https://docs.databricks.com/aws/en/machine-learning/feature-store/)
