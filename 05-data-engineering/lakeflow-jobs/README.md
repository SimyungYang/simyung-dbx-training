# 05-4. Lakeflow Jobs (워크플로우)

> 데이터 파이프라인과 작업을 스케줄링하고 오케스트레이션하는 방법을 학습합니다.

## 학습 목표
- Databricks Jobs/Workflows의 개념과 구성 요소 이해
- 멀티 태스크(Task) 의존성 그래프 설계
- 스케줄링, 트리거, 알림 설정
- 작업 실패 시 재시도 및 모니터링

## 문서 목록

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [Lakeflow Jobs란?](what-is-lakeflow-jobs.md) | 워크플로우 오케스트레이션 개념, Task 유형, 시스템 제약을 설명합니다 |
| 2 | [작업 구성](job-configuration.md) | 태스크, 의존성, 매개변수, Asset Bundles 설정을 다룹니다 |
| 3 | [스케줄링과 트리거](scheduling-and-triggers.md) | Cron, 파일 도착 트리거, Continuous 모드를 안내합니다 |
| 4 | [모니터링과 알림](monitoring-and-alerting.md) | 실행 이력, 시스템 테이블 쿼리, 비용 추적을 설명합니다 |

## 참고 문서
- [Databricks: Lakeflow Jobs](https://docs.databricks.com/aws/en/jobs/)
- [Azure Databricks: Workflows](https://learn.microsoft.com/en-us/azure/databricks/workflows/)
