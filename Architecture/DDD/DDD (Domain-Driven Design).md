
DDD는 **비즈니스 도메인 중심으로 설계하는 방법론**

---

## 기존 설계 vs DDD

### 기존: 기술 중심

```
"Controller 만들고, Service 만들고, Repository 만들고"

src/
├── controller/
├── service/
├── repository/
└── entity/
```

### DDD: 도메인 중심

```
"주문이라는 개념을 먼저 정의하고, 그걸 다루는 방법을 만들고"

src/
├── order/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
└── payment/
    ├── domain/
    ├── application/
    └── infrastructure/
```

---

## DDD 폴더 구조

```
domain/
└── user/
    ├── domain/           ← 핵심 비즈니스
    │   ├── User.java            (엔티티)
    │   ├── UserId.java          (값 객체)
    │   ├── UserRepository.java  (인터페이스만 = Port)
    │   └── UserDomainService.java
    │
    ├── application/      ← 유스케이스
    │   ├── CreateUserUseCase.java
    │   ├── GetUserUseCase.java
    │   └── UserDto.java
    │
    ├── infrastructure/   ← 구현체 (= Adapter)
    │   ├── UserRepositoryImpl.java
    │   └── UserJpaEntity.java
    │
    └── presentation/     ← API
        └── UserController.java
```

---

## 기존 구조 vs DDD 구조

|            | 기존                                | DDD                                |
| ---------- | --------------------------------- | ---------------------------------- |
| Entity 위치  | entity/                           | domain/                            |
| Repository | JpaRepository 바로 사용               | domain에 인터페이스, infrastructure에 구현체 |
| Service    | 하나                                | DomainService + UseCase 분리         |
| 의존성        | controller → service → repository | 모두 → domain                        |

---

## 의존성 방향

### 기존

```
Controller → Service → Repository → DB
(한 방향으로 흘러감)
```

### DDD

```
Presentation ─┐
Application  ─┼─→ Domain ←── Infrastructure
              │
(모두 Domain을 향함, Domain은 아무것도 의존 안 함)
```

---

## 포트-어댑터 (헥사고날) 아키텍처

DDD에서 자주 쓰는 구조.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Controller ─────→ Port ←───── JPA Adapter                │
│   (Input Adapter)  (인터페이스)   (Output Adapter)           │
│                       │                                     │
│                       ▼                                     │
│                    Domain                                   │
│                 (비즈니스 로직)                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

|용어|설명|위치|
|---|---|---|
|Port|인터페이스|domain|
|Adapter|구현체|infrastructure|

---

## 왜 이렇게 복잡하게?

### 기존 구조에서 DB 변경하면

```java
// Service가 JPA에 의존
@Service
public class UserService {
    private final UserRepository userRepository;  // JpaRepository
}
```

MongoDB로 바꾸려면? Service 코드 전부 수정.

### DDD 구조에서 DB 변경하면

```java
// Service는 Port(인터페이스)만 알고 있음
@Service
public class UserService {
    private final UserRepository userRepository;  // 인터페이스
}

// Adapter만 교체
public class UserMongoAdapter implements UserRepository { }
```

domain 코드는 **수정 없음**.

---

## 언제 DDD를 쓰나

|상황|추천 구조|
|---|---|
|단순 CRUD|기존 구조|
|비즈니스 로직 복잡|DDD|
|소규모 팀, 단기 프로젝트|기존 구조|
|대규모 팀, 장기 운영|DDD|
|외부 기술 변경 가능성 높음|DDD|

---

## 모놀리식 + DDD

MSA 아니어도 DDD 쓸 수 있음.

```
하나의 애플리케이션 안에서 도메인별로 코드 분리
→ 나중에 MSA로 전환하기 쉬움
```

---

## 관련 개념

- [[Microservices Architecture (MSA)]]
- [[Event-Driven Architecture]]
- [[Saga Pattern]]

---

## Tags

#DDD #Architecture #헥사고날 #포트어댑터