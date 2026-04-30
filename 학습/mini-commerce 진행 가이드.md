# 📘 mini-commerce 진행 가이드 (v3)

> v2 → v3 변경: 본인 프로젝트 실제 상태 검토 후 STEP03 현황을 정확히 반영. "결정 사항" → "이미 결정됨"으로 정리. 참고 레포: https://github.com/discphy/e-commerce

---

## 📌 이 문서의 목적

레퍼런스 작성자는 단순히 코드를 짠 게 아니라 **WIL/Study/보고서를 통해 학습 내용을 누적**시켰다. 이 문서는 그 학습 흐름까지 따라가서, **각 STEP에서 어떤 개념을 학습하고 어떤 결정을 내려야 하는지** 정리한다.

본인이 STEP을 진행하면서 동시에 작성해야 할 산출물:

1. **코드 + 테스트** (PR로 머지)
2. **WIL** (주차별 회고)
3. **Study** (개념 학습 정리)
4. **Report** (단계별 기술 보고서)

---

## 🎯 핵심 원칙

### 1️⃣ STEP을 건너뛰지 마라

레퍼런스도 처음엔 락 없이, 캐시 없이, 이벤트 없이 구현했다.

### 2️⃣ 문서가 코드보다 먼저다

`[DOCS] 보고서 작성` → `[REFACTOR] 코드 변경` 순서. AS-IS / 문제 / 해결방안 / TO-BE.

### 3️⃣ `[REFACTOR]`는 죄가 아니다

274 커밋 중 약 35%가 `[REFACTOR]`. 점진적 개선이 핵심.

### 4️⃣ 같은 코드를 여러 번 갈아엎는다

**인기상품 조회는 5번 갈아엎혔다**: DB 집계 → 배치 → Redis 캐시 → Sorted Set → 실시간 이벤트.

### 5️⃣ 학습 → 적용 → 회고 사이클

레퍼런스 패턴: WIL에 개념 정리 → 코드에 적용 → Report에 측정 결과 기록.

---

## 🏛️ 채택된 아키텍처 — 클린 레이어드 + DIP

레이어형 구조 + DIP(의존성 역전 원칙)를 결합한 구조. **도메인 레이어가 외부 어떤 레이어도 알지 않는다**는 점이 핵심.

```
┌─────────────────────────────────────────────────┐
│  interfaces (Presentation Layer)                │
│  - Controller, Request, Response                │
│  - HTTP 요청 받기, 입력 검증, 응답 변환          │
└──────────────┬──────────────────────────────────┘
               │ 의존
               ▼
┌─────────────────────────────────────────────────┐
│  application (Application Layer)                │
│  - Facade, Criteria, Result                     │
│  - 유즈케이스 조합, 트랜잭션 단위 관리          │
└──────────────┬──────────────────────────────────┘
               │ 의존
               ▼
┌─────────────────────────────────────────────────┐
│  domain (Domain Layer)         ⭐ 중심           │
│  - Entity, Service, Repository(인터페이스)       │
│  - Command, Info, Enum, VO                      │
│  - 의존 대상 없음                                │
└──────────────▲──────────────────────────────────┘
               │ 구현 (DIP)
               │
┌──────────────┴──────────────────────────────────┐
│  infrastructure (Infrastructure Layer)          │
│  - Repository 구현체, JPA, Redis, Kafka, 외부API│
└─────────────────────────────────────────────────┘
```

### 패키지/클래스 네이밍 규칙

|레이어|패키지명|클래스|요청 DTO|응답 DTO|
|---|---|---|---|---|
|Presentation|`interfaces`|`XxxController`|`XxxRequest`|`XxxResponse`|
|Application|`application`|`XxxFacade`|`XxxCriteria`|`XxxResult`|
|Domain|`domain`|`XxxService`|`XxxCommand`|`XxxInfo`|
|Infrastructure|`infrastructure`|`XxxRepositoryImpl`|-|-|

### ✅ 본인 프로젝트 현재 패키지 구조 (검토 결과)

```
com.github.gokid96.e_commerce
├── balance/
│   ├── interfaces/    ✅ Controller, Request, Response
│   ├── application/   ⚠️ 비어있음 — Service/Facade 작성 필요
│   ├── domain/        ✅ Balance, BalanceTransaction, Repository 인터페이스, TransactionType
│   └── infrastructure/✅ CoreRepository(impl), JpaRepository
├── coupon/            ⚠️ interfaces만 존재, 나머지 모두 비어있음
├── order/             ⚠️ interfaces만 존재, 나머지 모두 비어있음
├── product/           ⚠️ interfaces만 존재 (rank 하위 패키지만 비어있음 상태)
├── common/            ✅ ApiResponse, ApiControllerAdvice
└── ECommerceApplication.java
```

> **결론**: 4-Layer 구조는 **이미 도입 완료**. v2에서 적었던 "STEP03 진입 시 결정 사항" 항목은 사라짐. 이제는 비어있는 application/domain/infrastructure 패키지를 도메인별로 채워나가는 단계.

---

## 🧠 레퍼런스가 학습한 핵심 개념 정리

### WIL 2주차 — 설계의 중요성

**핵심 명언**:

> "설계가 명확하면, 코드를 치는 행위는 목표를 달성하는 수단이 된다. 설계가 명확하지 않으면, 코드를 치는 행위는 불필요한 노동이 된다." (허재 코치)

**구체화 과정** — 추상적 개념을 단계별로 분해:

```
추상: "백엔드 기능을 개발했다."
구체:
- 기능에 대한 요구사항을 파악한다.
- 요구사항을 확인하고 마일스톤을 작성한다.
- 시나리오 설계 문서를 작성한다 (UseCase, 시퀀스 다이어그램).
- ERD를 설계 및 작성한다.
- API 명세 및 Mock API를 구현한다.
- TDD로 단위 테스트 및 비즈니스 로직을 개발한다.
- 통합 테스트 코드를 작성한다.
- PR을 올리고 코드리뷰를 받으며 리팩토링한다.
```

