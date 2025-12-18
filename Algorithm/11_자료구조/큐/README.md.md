# 큐 (Queue)

## 큐란?

**FIFO** (First In, First Out) - 선입선출 먼저 넣은 것이 먼저 나온다.

```
offer(1) → [1]
offer(2) → [1, 2]
offer(3) → [1, 2, 3]
poll()   → 1 반환, [2, 3]
poll()   → 2 반환, [3]
```

---

## 실생활 예시

- 줄 서기
- 프린터 대기열
- 은행 번호표

---

## Java 구현

### LinkedList 사용

```java
import java.util.LinkedList;
import java.util.Queue;

Queue<Integer> queue = new LinkedList<>();

queue.offer(1);     // 삽입 (또는 add)
queue.offer(2);
queue.offer(3);

queue.peek();       // 1 (조회, 제거 안 함)
queue.poll();       // 1 (제거 + 반환)
queue.isEmpty();    // false
queue.size();       // 2
```

### ArrayDeque 사용 (권장)

```java
import java.util.ArrayDeque;
import java.util.Deque;

Deque<Integer> queue = new ArrayDeque<>();

queue.offer(1);     // 뒤에 삽입
queue.peek();       // 앞 조회
queue.poll();       // 앞에서 제거
queue.isEmpty();    // 비어있는지
```

> 💡 LinkedList보다 ArrayDeque가 더 빠름

---

## 주요 메서드

|메서드|설명|시간복잡도|
|---|---|---|
|offer(x)|뒤에 삽입|O(1)|
|poll()|앞에서 제거 + 반환|O(1)|
|peek()|앞 조회 (제거 안 함)|O(1)|
|isEmpty()|비어있는지 확인|O(1)|
|size()|크기 반환|O(1)|

### add vs offer / remove vs poll

||실패 시|
|---|---|
|add()|예외 발생|
|offer()|false 반환|
|remove()|예외 발생|
|poll()|null 반환|

> 💡 offer/poll 사용 권장 (예외 처리 필요 없음)

---

## 언제 큐를 쓰는가?

- **순서대로 처리**: 먼저 온 순서대로
- **BFS**: 너비 우선 탐색
- **대기열 시뮬레이션**: 줄 서기, 프로세스 스케줄링

---

## 활용 예제: 카드 게임

```java
// 1~n 카드, 맨 위 버리고 그 다음 맨 아래로
public int lastCard(int n) {
    Deque<Integer> queue = new ArrayDeque<>();
    
    // 1~n 카드 넣기
    for (int i = 1; i <= n; i++) {
        queue.offer(i);
    }
    
    // 한 장 남을 때까지
    while (queue.size() > 1) {
        queue.poll();              // 맨 위 버리기
        queue.offer(queue.poll()); // 다음 카드 맨 아래로
    }
    
    return queue.poll();
}

// n=4: [1,2,3,4] → [3,4,2] → [2,4] → [4] → 4
```

---

## 자주 하는 실수

- `poll()` 전에 `isEmpty()` 체크 안 함
- 스택과 혼동 (스택은 LIFO, 큐는 FIFO)
- `add()`와 `offer()` 차이 모름

---

## 정리

> 💡 큐 = 선입선출 (FIFO) 💡 순서대로 처리, BFS에 사용 💡 구현은 ArrayDeque 권장