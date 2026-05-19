# 📘 mini-commerce 진행 가이드 (v7)

> **v7 = 라이트 DDD 아키텍처 명확화 + GitHub 이슈 39개 매핑 + 진행 상태 반영**
> 
> 현재 나의 레포: `C:\Users\eborder\sungmin\git\e-commerce` 참고 레퍼런스 (멀티모듈 최종 상태): `C:\Users\eborder\sungmin\git\e-commerce-reference` 레퍼런스 URL: https://github.com/discphy/e-commerce 본인 레포: https://github.com/gokid96/e-commerce

---

## 📌 v6 → v7 변경 핵심

| #   | 항목                              | v6              | v7                                                      |
| --- | ------------------------------- | --------------- | ------------------------------------------------------- |
| 1   | **아키텍처 명칭**                     | "클린 레이어드 + DIP" | ✅ **라이트 DDD (Persistence-aware Domain Model)** 명확화      |
| 2   | **GitHub 이슈 매핑**                | 없음              | ✅ 39개 이슈를 STEP별로 매핑                                     |
| 3   | **STEP05 Filter/Interceptor**   | 누락              | ✅ 이슈 #31에서 발견 — 추가 작업 명시                                |
| 4   | **STEP06 보고서 수**                | 2개 (03, 04)     | ✅ **3개**로 정정 — 이슈 #34 "Redis 분산락 동시성 보고서 추가"            |
| 5   | **STEP07 refund/BalanceClient** | 별도 명시           | ⚠️ 이슈 #44에 묻혀있음 → sub-task로 쪼개 추적 권장                    |
| 6   | **STEP08 인기상품 실시간 이벤트**         | 별도 명시           | ⚠️ 별도 이슈 없음, #47/#48 안에 묻힘                              |
| 7   | **User 도메인**                    | "사전작업 권장"       | ✅ **완료** — `feat/step03-user-domain` 머지됨                |
| 8   | **user/infrastructure/jpa/ 구조** | 미언급             | ✅ 레퍼런스 확인 — JPA Repository는 `infrastructure/jpa/` 서브폴더에 |
| 9   | **Balance Step 1 도메인 코드**       | 큰 그림            | ✅ 실제 코드 예시 추가                                           |

---

## 🏛️ 채택된 아키텍처 — 라이트 DDD (Persistence-aware Domain Model)

### 정확한 정체

> **"JPA 어노테이션은 허용하되, setter 없는 캡슐화 + 풍부한 도메인 메서드를 강제하는 실무 표준 절충안"**

부르는 이름들:

- Persistence-aware Domain Model (Vaughn Vernon)
- Pragmatic DDD / Lightweight DDD
- 라이트 DDD / 실용적 DDD (한국 커뮤니티)

전부 같은 패턴을 가리키는 다른 이름.

### 4가지 핵심 원칙

|#|원칙|코드 표현|
|---|---|---|
|1|**불변 우선**|`@Setter` 없음, 필드 모두 `private`|
|2|**생성은 정적 팩토리로**|`Balance.create(userId, amount)` + 검증|
|3|**상태 변경은 의도 있는 메서드로**|`setAmount()` ❌ → `charge()`, `use()` ⭕|
|4|**검증은 도메인 내부에서**|`charge()` 안에서 `amount <= 0` 체크|

### 허용 vs 금지

```java
@Entity                                                    // ✅ 허용 (JPA 필수)
@NoArgsConstructor(access = AccessLevel.PROTECTED)         // ✅ 허용 (JPA 리플렉션)
@Getter                                                    // ✅ 허용 (읽기 전용)
public class Balance {

    @Id @GeneratedValue(...)                               // ✅ 허용 (JPA 필수)
    private Long id;

    private long amount;                                   // ✅ private
    
    @Builder
    private Balance(...) { ... }                           // ✅ 생성자 private
    
    public static Balance create(...) { ... }              // ✅ 정적 팩토리
    
    public void charge(long amount) { ... }                // ✅ 비즈니스 메서드
    
    // ❌ public void setAmount(long amount) — 금지!
}
```

### 순수 DDD/헥사고날과의 차이

|구분|도메인이 JPA를 아나?|비즈니스 로직 위치|클래스 개수|
|---|---|---|---|
|순수 DDD/헥사고날|❌ 모름 (Mapper로 분리)|도메인 객체|많음|
|**라이트 DDD (현재)**|⚠️ **앎**|**도메인 객체**|적음|
|Anemic (피해야 함)|❌ 모름|Service에 흩어짐|적음|

