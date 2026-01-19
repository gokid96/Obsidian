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

```java
// 주문 서비스
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = Order.builder()
            .items(request.getItems())
            .status(OrderStatus.PENDING)
            .build();
        orderRepository.save(order);
        
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId(), request.getItems()));
        return order;
    }
    
    // 주문 취소 (보상)
    @EventListener
    public void handleInventoryReserveFailed(InventoryReserveFailedEvent event) {
        orderRepository.updateStatus(event.getOrderId(), OrderStatus.CANCELLED);
        eventPublisher.publishEvent(new OrderCancelledEvent(event.getOrderId()));
    }
}

// 재고 서비스
@Service
@RequiredArgsConstructor
public class InventoryService {
    private final InventoryRepository inventoryRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            reserve(event.getItems());
            eventPublisher.publishEvent(new InventoryReservedEvent(event.getOrderId()));
        } catch (Exception e) {
            eventPublisher.publishEvent(new InventoryReserveFailedEvent(event.getOrderId()));
        }
    }
    
    // 재고 복구 (보상)
    @EventListener
    public void handlePaymentFailed(PaymentFailedEvent event) {
        release(event.getItems());
        eventPublisher.publishEvent(new InventoryReserveFailedEvent(event.getOrderId()));
    }
}

// 결제 서비스
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentGateway paymentGateway;
    private final ApplicationEventPublisher eventPublisher;
    
    @EventListener
    public void handleInventoryReserved(InventoryReservedEvent event) {
        try {
            paymentGateway.charge(event.getAmount());
            eventPublisher.publishEvent(new PaymentCompletedEvent(event.getOrderId()));
        } catch (Exception e) {
            eventPublisher.publishEvent(new PaymentFailedEvent(event.getOrderId()));
        }
    }
}
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

```java
@Service
@RequiredArgsConstructor
public class OrderSagaOrchestrator {
    private final OrderService orderService;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    private final SagaLogRepository sagaLogRepository;
    
    public Order execute(OrderRequest orderData) {
        SagaLog sagaLog = sagaLogRepository.save(new SagaLog());
        
        try {
            // Step 1: 주문 생성
            Order order = orderService.create(orderData);
            logStep(sagaLog, StepType.ORDER_CREATED, order.getId());
            
            // Step 2: 재고 차감
            inventoryService.reserve(order.getItems());
            logStep(sagaLog, StepType.INVENTORY_RESERVED, order.getId());
            
            // Step 3: 결제
            paymentService.charge(order.getTotalAmount());
            logStep(sagaLog, StepType.PAYMENT_COMPLETED, order.getId());
            
            // Step 4: 배송 준비
            shippingService.prepare(order.getId());
            logStep(sagaLog, StepType.SHIPPING_PREPARED, order.getId());
            
            completeSaga(sagaLog);
            return order;
            
        } catch (Exception e) {
            compensate(sagaLog);
            throw e;
        }
    }
    
