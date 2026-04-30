# 📘 mini-commerce 진행 가이드 (v6)

> **v6 = 학습 자료(report/WIL/study) 작성 시점까지 git log로 검증 완료**. v5 → v6: 산출물 체크리스트와 WIL 작성 전략을 사실 기반으로 정정.
> 
> 현재 나의 레포 : C:\Users\eborder\sungmin\git\e-commerce
> 
> 
> 참고 레포 로컬: `C:\Users\eborder\sungmin\git\e-commerce-reference` 참고 
> 레포 URL: https://github.com/discphy/e-commerce

---

## 📌 v5 → v6 변경 핵심 (학습 자료 작성 시점 정정)

|항목|v5 (잘못됨)|v6 (git log 확정)|
|---|---|---|
|**WIL 3주차 작성 시점**|"STEP03 끝"|✅ **STEP05 끝(`73d868f`, 2025-04-25)** — 작성중→완성은 STEP05 끝|
|**WIL 4주차 작성 시점**|"STEP04 끝"|✅ **STEP09 후 한참 뒤(`1a62e9e`, 2025-06-16)**|
|**WIL 5주차 작성 시점**|"STEP05~06 끝"|✅ **STEP09 후(`48e6409`, 2025-06-27)**|
|**WIL 작성 패턴**|"STEP 끝마다 즉시"|⚠️ **레퍼런스는 한 번에 몰아 작성** — 보고서와 대비|
|**`02.Cache.md` (Study)**|"STEP06 시작"|✅ **2025-07-09** — 멀티모듈 분리 마지막 정리|
|**`01.Kafka.md` (Study)**|"STEP08 시작"|✅ **STEP08 진행 중(`c7e81e2`, 2025-05-29)** — Outbox 구현 같은 날|
|**architecture 06번 파일**|(언급 없음)|✅ **`06.SpringRestDocs.md` 존재** (STEP02)|
|**architecture 03번 파일명**|`03.SequenceDiagram.md`|⚠️ **`03-1.SequenceDiagram.md`** (Sequence/State 분리)|

### 💡 핵심 통찰 — 레퍼런스의 학습 자료 작성 패턴

> **보고서는 즉시, WIL은 나중에, Study는 그 시점**

- **report (보고서)**: 그 STEP 끝나면 **즉시 작성**. 측정 결과/스크린샷 휘발 전에 잡아둠.
- **WIL (회고)**: STEP 끝마다 즉시 안 씀. **2~3개월 후 몰아서 작성**. (단, "작성중" 메모는 그 시점에 남김)
- **Study (개념 학습)**: 그 시점 또는 후반에. Kafka는 STEP08 진행 중 동시 작성, Cache는 멀티모듈 분리 시점에 정리.

### 🔄 v4 → v5 변경 사항도 유지 (Balance 도메인 진화)

|항목|v4|v5 (git log 확정)|
|---|---|---|
|**`Rank` 도메인 분리 시점**|"STEP06"|✅ **STEP05 끝** (`abd2b34`, 2025-04-23, "인기 상품 조회 배치 프로세스 전환")|
|**`@Version` 추가**|"STEP05"|✅ **STEP05 정확히** (`4316790`, 2025-04-22)|
|**`@Transactional` on BalanceService**|"STEP04"|❌ **STEP05 낙관적 락 도입과 동시에 추가** (`4316790`, 2025-04-22)|
|**`@OneToMany` cascade 제거**|(언급 없음)|⚠️ **STEP05 낙관적 락 적용 시 제거** — 연관관계 → ID 참조로 후퇴|
|**`Balance.create()` 시그니처 변경**|(언급 없음)|⚠️ **STEP04: `(userId, amount)` → STEP05: `(userId)` amount=0**|
|**Redisson 첫 등장**|"STEP06"|✅ **STEP06 시작** (`7fc20a3`, 2025-04-30)|
|**`@Cacheable` 첫 등장**|"STEP06"|✅ **STEP06 P2** (`cf10b37`, 2025-05-04)|
|**`ApplicationEventPublisher` 첫 등장**|"STEP07"|✅ **STEP07 시작** (`50bec11`, 2025-05-21)|
|**`@EnableAsync`**|"STEP07"|✅ **STEP07** (`3fc512e`, 2025-05-22)|
|**`KafkaTemplate` 첫 등장**|"STEP08"|✅ **STEP08 시작** (`57a615f`, 2025-05-29)|
|**`Outbox` 첫 등장**|"STEP08"|✅ **STEP08** (`c48d9e0`, 2025-05-29)|
|**인기상품 배치 → 실시간 이벤트**|"STEP08"|✅ **STEP08 끝** (`83bd6c6`, 2025-06-01)|
|**`BaseEntity`**|"STEP04 분리"|❌ **레퍼런스에 BaseEntity 없음** — `createdAt` 필드 자체를 안 둠|

---

## 🎯 핵심 원칙

1. **STEP을 건너뛰지 마라** — 락 없이, 캐시 없이, 이벤트 없이 시작
2. **문서가 코드보다 먼저** — 보고서 작성 후 코드 변경
3. **`[REFACTOR]`는 죄가 아니다** — 274 커밋 중 ~35%
4. **같은 코드를 여러 번 갈아엎는다** — 인기상품 5번 진화, Balance도 STEP04→05에서 큰 리팩토링
5. **학습 → 적용 → 회고 사이클** — WIL/Study/Report 누적

---

## 🏛️ 채택된 아키텍처 — 클린 레이어드 + DIP

```
interfaces  →  application (Facade)  →  domain (Service/Entity/Repo I/F)  ←  infrastructure (Repo Impl, JPA)
```

### 패키지/클래스 네이밍 규칙

|레이어|패키지명|클래스|입력 DTO|출력 DTO|
|---|---|---|---|---|
|Presentation|`interfaces`|`XxxController`|`XxxRequest.Charge` (정적 내부)|`XxxResponse.Balance` (정적 내부)|
|Application|`application`|`XxxFacade`|`XxxCriteria.Charge`|`XxxResult.Balance`|
|Domain|`domain`|`XxxService`|`XxxCommand.Charge`|`XxxInfo.Balance`|
|Infrastructure|`infrastructure`|`XxxCoreRepository` (impl)|-|-|

DTO 변환 흐름: `Request.toCriteria(id)` → `Criteria.toCommand()` → Service.

### 본인 프로젝트 패키지 구조 (도메인 우선 — 멀티모듈 대비)

```
com.github.gokid96.e_commerce
├── balance/{interfaces, application, domain, infrastructure}
├── coupon/, order/, product/   ← interfaces만 채워짐
├── common/                     ← ApiResponse, ApiControllerAdvice
└── ECommerceApplication.java
```

> 본인 패키지 구조 유지 OK. 멀티모듈 분리 시 도메인 우선이 자연스러움.

---

## 🗓️ STEP별 진행 가이드

### STEP01 - 설계 기본 ✅ 완료

산출물: `docs/architecture/` 5종

### STEP02 - 설계 심화 ✅ 완료

산출물: 03-2.StateDiagram.md, REST Docs

---

### STEP03 - 도메인 구현 (잔액/쿠폰/상품) 🎯 **현재 진행 중**

#### 📅 레퍼런스 STEP03 (2025-04-10 ~ 04-12, 16 커밋)

