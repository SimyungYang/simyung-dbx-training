# AI 함수 — SQL에서 직접 LLM 호출하기

## 왜 SQL에 AI 함수가 필요한가요?

전통적으로 텍스트 분류, 감정 분석, 정보 추출 같은 작업을 하려면 Python 코드를 작성하고, ML 모델을 별도로 호출해야 했습니다. Databricks의 **AI 함수**를 사용하면 이 모든 것을 **SQL 쿼리 한 줄**로 수행할 수 있습니다.

> 💡 AI 함수는 내부적으로 **Model Serving 엔드포인트**를 호출합니다. Foundation Model API에서 제공하는 LLM(Llama, DBRX 등)이나 외부 모델(OpenAI, Anthropic) 엔드포인트를 사용할 수 있습니다. SQL Warehouse(Pro 또는 Serverless)에서 실행해야 합니다.

---

## ai_query() — 범용 LLM 호출

가장 유연한 AI 함수로, 자유 형식의 프롬프트를 LLM에 전달하고 응답을 받습니다.

### 기본 문법

```sql
ai_query(
    endpoint,           -- Model Serving 엔드포인트 이름
    request,            -- 프롬프트 (문자열 또는 STRUCT)
    returnType,         -- 반환 타입 (기본: STRING)
    failOnError,        -- 에러 시 실패 여부 (기본: true)
    modelParameters     -- 모델 파라미터 (temperature, max_tokens 등)
)
```

### 사용 예시

```sql
-- 1. 기본 사용: 텍스트 요약
SELECT
    article_id,
    ai_query(
        'databricks-meta-llama-3-3-70b-instruct',
        CONCAT('다음 기사를 한국어 3줄로 요약해 주세요:\n\n', article_text)
    ) AS summary
FROM news_articles;

-- 2. 구조화된 응답 반환 (JSON → STRUCT)
SELECT
    review_id,
    ai_query(
        'databricks-meta-llama-3-3-70b-instruct',
        CONCAT('다음 리뷰를 분석하여 JSON으로 반환해 주세요: ', review_text),
        returnType => 'STRUCT<sentiment STRING, topics ARRAY<STRING>, score INT>'
    ) AS analysis
FROM customer_reviews;

-- 3. 모델 파라미터 지정
SELECT ai_query(
    'databricks-meta-llama-3-3-70b-instruct',
    '데이터 레이크하우스의 장점을 5가지 나열해 주세요.',
    modelParameters => named_struct(
        'temperature', 0.1,           -- 낮을수록 결정적 (0.0~2.0)
        'max_tokens', 500,            -- 최대 응답 토큰 수
        'top_p', 0.9                  -- 확률 기반 토큰 선택 범위
    )
) AS response;

-- 4. 시스템 프롬프트 + 사용자 프롬프트 (Chat 형식)
SELECT ai_query(
    'databricks-meta-llama-3-3-70b-instruct',
    named_struct(
        'messages', ARRAY(
            named_struct('role', 'system', 'content', '당신은 전문 데이터 분석가입니다. 한국어로 답변해 주세요.'),
            named_struct('role', 'user', 'content', CONCAT('다음 데이터를 분석해 주세요: ', data_text))
        )
    )
) AS analysis
FROM reports;

-- 5. 배치 처리 시 에러 무시 (일부 실패해도 전체 쿼리 계속)
SELECT
    doc_id,
    ai_query(
        'databricks-meta-llama-3-3-70b-instruct',
        CONCAT('요약: ', content),
        failOnError => false
    ) AS summary
FROM documents;
```

### 사용 가능한 엔드포인트

| 엔드포인트 이름 | 모델 | 설명 |
|----------------|------|------|
| `databricks-meta-llama-3-3-70b-instruct` | Llama 3.3 70B | Meta의 최신 오픈소스 LLM입니다 |
| `databricks-dbrx-instruct` | DBRX | Databricks가 만든 오픈소스 LLM입니다 |
| `databricks-gte-large-en` | GTE Large | 영어 텍스트 임베딩 모델입니다 |
| `databricks-bge-large-en` | BGE Large | 영어 텍스트 임베딩 모델입니다 |
| 사용자 지정 | 커스텀 모델 | Model Serving에 배포한 사용자 모델입니다 |
| 사용자 지정 | 외부 모델 | OpenAI, Anthropic 등의 프록시 엔드포인트입니다 |

