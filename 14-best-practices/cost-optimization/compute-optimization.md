# 컴퓨트 최적화

## 2. 컴퓨트 최적화

### 2.1 Serverless vs Classic 선택 기준

| 판단 기준 | Serverless 추천 | Classic 추천 |
|----------|----------------|-------------|
| **워크로드 패턴** | 간헐적, 불규칙 실행 | 24시간 상시 실행 |
| **시작 시간 민감도** | 즉시 시작 필요 (< 10초) | 수 분 대기 가능 |
| **비용 예측 필요** | 사용량 기반 과금 OK | 고정 예산 필수 |
| **특수 라이브러리** | 표준 라이브러리만 사용 | 커스텀 Docker, GPU, R |
| **데이터 규모** | 소~중규모 (TB 미만) | 대규모 (수십 TB 이상) |
| **관리 여력** | 인프라 팀 없음 | 전담 플랫폼 팀 있음 |

> 💡 **의사결정 공식**: 일일 평균 가동률이 **40% 미만**이면 Serverless가 경제적입니다. 40% 이상 상시 가동한다면 Classic + Reserved Capacity가 유리합니다.

### 2.2 Auto Stop 설정 전략

Auto Stop은 유휴 클러스터를 자동으로 종료하여 불필요한 비용을 방지합니다.

| 환경 | 권장 Auto Stop | 이유 |
|------|--------------|------|
| **개발 (Interactive)** | 10분 | 개발자가 잊고 퇴근해도 자동 종료 |
| **Ad-hoc 분석** | 15분 | 쿼리 간 대기 시간 고려 |
| **공유 클러스터** | 30분 | 여러 사용자의 간헐적 사용 |
| **프로덕션 (Jobs)** | 설정 불필요 | Jobs는 작업 완료 시 자동 종료 |
| **SQL Warehouse (개발)** | 10분 | 빠른 종료로 유휴 비용 제거 |
| **SQL Warehouse (프로덕션)** | 5~10분 | 자동 재시작 기능이 있으므로 공격적 설정 가능 |

```python
# Compute Policy로 Auto Stop 강제 적용 (관리자용)
{
  "autotermination_minutes": {
    "type": "range",
    "maxValue": 30,
    "defaultValue": 10
  },
  "custom_tags.CostCenter": {
    "type": "fixed",
    "value": "engineering"
  }
}
```

### 2.3 Spot 인스턴스 활용

Spot 인스턴스(AWS) / Spot VM(Azure)을 활용하면 **60~90% 비용 절감**이 가능합니다.

| 구성 요소 | Spot 사용 가능 | 권장 비율 | 주의사항 |
|----------|--------------|----------|---------|
| **Driver** | ❌ On-Demand 필수 | 0% Spot | 드라이버 중단 시 전체 작업 실패 |
| **Worker (배치)** | ✅ 적극 활용 | 80-100% Spot | 재시도 로직 필수 |
| **Worker (스트리밍)** | ⚠️ 주의 필요 | 50% 이하 Spot | 인터럽트 시 재처리 지연 |
| **Worker (ML 학습)** | ⚠️ 체크포인트 필수 | 50-80% Spot | 체크포인트 주기 설정 |

```python
# Spot 인스턴스 + Fallback 설정 예시
{
  "aws_attributes": {
    "first_on_demand": 1,          # 드라이버는 On-Demand
    "availability": "SPOT_WITH_FALLBACK",  # Spot 우선, 불가 시 On-Demand
    "spot_bid_price_percent": 100   # On-Demand 가격까지 입찰
  }
}
```

### 2.4 클러스터 사이징 가이드

워크로드 유형에 따른 인스턴스 선택 기준입니다.

| 워크로드 유형 | 병목 지점 | 권장 인스턴스 | 시작 구성 |
|-------------|----------|-------------|----------|
| **기본 ETL** | I/O | Storage Optimized (i3, i4i) | 4 workers, Medium |
| **복잡한 ETL (대량 조인)** | Memory + Shuffle | Memory Optimized (r5, r6i) | 8 workers, Large |
| **ML 학습** | CPU/GPU | Compute Optimized (c5) 또는 GPU (p3, g5) | 1 node (단일), Large |
| **Ad-hoc 분석** | 다양 | General Purpose (m5, m6i) | 2-4 workers, Medium |
| **스트리밍** | CPU + I/O | Compute Optimized (c5, c6i) | 4 workers, Medium |

> 💡 **사이징 공식**: `필요 코어 수 = (데이터 크기 GB / 목표 처리 시간 분) × 2`. 예를 들어 100GB 데이터를 10분 안에 처리하려면 약 20코어가 필요합니다. 이를 기준으로 워커 수를 결정하세요.

### 2.5 Instance Pool 활용

Instance Pool은 유휴 인스턴스를 미리 확보하여 클러스터 시작 시간을 단축합니다.

```python
# Instance Pool 설정 예시
{
  "instance_pool_name": "data-engineering-pool",
  "node_type_id": "i3.xlarge",
  "min_idle_instances": 2,        # 항상 2대 대기 (비용 발생)
  "max_capacity": 20,              # 최대 20대까지 확장
  "idle_instance_autotermination_minutes": 15,  # 유휴 인스턴스 15분 후 반환
  "preloaded_spark_versions": ["15.4.x-scala2.12"]  # 런타임 사전 로드
}
```

| 상황 | Pool 사용 권장 | 이유 |
|------|--------------|------|
| 하루 10회 이상 클러스터 시작 | ✅ | 시작 시간 5분 → 30초 |
| 하루 1-2회 실행 | ❌ | 유휴 비용 > 절감 효과 |
| 다수 팀 공유 환경 | ✅ | 시작 시간 SLA 보장 |

---
