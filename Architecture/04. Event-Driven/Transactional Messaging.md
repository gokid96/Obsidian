Transactional Messaging은 **DB 트랜잭션과 메시지 발행을 원자적으로 처리하는 방법**이다.

DB 저장과 메시지 발행이 둘 다 성공하거나 둘 다 실패하게 보장한다.

---
## 문제 상황

```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);      // 1. DB 저장 ✓
    kafkaTemplate.send(orderEvent);   // 2. 메시지 발행 ← 여기서 실패하면?
}
```

DB는 저장됐는데 Kafka 발행이 실패하면 데이터 불일치 발생.
```
Order DB: 주문 있음
Kafka: 이벤트 없음 → 재고 차감 안 됨, 알림 안 감
```

반대로 Kafka는 발행됐는데 DB 커밋이 실패해도 문제.
```
Order DB: 주문 없음
Kafka: 이벤트 있음 → 존재하지 않는 주문 처리 시도
````

---

## 해결 방법

### 1. Outbox 패턴

메시지를 Kafka로 바로 보내지 않고, 같은 DB에 먼저 저장한다.

java

````java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);           // 1. 주문 저장
    outboxRepository.save(orderEvent);     // 2. 이벤트도 같은 DB에 저장
}
// 같은 트랜잭션 → 둘 다 성공하거나 둘 다 실패
```

별도 스케줄러가 Outbox 테이블을 읽어서 Kafka로 발행.
```
┌─────────────────────────────────────┐
│              Database               │
│  ┌───────────┐    ┌──────────────┐  │
│  │   Order   │    │    Outbox    │  │
│  └───────────┘    └──────┬───────┘  │
└──────────────────────────┼──────────┘
                           │
                    스케줄러가 읽어서
                           ↓
                    ┌─────────────┐
                    │    Kafka    │
                    └─────────────┘
```

### 2. Transactional Outbox + CDC

Change Data Capture로 Outbox 테이블 변경을 감지해서 자동 발행.
```
DB 저장 → CDC(Debezium)가 감지 → Kafka로 자동 발행
```

스케줄러 없이 실시간에 가깝게 처리 가능.
```
┌─────────────┐      ┌───────────┐      ┌─────────┐
│  Database   │ ──▶  │ Debezium  │ ──▶  │  Kafka  │
│  (Outbox)   │      │   (CDC)   │      │         │
└─────────────┘      └───────────┘      └─────────┘
```

### 3. Transaction Log Tailing

Outbox 테이블 없이 DB의 트랜잭션 로그 자체를 읽어서 이벤트 발행.
```
DB 커밋 → 트랜잭션 로그 → CDC가 읽음 → Kafka 발행
````

---

## 비교

|방식|장점|단점|
|---|---|---|
|Outbox + 스케줄러|구현 단순|폴링 지연|
|Outbox + CDC|실시간에 가까움|인프라 복잡 (Debezium 등)|
|Log Tailing|Outbox 테이블 불필요|DB 종속적, 설정 복잡|

---

## Consumer 멱등성

메시지가 중복 발행될 수 있어서 Consumer는 멱등성 처리 필수.
```java
@KafkaListener(topics = "order-events")
public void handle(OrderCreatedEvent event) {
    // 이미 처리한 이벤트인지 확인
    if (processedEventRepository.exists(event.getId())) {
        return;  // 중복이면 무시
    }
    
    // 처리
    stockService.decrease(event.getOrderId());
    processedEventRepository.save(event.getId());
}
```

---

## 언제 쓰나?

|상황|필요 여부|
|---|---|
|DB 저장 + 이벤트 발행이 함께 필요|✓|
|데이터 일관성이 중요|✓|
|단순 로깅/알림 (실패해도 괜찮음)|✗|

---

## 관련 개념

- [[Outbox Pattern]]
- [[Event-Driven Architecture]]
- [[Saga Pattern]]
- [[CDC (Change Data Capture)]]