
## 현재 상태 분석

현재 `untitles-api`는 **모놀리식 아키텍처**로, 하나의 Spring Boot 프로젝트 안에 모든 도메인이 존재한다.

### 현재 도메인 간 의존성 문제

서비스들이 다른 도메인의 Repository를 직접 참조하고 있어 MSA로 분리가 불가능한 상태다.

```
AuthService
  └── EmailService 직접 호출
  └── WorkspaceRepository 직접 접근
  └── WorkspaceMemberRepository 직접 접근

WorkspaceService
  └── UserRepository 직접 접근
  └── PostRepository 직접 접근
  └── FolderRepository 직접 접근

PostService
  └── FolderRepository 직접 접근
  └── WorkspaceMemberRepository 직접 접근
  └── WorkspaceRepository 직접 접근

PublishService
  └── FolderRepository 직접 접근
  └── PostRepository 직접 접근
  └── WorkspaceMemberRepository 직접 접근
  └── UserRepository 직접 접근
```

---

## 목표 아키텍처

```
[클라이언트]
     ↓
[API Gateway]  ← JWT 검증 중앙화
     ↓
┌────────────────────────────────────────┐
│  auth-service        :8081             │
│  content-service     :8082             │
│  workspace-service   :8083             │
│  notification-service :8084            │
│  publish-service     :8085             │
└────────────────────────────────────────┘
     ↓ 이벤트
[Kafka]
     ↓
각 서비스 Consumer
```

---

## STEP 1. 서비스 분리 설계

### 서비스 경계 정의

|서비스|포함 도메인|포트|
|---|---|---|
|`auth-service`|auth, user, jwt, oauth, security|8081|
|`content-service`|post, folder, image|8082|
|`workspace-service`|workspace|8083|
|`notification-service`|email|8084|
|`publish-service`|publish|8085|
|`api-gateway`|라우팅, JWT 검증|8080|

### 프로젝트 구조

```
untitles/
├── auth-service/
│   ├── build.gradle
│   └── src/main/java/com/untitles/auth/
├── content-service/
│   ├── build.gradle
│   └── src/main/java/com/untitles/content/
├── workspace-service/
│   ├── build.gradle
│   └── src/main/java/com/untitles/workspace/
├── notification-service/
│   ├── build.gradle
│   └── src/main/java/com/untitles/notification/
├── publish-service/
│   ├── build.gradle
│   └── src/main/java/com/untitles/publish/
├── api-gateway/
│   ├── build.gradle
│   └── src/main/java/com/untitles/gateway/
└── docker-compose.yml
```

### DB 분리

각 서비스는 자신의 DB만 소유한다. 다른 서비스의 DB에 직접 접근하지 않는다.

```
auth-service        → MySQL: untitles_auth
content-service     → MySQL: untitles_content
workspace-service   → MySQL: untitles_workspace
notification-service → MySQL: untitles_notification
publish-service     → MySQL: untitles_publish
```

#### 트레이드오프: DB 분리

|장점|단점|
|---|---|
|서비스별 독립 배포 가능|JOIN 쿼리 불가|
|장애 격리|데이터 정합성 관리 복잡|
|기술 스택 자유 (MySQL, Redis 혼용 등)|분산 트랜잭션 필요 (Saga 패턴)|

---

## STEP 2. 서비스 간 직접 참조 제거

### 기존 코드의 문제

```java
// WorkspaceService.java - 다른 도메인 Repository 직접 접근
private final UserRepository userRepository;      // auth 도메인
private final PostRepository postRepository;      // content 도메인
private final FolderRepository folderRepository;  // content 도메인
```

### 해결 방법: Feign Client (REST)

서비스가 분리되면 직접 접근 대신 HTTP 호출로 교체한다.

```java
// workspace-service 안에서 auth-service 호출
@FeignClient(name = "auth-service")
public interface AuthClient {
    @GetMapping("/internal/users/{userId}")
    UserResponse getUser(@PathVariable Long userId);
}

// WorkspaceService.java 변경
private final AuthClient authClient;  // UserRepository 대신

public WorkspaceResponse createWorkspace(Long userId, WorkspaceCreateRequest request) {
    UserResponse user = authClient.getUser(userId);  // HTTP 호출
    // ...
}
```

### 어떤 호출을 REST로 유지할까?

**응답 결과가 즉시 필요한 것 → REST (동기)**