### 레이어 구조

```
interfaces      →  application (Facade)  →  domain (Service/Entity/Repo I/F)  ←  infrastructure (Repo Impl, JPA)
   ↑                       ↑                          ↑                                  ↑
 Controller             Facade                     Service                       CoreRepository
 Request/Response       Criteria/Result            Entity                        JpaRepository
                                                   Command/Info
                                                   Repository (interface)
```

### 패키지/클래스 네이밍 규칙

|레이어|패키지명|클래스|입력 DTO|출력 DTO|
|---|---|---|---|---|
|Presentation|`interfaces`|`XxxController`|`XxxRequest.Charge` (정적 내부)|`XxxResponse.Balance`|
|Application|`application`|`XxxFacade`|`XxxCriteria.Charge`|`XxxResult.Balance`|
|Domain|`domain`|`XxxService`|`XxxCommand.Charge`|`XxxInfo.Balance`|
|Infrastructure|`infrastructure` + `infrastructure/jpa/`|`XxxCoreRepository` (impl) + `XxxJpaRepository`|-|-|

DTO 변환 흐름: `Request.toCriteria(id)` → `Criteria.toCommand()` → Service.

### 본인 프로젝트 패키지 구조

```
com.github.gokid96.e_commerce
├── balance/{interfaces, application, domain, infrastructure/jpa}
├── coupon/, order/, product/   ← interfaces만 채워짐 (mock)
├── user/                       ← 완료 ✅
│   ├── domain/      User, UserInfo, UserRepository, UserService
│   └── infrastructure/
│       ├── jpa/     UserJpaRepository
│       └── UserCoreRepository
├── common/                     ← ApiResponse, ApiControllerAdvice
└── ECommerceApplication.java
```

> ⚠️ **현재 본인 패키지가 `interfaces` 대신 `controller` 사용 중**. 레퍼런스 STEP03 `eae6310`에서 `interfaces`로 변경됨. STEP03 작업 중 같이 정리 가능.

---

## 🏛️ STEP별 아키텍처 진화 (한눈에)

```
STEP03  ▶ [기본 골격]
         interfaces → application(Facade) → domain → infrastructure/jpa
         단방향 의존, DIP 적용, mock 응답 제거

STEP04  ▶ [+ Infra 강화]
         JPA Repository 구현체 채우기, Testcontainers 통합테스트
         + Facade에 @Transactional, + 인덱스 적용
         + Repository 인터페이스 어노테이션 @Repository → @Component

STEP05  ▶ [+ 동시성]
         낙관적 락(@Version) / 비관적 락(@Lock) 도입
         ⚠️ Balance 큰 리팩토링: @OneToMany 끊고 ID 참조로 후퇴
         + Service에 @Transactional 추가
         + Rank 도메인 분리 (인기상품 배치 스케줄러)
         + Filter/Interceptor 도입 (이슈 #31, v7 추가)

STEP06  ▶ [+ Redis]
         Redisson 분산락 (AOP 패턴, 11개 신규 파일)
         @Cacheable Read-Through 캐시
         인기상품 Redis Sorted Set, 40일 후 RDB 영속화
         쿠폰 발급 요청/완료 분리

STEP07  ▶ [- Facade, + EDA]
         Facade 클래스 통째로 삭제 ❗
         ApplicationEventPublisher + @TransactionalEventListener
         + BalanceClient 인터페이스 (도메인 격리, 헥사고날 한 발 근접)
         + refund / REFUND enum / Saga 보상 트랜잭션
         + @EnableAsync

STEP08  ▶ [+ Kafka]
         KafkaTemplate, Outbox 패턴
         인기상품 실시간 이벤트 전환 (배치 스케줄러 제거 — 5번째 진화)
         쿠폰 발급 카프카 직렬 처리 (단일 파티션)

STEP09  ▶ [+ 관측/부하]
         Prometheus + Grafana, K6 시나리오, 병목 분석/개선
         카프카 에러 로깅, Concurrency 옵션

(옵션)  ▶ [멀티모듈 분리]
         common/service/support 모듈 분리 (헥사고날에 근접)
```

---

## 🎯 핵심 원칙

