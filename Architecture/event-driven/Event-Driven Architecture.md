
Event-Driven Architecture는 **서비스 간 이벤트로 통신하는 방식**이다.

---

## 기존 방식 vs Event-Driven

### 기존 방식 (직접 호출)

```
order-service → payment-service 호출
order-service → notification-service 호출
order-service → stock-service 호출
```

order-service가 다른 서비스들을 직접 알고 있어야 함.

### Event-Driven

```
order-service → "주문 생성됨" 이벤트 발행 → 끝
```

누가 듣든 말든 order-service는 신경 안 씀.

---

## 구조

```
┌─────────────────┐
│  order-service  │ ← Producer
└────────┬────────┘
         │ 이벤트 발행
         ▼
┌─────────────────────────────────────────────────────┐
│                      Kafka                          │
└────────┬──────────────┬──────────────┬─────────────┘
         │              │              │
         ▼              ▼              ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   stock     │  │notification │  │  analytics  │ ← Consumer
│  service    │  │  service    │  │  service    │
└─────────────┘  └─────────────┘  └─────────────┘
```

---

## 장점

|장점|설명|
|---|---|
|느슨한 결합|서비스끼리 서로 몰라도 됨|
|확장 용이|새 Consumer 추가 시 기존 코드 수정 없음|
|장애 격리|한 서비스가 죽어도 다른 서비스에 영향 없음|
|비동기 처리|응답 안 기다리니까 빠름|

---

## 단점

|단점|설명|
|---|---|
|디버깅 어려움|이벤트 흐름 추적이 복잡함|
|순서 보장 어려움|이벤트 도착 순서가 보장 안 될 수 있음|
|트랜잭션 관리|분산 트랜잭션 처리 필요 (Saga 패턴)|

---

## MSA와의 관계

```
┌─────────────────────────────────────────┐
│                 MSA                     │
│          (서비스 분리 방식)               │
│                                         │
│   통신 방식 선택:                         │
│   ├── 동기 (REST, gRPC)                 │
│   └── 비동기 (Event-Driven)             │
│                                         │
└─────────────────────────────────────────┘
```

| |MSA|Event-Driven|
|---|---|---|
|관심사|서비스 분리|서비스 간 통신|
|필수 관계|❌|❌|
MSA가 아니어도 Event-Driven 쓸 수 있고, MSA여도 REST로 통신할 수 있음.

---

## 모놀리식에서도 사용 가능

```java
// 하나의 서비스 안에서 이벤트로 통신
@Service
public class OrderService {
    public void createOrder(Order order) {
        orderRepository.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order));
    }
}

@Component
public class NotificationHandler {
    @EventListener
    public void handle(OrderCreatedEvent event) {
        // 알림 발송
    }
}
```

나중에 MSA로 전환할 때 Kafka로 바꾸기만 하면 됨.

---

## 관련 개념

- [[Microservices Architecture]]
- [[Saga Pattern]]
- [[Kafka]]
- [[Redis]]

---

## Tags

#EventDriven #Architecture #MSA #비동기