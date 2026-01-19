## Circuit Breaker란?

외부 서비스 장애 시 빠르게 실패하여 전체 시스템 장애 전파를 방지하는 패턴

---

## 왜 필요한가?

### 문제 상황: 장애 전파 (Cascading Failure)

```
[서비스 A] → [서비스 B] → [서비스 C (장애)]
                              ↓
                         응답 없음 (30초 타임아웃)
                              ↓
[서비스 A]의 스레드 풀 고갈 → 서비스 A도 장애
```

### Circuit Breaker 적용

```
[서비스 A] → [Circuit Breaker] → [서비스 B (장애)]
                   ↓
           빠르게 실패 반환 (즉시)
                   ↓
           서비스 A는 정상 동작 유지
```

---

## 상태 머신

```
         실패율 임계치 초과
    ┌────────────────────────┐
    │                        ↓
[CLOSED] ←────────────    [OPEN]
    ↑                        │
    │                        │ 대기 시간 경과
    │                        ↓
    └──────────────────── [HALF-OPEN]
       성공                  │
                             │ 실패
                             ↓
                          [OPEN]
```

### 상태 설명

| 상태        | 동작              | 전이 조건                  |
| --------- | --------------- | ---------------------- |
| CLOSED    | 요청 정상 통과        | 실패율 > 임계치 → OPEN       |
| OPEN      | 요청 즉시 거부        | 대기 시간 경과 → HALF-OPEN   |
| HALF-OPEN | 일부 요청만 통과 (테스트) | 성공 → CLOSED, 실패 → OPEN |

---

## 핵심 파라미터

| 파라미터               | 설명            | 예시  |
| ------------------ | ------------- | --- |
| Failure Threshold  | 실패율 임계치       | 50% |
| Request Volume     | 최소 요청 수       | 10개 |
| Timeout            | OPEN 상태 유지 시간 | 30초 |
| Half-Open Requests | 테스트 요청 수      | 5개  |

---

## 구현

### 기본 구현

```java
public enum CircuitState {
    CLOSED, OPEN, HALF_OPEN
}

@Component
public class CircuitBreaker {
    private CircuitState state = CircuitState.CLOSED;
    private int failureCount = 0;
    private int successCount = 0;
    private Long lastFailureTime = null;
    
    private final int failureThreshold;
    private final long resetTimeout;
    private final int halfOpenRequests;
    
    public CircuitBreaker() {
        this(5, 30000, 3);
    }
    
    public CircuitBreaker(int failureThreshold, long resetTimeout, int halfOpenRequests) {
        this.failureThreshold = failureThreshold;
        this.resetTimeout = resetTimeout;
        this.halfOpenRequests = halfOpenRequests;
    }
    
    public <T> T execute(Supplier<T> supplier) {
        // OPEN 상태: 즉시 거부
        if (state == CircuitState.OPEN) {
            if (shouldAttemptReset()) {
                state = CircuitState.HALF_OPEN;
                successCount = 0;
            } else {
                throw new CircuitBreakerOpenException("Circuit is OPEN");
            }
        }
        
        try {
            T result = supplier.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }
    
    private synchronized boolean shouldAttemptReset() {
        return lastFailureTime != null && 
               System.currentTimeMillis() - lastFailureTime >= resetTimeout;
    }
    
    private synchronized void onSuccess() {
        if (state == CircuitState.HALF_OPEN) {
            successCount++;
            if (successCount >= halfOpenRequests) {
                state = CircuitState.CLOSED;
                failureCount = 0;
            }
        } else {
            failureCount = 0;
        }
    }
    
    private synchronized void onFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        
        if (failureCount >= failureThreshold) {
            state = CircuitState.OPEN;
        }
        
        if (state == CircuitState.HALF_OPEN) {
            state = CircuitState.OPEN;
        }
    }
    
    public CircuitState getState() {
        return state;
    }
}
```

### 사용 예시

```java
@Service
@RequiredArgsConstructor
public class ExternalApiService {
    private final CircuitBreaker circuitBreaker;
    private final RestTemplate restTemplate;
    private final CacheService cacheService;
    
    public String callExternalService() {
        try {
            return circuitBreaker.execute(() -> {
                ResponseEntity<String> response = restTemplate.getForEntity(
                    "https://api.external.com/data", String.class);
                if (!response.getStatusCode().is2xxSuccessful()) {
                    throw new RuntimeException("API Error");
                }
                return response.getBody();
            });
        } catch (CircuitBreakerOpenException e) {
            // Fallback 로직
            return cacheService.getCachedData();
        }
    }
}
```

---

## Fallback 전략

Circuit이 OPEN일 때 대안 제공

### 1. 캐시된 데이터 반환

```java
@Service
@RequiredArgsConstructor
public class ProductService {
    private final CircuitBreaker circuitBreaker;
    private final ProductClient productClient;
    private final RedisTemplate<String, Product> redisTemplate;
    
    public Product getProductData(String productId) {
        try {
            return circuitBreaker.execute(() -> productClient.getProduct(productId));
        } catch (CircuitBreakerOpenException e) {
            // 캐시된 데이터 반환
            return redisTemplate.opsForValue().get("product:" + productId);
        }
    }
}
```

### 2. 기본값 반환