1. **STEP을 건너뛰지 마라** — 락 없이, 캐시 없이, 이벤트 없이 시작
2. **문서가 코드보다 먼저** — 보고서 작성 후 코드 변경
3. **`[REFACTOR]`는 죄가 아니다** — 레퍼런스 274 커밋 중 ~35%
4. **같은 코드를 여러 번 갈아엎는다** — 인기상품 5번 진화, Balance도 STEP04→05 큰 리팩토링
5. **학습 → 적용 → 회고 사이클** — WIL/Study/Report 누적
6. **라이트 DDD 4원칙 준수** — setter 없음, 정적 팩토리, 의도 있는 메서드, 도메인 내부 검증

---

## 📋 GitHub 이슈 매핑 (39개)

### STEP01~02 (완료, 22 closed) ✅

설계 문서 및 REST Docs.

### STEP03 - 도메인 구현(잔액/쿠폰/상품) — 4 issues 🎯

|#|제목|라벨|가이드 매핑|
|---|---|---|---|
|#3|[STEP03] 도메인 구현 - 잔액/쿠폰/상품|enhancement|부모 트래커|
|**#20**|잔액 비즈니스 로직 구현 및 단위 테스트|Feature|**🎯 현재 작업 중**|
|#21|쿠폰 비즈니스 로직 구현 및 단위 테스트|Feature||
|#22|상품 비즈니스 로직 구현 및 단위 테스트|Feature||

### STEP04 - 도메인 구현(주문/결제) — 6 issues

|#|제목|라벨|가이드 매핑|
|---|---|---|---|
|#4|[STEP04] 도메인 구현 - 주문/결제|enhancement|부모|
|#23|주문/결제 비즈니스 로직 구현 및 단위 테스트|Feature|OrderFacade|
|#24|인프라 레이어 구현체 작성|Feature|각 도메인 JPA Repository|
|#25|기능별 통합 테스트 작성|Test|Testcontainers|
|#26|**주요 기능별 동시성 실패 테스트** 작성|Test|⭐ STEP04 끝 미리 작성|
|#27|병목 예상 쿼리 분석 및 최적화 보고서 작성|Document|`01.DBPerformanceOptimizationReport.md`|

### STEP05 - 동시성 — 6 issues

|#|제목|라벨|가이드 매핑|
|---|---|---|---|
|#5|[STEP05] 동시성 이슈 해결|enhancement|부모|
|#28|주요 기능별 동시성 테스트 작성|Test|동시성 테스트 정식|
|#29|주요 기능 동시성 이슈 식별 및 해결|Feature|낙관적/비관적 락|
|#30|동시성 이슈 분석 및 해결 보고서 작성|Document|`02.ConcurrencyReport.md`|
|**#31**|**Filter/Interceptor/Scheduler 부가 로직 구현**|Feature|⚠️ v7 신규 인식 — Rank 도메인 + 배치 + Filter/Interceptor|
|#32|모든 API 정상 작동 및 가용성 확보|Feature|RestAssured E2E|

### STEP06 - DB 성능 + 캐시 — 8 issues (가장 큼)

|#|제목|라벨|가이드 매핑|
|---|---|---|---|
|#6|[STEP06] DB 성능 최적화/Redis|enhancement|부모|
|#33|Redis 기반 분산락 구현 및 적용|Feature|Phase 1 — Redisson|
|**#34**|**Redis 분산락 동시성 보고서 추가**|Document|⚠️ v7 신규 — `02.ConcurrencyReport.md` 부록 또는 별도|
|#35|Redis 기반 캐싱 전략 설정 및 적용|Feature|Phase 2 — @Cacheable|
|#36|캐싱 전략 및 성능 개선 보고서 작성|Document|`03.CacheStrategyArchitectureReport.md`|
|#37|인기상품 Redis 기반 설계 및 구현|Refactor|Phase 3 — Sorted Set|
|#38|선착순 쿠폰발급 Redis 기반 설계 및 구현|Refactor|쿠폰 요청/완료 분리|
|#39|Redis 디자인 설계 보고서 작성|Document|`04.RedisDesignArchitectureReport.md`|

### STEP07 - EDA — 4 issues

|#|제목|라벨|가이드 매핑|
|---|---|---|---|
|#7|[STEP07] MSA 기반 이벤트 아키텍처|enhancement|부모|
|#43|주문/결제 완료 시 이벤트 기반 외부 데이터 플랫폼 전송|Refactor|Phase 1 ApplicationEventPublisher|
|**#44**|**파사드 클래스 제거 및 이벤트 기반 도메인 서비스 구현**|Refactor|⚠️ refund + BalanceClient + REFUND enum이 묻혀있음|
|#45|MSA 기반 이벤트 아키텍처 설계 문서 작성|Document|`05.MsaEventDrivenArchitectureReport.md`|

