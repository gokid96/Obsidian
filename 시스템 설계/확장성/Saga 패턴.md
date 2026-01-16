## Saga 패턴이란?

MSA 환경에서 **여러 서비스에 걸친 트랜잭션**을 관리하는 패턴

실패 시 **보상 트랜잭션**으로 데이터 일관성 유지

---

## 왜 필요한가?

### 모놀리식에서는

```
[하나의 DB]
BEGIN TRANSACTION;
  INSERT INTO orders ...
  UPDATE inventory ...
  INSERT INTO payments ...
COMMIT;  
-- 전부 성공 or 전부 실패
```

→ DB 트랜잭션으로 간단하게 처리

### MSA에서는

```
[주문 DB]     [재고 DB]     [결제 DB]
    │             │             │
    └─────────────┴─────────────┘
         분산되어 있음
         
	하나의 트랜잭션으로 묶을 수 없음
```

→ Saga 패턴 필요

---

## 기본 개념

### 정상 흐름

```
T1: 주문 생성 → T2: 재고 차감 → T3: 결제 → T4: 배송 준비
     ✅             ✅            ✅           ✅
```

### 실패 시 보상 트랜잭션

```
T1: 주문 생성 → T2: 재고 차감 → T3: 결제 실패!
     ✅             ✅            ❌
                    │             │
                    ▼             ▼
              C2: 재고 복구 ← C1: 주문 취소
              
T = Transaction (정방향)
C = Compensation (보상, 역방향)
```

---

## 두 가지 구현 방식

### 1. Choreography (안무)

각 서비스가 이벤트를 발행하고 구독하며 **스스로 판단**

```
[주문 서비스]
     │
     │ OrderCreated
     ▼
[재고 서비스] ──── InventoryReserved ────→ [결제 서비스]
     │                                          │
     │                                    PaymentFailed
     │                                          │
     │←─────── InventoryReserveFailed ──────────┘
     │
     │ 재고 복구
     ▼
[주문 서비스] ← OrderCancelled
```

**코드 예시**

```typescript
// 주문 서비스
async function createOrder(data) {
  const order = await orderRepo.save({ ...data, status: 'PENDING' });
  await eventBus.publish('OrderCreated', { orderId: order.id, items: data.items });
  return order;
}

// 주문 취소 (보상)
eventBus.subscribe('InventoryReserveFailed', async (event) => {
  await orderRepo.updateStatus(event.orderId, 'CANCELLED');
  await eventBus.publish('OrderCancelled', { orderId: event.orderId });
});

// 재고 서비스
eventBus.subscribe('OrderCreated', async (event) => {
  try {
    await inventoryService.reserve(event.items);
    await eventBus.publish('InventoryReserved', { orderId: event.orderId });
  } catch (error) {
    await eventBus.publish('InventoryReserveFailed', { orderId: event.orderId });
  }
});

// 재고 복구 (보상)
eventBus.subscribe('PaymentFailed', async (event) => {
  await inventoryService.release(event.items);
  await eventBus.publish('InventoryReserveFailed', { orderId: event.orderId });
});

// 결제 서비스
eventBus.subscribe('InventoryReserved', async (event) => {
  try {
    await paymentService.charge(event.amount);
    await eventBus.publish('PaymentCompleted', { orderId: event.orderId });
  } catch (error) {
    await eventBus.publish('PaymentFailed', { orderId: event.orderId });
  }
});
```

|장점|단점|
|---|---|
|느슨한 결합|흐름 파악 어려움|
|단순한 구조|순환 참조 위험|
|단일 장애점 없음|디버깅 어려움|

---

### 2. Orchestration (오케스트라)

**중앙 오케스트레이터**가 전체 흐름을 제어

```
              [Saga Orchestrator]
                     │
    ┌────────────────┼────────────────┐
    │                │                │
    ▼                ▼                ▼
[주문 서비스]   [재고 서비스]   [결제 서비스]

오케스트레이터가 순서대로 호출하고
실패 시 보상 트랜잭션 실행
```

**코드 예시**

```typescript
class OrderSagaOrchestrator {
  async execute(orderData) {
    const sagaLog = await this.createSagaLog();
    
    try {
      // Step 1: 주문 생성
      const order = await this.orderService.create(orderData);
      await this.logStep(sagaLog, 'ORDER_CREATED', order.id);
      
      // Step 2: 재고 차감
      await this.inventoryService.reserve(order.items);
      await this.logStep(sagaLog, 'INVENTORY_RESERVED');
      
      // Step 3: 결제
      await this.paymentService.charge(order.totalAmount);
      await this.logStep(sagaLog, 'PAYMENT_COMPLETED');
      
      // Step 4: 배송 준비
      await this.shippingService.prepare(order.id);
      await this.logStep(sagaLog, 'SHIPPING_PREPARED');
      
      await this.completeSaga(sagaLog);
      return order;
      
    } catch (error) {
      await this.compensate(sagaLog);
      throw error;
    }
  }
  
  async compensate(sagaLog) {
    // 역순으로 보상 트랜잭션 실행
    const steps = await this.getCompletedSteps(sagaLog);
    
    for (const step of steps.reverse()) {
      switch (step.type) {
        case 'SHIPPING_PREPARED':
          await this.shippingService.cancel(step.orderId);
          break;
        case 'PAYMENT_COMPLETED':
          await this.paymentService.refund(step.paymentId);
          break;
        case 'INVENTORY_RESERVED':
          await this.inventoryService.release(step.items);
          break;
        case 'ORDER_CREATED':
          await this.orderService.cancel(step.orderId);
          break;
      }
    }
  }
}
```