> 💡 **Pay-per-token 엔드포인트**: `databricks-` 접두사가 붙은 내장 엔드포인트는 별도 설정 없이 바로 사용할 수 있으며, 사용한 토큰 수에 따라 과금됩니다.

---

## ai_classify() — 텍스트 분류

텍스트를 미리 정의한 카테고리 중 하나로 분류합니다. 별도의 학습 데이터 없이 LLM의 이해력만으로 분류를 수행합니다.

### 문법

```sql
ai_classify(content, labels)
-- content: 분류할 텍스트 (STRING)
-- labels:  카테고리 목록 (ARRAY<STRING>)
```

### 예시

```sql
-- 고객 문의 유형 분류
SELECT
    ticket_id,
    subject,
    ai_classify(
        subject || ': ' || body,
        ARRAY('배송문의', '반품/환불', '상품문의', '결제문제', '기타')
    ) AS category
FROM support_tickets;

-- 감정 분석 (긍정/부정/중립)
SELECT
    review_id,
    review_text,
    ai_classify(review_text, ARRAY('긍정', '부정', '중립')) AS sentiment
FROM customer_reviews;

-- 언어 감지
SELECT
    doc_id,
    ai_classify(content, ARRAY('한국어', '영어', '일본어', '중국어')) AS language
FROM documents;

-- 긴급도 분류
SELECT
    alert_id,
    message,
    ai_classify(message, ARRAY('긴급', '높음', '보통', '낮음')) AS priority
FROM system_alerts;
```

---

## ai_extract() — 정보 추출

텍스트에서 특정 정보를 추출합니다. 정규 표현식으로는 어려운 비정형 텍스트에서도 LLM의 이해력으로 정확하게 추출할 수 있습니다.

### 문법

```sql
ai_extract(content, labels)
-- content: 원본 텍스트 (STRING)
-- labels:  추출할 정보 항목 목록 (ARRAY<STRING>)
-- 반환:    STRUCT (각 항목이 필드로 포함)
```

### 예시

```sql
-- 이메일에서 핵심 정보 추출
SELECT
    email_id,
    ai_extract(
        email_body,
        ARRAY('발신자이름', '회사명', '요청사항', '긴급여부', '연락처')
    ) AS extracted
FROM inbox_emails;

-- 추출된 STRUCT에서 개별 필드 접근
SELECT
    email_id,
    extracted.발신자이름,
    extracted.회사명,
    extracted.요청사항
FROM (
    SELECT
        email_id,
        ai_extract(email_body, ARRAY('발신자이름', '회사명', '요청사항')) AS extracted
    FROM inbox_emails
);

-- 계약서에서 핵심 조건 추출
SELECT
    contract_id,
    ai_extract(
        contract_text,
        ARRAY('계약금액', '계약기간', '갱신조건', '해지조건', '위약금')
    ) AS terms
FROM contracts;

-- 뉴스에서 구조화된 정보 추출
SELECT
    ai_extract(
        article_text,
        ARRAY('기업명', '인물', '금액', '날짜', '장소', '사건유형')
    ) AS entities
FROM news_articles;
```

---

## ai_gen() — 텍스트 생성

주어진 컨텍스트를 기반으로 새로운 텍스트를 생성합니다.

### 예시

```sql
-- 상품 설명 자동 생성
SELECT
    product_id,
    product_name,
    ai_gen(
        CONCAT('다음 상품의 매력적인 한국어 마케팅 문구를 100자 이내로 작성해 주세요. ',
               '상품명: ', product_name,
               ', 카테고리: ', category,
               ', 주요 특징: ', features)
    ) AS marketing_copy
FROM products;

-- 데이터 기반 인사이트 생성
SELECT
    region,
    total_revenue,
    yoy_growth,
    ai_gen(
        CONCAT('다음 지역 매출 데이터에 대한 분석 코멘트를 한국어 2문장으로 작성해 주세요. ',
               '지역: ', region,
               ', 매출: ', FORMAT_NUMBER(total_revenue, 0), '원',
               ', 전년대비 성장률: ', ROUND(yoy_growth * 100, 1), '%')
    ) AS analyst_comment
FROM regional_sales_summary;

-- 번역
SELECT
    doc_id,
    ai_gen(CONCAT('Translate the following text to Korean:\n\n', english_text)) AS korean_text
FROM english_documents;
```

---

## ai_parse_document() — 문서 파싱

PDF, DOCX, PPTX, 이미지 등의 비정형 문서에서 텍스트를 추출합니다.

### 문법