```
fbe40fc  [FEAT] 잔액 충전 어플리케이션 구현       ← BalanceService 첫 등장
4755dfb  [FEAT] 주문 및 결제 어플리케이션 구현
f39ad26  [FEAT] 잔액 조회 어플리케이션 구현
686f4b8  [FEAT] 상품 조회 어플리케이션 구현
40b220f  [FEAT] 보유 쿠폰 목록 조회 어플리케이션 구현
56ebff0  [FEAT] 상위 상품 조회 어플리케이션 구현
eae6310  [REFACTOR] 인터페이스 패키지 구조 변경
a9e17e7  [REFACTOR] JPA 구현체 추가
67bf6fa  [REFACTOR] 정적 팩토리 메서드 추가
f7ac629  [FEAT] 도메인 클래스 검증 추가          ← STEP03 종료 시점
```

#### 🚫 STEP03 절대 하지 말 것

- `@Version` 낙관적 락 → STEP05 (`4316790`)
- `@Lock(PESSIMISTIC_WRITE)` → STEP05
- `@Transactional` on Service → STEP05 (낙관적 락과 함께)
- `Balance.refund()` → STEP07 (`366cfcb`)
- `BalanceTransaction.ofRefund()` → STEP07
- `BalanceTransactionType.REFUND` → STEP07
- `@Cacheable` → STEP06 P2 (`cf10b37`)
- `Redisson` → STEP06 시작 (`7fc20a3`)
- `ApplicationEventPublisher` → STEP07 시작 (`50bec11`)
- `KafkaTemplate` → STEP08 시작 (`57a615f`)
- `BalanceClient` 인터페이스 → STEP07 (`99c8dc7` 파사드 제거 시)
- `useBalance` 엔드포인트 → 멀티모듈 분리 (단, `BalanceService.useBalance` 메서드는 STEP03에 있음)
- `BalanceRepository.saveTransaction` → 존재하지 않음 (STEP05에서 등장)
- `BalanceClient` → STEP07 등장
- `Rank` 도메인 분리 → STEP05 끝 (`abd2b34`)

#### ✅ STEP03 작업 (이슈 #20, #21, #22)

##### 1) 사전작업: User 도메인 (BalanceFacade가 의존)

- [ ] `user/domain/User.java` (id, userName)
- [ ] `user/domain/UserService.java` — `getUser(Long): User`
- [ ] `user/domain/UserRepository.java` 인터페이스
- [ ] `user/infrastructure/UserCoreRepository.java` impl
- [ ] `user/infrastructure/UserJpaRepository.java`
- 사용자 없으면 `IllegalArgumentException("사용자가 존재하지 않습니다.")`

##### 2) Balance 마무리 — 이슈 #20 (`feat/step03-balance-service`)

**Balance 엔티티** (`balance/domain/Balance.java`):

- `@Builder` + `(Long id, Long userId, long amount)` 생성자
- 생성자 안에서 `addChargeTransaction(amount)` 호출
- `@OneToMany(mappedBy="balance", cascade=CascadeType.ALL)` `List<BalanceTransaction> balanceTransactions`
- `Balance.create(Long userId, Long amount)` — amount validation 후 builder
- `charge/use`에서 transaction 자동 추가
- **❌ refund / @Version / @Index 만들지 말 것**

**BalanceTransaction**:

- `@ManyToOne(fetch=LAZY) @JoinColumn(name="balance_id") Balance balance;`
- `userId` 필드 **삭제**, `createdAt` 필드 **삭제**
- `ofCharge(Balance, long amount)` — `+amount`
- `ofUse(Balance, long amount)` — `-amount` (가계부 스타일)
- **❌ ofRefund 만들지 말 것**

**TransactionType → BalanceTransactionType**:

```java
@Getter @RequiredArgsConstructor
public enum BalanceTransactionType {
    CHARGE("충전"), USE("사용"); // REFUND 없음
    private final String description;
}
```

**BalanceCommand** — 정적 내부 클래스 `Charge`, `Use` (Refund 없음)

**BalanceInfo** — 정적 내부 클래스 `Balance`

**BalanceRepository**:

- `@Repository` 인터페이스에 붙임
- `Optional<Balance> findOptionalByUserId(Long userId)`
- `Balance save(Balance balance)`
- **❌ saveTransaction 메서드 만들지 말 것** (cascade 처리)

**삭제할 파일**:

- `BalanceTransactionRepository.java`
- `BalanceTransactionCoreRepository.java`
- `BalanceTransactionJpaRepository.java`

**BalanceCoreRepository**:

- `@Component` (`@Repository` 아님)
- `BalanceJpaRepository`만 의존

**BalanceService** — **`@Transactional` 없이**:

- `chargeBalance(Charge)` — 있으면 charge, 없으면 create+save
- `useBalance(Use)` — 있으면 use, 없으면 예외
- `getBalance(userId)` — 없으면 0 반환
- `Optional.ifPresentOrElse` 패턴 사용

**BalanceFacade** (`balance/application/`):

```java
public void chargeBalance(BalanceCriteria.Charge criteria) {
    userService.getUser(criteria.getUserId());
    balanceService.chargeBalance(criteria.toCommand());
}
public BalanceResult.Balance getBalance(Long userId) {
    userService.getUser(userId);
    BalanceInfo.Balance balance = balanceService.getBalance(userId);
    return BalanceResult.Balance.of(balance.getAmount());
}
```

**BalanceCriteria.Charge** + `toCommand()` 메서드 **BalanceResult.Balance** 정적 내부 클래스

**BalanceController**:

- `BalanceFacade` 의존 (Service 아님)
- GET `/api/v1/users/{id}/balance`
- POST `/api/v1/users/{id}/balance/charge` (메서드명 `updateBalance` — STEP05에서 chargeBalance로 리네임)
- mock 응답 제거
- **❌ /use, /refund 엔드포인트 만들지 말 것**

**BalanceRequest.Charge** + `toCriteria(Long userId)` 메서드 **BalanceResponse.Balance** + `of(BalanceResult.Balance)` 매핑

**ApiControllerAdvice**:

```java
@ExceptionHandler(IllegalArgumentException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ApiResponse<Object> illegalArgumentException(IllegalArgumentException e) {
    return ApiResponse.fail(HttpStatus.BAD_REQUEST.value(), e.getMessage());
}
```

**테스트 4종**:

- `BalanceTest.java` — 7 케이스 (도메인 단위, transactions size도 검증)
- `BalanceServiceTest.java` — Mockito 9 케이스
- `BalanceFacadeTest.java` — InOrder로 userService → balanceService 순서 검증 2 케이스
- `BalanceControllerTest.java` 갱신 — `@MockBean BalanceFacade`

##### 3) Coupon 도메인 — 이슈 #21

##### 4) Product 도메인 — 이슈 #22

> 진입 시 git log로 정밀 확인 후 v6에서 갱신.

##### 5) WIL 3주차 작성 — STEP03 끝나면

---

### STEP04 - 도메인 구현 (주문/결제) ⏳

#### 📅 레퍼런스 흐름 (2025-04-13 ~ 04-17, 약 24 커밋)

