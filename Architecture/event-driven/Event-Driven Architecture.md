
Event-Driven Architecture는 **서비스 간 이벤트로 통신하는 방식**

---

## 기존 방식 vs Event-Driven

#### 기존 방식 (직접 호출)

```
order-service → payment-service 호출
order-service → notification-service 호출
order-service → stock-service 호출
```

order-service가 다른 서비스들을 직접 알고 있어야 함.

#### Event-Driven

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

## 언제 Event-Driven을 써야 하는가

|동기 방식 (REST/gRPC)|비동기 방식 (Event-Driven)|
|---|---|
|즉시 응답이 필요함|응답 안 기다려도 됨|
|단순한 요청-응답|한 이벤트에 여러 서비스가 반응|
|트랜잭션 보장 중요|느슨한 결합이 중요|
|호출 대상이 명확함|누가 구독할지 몰라도 됨|

### 예시

|상황|방식|이유|
|---|---|---|
|주문 조회|REST|즉시 응답 필요|
|주문 생성 → 알림/재고/통계|Event-Driven|여러 서비스가 반응, 응답 안 기다려도 됨|
|결제 요청|REST|결과 즉시 확인 필요|
|결제 완료 → 포인트 적립|Event-Driven|비동기로 처리해도 됨|

### 실무에서는 섞어서 사용

```
Client → API Gateway → order-service (REST)
                            │
                            ├── payment-service 호출 (REST) ← 결과 필요
                            │
                            └── 이벤트 발행 (Kafka)
                                    │
                                    ├── notification-service ← 비동기
                                    ├── analytics-service ← 비동기
                                    └── stock-service ← 비동기
```

---

## 모놀리식에서 Event-Driven을 쓰는 이유

### 직접 호출 방식

```java
public void createOrder(Order order) {
    orderRepository.save(order);
    notificationService.send(order);  // 직접 호출
    stockService.decrease(order);     // 직접 호출
    analyticsService.log(order);      // 직접 호출
}
```

OrderService가 모든 서비스를 알아야 함. 기능 추가할 때마다 여기 수정 필요.

### 이벤트 방식

```java
public void createOrder(Order order) {
    orderRepository.save(order);
    eventPublisher.publish(new OrderCreatedEvent(order));  // 끝
}
```

OrderService는 이벤트만 발행하고 끝. 결합도 낮아짐.

### 장점

|장점|설명|
|---|---|
|결합도 낮춤|OrderService가 다른 서비스 몰라도 됨|
|확장 쉬움|새 Handler 추가해도 OrderService 수정 없음|
|MSA 전환 쉬움|나중에 `eventPublisher` → `kafkaTemplate`으로 바꾸면 끝|

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

||MSA|Event-Driven|
|---|---|---|
|관심사|서비스 분리|서비스 간 통신|
|필수 관계|❌|❌|

MSA가 아니어도 Event-Driven 쓸 수 있고, MSA여도 REST로 통신할 수 있음.

---

## 관련 개념

- [[Microservices Architecture]]
- [[Saga Pattern]]
- [[Kafka]]
- [[Redis]]

---

## Tags

#EventDriven #Architecture #MSA #비동기