# 10. AI 에이전트 개발 (Agent Development)

> Databricks 위에서 AI 에이전트를 개발, 평가, 배포하는 방법을 학습합니다.

## 학습 목표
- AI 에이전트의 개념과 구성 요소(LLM, Tool, Retriever) 이해
- Databricks에서 에이전트를 구축하는 방법 (ChatAgent, Mosaic AI)
- Vector Search를 활용한 RAG(Retrieval-Augmented Generation) 구현
- 에이전트 평가(Agent Evaluation)와 프로덕션 배포

## 문서 목록

| 순서 | 파일명 | 내용 |
|------|--------|------|
| 1 | `what-is-ai-agent.md` | AI 에이전트란? — 개념, LLM + Tool 패턴, RAG 소개 |
| 2 | `vector-search.md` | Vector Search — 임베딩, 벡터 인덱스, 유사도 검색 |
| 3 | `rag-pipeline.md` | RAG 파이프라인 — 문서 파싱, 청킹, 임베딩, 검색 증강 생성 |
| 4 | `building-agents.md` | 에이전트 구축 — ChatAgent, UC Functions를 Tool로 활용 |
| 5 | `agent-evaluation.md` | 에이전트 평가 — MLflow 기반 평가, Scorer, 품질 모니터링 |
| 6 | `agent-deployment.md` | 에이전트 배포 — Model Serving 엔드포인트, Review App |
| 7 | `agent-bricks.md` | Agent Bricks — Knowledge Assistant, Genie, 멀티에이전트 |

## 참고 문서
- [Databricks: Generative AI](https://docs.databricks.com/aws/en/generative-ai/)
- [Databricks: Mosaic AI Agent Framework](https://docs.databricks.com/aws/en/generative-ai/agent-framework/)