**최소 스펙 vs 확장성** 갈등에서 **확장성을 선택**한 이유: 현업에서의 설계 역량을 키우는 게 우선.

### WIL 3주차 — 클린 아키텍처 적용 시 고민들

**고민 1: 도메인 간 협력 vs 강결합**

```java
// ❌ ID만 받는 방식 - 검증 불가, 테이블 지향적
class Order { private Order(Long couponId) { ... } }

// ✅ 도메인 객체를 받는 방식 - 협력 가능, 객체 지향적
class Order { private Order(Coupon coupon) { ... } }
```

**고민 2: 레이어 간 DTO 분리 = 오버 엔지니어링?**

3종(Request/Criteria/Command) 분리는 **각 레이어 책임 명확화 + 결합도 완충제**. API 스펙 변경이 도메인 레이어로 전파되지 않게 만든다.

**고민 3: 파사드 패턴 꼭 써야 할까?**

> "파사드 패턴은 울며 겨자먹기로 사용하는 경우가 많다."

여러 도메인 서비스를 조합해야 할 때만 사용. 단일 서비스만 쓰면 굳이 만들 필요 없음. 도메인 서비스 간 의존성 제거를 위한 **중간 조율자**.

> 잔액/쿠폰/상품처럼 **단일 도메인** 유즈케이스는 `BalanceService`만 두고 Facade 생략 가능. STEP04에서 주문/결제처럼 **여러 도메인 협력**이 필요해질 때 `OrderFacade` 도입.

**고민 4: 검증 로직은 어디에?**

> 도메인 객체 내부에 두는 것을 선호.

이유: Command에 두면 모든 DTO마다 중복 + 도메인 객체에 이중 검증 필요.

> 본인 코드에서 `Balance.charge()`가 이미 `MAX_BALANCE_AMOUNT` 검증을 캡슐화한 좋은 예시.

**고민 5: 도메인 클래스 vs JPA 엔티티 분리**

분리하지 않으면 발생하는 4가지 문제:

1. JPA에 의존적인 도메인 구조가 됨
2. 객체 간 협력이 ID 기반으로 제한됨
3. 도메인이 인프라에 의존하게 됨 (DIP 위반)
4. 도메인 관심사가 분리되지 않음

**현실적 절충**: 이번 과제에선 분리하지 않고 진행 (어설픈 분리는 오히려 독). 본인도 동일하게 `Balance` 엔티티에 `@Entity` + 비즈니스 메서드 같이 두는 방식.

### WIL 4주차 — 트랜잭션 + 인덱스

**격리 수준 4단계와 부정합 문제**:

|격리 수준|Dirty Read|Non-Repeatable Read|Phantom Read|
|---|---|---|---|
|READ UNCOMMITTED|❌ 발생|❌ 발생|❌ 발생|
|READ COMMITTED|✅ 차단|❌ 발생|❌ 발생|
|REPEATABLE READ (MySQL 기본)|✅ 차단|✅ 차단|⚠️ 발생 (InnoDB는 Next-Key Lock으로 차단)|
|SERIALIZABLE|✅ 차단|✅ 차단|✅ 차단|

**MVCC**: 데이터 버전 관리로 읽기 락 없이 일관된 읽기 보장. 쓰기와 읽기 충돌 방지.

**`@Transactional` 주의사항**:

- 자기 호출 시 트랜잭션 미적용 (프록시 우회)
- private 메서드 적용 안 됨
- readOnly: JPA에서 flush/Dirty Checking 생략으로 성능 최적화
- 이른 시점에 DB 커넥션 점유 (자원 낭비 가능)

**전파 속성**:

- `REQUIRED` (기본): 기존 트랜잭션 참여, 없으면 새로 시작
- `REQUIRES_NEW`: 기존 중단 + 새로 시작
- `NESTED`: 중첩 트랜잭션 (DB가 지원해야 함)

**인덱스 설계 원칙**:

- 한번에 찾을 수 있는 값 (중복 적은 컬럼)
- 데이터 삽입/수정 적은 컬럼
- 조회에 자주 사용되는 컬럼
- 너무 많지 않게 (한 테이블 5개 이하 권장, 8개 이상은 쓰기 성능 저하)

**복합 인덱스**: 카디널리티 높은 컬럼 순으로. 조건 순서가 일치해야 인덱스 작동.

**Covering Index**: 쿼리에서 필요한 모든 컬럼이 인덱스에 포함된 경우. 테이블 접근 없이 인덱스만으로 조회.

**B-Tree 구조**:

- 클러스터 인덱스(PK): leaf 노드에 실제 데이터 저장
- 넌클러스터 인덱스: leaf 노드에 PK 저장 → PK로 다시 클러스터 인덱스 탐색

**인덱스 유의사항**:

```sql
-- ✅ 올바른 사용
WHERE price > 10000 / 100

-- ❌ 컬럼에 연산 적용 시 인덱스 미사용
WHERE price * 100 > 10000

-- LIKE/BETWEEN/<,> 범위 조건 이후 컬럼은 인덱스 미적용
WHERE product_id = 1 AND ordered_at > '2024-01-01' AND status = 'PAID'
                                                     -- ↑ 인덱스 미적용

-- OR보다 UNION이 효율적
```

**Race Condition vs Lost Update**:

- Race Condition: 실행 순서에 따라 결과 달라짐 (재고 동시 차감 → 음수)
- Lost Update: 두 작업이 같은 데이터 수정 → 마지막 작업만 반영

