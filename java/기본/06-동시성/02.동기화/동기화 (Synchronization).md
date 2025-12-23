## 동기화란?

- **여러 스레드의 공유 자원 접근 제어**
- 데이터 일관성 보장
- 경쟁 상태(Race Condition) 방지

---

## 문제 상황: 경쟁 상태

```java
class Counter {
    private int count = 0;
    
    public void increment() {
        count++;  // 원자적이지 않음!
    }
    
    public int getCount() {
        return count;
    }
}

// 문제 발생
Counter counter = new Counter();

Thread t1 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) {
        counter.increment();
    }
});

Thread t2 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) {
        counter.increment();
    }
});

t1.start();
t2.start();
t1.join();
t2.join();

System.out.println(counter.getCount());  // 20000이 아닐 수 있음!
```

---

## synchronized 키워드

### 메서드 동기화

```java
class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```

### 블록 동기화

```java
class Counter {
    private int count = 0;
    private final Object lock = new Object();
    
    public void increment() {
        synchronized (lock) {
            count++;
        }
    }
    
    public void decrement() {
        synchronized (lock) {
            count--;
        }
    }
}
```

### this 동기화

```java
public void method() {
    synchronized (this) {
        // 임계 영역
    }
}
```

### 클래스 동기화 (static)

```java
class Counter {
    private static int count = 0;
    
    public static synchronized void increment() {
        count++;
    }
    
    // 또는
    public static void increment2() {
        synchronized (Counter.class) {
            count++;
        }
    }
}
```

---

## volatile 키워드

### 가시성 보장

```java
class Flag {
    private volatile boolean running = true;
    
    public void stop() {
        running = false;
    }
    
    public void run() {
        while (running) {
            // 작업
        }
    }
}
```

### volatile vs synchronized

|volatile|synchronized|
|---|---|
|가시성만 보장|가시성 + 원자성|
|단일 변수|코드 블록|
|락 없음|락 사용|
|빠름|느림|

```java
// volatile로 충분한 경우
volatile boolean flag;  // 단순 읽기/쓰기

// synchronized 필요한 경우
count++;  // 읽기-수정-쓰기 (원자적이지 않음)
```

---

## wait() / notify() / notifyAll()

### 스레드 간 통신

```java
class SharedBuffer {
    private int data;
    private boolean hasData = false;
    
    public synchronized void produce(int value) throws InterruptedException {
        while (hasData) {
            wait();  // 소비될 때까지 대기
        }
        data = value;
        hasData = true;
        System.out.println("생산: " + value);
        notify();  // 소비자 깨우기
    }
    
    public synchronized int consume() throws InterruptedException {
        while (!hasData) {
            wait();  // 생산될 때까지 대기
        }
        hasData = false;
        System.out.println("소비: " + data);
        notify();  // 생산자 깨우기
        return data;
    }
}
```

### 생산자-소비자 패턴

```java
SharedBuffer buffer = new SharedBuffer();

// 생산자
Thread producer = new Thread(() -> {
    for (int i = 1; i <= 5; i++) {
        buffer.produce(i);
    }
});

// 소비자
Thread consumer = new Thread(() -> {
    for (int i = 1; i <= 5; i++) {
        buffer.consume();
    }
});

producer.start();
consumer.start();
```

---

## 데드락 (Deadlock)

### 발생 조건

```java
Object lock1 = new Object();
Object lock2 = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock1) {
        System.out.println("T1: lock1 획득");
        Thread.sleep(100);
        synchronized (lock2) {  // lock2 대기
            System.out.println("T1: lock2 획득");
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock2) {
        System.out.println("T2: lock2 획득");
        Thread.sleep(100);
        synchronized (lock1) {  // lock1 대기 → 데드락!
            System.out.println("T2: lock1 획득");
        }
    }
});
```

### 방지 방법

```java
// 1. 락 순서 통일
Thread t1 = new Thread(() -> {
    synchronized (lock1) {
        synchronized (lock2) {
            // 작업
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock1) {  // lock1 먼저!
        synchronized (lock2) {
            // 작업
        }
    }
});

// 2. tryLock 사용 (Lock 인터페이스)
// 3. 타임아웃 설정
```

---

## 동기화 블록 범위

### 넓은 범위 (비효율)

```java
public synchronized void process() {
    // 동기화 불필요한 코드
    doSomething();
    
    // 동기화 필요한 코드
    count++;
    
    // 동기화 불필요한 코드
    doOther();
}
```

### 좁은 범위 (권장)

```java
public void process() {
    // 동기화 불필요
    doSomething();
    
    synchronized (this) {
        count++;  // 필요한 부분만 동기화
    }
    
    // 동기화 불필요
    doOther();
}
```

---

## 실전 예제

### 은행 계좌

```java
class BankAccount {
    private int balance;
    
    public BankAccount(int balance) {
        this.balance = balance;
    }
    
    public synchronized void deposit(int amount) {
        balance += amount;
        System.out.println("입금: " + amount + ", 잔액: " + balance);
    }
    
    public synchronized boolean withdraw(int amount) {
        if (balance >= amount) {
            balance -= amount;
            System.out.println("출금: " + amount + ", 잔액: " + balance);
            return true;
        }
        return false;
    }
    
    public synchronized int getBalance() {
        return balance;
    }
}
```

### 스레드 안전 싱글톤

```java
class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### 동기화된 카운터

```java
class SafeCounter {
    private int count = 0;
    private final Object lock = new Object();
    
    public void increment() {
        synchronized (lock) {
            count++;
        }
    }
    
    public void decrement() {
        synchronized (lock) {
            count--;
        }
    }
    
    public int getCount() {
        synchronized (lock) {
            return count;
        }
    }
}
```

---

## 동기화 문제점

### 성능 저하

- 락 획득/해제 오버헤드
- 스레드 대기 시간

### 해결책

- 동기화 범위 최소화
- Lock 인터페이스 사용 (다음 장)
- 동시성 컬렉션 사용
- Atomic 클래스 사용

---

> [!tip] 핵심 정리
> 
> - **synchronized**: 메서드 또는 블록 동기화
> - **volatile**: 가시성 보장 (원자성 X)
> - **wait()/notify()**: 스레드 간 통신
> - **데드락**: 락 순서 통일로 방지
> - 동기화 범위는 **최소화**
> - 필요시 Lock 인터페이스 사용

---

#Java #동기화 #Synchronized #동시성 #멀티스레딩