```
9d30393  [FEAT] 통합테스트 위한 설정 추가 및 변경     ← Testcontainers 도입
84ef024  [FEAT] 사용자 도메인 Infra Layer
d3c68d3  [FEAT] 잔액 도메인 Infra Layer            ← BalanceJpaRepository 첫 등장
c80751e  [FEAT] 쿠폰 도메인 Infra Layer
3862d72  [FEAT] 상품 도메인 Infra Layer
75614a8  [FEAT] 재고 도메인 Infra Layer
b845b3e  [FEAT] 주문 도메인 Infra Layer
2a8785e  [FEAT] 결제 도메인 Infra Layer
5866c85  [REFACTOR] 파사드 트랜잭션 적용             ← OrderFacade @Transactional
581d29b  [FEAT] 주문/결제 파사드 통합 테스트
f4bb847  [FEAT] 상품 파사드 통합 테스트
c897f7c  [REFACTOR] @Enumerated(EnumType.STRING) 적용
6611f80  [DOCS] DB 성능 최적화 보고서 작성          ← STEP04 핵심 산출물
f66c201  [REFACTOR] 엔티티 클래스 인덱스 적용         ← @Index 첫 등장
```

#### 🎯 STEP04 핵심 학습 포인트

**1) Testcontainers 통합 테스트**:

- `@SpringBootTest` + MySQL 컨테이너
- 본인 `application.yaml`의 `DataSourceAutoConfiguration` exclude 검토 필요
- `BalanceServiceIntegrationTest`, `BalanceRepositoryTest`, `BalanceFacadeIntegrationTest` 작성

**2) JPA Repository 추가**:

- `BalanceJpaRepository extends JpaRepository<Balance, Long>` — `findByUserId`
- `BalanceRepositoryImpl` (`@Component`) 가 `BalanceJpaRepository` 위임

**3) Repository 인터페이스 변경**:

- `@Repository` → `@Component` (인터페이스에)
- ※ STEP03→04에서 어노테이션 변경됨 (확인됨)

**4) OrderFacade — 진짜 facade의 진가**:

```java
@Service @RequiredArgsConstructor
public class OrderFacade {
    @Transactional
    public OrderResult createOrder(OrderCriteria criteria) {
        // 1. 주문 생성 → 2. 잔액 차감 → 3. 쿠폰 사용 → 4. 재고 차감
    }
}
```

**Facade에 @Transactional 적용** — Service에는 STEP05에 가서 추가됨.

**5) `@Enumerated(EnumType.STRING)`** 모든 enum 필드에 적용 (ordinal 위험 회피)

**6) 인덱스 적용**:

- 보고서 먼저 작성 (`6611f80`) → 코드 적용 (`f66c201`) 순서
- 10만 건 더미 + EXPLAIN ANALYZE
- 측정: 잔액/재고/쿠폰/상품/인기상품 조회

#### 📝 STEP04 산출물

- [ ] `docs/report/01.DBPerformanceOptimizationReport.md` — STEP04 마지막 작성
- [ ] WIL 4주차 — 트랜잭션 + 인덱스 학습

#### 🚦 STEP04 신호등 변경

- 🟢 추가 가능: 인덱스, `@Transactional` on Facade, Testcontainers
- 🔴 여전히 금지: `@Version`, `@Lock`, refund, BalanceClient, Redis, 캐시, 이벤트, Kafka

---

### STEP05 - 동시성 제어 ⏳

#### 📅 레퍼런스 흐름 (2025-04-22 ~ 04-25, 약 10 커밋)

```
5671ca9  [TEST] 동시성 테스트 작성                  ← STEP04 끝에 미리 작성된 실패 테스트
4316790  [REFACTOR] 잔액 낙관적 락 동시성 제어 구현  ← 핵심 커밋!
abd2b34  [REFACTOR] 인기 상품 조회 배치 프로세스 전환 - 랭크 도메인 추가  ← Rank 도메인 분리!
96c3b1c  [REFACTOR] 인기 상품 조회 스케줄러 적용
0a4279e  [TEST] 데이터 클렌징 추가 및 동시성 리팩토링
28d9607  [REFACTOR] 필요없는 파사드 동시성 테스트 파일 삭제
bddd975  [TEST] RestAssured E2E Test 추가
f710659  [REFACTOR] updateBalance 에서 chargeBalance 로 변경  ← API 메서드명 변경
22c67ff  [REFACTOR] ControllerTestSupport 패키지 이동
```

#### 🎯 STEP05 핵심 학습 포인트

**1) 락 전략**:

|자원|전략|이유|
|---|---|---|
|잔액|낙관적 락 (`@Version`)|동일 사용자 동시 충전 = 의도치 않은 중복. 하나만 처리|
|쿠폰|비관적 락 (`@Lock(PESSIMISTIC_WRITE)`)|선착순 모두 처리되어야 함|
|재고|비관적 락|동시 차감 시 음수 방지|

**2) STEP05 Balance 큰 리팩토링** (`4316790`):

```java
// AS-IS (STEP04):
public class Balance {
    @OneToMany(mappedBy="balance", cascade=CascadeType.ALL)
    private List<BalanceTransaction> balanceTransactions;
    
    public static Balance create(Long userId, Long amount) { ... }
    public void charge(long amount) { ...; addChargeTransaction(amount); }
}

// TO-BE (STEP05):
@Table(name="balance", indexes={@Index(name="idx_user_id", columnList="user_id")})
public class Balance {
    @Version private Integer version;
    // @OneToMany 제거! balanceTransactions 필드 제거!
    
    public static Balance create(Long userId) {
        return Balance.builder().userId(userId).amount(0L).build();
    }
    public void charge(long amount) { ...; this.amount += amount; }  // transaction 추가 안 함
}

@Entity
public class BalanceTransaction {
    private Long balanceId;  // ← Balance 참조 → Long 으로!
}
```

→ **이유**: 낙관적 락 적용 시 cascade가 영속성 컨텍스트에서 문제 일으킴. 연관관계 깨고 ID 참조로 후퇴 + BalanceService에서 `saveTransaction()` 명시 호출.

**3) BalanceRepository 변경**:

```java
// 추가
BalanceTransaction saveTransaction(BalanceTransaction transaction);
```

**4) BalanceService에 `@Transactional` 추가**:

```java
@Transactional
public void chargeBalance(BalanceCommand.Charge command) {
    Balance balance = balanceRepository.findOptionalByUserId(command.getUserId())
        .orElseGet(() -> balanceRepository.save(Balance.create(command.getUserId())));
    balance.charge(command.getAmount());
    balanceRepository.saveTransaction(BalanceTransaction.ofCharge(balance, command.getAmount()));
}
```

**5) Rank 도메인 분리** (`abd2b34`):

- 인기상품 조회를 Product 도메인에서 분리
- 배치 프로세스로 전환 (스케줄러 적용 — `96c3b1c`)
- 이유: 인기상품 집계 쿼리가 무거움 → 배치 사전 계산

**6) API 메서드명 변경** (`f710659`):

- Controller `updateBalance` → `chargeBalance`
- 의미 명확화

**7) 동시성 테스트 패턴**:

```java
ExecutorService executor = Executors.newFixedThreadPool(N);
CountDownLatch latch = new CountDownLatch(N);
List<CompletableFuture<Void>> futures = ...;
// 실패 테스트 먼저 → 락 적용 → 성공 확인
```

#### 📝 STEP05 산출물

- [ ] `docs/report/02.ConcurrencyReport.md` — AS-IS/TO-BE 측정 결과
- [ ] WIL 5주차 — 동시성 + 락 학습

