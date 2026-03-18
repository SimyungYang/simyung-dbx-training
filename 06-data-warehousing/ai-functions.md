# AI 함수

## SQL 안에서 AI 활용하기

> 💡 Databricks는 SQL 쿼리 안에서 직접 **LLM(대규모 언어 모델)**을 호출할 수 있는 **AI 함수**를 제공합니다. 별도의 Python 코드 없이 SQL만으로 텍스트 분류, 정보 추출, 요약 등을 수행할 수 있습니다.

---

## 주요 AI 함수

| 함수 | 역할 | 예시 |
|------|------|------|
| **ai_query()** | 범용 LLM 호출 | 자유 형식 프롬프트로 LLM에 질의합니다 |
| **ai_classify()** | 텍스트 분류 | 리뷰를 "긍정/부정/중립"으로 분류합니다 |
| **ai_extract()** | 정보 추출 | 텍스트에서 이름, 날짜, 금액 등을 추출합니다 |
| **ai_gen()** | 텍스트 생성 | 요약, 번역, 답변 생성 등을 수행합니다 |
| **ai_parse_document()** | 문서 파싱 | PDF, 이미지에서 텍스트를 추출합니다 |

---

## 사용 예시

```sql
-- 고객 리뷰 감정 분석
SELECT
    review_id,
    review_text,
    ai_classify(review_text, ARRAY('긍정', '부정', '중립')) AS sentiment
FROM customer_reviews;

-- 텍스트에서 정보 추출
SELECT
    email_body,
    ai_extract(email_body, ARRAY('고객명', '주문번호', '문의유형')) AS extracted
FROM support_emails;

-- 범용 LLM 질의
SELECT ai_query(
    'databricks-meta-llama-3-3-70b-instruct',
    '다음 텍스트를 3줄로 요약해 주세요: ' || article_text
) AS summary
FROM news_articles;
```

---

## 참고 링크

- [Databricks: AI functions](https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_query.html)
