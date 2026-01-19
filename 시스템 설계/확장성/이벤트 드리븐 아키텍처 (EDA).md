## EDA란?

**Event-Driven Architecture**

시스템 간 통신을 **이벤트** 중심으로 설계하는 아키텍처 패턴

---

## 기본 개념

### 동기 방식 (전통적)

```
[주문 서비스] → API 호출 → [재고 서비스]
                  │
                  └─────→ [알림 서비스]
                  │
                  └─────→ [정산 서비스]
```

- 순차적으로 호출
- 하나 실패하면 전체 실패
- 강한 결합

### 이벤트 드리븐 방식

```
[주문 서비스] → OrderCreated 이벤트 발행
                        │
                        ▼
                   [메시지 브로커]
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
    [재고 서비스]   [알림 서비스]   [정산 서비스]
    (이벤트 구독)   (이벤트 구독)   (이벤트 구독)
```

- 이벤트 발행 후 끝
- 각 서비스가 독립적으로 처리
- 느슨한 결합

---

## 핵심 구성 요소

### 1. 이벤트 (Event)

**"무슨 일이 일어났다"**를 나타내는 메시지

```typescript
// 이벤트 예시
{
  eventType: "OrderCreated",
  timestamp: "2024-01-15T10:30:00Z",
  data: {
    orderId: "order-123",
    userId: "user-456",
    items: [...],
    totalAmount: 50000
  }
}
```

**이벤트 네이밍**: 과거형 사용

- ✅ OrderCreated, PaymentCompleted, UserRegistered
- ❌ CreateOrder, CompletePayment

### 2. 프로듀서 (Producer)

이벤트를 발행하는 주체

```typescript
// 주문 서비스 (프로듀서)
async function createOrder(orderData) {
  // 1. 주문 저장
  const order = await orderRepository.save(orderData);
  
  // 2. 이벤트 발행
  await eventBus.publish('OrderCreated', {
    orderId: order.id,
    userId: order.userId,
    items: order.items,
    totalAmount: order.totalAmount
  });
  
  return order;
}
```

### 3. 컨슈머 (Consumer)

이벤트를 구독하고 처리하는 주체

```typescript
// 재고 서비스 (컨슈머)
eventBus.subscribe('OrderCreated', async (event) => {
  for (const item of event.data.items) {
    await inventoryService.decrease(item.productId, item.quantity);
  }
});

// 알림 서비스 (컨슈머)
eventBus.subscribe('OrderCreated', async (event) => {
  await notificationService.send(event.data.userId, '주문이 완료되었습니다');
});

// 정산 서비스 (컨슈머)
eventBus.subscribe('OrderCreated', async (event) => {
  await settlementService.record(event.data);
});
```

### 4. 메시지 브로커 (Message Broker)

이벤트를 전달하는 중간 매개체

| 브로커               | 특징              | 사용 사례         |
| ----------------- | --------------- | ------------- |
| **Kafka**         | 고성능, 내구성, 순서 보장 | 대규모, 로그, 스트리밍 |
| **RabbitMQ**      | 유연한 라우팅, 간단     | 중소규모, 작업 큐    |
| **AWS SQS/SNS**   | 관리형, 간편         | AWS 환경        |
| **Redis Pub/Sub** | 빠름, 간단          | 실시간, 간단한 이벤트  |

---

## EDA 패턴

### 1. Pub/Sub (발행/구독)

```
[Publisher] → [Topic] → [Subscriber 1]
                      → [Subscriber 2]
                      → [Subscriber 3]
```

- 1:N 브로드캐스트
- 구독자가 각자 처리

```typescript
// 발행
await kafka.publish('order-events', { type: 'OrderCreated', data: order });

// 구독 (여러 서비스)
kafka.subscribe('order-events', handleOrderEvent);  // 재고
kafka.subscribe('order-events', handleOrderEvent);  // 알림
kafka.subscribe('order-events', handleOrderEvent);  // 정산
```

### 2. Event Sourcing

모든 상태 변경을 이벤트로 저장

```
[이벤트 스토어]
├── OrderCreated { orderId: 1, items: [...] }
├── OrderPaid { orderId: 1, amount: 50000 }
├── OrderShipped { orderId: 1, trackingNo: "..." }
└── OrderDelivered { orderId: 1 }

→ 이벤트들을 재생하면 현재 상태 복원 가능
```

**장점**

- 모든 변경 이력 보존
- 시간 여행 (특정 시점 상태 복원)
- 감사 로그 자동화

**단점**

- 복잡도 높음
- 이벤트 스키마 변경 어려움

### 3. CQRS + Event Sourcing

```
[Command] → 이벤트 저장 → [Event Store]
                              │
                         이벤트 반영
                              │
                              ▼
[Query] ←────────────── [Read Model]
```

- 쓰기: 이벤트 저장
- 읽기: 이벤트를 반영한 뷰에서 조회

---

## 실전 예시

