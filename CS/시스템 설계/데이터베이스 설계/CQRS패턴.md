## CQRS란?

**Command Query Responsibility Segregation**

읽기(Query)와 쓰기(Command)를 분리하는 아키텍처 패턴

---

## 기본 개념

### 일반적인 구조 (CRUD)

```
[클라이언트]
     │
     ▼
[하나의 모델]
     │
     ▼
[하나의 DB]
```

- 읽기/쓰기가 같은 모델, 같은 DB 사용
- 간단하지만 확장에 한계
### CQRS 구조

```
[클라이언트]
     │
     ├──── 쓰기 (Command) ────→ [Write Model] → [Write DB]
     │                                              │
     │                                         동기화 (이벤트)
     │                                              │
     └──── 읽기 (Query) ──────→ [Read Model] ← [Read DB]
```

- 읽기/쓰기가 각각 다른 모델, 다른 DB 사용
- 각각 독립적으로 최적화 가능

---

## 왜 분리하는가?

### 읽기와 쓰기는 요구사항이 다름

|구분|읽기 (Query)|쓰기 (Command)|
|---|---|---|
|목적|빠른 조회|데이터 일관성|
|빈도|많음 (80~99%)|적음 (1~20%)|
|모델|비정규화 (조인 없이 빠르게)|정규화 (무결성)|
|스케일링|수평 확장 (replica)|수직 확장 or 샤딩|

### 하나의 모델로는 둘 다 만족 어려움

```
정규화하면 → 쓰기 좋음, 읽기 느림 (조인 많음)
비정규화하면 → 읽기 좋음, 쓰기 복잡 (중복 갱신)
```


→ **분리하면 각각 최적화 가능**

---

## 구현 방식
### 레벨 1: 코드 레벨 분리 (간단)

같은 DB, 다른 모델

```
📁 src
   ├── 📁 command
   │      ├── CreateOrderCommand.java
   │      └── UpdateOrderCommand.java
   └── 📁 query
          ├── GetOrderQuery.java
          └── ListOrdersQuery.java
```

```java
// Command (쓰기)
@Service
@RequiredArgsConstructor
public class CreateOrderCommand {
    private final OrderRepository orderRepository;
    private final OrderItemRepository orderItemRepository;
    
    @Transactional
    public void execute(OrderInput data) {
        // 검증, 비즈니스 로직
        // 정규화된 테이블에 저장
        Order order = Order.from(data);
        orderRepository.save(order);
        orderItemRepository.saveAll(order.getItems());
    }
}

// Query (읽기)
@Service
@RequiredArgsConstructor
public class GetOrderQuery {
    private final OrderReadRepository orderReadRepository;
    
    public OrderDto execute(String orderId) {
        // 조인된 뷰 또는 비정규화 테이블에서 조회
        return orderReadRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

**장점**: 구현 쉬움, 기존 시스템에 적용 가능 **단점**: DB 부하 분산 안 됨

---

### 레벨 2: DB 분리 (중간)

Write DB → Read DB 동기화

```
[Command] → [Write DB (MySQL)]
                   │
                   │ 복제 (Replication)
                   ▼
[Query] ← [Read DB (MySQL Replica)]
```

```java
// 쓰기는 Master로
@Service
@RequiredArgsConstructor
public class OrderCommandService {
    
    @PersistenceContext(unitName = "master")
    private EntityManager masterEntityManager;
    
    @Transactional
    public void createOrder(OrderInput input) {
        Order order = Order.from(input);
        masterEntityManager.persist(order);
    }
}

// 읽기는 Replica로
@Service
@RequiredArgsConstructor
public class OrderQueryService {
    
    @PersistenceContext(unitName = "replica")
    private EntityManager replicaEntityManager;
    
    @Transactional(readOnly = true)
    public OrderDto findById(String orderId) {
        return replicaEntityManager.find(Order.class, orderId);
    }
}
```

**장점**: 읽기 부하 분산 **단점**: 복제 지연 (몇 ms ~ 몇 초)

---

### 레벨 3: 이벤트 소싱 + 다른 DB (고급)

Write DB와 Read DB가 완전히 다른 종류

```
[Command] → [Write DB (PostgreSQL)] 
                   │
                   │ 이벤트 발행 (Kafka)
                   ▼
            [Event Handler]
                   │
                   ▼
[Query] ← [Read DB (Elasticsearch, Redis)]
```

```java
// 쓰기: 이벤트 발행
@Service
@RequiredArgsConstructor
public class CreateOrderCommand {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void execute(OrderInput data) {
        Order order = orderRepository.save(Order.from(data));
        
        // 이벤트 발행
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
    }
}

// 이벤트 핸들러: Read DB 갱신
@Component
@RequiredArgsConstructor
public class OrderCreatedHandler {
    private final ElasticsearchOperations elasticsearchOperations;
    