|장점|단점|
|---|---|
|흐름 명확|오케스트레이터에 로직 집중|
|디버깅 쉬움|단일 장애점 가능성|
|복잡한 흐름 처리 가능|서비스 간 결합도 증가|

---

## 방식 비교

|항목|Choreography|Orchestration|
|---|---|---|
|흐름 제어|분산 (각 서비스)|중앙 (오케스트레이터)|
|결합도|낮음|중간|
|복잡한 흐름|어려움|쉬움|
|디버깅|어려움|쉬움|
|단일 장애점|없음|오케스트레이터|
|적합한 경우|단순한 흐름|복잡한 비즈니스 로직|

---

## 실전 예시: 주문 프로세스

### 상태 머신

```
PENDING → INVENTORY_RESERVED → PAYMENT_COMPLETED → SHIPPED → DELIVERED
    │              │                    │
    ▼              ▼                    ▼
CANCELLED    INVENTORY_FAILED     PAYMENT_FAILED
                   │                    │
                   └────────────────────┘
                            │
                      보상 트랜잭션
```

### 각 단계별 처리

|단계|정상 처리|보상 트랜잭션|
|---|---|---|
|주문 생성|주문 저장 (PENDING)|주문 취소 (CANCELLED)|
|재고 예약|재고 차감|재고 복구|
|결제|결제 승인|환불|
|배송 준비|배송 요청|배송 취소|

### 전체 코드 흐름

```typescript
// 1. 주문 생성
order = { id: 'order-123', status: 'PENDING', items: [...] }

// 2. 재고 예약 성공
inventory.reserve('product-1', 2)  // 재고 10 → 8
order.status = 'INVENTORY_RESERVED'

// 3. 결제 실패!
payment.charge(50000)  // ❌ 카드 한도 초과

// 4. 보상 트랜잭션 시작
inventory.release('product-1', 2)  // 재고 8 → 10 (복구)
order.status = 'CANCELLED'
```

---

## 고려 사항

### 1. 멱등성

보상 트랜잭션이 여러 번 실행되어도 안전해야 함

```typescript
// ❌ 멱등하지 않음
async function refund(paymentId) {
  await payment.refund(paymentId);  // 중복 환불 위험
}

// ✅ 멱등함
async function refund(paymentId) {
  const payment = await paymentRepo.find(paymentId);
  if (payment.status === 'REFUNDED') return;  // 이미 처리됨
  
  await payment.refund(paymentId);
  await paymentRepo.updateStatus(paymentId, 'REFUNDED');
}
```

### 2. 타임아웃

```typescript
// 각 단계에 타임아웃 설정
const result = await Promise.race([
  inventoryService.reserve(items),
  timeout(5000)  // 5초 초과 시 실패 처리
]);
```

### 3. 재시도

```typescript
// 일시적 오류는 재시도
async function withRetry(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await sleep(1000 * (i + 1));  // 점점 늘어나는 대기
    }
  }
}
```

### 4. Saga 로그

실패 시 어디까지 진행됐는지 알기 위해 로그 저장

```typescript
// Saga 로그 테이블
{
  sagaId: 'saga-123',
  orderId: 'order-456',
  steps: [
    { type: 'ORDER_CREATED', status: 'COMPLETED', timestamp: '...' },
    { type: 'INVENTORY_RESERVED', status: 'COMPLETED', timestamp: '...' },
    { type: 'PAYMENT', status: 'FAILED', error: '카드 한도 초과', timestamp: '...' }
  ],
  status: 'COMPENSATING'  // PENDING, COMPLETED, COMPENSATING, FAILED
}
```

---

## Saga vs 2PC

|항목|Saga|2PC (Two-Phase Commit)|
|---|---|---|
|일관성|Eventual|Strong|
|가용성|높음|낮음 (락 필요)|
|성능|좋음|느림|
|복잡도|보상 로직 필요|코디네이터 필요|
|MSA 적합성|✅ 적합|❌ 부적합|

**MSA에서는 Saga가 표준**

---

## 언제 Saga를 쓰는가?

### 적합한 경우

|상황|이유|
|---|---|
|MSA 환경|분산 트랜잭션 필요|
|긴 트랜잭션|락 오래 잡으면 안 될 때|
|여러 서비스 연동|주문 → 재고 → 결제 → 배송|

### 부적합한 경우

|상황|이유|
|---|---|
|모놀리식|DB 트랜잭션이 더 간단|
|강한 일관성 필수|보상으로 부족할 때|
|단순한 흐름|오버엔지니어링|

---

## 핵심 요약

1. **Saga = 분산 트랜잭션 관리 패턴**
2. **실패 시 보상 트랜잭션으로 롤백**
3. **두 가지 방식**
    - Choreography: 이벤트 기반, 느슨한 결합
    - Orchestration: 중앙 제어, 명확한 흐름
4. **필수 고려**: 멱등성, 타임아웃, 재시도, 로그
5. **MSA 환경의 표준 패턴**