### WIL 5주차 — 동시성 해결

**Java 동시성 도구**:

- `synchronized`: 단일 스레드 진입 제한, 무한 대기, 공정성 X
- `ReentrantLock`: 재진입 가능, 공정성 설정 가능, 세밀한 제어
- `Atomic`: CAS(Compare-And-Set) 기반, 락 없이 원자성 보장

**락 전략 결정 기준**:

> 동일 자원에는 일관된 락 전략. 반드시 성공해야 하면 비관적 락, 그렇지 않으면 낙관적 락.

**락 범위 결정 기준**:

> 가능한 최소 범위로. 넓을수록 성능 저하 + 데드락 위험 증가.

**분산 락 + 트랜잭션 순서 (매우 중요)**:

```
✅ 올바른 순서
1. 분산 락 획득
2. 트랜잭션 시작
3. 비즈니스 로직 실행
4. 트랜잭션 커밋
5. 분산 락 해제

❌ 문제 1: 트랜잭션 먼저 시작
- 조회 시점에 락 미적용 → Race Condition
- 락 획득 실패해도 DB 커넥션 점유

❌ 문제 2: 트랜잭션 커밋 전 락 해제
- 다른 트랜잭션이 미커밋 데이터 조회 → 정합성 깨짐
```

**구현 방법**: `@Order(Ordered.HIGHEST_PRECEDENCE)`로 락 AOP가 트랜잭션보다 먼저 실행되게 + `TransactionSynchronizationManager`로 커밋 후 락 해제.

**분산 락 종류**:

- **Simple Lock**: 실패 시 즉시 예외
- **Spin Lock**: 일정 시간/횟수 재시도 (Busy-wait)
- **Pub/Sub Lock**: Redis Pub/Sub으로 락 해제 이벤트 수신 (Redisson 기반)

### Study 01 — Kafka

**Kafka 핵심 구성**:

- Broker: Kafka 서버 인스턴스
- Producer: 메시지 발행
- Consumer: 메시지 소비, offset 관리
- Topic: 메시지 분류 단위 (논리적)
- Partition: Topic의 물리적 단위, 순서 보장
- Consumer Group: 병렬 처리 단위, 1 Partition = 1 Consumer
- Replication: Leader + Follower로 데이터 내구성 보장

**파티션 vs 컨슈머 비율**:

|구성|효과|사용처|
|---|---|---|
|파티션 > 컨슈머|1 Consumer가 다수 Partition 처리, 자연 throttling|DB 부하 분산|
|파티션 = 컨슈머|모든 Partition 병렬 처리 (이상적)|처리량 극대화|
|파티션 < 컨슈머|일부 idle, 장애 시 즉시 대체|고가용성|

**Kafka 주의사항 3가지**:

1. **메시지 유실 방지** (트랜잭셔널 메시징)
    
    - **Outbox 패턴**: 도메인 트랜잭션과 함께 Outbox 테이블에 이벤트 저장 → 별도 프로세스가 Kafka로 발행
    - **CDC**: DB 변경 사항을 Kafka로 자동 변환
2. **메시지 중복 처리** (멱등성)
    
    - 리밸런싱 시 메시지 중복 가능성
    - **Inbox 패턴**: 동일 메시지 들어와도 같은 결과 보장
3. **메시지 실패 대응** (DLQ)
    
    - **Dead Letter Queue**: 실패 메시지를 별도 토픽에 저장
    - Spring Kafka는 `-dlt` 접미사로 자동 생성

**Outbox 구현 흐름**:

```
1. Auto 이벤트 (트랜잭션 있음)
   - BEFORE_COMMIT: Outbox 테이블에 저장
   - AFTER_COMMIT (@Async): Kafka로 발행

2. Manual 이벤트 (트랜잭션 없음)
   - 즉시 Outbox 저장 + Kafka 발행

3. Outbox 정리
   - Kafka 소비 완료 시 eventId로 Outbox 레코드 삭제
```

### Study 02 — Cache

**캐시 사용 이유**:

- RDB 스케일업/아웃은 비용 ↑
- 외부 API는 직접 제어 불가
- 빠른 메모리에 자주 쓰는 데이터 저장

**캐시 문제**: 데이터 정합성. 원본 변경 시 캐시 갱신 안 되면 불일치.

**읽기 전략**:

|전략|동작|사용처|
|---|---|---|
|**Look Aside**|캐시 → 미스 시 RDB → 캐시 저장|상품 상세 (양 많음, 자주 안 봄)|
|**Read Through**|캐시만 조회 (없으면 장애)|인기 상품 (자주 봄, 캐시 웜업 필요)|

**쓰기 전략**:

|전략|동작|장단점|
|---|---|---|
|**Write Around**|RDB만 저장, 캐시 미스 시 갱신|정합성 약함|
|**Write Back**|캐시만 저장 → 배치로 RDB|쓰기 부하 ↓, 캐시 장애 시 유실|
|**Write Through**|RDB + 캐시 동시 저장|정합성 ↑, 리소스 ↓|

**캐시 웜업** (Cache Warm-up): 미리 캐시 적재 → 트래픽 급증 시 첫 사용자도 빠른 응답.

**캐시 스탬피드** (Cache Stampede): 캐시 비었을 때 RDB로 트래픽 몰림 → 장애. **Read Through + 웜업**으로 방지.

**캐시 무효화**:

- TTL: 시간 기반 자동 삭제 (정합성 덜 중요할 때)
- 명시적 무효화: RDB 변경 시 캐시 직접 갱신

**캐시 종류**:

- 로컬 캐시: 인스턴스 내부, 빠르나 인스턴스 간 불일치
- 글로벌 캐시 (Redis): 인스턴스 공유, 일관성 ↑, 네트워크 오버헤드

