
Outbox 패턴은 **이벤트 발행을 보장하는 방법**이다.

---

## 문제 상황

```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);      // 1. DB 저장 ✓
    kafkaTemplate.send(orderEvent);   // 2. 이벤트 발행 ← 여기서 실패하면?
}
```

DB는 저장됐는데 Kafka 발행이 실패하면? 데이터 불일치 발생.

```
Order DB: 주문 있음
Kafka: 이벤트 없음 → 재고 차감 안 됨, 알림 안 감
```

---

## 해결: Outbox 테이블

이벤트를 Kafka로 바로 보내지 않고, **같은 DB에 먼저 저장**한다.

```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);           // 1. 주문 저장
    outboxRepository.save(orderEvent);     // 2. 이벤트도 같은 DB에 저장
}
// 같은 트랜잭션 → 둘 다 성공하거나 둘 다 실패
```

---

## 구조

```
┌─────────────────────────────────────────────────────────────┐
│                        Database                             │
│  ┌─────────────┐          ┌─────────────┐                  │
│  │ Order Table │          │ Outbox Table│                  │
│  │             │          │             │                  │
│  │ id: 1       │          │ id: 1       │                  │
│  │ product: A  │          │ event: JSON │                  │
│  │ amount: 100 │          │ status: PENDING               │
│  └─────────────┘          └──────┬──────┘                  │
│                                  │                          │
└──────────────────────────────────┼──────────────────────────┘
                                   │
                           스케줄러가 읽어서
                                   │
                                   ▼
                            ┌─────────────┐
                            │    Kafka    │
                            └─────────────┘
```

---

## 스케줄러로 발행

```java
@Scheduled(fixedRate = 1000)  // 1초마다 실행
public void publishEvents() {
    List<Outbox> events = outboxRepository.findByStatus("PENDING");
    
    for (Outbox event : events) {
        try {
            kafkaTemplate.send(event.getTopic(), event.getPayload());
            event.setStatus("PUBLISHED");
            outboxRepository.save(event);
        } catch (Exception e) {
            // 실패하면 다음에 다시 시도
        }
    }
}
```

---

## Outbox 테이블 예시

|id|aggregate_type|aggregate_id|event_type|payload|status|created_at|
|---|---|---|---|---|---|---|
|1|Order|123|OrderCreated|{"orderId": 123, ...}|PENDING|2024-01-01|
|2|Order|124|OrderCreated|{"orderId": 124, ...}|PUBLISHED|2024-01-01|

---

## 장점

|장점|설명|
|---|---|
|데이터 일관성|DB 저장과 이벤트 저장이 같은 트랜잭션|
|발행 보장|실패해도 재시도 가능|
|순서 보장|테이블에 저장된 순서대로 발행|

---

## 단점

|단점|설명|
|---|---|
|지연 발생|스케줄러 주기만큼 딜레이|
|테이블 관리|Outbox 테이블 정리 필요|
|중복 발행 가능|Consumer에서 멱등성 처리 필요|

---

## 관련 개념

- [[Kafka]]
- [[Event-Driven Architecture]]
- [[Saga Pattern]]

---

## Tags

#Outbox #EventDriven #MSA #분산시스템