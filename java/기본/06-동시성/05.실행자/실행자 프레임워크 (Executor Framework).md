## Executor란?

- **스레드 풀** 관리
- 스레드 생성/소멸 비용 절감
- java.util.concurrent 패키지

---

## 직접 생성 vs Executor
```java
// ❌ 직접 생성 - 비효율, 관리 어려움
for (int i = 0; i < 1000; i++) {
    new Thread(() -> doTask()).start();
}

// ✅ Executor - 재사용, 효율적
ExecutorService executor = Executors.newFixedThreadPool(10);
for (int i = 0; i < 1000; i++) {
    executor.submit(() -> doTask());
}
executor.shutdown();
```

---

## Executor 계층
```
Executor (execute)
    ↓
ExecutorService (submit, shutdown)
    ↓
ScheduledExecutorService (schedule)
```

---

## ExecutorService 생성
```java
// 고정 크기 (CPU 바운드 작업)
ExecutorService fixed = Executors.newFixedThreadPool(10);

// 캐시 (IO 바운드, 짧은 작업)
// 필요시 생성, 60초 유휴시 제거
ExecutorService cached = Executors.newCachedThreadPool();

// 단일 스레드 (순차 실행)
ExecutorService single = Executors.newSingleThreadExecutor();

// Work Stealing (Java 8+, CPU 코어 활용)
ExecutorService workStealing = Executors.newWorkStealingPool();
```

---

## execute vs submit
```java
// execute - 반환값 없음
executor.execute(() -> {
    System.out.println("실행");
});

// submit - Future 반환
Future<?> future = executor.submit(() -> {
    System.out.println("실행");
});

Future<String> future = executor.submit(() -> {
    return "결과";
});
```

---

## Runnable vs Callable
```java
// Runnable - 반환값 X, 예외 X
Runnable task = () -> {
    System.out.println("작업");
};

// Callable - 반환값 O, 예외 O
Callable<Integer> task = () -> {
    return 42;
};
```

---

## Future

### 비동기 결과
```java
ExecutorService executor = Executors.newFixedThreadPool(2);

Future<String> future = executor.submit(() -> {
    Thread.sleep(2000);
    return "완료!";
});

// 다른 작업 수행 가능
System.out.println("다른 작업...");

// 결과 대기
String result = future.get();  // 블로킹
System.out.println(result);
```

### 주요 메서드
```java
future.get();                         // 결과 대기 (블로킹)
future.get(5, TimeUnit.SECONDS);      // 타임아웃
future.isDone();                      // 완료 여부
future.isCancelled();                 // 취소 여부
future.cancel(true);                  // 취소 (true: 인터럽트)
```

---

## 종료
```java
executor.shutdown();      // 새 작업 거부, 기존 완료
executor.shutdownNow();   // 즉시 종료 시도

// 권장 패턴
executor.shutdown();
try {
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow();
        if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
            System.err.println("종료 실패!");
        }
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

> [!warning] shutdown() 호출 필수!
> 안 하면 JVM이 종료되지 않음

---

## ScheduledExecutorService

### 예약 실행
```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// 5초 후 실행
scheduler.schedule(() -> {
    System.out.println("5초 후 실행");
}, 5, TimeUnit.SECONDS);

// 주기적 실행 (시작 기준)
// 0초 후 시작, 이후 1초마다
scheduler.scheduleAtFixedRate(() -> {
    System.out.println("주기 작업");
}, 0, 1, TimeUnit.SECONDS);

// 주기적 실행 (종료 기준)
// 이전 작업 끝난 후 1초 뒤
scheduler.scheduleWithFixedDelay(() -> {
    System.out.println("딜레이 작업");
}, 0, 1, TimeUnit.SECONDS);
```

|메서드|설명|
|---|---|
|schedule|지연 후 1회 실행|
|scheduleAtFixedRate|시작 시간 기준 주기|
|scheduleWithFixedDelay|종료 후 딜레이|

---

## ThreadPoolExecutor

### 세밀한 설정
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                      // corePoolSize (기본 스레드)
    10,                     // maximumPoolSize (최대 스레드)
    60L,                    // keepAliveTime (유휴 시간)
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100),  // 작업 큐
    new ThreadPoolExecutor.CallerRunsPolicy()  // 거부 정책
);
```