#### 🚦 STEP05 신호등 변경

- 🟢 추가: `@Version`, `@Lock`, `@Transactional` on Service, Rank 도메인 분리, 배치 스케줄러
- 🔴 여전히 금지: refund, BalanceClient, Redis, 캐시, 이벤트, Kafka

#### ⚠️ STEP05 진입 시 본인 코드 큰 변경 예고

- `Balance.create(userId, amount)` → `Balance.create(userId)` 시그니처 변경
- `Balance` 의 `@OneToMany` 제거 + `balanceTransactions` 필드 제거
- `BalanceTransaction.balance` 참조 → `balanceId` Long으로
- `BalanceRepository.saveTransaction` 메서드 추가
- `Balance` 에 `@Version` 추가
- `Balance` 에 `@Index` 추가 (`@Table(indexes=...)`)
- `BalanceService` 에 `@Transactional` 추가
- `BalanceController.updateBalance` → `chargeBalance`

---

### STEP06 - DB 성능 + Redis (분산락 + 캐시) ⏳

#### 📅 레퍼런스 흐름 (3 Phase, 2025-04-30 ~ 05-15)

**Phase 1: Redis + 분산 락** (2025-04-30)

```
7fc20a3  [FEAT] Redis 프로덕션 코드 설정             ← Redisson 첫 등장
756fa05  [FEAT] 분산 락 AOP 적용                    ← @DistributedLock + 11 파일 신규
                                                       (LockTemplate, KeyGenerator, Strategy 등)
```

**Phase 2: 인기상품 캐시** (2025-05-04 ~ 05-09)

```
cf10b37  [FEAT] Rank 도메인 "인기 상품 조회" 기능 추가   ← @Cacheable 첫 등장
                                                          (TTL 49h 설계)
28e781f  [REFACTOR] 분산락 단위테스트 작성 및 통합테스트 작성
```

**Phase 3: Redis 자료구조 전환** (2025-05-15 ~ 05-16)

```
82ee0f0  [FEAT] 인기상품 Redis 자료구조 전환          ← Sorted Set
c3c687b  [FEAT] 인기상품 - Redis에서 40일 이후 DB 영속화
b891d2e  [REFACTOR] 사용자 쿠폰 도메인 - 발급 요청/완료 여부 기능 구현
```

#### 🎯 STEP06 핵심 학습 포인트

**1) 분산락 AOP 패턴** (`756fa05`, 11개 신규 파일):

- `support/lock/DistributedLock.java` — 어노테이션
- `support/lock/DistributedLockAspect.java` — AOP
- `support/lock/LockKeyGenerator.java` — SpEL 기반 key 생성
- `support/lock/LockStrategy.java` + `LockStrategyRegistry` — 전략 패턴
- `support/lock/LockType.java` — Spin / PubSub
- `support/lock/LockTemplate.java` — 추상 인터페이스
- `support/lock/LockCallback.java`
- `infrastructure/lock/PubSubLockTemplate.java` — Redisson Pub/Sub
- `infrastructure/lock/SpinLockTemplate.java` — Busy-wait

**2) 분산 락 + 트랜잭션 순서 (매우 중요)**:

```
1. 분산 락 획득 (트랜잭션 밖)
2. 트랜잭션 시작
3. 비즈니스 로직
4. 트랜잭션 커밋
5. 분산 락 해제 (TransactionSynchronizationManager)
```

구현: `@Order(Ordered.HIGHEST_PRECEDENCE)` 락 AOP가 트랜잭션보다 먼저 실행

**3) 캐시 전략**:

- 읽기: **Read-Through** (`@Cacheable`)
- 쓰기: **Write-Through** (`@CachePut` + 매일 00:05 스케줄러)
- TTL: **49시간** (24시간이면 배치 시각과 만료 겹침 + hotfix 여유)

**4) Redis Sorted Set 전환**:

- 인기상품을 Sorted Set으로 (score = 판매량)
- 40일 이후 RDB 영속화 스케줄러

**5) DB 락 + Redis 분산 락 2중 적용**:

> Redis 장애 시에도 정합성 보장. 비관적 락 (DB) + Redisson PubSub Lock (Redis) 둘 다.

#### 📝 STEP06 산출물

- [ ] `docs/study/02.Cache.md` — 시작 시점 학습 정리
- [ ] `docs/report/03.CacheStrategyArchitectureReport.md` — Phase 2 끝
- [ ] `docs/report/04.RedisDesignArchitectureReport.md` — Phase 3 끝

#### 🚦 STEP06 신호등 변경

- 🟢 추가: Redis 분산 락, `@Cacheable`, 배치 스케줄러
- 🔴 여전히 금지: refund, BalanceClient, 이벤트, Kafka

---

### STEP07 - EDA (이벤트 기반 + 파사드 제거) ⏳

#### 📅 레퍼런스 흐름 (2025-05-21 ~ 05-22, 약 18 커밋, 거의 하루 만에 폭주)

**Phase 1: 외부 데이터 플랫폼 전송 이벤트화** (5/21)

```
50bec11  [REFACTOR] 주문 결제시, 외부 플랫폼 데이터 전송 방식 이벤트 기반으로 변경
         ← ApplicationEventPublisher + @TransactionalEventListener 첫 등장
```

**Phase 2: 도메인 이벤트 + 파사드 제거** (5/22)

```
3fc512e  [FEAT] 비동기 설정 추가                       ← @EnableAsync 첫 등장
ecd21f3  [FEAT] Balance 이벤트 작성
f98e6eb  [FEAT] Coupon 이벤트 작성
311ee76  [FEAT] Payment 이벤트 작성
15ee835  [FEAT] Stock 이벤트 작성
ec08379  [FEAT] Message 이벤트 작성
93adde2  [FEAT] Rank 이벤트 작성
5647f05  [FEAT] Order 이벤트 작성

87f7874  [REFACTOR] MSA 기반 Rank 파사드 클래스 제거
99c8dc7  [REFACTOR] MSA 기반 Balance 파사드 클래스 제거    ← BalanceClient 첫 등장
                                                            (BalanceCriteria/Result 삭제,
                                                             BalanceClient 인터페이스 추가)
... (다른 도메인도 모두 파사드 제거)

366cfcb  [REFACTOR] Balance 잔액 환불 구현              ← refund 메서드 첫 등장!
                                                          (Balance/Command/Service/Transaction
                                                           +REFUND enum 추가)

fa25a65  [REFACTOR] Balance 패키지 이동
6e87eca  [REFACTOR] @TransactionalEventListener AFTER_COMMIT 변경
```

#### 🎯 STEP07 핵심 학습 포인트

**1) 이벤트 패턴**:

```java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(OrderEvent.Created event) {
    // 트랜잭션 커밋 후 비동기 처리
}
```

**2) Balance 환불 구현** (`366cfcb`, +162줄):

- `Balance.refund(amount)` — 음수/0 검증만 (한도 검증 X)
- `BalanceCommand.Refund` 정적 내부 클래스 추가
- `BalanceService.refundBalance(Refund command)` 추가
- `BalanceTransaction.ofRefund(balance, amount)` (+amount)
- `BalanceTransactionType.REFUND("환불")` enum 추가
- 통합/단위 테스트 추가

**3) 파사드 제거 패턴**:

