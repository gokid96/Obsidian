
## 스레드란?

- **프로세스 내 실행 단위**
- 하나의 프로그램에서 **동시에 여러 작업** 수행
- 자원 공유로 효율적

---

## 프로세스 vs 스레드

|구분|프로세스|스레드|
|---|---|---|
|정의|실행 중인 프로그램|프로세스 내 실행 흐름|
|메모리|독립적|공유 (Heap)|
|통신|IPC 필요|직접 접근|
|생성 비용|높음|낮음|
|오류 영향|독립적|전체 영향|

---

## 스레드 생성

### 1. Thread 클래스 상속

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread: " + getName());
    }
}

// 사용
MyThread t = new MyThread();
t.start();  // run() 아님!
```

### 2. Runnable 인터페이스 구현 (권장)

```java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable 실행");
    }
}

// 사용
Thread t = new Thread(new MyRunnable());
t.start();
```

### 3. 람다 표현식

```java
Thread t = new Thread(() -> {
    System.out.println("람다 스레드");
});
t.start();

// 간단하게
new Thread(() -> System.out.println("Hello")).start();
```

---

## start() vs run()

```java
Thread t = new Thread(() -> {
    System.out.println("Thread: " + Thread.currentThread().getName());
});

t.run();    // main 스레드에서 실행 (새 스레드 X)
t.start();  // 새로운 스레드에서 실행 (O)
```

---

## 스레드 상태

```
NEW → RUNNABLE ↔ BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```

|상태|설명|
|---|---|
|NEW|생성됨, 아직 start() 안 함|
|RUNNABLE|실행 가능/실행 중|
|BLOCKED|락 획득 대기|
|WAITING|무기한 대기|
|TIMED_WAITING|시간 제한 대기|
|TERMINATED|종료됨|

```java
Thread t = new Thread(() -> {});
System.out.println(t.getState());  // NEW

t.start();
System.out.println(t.getState());  // RUNNABLE

// 종료 후
Thread.sleep(100);
System.out.println(t.getState());  // TERMINATED
```

---

## 주요 메서드

### sleep() - 일시 정지

```java
try {
    Thread.sleep(1000);  // 1초 대기
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

### join() - 종료 대기

```java
Thread t = new Thread(() -> {
    // 작업
});
t.start();
t.join();  // t가 끝날 때까지 대기
System.out.println("t 종료 후 실행");
```

### interrupt() - 인터럽트

```java
Thread t = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // 작업
    }
    System.out.println("인터럽트됨!");
});

t.start();
Thread.sleep(1000);
t.interrupt();  // 인터럽트 요청
```

### yield() - 양보

```java
Thread.yield();  // 다른 스레드에게 실행 양보
```

---

## 스레드 정보

```java
Thread t = Thread.currentThread();

t.getName();       // 스레드 이름
t.getId();         // 스레드 ID
t.getPriority();   // 우선순위 (1~10)
t.getState();      // 상태
t.isAlive();       // 실행 중인지
t.isDaemon();      // 데몬 스레드인지
```

---

## 스레드 우선순위

```java
Thread t = new Thread(() -> {});

t.setPriority(Thread.MIN_PRIORITY);   // 1
t.setPriority(Thread.NORM_PRIORITY);  // 5 (기본)
t.setPriority(Thread.MAX_PRIORITY);   // 10

// 우선순위는 힌트일 뿐, OS가 결정
```

---

## 데몬 스레드

### 백그라운드 스레드

- 모든 일반 스레드 종료 시 자동 종료
- GC, 모니터링 등에 사용

```java
Thread daemon = new Thread(() -> {
    while (true) {
        System.out.println("데몬 실행 중...");
        Thread.sleep(1000);
    }
});

daemon.setDaemon(true);  // start() 전에 설정
daemon.start();

// main 종료 시 데몬도 종료
```

---

## 스레드 그룹

```java
ThreadGroup group = new ThreadGroup("MyGroup");

Thread t1 = new Thread(group, () -> {}, "Thread-1");
Thread t2 = new Thread(group, () -> {}, "Thread-2");

t1.start();
t2.start();

System.out.println("활성 스레드: " + group.activeCount());
group.interrupt();  // 그룹 전체 인터럽트
```

---

## 실전 예제

### 병렬 다운로드

```java
class Downloader implements Runnable {
    private String url;
    
    Downloader(String url) {
        this.url = url;
    }
    
    @Override
    public void run() {
        System.out.println("다운로드 시작: " + url);
        try {
            Thread.sleep(2000);  // 다운로드 시뮬레이션
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("다운로드 완료: " + url);
    }
}

// 사용
String[] urls = {"file1.zip", "file2.zip", "file3.zip"};
List<Thread> threads = new ArrayList<>();

for (String url : urls) {
    Thread t = new Thread(new Downloader(url));
    threads.add(t);
    t.start();
}

// 모든 다운로드 완료 대기
for (Thread t : threads) {
    t.join();
}
System.out.println("모든 다운로드 완료!");
```

### 타이머

```java
Thread timer = new Thread(() -> {
    int count = 0;
    while (!Thread.currentThread().isInterrupted()) {
        System.out.println("경과 시간: " + count++ + "초");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            break;
        }
    }
});

timer.start();
Thread.sleep(5000);
timer.interrupt();  // 5초 후 종료
```

### 진행률 표시

```java
class ProgressTask implements Runnable {
    private int progress = 0;
    
    @Override
    public void run() {
        while (progress < 100) {
            progress += 10;
            System.out.println("진행률: " + progress + "%");
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public int getProgress() {
        return progress;
    }
}
```

---

## 주의사항

### 1. start()는 한 번만

```java
Thread t = new Thread(() -> {});
t.start();
// t.start();  // IllegalThreadStateException
```

### 2. run() 직접 호출 금지

```java
t.run();    // 새 스레드 아님!
t.start();  // 올바른 방법
```

### 3. 공유 자원 주의

```java
// 여러 스레드가 접근하면 문제 발생
int count = 0;  // 공유 변수

// → 동기화 필요 (다음 장에서)
```

---

> [!tip] 핵심 정리
> 
> - **Thread 생성**: extends Thread 또는 implements Runnable
> - **start()**: 새 스레드 시작 (run() 아님)
> - **sleep()**: 일시 정지
> - **join()**: 스레드 종료 대기
> - **interrupt()**: 인터럽트 요청
> - **데몬 스레드**: 백그라운드, 자동 종료

---

#Java #스레드 #Thread #동시성 #멀티스레딩