---

## 🗓️ STEP별 진행 가이드

### STEP01 - 설계 기본 ✅ 본인 완료

**산출물**: docs/architecture/ 5종 (요구사항/마일스톤/시퀀스/ERD/API)

### STEP02 - 설계 심화 ✅ 본인 완료

**산출물**: docs/architecture/03-2.StateDiagram.md, REST Docs 자동 문서화

---

### STEP03 - 도메인 구현 (잔액/쿠폰/상품) 🎯 **현재 진행 중 (Balance 부분 완료)**

#### 📊 현재 본인 프로젝트 진척도

**Balance 도메인** — 절반 정도 진행됨:

|항목|상태|비고|
|---|---|---|
|4-Layer 패키지 구조 (`interfaces/application/domain/infrastructure`)|✅|이미 적용됨|
|`Balance` 엔티티 + `charge/use` 비즈니스 메서드|✅|`MAX_BALANCE_AMOUNT` 검증 캡슐화 OK|
|`Balance.create()` 정적 팩토리|✅||
|`BalanceTransaction` 엔티티 + `TransactionType` enum|✅||
|`BalanceTransaction.ofCharge() / ofUse()` 정적 팩토리|✅||
|`BalanceRepository` 인터페이스(`domain/`)|✅||
|`BalanceCoreRepository` 구현체(`infrastructure/`)|✅|JPA 위임 OK|
|`BalanceTransactionRepository` 인터페이스 + 구현체|✅||
|**`Balance.refund()` 비즈니스 메서드**|❌|enum에 `REFUND`는 있는데 메서드 없음|
|**`BalanceTransaction.ofRefund()` 정적 팩토리**|❌||
|**`BalanceService` (application 레이어)**|❌|**application/ 패키지 비어있음**|
|**`BalanceCommand`, `BalanceInfo` 도메인 DTO**|❌||
|**`BalanceController`에서 mock 응답 제거 → Service 호출**|❌|여전히 `1000000L` 하드코딩|
|**`Balance` 단위 테스트** (`balance/domain/`)|❌|디렉터리 비어있음|
|**`BalanceService` 단위 테스트 (Mockito)**|❌||
|**`BalanceIntegrationTest` (Testcontainers)**|❌||
|**`ApiControllerAdvice`에 `IllegalArgumentException` 핸들러**|❌|현재 `BindException`만 처리|

**Coupon / Order / Product 도메인** — `interfaces/`만 작성됨 (Mock 컨트롤러 수준):

|항목|상태|
|---|---|
|Coupon: domain/application/infrastructure 구현|❌ 모두 비어있음|
|Order: domain/application/infrastructure 구현|❌ 모두 비어있음|
|Product: domain/application/infrastructure 구현|❌ rank 하위 패키지만 비어있음|

#### 🚫 절대 하지 말 것

- `@Version` 낙관적 락 (STEP05)
- `@Lock(PESSIMISTIC_WRITE)` 비관적 락 (STEP05)
- `@Cacheable` 캐시 (STEP06)
- `ApplicationEventPublisher` (STEP07)
- Redis, Kafka 의존성 (STEP06/08)
- **`OrderFacade` 같은 파사드 클래스 도입** (STEP04)

#### ✅ STEP03에서 남은 작업 (이슈 #20, #21, #22 기준)

##### 1) Balance 마무리 — 이슈 #20 (`feat/step03-balance-service`)

**도메인 보강**:

- [ ] `Balance.refund(long amount)` 메서드 추가 (검증 + 잔액 회복)
- [ ] `BalanceTransaction.ofRefund()` 정적 팩토리 추가

**Application 레이어 작성**:

- [ ] `application/BalanceService.java`
    - `chargeBalance(BalanceCommand.Charge)` → `BalanceInfo`
    - `useBalance(BalanceCommand.Use)` → `BalanceInfo`
    - `getBalance(Long userId)` → `BalanceInfo`
- [ ] `domain/BalanceCommand.java` (record 또는 정적 내부 클래스 — 레퍼런스 스타일은 sealed/record 활용)
- [ ] `domain/BalanceInfo.java`

> 단일 도메인이므로 **Facade 없이 Service만으로 충분**. Facade는 STEP04 주문/결제에서 도입.

**Interfaces 레이어 정리**:

- [ ] `BalanceController.getBalance` → `balanceService.getBalance(userId)` 호출 후 `BalanceResponse.from(info)`
- [ ] `BalanceController.chargeBalance` → 동일하게 Service 호출
- [ ] `BalanceResponse.from(BalanceInfo)` 매핑 메서드 추가

**예외 처리**:

- [ ] `ApiControllerAdvice`에 `@ExceptionHandler(IllegalArgumentException.class)` 추가 (`HttpStatus.BAD_REQUEST` + 메시지 그대로 반환)
- [ ] (선택) `CoreException` + `ErrorCode` 도입 — 레퍼런스 스타일. 부담되면 STEP05 "예외 처리 추가" 단계까지 미뤄도 됨

**테스트**:

- [ ] `balance/domain/BalanceTest.java`
    - 충전 한도 초과 시 예외
    - 음수/0 충전 시 예외
    - 정상 충전 후 amount 반영
    - 잔액 부족 사용 시 예외
    - 정상 사용 후 amount 차감
    - 환불 시 amount 회복
- [ ] `balance/service/BalanceServiceTest.java` (Mockito, `@ExtendWith(MockitoExtension.class)`)
- [ ] `balance/controller/BalanceControllerTest.java` 갱신 (Service mock 주입, mock 응답 의존 제거)

> 통합 테스트(Testcontainers)는 **STEP04 #19 "기능별 통합테스트 작성"** 단계로 미루는 것이 레퍼런스 흐름과 일치.

