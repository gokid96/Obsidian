## Rate Limiting이란?

일정 시간 동안 허용되는 요청 수를 제한하는 기술

---

## 왜 필요한가?

### 문제 상황

```
악의적 사용자 → 초당 10,000 요청 → 서버 다운
```

### Rate Limiting 적용

```
악의적 사용자 → 초당 10,000 요청 → [Rate Limiter] → 100개만 통과
                                              ↓
                                          9,900개 거부 (429)
```

**효과**
- DDoS 공격 방어
- API 남용 방지
- 리소스 공정 분배
- 비용 제어

---

## 주요 알고리즘

### 1. Fixed Window

고정된 시간 윈도우 내 요청 수 제한

```
윈도우: 1분, 제한: 100개

00:00 ~ 00:59 → 100개 허용
01:00 ~ 01:59 → 100개 허용 (카운터 리셋)
```

**구현**

```java
@Component
public class FixedWindowLimiter {
    private final Map<String, WindowRecord> counts = new ConcurrentHashMap<>();
    
    public boolean isAllowed(String key, int limit, long windowMs) {
        long now = System.currentTimeMillis();
        long windowStart = (now / windowMs) * windowMs;
        
        WindowRecord record = counts.compute(key, (k, v) -> {
            if (v == null || v.windowStart != windowStart) {
                return new WindowRecord(1, windowStart);
            }
            if (v.count < limit) {
                return new WindowRecord(v.count + 1, windowStart);
            }
            return v;
        });
        
        return record.count <= limit;
    }
    
    @AllArgsConstructor
    private static class WindowRecord {
        int count;
        long windowStart;
    }
}
```

| 장점 | 단점 |
|------|------|
| 구현 간단 | 경계 문제 (boundary problem) |
| 메모리 효율 | 버스트 트래픽에 취약 |

**경계 문제 예시**
```
제한: 분당 100개

00:59 → 100개 요청
01:00 → 100개 요청 (리셋됨)

→ 2초 사이에 200개 요청 통과
```

---

### 2. Sliding Window Log

요청 타임스탬프를 모두 저장하고 윈도우 내 요청 수 계산

```
현재: 01:30
윈도우: 1분

저장된 타임스탬프:
- 00:25 (윈도우 밖, 제외)
- 01:00 (윈도우 내)
- 01:15 (윈도우 내)
- 01:28 (윈도우 내)

→ 윈도우 내 요청: 3개
```

**구현**

```java
@Component
public class SlidingWindowLogLimiter {
    private final Map<String, List<Long>> logs = new ConcurrentHashMap<>();
    
    public synchronized boolean isAllowed(String key, int limit, long windowMs) {
        long now = System.currentTimeMillis();
        long windowStart = now - windowMs;
        
        List<Long> timestamps = logs.computeIfAbsent(key, k -> new ArrayList<>());
        
        // 윈도우 밖 타임스탬프 제거
        timestamps.removeIf(t -> t <= windowStart);
        
        if (timestamps.size() < limit) {
            timestamps.add(now);
            return true;
        }
        
        return false;
    }
}
```

| 장점 | 단점 |
|------|------|
| 정확함 | 메모리 사용량 높음 |
| 경계 문제 없음 | 타임스탬프 저장 비용 |

---

### 3. Sliding Window Counter

Fixed Window + 이전 윈도우 비율 계산 (근사치)

```
현재: 01:30 (현재 윈도우 50% 지점)
제한: 분당 100개

이전 윈도우 (00:00~00:59): 80개
현재 윈도우 (01:00~01:59): 30개

계산: 80 * 0.5 + 30 = 70개
→ 아직 30개 여유
```

**구현**

```java
@Component
public class SlidingWindowCounterLimiter {
    private final Map<String, WindowCounterRecord> windows = new ConcurrentHashMap<>();
    
    public boolean isAllowed(String key, int limit, long windowMs) {
        long now = System.currentTimeMillis();
        long windowStart = (now / windowMs) * windowMs;
        double windowProgress = (double) (now - windowStart) / windowMs;
        
        WindowCounterRecord record = windows.compute(key, (k, v) -> {
            if (v == null || v.windowStart < windowStart - windowMs) {
                return new WindowCounterRecord(0, 0, windowStart);
            } else if (v.windowStart < windowStart) {
                return new WindowCounterRecord(v.curr, 0, windowStart);
            }
            return v;
        });
        
        double estimatedCount = record.prev * (1 - windowProgress) + record.curr;
        
        if (estimatedCount < limit) {
            record.curr++;
            return true;
        }
        
        return false;
    }
    
    @Data
    @AllArgsConstructor
    private static class WindowCounterRecord {
        int prev;
        int curr;
        long windowStart;
    }
}
```