```sql
ai_parse_document(
    path,               -- Volume 내 파일 경로 (STRING)
    mode                -- 'text' 또는 'markdown' (추출 형식)
)
```

### 예시

```sql
-- PDF에서 텍스트 추출
SELECT ai_parse_document(
    '/Volumes/catalog/schema/contracts/agreement_2025.pdf',
    'text'
) AS parsed_text;

-- Markdown 형식으로 추출 (제목, 표 등 구조 유지)
SELECT ai_parse_document(
    '/Volumes/catalog/schema/reports/quarterly_report.pdf',
    'markdown'
) AS parsed_markdown;

-- Volume의 여러 파일을 일괄 파싱
SELECT
    file_name,
    ai_parse_document(
        CONCAT('/Volumes/catalog/schema/documents/', file_name),
        'markdown'
    ) AS content
FROM (
    SELECT file_name FROM LIST('/Volumes/catalog/schema/documents/')
    WHERE file_name LIKE '%.pdf'
);

-- 파싱 후 AI 분석 연결 (파싱 → 요약 파이프라인)
WITH parsed AS (
    SELECT
        file_name,
        ai_parse_document(
            CONCAT('/Volumes/catalog/schema/docs/', file_name), 'text'
        ) AS content
    FROM document_list
)
SELECT
    file_name,
    ai_query(
        'databricks-meta-llama-3-3-70b-instruct',
        CONCAT('다음 문서를 5줄로 요약해 주세요:\n\n', content)
    ) AS summary,
    ai_classify(content, ARRAY('계약서', '보고서', '제안서', '매뉴얼')) AS doc_type,
    ai_extract(content, ARRAY('작성자', '작성일', '주요키워드')) AS metadata
FROM parsed;
```

### 지원 파일 형식

| 형식 | 확장자 | 설명 |
|------|--------|------|
| PDF | `.pdf` | 텍스트 기반 및 스캔된 PDF 모두 지원합니다 |
| Word | `.docx` | Microsoft Word 문서입니다 |
| PowerPoint | `.pptx` | Microsoft PowerPoint 프레젠테이션입니다 |
| 이미지 | `.png`, `.jpg`, `.tiff` | OCR로 텍스트를 추출합니다 |
| HTML | `.html` | 웹 페이지 내용을 추출합니다 |

---

## ai_similarity() — 텍스트 유사도

두 텍스트 간의 의미적 유사도를 0~1 사이의 점수로 반환합니다.

```sql
-- 중복 고객 문의 감지
SELECT
    a.ticket_id AS ticket_a,
    b.ticket_id AS ticket_b,
    ai_similarity(a.description, b.description) AS similarity_score
FROM support_tickets a
CROSS JOIN support_tickets b
WHERE a.ticket_id < b.ticket_id
  AND ai_similarity(a.description, b.description) > 0.85;
```

---

## ai_forecast() 및 ai_anomaly_detection() — 시계열 분석

SQL만으로 시계열 예측과 이상 감지를 수행할 수 있습니다.

```sql
-- 매출 예측 (향후 30일)
SELECT * FROM ai_forecast(
    TABLE(SELECT sale_date, total_revenue FROM gold_daily_revenue),
    horizon => 30,
    time_col => 'sale_date',
    value_col => 'total_revenue'
);

-- 이상 감지
SELECT * FROM ai_anomaly_detection(
    TABLE(SELECT timestamp, metric_value FROM server_metrics),
    time_col => 'timestamp',
    value_col => 'metric_value'
);
```

---

## 성능과 비용 최적화

### 권장 사항

| 항목 | 권장 사항 |
|------|----------|
| **배치 크기**| 한 번에 수천 행을 처리하면 오버헤드가 줄어듭니다. `LIMIT`으로 테스트 후 전체 실행을 권장합니다 |
| ** 모델 선택**| 간단한 분류는 작은 모델, 복잡한 생성은 큰 모델을 사용합니다 |
| ** 캐싱**| 동일한 입력에 대한 반복 호출은 결과를 테이블에 저장하여 재사용합니다 |
| **failOnError**| 대량 처리 시 `failOnError => false`로 설정하여 일부 실패가 전체를 중단하지 않도록 합니다 |
| ** 프롬프트 최적화**| 프롬프트를 짧고 명확하게 작성하면 토큰 비용이 절약됩니다 |
| **Provisioned Throughput**| 대량/정기적 사용 시 Pay-per-token보다 비용 효율적입니다 |

### 비용 구조

