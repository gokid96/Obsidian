
# 스택 (Stack)

## 스택이란?
**LIFO** (Last In, First Out) - 후입선출  
**마지막에 넣은 것이 먼저 나온다.**

```java
push(1) → [1]
push(2) → [1, 2]
push(3) → [1, 2, 3]
pop()   → 3 반환, [1, 2]  ← 마지막에 넣은 3이 먼저 나옴
pop()   → 2 반환, [1]
```

### 실생활 예시

- 접시 쌓기 (위에 쌓고, 위에서 꺼냄)
- 브라우저 뒤로 가기
- Ctrl+Z (실행 취소)

---

## Java 구현

### ArrayDeque 사용 (권장)


```java
import java.util.ArrayDeque;
import java.util.Deque;

Deque<Integer> stack = new ArrayDeque<>();

stack.push(1);      // 삽입
stack.push(2);
stack.push(3);      // 현재: [1, 2, 3] (3이 top)

stack.peek();       // 3 반환 (조회만, 제거 안 함)
stack.pop();        // 3 반환 + 제거, 현재: [1, 2]

stack.isEmpty();    // false
stack.size();       // 2
```

### Stack 클래스 (옛날 방식)

```java
import java.util.Stack;

Stack<Integer> stack = new Stack<>();
// 사용법은 동일
```

> 💡 **Stack 클래스보다 ArrayDeque가 더 빠름** (동기화 오버헤드 없음)

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

1. **괄호 검사**: 여는 괄호 push, 닫는 괄호 만나면 pop
2. **역순 처리**: 넣은 순서 반대로 꺼내야 할 때
3. **DFS (깊이 우선 탐색)**: 재귀 대신 스택으로 구현
4. **수식 계산**: 후위 표기법 계산

---

## 활용 예제: 괄호 검사

```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    
    for (char c : s.toCharArray()) {
        if (c == '(') {
            stack.push(c);  // 여는 괄호는 push
        } else if (c == ')') {
            if (stack.isEmpty()) return false;  // 짝이 없음
            stack.pop();  // 짝 맞으면 pop
        }
    }
    
    return stack.isEmpty();  // 남은 게 없어야 함
}

// "(())" → true
// "(()"  → false (닫는 괄호 부족)
// "())(" → false (여는 괄호 부족)
```

---

## 자주 하는 실수

```java
// ❌ pop() 전에 isEmpty() 체크 안 함
stack.pop();  // 비어있으면 예외 발생!

// ✅ 항상 체크
if (!stack.isEmpty()) {
    stack.pop();
}
```

---

## 정리

> 💡 스택 = **후입선출 (LIFO)** - 마지막에 넣은 게 먼저 나옴  
> 💡 괄호 검사, 역순 처리, DFS에 사용  
> 💡 구현은 **ArrayDeque 권장**  
> 💡 pop() 전에 **isEmpty() 체크** 필수