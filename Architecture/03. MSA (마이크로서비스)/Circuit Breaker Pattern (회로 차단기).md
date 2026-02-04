Circuit Breaker는 **외부 서비스 호출 실패가 반복되면 호출을 차단해서 장애 전파를 막는 패턴**이다. Resilience4j, Hystrix 등의 라이브러리로 구현한다.

---

## 문제 상황

```java
// 주문 서비스 → 재고 서비스 호출
public void createOrder(Order order) {
    orderRepository.save(order);
    
    // 재고 서비스가 죽어있다면?
    restTemplate.post("http://inventory-service/deduct", order);  // 계속 대기...
}
```

재고 서비스 장애 시
```
사용자 요청 → 주문 서비스 → 재고 서비스 (응답 없음)
                   ↓
              스레드 점유 (30초 대기)
                   ↓
              스레드 풀 고갈
                   ↓
              주문 서비스도 장애 ← 장애 전파!
```

---

## 해결: Circuit Breaker

전기 회로의 차단기처럼, **실패가 반복되면 호출 자체를 차단**한다.
```
정상 상태        실패 감지         회로 열림
    │               │                │
    ▼               ▼                ▼
[CLOSED] ──실패 N회──▶ [OPEN] ──일정 시간──▶ [HALF-OPEN]
    ▲                                            │
    │                                            │
    └────────────성공하면 CLOSED로────────────────┘
                   실패하면 다시 OPEN
```

---

## 3가지 상태

| 상태            | 동작     | 설명                     |
| ------------- | ------ | ---------------------- |
| **CLOSED**    | 정상 호출  | 요청 통과, 실패 횟수 카운트       |
| **OPEN**      | 즉시 실패  | 호출 안 함, 바로 fallback 반환 |
| **HALF-OPEN** | 테스트 호출 | 일부 요청만 통과시켜 복구 확인      |

---

## 코드 예시 (Resilience4j)


```java
// 의존성 추가
// implementation 'io.github.resilience4j:resilience4j-spring-boot2'

@Service
public class OrderService {
    
    @CircuitBreaker(name = "inventory", fallbackMethod = "fallback")
    public void deductInventory(Order order) {
        restTemplate.post("http://inventory-service/deduct", order);
    }
    
    // 회로 열리면 이게 실행됨
    public void fallback(Order order, Exception e) {
        log.warn("재고 서비스 장애, 나중에 처리: {}", order.getId());
        pendingQueue.add(order);  // 큐에 저장해두고 나중에 재시도
    }
}
```

---

## 설정 예시 (application.yml)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory:
        failureRateThreshold: 50          # 실패율 50% 넘으면 OPEN
        slowCallRateThreshold: 100        # 느린 호출 비율
        slowCallDurationThreshold: 3s     # 3초 이상이면 느린 호출
        waitDurationInOpenState: 10s      # OPEN 상태 유지 시간
        permittedNumberOfCallsInHalfOpenState: 3  # HALF-OPEN에서 테스트 호출 수
        slidingWindowSize: 10             # 최근 10개 요청 기준으로 판단
```

---

## 동작 흐름
```
1. 정상 상태 (CLOSED)
   요청 1 ✓ 성공
   요청 2 ✓ 성공
   요청 3 ✗ 실패
   요청 4 ✗ 실패
   요청 5 ✗ 실패  ← 실패율 50% 초과!

2. 회로 열림 (OPEN)
   요청 6 → 호출 안 함 → 바로 fallback 실행
   요청 7 → 호출 안 함 → 바로 fallback 실행
   (10초 대기...)

3. 반열림 (HALF-OPEN)
   요청 8 → 테스트 호출 → 성공!
   요청 9 → 테스트 호출 → 성공!
   요청 10 → 테스트 호출 → 성공!

4. 회로 닫힘 (CLOSED)
   정상 운영 재개
````

---

## Fallback 전략들

|전략|설명|예시|
|---|---|---|
|**캐시 반환**|이전 성공 데이터 반환|상품 정보 캐시|
|**기본값 반환**|디폴트 값 반환|추천 상품 = 빈 리스트|
|**큐에 저장**|나중에 재처리|재고 차감 → 큐 저장|
|**대체 서비스**|백업 서비스 호출|결제 → 백업 PG사|

---

## Circuit Breaker vs Retry

|패턴|목적|사용 시점|
|---|---|---|
|**Retry**|일시적 실패 극복|네트워크 순단, 타임아웃|
|**Circuit Breaker**|장애 전파 방지|서비스 다운, 지속적 실패|

```java
// 보통 같이 씀
@Retry(name = "inventory")
@CircuitBreaker(name = "inventory", fallbackMethod = "fallback")
public void deductInventory(Order order) {
    // 먼저 Retry 시도 → 계속 실패하면 Circuit Breaker 발동
}
```

---

## 장단점

|장점|단점|
|---|---|
|장애 전파 방지|설정 튜닝 필요 (임계값 등)|
|빠른 실패 (사용자 대기 ↓)|fallback 로직 구현 필요|
|서비스 복구 시간 확보|모니터링 필수|

---

## 관련 개념

- [[01. Microservices Architecture (기본 개념)]]
- [[Retry Pattern]]
- [[Bulkhead Pattern]] - 격리 패턴
- [[Timeout Pattern]]

---
