## 정의
입력/출력에서 유해 콘텐츠를 감지하는 API

## 왜 필요한가
- LLM이 부적절한 콘텐츠 생성할 수 있음
- 사용자가 악의적 입력을 넣을 수 있음
- 서비스 출시 전 안전장치 필수

## OpenAI Moderation API

### 감지 카테고리
| 카테고리 | 설명 |
|----------|------|
| hate | 혐오 발언 |
| harassment | 괴롭힘 |
| self-harm | 자해 |
| sexual | 성적 콘텐츠 |
| violence | 폭력 |

### 코드 예시
```python
from openai import OpenAI

client = OpenAI()

response = client.moderations.create(
    input="검사할 텍스트"
)

result = response.results[0]

# 유해 여부
print(result.flagged)  # True/False

# 카테고리별 점수
print(result.category_scores)
# {'hate': 0.0001, 'violence': 0.002, ...}
```

## 적용 패턴

### 입력 필터링
```python
def check_input(user_message):
    response = client.moderations.create(input=user_message)
    if response.results[0].flagged:
        return "부적절한 내용이 감지되었습니다."
    return None  # 통과

# 사용
error = check_input(user_input)
if error:
    return error
else:
    # LLM 호출
```

### 출력 필터링
```python
def safe_generate(prompt):
    # 1. 입력 체크
    if client.moderations.create(input=prompt).results[0].flagged:
        return "입력이 부적절합니다."
    
    # 2. LLM 생성
    response = client.chat.completions.create(...)
    output = response.choices[0].message.content
    
    # 3. 출력 체크
    if client.moderations.create(input=output).results[0].flagged:
        return "응답을 생성할 수 없습니다."
    
    return output
```

## Guardrails 개념
Moderation보다 넓은 개념:
- 유해 콘텐츠 필터링
- 프롬프트 인젝션 방지
- 주제 이탈 방지
- 개인정보 마스킹
- 출력 형식 검증

### 간단한 Guardrail 예시
```python
BLOCKED_TOPICS = ["정치", "종교", "경쟁사"]

def topic_guard(text):
    for topic in BLOCKED_TOPICS:
        if topic in text:
            return f"{topic} 관련 질문은 답변드리기 어렵습니다."
    return None
```

## 실무 체크리스트
- [ ] 입력 Moderation
- [ ] 출력 Moderation
- [ ] 프롬프트 인젝션 방지
- [ ] 민감 주제 필터링
- [ ] 로깅 및 모니터링