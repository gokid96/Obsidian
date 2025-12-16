

## 정의
검색(Retrieval) + 생성(Generation)을 결합한 방식

## 왜 필요한가
LLM의 한계:
- 학습 데이터 이후 정보 모름
- 할루시네이션 (없는 거 지어냄)
- 내부 문서/DB 내용 모름

## 파이프라인
```
[인덱싱 단계]
문서 → 청크 분할 → 임베딩 → 벡터DB 저장

[쿼리 단계]
질문 → 임베딩 → 유사 청크 검색 → LLM 컨텍스트에 삽입 → 답변
```

## 청크 분할 전략

| 방식 | 설명 |
|------|------|
| 고정 크기 | 500토큰씩 자르기 |
| 문장 기반 | 문장 단위로 자르기 |
| 의미 기반 | 주제 바뀌는 지점에서 자르기 |

- 청크가 너무 작으면: 문맥 손실
- 청크가 너무 크면: 검색 정확도 저하
- 보통 200~500 토큰, overlap 50~100 토큰

## 벡터DB

| DB | 특징 |
|----|------|
| Chroma | 가볍고 로컬용, 입문에 좋음 |
| Pinecone | 클라우드, 관리형 |
| Weaviate | 오픈소스, 다기능 |
| FAISS | Meta, 대규모 처리 빠름 |

## RAG vs 파인튜닝

| 기준 | RAG | 파인튜닝 |
|------|-----|----------|
| 최신 정보 | O | X |
| 출처 명시 | O | 어려움 |
| 구현 난이도 | 낮음 | 높음 |
| 비용 | 낮음 | 높음 |
| 모델 행동 변경 | X | O |

## 코드 예시 (간단 버전)
```python
# 1. 문서 임베딩
chunks = split_documents(documents)
vectors = [embed(chunk) for chunk in chunks]
vector_db.add(vectors, chunks)

# 2. 검색 + 생성
query_vector = embed(question)
relevant_chunks = vector_db.search(query_vector, top_k=3)

prompt = f"""
다음 문서를 참고해서 질문에 답해주세요.

문서:
{relevant_chunks}

질문: {question}
"""

answer = llm.generate(prompt)
```