|호출|이유|
|---|---|
|워크스페이스 생성 시 유저 존재 확인|없는 유저면 생성 불가, 응답 필요|
|게시글 작성 시 워크스페이스 멤버 권한 확인|권한 없으면 차단해야 함|
|회원가입 시 이메일 인증 여부 확인|미인증이면 회원가입 불가|
|API Gateway JWT 검증|인증 실패시 요청 차단|

#### 트레이드오프: REST (동기 호출)

|장점|단점|
|---|---|
|구현 단순|호출한 서비스가 죽으면 같이 실패|
|즉각적인 응답|네트워크 지연 추가|
|에러 처리 명확|서비스 간 결합도 존재|

---

## STEP 3. 이벤트 드리븐 적용 (Kafka)

### 어떤 호출을 이벤트로 바꿀까?

**메인 플로우 결과에 영향 없는 부가 작업 → Kafka (비동기)**

|이벤트|Producer|Consumer|이유|
|---|---|---|---|
|`UserRegisteredEvent`|auth-service|notification-service|메일 삭제 실패해도 회원가입은 성공해야 함|
|`MemberInvitedEvent`|workspace-service|notification-service|메일 발송 실패해도 초대 자체는 성공해야 함|
|`WorkspaceDeletedEvent`|workspace-service|content-service|워크스페이스 삭제 후 게시글/폴더 정리는 비동기 가능|
|`PostPublishedEvent`|content-service|publish-service|캐시 갱신은 비동기 가능|

### 이벤트 흐름 예시: 회원가입

```
[auth-service]
  회원가입 완료
    → UserRegisteredEvent 발행 (Kafka)
        → [notification-service] 이메일 인증 데이터 삭제
```

기존 코드:

```java
// AuthService.java - 동기 직접 호출
emailService.deleteVerification(request.getEmail());
```

변경 후:

```java
// AuthService.java - 이벤트 발행
kafkaProducer.send("user-registered",
    new UserRegisteredEvent(savedUser.getUserId(), request.getEmail()));

// notification-service의 Consumer
@KafkaListener(topics = "user-registered")
public void handleUserRegistered(UserRegisteredEvent event) {
    emailService.deleteVerification(event.getEmail());
}
```

### 이벤트 흐름 예시: 워크스페이스 삭제

기존 코드:

```java
// WorkspaceService.java - 직접 다른 도메인 Repository 접근
postRepository.deleteByWorkspaceWorkspaceId(workspaceId);
folderRepository.clearParentByWorkspaceId(workspaceId);
folderRepository.deleteAllByWorkspaceId(workspaceId);
```

변경 후:

```java
// workspace-service - 이벤트 발행
kafkaProducer.send("workspace-deleted",
    new WorkspaceDeletedEvent(workspaceId));

// content-service의 Consumer
@KafkaListener(topics = "workspace-deleted")
public void handleWorkspaceDeleted(WorkspaceDeletedEvent event) {
    postRepository.deleteByWorkspaceId(event.getWorkspaceId());
    folderRepository.deleteAllByWorkspaceId(event.getWorkspaceId());
}
```

#### 트레이드오프: Kafka (비동기 이벤트)

|장점|단점|
|---|---|
|서비스 간 결합 완전 제거|즉각적인 응답 불가|
|Consumer 죽어도 Producer는 정상 동작|이벤트 순서 보장 어려움|
|이벤트 재처리 가능 (offset)|중복 처리 방어 로직 필요 (idempotent)|
|장애 전파 차단|디버깅/추적 복잡 (분산 추적 필요)|

---

## STEP 4. API Gateway 설정

### 역할

- 모든 외부 요청의 진입점
- JWT 검증을 Gateway에서 중앙화 (각 서비스에서 제거)
- 각 서비스로 라우팅

### 라우팅 설계

```yaml
# api-gateway/src/main/resources/application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: auth-service
          uri: http://auth-service:8081
          predicates:
            - Path=/api/v1/auth/**, /api/v1/email/**, /oauth2/**

        - id: workspace-service
          uri: http://workspace-service:8083
          predicates:
            - Path=/api/v1/workspaces/**

        - id: content-service
          uri: http://content-service:8082
          predicates:
            - Path=/api/v1/workspaces/*/posts/**, /api/v1/workspaces/*/folders/**

        - id: publish-service
          uri: http://publish-service:8085
          predicates:
            - Path=/api/v1/public/**, /api/v1/workspaces/*/publish/**
```

