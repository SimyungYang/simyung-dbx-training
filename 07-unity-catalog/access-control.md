# 권한 관리

## GRANT / REVOKE

```sql
-- 스키마 사용 권한 부여
GRANT USE SCHEMA ON SCHEMA production.ecommerce TO `analysts`;

-- 테이블 읽기 권한 부여
GRANT SELECT ON TABLE production.ecommerce.orders TO `analysts`;

-- 전체 권한 부여
GRANT ALL PRIVILEGES ON SCHEMA production.ecommerce TO `data_engineers`;

-- 권한 회수
REVOKE SELECT ON TABLE production.ecommerce.orders FROM `analysts`;
```

## 행/열 수준 보안

### 행 필터 (Row Filter)

```sql
-- 사용자의 지역에 해당하는 행만 보이도록 설정
CREATE FUNCTION region_filter(region STRING)
RETURN IF(IS_MEMBER('admin'), TRUE, region = CURRENT_USER_ATTRIBUTE('region'));

ALTER TABLE orders SET ROW FILTER region_filter ON (region);
```

### 컬럼 마스킹 (Column Mask)

```sql
-- 이메일을 마스킹하는 함수
CREATE FUNCTION mask_email(email STRING)
RETURN IF(IS_MEMBER('admin'), email, CONCAT('***@', SPLIT(email, '@')[1]));

ALTER TABLE customers ALTER COLUMN email SET MASK mask_email;
```

---

## 참고 링크

- [Databricks: Manage privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/)