##### 2) Coupon — 이슈 #21 (`feat/step03-coupon-domain`)

레퍼런스 패턴 그대로:

- [ ] `domain/Coupon.java` (정책 정보: discount type, amount, total qty, issued qty, period)
- [ ] `domain/UserCoupon.java` (사용자별 발급/사용 상태)
- [ ] `domain/CouponStatus.java`, `domain/DiscountType.java` enum
- [ ] `Coupon.issue()` — 재고 차감 + 기간 검증 (이 시점에는 락 없이 단순 차감만)
- [ ] `UserCoupon.use()` — 상태 전이 + 중복 사용 방지
- [ ] `CouponRepository`, `UserCouponRepository` 인터페이스 + 구현체
- [ ] `application/CouponService` — `issue`, `use`, `getUserCoupons`
- [ ] `domain/CouponCommand`, `domain/CouponInfo`
- [ ] Controller mock 제거 → Service 호출
- [ ] `Coupon`, `UserCoupon` 단위 테스트
- [ ] `CouponService` 단위 테스트

> 선착순 동시성 제어는 **STEP05**, 분산 락은 **STEP06**. 여기서는 절대 손대지 않음.

##### 3) Product — 이슈 #22 (`feat/step03-product-domain`)

- [ ] `domain/Product.java` (id, name, price, …)
- [ ] `domain/Stock.java` (productId, quantity) — 재고를 별도 엔티티로 분리하는 게 레퍼런스 방식
- [ ] `Product`/`Stock` 비즈니스 메서드 (`Stock.decrease()`, `Stock.restore()`)
- [ ] `ProductRepository`, `StockRepository` 인터페이스 + 구현체
- [ ] `application/ProductService` — `getProducts`, `getProduct`
- [ ] **인기 상품 조회는 STEP06까지 단순 DB 집계로 두고, 도메인 분리 X** (STEP06에서 `Rank` 도메인 분리)
- [ ] `domain/ProductCommand`, `domain/ProductInfo`
- [ ] Controller mock 제거 → Service 호출
- [ ] `Product`, `Stock` 단위 테스트
- [ ] `ProductService` 단위 테스트

#### 📝 STEP03 작업 순서 권장

1. **Balance 마무리** (Service 작성 + Controller 연결 + 단위 테스트) — 이슈 #20
2. **Coupon 도메인 구현** — 이슈 #21
3. **Product 도메인 구현** — 이슈 #22
4. STEP03 완료 후 **WIL 3주차 작성** — 클린 아키텍처 적용 회고 (위 "고민 1~5" 본인 결정 기록)

---

### STEP04 - 도메인 구현 (주문/결제) ⏳

**레퍼런스 흐름** (2025-04-13 ~ 04-17)

```
[FEAT] 통합테스트 위한 설정 추가 및 변경            ← Testcontainers
[FEAT] 사용자/잔액/쿠폰/상품/재고/주문/결제 도메인 Infra Layer
[REFACTOR] 파사드 트랜잭션 적용                    ← OrderFacade 등장
[FEAT] 주문/결제 파사드 통합 테스트
[REFACTOR] @Enumerated(EnumType.STRING) 적용
[DOCS] DB 성능 최적화 보고서 작성                 ← STEP04 마지막!
[REFACTOR] 엔티티 클래스 인덱스 적용              ← 보고서 결과 반영
```

#### 🎯 핵심 학습 포인트

**파사드 패턴 도입** (WIL 3주차 학습 그대로):

```java
@Service
@RequiredArgsConstructor
public class OrderFacade {
    private final OrderService orderService;
    private final BalanceService balanceService;
    private final CouponService couponService;
    private final StockService stockService;

    @Transactional
    public OrderResult createOrder(OrderCriteria criteria) {
        // 1. 주문 생성
        Order order = orderService.createOrder(criteria.toCommand());
        // 2. 잔액 차감
        balanceService.useBalance(order.getUserId(), order.getTotalPrice());
        // 3. 쿠폰 사용
        couponService.useCoupon(criteria.getUserCouponId());
        // 4. 재고 차감
        stockService.decreaseStock(order.getProducts());
        return OrderResult.of(order);
    }
}
```

**STEP07에서 제거할 것이므로 일부러 단순하게 작성** — 직접 호출 방식.

**Testcontainers 도입 시점**: STEP04. 이슈 #19 "기능별 통합테스트 작성" 단계에서 `@SpringBootTest` + Testcontainers MySQL 셋업.

> ⚠️ 본인 `application.yaml`에 `DataSourceAutoConfiguration` exclude 되어 있음. 통합 테스트 시작할 때 이 설정 검토 필요 (실 DB 연결 또는 Testcontainers profile 분리).

**DB 성능 보고서 작성 가이드** (레퍼런스 `01.DBPerformanceOptimizationReport.md` 참고):

- 10만 건 더미 데이터로 측정
- `EXPLAIN ANALYZE` 명령어로 인덱스 적용 전후 비교
- 측정 대상: 잔액 조회, 재고 조회, 사용자 쿠폰 조회 (2종), 상품 조회, 인기 상품 조회
- 결과를 표로 정리 (인덱스 전 시간 / 후 시간 / 개선율)

**예상 성능 개선** (레퍼런스 결과):

- 잔액 조회: 30.9ms → 0.258ms (99.2%)
- 재고 조회: 18.7ms → 0.224ms (98.8%)
- 사용자 쿠폰 조회: 33.3ms → 0.101ms (99.7%)

---

### STEP05 - 동시성 ⏳

**레퍼런스 흐름** (2025-04-22 ~ 04-25)

