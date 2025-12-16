## 정의
RAG, 코드 실행 등을 쉽게 구현할 수 있는 OpenAI의 고수준 API

## 핵심 개념

| 개념 | 설명 |
|------|------|
| Assistant | 설정된 AI (시스템 프롬프트, 도구 등) |
| Thread | 대화 세션 |
| Message | 대화 메시지 |
| Run | Assistant가 Thread를 처리하는 실행 단위 |

## 지원 도구

| 도구 | 기능 |
|------|------|
| File Search | 파일 업로드 후 검색 (RAG) |
| Code Interpreter | Python 코드 실행, 파일 생성 |
| Function Calling | 외부 함수 호출 |

## RAG 구현 (File Search)
```python
from openai import OpenAI

client = OpenAI()

# 1. Assistant 생성 (File Search 활성화)
assistant = client.beta.assistants.create(
    name="문서 Q&A",
    instructions="업로드된 문서를 기반으로 답변해주세요.",
    model="gpt-4o",
    tools=[{"type": "file_search"}],
)

# 2. 파일 업로드
file = client.files.create(
    file=open("document.pdf", "rb"),
    purpose="assistants"
)

# 3. Vector Store 생성 및 연결
vector_store = client.beta.vector_stores.create(name="my_docs")
client.beta.vector_stores.files.create(
    vector_store_id=vector_store.id,
    file_id=file.id
)

# 4. Thread 생성 및 질문
thread = client.beta.threads.create()
client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="이 문서의 핵심 내용이 뭐야?"
)

# 5. 실행
run = client.beta.threads.runs.create_and_poll(
    thread_id=thread.id,
    assistant_id=assistant.id
)

# 6. 결과 확인
messages = client.beta.threads.messages.list(thread_id=thread.id)
print(messages.data[0].content[0].text.value)
```

## 장점
- 벡터DB 직접 관리 안 해도 됨
- 청킹, 임베딩 자동 처리
- 대화 히스토리 자동 관리

## 단점
- OpenAI 종속
- 커스터마이징 제한적
- 비용 (저장소, 검색 비용 별도)