### 거부 정책

|정책|동작|
|---|---|
|AbortPolicy|RejectedExecutionException (기본)|
|CallerRunsPolicy|호출 스레드가 직접 실행|
|DiscardPolicy|조용히 버림|
|DiscardOldestPolicy|가장 오래된 작업 버리고 재시도|
```java
// 거부 정책 예시
executor.setRejectedExecutionHandler(
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

---

## CompletableFuture (Java 8+)

### 비동기 체이닝
```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> {
        return "Hello";
    })
    .thenApply(s -> s + " World")    // 변환
    .thenApply(String::toUpperCase); // 변환

String result = cf.join();  // "HELLO WORLD"
```

### 주요 메서드
```java
// 생성
CompletableFuture.supplyAsync(() -> "결과");     // 반환값 O
CompletableFuture.runAsync(() -> doSomething()); // 반환값 X

// 변환
cf.thenApply(s -> s.toUpperCase());    // Function
cf.thenAccept(s -> System.out.println(s));  // Consumer
cf.thenRun(() -> System.out.println("완료")); // Runnable

// 조합
cf1.thenCombine(cf2, (r1, r2) -> r1 + r2);  // 둘 다 완료 후
CompletableFuture.allOf(cf1, cf2, cf3);      // 모두 완료
CompletableFuture.anyOf(cf1, cf2, cf3);      // 하나만 완료

// 예외 처리
cf.exceptionally(ex -> "기본값");
cf.handle((result, ex) -> {
    if (ex != null) return "에러";
    return result;
});
```

### 실행 스레드
```java
cf.thenApply(...)       // 이전과 같은 스레드
cf.thenApplyAsync(...)  // ForkJoinPool
cf.thenApplyAsync(..., executor)  // 지정 Executor
```

### 예제: 여러 API 호출
```java
CompletableFuture<String> user = CompletableFuture.supplyAsync(() -> 
    fetchUser(userId)
);
CompletableFuture<String> orders = CompletableFuture.supplyAsync(() -> 
    fetchOrders(userId)
);

// 둘 다 완료 후 합치기
CompletableFuture<String> result = user.thenCombine(orders, (u, o) -> 
    "User: " + u + ", Orders: " + o
);

System.out.println(result.join());
```

---

## 실전 예제

### 웹 크롤러
```java
ExecutorService executor = Executors.newFixedThreadPool(10);

List<String> urls = Arrays.asList(
    "https://example1.com",
    "https://example2.com",
    "https://example3.com"
);

List<Future<String>> futures = new ArrayList<>();
for (String url : urls) {
    Future<String> future = executor.submit(() -> fetchPage(url));
    futures.add(future);
}

// 결과 수집
for (Future<String> future : futures) {
    String content = future.get();
    System.out.println(content.length() + " bytes");
}

executor.shutdown();
```

### 배치 작업
```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

// 매일 자정 실행
scheduler.scheduleAtFixedRate(() -> {
    System.out.println("배치 작업 실행: " + LocalDateTime.now());
    // 일일 정산, 리포트 생성 등
}, 0, 24, TimeUnit.HOURS);
```

### 타임아웃 처리
```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Future<String> future = executor.submit(() -> {
    Thread.sleep(10000);  // 오래 걸리는 작업
    return "완료";
});

try {
    String result = future.get(3, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    System.out.println("타임아웃!");
    future.cancel(true);
}

executor.shutdown();
```

---

## 선택 가이드

|상황|선택|
|---|---|
|CPU 바운드|FixedThreadPool (코어 수)|
|IO 바운드|CachedThreadPool|
|순차 실행|SingleThreadExecutor|
|예약/주기 작업|ScheduledThreadPool|
|비동기 체이닝|CompletableFuture|

---

> [!tip] 핵심 정리
> 
> - **ExecutorService**: 스레드 풀 관리
> - **submit()**: Future로 결과 받기
> - **shutdown()**: 반드시 호출!
> - **ScheduledExecutorService**: 예약/주기 작업
> - **CompletableFuture**: 비동기 체이닝

---

#Java #Executor #ThreadPool #CompletableFuture #동시성