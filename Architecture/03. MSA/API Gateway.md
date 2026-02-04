
API Gateway는 **모든 클라이언트 요청의 단일 진입점**이다.

---

## 왜 필요한가

### API Gateway 없이

```
Client → order-service (8081)
Client → payment-service (8082)
Client → delivery-service (8083)
```

- 클라이언트가 각 서비스 주소를 다 알아야 함
- 서비스 추가될 때마다 클라이언트 수정
- 각 서비스마다 JWT 검증 로직 중복

```java
// order-service
@Component
public class JwtFilter { /* JWT 검증 */ }

// payment-service
@Component
public class JwtFilter { /* JWT 검증 (중복) */ }

// delivery-service
@Component
public class JwtFilter { /* JWT 검증 (중복) */ }
```

JWT 로직 수정하면? 모든 서비스 다 수정하고 전부 배포해야 함.

### API Gateway 있으면

```
Client → API Gateway (80) → order-service
                         → payment-service
                         → delivery-service
```

- 클라이언트는 Gateway 주소만 알면 됨
- JWT 검증은 Gateway에서 한 번만
- 각 서비스는 비즈니스 로직에만 집중

```java
// Gateway에서만 JWT 검증
@Component
public class AuthFilter implements GlobalFilter {
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // JWT 검증
        // 유저 정보를 헤더에 담아서 서비스로 전달
        exchange.getRequest().mutate()
            .header("X-User-Id", userId)
            .build();
        return chain.filter(exchange);
    }
}
```

각 서비스는 헤더에서 유저 정보만 꺼내 씀. 인증 로직 몰라도 됨.

---

## 구조

```
┌──────────────────────────────────────────────────────────────┐
│                        Client                                │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                     API Gateway                              │
│                                                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐            │
│  │  인증   │ │ 라우팅  │ │ 로드    │ │ 로깅/   │            │
│  │        │ │         │ │ 밸런싱  │ │ 모니터링│            │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘            │
└─────────────────────────┬────────────────────────────────────┘
                          │
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   order     │    │   payment   │    │  delivery   │
│   service   │    │   service   │    │   service   │
└─────────────┘    └─────────────┘    └─────────────┘
```

---

## 주요 기능

|기능|설명|
|---|---|
|라우팅|`/orders` → order-service, `/payments` → payment-service|
|인증/인가|JWT 검증, 권한 체크|
|로드밸런싱|여러 인스턴스에 트래픽 분산|
|Rate Limiting|요청 수 제한 (초당 100개 등)|
|로깅/모니터링|요청 추적, 메트릭 수집|
|캐싱|자주 요청되는 응답 캐싱|

---

## 라우팅 예시 (Spring Cloud Gateway)

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: http://order-service:8081
          predicates:
            - Path=/orders/**
            
        - id: payment-service
          uri: http://payment-service:8082
          predicates:
            - Path=/payments/**
```

```
GET /orders/123     → order-service:8081/orders/123
POST /payments      → payment-service:8082/payments
```

---

## 주요 도구

|도구|설명|
|---|---|
|Spring Cloud Gateway|Spring 기반, Java|
|Kong|오픈소스, 플러그인 풍부|
|AWS API Gateway|AWS 관리형|
|Nginx|가볍고 빠름|
|Caddy|리버스 프록시, HTTPS 자동|

---

## 실무 조합

### 소규모: Nginx만

```
Client → Nginx → Services
           │
      라우팅, SSL, 로드밸런싱
```

단순 라우팅만 필요하면 Nginx로 충분.

```nginx
location /orders {
    proxy_pass http://order-service:8081;
}
location /payments {
    proxy_pass http://payment-service:8082;
}
```

### 대규모: Nginx + API Gateway

```
Client → Nginx (앞단) → API Gateway (뒷단) → Services
            │                   │
       SSL 처리            JWT 검증
       로드밸런싱           라우팅 로직
       정적 파일            필터, 변환
```

**왜 나누는가?**

|역할|Nginx|API Gateway|
|---|---|---|
|SSL 인증서|✅ 처리 잘함|굳이 여기서 안 해도 됨|
|정적 파일|✅ 빠름|❌ 못함|
|로드밸런싱|✅ 가벼움|가능하지만 무거움|
|JWT 검증|❌ 복잡함|✅ Java로 쉽게|
|복잡한 필터|❌ 설정 한계|✅ 코드로 자유롭게|

각자 잘하는 걸 맡기는 것.

### 클라우드: AWS API Gateway

```
Client → AWS API Gateway → Lambda / ECS / EC2
                │
          관리형 (서버 없음)
          자동 스케일링
          AWS 연동 쉬움
```

---

## 장단점

|장점|단점|
|---|---|
|클라이언트 단순화|단일 장애점(SPOF) 가능|
|공통 기능 집중 관리|병목 지점이 될 수 있음|
|보안 강화|추가 네트워크 홉|

---

## SPOF 해결

```
┌─────────┐
│  Client │
└────┬────┘
     │
     ▼
┌─────────────┐
│Load Balancer│
└──────┬──────┘
       │
  ┌────┴────┐
  ▼         ▼
┌─────────┐ ┌─────────┐
│ Gateway │ │ Gateway │   ← 이중화
│   #1    │ │   #2    │
└─────────┘ └─────────┘
```

---

## 관련 개념

- [[Microservices Architecture (MSA)]]
- [[Service Discovery]]
- [[Circuit Breaker]]

---

## Tags

#APIGateway #MSA #라우팅 #인증