**SSE(Server-Sent Events)**
- **단방향 통신**: 서버에서 클라이언트로만 데이터를 전송
- **HTTP 프로토콜 사용**: 일반 HTTP 연결을 유지
- **서버 푸시만 필요한 경우**: 실시간 뉴스 피드, 주식 시세, 알림 등에 적합
- **자동 재연결**: 연결이 끊어지면 자동으로 재연결을 시도


**Spring MVC에서 SSE:**
```java
@GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter();
    
    // 별도 스레드에서 이벤트 전송
    executor.execute(() -> {
        try {
            for (int i = 0; i < 10; i++) {
                emitter.send("Event " + i);
                Thread.sleep(1000);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });
    
    return emitter;
}
```

**WebFlux에서 SSE:**
```java
@GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamEvents() {
    return Flux.interval(Duration.ofSeconds(1))
               .map(i -> "Event " + i)
               .take(10);
}
```

**리소스 효율성**
- 여러 클라이언트가 동시에 구독해도 적은 스레드로 처리 가능
- MVC는 각 연결마다 스레드 풀 리소스를 소비

**백프레셔(Backpressure) 지원**
- 클라이언트가 처리 속도를 따라가지 못할 때 자동으로 조절

**주기적 스트리밍에 최적화**
- `Flux.interval`은 이런 주기적 전송을 위해 설계됨
- 에러 처리, 취소, 완료 처리가 리액티브하게 통합됨


**주기적이고 지속적인 서버 푸시 시나리오**에서 WebFlux의 장점

**1. 많은 동시 연결 처리**
- 100명이 동시에 30초마다 프로세스 정보를 받는다면?
    - **MVC**: 100개의 스레드 + 100개의 스케줄러 필요 → 리소스 많이 소모
    - **WebFlux**: 소수의 이벤트 루프 스레드로 처리 → 훨씬 효율적


**2. 메모리 효율성**
```java
// 장시간 연결이 유지되는 SSE에서
// WebFlux는 스레드를 블로킹하지 않아 메모리 사용량이 적음
```

**3. 타이머/스케줄링 관리 용이**
- `Flux.interval`이 자동으로 관리
- MVC는 `ScheduledExecutorService` 생성/종료를 직접 관리해야 함

**4. 연결 해제 처리가 자연스러움**
```java
// 클라이언트가 연결을 끊으면 Flux가 자동으로 취소됨
// MVC는 emitter 상태 확인과 executor 종료를 수동으로 처리
```

**5. 확장성**
- 모니터링 대시보드처럼 많은 사용자가 실시간으로 데이터를 받아야 하는 경우
- WebFlux는 서버 리소스를 훨씬 덜 사용하면서 더 많은 클라이언트를 처리 가능

**WebFlux가 더 나은 경우:**
- ✅ 많은 동시 SSE 연결 (수십~수백 명 이상)
- ✅ 장시간 연결 유지
- ✅ 주기적/지속적 데이터 스트리밍
- ✅ 이미 리액티브 스택 사용 중 (R2DBC, Reactive MongoDB 등)

**Spring MVC가 더 나은 경우:**
- ✅ 기존 MVC 프로젝트 (JDBC, JPA 사용 중)
- ✅ SSE 연결이 소수 (10~20개 이하)
- ✅ 일회성 또는 단기간 이벤트 전송
- ✅ 팀이 리액티브 프로그래밍에 익숙하지 않음