- `BalanceFacade` 삭제, `BalanceCriteria/Result` 삭제
- `BalanceClient` 인터페이스 신규 (`domain/balance/`)
    
    ```java
    public interface BalanceClient {    BalanceInfo.User getUser(Long userId);}
    ```
    
- `BalanceApiClient` impl (`infrastructure/balance/client/`) — UserService 직접 호출 X, 외부 API 호출 형태로
- `BalanceService`가 `BalanceClient` 의존
- `BalanceInfo.User` 정적 내부 클래스 추가 (BalanceClient 응답용)

**4) Saga 보상 트랜잭션**:

- 결제 실패 → 잔액 환불 (`Balance.refund`)
- 결제 실패 → 쿠폰 사용 취소
- 결제 실패 → 재고 복구 (`Stock.restore`)

**5) BalanceService 최종 형태** (`366cfcb` 시점):

```java
@Service
@RequiredArgsConstructor
public class BalanceService {
    private final BalanceClient balanceClient;
    private final BalanceRepository balanceRepository;

    @Transactional
    public void chargeBalance(BalanceCommand.Charge command) {
        balanceClient.getUser(command.getUserId());  // ← Facade 제거, Service가 검증
        Balance balance = balanceRepository.findOptionalByUserId(command.getUserId())
            .orElseGet(() -> balanceRepository.save(Balance.create(command.getUserId())));
        balance.charge(command.getAmount());
        balanceRepository.saveTransaction(BalanceTransaction.ofCharge(balance, command.getAmount()));
    }

    @Transactional
    public void refundBalance(BalanceCommand.Refund command) {
        Balance balance = balanceRepository.findOptionalByUserId(command.getUserId())
            .orElseThrow(() -> new IllegalArgumentException("잔고가 존재하지 않습니다."));
        balance.refund(command.getAmount());
        balanceRepository.saveTransaction(BalanceTransaction.ofRefund(balance, command.getAmount()));
    }
    // ... useBalance, getBalance
}
```

#### 📝 STEP07 산출물

- [ ] `docs/report/05.MsaEventDrivenArchitectureReport.md`

#### 🚦 STEP07 신호등 변경

- 🟢 추가: refund, BalanceClient, ApplicationEvent, @TransactionalEventListener, @EnableAsync
- 🔴(제거): 파사드 클래스, 배치 스케줄러 (인기상품 배치는 STEP08에서 제거)
- 🔴 여전히 금지: Kafka, Outbox

---

### STEP08 - Kafka + Outbox ⏳

#### 📅 레퍼런스 흐름 (2025-05-29 ~ 06-02, 약 24 커밋, 하루 폭주!)

```
57a615f  [FEAT] Kafka 설정                          ← KafkaTemplate 첫 등장
ec78818  [REFACTOR] 기존 Message 도메인 제거
5865be7  [FEAT] 데이터 역직렬화 클래스 작성
2b88ec5  [FEAT] Message - Kafka 인터페이스 작성
424c904  [FEAT] Event 객체 생성 및 토픽, 컨슈머 그룹 정의
c48d9e0  [FEAT] Outbox 구현                         ← Outbox 첫 등장
a3d08cf  [REFACTOR] 주문 완료 시, 외부 데이터 플랫폼 주문 데이터 전송 카프카 이벤트 발행으로 변경
c7e81e2  [DOCS] 카프카 기초 및 핵심 개념 문서 작성  ← Study 01 작성
54c8e37  [REFACTOR] Kafka를 활용한 쿠폰 발급 프로세스 변경
ab4cdfd  [FEAT] 쿠폰 이벤트 리스너 작성
07b5283  [TEST] 쿠폰 발급 요청 및 발급 완료 이벤트 테스트
b74571f  [REFACTOR] 쿠폰 발급 변경으로 인한 api 코드 변경
7d6e701  [REFACTOR] 주문 생성 로직 간소화
32a5048  [REFACTOR] 결제 시, 포인트 차감 및 쿠폰 사용 로직 추가
eaa03bc  [FEAT] 주문/결제 관련 이벤트 리스너 작성
897a9b4  [TEST] 주문/결제 완료 이벤트 테스트
f8ca4e6  [DOCS] 쿠폰 발급 프로세스 카프카 기반 설계 문서 작성

83bd6c6  [REFACTOR] 인기상품 배치 제거 및 실시간 이벤트 변경  ← 5번째 진화!
                                                                (배치 → 실시간 이벤트)
bf49124  [REFACTOR] Outbox 수동 제거, @Transactional 없는 로직 바로 카프카 메시지 발행하게끔 수정
```

#### 🎯 STEP08 핵심 학습 포인트

**1) Kafka 핵심 구성**:

- Broker, Producer, Consumer, Topic, Partition, Consumer Group, Replication

**2) 파티션 vs 컨슈머 비율**:

|구성|효과|
|---|---|
|파티션 > 컨슈머|자연 throttling, DB 부하 분산|
|파티션 = 컨슈머|처리량 극대화 (이상적)|
|파티션 < 컨슈머|일부 idle, 장애 시 즉시 대체|

**3) Outbox 흐름** (`c48d9e0`):

```
Auto 이벤트:
  BEFORE_COMMIT: Outbox 테이블 저장
  AFTER_COMMIT (@Async): Kafka 발행
Manual 이벤트:
  즉시 Outbox 저장 + Kafka 발행
Consumer 처리 완료 시:
  eventId로 Outbox 삭제
```

**4) 후기 수정** (`bf49124`, 6/2):

- "Outbox 수동 제거, @Transactional 없는 로직 바로 카프카 메시지 발행하게끔"
- → 모든 케이스에 Outbox 적용 X. 트랜잭션 있는 것만 Outbox.

**5) 쿠폰 발급 카프카 직렬 처리** (`54c8e37`):

- 선착순 = 순서 보장 필요
- 단일 파티션으로 직렬 처리
- → STEP05 비관적 락 + STEP06 분산 락의 "공정성 한계" 극복

**6) 인기상품 5번째 진화** (`83bd6c6`):

1. DB 집계 (STEP03~STEP04)
2. 배치 프로세스 (STEP05 — `abd2b34`)
3. Redis 캐시 (STEP06 — `cf10b37`)
4. Redis Sorted Set (STEP06 — `82ee0f0`)
5. **실시간 이벤트** (STEP08 — `83bd6c6`) — 주문 완료 이벤트로 즉시 score 갱신

**7) DLQ (Dead Letter Queue)**:

- 실패 메시지 별도 토픽
- Spring Kafka는 `-dlt` 접미사로 자동 생성

#### 📝 STEP08 산출물

- [ ] `docs/study/01.Kafka.md` — 시작 시점 학습 정리
- [ ] `docs/report/06.KafkaDesignArchitectureReport.md`

#### 🚦 STEP08 신호등 변경

- 🟢 추가: KafkaTemplate, Outbox 패턴
- 🔴(제거): 인기상품 배치 스케줄러
- 🔴 여전히 금지: 없음 (모든 도구 해금)

---

### STEP09 - 부하테스트 ⏳

#### 📅 레퍼런스 흐름 (2025-06-05 ~ 06-06, 약 16 커밋)

