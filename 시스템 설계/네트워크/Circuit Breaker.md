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

```typescript
enum CircuitState {
  CLOSED = 'CLOSED',
  OPEN = 'OPEN',
  HALF_OPEN = 'HALF_OPEN'
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime: number | null = null;
  
  constructor(
    private failureThreshold: number = 5,
    private resetTimeout: number = 30000,  // 30초
    private halfOpenRequests: number = 3
  ) {}
  
  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // OPEN 상태: 즉시 거부
    if (this.state === CircuitState.OPEN) {
      if (this.shouldAttemptReset()) {
        this.state = CircuitState.HALF_OPEN;
        this.successCount = 0;
      } else {
        throw new Error('Circuit is OPEN');
      }
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  private shouldAttemptReset(): boolean {
    return (
      this.lastFailureTime !== null &&
      Date.now() - this.lastFailureTime >= this.resetTimeout
    );
  }
  
  private onSuccess(): void {
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.halfOpenRequests) {
        this.state = CircuitState.CLOSED;
        this.failureCount = 0;
      }
    } else {
      this.failureCount = 0;
    }
  }
  
  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = CircuitState.OPEN;
    }
    
    if (this.state === CircuitState.HALF_OPEN) {
      this.state = CircuitState.OPEN;
    }
  }
  
  getState(): CircuitState {
    return this.state;
  }
}
```

### 사용 예시

```typescript
const circuitBreaker = new CircuitBreaker(5, 30000, 3);

async function callExternalService() {
  try {
    const result = await circuitBreaker.execute(async () => {
      const response = await fetch('https://api.external.com/data');
      if (!response.ok) throw new Error('API Error');
      return response.json();
    });
    return result;
  } catch (error) {
    if (error.message === 'Circuit is OPEN') {
      // Fallback 로직
      return getCachedData();
    }
    throw error;
  }
}
```

---

## Sliding Window 기반 실패율 계산

단순 카운터 대신 최근 N개 요청의 실패율로 판단

```typescript
class SlidingWindowCircuitBreaker {
  private results: boolean[] = [];  // true: 성공, false: 실패
  private windowSize: number;
  private failureRateThreshold: number;
  
  constructor(windowSize = 10, failureRateThreshold = 0.5) {
    this.windowSize = windowSize;
    this.failureRateThreshold = failureRateThreshold;
  }
  
  private recordResult(success: boolean): void {
    this.results.push(success);
    if (this.results.length > this.windowSize) {
      this.results.shift();
    }
  }
  
  private getFailureRate(): number {
    if (this.results.length < this.windowSize) {
      return 0;  // 최소 요청 수 미달
    }
    const failures = this.results.filter(r => !r).length;
    return failures / this.results.length;
  }
  
  private shouldTrip(): boolean {
    return this.getFailureRate() >= this.failureRateThreshold;
  }
}
```

---

## Fallback 전략

Circuit이 OPEN일 때 대안 제공

### 1. 캐시된 데이터 반환

```typescript
async function getProductData(productId: string) {
  try {
    return await circuitBreaker.execute(() => 
      productService.getProduct(productId)
    );
  } catch (error) {
    // 캐시된 데이터 반환
    return cache.get(`product:${productId}`);
  }
}
```

### 2. 기본값 반환

```typescript
async function getRecommendations(userId: string) {
  try {
    return await circuitBreaker.execute(() => 
      recommendationService.getPersonalized(userId)
    );
  } catch (error) {
    // 인기 상품 목록 반환
    return getPopularProducts();
  }
}
```

### 3. 에러 메시지 반환

```typescript
async function processPayment(order: Order) {
  try {
    return await circuitBreaker.execute(() => 
      paymentService.charge(order)
    );
  } catch (error) {
    // 사용자에게 재시도 안내
    throw new ServiceUnavailableError(
      '결제 서비스가 일시적으로 불가합니다. 잠시 후 다시 시도해주세요.'
    );
  }
}
```

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

### 알림 설정

```
조건: circuit_state == OPEN
알림: "서비스 X의 Circuit Breaker가 OPEN 상태입니다"
```

---

## 라이브러리

### Node.js: opossum

```typescript
import CircuitBreaker from 'opossum';

const options = {
  timeout: 3000,           // 요청 타임아웃
  errorThresholdPercentage: 50,  // 실패율 임계치
  resetTimeout: 30000      // OPEN 유지 시간
};

const breaker = new CircuitBreaker(asyncFunction, options);

breaker.on('open', () => console.log('Circuit opened'));
breaker.on('halfOpen', () => console.log('Circuit half-opened'));
breaker.on('close', () => console.log('Circuit closed'));

breaker.fallback(() => 'Fallback response');

const result = await breaker.fire(args);
```

### Java: Resilience4j

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(5)
    .slidingWindowSize(10)
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("myService", config);

Supplier<String> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> myService.call());

String result = Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> "Fallback")
    .get();
```

---

## 서비스별 Circuit Breaker

각 외부 서비스마다 별도 Circuit Breaker 사용

```typescript
const circuitBreakers = {
  payment: new CircuitBreaker({ failureThreshold: 3, resetTimeout: 60000 }),
  inventory: new CircuitBreaker({ failureThreshold: 5, resetTimeout: 30000 }),
  notification: new CircuitBreaker({ failureThreshold: 10, resetTimeout: 10000 })
};

// 결제는 민감 → 낮은 임계치, 긴 대기
await circuitBreakers.payment.execute(() => paymentService.charge());

// 알림은 덜 중요 → 높은 임계치, 짧은 대기
await circuitBreakers.notification.execute(() => notificationService.send());
```

---

## 관련 패턴

### Retry + Circuit Breaker

```typescript
async function callWithRetryAndCircuitBreaker() {
  return circuitBreaker.execute(async () => {
    return retry(
      () => externalService.call(),
      { retries: 3, delay: 1000 }
    );
  });
}
```

재시도 실패도 Circuit Breaker의 실패로 카운트됨

### Timeout + Circuit Breaker

```typescript
async function callWithTimeout() {
  return circuitBreaker.execute(async () => {
    return Promise.race([
      externalService.call(),
      timeout(5000)
    ]);
  });
}
```

타임아웃도 실패로 카운트됨

### Bulkhead + Circuit Breaker

```
[요청] → [Bulkhead (동시 요청 제한)] → [Circuit Breaker] → [서비스]
```

둘 다 적용하면 더 안전

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

---
