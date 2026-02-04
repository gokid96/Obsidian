CQRS는 **명령(Command)과 조회(Query)를 분리하는 패턴**으로, 
읽기가 많고 조회 성능 최적화가 필요할 때 사용한다.

---
## 문제 상황

일반적인 CRUD에서는 같은 모델로 읽기/쓰기를 다 처리한다.
```java
// 같은 Entity, 같은 Repository
Order order = orderRepository.findById(id);  // 조회
orderRepository.save(order);                  // 저장
```

근데 현실에서는 읽기와 쓰기의 요구사항이 다르다.
```
쓰기: 주문 생성 시 복잡한 비즈니스 검증 필요
읽기: 주문 목록 조회 시 여러 테이블 조인 + 빠른 응답 필요
```

하나의 모델로 둘 다 처리하려니 어느 쪽도 최적화가 안 된다.

---

## 해결: Command와 Query 분리
```
                     Client
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
     ┌─────────┐                 ┌─────────┐
     │ Command │                 │  Query  │
     │  (쓰기) │                 │  (읽기) │
     └────┬────┘                 └────┬────┘
          │                           │
          ▼                           ▼
     ┌─────────┐                 ┌─────────┐
     │Write DB │ ───── 동기화 ──▶│ Read DB │
     │ (정규화)│                 │(비정규화)│
     └─────────┘                 └─────────┘
````

---

## 각 모델의 역할

### Command (쓰기)
```java
public class CreateOrderCommand {
    private String customerId;
    private List<OrderItem> items;
    private String shippingAddress;
}

@Service
public class OrderCommandService {
    public void handle(CreateOrderCommand cmd) {
        // 복잡한 비즈니스 로직
        validateCustomer(cmd.getCustomerId());
        checkStock(cmd.getItems());
        calculatePrice(cmd.getItems());
        
        Order order = Order.create(cmd);
        orderRepository.save(order);
        
        // 이벤트 발행 → Read DB 동기화
        eventPublisher.publish(new OrderCreatedEvent(order));
    }
}
```

### Query (읽기)
```java
public class OrderSummaryDto {
    private Long orderId;
    private String customerName;  // 조인 없이 바로
    private String productNames;  // 미리 합쳐둠
    private int totalPrice;
}

@Service
public class OrderQueryService {
    public List<OrderSummaryDto> getOrders(String customerId) {
        // 단순 조회, 비즈니스 로직 없음
        return orderReadRepository.findByCustomerId(customerId);
    }
}
```

---

## 동기화 방식

### 1. 이벤트 기반 (비동기)
```
Write DB 저장 → 이벤트 발행 → Read DB 업데이트

┌──────────┐      ┌─────────┐      ┌──────────┐
│ Command  │ ──▶  │  Kafka  │ ──▶  │  Query   │
│ Service  │      │         │      │ Updater  │
└──────────┘      └─────────┘      └──────────┘
````

```java
@KafkaListener(topics = "order-events")
public void updateReadModel(OrderCreatedEvent event) {
    OrderSummary summary = OrderSummary.from(event);
    orderSummaryRepository.save(summary);
}
```

### 2. DB 동기화 (동기)

```java
@Transactional
public void createOrder(CreateOrderCommand cmd) {
    orderRepository.save(order);           // Write DB
    orderSummaryRepository.save(summary);  // Read DB (같은 트랜잭션)
}
```

---

## Read DB 구조 예시

**Write DB (정규화)**
```
orders: id, customer_id, status, created_at
order_items: id, order_id, product_id, quantity, price
customers: id, name, email
products: id, name, price
```

**Read DB (비정규화)**
```
order_summaries: id, customer_name, product_names, total_price, status, created_at
```

조회 시 조인 없이 한 번에 가져옴.

---

## 언제 쓰나?

| 상황 | CQRS 필요 |
|------|-----------|
| 읽기/쓰기 비율 차이 큼 (읽기 90%, 쓰기 10%) | ✓ |
| 조회 성능 최적화 필요 | ✓ |
| 복잡한 도메인 로직 | ✓ |
| 단순 CRUD | ✗ 오버엔지니어링 |

---

## 장단점

| 장점 | 단점 |
|------|------|
| 읽기/쓰기 각각 최적화 | 복잡도 증가 |
| 조회 성능 향상 | 데이터 동기화 관리 필요 |
| 스케일 아웃 용이 | Eventual Consistency |
| 도메인 로직 분리 | 러닝 커브 |

---

## Event Sourcing과의 관계

CQRS는 종종 Event Sourcing과 함께 쓰인다.
```
Event Sourcing: 상태 변경을 이벤트로 저장
CQRS: 그 이벤트를 읽기 모델로 변환

주문 생성 이벤트 → Write Store (이벤트 저장)
                        ↓
                  Read Model 업데이트
````

하지만 CQRS 단독으로도 충분히 쓸 수 있다.

---

## 관련 개념

- [[Event Sourcing]]
- [[Event-Driven Architecture]]
- [[Saga Pattern]]