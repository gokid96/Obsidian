
API Gateway는 **모든 클라이언트 요청의 단일 진입점

---

## 왜 필요한가

### API Gateway 없이

```
Client → order-service (8081)
Client → payment-service (8082)
Client → delivery-service (8083)
```

클라이언트가 각 서비스 주소를 다 알아야 함. 서비스 추가될 때마다 클라이언트 수정.

### API Gateway 있으면

```
Client → API Gateway (80) → order-service
                         → payment-service
                         → delivery-service
```

클라이언트는 Gateway 주소만 알면 됨.

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
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐             │
│  │  인증    │ │ 라우팅   │ │ 로드    │ │ 로깅/   │              │
│  │         │ │         │ │ 밸런싱   │ │ 모니터링 │             │  
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘             │
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

| 도구                   | 설명                |
| -------------------- | ----------------- |
| Spring Cloud Gateway | Spring 기반, Java   |
| Kong                 | 오픈소스, 플러그인 풍부     |
| AWS API Gateway      | AWS 관리형           |
| Nginx                | 가볍고 빠름            |
| Caddy                | 리버스 프록시, HTTPS 자동 |
## 실무 조합

```
Client → Nginx/Caddy (앞단) → API Gateway (뒷단) → Services
              │                      │
         SSL 처리              인증, 라우팅
         로드밸런싱            비즈니스 로직
```

소규모면 **Nginx만**으로도 충분하고, MSA 규모가 커지면 **뒤에 API Gateway** 붙이는 식

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