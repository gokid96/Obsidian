# 스택 (Stack)

## 스택이란?

**LIFO** (Last In, First Out) - 후입선출 마지막에 넣은 것이 먼저 나온다.

```
push(1) → [1]
push(2) → [1, 2]
push(3) → [1, 2, 3]
pop()   → 3 반환, [1, 2]
pop()   → 2 반환, [1]
```

---

## 실생활 예시

- 접시 쌓기
- 뒤로 가기 버튼
- Ctrl+Z (실행 취소)

---

## Java 구현

### Stack 클래스

```java
import java.util.Stack;

Stack<Integer> stack = new Stack<>();

stack.push(1);      // 삽입
stack.push(2);
stack.push(3);

stack.peek();       // 3 (조회, 제거 안 함)
stack.pop();        // 3 (제거 + 반환)
stack.isEmpty();    // false
stack.size();       // 2
```

### ArrayDeque 사용 (권장)

```java
import java.util.ArrayDeque;
import java.util.Deque;

Deque<Integer> stack = new ArrayDeque<>();

stack.push(1);      // 삽입
stack.peek();       // 조회
stack.pop();        // 제거
stack.isEmpty();    // 비어있는지
```

> 💡 Stack 클래스보다 ArrayDeque가 더 빠름

---

## 주요 메서드

|메서드|설명|시간복잡도|
|---|---|---|
|push(x)|맨 위에 삽입|O(1)|
|pop()|맨 위 제거 + 반환|O(1)|
|peek()|맨 위 조회 (제거 안 함)|O(1)|
|isEmpty()|비어있는지 확인|O(1)|
|size()|크기 반환|O(1)|

---

## 언제 스택을 쓰는가?

- **괄호 검사**: 여는 괄호 push, 닫는 괄호 pop
- **역순 처리**: 넣은 순서 반대로 꺼내야 할 때
- **DFS**: 깊이 우선 탐색
- **재귀 → 반복문 변환**: 재귀 호출 스택 대체

---

## 활용 예제: 괄호 검사

```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    
    for (char c : s.toCharArray()) {
        if (c == '(') {
            stack.push(c);
        } else if (c == ')') {
            if (stack.isEmpty()) return false;
            stack.pop();
        }
    }
    
    return stack.isEmpty();
}

// "(())" → true
// "(()"  → false
// "())(" → false
```

---

## 자주 하는 실수

- `pop()` 전에 `isEmpty()` 체크 안 함 → EmptyStackException
- `peek()`과 `pop()` 헷갈림 → peek은 제거 안 함

---

## 정리

> 💡 스택 = 후입선출 (LIFO) 💡 괄호 검사, 역순 처리, DFS에 사용 💡 구현은 ArrayDeque 권장