## Lock이란?

- **synchronized보다 유연한 동기화**
- 명시적 락 획득/해제
- 타임아웃, 인터럽트 지원
- `java.util.concurrent.locks` 패키지

---

## synchronized vs Lock

|synchronized|Lock|
|---|---|
|암시적|명시적|
|블록 벗어나면 자동 해제|수동 해제 필요|
|타임아웃 X|타임아웃 O|
|인터럽트 X|인터럽트 O|
|공정성 설정 X|공정성 설정 O|

---

## ReentrantLock

### 기본 사용

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private int count = 0;
    private final Lock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();  // 락 획득
        try {
            count++;
        } finally {
            lock.unlock();  // 반드시 해제!
        }
    }
    
    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

### 공정 락 (Fair Lock)

```java
// 먼저 대기한 스레드가 먼저 획득
Lock fairLock = new ReentrantLock(true);

// 기본: 불공정 락 (성능 우선)
Lock unfairLock = new ReentrantLock(false);
```

---

## tryLock() - 시도 후 포기

### 즉시 시도

```java
Lock lock = new ReentrantLock();

if (lock.tryLock()) {
    try {
        // 작업 수행
    } finally {
        lock.unlock();
    }
} else {
    // 락 획득 실패 시 대체 작업
    System.out.println("락 획득 실패");
}
```

### 타임아웃

```java
try {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            // 작업 수행
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

---

## lockInterruptibly() - 인터럽트 가능

```java
Lock lock = new ReentrantLock();

Thread t = new Thread(() -> {
    try {
        lock.lockInterruptibly();  // 인터럽트 가능
        try {
            // 작업
        } finally {
            lock.unlock();
        }
    } catch (InterruptedException e) {
        System.out.println("락 대기 중 인터럽트됨");
    }
});

t.start();
Thread.sleep(100);
t.interrupt();  // 락 대기 중 인터럽트
```

---

## ReadWriteLock

### 읽기/쓰기 분리

- **읽기**: 여러 스레드 동시 가능
- **쓰기**: 한 스레드만 독점

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class Cache {
    private final Map<String, String> map = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    
    public String get(String key) {
        readLock.lock();  // 읽기 락
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }
    
    public void put(String key, String value) {
        writeLock.lock();  // 쓰기 락
        try {
            map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
    
    public void remove(String key) {
        writeLock.lock();
        try {
            map.remove(key);
        } finally {
            writeLock.unlock();
        }
    }
}
```

---

## Condition

### wait/notify 대체

```java
import java.util.concurrent.locks.Condition;

class BoundedBuffer<T> {
    private final Object[] items;
    private int putIndex, takeIndex, count;
    
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public BoundedBuffer(int capacity) {
        items = new Object[capacity];
    }
    
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();  // 가득 차면 대기
            }
            items[putIndex] = item;
            putIndex = (putIndex + 1) % items.length;
            count++;
            notEmpty.signal();  // 소비자 깨우기
        } finally {
            lock.unlock();
        }
    }
    
    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 비어있으면 대기
            }
            T item = (T) items[takeIndex];
            takeIndex = (takeIndex + 1) % items.length;
            count--;
            notFull.signal();  // 생산자 깨우기
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### Condition 메서드

|Object|Condition|
|---|---|
|wait()|await()|
|wait(timeout)|await(time, unit)|
|notify()|signal()|
|notifyAll()|signalAll()|

---

## StampedLock (Java 8+)

### 낙관적 읽기

```java
import java.util.concurrent.locks.StampedLock;

class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();
    
    // 쓰기
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
    
    // 낙관적 읽기
    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();  // 락 없이 읽기 시도
        double currentX = x;
        double currentY = y;
        
        if (!lock.validate(stamp)) {  // 검증 실패 시
            stamp = lock.readLock();   // 읽기 락으로 전환
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

---

## 데드락 방지

### tryLock 활용

```java
Lock lock1 = new ReentrantLock();
Lock lock2 = new ReentrantLock();

void transferMoney() {
    boolean acquired = false;
    while (!acquired) {
        if (lock1.tryLock()) {
            try {
                if (lock2.tryLock()) {
                    try {
                        // 송금 처리
                        acquired = true;
                    } finally {
                        lock2.unlock();
                    }
                }
            } finally {
                if (!acquired) {
                    lock1.unlock();
                }
            }
        }
        
        if (!acquired) {
            Thread.sleep(10);  // 잠시 대기 후 재시도
        }
    }
}
```

---

## 실전 예제

### 스레드 안전 캐시

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
    
    public V computeIfAbsent(K key, Function<K, V> mappingFunction) {
        // 먼저 읽기 락으로 확인
        lock.readLock().lock();
        try {
            V value = cache.get(key);
            if (value != null) {
                return value;
            }
        } finally {
            lock.readLock().unlock();
        }
        
        // 없으면 쓰기 락으로 생성
        lock.writeLock().lock();
        try {
            V value = cache.get(key);
            if (value == null) {
                value = mappingFunction.apply(key);
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
> - **tryLock()**: 타임아웃, 데드락 방지
> - **ReadWriteLock**: 읽기/쓰기 분리
> - **Condition**: wait/notify 대체
> - **반드시 finally에서 unlock()**
> - 읽기 많으면 ReadWriteLock 고려

---

#Java #Lock #ReentrantLock #동시성 #멀티스레딩