```
196a608  [CHORE] 도커 및 어플리케이션 부하 테스트 환경 구축
                  ← Prometheus, Grafana
e2af7eb  [FEAT] 주문 상세 조회 API 구현
d7e51e8  [FEAT] 상품 조회 커서 페이징 적용
90bbb45  [REFACTOR] Payment useCoupon 메서드 시그니처 수정
4eb4502  [REFACTOR] 사용자 쿠폰 상세 조회 메서드 시그니처 변경
4ea643e  [FEAT] 부하 테스트 픽스쳐 작성
7d377b0  [FEAT] 부하 테스트 스크립트 작성             ← K6
0aa7f6e  [DOCS] 부하 테스트 대상 선정 및 목적, 시나리오 보고서 작성

a8f756c  [DOCS] 부하 테스트 보고서 이미지 추가
c1ef732  [REFACTOR] 카프카 에러 로깅 추가 작성
43836b4  [REFACTOR] 비동기 에러 로깅 추가 작성
df90f12  [REFACTOR] 카프카 리스너 Concurrency 옵션 추가
146b77f  [REFACTOR] 카프카 리스너 CoreException 핸들링
32f21aa  [DOCS] 부하 테스트 성능 지표 분석, 병목 탐색 및 개선 내용 추가
```

#### 🎯 STEP09 핵심 학습 포인트

**1) 부하 테스트 환경**:

- Spring Actuator + Prometheus + Grafana
- Docker 리소스 제한 (CPU 2 vCPU, 메모리 4GB)
- K6 스크립트

**2) 시나리오**:

- **주문/결제** (현실적 트래픽 분포):
    1. 인기상품 조회 (전체)
    2. 잔액 충전 (전체의 20%)
    3. 잔액 조회
    4. 주문/결제 (충전자의 10%)
    5. 주문 완료 확인
- **선착순 쿠폰 발급** (Peak Test): 단일 시점 동시 폭주

**3) 사전 작업**:

- 주문 상세 조회 API 추가 (시나리오용)
- 상품 조회 커서 페이징 (대량 페이지네이션 대비)
- 부하 테스트 픽스쳐 (대량 더미 데이터 생성기)

**4) 사후 개선**:

- 카프카 에러 로깅
- 비동기 에러 로깅
- 카프카 리스너 Concurrency 옵션
- CoreException 핸들링

#### 📝 STEP09 산출물

- [ ] `docs/report/07.LoadTestReport.md`

---

### 🏁 멀티모듈 분리 (옵션, STEP09 이후)

레퍼런스 흐름 (2025-07-07, 하루 만에):

```
공통:캐시/클라이언트/락/메시지/아웃박스/직렬화/스토리지 모듈 분리
서비스:잔액/쿠폰/주문/결제/상품/유저 모듈 분리
지원:RestDocs 모듈 분리
기존 모놀리식 코드 삭제
```

본인 패키지가 도메인 우선 구조라 멀티모듈 전환이 쉬움.

---

## 📐 작업 표준

### 브랜치 명명

```
feat/step03-user-domain
feat/step03-balance-service        ← 본인 다음 브랜치
feat/step03-coupon-domain
feat/step03-product-domain
test/step05-concurrency
refactor/step07-order-event
```

### 커밋 메시지

|태그|사용처|
|---|---|
|`[DOCS]`|문서|
|`[FEAT]`|새 기능|
|`[TEST]`|테스트|
|`[REFACTOR]`|개선|
|`[FIX]`|버그|
|`[CHORE]`|빌드/설정|
|`[REVERT]`|되돌리기|

### PR 사이클

1. 이슈 #N 확인 → 2. 브랜치 생성 → 3. TDD → 4. 커밋 → 5. PR 셀프리뷰 → 6. 머지

---

## 🚦 단계별 신호등 (v5 git log 기반 정정)

|작업|S03|S04|S05|S06|S07|S08|S09|
|---|---|---|---|---|---|---|---|
|도메인 엔티티 비즈니스 메서드|🟢|🟢|🟢|🟢|🟢|🟢|🟢|
|4-Layer 클린 아키텍처|🟢|🟢|🟢|🟢|🟢|🟢|🟢|
|Repository 인터페이스 + 구현체 분리|🟢|🟢|🟢|🟢|🟢|🟢|🟢|
|단위 테스트|🟢|🟢|🟢|🟢|🟢|🟢|🟢|
|**파사드 클래스 도입**|**🟢**|🟢|🟢|🟢|🔴(제거)|🔴|🔴|
|**`@OneToMany` cascade**|🟢|🟢|🔴(제거)|🔴|🔴|🔴|🔴|
|**`@Transactional` on Facade**|🔴|🟢|🟢|🟢|🔴(제거)|🔴|🔴|
|**`@Transactional` on Service**|🔴|🔴|🟢|🟢|🟢|🟢|🟢|
|통합 테스트 (Testcontainers)|🔴|🟢|🟢|🟢|🟢|🟢|🟢|
|인덱스 적용|🔴|🟢|🟢|🟢|🟢|🟢|🟢|
|`@Enumerated(EnumType.STRING)`|🟢|🟢|🟢|🟢|🟢|🟢|🟢|
|**`@Version` 낙관적 락**|🔴|🔴|🟢|🟢|🟢|🟢|🟢|
|**`@Lock` 비관적 락**|🔴|🔴|🟢|🟢|🟢|🟢|🟢|
|**`Rank` 도메인 분리**|🔴|🔴|🟢|🟢|🟢|🟢|🟢|
|배치 스케줄러|🔴|🔴|🟢|🟢|🟢|🔴(제거)|🔴|
|Redisson 분산 락|🔴|🔴|🔴|🟢|🟢|🟢|🟢|
|`@Cacheable` 캐시|🔴|🔴|🔴|🟢|🟢|🟢|🟢|
|Redis Sorted Set|🔴|🔴|🔴|🟢|🟢|🟢|🟢|
|**`refund` 메서드**|🔴|🔴|🔴|🔴|🟢|🟢|🟢|
|**`BalanceClient` 인터페이스**|🔴|🔴|🔴|🔴|🟢|🟢|🟢|
|**`REFUND` enum**|🔴|🔴|🔴|🔴|🟢|🟢|🟢|
|`ApplicationEventPublisher`|🔴|🔴|🔴|🔴|🟢|🟢|🟢|
|`@TransactionalEventListener`|🔴|🔴|🔴|🔴|🟢|🟢|🟢|
|`@EnableAsync`|🔴|🔴|🔴|🔴|🟢|🟢|🟢|
|Saga (보상 트랜잭션)|🔴|🔴|🔴|🔴|🟢|🟢|🟢|
|Kafka Producer/Consumer|🔴|🔴|🔴|🔴|🔴|🟢|🟢|
|Outbox 패턴|🔴|🔴|🔴|🔴|🔴|🟢|🟢|
|K6 부하테스트|🔴|🔴|🔴|🔴|🔴|🔴|🟢|
|Prometheus + Grafana|🔴|🔴|🔴|🔴|🔴|🔴|🟢|

🟢 가능 / 🔴 금지

---

## 📝 산출물 체크리스트 (git log 검증)

### docs/architecture (STEP01~02 완료 ✅)

