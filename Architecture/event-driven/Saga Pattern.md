
Saga 패턴은  **MSA 분산 환경에서 트랜잭션을 관리하는 방법**이다.
서비스마다 DB가 분리되어 있어 하나의 트랜잭션으로 묶을 수가 없어서 Saga 패턴을 사용

---
## 문제 상황

주문 처리에 여러 서비스가 관여한다고 해보자.

```
주문 생성 → 결제 → 재고 차감 → 배송 요청
```

만약 재고 차감에서 실패하면? 이미 완료된 결제를 취소해야 한다. 근데 서비스가 다 분리되어 있으니 DB 트랜잭션으로 한번에 롤백할 수가 없다.

---

## 해결 방식

각 단계마다 **보상 트랜잭션(Compensating Transaction)**을 정의해둔다.

```
주문 생성  ←  주문 취소
결제 완료  ←  결제 환불
재고 차감  ←  재고 복구
배송 요청  ←  배송 취소
```

중간에 실패하면 이미 완료된 작업들을 역순으로 보상 처리한다.

---

## 예시 흐름

```
정상: 주문 → 결제 ✓ → 재고 ✓ → 배송 ✓ → 완료

실패: 주문 → 결제 ✓ → 재고 ✗ (품절)
                ↓
      결제 환불 ← 주문 취소
```

---

## 구현 방식

### 1. Choreography (코레오그래피)

각 서비스가 이벤트를 발행하고 다음 서비스가 알아서 반응한다. 중앙 관리자가 없다.

```
order-service → "주문 생성됨" 발행
                      ↓
payment-service → "결제 완료됨" 발행
                      ↓
stock-service → "재고 차감됨" 발행
                      ↓
delivery-service → "배송 요청됨" 발행
```

**장점:** 서비스 간 결합도 낮음, 단순한 흐름에 적합
**단점:** 전체 흐름 파악 어려움, 복잡해지면 관리 힘듦

### 2. Orchestration (오케스트레이션)

중앙 Orchestrator가 전체 흐름을 제어한다.

```
┌─────────────────┐
│   Orchestrator  │
│   (Saga 관리자)  │
└────────┬────────┘
         │
    ┌────┴────┬──────────┬──────────┐
    ↓         ↓          ↓          ↓
 order    payment     stock     delivery
 service  service    service    service
```

```java
// Orchestrator 내부 로직
public void processSaga(OrderEvent event) {
    try {
        orderService.create(event);
        paymentService.pay(event);
        stockService.decrease(event);
        deliveryService.request(event);
    } catch (Exception e) {
        // 실패 시 보상 트랜잭션 실행
        compensate(event);
    }
}

private void compensate(OrderEvent event) {
    deliveryService.cancel(event);
    stockService.restore(event);
    paymentService.refund(event);
    orderService.cancel(event);
}
```

**장점:** 전체 흐름을 한 곳에서 관리, 복잡한 비즈니스 로직에 적합
**단점:** Orchestrator가 단일 장애점(SPOF)이 될 수 있음

**Orchestration**은 간단하면 직접 만들고, 복잡한 비즈니스 로직이면 프레임워크를 쓰는 경우가 많음

| 프레임워크          | 설명                      |
| -------------- | ----------------------- |
| Axon Framework | Java/Spring 기반, Saga 지원 |
| Camunda        | BPMN 기반 워크플로우 엔진        |
| Temporal       | 분산 워크플로우 관리             |

---

## Choreography vs Orchestration

| |Choreography|Orchestration|
|---|---|---|
|중앙 관리자|없음|있음|
|서비스 결합도|낮음|중간|
|흐름 파악|어려움|쉬움|
|적합한 경우|단순한 흐름|복잡한 비즈니스 로직|
## 이벤트 드리븐만 쓰는 경우

트랜잭션 보장이 필요 없으면 Saga 없이도 됨.

```
주문 생성 → Kafka → 알림 발송
                 → 로그 저장
                 → 통계 수집
```
## Saga가 필요한 경우

여러 서비스가 **하나의 비즈니스 트랜잭션**으로 묶여야 할 때.

```
주문 → 결제 → 재고 차감
```
---

## 관련 개념

- [[Saga 패턴]]

---