### JWT 검증 위치 변경

```
기존: 각 서비스의 JwtFilter에서 검증
변경: API Gateway의 GlobalFilter에서 검증 후 userId를 헤더로 전달

X-User-Id: 123  ← Gateway가 각 서비스에 전달
```

#### 트레이드오프: Gateway JWT 중앙화

|장점|단점|
|---|---|
|각 서비스에서 인증 로직 제거|Gateway가 SPOF(단일 장애점)가 될 수 있음|
|보안 정책 한 곳에서 관리|Gateway 장애시 전체 서비스 접근 불가|
|서비스 코드 단순화|Gateway 이중화 필요|

---

## STEP 5. Docker Compose로 전체 구성

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Kafka 인프라
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    ports:
      - "9092:9092"

  # DB
  mysql-auth:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: untitles_auth

  mysql-content:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: untitles_content

  mysql-workspace:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: untitles_workspace

  # 서비스
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on: [auth-service, workspace-service, content-service]

  auth-service:
    build: ./auth-service
    ports:
      - "8081:8081"
    depends_on: [mysql-auth]

  workspace-service:
    build: ./workspace-service
    ports:
      - "8083:8083"
    depends_on: [mysql-workspace, kafka]

  content-service:
    build: ./content-service
    ports:
      - "8082:8082"
    depends_on: [mysql-content, kafka]

  notification-service:
    build: ./notification-service
    ports:
      - "8084:8084"
    depends_on: [kafka]
```

---

## 기술적 트레이드오프 종합

### 1. 분산 트랜잭션 문제

모놀리스에서는 `@Transactional` 하나로 해결되던 것이 MSA에서는 불가능하다.

```
기존 (모놀리스):
@Transactional
public void signup() {
    userRepository.save(user);          // 실패시 전체 롤백
    workspaceRepository.save(workspace); // 실패시 전체 롤백
    emailService.deleteVerification();   // 실패시 전체 롤백
}

MSA에서는:
auth-service에서 user 저장 성공 후
workspace-service 호출 실패하면?
→ user는 저장됐는데 workspace는 없는 상태
```

**해결책: Saga 패턴**

- 각 서비스가 성공/실패 이벤트를 발행
- 실패시 보상 트랜잭션(Compensating Transaction) 실행
- 구현 복잡도가 높음

### 2. 데이터 정합성 문제

```
workspace-service의 WorkspaceMember 테이블에 userId가 있는데
auth-service의 User가 삭제되면?
→ 참조 무결성을 DB FK로 보장 불가
→ 이벤트로 처리해야 함
```

### 3. 네트워크 지연

```
모놀리스: 메서드 호출 = 나노초
MSA REST: HTTP 호출 = 수십ms ~ 수백ms
```

내부 서비스 간 호출이 많아질수록 응답시간 누적

### 4. 운영 복잡도

|항목|모놀리스|MSA|
|---|---|---|
|배포|1번|서비스 수만큼|
|로그 추적|하나의 로그 파일|분산 추적 필요 (Zipkin, Jaeger)|
|모니터링|단순|서비스별 모니터링 필요|
|로컬 실행|`./gradlew bootRun` 1번|Docker Compose 전체 실행|

### 5. Kafka 이벤트 중복 처리

```
Consumer가 이벤트를 처리하다 죽으면
재시작 후 같은 이벤트를 다시 처리할 수 있음
→ 멱등성(Idempotent) 보장 필요

예시: MemberInvitedEvent를 두 번 처리하면 초대 메일이 두 번 발송됨
해결: 이벤트 ID로 중복 처리 방어
```

### 6. 서비스 디스커버리

```
지금: auth-service가 workspace-service를
      "http://workspace-service:8083" 하드코딩으로 호출

문제: EKS에서 Pod IP는 계속 바뀜
     서비스가 늘어나면 하드코딩 관리 불가