```
[TEST] 동시성 테스트 작성                          ← 실패 테스트 먼저!
[REFACTOR] 재고 비관적 락 동시성 제어 구현
[REFACTOR] 쿠폰 비관적 락 동시성 제어 구현
[REFACTOR] 잔액 낙관적 락 동시성 제어 구현
[REFACTOR] 사용자 쿠폰 유니크 제약조건 추가
[REFACTOR] "XxxWithLock" postfix 일관성
[FEAT] 로깅 필터 적용
[REFACTOR] 예외 처리 추가
[DOCS] 동시성 이슈 분석 및 해결 보고서 작성       ← 핵심 산출물
```

#### 🎯 락 전략 (검증 완료)

|자원|전략|이유|
|---|---|---|
|잔액|**낙관적 락** (`@Version`)|동일 사용자가 동시 충전 = 의도치 않은 중복. 하나만 처리|
|쿠폰|**비관적 락** (`@Lock(PESSIMISTIC_WRITE)`)|선착순은 모두 처리되어야 함. 충돌 잦음|
|재고|**비관적 락**|동시 차감 시 음수 방지. 정확성 최우선|

#### 작업 순서

1. **실패하는 동시성 테스트** (`ExecutorService` + `CompletableFuture`)
2. 테스트 실패 확인 (의도된 실패)
3. 락 적용
4. 테스트 성공 확인
5. 보고서 작성 (AS-IS/TO-BE 스크린샷)

**WIL 5주차 작성**: 동시성 학습 내용 정리 (synchronized vs ReentrantLock vs Atomic, MVCC, 격리 수준).

---

### STEP06 - DB 성능 + 캐시 ⏳

**레퍼런스 흐름** (3 Phase로 분리)

**Phase 1: Redis 분산 락 도입** (2025-04-30)

```
[CHORE] Redis 컨테이너 및 Redisson 추가
[FEAT] Redis 프로덕션 코드 설정
[FEAT] 분산 락 AOP 적용                            ← @DistributedLock
[FEAT] 선착순 쿠폰 분산 락 적용                    ← STEP05 비관적 락에 추가
```

**Phase 2: 인기상품 캐시** (2025-05-04 ~ 05-07)

```
[FEAT] Redis 캐시 관련 클래스 작성
[REFACTOR] Product 도메인에서 인기 상품 조회 분리
[FEAT] Rank 도메인 추가
[REFACTOR] 인기 상품 조회 배치 프로세스 전환
[REFACTOR] 인기 상품 조회 스케줄러 적용
[TEST] 캐시 성능 테스트 위한 K6 스크립트
[DOCS] 캐시 전략 설계 보고서 작성                 ← TTL 49h 설계
```

**Phase 3: Redis 자료구조 전환** (2025-05-15)

```
[FEAT] 인기상품 Redis 자료구조 전환                ← Sorted Set
[DOCS] Redis 디자인 아키텍처 보고서 작성
[FEAT] 인기상품 - Redis에서 40일 이후 DB 영속화
```

#### 🎯 캐시 전략 (Study 02 + 보고서 03 기반)

- 읽기: **Read-Through** (`@Cacheable`)
- 쓰기: **Write-Through** (`@CachePut` + 매일 00:05 스케줄러)
- TTL: **49시간** (24시간이면 배치 시각과 만료 겹침 + hotfix 여유)

#### 🎯 분산 락 + 트랜잭션 순서 (WIL 5주차)

```
1. 분산 락 획득 (트랜잭션 밖)
2. 트랜잭션 시작
3. 비즈니스 로직
4. 트랜잭션 커밋
5. 분산 락 해제 (TransactionSynchronizationManager)
```

> Redis 장애 시에도 정합성 보장 위해 **DB 락 + Redis 분산 락 2중 적용**.

**WIL 5주차 작성 시점**: 5/6주차 동시성 + 분산락 통합 회고.

---

### STEP07 - EDA (이벤트 기반) ⏳

**레퍼런스 흐름** (2025-05-21 ~ 05-22, **거의 하루 만에 폭풍 작업**)

**Phase 1: 외부 데이터 플랫폼 전송 분리** (5/21)

```
[REFACTOR] Order interfaces api 패키지 하위 이동
[REFACTOR] Message 도메인 개념 추출
[REFACTOR] 주문 결제 외부 플랫폼 전송 → 이벤트 기반
```

**Phase 2: 도메인 이벤트 + 파사드 제거** (5/22)

```
[FEAT] 비동기 설정 추가                           ← @EnableAsync
[FEAT] Order/Balance/Coupon/Payment/Stock/Message/Rank 이벤트 작성
[REFACTOR] MSA 기반 Balance/Product/Rank/Coupon/Order 파사드 제거
[REFACTOR] Balance 환불/Coupon 사용 취소/Payment 결제 취소/Stock 재고 복구
[REFACTOR] @Transactional 누락 추가
[DOCS] MSA 기반 이벤트 아키텍처 설계 보고서
```

#### 🎯 이벤트 패턴 (검증 완료)

```java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(OrderEvent.Created event) {
    // 트랜잭션 커밋 후 비동기 처리
}
```

#### 작업 순서

1. 가장 단순한 케이스 먼저 (외부 데이터 플랫폼 전송)
2. 도메인별 이벤트 객체 정의
3. 이벤트 리스너 작성 (`XxxEventListener`)
4. 파사드 클래스를 도메인별로 하나씩 제거
5. 보상 트랜잭션 메서드 추가 (refund, cancel, restore)

#### 🎯 보상 트랜잭션 (Saga 패턴)

결제 실패 시 잔액 환불, 재고 복구.

---

### STEP08 - Kafka ⏳

**레퍼런스 흐름** (2025-05-29 ~ 06-02)