| 장점 | 단점 |
|------|------|
| 메모리 효율 | 근사치 (완벽하지 않음) |
| 경계 문제 완화 | - |

**가장 많이 사용되는 방식**

---

### 4. Token Bucket

버킷에 토큰이 일정 속도로 채워지고, 요청 시 토큰 소비

```
버킷 용량: 10개
충전 속도: 1개/초

[토큰: 10개]
    │
    ├─ 요청 3개 → 토큰 7개 남음
    │
    │ (2초 후, 2개 충전)
    │
    ├─ 토큰: 9개
    │
    ├─ 요청 15개 → 9개 허용, 6개 거부
    │
    └─ 토큰: 0개
```

**구현**

```java
@Component
public class TokenBucketLimiter {
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    public boolean isAllowed(String key, int bucketSize, double refillRate) {
        long now = System.currentTimeMillis();
        
        Bucket bucket = buckets.compute(key, (k, v) -> {
            if (v == null) {
                return new Bucket(bucketSize, now);
            }
            
            // 토큰 충전
            double elapsed = (now - v.lastRefill) / 1000.0;
            double newTokens = Math.min(bucketSize, v.tokens + elapsed * refillRate);
            return new Bucket(newTokens, now);
        });
        
        if (bucket.tokens >= 1) {
            bucket.tokens -= 1;
            return true;
        }
        
        return false;
    }
    
    @Data
    @AllArgsConstructor
    private static class Bucket {
        double tokens;
        long lastRefill;
    }
}
```

| 장점 | 단점 |
|------|------|
| 버스트 허용 (유연함) | 구현 복잡도 |
| 평균 속도 제한 | 파라미터 튜닝 필요 |

**AWS, Stripe 등에서 사용**

---

### 5. Leaky Bucket

요청이 버킷에 들어가고, 일정 속도로 처리 (큐처럼 동작)

```
버킷 용량: 10개
처리 속도: 1개/초

요청 5개 도착 → 버킷에 5개 대기
              → 1초마다 1개씩 처리

요청 20개 도착 → 10개만 버킷에, 10개 거부
```

| 장점 | 단점 |
|------|------|
| 출력 속도 일정 | 버스트 처리 불가 |
| 트래픽 평활화 | 지연 발생 |

**Token Bucket과 차이**
- Token Bucket: 버스트 허용, 유연함
- Leaky Bucket: 출력 속도 일정, 엄격함

---

## 알고리즘 비교

| 알고리즘 | 정확도 | 메모리 | 버스트 허용 | 복잡도 |
|----------|--------|--------|-------------|--------|
| Fixed Window | 낮음 | 낮음 | 불가 | 낮음 |
| Sliding Window Log | 높음 | 높음 | 불가 | 중간 |
| Sliding Window Counter | 중간 | 낮음 | 불가 | 중간 |
| Token Bucket | 높음 | 낮음 | 가능 | 중간 |
| Leaky Bucket | 높음 | 중간 | 불가 | 중간 |

---

## 분산 환경에서의 Rate Limiting

### 문제

```
[서버 1] → 로컬 카운터: 50개
[서버 2] → 로컬 카운터: 50개

→ 실제로는 100개 요청이 통과됨
```

### 해결: 중앙 집중식 저장소

```
[서버 1] ──┐
           ├──→ [Redis] ← 공유 카운터
[서버 2] ──┘
```

**Redis 구현 예시**

```java
@Component
@RequiredArgsConstructor
public class DistributedRateLimiter {
    private final RedisTemplate<String, String> redisTemplate;
    
    public boolean isAllowed(String key, int limit, long windowMs) {
        long windowStart = System.currentTimeMillis() / windowMs;
        String windowKey = "ratelimit:" + key + ":" + windowStart;
        
        Long count = redisTemplate.opsForValue().increment(windowKey);
        
        if (count == 1) {
            redisTemplate.expire(windowKey, windowMs, TimeUnit.MILLISECONDS);
        }
        
        return count != null && count <= limit;
    }
}
```

**Redis + Lua 스크립트 (원자성 보장)**

