
## 정의
LLM이 "이 함수를 이 파라미터로 호출해"라고 JSON을 출력하는 기능

## 핵심
- LLM이 직접 함수를 실행하는 게 아님
- LLM은 어떤 함수를 어떤 인자로 호출할지 결정만 함
- 실제 실행은 개발자 코드가 담당

## 흐름
```
사용자: "서울 날씨 알려줘"
    ↓
LLM: {"name": "get_weather", "arguments": {"city": "서울"}}
    ↓
개발자 코드: get_weather("서울") 실행 → "맑음, 15도"
    ↓
LLM에 결과 전달
    ↓
LLM: "서울은 현재 맑고 15도입니다"
```

## 코드 예시
```python
from openai import OpenAI

client = OpenAI()

# 1. 함수 정의
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "특정 도시의 현재 날씨를 조회합니다",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "도시 이름"
                    }
                },
                "required": ["city"]
            }
        }
    }
]

# 2. 첫 번째 호출
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "서울 날씨 알려줘"}],
    tools=tools,
)

# 3. 함수 호출 감지 및 실행
tool_call = response.choices[0].message.tool_calls[0]
if tool_call.function.name == "get_weather":
    args = json.loads(tool_call.function.arguments)
    result = get_weather(args["city"])  # 실제 함수 실행

# 4. 결과 전달하여 최종 답변 생성
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "user", "content": "서울 날씨 알려줘"},
        response.choices[0].message,
        {"role": "tool", "tool_call_id": tool_call.id, "content": result}
    ],
    tools=tools,
)

print(response.choices[0].message.content)
```

## 활용 사례
- 외부 API 호출 (날씨, 주식, 검색)
- 데이터베이스 조회/수정
- 내부 시스템 연동
- 여러 함수 조합 (에이전트)

## 에이전트와의 관계
Function Calling = 에이전트의 기초
- 단일 함수: Function Calling
- 여러 함수 + 자율적 판단 + 루프: Agent