|파일|상태|작성 커밋|작성 시점|
|---|---|---|---|
|`01.Requirements.md`|[x]|`c0c7176`|2025-03-31 (STEP01)|
|`02.Milestones.md`|[x]|`c347d56`|2025-03-31 (STEP01)|
|`03-1.SequenceDiagram.md`|[x]|`1e8da44`|2025-04-01 (STEP01)|
|`03-2.StateDiagram.md`|[x]|`6409495`|2025-04-03 (STEP02)|
|`04.ERD.md`|[x]|`ab98dc3`|2025-04-01 (STEP01)|
|`05.ApiDocument.md`|[x]|`2672bb3`|2025-04-01 (STEP01)|
|`06.SpringRestDocs.md`|[ ]|`72acc15`|2025-04-02 (STEP02) ⚠️ **본인 누락**|

> ⚠️ 본인 프로젝트엔 `06.SpringRestDocs.md`가 없음. STEP02에서 Spring REST Docs 적용 후 작성한 가이드 문서. 작성 추천(선택).

### docs/report (그 STEP 끝나면 즉시 작성!)

|파일|작성 시점|핵심 내용|
|---|---|---|
|[ ] `01.DBPerformanceOptimizationReport.md`|**STEP04 끝** (2025-04-17, `6611f80`)|10만 건 더미 + EXPLAIN ANALYZE 인덱스 효과 측정|
|[ ] `02.ConcurrencyReport.md`|**STEP05 끝** (2025-04-24, `b8f4747`)|동시성 실패 테스트 → 락 적용 후 성공 (잔액 낙관적, 쿠폰/재고 비관적)|
|[ ] `03.CacheStrategyArchitectureReport.md`|**STEP06 P2 끝** (2025-05-06, `b77c6f5`)|Read-Through, TTL 49h 설계, K6 측정|
|[ ] `04.RedisDesignArchitectureReport.md`|**STEP06 P3 끝** (2025-05-15, `d684e22`)|Sorted Set, 40일 후 RDB 영속화|
|[ ] `05.MsaEventDrivenArchitectureReport.md`|**STEP07 끝** (2025-05-22, `d14a7e9`)|EDA, 보상 트랜잭션|
|[ ] `06.KafkaDesignArchitectureReport.md`|**STEP08 진행 중** (2025-05-29, `f8ca4e6`)|쿠폰 발급 카프카 직렬 처리|
|[ ] `07.LoadTestReport.md`|**STEP09 시작 → 종료** (2025-06-05~06, `0aa7f6e`→`32f21aa`)|K6 시나리오, 병목 분석/개선|

### docs/study (개념 학습 정리)

|파일|작성 시점|내용|
|---|---|---|
|[ ] `01.Kafka.md`|**STEP08 진행 중** (2025-05-29, `c7e81e2`) — Outbox 구현 같은 날|Kafka 기초 + 핵심 개념|
|[ ] `02.Cache.md`|**2025-07-09** (`f950c1c`) — 멀티모듈 분리 마지막에 정리|캐시 전략, Look Aside, Read Through, Write Through|

> 💡 Study 자료는 **그 STEP 진행하면서 동시 또는 후반에 정리**. Kafka는 8단계 진행과 동시 작성, Cache는 6단계 했던 내용을 한참 후 정리.

### docs/WIL (회고 — 몰아 작성 패턴!)

|파일|본인 위치|작성 시점|다루는 STEP|
|---|---|---|---|
|[x] 2주차|`devlog/이커머스 아키텍처 설계.md` 또는 `docs/WIL/week2/`|2025-04-06 (`3abcc6b`)|STEP01~02 (설계)|
|[ ] 3주차|`docs/WIL/week3/`|**2025-04-25 (`73d868f`) — STEP05 끝**|STEP03~04 (클린 아키텍처)|
|[ ] 4주차|`docs/WIL/week4/`|**2025-06-16 (`1a62e9e`) — STEP09 후 한참 뒤**|STEP04 트랜잭션 + 인덱스|
|[ ] 5주차|`docs/WIL/week5/`|**2025-06-27 (`48e6409`) — STEP09 후**|STEP05~06 동시성 + 분산락|

> ⚠️ **WIL 작성 패턴 — 매우 중요**: 레퍼런스는 **WIL을 STEP 끝마다 즉시 쓰지 않음**. 보고서가 즉시 작성되는 것과 대비.
> 
> - 작성중 메모는 그 시점에 (`6245908` "3주차 WIL 작성중")
> - 최종 완성은 한참 후 몰아서
> 
> **본인 학습/포폴 전략**:
> 
> - **A안 (즉시 작성)**: STEP 끝마다 WIL 작성 — 학습 임팩트 강함, 작업 부담 큼
> - **B안 (몰아 작성, 레퍼런스 그대로)**: STEP 끝에 5분 메모만, 후반에 정리 — 효율적, 망각 위험
> - **C안 (균형)**: 핵심 결정/고민만 그 시점에 단문 메모, STEP09 후 정식 회고 — **추천**

### Balance 도메인 진화 추적표

|항목|STEP03|STEP04|STEP05|STEP06|STEP07|STEP08|
|---|---|---|---|---|---|---|
|`Balance.create`|`(userId, amount)`|동일|**`(userId)`**|동일|동일|동일|
|`@OneToMany`|✅ cascade|✅|**❌ 제거**|❌|❌|❌|
|`BalanceTransaction` FK|`Balance`|`Balance`|**`balanceId` Long**|Long|Long|Long|
|`@Version`|❌|❌|**✅**|✅|✅|✅|
|`@Index`|❌|**✅ 끝에**|✅|✅|✅|✅|
|`@Transactional` (Service)|❌|❌|**✅**|✅|✅|✅|
|`BalanceFacade`|✅|✅|✅|✅|**❌ 삭제**|❌|
|`BalanceClient`|❌|❌|❌|❌|**✅**|✅|
|`refund` 메서드|❌|❌|❌|❌|**✅**|✅|
|`REFUND` enum|❌|❌|❌|❌|**✅**|✅|
|`addChargeTransaction` private|✅|✅|**❌ 제거**|❌|❌|❌|
|`BalanceRepository.saveTransaction`|❌|❌|**✅**|✅|✅|✅|
|`BalanceController` 메서드명|`updateBalance`|`updateBalance`|**`chargeBalance`**|동일|동일|동일|

---

## 📅 학습 자료 작성 타임라인 (시간순 — 레퍼런스 패턴)