### 예시 1: 이커머스 주문 플로우

```
1. 고객이 주문

2. 주문 서비스
   └─ OrderCreated 이벤트 발행

3. 각 서비스가 독립 처리
   ├─ 재고 서비스: 재고 차감
   ├─ 결제 서비스: 결제 요청
   ├─ 알림 서비스: 주문 확인 메일
   └─ 추천 서비스: 구매 이력 업데이트

4. 결제 서비스
   └─ PaymentCompleted 이벤트 발행

5. 배송 서비스
   └─ PaymentCompleted 구독 → 배송 준비
```

### 예시 2: 회원가입 플로우

```typescript
// 회원 서비스
async function registerUser(userData) {
  const user = await userRepository.save(userData);
  
  await eventBus.publish('UserRegistered', {
    userId: user.id,
    email: user.email,
    name: user.name
  });
  
  return user;
}

// 각 서비스가 구독
// 이메일 서비스
eventBus.subscribe('UserRegistered', async (event) => {
  await sendWelcomeEmail(event.data.email);
});

// 쿠폰 서비스
eventBus.subscribe('UserRegistered', async (event) => {
  await createWelcomeCoupon(event.data.userId);
});

// 분석 서비스
eventBus.subscribe('UserRegistered', async (event) => {
  await analytics.trackSignup(event.data);
});
```

---

## 이벤트 기반 vs EDA

|구분|이벤트 기반|EDA|
|---|---|---|
|범위|부분 적용|전체 아키텍처|
|예시|캐시 무효화에 이벤트 사용|모든 서비스 간 통신이 이벤트|
|복잡도|낮음|높음|

```
이벤트 기반 = "이벤트를 쓴다" (기법)
EDA = "이벤트로 시스템을 설계한다" (아키텍처)
```

---

## 장단점

### 장점

|장점|설명|
|---|---|
|**느슨한 결합**|서비스 간 직접 의존 없음|
|**확장성**|컨슈머 추가/제거 쉬움|
|**장애 격리**|하나 실패해도 다른 서비스 영향 없음|
|**비동기 처리**|응답 빠름, 처리량 높음|
|**유연성**|새 기능 추가 시 구독만 하면 됨|

### 단점

|단점|설명|
|---|---|
|**복잡도**|디버깅, 추적 어려움|
|**eventual consistency**|즉각적 일관성 보장 안 됨|
|**이벤트 순서**|순서 보장 어려울 수 있음|
|**멱등성 필요**|중복 처리 대비 필요|
|**인프라 비용**|메시지 브로커 운영 필요|

---

## 고려 사항

### 1. 멱등성 (Idempotency)

같은 이벤트가 여러 번 와도 결과가 같아야 함

```typescript
// ❌ 멱등하지 않음
eventBus.subscribe('OrderCreated', async (event) => {
  await inventory.decrease(event.productId, 1);  // 중복 처리 시 계속 차감
});

// ✅ 멱등함
eventBus.subscribe('OrderCreated', async (event) => {
  const processed = await checkProcessed(event.eventId);
  if (processed) return;  // 이미 처리됨
  
  await inventory.decrease(event.productId, 1);
  await markProcessed(event.eventId);
});
```

### 2. 이벤트 순서

```
// 문제: 이벤트 순서가 뒤바뀌면?
OrderCreated → OrderCancelled  (정상)
OrderCancelled → OrderCreated  (문제!)
```

**해결**

- Kafka 파티션 키 사용 (같은 키는 순서 보장)
- 이벤트에 시퀀스 번호 포함
- 상태 기반 검증

### 3. 실패 처리

```typescript
// Dead Letter Queue (DLQ) 사용
eventBus.subscribe('OrderCreated', async (event) => {
  try {
    await processOrder(event);
  } catch (error) {
    // 재시도 후에도 실패하면 DLQ로
    await dlq.send(event);
  }
});
```

### 4. 모니터링

- 이벤트 발행/소비 지연 시간
- 실패율
- 큐 깊이 (처리 못 한 이벤트 수)

---

## 언제 EDA를 쓰는가?

### 적합한 경우

|상황|이유|
|---|---|
|MSA 환경|서비스 간 느슨한 결합|
|비동기 처리 필요|알림, 로그, 분석|
|확장성 필요|컨슈머 독립적 스케일링|
|여러 시스템 연동|이벤트로 통합|

### 부적합한 경우

|상황|이유|
|---|---|
|단순한 CRUD|오버엔지니어링|
|강한 일관성 필수|동기식이 나음|
|소규모 모놀리식|복잡도만 증가|

---

## 핵심 요약

1. **EDA = 이벤트 중심 시스템 설계**
2. **구성**: 프로듀서 → 브로커 → 컨슈머
3. **장점**: 느슨한 결합, 확장성, 장애 격리
4. **단점**: 복잡도, eventual consistency
5. **필수 고려**: 멱등성, 순서, 실패 처리
6. **MSA에서 많이 사용**