```

**선택지:**

- Docker Compose 환경: 서비스 이름으로 자동 해결 (별도 작업 불필요)
- EKS 환경: Kubernetes Service가 자동으로 서비스 디스커버리 제공
- Spring Cloud Eureka: EKS 없이 자체 구축할 경우

|환경|방법|
|---|---|
|Docker Compose|서비스 이름으로 자동|
|EKS|Kubernetes Service 자동|
|온프레미스|Eureka 직접 구축|

---

### 7. 서킷 브레이커

```
PostService → workspace-service로 권한 확인 HTTP 호출
workspace-service가 응답 없으면?
→ PostService 스레드가 계속 대기
→ 스레드 풀 고갈
→ content-service 전체 다운
```

**해결책: Resilience4j**

```java
@CircuitBreaker(name = "workspace-service", fallbackMethod = "fallback")
public UserResponse getUser(Long userId) {
    return authClient.getUser(userId);
}

// workspace-service 죽었을 때 fallback
public UserResponse fallback(Long userId, Exception e) {
    throw new BusinessException(ErrorCode.SERVICE_UNAVAILABLE);
}
```

|고민할 것|내용|
|---|---|
|임계값|몇 번 실패하면 서킷 열지|
|복구 시간|얼마 후에 다시 시도할지|
|fallback 전략|차단됐을 때 기본값 반환할지, 예외 던질지|

---

### 8. 분산 추적 (Distributed Tracing)

```
현재 모놀리스: 로그 하나만 보면 전체 흐름 파악 가능

MSA에서 요청 실패 시:
api-gateway → auth-service → kafka → notification-service
어느 서비스에서 실패했는지 로그 하나로 못 찾음
```

**해결책: Zipkin + traceId**

```
요청마다 traceId 발급
api-gateway    [traceId: abc123] 요청 수신
auth-service   [traceId: abc123] 회원가입 처리
notification   [traceId: abc123] 이메일 발송
→ traceId로 전체 흐름 한눈에 추적 가능
```

이미 `LoggingAspect.java`가 있으니 여기에 traceId 추가하면 됨

---

### 9. 캐시 정합성 문제

```
현재 코드에 이미 Caffeine Cache (로컬 캐시) 가 있음
@CacheEvict(value = "workspaceTree", key = "#workspaceId")
@CacheEvict(value = "publicWorkspace", ...)
@CacheEvict(value = "publicPost", ...)

문제 1: EC2 2대에서 EC2-1 캐시 evict해도 EC2-2 캐시는 그대로
문제 2: MSA에서 content-service가 post 수정 →
        publish-service의 캐시는 언제 갱신?
```

**해결책: Caffeine → Redis 분산 캐시 전환**

```
content-service가 PostUpdatedEvent 발행
→ publish-service가 구독해서 자기 캐시 evict
```

|항목|Caffeine (현재)|Redis|
|---|---|---|
|속도|매우 빠름 (로컬 메모리)|약간 느림 (네트워크)|
|서비스 간 공유|불가|가능|
|캐시 정합성|서버별 불일치|중앙화로 일치|
|비용|무료|Redis 서버 추가|

---

### 10. 내부 서비스 간 인증

```
현재: 각 서비스가 JWT를 직접 검증
변경: Gateway에서 검증 후 X-User-Id 헤더로 전달

문제: 서비스 간 내부 호출은 Gateway를 안 거침
     workspace-service → content-service 직접 호출 시
     이건 누가 인증하나?
```

**선택지:**

|방법|설명|복잡도|
|---|---|---|
|Internal Token|내부 전용 토큰 별도 발급|중간|
|IP 화이트리스트|내부 IP만 허용|단순|
|서비스 메시 (Istio)|인프라 레벨에서 mTLS 처리|높음|
|신뢰 기반|내부 네트워크면 그냥 허용|단순 (보안 약함)|

포폴 수준에서는 **X-Internal-Secret 헤더** 방식이 현실적

```java
// 내부 호출 시 헤더 추가
headers.add("X-Internal-Secret", "내부_공유_시크릿");

// 수신 서비스에서 검증
if (!request.getHeader("X-Internal-Secret").equals(internalSecret)) {
    throw new BusinessException(ErrorCode.ACCESS_DENIED);
}
```

---

### 11. API 버전 관리

```
content-service API를 변경했을 때
이미 배포된 api-gateway, publish-service는 구버전 API 호출 중

배포 순서를 잘못 잡으면 순간적으로 서비스 오류 발생
```

**고민할 것:**

- `/api/v1/`, `/api/v2/` 버전 분리 전략
- 하위 호환성 유지 기간
- 어느 서비스를 먼저 배포할지 (배포 순서 의존성)

---

### 12. Kafka 이벤트 스키마 관리

```
auth-service가 UserRegisteredEvent 발행
notification-service가 구독

