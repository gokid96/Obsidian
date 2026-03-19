## 전체 구조

```
[사용자]
    ↓ HTTPS
[Cloudflare DNS]
    ↓
[AWS ALB] (ap-northeast-2a / ap-northeast-2c)
    ↓ Round Robin
┌───────────────┬───────────────┐
│   EC2-1       │   EC2-2       │
│ 3.37.79.4     │ 43.200.44.47  │
│               │               │
│ Docker        │ Docker        │
│ untitles-api  │ untitles-api  │
│ :8070         │ :8070         │
│               │               │
│ Caffeine      │ Caffeine      │
│ Cache (로컬)  │ Cache (로컬)  │
└───────┬───────┴───────┬───────┘
        └───────┬───────┘
                ↓
           [AWS RDS]
           MySQL 8.0
           (ap-northeast-2a)
```

---

## 인프라

|구성 요소|상세|
|---|---|
|**ALB**|Application Load Balancer, Round Robin, AZ-A/C|
|**EC2-1**|탄력적 IP `3.37.79.4`, 인스턴스 `i-010bcef68fdb077c0`|
|**EC2-2**|탄력적 IP `43.200.44.47`, 인스턴스 `i-06380465df597885a`|
|**RDS**|MySQL 8.0, `untitles-db.c7si8i2ao8v3.ap-northeast-2.rds.amazonaws.com`|
|**Cloudflare R2**|이미지 스토리지, `images.untitles.net`|
|**AWS SES**|이메일 발송 (이메일 인증)|

---

## 애플리케이션 스택

|구성 요소|상세|
|---|---|
|**Language**|Java 21|
|**Framework**|Spring Boot 3.5.7|
|**Build**|Gradle 8.5|
|**Container**|Docker (eclipse-temurin:21-jre)|

---

## 인증

```
일반 로그인
[클라이언트] → POST /auth/login → JWT 발급 (AccessToken 30분 / RefreshToken 7일)
               → localStorage에 토큰 저장

OAuth2 로그인
[클라이언트] → Google/Kakao/Naver → OAuth2 콜백
             → JWT 발급 → /oauth/callback?accessToken=xxx&refreshToken=xxx
             → localStorage에 토큰 저장

인증 방식: Stateless JWT (서버에 세션 없음)
→ ALB Round Robin 환경에서도 어느 서버로 가든 독립적으로 검증 가능
```

---

## 캐시 (현재 문제)

```
EC2-1 Caffeine Cache          EC2-2 Caffeine Cache
┌─────────────────────┐       ┌─────────────────────┐
│ workspaceTree:2014  │       │ workspaceTree:2014  │
│ → [folder1, 2, 3]  │       │ → [folder1, 2]      │ ← 불일치!
└─────────────────────┘       └─────────────────────┘

문제: EC2-1에서 폴더 생성 시 EC2-1 캐시만 삭제됨
      EC2-2는 이전 캐시를 그대로 유지 → 캐시 불일치 발생
해결: Redis 분산 캐시 도입 필요
```

---

## 배포 파이프라인

```
[GitHub main 브랜치 push]
    ↓
[GitHub Actions]
    ↓ Build & Push
[GHCR (GitHub Container Registry)]
    ↓ SSH 배포
┌───────────────┬───────────────┐
│   EC2-1       │   EC2-2       │
│ docker pull   │ docker pull   │
│ docker run    │ docker run    │
└───────────────┴───────────────┘
```

---

## 모니터링

### CloudWatch

|메트릭|설명|
|---|---|
|EC2-1 CPUUtilization|EC2-1 CPU 사용률|
|EC2-2 CPUUtilization|EC2-2 CPU 사용률|
|RDS CPUUtilization|DB CPU 사용률|
|RDS DatabaseConnections|DB 커넥션 수 (현재 약 32개)|
|RDS FreeStorageSpace|DB 디스크 여유 공간 (현재 약 19.22G)|
|ALB HealthyHostCount|정상 서버 수 (정상: 2)|
|ALB UnHealthyHostCount|비정상 서버 수 (정상: 0)|

### Grafana + Prometheus

- WSL Ubuntu에서 Docker Compose로 실행
- EC2-1, EC2-2 각각 직접 스크랩 (`http://IP:8070/actuator/prometheus`)
- Spring Boot JVM, HTTP 응답시간, DB 커넥션 등 상세 메트릭

---

## 프론트엔드

|구성 요소|상세|
|---|---|
|**Framework**|Vue 3|
|**Build Tool**|Vite|
|**배포**|Vercel (`untitles.net`, `www.untitles.net`)|
|**API 연동**|`api.untitles.net` (ALB DNS)|

---

## 현재 이슈 및 개선 계획

| 구분          | 내용                                    | 상태        |
| ----------- | ------------------------------------- | --------- |
| 캐시 불일치      | Caffeine 로컬 캐시로 서버 간 캐시 불일치 발생        | ❌ 미해결     |
| Redis 도입    | 분산 캐시로 캐시 불일치 해결                      | 🔜 예정     |
| RDS 퍼블릭 액세스 | RDS에 불필요한 탄력적 IP 연결                   | 🔜 정리 예정  |
| prod SQL 로그 | application-prod.yml에 DEBUG 레벨 SQL 로그 | 🔜 정리 예정  |
| RateLimit   | RateLimitFilter @Component 주석 처리 상태   | 🔜 활성화 예정 |


---
# untitles-api MSA + 이벤트 드리븐 전환 가이드

---

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


