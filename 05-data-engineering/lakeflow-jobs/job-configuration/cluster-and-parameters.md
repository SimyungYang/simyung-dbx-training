# 클러스터 설정과 파라미터

## 클러스터 설정

### Job Cluster vs All-Purpose Cluster

태스크에 할당하는 컴퓨트 유형에 따라 비용과 성능이 크게 달라집니다.

| 구분 | Job Cluster | All-Purpose Cluster |
|------|-------------|---------------------|
| **생명주기**| Job 실행 시 생성, 완료 시 자동 종료됩니다 | 수동 시작/종료 또는 자동 종료 설정이 필요합니다 |
| **비용**| Job 컴퓨트 요금 적용 (더 저렴) | All-Purpose 요금 적용 (약 2~3배 비쌈) |
| **적합한 용도**| 프로덕션 배치 Job | 개발, 디버깅, 탐색적 분석 |
| **시작 시간**| 매 실행 시 클러스터 시작 대기 필요 (2~5분) | 이미 실행 중이면 즉시 사용 가능합니다 |
| **자원 공유**| 해당 Job 전용입니다 | 여러 사용자/노트북이 공유합니다 |

> ⚠️ **프로덕션 Job에는 반드시 Job Cluster 또는 Serverless를 사용하세요.**All-Purpose Cluster는 개발 단계에서만 사용하고, 운영 환경에서는 비용이 크게 증가할 수 있습니다.

### Serverless Compute

Serverless를 선택하면 클러스터 구성을 별도로 관리할 필요가 없습니다. Databricks가 자동으로 인프라를 프로비저닝하고, 실행이 끝나면 즉시 해제합니다.

| 장점 | 단점 |
|------|------|
| 클러스터 시작 대기 시간 없음 (수 초) | 커스텀 라이브러리 설치 제한 |
| 인프라 관리 불필요 | 특정 인스턴스 타입 지정 불가 |
| 자동 스케일링 | 일부 워크로드에서 비용이 더 높을 수 있음 |

### 인스턴스 타입 선택 가이드

| 워크로드 유형 | 권장 인스턴스 | 이유 |
|---------------|---------------|------|
| **ETL / 데이터 변환**| 메모리 최적화 (r5, r6i) | 셔플, 조인 시 메모리 사용량이 높습니다 |
| **ML 학습**| GPU 인스턴스 (p3, g5) | 딥러닝 모델 학습 가속화 |
| **가벼운 집계 / SQL**| 범용 (m5, m6i) | 비용 대비 균형 잡힌 성능 |
| **대용량 셔플** | 스토리지 최적화 (i3, i4i) | 로컬 NVMe SSD로 셔플 성능 극대화 |

---

## 파라미터 전달

### Job 레벨 파라미터

Job 레벨에서 파라미터를 정의하면 모든 태스크에서 참조할 수 있습니다.

```python
# 노트북에서 Job 파라미터 받기
dbutils.widgets.text("target_date", "2025-03-18")
dbutils.widgets.dropdown("mode", "full", ["full", "incremental"])

target_date = dbutils.widgets.get("target_date")
mode = dbutils.widgets.get("mode")

# 파라미터를 사용한 조건부 처리
if mode == "incremental":
    df = spark.sql(f"""
        SELECT * FROM silver.orders
        WHERE order_date = '{target_date}'
    """)
else:
    df = spark.sql("SELECT * FROM silver.orders")
```

### 태스크 간 값 전달 (Task Values)

선행 태스크에서 계산한 값을 후행 태스크로 전달할 수 있습니다.

```python
# [Task A] 값 설정
row_count = df.count()
dbutils.jobs.taskValues.set(key="processed_rows", value=row_count)
dbutils.jobs.taskValues.set(key="max_date", value="2025-03-18")
```

```python
# [Task B] 값 읽기 (Task A의 결과 참조)
processed_rows = dbutils.jobs.taskValues.get(
    taskKey="task_a",
    key="processed_rows",
    default=0
)
max_date = dbutils.jobs.taskValues.get(
    taskKey="task_a",
    key="max_date",
    default="1970-01-01"
)

print(f"Task A에서 처리한 행 수: {processed_rows}")
print(f"최대 날짜: {max_date}")
```

> 💡 **동적 값 참조**: YAML 설정에서 `{{tasks.task_a.values.processed_rows}}` 형태로 다른 태스크의 Task Value를 참조할 수도 있습니다.

---