    private void compensate(SagaLog sagaLog) {
        // 역순으로 보상 트랜잭션 실행
        List<SagaStep> steps = sagaLog.getCompletedSteps();
        Collections.reverse(steps);
        
        for (SagaStep step : steps) {
            switch (step.getType()) {
                case SHIPPING_PREPARED:
                    shippingService.cancel(step.getOrderId());
                    break;
                case PAYMENT_COMPLETED:
                    paymentService.refund(step.getPaymentId());
                    break;
                case INVENTORY_RESERVED:
                    inventoryService.release(step.getItems());
                    break;
                case ORDER_CREATED:
                    orderService.cancel(step.getOrderId());
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

---

## 고려 사항

### 1. 멱등성

보상 트랜잭션이 여러 번 실행되어도 안전해야 함

```java
// 멱등하지 않음
public void refund(String paymentId) {
    paymentGateway.refund(paymentId);  // 중복 환불 위험
}

// 멱등함
public void refund(String paymentId) {
    Payment payment = paymentRepository.findById(paymentId).orElseThrow();
    if (payment.getStatus() == PaymentStatus.REFUNDED) {
        return;  // 이미 처리됨
    }
    
    paymentGateway.refund(paymentId);
    payment.setStatus(PaymentStatus.REFUNDED);
    paymentRepository.save(payment);
}
```

### 2. 타임아웃

```java
// 각 단계에 타임아웃 설정
public void reserveWithTimeout(List<Item> items) {
    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> 
        inventoryService.reserve(items)
    );
    
    try {
        future.get(5, TimeUnit.SECONDS);  // 5초 초과 시 실패
    } catch (TimeoutException e) {
        throw new SagaStepTimeoutException("Inventory reservation timeout");
    }
}
```

### 3. 재시도

```java
// 일시적 오류는 재시도
@Retryable(value = TransientException.class, maxAttempts = 3, backoff = @Backoff(delay = 1000, multiplier = 2))
public void reserveWithRetry(List<Item> items) {
    inventoryService.reserve(items);
}
```

### 4. Saga 로그

실패 시 어디까지 진행됐는지 알기 위해 로그 저장

```java
@Entity
public class SagaLog {
    @Id
    private String sagaId;
    private String orderId;
    
    @OneToMany(cascade = CascadeType.ALL)
    private List<SagaStep> steps;
    
    @Enumerated(EnumType.STRING)
    private SagaStatus status;  // PENDING, COMPLETED, COMPENSATING, FAILED
}

@Entity
public class SagaStep {
    @Id
    private Long id;
    
    @Enumerated(EnumType.STRING)
    private StepType type;
    
    @Enumerated(EnumType.STRING)
    private StepStatus status;
    
    private String errorMessage;
    private LocalDateTime timestamp;
}
```

---

## 보상 트랜잭션 실패 처리

### 문제 상황

```
주문 생성 → 재고 차감 → 결제 실패!
                         ↓
              재고 복구 실패!  ← 보상도 실패하면?
```

### 해결 방법

**1. 재시도 큐**

```java
@Service
@RequiredArgsConstructor
public class CompensationService {
    private final RabbitTemplate rabbitTemplate;
    
    public void compensateWithRetry(Runnable compensation, int maxRetries) {
        for (int i = 0; i < maxRetries; i++) {
            try {
                compensation.run();
                return;
            } catch (Exception e) {
                if (i == maxRetries - 1) {
                    // DLQ로 전송
                    rabbitTemplate.convertAndSend("dead-letter-queue", 
                        new FailedCompensation(compensation.toString(), e.getMessage()));
                }
                try {
                    Thread.sleep((long) Math.pow(2, i) * 1000);  // 지수 백오프
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }
}
```

**2. Dead Letter Queue (DLQ)**

보상 실패 건을 별도 큐에 저장 후 수동/자동 처리

```java
@Component
@RequiredArgsConstructor
public class DlqProcessor {
    private final FailedCompensationRepository repository;
    private final AlertService alertService;
    
    @Scheduled(fixedRate = 60000)  // 1분마다 실행
    public void processDLQ() {
        List<FailedCompensation> failedItems = repository.findAll();
        
        for (FailedCompensation item : failedItems) {
            try {
                retryCompensation(item);
                repository.delete(item);
            } catch (Exception e) {
                // 알림 발송 후 수동 처리 필요
                alertService.notifyOps(item);
            }
        }
    }
}
```

**3. 수동 복구 대시보드**

자동 복구 불가 시 운영자가 직접 처리

```
[Saga Dashboard]
- 실패한 Saga 목록
- 각 단계 상태 확인
- 수동 보상 실행 버튼
- 강제 완료 처리
```

---

## Saga vs 2PC

|항목|Saga|2PC (Two-Phase Commit)|
|---|---|---|
|일관성|Eventual|Strong|
|가용성|높음|낮음 (락 필요)|
|성능|좋음|느림|
|복잡도|보상 로직 필요|코디네이터 필요|
|MSA 적합성|적합|부적합|

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