나중에 UserRegisteredEvent에 필드 추가하면?
→ notification-service가 역직렬화 실패할 수 있음
```

**선택지:**

|방법|설명|
|---|---|
|Avro + Schema Registry|스키마 버전 관리, 호환성 검증 자동화|
|하위 호환 규칙|필드 추가만 허용, 삭제/변경 금지|
|이벤트 버전 필드 추가|`version: "v1"` 필드로 Consumer가 분기 처리|

포폴 수준에서는 **하위 호환 규칙 + 이벤트 버전 필드** 조합이 현실적

---

### 13. 무중단 배포 전략

```
workspace-service 새 버전 배포 중
기존 버전과 새 버전이 동시에 떠있는 순간이 생김
→ 같은 Kafka 이벤트를 두 버전이 동시에 처리할 수 있음
→ Consumer Group 관리 필요
```

|전략|설명|다운타임|
|---|---|---|
|Rolling Update|하나씩 교체|없음|
|Blue/Green|구버전 유지하다 한번에 전환|없음|
|Canary|일부 트래픽만 새 버전으로|없음|

EKS 쓰면 Rolling Update 기본 제공

---

## 추가 고민사항 우선순위

```
당장 구현 필수 (없으면 MSA 자체가 불안정)
├── 서킷 브레이커       ← 장애 전파 차단
├── 분산 추적           ← 디버깅 불가능해짐
└── Redis 분산 캐시     ← 이미 캐시 쓰고 있으니까

구현하면 좋은 것
├── 내부 서비스 간 인증  ← 보안
├── 이벤트 스키마 버전   ← Kafka 안정성
└── API 버전 관리        ← 서비스 간 배포 독립성

EKS 갈 때 자동 해결되는 것
├── 서비스 디스커버리    ← K8s Service가 처리
└── 무중단 배포          ← Rolling Update 기본 제공
```

---

## 추가 고민사항: 트랜잭셔널 아웃박스 패턴

### 문제: DB 커밋과 Kafka 발행의 원자성

이벤트 드리븐에서 가장 놓치기 쉬운 문제다.

```java
// AuthService signup() - 현재 모놀리스 코드
@Transactional
public LoginResponse signup(...) {
    Users savedUser = userRepository.save(user);          // 1. DB 저장 성공
    workspaceRepository.save(personalWorkspace);           // 2. DB 저장 성공
    emailService.deleteVerification(request.getEmail());   // 3. 동기 호출 성공
}
```

MSA + Kafka 전환 후:

```java
// AuthService signup() - MSA 전환 후
@Transactional
public LoginResponse signup(...) {
    Users savedUser = userRepository.save(user);          // 1. DB 저장 성공
    workspaceRepository.save(personalWorkspace);           // 2. DB 저장 성공 → 트랜잭션 커밋
    
    kafkaProducer.send("user-registered", event);         // 3. Kafka 발행 실패하면?
    // → DB는 이미 커밋됐는데 이벤트는 발행 안 된 상태
    // → notification-service는 이메일 인증 데이터를 영원히 못 지움
}
```

**발생 가능한 시나리오:**

```
1. DB 저장 성공 → Kafka 서버 일시 다운 → 이벤트 유실
2. DB 저장 성공 → 애플리케이션 강제 종료 → 이벤트 유실
3. DB 저장 성공 → 네트워크 순단 → 이벤트 유실
```

---

### 해결책: Transactional Outbox 패턴

이벤트를 Kafka에 직접 보내는 대신, **같은 DB 트랜잭션 안에 outbox 테이블에 저장**한다. 별도 스케줄러가 outbox를 읽어서 Kafka에 발행한다.

```
[auth-service]

@Transactional
signup() {
    userRepository.save(user)                    ─┐
    workspaceRepository.save(workspace)            ├─ 같은 트랜잭션
    outboxRepository.save(UserRegisteredEvent)    ─┘  → 원자적으로 커밋
}

[Outbox Scheduler] (별도 스케줄러)
    outbox 테이블에서 sent=false 조회
    → Kafka 발행
    → sent=true 업데이트
