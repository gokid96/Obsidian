# 📘 mini-commerce 진행 가이드 (v2)

> 레퍼런스 레포(discphy/e-commerce, 274 커밋)의 **시간순 분석 + 학습 자료(WIL 4편 + Study 2편 + 보고서 7편) 통합 정리** v1과의 차이: STEP별 가이드에 더해 **레퍼런스가 학습한 핵심 개념과 결정 근거**를 모두 통합
- 참고 레포: __[https://github.com/discphy/e-commerce__](https://github.com/discphy/e-commerce__)
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

## 🏛️ 레퍼런스가 채택한 아키텍처 (WIL 3주차 기반)

### 클린 레이어드 아키텍처

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

### 본인 패키지 구조와 비교

본인 현재 구조:

```
balance/
├── controller/   ← 레퍼런스의 interfaces
├── dto/          ← request, response
├── service/      ← 레퍼런스의 application + domain 혼재 (현재 비어있음)
├── domain/       ← 비어있음
└── infrastructure/ ← 비어있음
```

**STEP03 진입 시 결정 사항**: 본인 현재 구조를 유지할지, 레퍼런스의 4-Layer로 전환할지.

> 추천: 4-Layer로 전환. 이유는 레퍼런스 학습의 목적이 클린 아키텍처 체득이고, STEP07에서 **"파사드 클래스 제거"** 가 핵심 이벤트인데, 파사드가 없는 구조에서는 이 학습이 안 됨.

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

**고민 4: 검증 로직은 어디에?**

> 도메인 객체 내부에 두는 것을 선호.

이유: Command에 두면 모든 DTO마다 중복 + 도메인 객체에 이중 검증 필요.

**고민 5: 도메인 클래스 vs JPA 엔티티 분리**

분리하지 않으면 발생하는 4가지 문제:

1. JPA에 의존적인 도메인 구조가 됨
2. 객체 간 협력이 ID 기반으로 제한됨
3. 도메인이 인프라에 의존하게 됨 (DIP 위반)
4. 도메인 관심사가 분리되지 않음

**현실적 절충**: 이번 과제에선 분리하지 않고 진행 (어설픈 분리는 오히려 독).

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

**레퍼런스 흐름** (2025-03-31 ~ 04-01)

```
[DOCS] 요구사항 분석 → 마일스톤 → 시퀀스 다이어그램 → ERD → API 명세
```

**산출물**: docs/architecture/ 5종 (요구사항/마일스톤/시퀀스/ERD/API)

**WIL 작성**: 2주차 회고에 "최소 스펙 vs 확장성" 같은 본인의 설계 결정 근거 기록.

---

### STEP02 - 설계 심화 ✅ 본인 완료

**레퍼런스 흐름** (2025-04-02 ~ 04-03)

```
[FEAT] Mock API 작성
[DOCS] Spring REST Docs
[DOCS] 상태 다이어그램
[DOCS] http 테스트 추가
[DOCS] ERD 상태/타입 정의 + 설계 의도
```

**산출물**: docs/architecture/03-2.StateDiagram.md, REST Docs 자동 문서화

---

### STEP03 - 도메인 구현 (잔액/쿠폰/상품) 🎯 **다음 작업**

**레퍼런스 흐름** (2025-04-10 ~ 04-12)

```
[FEAT] Mock 테스트 지원 클래스 추가
[FEAT] 잔액 충전 어플리케이션 구현              ← Service 레이어 (락 없음!)
[FEAT] 잔액 조회/상품 조회/쿠폰 발급/주문결제 어플리케이션
[REFACTOR] JPA 구현체 추가                       ← 인프라 레이어
[REFACTOR] 정적 팩토리 메서드 추가
[FEAT] 도메인 클래스 검증 추가                   ← 엔티티에 비즈니스 규칙
[REFACTOR] 주문 도메인 리팩토링
```

**🚫 절대 하지 말 것**

- `@Version` 낙관적 락 (STEP05)
- `@Lock(PESSIMISTIC_WRITE)` 비관적 락 (STEP05)
- `@Cacheable` 캐시 (STEP06)
- `ApplicationEventPublisher` (STEP07)
- Redis, Kafka 의존성 (STEP06/08)

**✅ 반드시 할 것**

- 4-Layer 클린 아키텍처 도입 (`interfaces / application / domain / infrastructure`)
- 엔티티에 **비즈니스 메서드** (`balance.charge()`, `balance.use()`)
- 엔티티 메서드 안에서 **검증 로직 캡슐화** (`MAX_BALANCE_AMOUNT` 등)
- Repository 인터페이스(`domain/`) + 구현체(`infrastructure/`) 분리
- 정적 팩토리 메서드 (`Balance.create(userId)`, `BalanceTransaction.ofCharge(...)`)
- 단위 테스트 → 구현 → 통합 테스트 순서
- `BalanceCommand`, `BalanceInfo` 같은 도메인 전용 DTO

**본인 첫 PR — `feat/step03-balance-domain`**

체크리스트:

- [ ] 패키지 4-Layer 구조로 정리
- [ ] `Balance` 엔티티 + `charge/use/refund` 비즈니스 메서드
- [ ] `BalanceTransaction` 엔티티 + `transaction_type` enum
- [ ] `BalanceRepository` 인터페이스 (`domain/`)
- [ ] `BalanceCoreRepository` 구현체 (`infrastructure/`, JPA 위임)
- [ ] `BalanceJpaRepository`, `BalanceTransactionJpaRepository`
- [ ] `BalanceService` (chargeBalance, useBalance, getBalance)
- [ ] `BalanceCommand`, `BalanceInfo` (도메인 입출력)
- [ ] `BalanceController`에서 mock 응답 제거 → Service 호출
- [ ] `Balance` 단위 테스트 (충전 한도, 차감 부족 등)
- [ ] `BalanceService` 단위 테스트 (Mockito)
- [ ] `BalanceIntegrationTest` (`@SpringBootTest` + Testcontainers MySQL)
- [ ] `ApiControllerAdvice`에 `IllegalArgumentException` 핸들러
- [ ] `BalanceControllerDocsTest` 갱신 (실제 Service 동작)

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

**🎯 핵심 학습 포인트**

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

**DB 성능 보고서 작성 가이드** (레퍼런스 01.DBPerformanceOptimizationReport.md 참고):

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

**🎯 락 전략 (검증 완료)**

|자원|전략|이유|
|---|---|---|
|잔액|**낙관적 락** (`@Version`)|동일 사용자가 동시 충전 = 의도치 않은 중복. 하나만 처리|
|쿠폰|**비관적 락** (`@Lock(PESSIMISTIC_WRITE)`)|선착순은 모두 처리되어야 함. 충돌 잦음|
|재고|**비관적 락**|동시 차감 시 음수 방지. 정확성 최우선|

**작업 순서**

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

**🎯 캐시 전략 (Study 02 + 보고서 03 기반)**

- 읽기: **Read-Through** (`@Cacheable`)
- 쓰기: **Write-Through** (`@CachePut` + 매일 00:05 스케줄러)
- TTL: **49시간** (24시간이면 배치 시각과 만료 겹침 + hotfix 여유)

**🎯 분산 락 + 트랜잭션 순서 (WIL 5주차)**

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

**🎯 이벤트 패턴 (검증 완료)**

```java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(OrderEvent.Created event) {
    // 트랜잭션 커밋 후 비동기 처리
}
```

**작업 순서**

1. 가장 단순한 케이스 먼저 (외부 데이터 플랫폼 전송)
2. 도메인별 이벤트 객체 정의
3. 이벤트 리스너 작성 (`XxxEventListener`)
4. 파사드 클래스를 도메인별로 하나씩 제거
5. 보상 트랜잭션 메서드 추가 (refund, cancel, restore)

**🎯 보상 트랜잭션 (Saga 패턴)**: 결제 실패 시 잔액 환불, 재고 복구.

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

**🎯 핵심 학습 포인트 (Study 01)**

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

**🎯 부하 테스트 시나리오**

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
feat/step03-balance-domain
test/step05-concurrency
refactor/step07-order-event
```

### 커밋 메시지

|태그|사용처|예시|
|---|---|---|
|`[DOCS]`|문서|`[DOCS] 동시성 이슈 분석 및 해결 보고서 작성`|
|`[FEAT]`|새 기능|`[FEAT] Balance 도메인 비즈니스 메서드 구현`|
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
- 테이블 뷰 (View 4): 상위 이슈 + sub-issue 계층 구조

### 마일스톤 및 이슈 구조
```

STEP01 - 설계 기본 (#1) - 완료
  ├── 요구사항 분석 문서 작성 (#10) - 완료
  ├── 마일스톤 문서화 및 GitHub 연동 (#11) - 완료
  ├── 시퀀스 다이어그램 작성 (#12) - 완료
  ├── ERD 설계 및 작성 (#13) - 완료
  └── API 명세 작성 (#14) - 완료

STEP02 - 설계 심화 (#2) - 완료
  ├── Mock API 구현 (#15) - 완료
  ├── Spring REST Docs 문서화 (#16) - 완료
  ├── E2E 테스트 작성 (#17) - 완료
  ├── API Request http 파일 작성 (#18) - 완료
  └── 상태 다이어그램 작성 (#19) - 완료

STEP03 - 도메인 구현 잔액/쿠폰/상품 (#3)
  ├── 잔액 비즈니스 로직 구현 및 단위 테스트 (#20)
  ├── 쿠폰 비즈니스 로직 구현 및 단위 테스트 (#21)
  └── 상품 비즈니스 로직 구현 및 단위 테스트 (#22)

STEP04 - 도메인 구현 주문/결제 (#4)
  ├── 주문/결제 비즈니스 로직 구현 및 단위 테스트 (#23)
  ├── 인프라 레이어 구현체 작성 (#24)
  ├── 기능별 통합 테스트 작성 (#25)
  ├── 주요 기능별 동시성 실패 테스트 작성 (#26)
  └── 병목 예상 쿼리 분석 및 최적화 보고서 작성 (#27)

STEP05 - 동시성 (#5)
  ├── 주요 기능별 동시성 테스트 작성 (#28)
  ├── 주요 기능 동시성 이슈 식별 및 해결 (#29)
  ├── 동시성 이슈 분석 및 해결 보고서 작성 (#30)
  ├── Filter/Interceptor/Scheduler 부가 로직 구현 (#31)
  └── 모든 API 정상 작동 및 가용성 확보 (#32)

STEP06 - DB 성능 + 캐시 (#6)
  ├── Redis 기반 분산락 구현 및 적용 (#33)
  ├── Redis 분산락 동시성 보고서 추가 (#34)
  ├── Redis 기반 캐싱 전략 설정 및 적용 (#35)
  ├── 캐싱 전략 및 성능 개선 보고서 작성 (#36)
  ├── 인기상품 Redis 기반 설계 및 구현 (#37)
  ├── 선착순 쿠폰발급 Redis 기반 설계 및 구현 (#38)
  └── Redis 디자인 설계 보고서 작성 (#39)

STEP07 - EDA (#7)
  ├── 주문/결제 완료 시 이벤트 기반 외부 데이터 플랫폼 전송 (#43)
  ├── 파사드 클래스 제거 및 이벤트 기반 도메인 서비스 구현 (#44)
  └── MSA 기반 이벤트 아키텍처 설계 문서 작성 (#45)

STEP08 - Kafka (#8)
  ├── 카프카 기초 및 핵심 개념 문서 작성 (#46)
  ├── 주문 완료 시 데이터 플랫폼으로 카프카 메시지 발행 (#47)
  ├── 대용량 트래픽 프로세스 카프카 활용 구현 (#48)
  ├── Outbox 패턴 적용 (#49)
  └── 카프카 기반 설계 문서 작성 (#50)

STEP09 - 부하테스트 (#9)
  ├── 부하테스트 대상 선정 및 시나리오 계획 문서 작성 (#51)
  ├── 부하테스트 스크립트 작성 (#52)
  ├── 부하테스트 결과 기반 병목 탐색 및 개선 (#53)
  └── 성능 테스트 및 장애대응 보고서 작성 (#54)

```

### discphy e-commerce 전체 이슈 목록

```
[2주차] 서버 구축에 필요한 설계 문서 작성
  ├── 요구사항 분석 문서 작성                              #1  - 완료
  ├── 마일스톤 문서화 및 Github 이슈 및 프로젝트 연동      #2  - 완료
  ├── 시퀀스 다이어그램 작성                               #3  - 완료
  ├── ERD 설계 및 작성                                    #4  - 완료
  ├── API 명세 작성                                       #5  - 완료
  ├── Spring REST Docs 문서화                             #6  - 완료
  ├── API 명세 기반 Mock API 구현                          #7  - 완료
  ├── Mock API 기반 E2E 테스트 작성                        #8  - 완료
  ├── [STEP03] 서버구축-설계 기본과제 (PR)                 #9  - 완료
  ├── [STEP04] 서버구축-설계 심화과제 (PR)                 #10 - 완료
  ├── API Request http 파일 작성                          #11 - 완료
  └── 상태 다이어그램 작성                                 #12 - 완료

[3주차] 클린 아키텍처 기반 비즈니스 로직 구현 및 단위테스트
  ├── 잔액 조회 및 충전 비즈니스 로직 구현 및 단위테스트   #13 - 완료
  ├── 보유 쿠폰 목록 조회 및 선착순 쿠폰 발급 비즈니스 로직 #14 - 완료
  ├── 상품 조회 및 상위 상품 조회 비즈니스 로직            #15 - 완료
  ├── 주문/결제 비즈니스 로직 구현 및 단위테스트           #16 - 완료
  └── [STEP05/06] 비즈니스 로직 개발 및 단위 테스트 (PR)  #17 - 완료

[4주차] 인프라 레이어 구현 / 통합테스트 / DB 성능 / 동시성
  ├── Infrastructure Layer 구현체 작성                   #18 - 완료
  ├── 기능별 통합테스트 작성                              #19 - 완료
  ├── 병목 예상 쿼리 분석 및 최적화 보고서 작성            #20 - 완료
  ├── 주요 기능별 동시성 테스트 작성                      #21 - 완료
  └── [STEP07/08] (PR)                                  #22 - 완료

[5주차] 동시성 이슈 해결 및 API 가용성 확보
  ├── 주요 기능 동시성 이슈 식별 및 해결                  #23 - 완료
  ├── 동시성 이슈 분석 및 해결 보고서 작성                #24 - 완료
  ├── Filter/Interceptor/Scheduler 부가 로직 구현        #25 - 완료
  └── 모든 API 정상 작동 및 가용성 확보                  #26 - 완료

[6주차] Redis 분산락 및 캐싱 처리
  ├── Redis 기반 분산락 구현 및 적용                     #29 - 완료
  ├── Redis 분산락 동시성 보고서 추가                    #30 - 완료
  ├── Redis 기반 캐싱 전략 설정 및 적용                  #31 - 완료
  └── 캐싱 전략 및 성능 개선 보고서 작성                  #32 - 완료

[7주차] Redis 자료구조 디자인 설계 및 구현
  ├── 인기상품 Redis 기반 설계 및 구현                   #36 - 완료
  ├── 선착순 쿠폰발급 기능 Redis 기반 설계 및 구현        #37 - 완료
  ├── 로깅 필터 수정 및 단위/통합테스트 리팩토링           #35 - 완료
  └── Redis 디자인 설계 보고서 작성                      #40 - 완료

[8주차] MSA 기반 이벤트 아키텍처 설계 및 구현
  ├── 주문/결제 완료 시 이벤트 기반 외부 데이터 플랫폼 전송 #41 - 완료
  ├── 파사드 클래스 제거 및 이벤트 기반 도메인 서비스 구현  #42 - 완료
  └── MSA 기반 이벤트 아키텍처 설계 문서 작성             #43 - 완료

[9주차] 카프카 활용 및 비즈니스 고가용성 확보
  ├── 카프카 기초 및 핵심 개념 문서 작성                  #48 - 완료
  ├── 카프카 기반 설계 문서 작성                          #49 - 완료
  ├── 주문 완료 시 데이터 플랫폼으로 카프카 메시지 발행    #46 - 완료
  └── 대용량 트래픽 프로세스 카프카 활용 구현              #47 - 완료

[10주차] 장애대응
  ├── 부하테스트 대상 선정 및 목적, 시나리오 계획 문서 작성 #52 - 완료
  ├── 부하테스트 성능 지표 분석 문서 작성                  #53 - 완료
  ├── 부하 테스트 스크립트 작성                           #54 - 완료
  ├── 부하테스트 결과 기반 병목 탐색 및 개선               #55 - 완료
  ├── [STEP19] 부하테스트 계획 수립 및 문서, 스크립트 작성  #56 - 완료
  └── [STEP20] 부하테스트 성능 지표 분석 및 개선           #57 - 완료
```
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

### docs/WIL (주차별 회고)

- [x] 2주차 (devlog/이커머스 아키텍처 설계.md)
- [ ] 3주차: 클린 아키텍처 적용 회고 (STEP03~04 진행 중)
- [ ] 4주차: 트랜잭션 + 인덱스 학습 (STEP04 끝)
- [ ] 5주차: 동시성 + 분산락 학습 (STEP05~06 끝)

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

1. STEP 시작 전 → 해당 STEP 섹션 정독  (사용자에게 뭐 해야하는지 뭐할건지 알려주기)
2. "절대 하지 말 것" 체크 → 미래 도구 안 쓰기
3. 작업 중 → 신호등 표 확인
4. STEP 완료 → 산출물 체크리스트 갱신 (체크리스트 어디를 갱신해야하는지 알려주기)
5. 막히면 → 레퍼런스 동일 시점 커밋 확인



---

> 📌 살아있는 문서. STEP 진행 중 발견하는 것들 추가 갱신할 것.