```
비용 = 입력 토큰 수 × 입력 토큰 단가 + 출력 토큰 수 × 출력 토큰 단가
```

> 💡 ** 토큰(Token)이란?**LLM이 텍스트를 처리하는 기본 단위입니다. 영어 기준 대략 1단어 ≈ 1~1.5 토큰이며, 한국어는 1글자 ≈ 1~3 토큰으로 다소 많이 소모됩니다. 프롬프트(입력)와 응답(출력) 모두 토큰 수로 과금됩니다.

---

## 실무 활용 시나리오

### 시나리오 1: 고객 서비스 자동 분류 + 우선순위 파이프라인

```sql
-- Silver 테이블에 AI 분석 결과를 추가한 Gold 테이블
CREATE OR REFRESH MATERIALIZED VIEW gold_enriched_tickets AS
SELECT
    ticket_id,
    created_at,
    subject,
    body,
    -- AI 함수로 자동 분류
    ai_classify(
        subject || ': ' || body,
        ARRAY('배송', '반품/환불', '상품불량', '결제', '회원', '기타')
    ) AS auto_category,
    ai_classify(
        body,
        ARRAY('긴급', '높음', '보통', '낮음')
    ) AS auto_priority,
    ai_extract(
        body,
        ARRAY('주문번호', '상품명', '요청사항')
    ) AS extracted_info,
    ai_query(
        'databricks-meta-llama-3-3-70b-instruct',
        CONCAT('고객 문의에 대한 초안 답변을 한국어로 작성해 주세요. 정중하고 전문적인 톤으로 작성하되, 100자 이내로 요약해 주세요.\n\n문의 내용: ', body),
        modelParameters => named_struct('temperature', 0.3, 'max_tokens', 200)
    ) AS draft_response
FROM silver_support_tickets
WHERE created_at >= CURRENT_DATE() - INTERVAL 1 DAY;
```

### 시나리오 2: 비정형 문서 → 구조화 데이터 파이프라인

```sql
-- PDF 계약서를 구조화된 테이블로 변환
CREATE OR REFRESH MATERIALIZED VIEW gold_contract_summary AS
WITH parsed_contracts AS (
    SELECT
        file_name,
        ai_parse_document(
            CONCAT('/Volumes/legal/contracts/originals/', file_name), 'text'
        ) AS full_text
    FROM contract_file_list
)
SELECT
    file_name,
    ai_extract(
        full_text,
        ARRAY('계약 당사자', '계약 금액', '계약 시작일', '계약 종료일', '자동갱신 여부', '해지 통보 기한')
    ) AS contract_terms,
    ai_classify(
        full_text,
        ARRAY('소프트웨어 라이선스', 'SaaS 구독', '컨설팅', '유지보수', '기타')
    ) AS contract_type,
    ai_query(
        'databricks-meta-llama-3-3-70b-instruct',
        CONCAT('다음 계약서의 주요 리스크 요인을 3가지 이내로 한국어로 요약해 주세요:\n\n', full_text),
        modelParameters => named_struct('temperature', 0.1)
    ) AS risk_summary
FROM parsed_contracts;
```

---

## 정리

| AI 함수 | 용도 | 반환 타입 |
|---------|------|----------|
| **ai_query()**| 범용 LLM 호출 (요약, 생성, 번역, 분석) | STRING 또는 지정 STRUCT |
| **ai_classify()**| 텍스트를 사전 정의 카테고리로 분류 | STRING |
| **ai_extract()**| 텍스트에서 특정 정보 추출 | STRUCT |
| **ai_gen()**| 컨텍스트 기반 텍스트 생성 | STRING |
| **ai_parse_document()**| PDF, 이미지 등에서 텍스트 추출 | STRING |
| **ai_similarity()**| 두 텍스트 간 의미적 유사도 측정 | DOUBLE |
| **ai_forecast()**| 시계열 예측 | TABLE |
| **ai_anomaly_detection()** | 시계열 이상 감지 | TABLE |

---

## 참고 링크

- [Databricks: ai_query()](https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_query.html)
- [Databricks: ai_classify()](https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_classify.html)
- [Databricks: ai_extract()](https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_extract.html)
- [Databricks: ai_gen()](https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_gen.html)
- [Databricks: ai_parse_document()](https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_parse_document.html)
- [Databricks: ai_forecast()](https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_forecast.html)
- [Databricks: Foundation Model APIs](https://docs.databricks.com/aws/en/machine-learning/foundation-models/)
- [Databricks Blog: AI Functions](https://www.databricks.com/blog)