```

---

### 구현 예시

**Outbox 테이블:**

```sql
CREATE TABLE outbox_events (
    id          VARCHAR(36) PRIMARY KEY,
    event_type  VARCHAR(100) NOT NULL,   -- 'UserRegisteredEvent'
    payload     TEXT NOT NULL,           -- JSON 직렬화된 이벤트
    sent        BOOLEAN DEFAULT FALSE,
    created_at  DATETIME DEFAULT NOW()
);
```

**Outbox Entity:**

```java
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id
    private String id;           // UUID
    private String eventType;
    private String payload;      // JSON
    private boolean sent;
    private LocalDateTime createdAt;
}
```

**signup() 변경:**

```java
// AuthService.java
@Transactional
public LoginResponse signup(UserCreateRequestDTO request) {
    Users savedUser = userRepository.save(user);
    workspaceRepository.save(personalWorkspace);

    // Kafka 직접 발행 대신 outbox에 저장 (같은 트랜잭션)
    OutboxEvent outbox = OutboxEvent.builder()
        .id(UUID.randomUUID().toString())
        .eventType("UserRegisteredEvent")
        .payload(objectMapper.writeValueAsString(
            new UserRegisteredEvent(savedUser.getUserId(), request.getEmail())
        ))
        .sent(false)
        .build();
    outboxRepository.save(outbox);  // DB 트랜잭션 안에서 저장

    return LoginResponse.of(...);
}
```

**Outbox Scheduler:**

```java
@Component
@RequiredArgsConstructor
public class OutboxScheduler {

    private final OutboxRepository outboxRepository;
    private final KafkaProducer kafkaProducer;

    @Scheduled(fixedDelay = 1000) // 1초마다 실행
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pendingEvents = outboxRepository.findBySentFalse();

        for (OutboxEvent event : pendingEvents) {
            kafkaProducer.send(event.getEventType(), event.getPayload());
            event.markAsSent();
            outboxRepository.save(event);
        }
    }
}
```

---

### 트레이드오프

|장점|단점|
|---|---|
|DB 커밋과 이벤트 발행의 원자성 보장|outbox 테이블 관리 비용|
|Kafka 장애 시에도 이벤트 유실 없음|이벤트 발행에 최대 1초 지연 (스케줄러 주기)|
|서버 강제 종료 시에도 안전|sent=false 쌓이면 스케줄러 부하 가능성|
|재처리 가능 (sent=false로 되돌리면 됨)|Consumer 중복 처리 방어 여전히 필요|

---

### 이 프로젝트에서 아웃박스가 필요한 곳

|이벤트|이유|
|---|---|
|`UserRegisteredEvent`|회원가입 DB 커밋 후 이메일 인증 삭제 이벤트 유실 방지|
|`MemberInvitedEvent`|멤버 초대 DB 커밋 후 초대 메일 발송 이벤트 유실 방지|
|`WorkspaceDeletedEvent`|워크스페이스 삭제 후 content-service 정리 이벤트 유실 방지|

---

## 전환 순서 요약

```
STEP 1. 서비스 경계 정의 및 프로젝트 분리
         └── 각 서비스별 독립 Spring Boot 프로젝트 생성
         └── 서비스별 DB 분리

STEP 2. 다른 도메인 직접 참조 제거
         └── Repository 직접 접근 → Feign Client (REST)로 교체

STEP 3. Kafka 도입 및 이벤트 드리븐 적용
         └── 비동기 가능한 호출 → 이벤트 발행/구독으로 교체

STEP 4. API Gateway 구성
         └── JWT 검증 중앙화
         └── 서비스별 라우팅

STEP 5. Docker Compose로 전체 로컬 실행 검증
         └── 전체 서비스 + Kafka + DB 통합 테스트

STEP 6. (선택) EKS 배포
         └── Docker Compose → Kubernetes 매니페스트로 전환
```

---

## 면접 답변 포인트

> "처음에는 모놀리스로 빠르게 개발했고, 도메인 간 의존성을 파악한 뒤 서비스 경계를 정의해서 MSA로 전환했습니다. 서비스 간 통신은 즉각적인 응답이 필요한 경우 REST Feign Client를 사용했고, 이메일 발송이나 워크스페이스 삭제 후 데이터 정리처럼 메인 플로우에 영향을 주지 않아도 되는 작업은 Kafka 이벤트로 분리해서 장애 전파를 차단했습니다. 트레이드오프로는 분산 트랜잭션 처리가 가장 어려웠고, Saga 패턴을 적용해서 해결했습니다."