### STEP08 - MSA 분리 — 6 issues

|#|제목|라벨|가이드 매핑|
|---|---|---|---|
|#8|[STEP08] 카프카 활용|enhancement|부모|
|#46|카프카 기초 및 핵심 개념 문서 작성|Document|`docs/study/01.Kafka.md`|
|#47|주문 완료 시 데이터 플랫폼으로 카프카 메시지 발행|Refactor|주문 이벤트 카프카 발행|
|#48|대용량 트래픽 프로세스 카프카 활용 구현|Refactor|쿠폰 발급 카프카 + ⚠️ 인기상품 실시간 이벤트 묻힘|
|#49|Outbox 패턴 적용|Refactor|Outbox 구현|
|#50|카프카 기반 설계 문서 작성|Document|`06.KafkaDesignArchitectureReport.md`|

### STEP09 - 부하테스트 — 5 issues

|#|제목|라벨|가이드 매핑|
|---|---|---|---|
|#9|[STEP09] 부하테스트 및 장애대응|enhancement|부모|
|#51|부하테스트 대상 선정 및 시나리오 계획 문서 작성|Document|시나리오 문서|
|#52|부하테스트 스크립트 작성|Test|K6|
|#53|부하테스트 결과 기반 병목 탐색 및 개선|Refactor|후속 개선|
|#54|성능 테스트 및 장애대응 보고서 작성|Document|`07.LoadTestReport.md`|

---

## 🗓️ STEP03 - 도메인 구현 (잔액/쿠폰/상품) 🎯

### 사전작업 ✅ User 도메인 완료 (`feat/step03-user-domain` 머지됨)

User 도메인은 BalanceFacade의 의존이므로 별도 PR로 먼저 머지함.

### 이슈 #20: 잔액 비즈니스 로직 구현 — `feat/step03-balance-service` 브랜치

#### 🚫 STEP03 절대 하지 말 것

- ❌ `@Version` 낙관적 락 → STEP05
- ❌ `@Lock(PESSIMISTIC_WRITE)` → STEP05
- ❌ `@Transactional` on Service → STEP05 (낙관적 락과 함께)
- ❌ `Balance.refund()` → STEP07
- ❌ `BalanceTransaction.ofRefund()` → STEP07
- ❌ `BalanceTransactionType.REFUND` → STEP07
- ❌ `@Cacheable` → STEP06 P2
- ❌ `Redisson` → STEP06
- ❌ `ApplicationEventPublisher` → STEP07
- ❌ `KafkaTemplate` → STEP08
- ❌ `BalanceClient` 인터페이스 → STEP07 (파사드 제거 시)
- ❌ `BaseEntity` / `createdAt` 필드 → 레퍼런스에 없음

#### ✅ 작업 단계 (총 8 Step)

**Step 1: Balance 도메인 모델 3종** ⬅️ 현재

- `BalanceTransactionType.java` — enum (CHARGE, USE만)
- `BalanceTransaction.java` — @ManyToOne Balance, 가계부 스타일
- `Balance.java` — @OneToMany cascade, charge/use 비즈니스 메서드

**Step 2: Application/Domain DTO**

- `BalanceCommand.java` — `Charge`, `Use` 정적 내부 클래스
- `BalanceInfo.java` — `Balance` 정적 내부 클래스

**Step 3: Repository 3종**

- `balance/domain/BalanceRepository.java` — `@Repository` interface
- `balance/infrastructure/jpa/BalanceJpaRepository.java`
- `balance/infrastructure/BalanceCoreRepository.java` — `@Component` impl

**Step 4: BalanceService** — `@Transactional` 없이

- `chargeBalance(Charge)` — Optional.ifPresentOrElse 패턴
- `useBalance(Use)`
- `getBalance(userId)` — 없으면 0

**Step 5: Facade Layer**

- `BalanceCriteria.java` — `Charge` 정적 내부 + `toCommand()`
- `BalanceResult.java` — `Balance` 정적 내부 + `of()`
- `BalanceFacade.java` — UserService 검증 → BalanceService 호출

**Step 6: Controller 갱신**

- `BalanceController` — Facade 의존, mock 응답 제거
- `BalanceRequest.java` — `Charge` 정적 내부 + `toCriteria(userId)`
- `BalanceResponse.java` — `Balance` 정적 내부 + `of(Result)`
- (선택) `controller/` → `interfaces/` 패키지 리네임

