## Lock이란?

- `synchronized`보다 **유연한** 동기화
- java.util.concurrent.locks 패키지

---

## synchronized vs Lock

|구분|synchronized|Lock|
|---|---|---|
|락 해제|자동 (블록 종료)|수동 (unlock)|
|타임아웃|X|O (tryLock)|
|인터럽트|X|O (lockInterruptibly)|
|공정성|X|O (fair lock)|
|조건 변수|1개 (wait/notify)|여러 개 (Condition)|

---

## ReentrantLock

### 기본 사용
```java
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();  // 반드시 finally에서!
        }
    }
}
```

> [!danger] unlock()은 반드시 finally에서!
> 예외 발생 시에도 락 해제 보장

---

### tryLock() - 시도 후 포기
```java
if (lock.tryLock()) {
    try {
        // 작업
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("락 획득 실패, 다른 작업 수행");
}
```

### tryLock(timeout) - 시간 제한
```java
try {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            // 작업
        } finally {
            lock.unlock();
        }
    } else {
        System.out.println("1초 내 락 획득 실패");
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### lockInterruptibly() - 인터럽트 가능
```java
try {
    lock.lockInterruptibly();  // 대기 중 인터럽트 가능
    try {
        // 작업
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    System.out.println("대기 중 인터럽트됨");
}
```

---

## 공정성 (Fairness)
```java
// 비공정 락 (기본) - 성능 좋음
ReentrantLock unfairLock = new ReentrantLock();
ReentrantLock unfairLock = new ReentrantLock(false);

// 공정 락 - 대기 순서대로 획득
ReentrantLock fairLock = new ReentrantLock(true);
```

|구분|비공정|공정|
|---|---|---|
|순서|보장 X|FIFO|
|성능|빠름|느림|
|기아|발생 가능|방지|

---

## 주요 메서드

|메서드|설명|
|---|---|
|lock()|락 획득 (블로킹)|
|unlock()|락 해제|
|tryLock()|즉시 시도, 실패시 false|
|tryLock(time, unit)|시간 내 시도|
|lockInterruptibly()|인터럽트 가능한 락|
|isLocked()|락 상태|
|isHeldByCurrentThread()|현재 스레드가 보유?|
|getHoldCount()|재진입 횟수|

---

## ReadWriteLock

### 읽기/쓰기 분리

- **읽기**: 여러 스레드 동시 가능
- **쓰기**: 하나의 스레드만
```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class Cache {
    private final Map<String, String> map = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    
    // 읽기 - 동시 접근 가능
    public String get(String key) {
        rwLock.readLock().lock();
        try {
            return map.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }
    
    // 쓰기 - 독점
    public void put(String key, String value) {
        rwLock.writeLock().lock();
        try {
            map.put(key, value);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

> [!tip] 읽기 많고 쓰기 적을 때 성능 향상

---

## StampedLock (Java 8+)

### 낙관적 읽기
```java
import java.util.concurrent.locks.StampedLock;

class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();
    
    // 쓰기
    public void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }
    
    // 낙관적 읽기 (락 없이!)
    public double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();  // 락 X
        double currentX = x, currentY = y;
        
        if (!sl.validate(stamp)) {  // 그 사이 쓰기 발생?
            stamp = sl.readLock();  // 실패 시 진짜 락
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

|모드|설명|
|---|---|
|tryOptimisticRead()|락 없이 읽기 시도|
|validate(stamp)|쓰기 발생 여부 확인|
|readLock()|일반 읽기 락|
|writeLock()|쓰기 락|

---

## Condition

### wait/notify의 Lock 버전
```java
class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }
    
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // wait()
            }
            queue.add(item);
            notEmpty.signal();    // notify()
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            T item = queue.poll();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### Object vs Condition

|Object|Condition|
|---|---|
|wait()|await()|
|wait(ms)|await(time, unit)|
|notify()|signal()|
|notifyAll()|signalAll()|

> [!tip] Condition은 여러 개 생성 가능 → 세밀한 제어

---

## 실전 예제

### 캐시 with ReadWriteLock
```java
class SimpleCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public V get(K key) {
        lock.readLock().lock();
        try {
            return cache.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public void put(K key, V value) {
        lock.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public V computeIfAbsent(K key, Function<K, V> loader) {
        // 먼저 읽기 시도
        lock.readLock().lock();
        try {
            V value = cache.get(key);
            if (value != null) return value;
        } finally {
            lock.readLock().unlock();
        }
        
        // 없으면 쓰기
        lock.writeLock().lock();
        try {
            // 다시 확인 (다른 스레드가 넣었을 수 있음)
            V value = cache.get(key);
            if (value == null) {
                value = loader.apply(key);
                cache.put(key, value);
            }
            return value;
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

> [!tip] 핵심 정리
> 
> - **ReentrantLock**: synchronized 대체, 더 유연
> - **tryLock**: 락 획득 시도 (타임아웃)
> - **ReadWriteLock**: 읽기/쓰기 분리
> - **StampedLock**: 낙관적 읽기 (고성능)
> - **Condition**: 여러 조건 변수

---

#Java #Lock #ReentrantLock #ReadWriteLock #동시성