```
[FEAT] Kafka 설정
[FEAT] 데이터 역직렬화 클래스
[FEAT] Message - Kafka 인터페이스
[FEAT] Event 객체 + 토픽/컨슈머 그룹 정의
[FEAT] Outbox 구현                                ← 트랜잭션-메시지 정합성
[REFACTOR] 외부 데이터 전송 → Kafka 발행
[DOCS] 카프카 기초 및 핵심 개념 문서 작성
[REFACTOR] Kafka 활용 쿠폰 발급 프로세스 변경     ← 선착순 순서 보장!
[FEAT] 쿠폰/주문/결제 이벤트 리스너 작성
[REFACTOR] 인기상품 배치 제거 → 실시간 이벤트     ← 5번째 진화!
[DOCS] 쿠폰 발급 카프카 기반 설계 문서
```

#### 🎯 핵심 학습 포인트 (Study 01)

- ApplicationEvent → Kafka로 발행처만 교체 (리스너 거의 그대로)
- **Outbox 패턴**: 트랜잭션 + 메시지 발행 정합성
- 쿠폰 발급을 Kafka로 직렬 처리 → STEP05 "공정성 한계" 해결
- **DLQ**: 실패 메시지 별도 토픽 (`-dlt` 접미사)

**Outbox 흐름**:

```
1. BEFORE_COMMIT: Outbox 테이블 저장
2. AFTER_COMMIT (@Async): Kafka 발행
3. Consumer 처리 완료 시: eventId로 Outbox 삭제
```

---

### STEP09 - 부하테스트 ⏳

**레퍼런스 흐름** (2025-06-05 ~ 06-06)

```
[CHORE] 도커 + 부하 테스트 환경 구축              ← Prometheus, Grafana
[FEAT] 주문 상세 조회 API
[FEAT] 상품 조회 커서 페이징
[FEAT] 부하 테스트 픽스쳐
[FEAT] 부하 테스트 스크립트
[DOCS] 부하 테스트 시나리오 보고서
[REFACTOR] 카프카 에러 로깅 / Concurrency 옵션 / CoreException 핸들링
[DOCS] 부하 테스트 성능 지표 + 병목 + 개선
```

#### 🎯 부하 테스트 시나리오

- **주문/결제** (현실적 트래픽 분포)
    1. 인기상품 조회
    2. 잔액 충전 (전체의 20%)
    3. 잔액 조회
    4. 주문/결제 (충전자의 10%)
    5. 주문 완료 확인
- **선착순 쿠폰 발급** (Peak Test)
    - 단일 시점 동시 폭주 시나리오

**환경 구축**:

- Spring Actuator + Prometheus + Grafana
- Docker 리소스 제한 (CPU 2 vCPU, 메모리 4GB)
- K6 스크립트

---

### 🏁 멀티모듈 분리 (옵션, STEP09 이후)

**레퍼런스 흐름** (2025-07-07, 하루 만에)

```
[REFACTOR] 공통:캐시/클라이언트/락/메시지/아웃박스/직렬화/스토리지 모듈 분리
[REFACTOR] 서비스:잔액/쿠폰/주문/결제/상품/유저 모듈 분리
[REFACTOR] 지원:RestDocs 모듈 분리
[REFACTOR] 기존 모놀리식 코드 삭제
```

---

## 📐 작업 표준

### 브랜치 명명

```
docs/step01-requirements
feat/step03-balance-service        ← 본인 다음 브랜치
feat/step03-coupon-domain
feat/step03-product-domain
test/step05-concurrency
refactor/step07-order-event
```

### 커밋 메시지

|태그|사용처|예시|
|---|---|---|
|`[DOCS]`|문서|`[DOCS] 동시성 이슈 분석 및 해결 보고서 작성`|
|`[FEAT]`|새 기능|`[FEAT] Balance Service 및 Command/Info 작성`|
|`[TEST]`|테스트|`[TEST] BalanceService 단위 테스트 작성`|
|`[REFACTOR]`|개선|`[REFACTOR] 잔액 낙관적 락 동시성 제어 구현`|
|`[FIX]`|버그|`[FIX] 분산락, 트랜잭션 해제 순서 보장 문제 수정`|
|`[CHORE]`|빌드/설정|`[CHORE] Redis 컨테이너 및 Redisson 추가`|
|`[REVERT]`|되돌리기|`[REVERT] 캐시 delimiter 수정`|

### PR 사이클

```
1. 이슈 #N 확인 (sub-issue 단위)
2. 브랜치 생성
3. TDD: 실패 테스트 → 구현 → 통과 테스트
4. 커밋
5. Push → PR → 셀프 리뷰
6. main 머지 → 이슈 close → 칸반 Done
```

---

## 🚦 단계별 신호등

|작업|S03|S04|S05|S06|S07|S08|
|---|---|---|---|---|---|---|
|도메인 엔티티 비즈니스 메서드|🟢|🟢|🟢|🟢|🟢|🟢|
|4-Layer 클린 아키텍처|🟢|🟢|🟢|🟢|🟢|🟢|
|Repository 인터페이스 + 구현체 분리|🟢|🟢|🟢|🟢|🟢|🟢|
|단위/통합 테스트|🟢|🟢|🟢|🟢|🟢|🟢|
|파사드 클래스 도입|🔴|🟢|🟢|🟢|🔴(제거)|🔴|
|인덱스 적용|🔴|🟢|🟢|🟢|🟢|🟢|
|`@Version` 낙관적 락|🔴|🔴|🟢|🟢|🟢|🟢|
|`@Lock` 비관적 락|🔴|🔴|🟢|🟢|🟢|🟢|
|Redis 분산 락|🔴|🔴|🔴|🟢|🟢|🟢|
|`@Cacheable` 캐시|🔴|🔴|🔴|🟢|🟢|🟢|
|배치 스케줄러|🔴|🔴|🔴|🟢|🔴(제거)|🔴|
|`ApplicationEventPublisher`|🔴|🔴|🔴|🔴|🟢|🟢|
|`@TransactionalEventListener`|🔴|🔴|🔴|🔴|🟢|🟢|
|Kafka Producer/Consumer|🔴|🔴|🔴|🔴|🔴|🟢|
|Outbox 패턴|🔴|🔴|🔴|🔴|🔴|🟢|