**Step 7: ApiControllerAdvice**

- `IllegalArgumentException` 핸들러 추가 (400)

**Step 8: 테스트 4종**

- `BalanceTest.java` — 도메인 단위 7 케이스
- `BalanceServiceTest.java` — Mockito 9 케이스
- `BalanceFacadeTest.java` — InOrder 검증
- `BalanceControllerTest.java` — `@MockBean BalanceFacade`로 갱신

### 이슈 #21: 쿠폰, 이슈 #22: 상품

같은 패턴 반복. 진입 시 git log 정밀 확인.

### 산출물 (STEP03 끝)

- WIL 3주차 작성 (선택, C안 추천 — 핵심 메모만)

---

## ⏳ STEP04 - 도메인 구현 (주문/결제)

### 핵심 학습 포인트

1. **Testcontainers 통합 테스트** — `@SpringBootTest` + MySQL 컨테이너
2. **JPA Repository 채우기** — 각 도메인 `XxxJpaRepository extends JpaRepository`
3. **Repository 어노테이션 변경** — 인터페이스 `@Repository` → `@Component`
4. **OrderFacade — 진짜 Facade의 진가** — `@Transactional` Facade에 적용 (Service 아님!)
5. **`@Enumerated(EnumType.STRING)`** — 모든 enum 필드
6. **인덱스 적용** — **보고서 먼저(`6611f80`) → 코드(`f66c201`)** 순서. 10만 건 + EXPLAIN ANALYZE

### 산출물

- `docs/report/01.DBPerformanceOptimizationReport.md` (STEP04 끝, 이슈 #27)
- `5671ca9` 동시성 실패 테스트 미리 작성 (이슈 #26, STEP05 진입 준비)

### 🚦 신호등 변경

- 🟢 추가: 인덱스, `@Transactional` on Facade, Testcontainers
- 🔴 여전히 금지: `@Version`, `@Lock`, refund, BalanceClient, Redis, 캐시, 이벤트, Kafka

---

## ⏳ STEP05 - 동시성 제어

### 핵심 학습 포인트

**1) 락 전략**

|자원|전략|이유|
|---|---|---|
|잔액|낙관적 락 (`@Version`)|동일 사용자 중복 충전 = 의도치 않음. 하나만 처리|
|쿠폰|비관적 락 (`@Lock(PESSIMISTIC_WRITE)`)|선착순 모두 처리되어야 함|
|재고|비관적 락|동시 차감 시 음수 방지|

**2) Balance 큰 리팩토링** (`4316790`)

```java
// AS-IS (STEP04):
@OneToMany(mappedBy="balance", cascade=CascadeType.ALL)
private List<BalanceTransaction> balanceTransactions;
public static Balance create(Long userId, Long amount) { ... }

// TO-BE (STEP05):
@Version private Integer version;
// @OneToMany 제거! balanceTransactions 필드 제거!
public static Balance create(Long userId) { ... }  // amount=0
```

→ 이유: 낙관적 락 적용 시 cascade가 영속성 컨텍스트에서 충돌. 연관관계 끊고 ID 참조로 후퇴 + `saveTransaction()` 명시 호출.

**3) BalanceService에 `@Transactional` 추가**

**4) Rank 도메인 분리** (`abd2b34`) — 인기상품 배치 프로세스 전환

**5) Filter/Interceptor 도입** (이슈 #31, v7 신규) — 부가 로직 처리

**6) API 메서드명 변경** — Controller `updateBalance` → `chargeBalance`

### 산출물

