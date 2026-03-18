# 10. AI 에이전트 개발 (Agent Development)

> Databricks 위에서 AI 에이전트를 개발, 평가, 배포하는 방법을 학습합니다.

## 학습 목표
- AI 에이전트의 개념과 구성 요소(LLM, Tool, Retriever) 이해
- Databricks에서 에이전트를 구축하는 방법 (ChatAgent, Mosaic AI)
- Vector Search를 활용한 RAG(Retrieval-Augmented Generation) 구현
- 에이전트 평가(Agent Evaluation)와 프로덕션 배포

## 문서 목록

| 순서 | 문서 | 내용 |
|------|------|------|
| 1 | [AI 에이전트란?](what-is-ai-agent.md) | 개념, LLM + Tool 패턴, RAG 소개를 다룹니다 |
| 2 | [Vector Search](vector-search.md) | 임베딩, 3가지 인덱스 유형, Reranker, 유사도 검색을 상세히 설명합니다 |
| 3 | [RAG 파이프라인](rag-pipeline.md) | 문서 파싱, 청킹, 임베딩, 검색 증강 생성 구축을 안내합니다 |
| 4 | [에이전트 구축](building-agents.md) | ChatAgent 인터페이스, UC Functions를 Tool로 활용하는 방법을 다룹니다 |
| 5 | [에이전트 평가](agent-evaluation.md) | MLflow 기반 평가, Scorer, 품질 모니터링을 설명합니다 |
| 6 | [에이전트 배포](agent-deployment.md) | Model Serving 엔드포인트 배포, Review App을 안내합니다 |
| 7 | [Agent Bricks](agent-bricks.md) | Knowledge Assistant, Genie, Supervisor Agent(멀티에이전트)를 다룹니다 |

## 참고 문서
- [Databricks: Generative AI](https://docs.databricks.com/aws/en/generative-ai/)
- [Databricks: Mosaic AI Agent Framework](https://docs.databricks.com/aws/en/generative-ai/agent-framework/)
