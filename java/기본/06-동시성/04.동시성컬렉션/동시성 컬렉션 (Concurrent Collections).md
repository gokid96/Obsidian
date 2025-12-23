## 동시성 컬렉션이란?

- **스레드 안전한 컬렉션**
- synchronized 컬렉션보다 성능 우수
- `java.util.concurrent` 패키지

---

## 기존 방식의 문제

### synchronized 래퍼

```java
// Collections.synchronizedXxx() - 전체 락
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());

// 문제: 전체 컬렉션 락 → 성능 저하
```

### Vector, Hashtable

```java
// 레거시 - 모든 메서드 synchronized
Vector<String> vector = new Vector<>();
Hashtable<String, Integer> hashtable = new Hashtable<>();

// 문제: 불필요한 동기화, 성능 저하
```

---

## ConcurrentHashMap

### 특징

- **세그먼트 단위 락** (세밀한 동기화)
- null 키/값 불허
- 높은 동시성

```java
import java.util.concurrent.ConcurrentHashMap;

ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// 기본 연산
map.put("A", 1);
map.get("A");
map.remove("A");
```

### 원자적 연산

```java
// putIfAbsent - 없을 때만 추가
map.putIfAbsent("A", 1);

// computeIfAbsent - 없을 때만 계산 후 추가
map.computeIfAbsent("B", key -> expensiveComputation(key));

// computeIfPresent - 있을 때만 계산
map.computeIfPresent("A", (key, value) -> value + 1);

// compute - 항상 계산
map.compute("A", (key, value) -> value == null ? 1 : value + 1);

// merge - 병합
map.merge("A", 1, Integer::sum);  // 없으면 1, 있으면 기존값 + 1
```

### 대량 연산 (Java 8+)

```java
// forEach
map.forEach((key, value) -> System.out.println(key + "=" + value));

// search - 조건 만족하는 값 찾기
String result = map.search(1, (key, value) -> value > 100 ? key : null);

// reduce - 축소
int sum = map.reduce(1, (key, value) -> value, Integer::sum);
```

---

## CopyOnWriteArrayList

### 특징

- **쓰기 시 복사** (읽기 최적화)
- 읽기 락 없음
- 쓰기 느림, 읽기 빠름
- 이벤트 리스너 목록 등에 적합

```java
import java.util.concurrent.CopyOnWriteArrayList;

CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

list.add("A");
list.add("B");

// 순회 중 수정 가능 (ConcurrentModificationException X)
for (String item : list) {
    list.add("C");  // 복사본에 추가
}
```

### 사용 시점

```java
// 적합: 읽기 많고, 쓰기 적음
CopyOnWriteArrayList<EventListener> listeners = new CopyOnWriteArrayList<>();

// 부적합: 쓰기 많음 → 일반 List + 동기화
```

---

## CopyOnWriteArraySet

### 특징

- CopyOnWriteArrayList 기반
- 중복 불허

```java
import java.util.concurrent.CopyOnWriteArraySet;

CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();
set.add("A");
set.add("A");  // 무시됨
```

---

## ConcurrentSkipListMap

### 특징

- **정렬된** 동시성 Map
- TreeMap의 동시성 버전
- O(log n) 연산

```java
import java.util.concurrent.ConcurrentSkipListMap;

ConcurrentSkipListMap<String, Integer> map = new ConcurrentSkipListMap<>();
map.put("B", 2);
map.put("A", 1);
map.put("C", 3);

System.out.println(map);  // {A=1, B=2, C=3} - 정렬됨

// NavigableMap 메서드
map.firstKey();           // A
map.lastKey();            // C
map.headMap("B");         // {A=1}
map.tailMap("B");         // {B=2, C=3}
```

---

## ConcurrentSkipListSet

```java
import java.util.concurrent.ConcurrentSkipListSet;

ConcurrentSkipListSet<Integer> set = new ConcurrentSkipListSet<>();
set.add(3);
set.add(1);
set.add(2);

System.out.println(set);  // [1, 2, 3] - 정렬됨
```

---

## BlockingQueue

### 특징

- **블로킹 연산** 지원
- 생산자-소비자 패턴에 최적

### 주요 메서드

|연산|예외 발생|특수 값|블로킹|타임아웃|
|---|---|---|---|---|
|삽입|add(e)|offer(e)|put(e)|offer(e, time, unit)|
|삭제|remove()|poll()|take()|poll(time, unit)|
|조회|element()|peek()|-|-|

### ArrayBlockingQueue

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

// 고정 크기
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

// 생산자
queue.put("item");  // 가득 차면 대기

// 소비자
String item = queue.take();  // 비어있으면 대기
```

### LinkedBlockingQueue

```java
import java.util.concurrent.LinkedBlockingQueue;

// 무제한 (또는 크기 지정)
BlockingQueue<String> queue = new LinkedBlockingQueue<>();
BlockingQueue<String> bounded = new LinkedBlockingQueue<>(100);
```

### 생산자-소비자 패턴

```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

// 생산자
Thread producer = new Thread(() -> {
    for (int i = 0; i < 100; i++) {
        queue.put(i);
        System.out.println("생산: " + i);
    }
});

// 소비자
Thread consumer = new Thread(() -> {
    while (true) {
        Integer item = queue.take();
        System.out.println("소비: " + item);
    }
});

producer.start();
consumer.start();
```

---

## PriorityBlockingQueue

### 우선순위 + 블로킹

```java
import java.util.concurrent.PriorityBlockingQueue;

PriorityBlockingQueue<Integer> pq = new PriorityBlockingQueue<>();

pq.put(3);
pq.put(1);
pq.put(2);

pq.take();  // 1 (최소값 먼저)
pq.take();  // 2
pq.take();  // 3
```

---

## DelayQueue

### 지연된 요소 처리

```java
import java.util.concurrent.Delayed;
import java.util.concurrent.DelayQueue;
import java.util.concurrent.TimeUnit;

class DelayedTask implements Delayed {
    private String name;
    private long startTime;
    
    public DelayedTask(String name, long delayMillis) {
        this.name = name;
        this.startTime = System.currentTimeMillis() + delayMillis;
    }
    
    @Override
    public long getDelay(TimeUnit unit) {
        long diff = startTime - System.currentTimeMillis();
        return unit.convert(diff, TimeUnit.MILLISECONDS);
    }
    
    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.startTime, ((DelayedTask) o).startTime);
    }
}

DelayQueue<DelayedTask> queue = new DelayQueue<>();
queue.put(new DelayedTask("Task1", 3000));  // 3초 후 처리 가능
queue.put(new DelayedTask("Task2", 1000));  // 1초 후 처리 가능

queue.take();  // Task2 (1초 대기 후)
queue.take();  // Task1 (추가 2초 대기 후)
```

---

## 선택 가이드

|상황|컬렉션|
|---|---|
|일반 Map|ConcurrentHashMap|
|정렬된 Map|ConcurrentSkipListMap|
|읽기 많은 List|CopyOnWriteArrayList|
|생산자-소비자|BlockingQueue|
|우선순위 큐|PriorityBlockingQueue|
|지연 처리|DelayQueue|

---

> [!tip] 핵심 정리
> 
> - **ConcurrentHashMap**: 고성능 동시성 Map
> - **CopyOnWriteArrayList**: 읽기 최적화 List
> - **BlockingQueue**: 생산자-소비자 패턴
> - **put/take**: 블로킹, **offer/poll**: 논블로킹
> - synchronized 래퍼보다 성능 우수
> - null 보통 불허

---

#Java #동시성컬렉션 #ConcurrentHashMap #BlockingQueue