- `docs/report/02.ConcurrencyReport.md` (이슈 #30, 즉시 작성)

### 🚦 신호등 변경

- 🟢 추가: `@Version`, `@Lock`, `@Transactional` on Service, Rank 도메인, 배치 스케줄러, Filter/Interceptor
- 🔴 여전히 금지: refund, BalanceClient, Redis, 캐시, 이벤트, Kafka

---

## ⏳ STEP06 - DB 성능 + Redis (3 Phase)

### Phase 1: Redis + 분산 락

**분산락 AOP 패턴** (11개 신규 파일):

- `support/lock/` — DistributedLock, Aspect, KeyGenerator, Strategy, LockType, LockTemplate, LockCallback
- `infrastructure/lock/` — PubSubLockTemplate, SpinLockTemplate

**분산 락 + 트랜잭션 순서 (매우 중요)**:

```
1. 분산 락 획득 (트랜잭션 밖)
2. 트랜잭션 시작
3. 비즈니스 로직
4. 트랜잭션 커밋
5. 분산 락 해제 (TransactionSynchronizationManager)
```

구현: `@Order(Ordered.HIGHEST_PRECEDENCE)` 락 AOP가 트랜잭션보다 먼저 실행

### Phase 2: 인기상품 캐시

- `@Cacheable` Read-Through
- TTL **49시간** (배치 시각과 만료 겹침 방지 + hotfix 여유)

### Phase 3: Redis 자료구조 전환

- 인기상품 Sorted Set (score = 판매량)
- 40일 후 RDB 영속화 스케줄러
- 쿠폰 발급 요청/완료 분리

### 산출물 (보고서 3개! v6에서 2개로 표기했던 거 정정)

- `docs/study/02.Cache.md` (시작 시점)
- `docs/report/02.ConcurrencyReport.md` 부록 또는 별도 (이슈 #34, **v7 신규**)
- `docs/report/03.CacheStrategyArchitectureReport.md` (Phase 2 끝, 이슈 #36)
- `docs/report/04.RedisDesignArchitectureReport.md` (Phase 3 끝, 이슈 #39)

### 🚦 신호등 변경

- 🟢 추가: Redis 분산 락, `@Cacheable`, Redis Sorted Set
- 🔴 여전히 금지: refund, BalanceClient, 이벤트, Kafka

---

## ⏳ STEP07 - EDA (이벤트 기반 + 파사드 제거)

### Phase 1: 외부 데이터 플랫폼 이벤트화 (이슈 #43)

- ApplicationEventPublisher
- `@TransactionalEventListener(phase = AFTER_COMMIT)`

### Phase 2: 도메인 이벤트 + 파사드 제거 (이슈 #44 — ⚠️ 광범위)

이슈 #44 안에 실제로는 **3가지 큰 작업**이 묻혀있음:

1. **파사드 제거** — `BalanceFacade`, `BalanceCriteria`, `BalanceResult` 삭제
2. **`BalanceClient` 인터페이스** 신규 (`domain/`)
    - `BalanceApiClient` impl (`infrastructure/client/`)
    - `BalanceService`가 `BalanceClient` 의존
3. **`refund` 메서드 구현** (+162줄)
    - `Balance.refund(amount)` — 음수/0 검증
    - `BalanceCommand.Refund` 추가
    - `BalanceService.refundBalance()` 추가
    - `BalanceTransaction.ofRefund()` (+amount)
    - `BalanceTransactionType.REFUND("환불")` enum 추가

**Saga 보상 트랜잭션**:

- 결제 실패 → 잔액 환불 (`Balance.refund`)
- 결제 실패 → 쿠폰 사용 취소
- 결제 실패 → 재고 복구 (`Stock.restore`)

**기타**:

- `@EnableAsync`
- 도메인별 Event 클래스 (Balance/Coupon/Payment/Stock/Message/Rank/Order)

### 산출물

- `docs/report/05.MsaEventDrivenArchitectureReport.md` (이슈 #45)

### 🚦 신호등 변경

- 🟢 추가: refund, BalanceClient, ApplicationEvent, @TransactionalEventListener, @EnableAsync
- 🔴(제거): 파사드 클래스
- 🔴 여전히 금지: Kafka, Outbox

> 💡 **권장**: 이슈 #44를 작업 시 GitHub에 sub-task 댓글로 3가지(파사드 제거 / BalanceClient / refund) 분리 추적

---

## ⏳ STEP08 - Kafka + Outbox

### 핵심 작업

- Kafka 설정 (이슈 #47): KafkaTemplate, Topic/ConsumerGroup
- Outbox 패턴 (이슈 #49): `c48d9e0`
- 쿠폰 발급 카프카 직렬 처리 (이슈 #48, 단일 파티션)
- **인기상품 실시간 이벤트 전환** (⚠️ 별도 이슈 없음 — #47 또는 #48에 묻힘, **5번째 진화**)

### Outbox 흐름

```
Auto 이벤트:
  BEFORE_COMMIT: Outbox 테이블 저장
  AFTER_COMMIT (@Async): Kafka 발행
Manual 이벤트 (트랜잭션 없는 로직):
  즉시 Kafka 발행
Consumer 처리 완료 시: eventId로 Outbox 삭제
```

### 파티션 vs 컨슈머 비율

|구성|효과|
|---|---|
|파티션 > 컨슈머|자연 throttling, DB 부하 분산|
|파티션 = 컨슈머|처리량 극대화 (이상적)|
|파티션 < 컨슈머|일부 idle, 장애 시 즉시 대체|

### 인기상품 5번째 진화 (전체 5번 변천사)

```
1. DB 집계 (STEP03~04)
2. 배치 프로세스 (STEP05 abd2b34)
3. Redis 캐시 (STEP06 cf10b37)
4. Redis Sorted Set (STEP06 82ee0f0)
5. 실시간 이벤트 (STEP08 83bd6c6)
```

### 산출물

- `docs/study/01.Kafka.md` (이슈 #46, STEP08 진행 중 동시 작성)
- `docs/report/06.KafkaDesignArchitectureReport.md` (이슈 #50)

### 🚦 신호등 변경

- 🟢 추가: KafkaTemplate, Outbox 패턴
- 🔴(제거): 인기상품 배치 스케줄러

---

## ⏳ STEP09 - 부하테스트

### 환경 (이슈 #51, #52)

- Spring Actuator + Prometheus + Grafana
- Docker 리소스 제한 (CPU 2 vCPU, 메모리 4GB)
- K6 스크립트

### 시나리오

- **주문/결제** (현실적 트래픽):
    1. 인기상품 조회 (전체)
    2. 잔액 충전 (전체의 20%)
    3. 잔액 조회
    4. 주문/결제 (충전자의 10%)
    5. 주문 완료 확인
- **선착순 쿠폰 발급** (Peak Test): 단일 시점 동시 폭주

### 사후 개선 (이슈 #53)

- 카프카 에러 로깅
- 비동기 에러 로깅
- 카프카 리스너 Concurrency 옵션
- CoreException 핸들링

### 산출물

- `docs/report/07.LoadTestReport.md` (이슈 #54)

---

## 🏁 (옵션) 멀티모듈 분리

STEP09 이후, 본인 패키지가 도메인 우선 구조라 전환 쉬움:

```
공통: 캐시/클라이언트/락/메시지/아웃박스/직렬화/스토리지
서비스: 잔액/쿠폰/주문/결제/상품/유저
지원: RestDocs
```

순수 헥사고날 학습은 이 시점에 BalanceEntity 분리 리팩토링으로 가능.

---

## 📐 작업 표준

### 브랜치 명명

```
feat/step03-user-domain        ← 완료 ✅
feat/step03-balance-service    ← 현재 🎯
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

## 🚦 단계별 신호등

|작업|S03|S04|S05|S06|S07|S08|S09|
|---|---|---|---|---|---|---|---|
|도메인 엔티티 비즈니스 메서드|🟢|🟢|🟢|🟢|🟢|🟢|🟢|
|4-Layer 클린 아키텍처|🟢|🟢|🟢|🟢|🟢|🟢|🟢|
|Repository I/F + 구현체 분리|🟢|🟢|🟢|🟢|🟢|🟢|🟢|
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
|**Filter/Interceptor (v7 추가)**|🔴|🔴|🟢|🟢|🟢|🟢|🟢|
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

## 📝 산출물 체크리스트

### docs/architecture (완료)

|파일|상태|
|---|---|
|`01.Requirements.md`|✅|
|`02.Milestones.md`|✅|
|`03.SequenceDiagram.md`|✅|
|`03-2.StateDiagram.md`|✅|
|`04.ERD.md`|✅|
|`05.ApiDocument.md`|✅|
|`06.SpringRestDocs.md`|[ ] (선택, STEP02 종료 시 작성 가이드)|

### docs/report (그 STEP 끝나면 즉시 작성)

|파일|작성 시점|이슈|
|---|---|---|
|`01.DBPerformanceOptimizationReport.md`|STEP04 끝|#27|
|`02.ConcurrencyReport.md`|STEP05 끝|#30|
|**`02.ConcurrencyReport.md` 부록 또는 분산락 보고서**|STEP06 P1 끝|**#34 (v7 추가)**|
|`03.CacheStrategyArchitectureReport.md`|STEP06 P2 끝|#36|
|`04.RedisDesignArchitectureReport.md`|STEP06 P3 끝|#39|
|`05.MsaEventDrivenArchitectureReport.md`|STEP07 끝|#45|
|`06.KafkaDesignArchitectureReport.md`|STEP08 진행 중|#50|
|`07.LoadTestReport.md`|STEP09 시작 → 종료|#51, #54|

### docs/study

|파일|작성 시점|이슈|
|---|---|---|
|`01.Kafka.md`|STEP08 진행 중|#46|
|`02.Cache.md`|멀티모듈 분리 시점 (선택)|—|

### docs/WIL (몰아 작성 OK)

|파일|작성 시점|다루는 STEP|
|---|---|---|
|WIL 2주차|완료 ✅|STEP01~02|
|WIL 3주차|STEP05 끝 또는 STEP09 후|STEP03~04|
|WIL 4주차|STEP09 후|STEP04 트랜잭션 + 인덱스|
|WIL 5주차|STEP09 후|STEP05~06 동시성 + 분산락|

> 💡 **WIL 전략 — C안 (균형) 추천**: 핵심 결정/고민만 그 시점에 단문 메모, 정식 회고는 STEP09 후 정리

---

## 📊 Balance 도메인 진화 추적표

|항목|STEP03|STEP04|STEP05|STEP06|STEP07|STEP08|
|---|---|---|---|---|---|---|
|`Balance.create`|`(userId, amount)`|동일|**`(userId)`**|동일|동일|동일|
|`@OneToMany`|✅ cascade|✅|**❌ 제거**|❌|❌|❌|
|`BalanceTransaction` FK|`Balance`|`Balance`|**`balanceId` Long**|Long|Long|Long|
|`@Version`|❌|❌|**✅**|✅|✅|✅|
|`@Index`|❌|**✅**|✅|✅|✅|✅|
|`@Transactional` (Service)|❌|❌|**✅**|✅|✅|✅|
|`BalanceFacade`|✅|✅|✅|✅|**❌ 삭제**|❌|
|`BalanceClient`|❌|❌|❌|❌|**✅**|✅|
|`refund` 메서드|❌|❌|❌|❌|**✅**|✅|
|`REFUND` enum|❌|❌|❌|❌|**✅**|✅|
|`BalanceRepository.saveTransaction`|❌|❌|**✅**|✅|✅|✅|
|`BalanceController` 메서드명|`updateBalance`|`updateBalance`|**`chargeBalance`**|동일|동일|동일|

---

## 🔥 다음 액션

### 🎯 현재: STEP03 이슈 #20 — `feat/step03-balance-service` Step 1

**Step 1 (Balance 도메인 3종)**:

- `BalanceTransactionType.java`
- `BalanceTransaction.java`
- `Balance.java`

코드 가이드는 채팅에서 제공됨. Step 1 완료 시 Step 2(Command/Info DTO)로 진행.

### 📋 작업 사이클

1. STEP 시작 전 → 해당 STEP 섹션 정독
2. "절대 하지 말 것" 체크
3. 작업 중 → 신호등 표 확인
4. STEP 완료 → **report 즉시 작성** + 산출물 체크리스트 갱신
5. 막히면 → 레퍼런스 동일 시점 커밋 확인:
    
    ```bash
    git --no-pager show <hash>:<path>git --no-pager log --all --oneline --reverse --date=short \  --pretty=format:"%h %ad %s" -S "<keyword>" -- "*.java"
    ```
    

### 📚 학습 자료 작성 전략 (레퍼런스 패턴)

- **Report**: 그 STEP 끝나면 **즉시 작성** — 측정 결과 휘발 전에
- **Study**: 그 STEP 진행 중 또는 직후 — Kafka는 동시, Cache는 후반
- **WIL**: 한 번에 몰아 작성 OK — 그 시점엔 핵심 메모만

---

> 📌 살아있는 문서. STEP 진행 중 발견하는 것들 추가 갱신.
> 
> **v6 → v7 변경 요약**:
> 
> - 아키텍처 명칭 **"라이트 DDD (Persistence-aware Domain Model)"** 명확화
> - GitHub 이슈 39개를 STEP별로 매핑
> - STEP05에 **Filter/Interceptor** 추가 (이슈 #31)
> - STEP06 보고서 **3개**로 정정 (이슈 #34 추가)
> - STEP07 이슈 #44에 묻힌 작업 3가지(파사드 제거 / BalanceClient / refund) 명시
> - STEP08 인기상품 실시간 이벤트가 별도 이슈 없이 #47/#48에 묻힘 명시
> - **User 도메인 완료** 상태 반영
> - `user/infrastructure/jpa/` 구조 정정 (레퍼런스 확인)
> - Balance Step 1 코드 가이드 추가
> 
> STEP03 진행 중 발견 시 v8로 갱신 권장.