    @EventListener
    @Async
    public void handle(OrderCreatedEvent event) {
        // Elasticsearch에 비정규화된 형태로 저장
        OrderDocument doc = OrderDocument.builder()
            .orderId(event.getOrder().getId())
            .customerName(event.getOrder().getCustomer().getName())
            .items(event.getOrder().getItems())
            .totalAmount(event.getOrder().getTotalAmount())
            .build();
            
        elasticsearchOperations.save(doc);
    }
}

// 읽기: Elasticsearch에서 빠르게 조회
@Service
@RequiredArgsConstructor
public class SearchOrdersQuery {
    private final ElasticsearchOperations elasticsearchOperations;
    
    public List<OrderDocument> execute(String keyword) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q.match(m -> m.field("customerName").query(keyword)))
            .build();
            
        return elasticsearchOperations.search(query, OrderDocument.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }
}
```

**장점**: 각 DB 특성에 맞게 최적화 (검색은 ES, 캐시는 Redis) **단점**: 복잡도 높음, 동기화 지연

---

## 동기화 방식

### 1. 동기식 (Sync)

```
쓰기 → Write DB 저장 → Read DB 갱신 → 응답
```

- 일관성 보장
- 응답 느림

### 2. 비동기식 (Async) - 권장

```
쓰기 → Write DB 저장 → 응답
                │
                └→ 이벤트 → Read DB 갱신 (백그라운드)
```

- 응답 빠름
- 일시적 불일치 (Eventual Consistency)

### 동기화 도구

| 도구             | 설명                     |
| -------------- | ---------------------- |
| DB Replication | MySQL/PostgreSQL 기본 복제 |
| Kafka          | 이벤트 스트리밍               |
| Debezium       | DB 변경 감지 (CDC)         |
| Redis Pub/Sub  | 간단한 이벤트 전파             |

---

## 실전 예시

### 예시 1: 이커머스 주문 시스템

```
[주문 생성 Command]
     │
     ▼
[Write DB - PostgreSQL]
├── orders (정규화)
├── order_items
└── payments
     │
     │ Kafka 이벤트
     ▼
[Read DB - Elasticsearch]
└── orders_view (비정규화, 검색 최적화)
     │
     ▼
[주문 검색 Query] - 고객명, 상품명으로 빠른 검색
```

### 예시 2: SNS 피드

```
[게시글 작성 Command]
     │
     ▼
[Write DB - MySQL]
├── posts
├── users
└── follows
     │
     │ 이벤트
     ▼
[Read DB - Redis]
└── feed:{userId} (미리 계산된 피드)
     │
     ▼
[피드 조회 Query] - 조인 없이 바로 반환
```

---

## 언제 CQRS를 쓰는가?

### 적합한 경우

|상황|이유|
|---|---|
|읽기/쓰기 비율 차이 큼|각각 다르게 스케일링|
|복잡한 조회 요구사항|읽기 모델 따로 최적화|
|높은 트래픽|부하 분산 필요|
|도메인이 복잡|관심사 분리로 유지보수 용이|

### 부적합한 경우

|상황|이유|
|---|---|
|단순한 CRUD|오버엔지니어링|
|강한 일관성 필수|동기화 지연 허용 불가|
|소규모 서비스|복잡도 대비 이점 적음|

---

## 트레이드오프

|장점|단점|
|---|---|
|읽기/쓰기 독립 최적화|복잡도 증가|
|확장성 향상|동기화 로직 필요|
|각 DB 특성 활용|Eventual Consistency|
|관심사 분리|학습 곡선|

---

## Eventual Consistency 허용 범위

동기화 지연이 비즈니스에 미치는 영향 판단 필요

| 데이터 유형 | 허용 지연 | 설명 |
|------------|----------|------|
| 상품 정보 | 수 초 | 상품명, 설명 변경은 즉각적이지 않아도 됨 |
| 재고 수량 | 수백 ms | 재고 부족 표시 지연은 문제될 수 있음 |
| 결제 상태 | 허용 안 됨 | 강한 일관성 필요 |
| 사용자 프로필 | 수 초 | 본인 외에는 즉각 반영 불필요 |
| 좋아요/조회수 | 수 분 | 정확한 실시간 데이터 불필요 |

**판단 기준**: 지연으로 인한 장애가 비즈니스적 손실로 이어지는지 여부

---

## 핵심 요약

1. **CQRS = 읽기/쓰기 분리**
2. **단계적 적용 가능**: 코드 분리 → DB 분리 → 이벤트 소싱
3. **비동기 동기화**가 일반적 (Eventual Consistency 허용)
4. **복잡한 시스템에 적합**, 단순 CRUD는 오버엔지니어링
5. **읽기 많은 서비스**에서 효과 큼
