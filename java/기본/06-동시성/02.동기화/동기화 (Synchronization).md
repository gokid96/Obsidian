## 동기화란?

- 여러 스레드가 **공유 자원에 동시 접근**할 때 발생하는 문제 방지
- **데이터 일관성** 보장

---

## 경쟁 상태 (Race Condition)
```java
class Counter {
    private int count = 0;
    
    public void increment() {
        count++;  // 읽기 → 증가 → 쓰기 (3단계, 원자적 X)
    }
}

// 여러 스레드가 동시에 increment() 호출
// → 예상: 10000, 실제: 9876 (매번 다름)
```

---

## synchronized

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
    
    public void doSomething() {
        // 동기화 불필요한 코드
        System.out.println("시작");
        
        synchronized (this) {
            // 동기화 필요한 부분만
            count++;
        }
        
        // 동기화 불필요한 코드
        System.out.println("끝");
    }
}
```

### static 동기화
```java
class Counter {
    private static int count = 0;
    
    // 클래스 락 (Counter.class)
    public static synchronized void increment() {
        count++;
    }
    
    // 동일
    public static void increment2() {
        synchronized (Counter.class) {
            count++;
        }
    }
}
```

---

## volatile

### 가시성 문제
```java
class Task {
    private boolean running = true;
    
    public void stop() {
        running = false;  // 스레드 1에서 변경
    }
    
    public void run() {
        while (running) {  // 스레드 2에서 읽기
            // CPU 캐시에서 읽어서 변경 감지 못할 수 있음!
        }
    }
}
```

### volatile로 해결
```java
class Task {
    private volatile boolean running = true;  // 메인 메모리에서 읽기/쓰기
    
    public void stop() {
        running = false;
    }
    
    public void run() {
        while (running) {
            // 항상 최신 값 읽음
        }
    }
}
```

> [!warning] volatile 주의
> 가시성만 보장, **원자성은 X**
> `count++`는 여전히 위험!

---

## wait() / notify()

### 스레드 간 협력
```java
class Buffer {
    private int data;
    private boolean hasData = false;
    
    public synchronized void produce(int value) throws InterruptedException {
        while (hasData) {
            wait();  // 소비될 때까지 대기, 락 해제
        }
        data = value;
        hasData = true;
        System.out.println("생산: " + value);
        notify();  // 대기 중인 스레드 깨움
    }
    
    public synchronized int consume() throws InterruptedException {
        while (!hasData) {
            wait();  // 생산될 때까지 대기
        }
        hasData = false;
        System.out.println("소비: " + data);
        notify();
        return data;
    }
}
```

### 사용 예시
```java
Buffer buffer = new Buffer();

// 생산자
new Thread(() -> {
    for (int i = 0; i < 5; i++) {
        buffer.produce(i);
    }
}).start();

// 소비자
new Thread(() -> {
    for (int i = 0; i < 5; i++) {
        buffer.consume();
    }
}).start();
```

|메서드|설명|
|---|---|
|wait()|락 해제 후 대기|
|wait(ms)|시간 제한 대기|
|notify()|대기 스레드 1개 깨움|
|notifyAll()|대기 스레드 전부 깨움|

> [!caution] synchronized 블록 안에서만 호출 가능

---

## 데드락 (Deadlock)

### 교착 상태
```java
Object lockA = new Object();
Object lockB = new Object();

// 스레드 1
new Thread(() -> {
    synchronized (lockA) {
        Thread.sleep(100);
        synchronized (lockB) {  // B 대기
            // 작업
        }
    }
}).start();

// 스레드 2
new Thread(() -> {
    synchronized (lockB) {
        Thread.sleep(100);
        synchronized (lockA) {  // A 대기
            // 작업
        }
    }
}).start();

// → 서로 상대방 락 기다리며 영원히 대기!
```

### 해결 방법
```java
// 락 순서 통일
synchronized (lockA) {
    synchronized (lockB) {
        // 항상 A → B 순서
    }
}
```

---

## 실전 예제

### 은행 계좌
```java
class Account {
    private int balance;
    
    public Account(int balance) {
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
        System.out.println("잔액 부족!");
        return false;
    }
    
    public synchronized int getBalance() {
        return balance;
    }
}
```

### 티켓 예매
```java
class TicketOffice {
    private int tickets = 100;
    
    public synchronized boolean book(String name) {
        if (tickets > 0) {
            tickets--;
            System.out.println(name + " 예매 성공! 남은 티켓: " + tickets);
            return true;
        }
        System.out.println(name + " 예매 실패! 매진");
        return false;
    }
}

// 여러 스레드에서 동시 예매 시도
TicketOffice office = new TicketOffice();
for (int i = 0; i < 10; i++) {
    int num = i;
    new Thread(() -> office.book("고객" + num)).start();
}
```

---

## 주의사항

### 락 객체
```java
// ❌ String 리터럴 금지
synchronized ("lock") { }  // 다른 코드와 같은 객체일 수 있음!

// ❌ 락 객체 변경 금지
private Object lock = new Object();
lock = new Object();  // 위험!

// ✅ final 사용
private final Object lock = new Object();
```

### 동기화 범위
```java
// ❌ 너무 넓은 범위
public synchronized void process() {
    // 긴 작업... (다른 스레드 계속 대기)
}

// ✅ 필요한 부분만
public void process() {
    // 동기화 불필요한 작업
    synchronized (lock) {
        // 동기화 필요한 작업만
    }
    // 동기화 불필요한 작업
}
```

---

> [!tip] 핵심 정리
> 
> - **synchronized**: 한 번에 하나의 스레드만 접근
> - **volatile**: 변수 가시성 보장 (원자성 X)
> - **wait/notify**: 스레드 간 협력
> - **데드락**: 락 순서 통일로 방지
> - 동기화 범위는 **최소한**으로

---

#Java #동기화 #synchronized #volatile #멀티스레딩