|날짜|STEP|코드/문서 작업|
|---|---|---|
|2025-03-31|STEP01|요구사항 + 마일스톤|
|2025-04-01~03|STEP01~02|시퀀스/ERD/API/State 다이어그램, REST Docs|
|2025-04-06|STEP02 끝|**WIL 2주차 작성**|
|2025-04-10~12|**STEP03**|도메인/서비스/파사드 구현 (16 커밋)|
|2025-04-13~17|**STEP04**|인프라 + 통합테스트 + 인덱스|
|2025-04-17|STEP04 끝|**`01.DBPerformanceOptimizationReport.md` 즉시 작성** ✅|
|2025-04-21||"3주차 WIL 작성중" 메모|
|2025-04-22~25|**STEP05**|동시성 (낙관적/비관적 락) + Rank 도메인 분리|
|2025-04-24|STEP05 끝|**`02.ConcurrencyReport.md` 즉시 작성** ✅|
|2025-04-25|STEP05 끝|**WIL 3주차 작성 완료** (STEP03~04 회고)|
|2025-04-30~05-15|**STEP06**|Redis 분산락, 캐시, Sorted Set|
|2025-05-06|STEP06 P2 끝|**`03.CacheStrategyArchitectureReport.md` 즉시 작성** ✅|
|2025-05-15|STEP06 P3 끝|**`04.RedisDesignArchitectureReport.md` 즉시 작성** ✅|
|2025-05-21~22|**STEP07**|EDA + 파사드 제거 + 환불|
|2025-05-22|STEP07 끝|**`05.MsaEventDrivenArchitectureReport.md` 즉시 작성** ✅|
|2025-05-29~06-02|**STEP08**|Kafka + Outbox + 인기상품 실시간 이벤트|
|2025-05-29|STEP08 진행 중|**`01.Kafka.md` (Study) + `06.KafkaDesignArchitectureReport.md` 동시 작성**|
|2025-06-05~06|**STEP09**|부하테스트|
|2025-06-05|STEP09 시작|**`07.LoadTestReport.md` 시나리오 먼저 작성**|
|2025-06-06|STEP09 종료|LoadTestReport에 측정/병목 분석 추가|
|2025-06-16|(몰아 작성)|**WIL 4주차 작성** (STEP04 트랜잭션+인덱스 회고)|
|2025-06-27|(몰아 작성)|**WIL 5주차 작성** (STEP05~06 동시성+분산락 회고)|
|2025-07-07|멀티모듈 분리|공통/서비스 모듈 분리|
|2025-07-09|마지막 정리|**`02.Cache.md` (Study) 정리 작성**|

## 📊 레퍼런스 통계

- 총 커밋: **274**
- STEP별 커밋 수 추정:
    - STEP03: 16 (4/10~12)
    - STEP04: 24 (4/13~18)
    - STEP05: 10 (4/22~25)
    - STEP06: 30+ (4/30~5/16)
    - STEP07: 18 (5/21~22)
    - STEP08: 24 (5/29~6/02)
    - STEP09: 16 (6/5~6)
- 보고서: 7편, WIL: 4편, Study: 2편

**커밋 태그**: `[REFACTOR]` 35% / `[FEAT]` 30% / `[DOCS]` 25% / `[TEST]` 8% / 기타 2%

---

## ✅ 사용법

### 작업 사이클

1. STEP 시작 전 → 해당 STEP 섹션 정독 (Claude가 직접 수정하지 말고 알려주기 + 왜 이런 작업을 하는지 간략히 설명)
2. "절대 하지 말 것" 체크
3. 작업 중 → 신호등 표 확인
4. STEP 완료 → **report 즉시 작성** + 산출물 체크리스트 갱신
5. 막히면 → 레퍼런스 동일 시점 커밋 확인:
    
    ```bash
    git --no-pager show <hash>:<path>git --no-pager log --all --oneline --reverse --date=short --pretty=format:"%h %ad %s" -S "<keyword>" -- "*.java"
    ```
    

### 📚 학습 자료 작성 전략 (레퍼런스 패턴)

**Report (보고서)** — 그 STEP 끝나면 **즉시 작성**:

- 측정 결과/스크린샷이 휘발되기 전에 잡아둠
- AS-IS / 문제 / 해결방안 / TO-BE 구조
- 레퍼런스 패턴: 인덱스 적용한 날 보고서 작성, 동시성 락 적용한 날 보고서 작성

**Study (개념 학습)** — 그 STEP 진행 중 또는 직후:

- Kafka 같이 큰 개념은 진행하면서 동시 정리 (이해도가 가장 깊을 때)
- Cache 같이 여러 STEP에 걸친 개념은 후반에 종합 정리해도 OK

**WIL (회고)** — 한 번에 몰아 작성 OK (레퍼런스 패턴):

- 그 STEP 끝마다 즉시 안 써도 됨
- 작성중 메모는 그 시점에 (`6245908` 같은 "WIL 작성중" 커밋)
- 추천 전략 → **C안 (균형)**: 핵심 결정/고민만 그 시점에 단문 메모, 정식 회고는 STEP09 후 정리

### 🔥 다음 액션 (STEP03 #20)

**사전작업 — User 도메인 (별도 PR)**

User 도메인은 STEP03 시점부터 레퍼런스에 존재 (`f7ac629` 확인). BalanceFacade가 `UserService.getUser()`를 호출하므로 먼저 만들어야 함.

```bash
git checkout -b feat/step03-user-domain
```

작업 (User만, UserCoupon은 이슈 #21에서):

- `user/domain/User.java` (id, userName)
- `user/domain/UserService.java` — `getUser(Long): UserInfo`
- `user/domain/UserRepository.java` 인터페이스
- `user/domain/UserInfo.java` 정적 내부 클래스 DTO
- `user/infrastructure/UserCoreRepository.java` (impl)
- `user/infrastructure/UserJpaRepository.java`
- 사용자 없으면 `IllegalArgumentException("사용자가 존재하지 않습니다.")`

**이슈 #20 — Balance 마무리 16 커밋**

```bash
git checkout main && git pull
git checkout -b feat/step03-balance-service
```

1. `[REFACTOR] Balance 엔티티 @Builder + @OneToMany + create(userId, amount)`
2. `[REFACTOR] BalanceTransaction balance 참조 (@ManyToOne) + ofUse 음수`
3. `[REFACTOR] TransactionType → BalanceTransactionType + description (REFUND 제거)`
4. `[FEAT] BalanceCommand 정적 내부 클래스 (Charge/Use)`
5. `[FEAT] BalanceInfo 정적 내부 클래스 (Balance)`
6. `[REFACTOR] BalanceRepository findOptionalByUserId + Transaction Repository 삭제`
7. `[REFACTOR] BalanceCoreRepository @Component + Transaction CoreRepo 삭제`
8. `[FEAT] BalanceService chargeBalance/useBalance/getBalance (@Transactional 없이)`
9. `[FEAT] BalanceCriteria, BalanceResult Application DTO`
10. `[FEAT] BalanceFacade UserService 검증 + BalanceService 호출`
11. `[REFACTOR] BalanceController Facade 의존 + mock 응답 제거 + updateBalance`
12. `[REFACTOR] BalanceRequest/Response 정적 내부 클래스`
13. `[FEAT] ApiControllerAdvice IllegalArgumentException 핸들러`
14. `[TEST] Balance 도메인 + BalanceService 단위 테스트`
15. `[TEST] BalanceFacade + BalanceController 테스트`

> 이슈 #21 (Coupon), #22 (Product) 진입 시 그 시점 git log 정밀 확인 후 v7로 갱신.

---

> 📌 살아있는 문서. STEP 진행 중 발견하는 것들 추가 갱신.
> 
> **v5 → v6 변경 요약**:
> 
> - 학습 자료(report/WIL/study) 작성 시점을 git log로 정정
> - WIL은 즉시 작성 X, 후반 몰아 작성 패턴 발견
> - architecture에 `06.SpringRestDocs.md` 누락 항목 추가
> - architecture 파일명 `03-1.SequenceDiagram.md` 정정
> - "학습 자료 작성 타임라인" 시간순 표 신규 추가
> 
> v6는 2026-04-30 기준 STEP03 `f7ac629`, STEP04 `d3c68d3`, STEP05 `4316790`, STEP07 `366cfcb`, docs 작성 시점 모두 git log 확정.
> 
> STEP04 진입 시 v7로 갱신 권장 (그 시점 통합 테스트 셋업, ApiControllerAdvice 진화 등 정밀 확인).