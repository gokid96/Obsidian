## 실행자란?

- **스레드 풀 기반 작업 실행**
- 스레드 생성/관리 추상화
- 효율적인 스레드 재사용
- `java.util.concurrent` 패키지

---

## 직접 스레드 vs 실행자

### 직접 스레드 생성 (비효율)

```java
// 매번 새 스레드 생성
for (int i = 0; i < 1000; i++) {
    new Thread(() -> doTask()).start();
}
// 문제: 스레드 생성 비용, 자원 낭비
```

### 실행자 사용 (효율)

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

for (int i = 0; i < 1000; i++) {
    executor.submit(() -> doTask());
}
// 10개 스레드로 1000개 작업 처리

executor.shutdown();
```

---

## Executors 팩토리

### newFixedThreadPool

```java
// 고정 크기 스레드 풀
ExecutorService executor = Executors.newFixedThreadPool(4);
```

### newCachedThreadPool

```java
// 필요시 스레드 생성, 60초 미사용 시 제거
ExecutorService executor = Executors.newCachedThreadPool();
```

### newSingleThreadExecutor

```java
// 단일 스레드 (순차 실행)
ExecutorService executor = Executors.newSingleThreadExecutor();
```

### newScheduledThreadPool

```java
// 예약/주기적 실행
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
```

### newWorkStealingPool (Java 8+)

```java
// ForkJoinPool 기반, CPU 코어 수만큼
ExecutorService executor = Executors.newWorkStealingPool();
```

---

## ExecutorService 사용

### submit() - 작업 제출

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// Runnable (반환값 없음)
executor.submit(() -> System.out.println("Hello"));

// Callable (반환값 있음)
Future<Integer> future = executor.submit(() -> {
    return 42;
});

int result = future.get();  // 결과 대기
```

### execute() vs submit()

```java
// execute: 반환값 없음
executor.execute(() -> System.out.println("execute"));

// submit: Future 반환
Future<?> future = executor.submit(() -> System.out.println("submit"));
```

### 종료

```java
executor.shutdown();  // 새 작업 거부, 기존 작업 완료 후 종료

// 즉시 종료 시도
List<Runnable> pending = executor.shutdownNow();

// 종료 대기
if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
    executor.shutdownNow();
}
```

---

## Future

### 비동기 결과

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

Future<String> future = executor.submit(() -> {
    Thread.sleep(2000);
    return "결과";
});

// 완료 여부 확인
boolean done = future.isDone();

// 결과 대기 (블로킹)
String result = future.get();

// 타임아웃
String result2 = future.get(1, TimeUnit.SECONDS);

// 취소
boolean cancelled = future.cancel(true);
```

### 여러 Future

```java
List<Future<Integer>> futures = new ArrayList<>();

for (int i = 0; i < 10; i++) {
    final int num = i;
    Future<Integer> future = executor.submit(() -> num * num);
    futures.add(future);
}

// 모든 결과 수집
for (Future<Integer> future : futures) {
    System.out.println(future.get());
}
```

---

## invokeAll / invokeAny

### invokeAll - 모두 실행

```java
List<Callable<String>> tasks = Arrays.asList(
    () -> { Thread.sleep(1000); return "Task 1"; },
    () -> { Thread.sleep(2000); return "Task 2"; },
    () -> { Thread.sleep(500); return "Task 3"; }
);

List<Future<String>> results = executor.invokeAll(tasks);

for (Future<String> future : results) {
    System.out.println(future.get());
}
```

### invokeAny - 하나만 (가장 빠른 것)

```java
String fastest = executor.invokeAny(tasks);
System.out.println("가장 빠른 결과: " + fastest);
```

---

## ScheduledExecutorService

### 지연 실행

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// 3초 후 실행
scheduler.schedule(() -> System.out.println("3초 후 실행"), 
    3, TimeUnit.SECONDS);
```

### 주기적 실행

```java
// 1초 후 시작, 2초마다 실행
scheduler.scheduleAtFixedRate(
    () -> System.out.println("주기 실행"),
    1, 2, TimeUnit.SECONDS
);

// 이전 작업 종료 후 2초 뒤 실행
scheduler.scheduleWithFixedDelay(
    () -> System.out.println("딜레이 실행"),
    1, 2, TimeUnit.SECONDS
);
```

### FixedRate vs FixedDelay

```java
// scheduleAtFixedRate: 시작 시간 기준
// 작업: 0초, 2초, 4초, 6초...

// scheduleWithFixedDelay: 종료 시간 기준
// 작업 1초 걸리면: 0초, 3초, 6초, 9초...
```

---

## CompletableFuture (Java 8+)

### 비동기 실행

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // 비동기 작업
    return "결과";
});

String result = future.get();
```

### 체이닝

```java
CompletableFuture.supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World")      // 변환
    .thenApply(String::toUpperCase)
    .thenAccept(System.out::println);  // 소비
// 출력: HELLO WORLD
```

### 조합

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World");

// 둘 다 완료 후
CompletableFuture<String> combined = future1.thenCombine(future2, 
    (s1, s2) -> s1 + " " + s2);

// 둘 중 하나 완료 시
CompletableFuture<Object> any = CompletableFuture.anyOf(future1, future2);

// 모두 완료 대기
CompletableFuture<Void> all = CompletableFuture.allOf(future1, future2);
```

### 예외 처리

```java
CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("에러");
    return "결과";
})
.exceptionally(ex -> "기본값")  // 예외 시 대체값
.thenAccept(System.out::println);

// handle: 성공/실패 모두 처리
.handle((result, ex) -> {
    if (ex != null) return "에러 처리";
    return result;
});
```

---

## ThreadPoolExecutor

### 세밀한 제어

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                      // corePoolSize
    4,                      // maximumPoolSize
    60, TimeUnit.SECONDS,   // keepAliveTime
    new ArrayBlockingQueue<>(100),  // workQueue
    new ThreadPoolExecutor.CallerRunsPolicy()  // 거부 정책
);
```

### 거부 정책

```java
// AbortPolicy (기본): RejectedExecutionException 발생
// CallerRunsPolicy: 호출 스레드에서 실행
// DiscardPolicy: 조용히 버림
// DiscardOldestPolicy: 가장 오래된 작업 버리고 새 작업 추가
```

---

## 실전 예제

### 병렬 다운로드

```java
ExecutorService executor = Executors.newFixedThreadPool(5);

List<String> urls = Arrays.asList("url1", "url2", "url3", "url4", "url5");

List<Future<byte[]>> futures = urls.stream()
    .map(url -> executor.submit(() -> download(url)))
    .collect(Collectors.toList());

for (Future<byte[]> future : futures) {
    byte[] data = future.get();
    // 처리
}

executor.shutdown();
```

### 타임아웃 처리

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Future<String> future = executor.submit(() -> {
    Thread.sleep(5000);
    return "결과";
});

try {
    String result = future.get(2, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    future.cancel(true);
    System.out.println("타임아웃!");
}
```

---

> [!tip] 핵심 정리
> 
> - **ExecutorService**: 스레드 풀 관리
> - **newFixedThreadPool**: 고정 크기
> - **newCachedThreadPool**: 탄력적 크기
> - **Future**: 비동기 결과
> - **CompletableFuture**: 체이닝, 조합
> - **ScheduledExecutorService**: 예약/주기 실행
> - **반드시 shutdown() 호출**

---

#Java #Executor #스레드풀 #CompletableFuture #동시성