```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)

if current == 1 then
    redis.call('PEXPIRE', key, window)
end

if current > limit then
    return 0
end

return 1
```

```java
@Component
@RequiredArgsConstructor
public class RedisLuaRateLimiter {
    private final RedisTemplate<String, String> redisTemplate;
    
    private static final String SCRIPT = 
        "local key = KEYS[1] " +
        "local limit = tonumber(ARGV[1]) " +
        "local window = tonumber(ARGV[2]) " +
        "local current = redis.call('INCR', key) " +
        "if current == 1 then redis.call('PEXPIRE', key, window) end " +
        "if current > limit then return 0 end " +
        "return 1";
    
    public boolean isAllowed(String key, int limit, long windowMs) {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>(SCRIPT, Long.class);
        Long result = redisTemplate.execute(script, 
            Collections.singletonList(key), 
            String.valueOf(limit), 
            String.valueOf(windowMs));
        return result != null && result == 1;
    }
}
```

---

## 응답 처리

### HTTP 상태 코드

```
429 Too Many Requests
```

### 응답 헤더

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000
Retry-After: 60
```

| 헤더 | 설명 |
|------|------|
| X-RateLimit-Limit | 최대 허용 요청 수 |
| X-RateLimit-Remaining | 남은 요청 수 |
| X-RateLimit-Reset | 리셋 시간 (Unix timestamp) |
| Retry-After | 재시도까지 대기 시간 (초) |

### 응답 본문

```json
{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please retry after 60 seconds.",
  "retry_after": 60
}
```

---

## Rate Limit 키 설계

### 기준별 키

| 기준 | 키 예시 | 사용 사례 |
|------|---------|-----------|
| IP | `ip:192.168.1.1` | 익명 사용자 |
| 사용자 ID | `user:12345` | 로그인 사용자 |
| API 키 | `apikey:abc123` | API 클라이언트 |
| 엔드포인트 | `user:123:/api/search` | 특정 API별 제한 |

### 다단계 제한

```
전역: 10,000 req/분
사용자별: 100 req/분
엔드포인트별: 10 req/분 (/api/expensive)
```

---

## 실전 적용 (Spring)

### 인터셉터 방식

```java
@Component
@RequiredArgsConstructor
public class RateLimitInterceptor implements HandlerInterceptor {
    private final DistributedRateLimiter rateLimiter;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {
        String clientIp = request.getRemoteAddr();
        String key = "ip:" + clientIp;
        
        if (!rateLimiter.isAllowed(key, 100, 60000)) {
            response.setStatus(429);
            response.setHeader("Retry-After", "60");
            response.setContentType("application/json");
            response.getWriter().write(
                "{\"error\":\"rate_limit_exceeded\",\"retry_after\":60}");
            return false;
        }
        
        return true;
    }
}
```

### AOP 방식

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int limit() default 100;
    long windowMs() default 60000;
}

@Aspect
@Component
@RequiredArgsConstructor
public class RateLimitAspect {
    private final DistributedRateLimiter rateLimiter;
    
    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) 
            RequestContextHolder.currentRequestAttributes()).getRequest();
        
        String key = "user:" + getCurrentUserId();
        
        if (!rateLimiter.isAllowed(key, rateLimit.limit(), rateLimit.windowMs())) {
            throw new RateLimitExceededException("Too many requests");
        }
        
        return joinPoint.proceed();
    }
}

// 사용
@RestController
public class ApiController {
    
    @RateLimit(limit = 10, windowMs = 60000)
    @GetMapping("/api/search")
    public ResponseEntity<?> search() {
        // ...
    }
}
```

---

## 알고리즘 선택 가이드

| 상황 | 추천 알고리즘 |
|------|---------------|
| 간단한 구현 필요 | Fixed Window |
| 정확한 제한 필요 | Sliding Window Log |
| 메모리 효율 + 적당한 정확도 | Sliding Window Counter |
| 버스트 트래픽 허용 | Token Bucket |
| 일정한 처리 속도 보장 | Leaky Bucket |

---

## 핵심 요약

1. **목적**: API 남용 방지, DDoS 방어, 공정한 리소스 분배
2. **주요 알고리즘**: Fixed Window, Sliding Window, Token Bucket, Leaky Bucket
3. **분산 환경**: Redis 등 중앙 저장소 사용
4. **응답**: 429 상태 코드 + RateLimit 헤더
5. **키 설계**: IP, 사용자 ID, API 키 등 상황에 맞게