🟢 = 가능 / 🔴 = 금지

---

## 📝 산출물 체크리스트

### docs/architecture (STEP01~02 완료 ✅)

- [x] 01.Requirements.md
- [x] 02.Milestones.md
- [x] 03.SequenceDiagram.md
- [x] 03-2.StateDiagram.md
- [x] 04.ERD.md
- [x] 05.ApiDocument.md

### docs/report (STEP04부터 누적)

- [ ] 01.DBPerformanceOptimizationReport.md (STEP04 끝)
- [ ] 02.ConcurrencyReport.md (STEP05 끝)
- [ ] 03.CacheStrategyArchitectureReport.md (STEP06 Phase 2)
- [ ] 04.RedisDesignArchitectureReport.md (STEP06 Phase 3)
- [ ] 05.MsaEventDrivenArchitectureReport.md (STEP07)
- [ ] 06.KafkaDesignArchitectureReport.md (STEP08)
- [ ] 07.LoadTestReport.md (STEP09)

### docs/study (학습 정리)

- [ ] 01.Kafka.md (STEP08 시작 시)
- [ ] 02.Cache.md (STEP06 시작 시)

### docs/devlog 또는 docs/WIL (주차별 회고)

- [x] 2주차 (devlog/이커머스 아키텍처 설계.md)
- [ ] **3주차: 클린 아키텍처 적용 회고** ← STEP03 끝나고 작성
- [ ] 4주차: 트랜잭션 + 인덱스 학습 (STEP04 끝)
- [ ] 5주차: 동시성 + 분산락 학습 (STEP05~06 끝)

### 코드 진척 — Balance 도메인 상세 (현재 진행 중)

|파일|상태|
|---|---|
|`balance/domain/Balance.java`|✅ 작성됨 (refund 메서드만 추가하면 완료)|
|`balance/domain/BalanceTransaction.java`|✅ 작성됨 (ofRefund 추가 필요)|
|`balance/domain/TransactionType.java`|✅ 작성됨|
|`balance/domain/BalanceRepository.java`|✅ 작성됨|
|`balance/domain/BalanceTransactionRepository.java`|✅ 작성됨|
|`balance/infrastructure/BalanceCoreRepository.java`|✅ 작성됨|
|`balance/infrastructure/BalanceJpaRepository.java`|✅ 작성됨|
|`balance/infrastructure/BalanceTransactionCoreRepository.java`|✅ 작성됨|
|`balance/infrastructure/BalanceTransactionJpaRepository.java`|✅ 작성됨|
|`balance/interfaces/BalanceController.java`|⚠️ Mock 응답 — Service 호출로 교체 필요|
|`balance/interfaces/BalanceRequest.java`|✅ 작성됨|
|`balance/interfaces/BalanceResponse.java`|⚠️ `from(BalanceInfo)` 매핑 필요|
|`balance/application/BalanceService.java`|❌ 미작성|
|`balance/domain/BalanceCommand.java`|❌ 미작성|
|`balance/domain/BalanceInfo.java`|❌ 미작성|
|`common/ApiControllerAdvice.java`|⚠️ `IllegalArgumentException` 핸들러 추가 필요|
|`test/.../balance/domain/BalanceTest.java`|❌ 미작성 (디렉터리만 존재)|
|`test/.../balance/service/BalanceServiceTest.java`|❌ 미작성|

---

## 📊 레퍼런스 통계

- 총 커밋: **274**
- 개발 기간: 2024-12-27 ~ 2025-07-09 (약 6.5개월)
- 보고서: 7개
- WIL: 4편
- Study: 2편

**커밋 태그 분포**

- `[REFACTOR]` ~35%
- `[FEAT]` ~30%
- `[DOCS]` ~25%
- `[TEST]` ~8%
- `[FIX]/[CHORE]/[REVERT]` ~2%

**시사점**: 점진적 개선이 핵심. 완벽주의 금지.

---

## ✅ 사용법

1. STEP 시작 전 → 해당 STEP 섹션 정독 (너가 직접 수정하지말고 나에게 알려주기 +왜 이런작업을 하는지 간략히 설명)
2. "절대 하지 말 것" 체크 → 미래 도구 안 쓰기
3. 작업 중 → 신호등 표 확인
4. STEP 완료 → 산출물 체크리스트 갱신 
5. 막히면 → 레퍼런스 동일 시점 커밋 확인 

### 🔥 다음 액션 (지금 당장)

1. **이슈 #20 — Balance 마무리** 브랜치 생성: `feat/step03-balance-service`
2. `Balance.refund()` + `BalanceTransaction.ofRefund()` 추가
3. `BalanceCommand`, `BalanceInfo` 작성
4. `BalanceService` 작성 (`chargeBalance`, `useBalance`, `getBalance`)
5. `BalanceController` mock 제거 → Service 호출
6. `ApiControllerAdvice`에 `IllegalArgumentException` 핸들러 추가
7. `BalanceTest`, `BalanceServiceTest` 작성
8. PR → 머지

그 다음 이슈 #21 (Coupon), #22 (Product) 순서로 진행.

---

> 📌 살아있는 문서. STEP 진행 중 발견하는 것들 추가 갱신할 것. 본 v3는 2026-04-30 기준 본인 프로젝트 실제 상태 검토 결과 반영.