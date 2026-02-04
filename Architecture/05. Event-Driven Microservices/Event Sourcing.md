**현재 상태 대신 상태 변경 이벤트를 저장하는 패턴**
데이터를 덮어쓰지 않고, 모든 변경 내역을 이벤트로 쌓아둔다.

---

## 문제 상황

일반적인 방식은 현재 상태만 저장한다.
```java
// 잔액 1000원 → 500원 출금
account.setBalance(500);
accountRepository.save(account);
```

```
accounts 테이블
| id | balance |
|----|---------|
| 1  | 500     |
```

이전에 얼마였는지, 왜 바뀌었는지 알 수 없다.

---

## 해결: 이벤트를 저장

상태를 저장하는 대신, **일어난 일(이벤트)**을 저장한다.
```
events 테이블
| id | account_id | type        | amount | created_at |
|----|------------|-------------|--------|------------|
| 1  | 1          | Created     | 0      | 10:00      |
| 2  | 1          | Deposited   | 1000   | 10:01      |
| 3  | 1          | Withdrawn   | 500    | 10:02      |
```

현재 잔액 = 이벤트를 순서대로 재생하면 나온다.
```
0 → +1000 → -500 = 500원
```

---

## 일반 방식 vs Event Sourcing
```
일반 방식:
┌─────────┐
│ balance │  500원 (현재 상태만)
└─────────┘

Event Sourcing:
┌──────────────────────────────┐
│ Created    → 0원             │
│ Deposited  → +1000원         │
│ Withdrawn  → -500원          │
│ ─────────────────────────    │
│ 현재 상태  → 500원            │
└──────────────────────────────┘
````

---

## 코드 예시

### 이벤트 정의
```java
public interface AccountEvent {}

public record AccountCreated(String accountId) implements AccountEvent {}
public record MoneyDeposited(String accountId, int amount) implements AccountEvent {}
public record MoneyWithdrawn(String accountId, int amount) implements AccountEvent {}
```

### 이벤트 저장
```java
@Service
public class AccountService {
    public void withdraw(String accountId, int amount) {
        // 상태 변경 대신 이벤트 저장
        MoneyWithdrawn event = new MoneyWithdrawn(accountId, amount);
        eventStore.append(accountId, event);
    }
}
```

### 현재 상태 조회 (이벤트 재생)
```java
public Account getAccount(String accountId) {
    List<AccountEvent> events = eventStore.getEvents(accountId);
    
    Account account = new Account();
    for (AccountEvent event : events) {
        account.apply(event);  // 이벤트 순서대로 적용
    }
    return account;
}

public class Account {
    private int balance = 0;
    
    public void apply(AccountEvent event) {
        if (event instanceof AccountCreated) {
            this.balance = 0;
        } else if (event instanceof MoneyDeposited e) {
            this.balance += e.amount();
        } else if (event instanceof MoneyWithdrawn e) {
            this.balance -= e.amount();
        }
    }
}
```

---

## Snapshot

이벤트가 많아지면 매번 재생하기 느리다. 중간중간 스냅샷을 저장한다.
```
이벤트 1~1000 → Snapshot (balance: 50000)
이벤트 1001~1010 → 여기부터만 재생

조회 시: Snapshot + 이후 이벤트만 재생
```


```java
public Account getAccount(String accountId) {
    Snapshot snapshot = snapshotStore.getLatest(accountId);
    List<AccountEvent> events = eventStore.getEventsAfter(accountId, snapshot.getVersion());
    
    Account account = snapshot.toAccount();
    for (AccountEvent event : events) {
        account.apply(event);
    }
    return account;
}
```

---

## CQRS와 함께 쓰는 이유

Event Sourcing만 쓰면 조회할 때마다 이벤트 재생해야 해서 느리다.
```
Event Sourcing (Write) + CQRS (Read)

쓰기: 이벤트 저장
       ↓ 이벤트 발행
읽기: Read DB에 현재 상태 저장 → 빠른 조회
```
```
┌─────────────┐      ┌─────────┐      ┌─────────────┐
│ Event Store │ ──▶  │  Kafka  │ ──▶  │   Read DB   │
│ (이벤트저장) │      │         │      │ (현재상태)   │
└─────────────┘      └─────────┘      └─────────────┘
````

---

## 언제 쓰나?

|상황|Event Sourcing|
|---|---|
|변경 이력 추적 필수 (금융, 의료)|✓|
|감사 로그 필요|✓|
|시간 여행 디버깅 필요|✓|
|단순 CRUD|✗ 오버엔지니어링|

---

## 장단점

|장점|단점|
|---|---|
|완벽한 변경 이력|복잡도 증가|
|감사/디버깅 용이|조회 성능 (CQRS로 보완)|
|이벤트 재생으로 버그 재현|이벤트 스키마 변경 어려움|
|시점 복원 가능|러닝 커브|

---

## 관련 개념

- [[CQRS]]
- [[Event-Driven Architecture]]
- [[Saga Pattern]]