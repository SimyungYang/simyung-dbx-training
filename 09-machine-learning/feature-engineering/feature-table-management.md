# 피처 테이블 관리

## 피처 조회와 학습 데이터 생성

```python
from databricks.feature_engineering import FeatureEngineeringClient, FeatureLookup

fe = FeatureEngineeringClient()

# 학습 데이터에 피처 결합
training_set = fe.create_training_set(
    df=labels_df,  # 라벨이 있는 데이터
    feature_lookups=[
        FeatureLookup(
            table_name="catalog.schema.customer_features",
            lookup_key="customer_id"
        ),
        FeatureLookup(
            table_name="catalog.schema.product_features",
            lookup_key="product_id"
        )
    ],
    label="is_fraud"
)

training_df = training_set.load_df()
```

---

## 피처 업데이트

```python
# 새로운 피처 데이터로 테이블 업데이트
fe.write_table(
    name="catalog.schema.customer_features",
    df=new_features_df,
    mode="merge"  # 기존 데이터와 병합
)
```

---

## 참고 링크

- [Databricks: Feature tables](https://docs.databricks.com/aws/en/machine-learning/feature-store/feature-tables.html)
