## 정의
텍스트를 고차원 벡터(숫자 배열)로 변환

## 핵심 특성
- 의미가 비슷한 텍스트 → 벡터도 가까움
- 의미가 다른 텍스트 → 벡터도 멂

## 예시
```
"강아지" → [0.2, 0.8, 0.1, ...]
"개"     → [0.21, 0.79, 0.12, ...]  # 비슷함
"자동차" → [0.9, 0.1, 0.3, ...]     # 다름
```

## 유사도 계산
- 코사인 유사도 가장 많이 사용
- 1에 가까울수록 유사, 0에 가까울수록 다름
```python
from numpy import dot
from numpy.linalg import norm

def cosine_similarity(a, b):
    return dot(a, b) / (norm(a) * norm(b))
```

## 주요 임베딩 모델

| 모델 | 차원 | 특징 |
|------|------|------|
| text-embedding-3-small | 1536 | OpenAI, 가성비 |
| text-embedding-3-large | 3072 | OpenAI, 고성능 |
| BGE-M3 | 1024 | 오픈소스, 다국어 |

## 활용 사례
1. **시맨틱 검색**: 키워드 아닌 의미로 검색
2. **문서 분류**: 벡터 거리로 카테고리 분류
3. **클러스터링**: 비슷한 문서 그룹화
4. **추천**: 유사 콘텐츠 찾기
5. **중복 탐지**: 비슷한 텍스트 찾기

## 코드 예시 (OpenAI)
```python
from openai import OpenAI

client = OpenAI()

response = client.embeddings.create(
    input="안녕하세요",
    model="text-embedding-3-small"
)

vector = response.data[0].embedding  # 1536차원 벡터
```