```java
@Service
@RequiredArgsConstructor
public class RecommendationService {
    private final CircuitBreaker circuitBreaker;
    private final RecommendationClient recommendationClient;
    
    public List<Product> getRecommendations(String userId) {
        try {
            return circuitBreaker.execute(() -> 
                recommendationClient.getPersonalized(userId));
        } catch (CircuitBreakerOpenException e) {
            // 인기 상품 목록 반환
            return getPopularProducts();
        }
    }
    
    private List<Product> getPopularProducts() {
        // 기본 인기 상품 반환
        return List.of(/* 기본 상품들 */);
    }
}
```

### 3. 에러 메시지 반환

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final CircuitBreaker circuitBreaker;
    private final PaymentGateway paymentGateway;
    
    public PaymentResult processPayment(Order order) {
        try {
            return circuitBreaker.execute(() -> paymentGateway.charge(order));
        } catch (CircuitBreakerOpenException e) {
            throw new ServiceUnavailableException(
                "결제 서비스가 일시적으로 불가합니다. 잠시 후 다시 시도해주세요.");
        }
    }
}
```

---

## Resilience4j 사용

### 의존성 추가

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot2</artifactId>
    <version>1.7.1</version>
</dependency>
```

### 설정

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 5
        slidingWindowSize: 10
        slidingWindowType: COUNT_BASED
```

### 사용

```java
@Service
public class PaymentService {
    private final PaymentGateway paymentGateway;
    
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(Order order) {
        return paymentGateway.charge(order);
    }
    
    public PaymentResult paymentFallback(Order order, Exception e) {
        throw new ServiceUnavailableException(
            "결제 서비스가 일시적으로 불가합니다.");
    }
}
```

### 프로그래밍 방식

```java
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreaker paymentCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(5)
            .slidingWindowSize(10)
            .build();
            
        return CircuitBreaker.of("paymentService", config);
    }
}

@Service
@RequiredArgsConstructor
public class PaymentService {
    private final CircuitBreaker circuitBreaker;
    private final PaymentGateway paymentGateway;
    
    public PaymentResult processPayment(Order order) {
        Supplier<PaymentResult> supplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> paymentGateway.charge(order));
            
        return Try.ofSupplier(supplier)
            .recover(throwable -> PaymentResult.failed("서비스 일시 불가"))
            .get();
    }
}
```

---

## 서비스별 Circuit Breaker

각 외부 서비스마다 별도 Circuit Breaker 사용

```java
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        // 결제는 민감 → 낮은 임계치, 긴 대기
        CircuitBreakerConfig paymentConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(30)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .build();
            
        // 재고는 중간
        CircuitBreakerConfig inventoryConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .build();
            
        // 알림은 덜 중요 → 높은 임계치, 짧은 대기
        CircuitBreakerConfig notificationConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(70)
            .waitDurationInOpenState(Duration.ofSeconds(10))
            .build();
            
        return CircuitBreakerRegistry.of(Map.of(
            "payment", paymentConfig,
            "inventory", inventoryConfig,
            "notification", notificationConfig
        ));
    }
}
```

---

## 관련 패턴

### Retry + Circuit Breaker

```java
@Service
public class ExternalService {
    
    @Retry(name = "externalService")
    @CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
    public String call() {
        return externalClient.call();
    }
    
    public String fallback(Exception e) {
        return "Fallback response";
    }
}
```

재시도 실패도 Circuit Breaker의 실패로 카운트됨

### Timeout + Circuit Breaker

```java
@Service
public class ExternalService {
    
    @TimeLimiter(name = "externalService")
    @CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
    public CompletableFuture<String> call() {
        return CompletableFuture.supplyAsync(() -> externalClient.call());
    }
}
```

타임아웃도 실패로 카운트됨

### Bulkhead + Circuit Breaker

```java
@Service
public class ExternalService {
    
    @Bulkhead(name = "externalService")
    @CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
    public String call() {
        return externalClient.call();
    }
}
```

둘 다 적용하면 더 안전

---

## 모니터링

### 수집해야 할 메트릭

| 메트릭 | 설명 |
|--------|------|
| circuit_state | 현재 상태 (0: CLOSED, 1: OPEN, 2: HALF_OPEN) |
| failure_count | 실패 횟수 |
| success_count | 성공 횟수 |
| failure_rate | 실패율 |
| rejected_count | Circuit OPEN으로 거부된 요청 수 |

### Actuator 엔드포인트

```yaml
management:
  endpoints:
    web:
      exposure:
        include: circuitbreakers
  health:
    circuitbreakers:
      enabled: true
```

```
GET /actuator/circuitbreakers
GET /actuator/circuitbreakerevents
```

---

## Circuit Breaker vs Retry

| 항목 | Retry | Circuit Breaker |
|------|-------|-----------------|
| 목적 | 일시적 오류 복구 | 장애 전파 방지 |
| 동작 | 실패 시 재시도 | 실패 누적 시 차단 |
| 적합한 상황 | 네트워크 순간 끊김 | 외부 서비스 장애 |
| 부하 | 재시도로 부하 증가 | 빠른 실패로 부하 감소 |

---

## 핵심 요약

1. **목적**: 외부 서비스 장애 시 빠른 실패로 전체 시스템 보호
2. **상태**: CLOSED (정상) → OPEN (차단) → HALF-OPEN (테스트)
3. **파라미터**: 실패 임계치, 리셋 타임아웃, Half-Open 요청 수
4. **Fallback**: 캐시, 기본값, 에러 메시지 등 대안 제공
5. **조합**: Retry, Timeout, Bulkhead와 함께 사용
