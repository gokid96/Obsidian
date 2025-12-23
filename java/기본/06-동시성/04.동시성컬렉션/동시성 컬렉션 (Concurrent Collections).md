## 동시성 컬렉션이란?

- 멀티스레드 환경에서 **안전하게** 사용 가능
- java.util.concurrent 패키지

---

## 기존 방식의 문제
```java
// ❌ 일반 컬렉션 - 스레드 안전 X
Map<String, Integer> map = new HashMap<>();

// ❌ synchronizedMap - 성능 저하
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());

// ✅ ConcurrentHashMap - 효율적
Map<String, Integer> map = new ConcurrentHashMap<>();
```

---

## ConcurrentHashMap

### 기본 사용
```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// 기본 연산
map.put("a", 1);
map.get("a");
map.remove("a");
```

### 원자적 연산
```java
// putIfAbsent - 없으면 추가
map.putIfAbsent("a", 1);

// computeIfAbsent - 없으면 계산 후 추가
map.computeIfAbsent("a", key -> expensiveCalculation(key));

// computeIfPresent - 있으면 업데이트
map.computeIfPresent("a", (key, val) -> val + 1);

// merge - 병합
map.merge("a", 1, Integer::sum);  // 있으면 더하고, 없으면 1
```

### 복합 연산 주의
```java
// ❌ 위험! (원자적 X)
if (!map.containsKey("a")) {
    map.put("a", 1);
}

// ✅ 안전
map.putIfAbsent("a", 1);

// ❌ 위험!
Integer val = map.get("a");
if (val != null) {
    map.put("a", val + 1);
}

// ✅ 안전
map.merge("a", 1, Integer::sum);
```

### 벌크 연산 (Java 8+)
```java
// forEach
map.forEach((k, v) -> System.out.println(k + "=" + v));

// search - 조건 만족하는 첫 번째 값
String found = map.search(1, (k, v) -> v > 100 ? k : null);

// reduce - 집계
int sum = map.reduce(1, (k, v) -> v, Integer::sum);
```

---

## CopyOnWriteArrayList

### 쓰기 시 복사
```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

list.add("a");
list.get(0);
list.remove("a");
```

### 순회 중 수정 안전
```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("a");
list.add("b");

// 순회 중 수정해도 ConcurrentModificationException 없음!
for (String s : list) {
    System.out.println(s);
    list.add("c");  // 현재 순회에 영향 X (스냅샷)
}
```

> [!warning] 쓰기 비용 높음
> 읽기 >>> 쓰기 일 때만 사용

---

## CopyOnWriteArraySet
```java
CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();
set.add("a");
set.contains("a");
```

---

## BlockingQueue

### 스레드 간 데이터 전달
```
생산자 → [BlockingQueue] → 소비자
```

### 주요 메서드

|동작|예외 발생|특수값 반환|블로킹|타임아웃|
|---|---|---|---|---|
|삽입|add(e)|offer(e)|**put(e)**|offer(e, time, unit)|
|제거|remove()|poll()|**take()**|poll(time, unit)|
|조회|element()|peek()|-|-|

---

### ArrayBlockingQueue
```java
// 고정 크기
BlockingQueue<String> queue = new ArrayBlockingQueue<>(100);

// 생산자
new Thread(() -> {
    for (int i = 0; i < 10; i++) {
        queue.put("item-" + i);  // 가득 차면 대기
        System.out.println("생산: item-" + i);
    }
}).start();

// 소비자
new Thread(() -> {
    for (int i = 0; i < 10; i++) {
        String item = queue.take();  // 비면 대기
        System.out.println("소비: " + item);
    }
}).start();
```

### LinkedBlockingQueue
```java
// 무제한 (주의: 메모리)
BlockingQueue<String> queue = new LinkedBlockingQueue<>();

// 제한
BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);
```

### PriorityBlockingQueue
```java
// 우선순위 큐
BlockingQueue<Integer> queue = new PriorityBlockingQueue<>();
queue.put(3);
queue.put(1);
queue.put(2);

queue.take();  // 1
queue.take();  // 2
queue.take();  // 3
```

### DelayQueue
```java
class DelayedTask implements Delayed {
    private final String name;
    private final long executeTime;
    
    public DelayedTask(String name, long delayMs) {
        this.name = name;
        this.executeTime = System.currentTimeMillis() + delayMs;
    }
    
    @Override
    public long getDelay(TimeUnit unit) {
        long remaining = executeTime - System.currentTimeMillis();
        return unit.convert(remaining, TimeUnit.MILLISECONDS);
    }
    
    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.getDelay(TimeUnit.MILLISECONDS),
                           o.getDelay(TimeUnit.MILLISECONDS));
    }
}

// 사용
DelayQueue<DelayedTask> queue = new DelayQueue<>();
queue.put(new DelayedTask("작업1", 3000));  // 3초 후
queue.put(new DelayedTask("작업2", 1000));  // 1초 후

DelayedTask task = queue.take();  // 1초 후 "작업2"
```

### SynchronousQueue
```java
// 버퍼 없음, 직접 전달
SynchronousQueue<String> queue = new SynchronousQueue<>();

// put()은 take() 호출될 때까지 블로킹
new Thread(() -> queue.put("data")).start();
new Thread(() -> System.out.println(queue.take())).start();
```

---

## ConcurrentLinkedQueue

### 논블로킹 큐
```java
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();

queue.offer("a");  // 추가
queue.poll();      // 제거 (없으면 null)
queue.peek();      // 조회 (제거 X)
```

---

## 비교 정리

|컬렉션|특징|용도|
|---|---|---|
|ConcurrentHashMap|세그먼트 락|범용 맵|
|CopyOnWriteArrayList|쓰기 시 복사|읽기 많은 리스트|
|ArrayBlockingQueue|고정 크기, 블로킹|생산자-소비자|
|LinkedBlockingQueue|가변 크기, 블로킹|생산자-소비자|
|ConcurrentLinkedQueue|논블로킹|고성능 큐|
|PriorityBlockingQueue|우선순위|스케줄링|
|DelayQueue|지연|예약 작업|

---

## 실전 예제

### 단어 빈도수
```java
ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();

// 여러 스레드에서 안전하게 카운트
void countWord(String word) {
    wordCount.merge(word, 1, Integer::sum);
}
```

### 로그 수집기
```java
class LogCollector {
    private final BlockingQueue<String> queue = new LinkedBlockingQueue<>();
    
    // 여러 스레드에서 로그 추가
    public void log(String message) {
        queue.offer(message);
    }
    
    // 단일 스레드에서 처리
    public void startProcessor() {
        new Thread(() -> {
            while (true) {
                String log = queue.take();
                writeToFile(log);
            }
        }).start();
    }
}
```

---

> [!tip] 핵심 정리
> 
> - **ConcurrentHashMap**: 원자적 메서드 사용 (putIfAbsent, merge)
> - **CopyOnWriteArrayList**: 읽기 많을 때
> - **BlockingQueue**: 생산자-소비자 패턴
> - **ConcurrentLinkedQueue**: 논블로킹 필요 시

---

#Java #동시성컬렉션 #ConcurrentHashMap #BlockingQueue