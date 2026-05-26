# mini-commerce 진행 가이드 (v8)

> 본인 레포: https://github.com/gokid96/e-commerce 레퍼런스 (멀티모듈 최종): https://github.com/discphy/e-commerce 로컬 경로: `C:\Users\eborder\sungmin\git\e-commerce` 레퍼런스 로컬: `C:\Users\eborder\sungmin\git\e-commerce-reference`
> 
> v8 = STEP03 이슈 #20 실제 진행 결과 반영. 작업 방식, 트러블슈팅, 결정사항 정리.

---

## 1. 채택 아키텍처: 라이트 DDD (Persistence-aware Domain Model)

### 정체

JPA 어노테이션은 허용하되, setter 없는 캡슐화 + 풍부한 도메인 메서드를 강제하는 실무 표준 절충안.

부르는 이름들:

- Persistence-aware Domain Model (Vaughn Vernon)
- Pragmatic DDD / Lightweight DDD
- 라이트 DDD / 실용적 DDD (한국 커뮤니티)

### 4가지 원칙

|#|원칙|코드 표현|
|---|---|---|
|1|불변 우선|`@Setter` 없음, 필드 `private`|
|2|생성은 정적 팩토리|`Balance.create(userId, amount)` + 검증|
|3|상태 변경은 의도 있는 메서드|`setAmount()` 금지 → `charge()`, `use()`|
|4|검증은 도메인 내부|`charge()` 안에서 `amount <= 0` 체크|

### 허용/금지 예시

```java
@Entity                                                    // 허용 (JPA 필수)
@NoArgsConstructor(access = AccessLevel.PROTECTED)         // 허용 (JPA 리플렉션)
@Getter                                                    // 허용 (읽기 전용)
public class Balance {

    @Id @GeneratedValue(...)                               // 허용
    private Long id;

    private long amount;                                   // private

    @Builder
    private Balance(...) { ... }                           // 생성자 private

    public static Balance create(...) { ... }              // 정적 팩토리
    public void charge(long amount) { ... }                // 비즈니스 메서드

    // public void setAmount(long amount) — 금지
}
```

### 순수 DDD/헥사고날과의 차이

|구분|도메인이 JPA 아나?|비즈니스 로직 위치|클래스 개수|
|---|---|---|---|
|순수 DDD/헥사고날|모름 (Mapper 분리)|도메인 객체|많음|
|라이트 DDD (현재)|앎|도메인 객체|적음|
|Anemic (피해야 함)|모름|Service에 흩어짐|적음|

---

## 2. 레이어 구조

```
interfaces  →  application(Facade)  →  domain(Service/Entity/Repo I/F)  ←  infrastructure(Repo Impl, JPA)
   ↑                  ↑                          ↑                                ↑
Controller          Facade                     Service                     CoreRepository
Request/Response    Criteria/Result            Entity                      JpaRepository (jpa/ 하위)
                                               Command/Info
                                               Repository (interface)
```

### 네이밍 규칙

|레이어|패키지|클래스|입력 DTO|출력 DTO|
|---|---|---|---|---|
|Presentation|`interfaces`|`XxxController`|`XxxRequest.Charge`|`XxxResponse.Balance`|
|Application|`application`|`XxxFacade`|`XxxCriteria.Charge`|`XxxResult.Balance`|
|Domain|`domain`|`XxxService`|`XxxCommand.Charge`|`XxxInfo.Balance`|
|Infrastructure|`infrastructure` + `infrastructure/jpa/`|`XxxCoreRepository` + `XxxJpaRepository`|-|-|

### DTO 변환 흐름

```
HTTP JSON
  ↓
BalanceRequest.Charge     (interfaces)
  ↓ toCriteria(userId)
BalanceCriteria.Charge    (application)
  ↓ toCommand()
BalanceCommand.Charge     (domain)
  ↓
[도메인 처리]
  ↓
BalanceInfo.Balance       (domain)
  ↓ of()
BalanceResult.Balance     (application)
  ↓ of()
BalanceResponse.Balance   (interfaces)
  ↓
HTTP JSON
```

---

## 3. STEP별 아키텍처 진화

```
STEP03  [기본 골격]
  interfaces → application(Facade) → domain → infrastructure/jpa
  단방향 의존, DIP 적용, mock 응답 제거

STEP04  [+ Infra 강화]
  JPA Repository 구현체 채우기, Testcontainers 통합 테스트
  Facade에 @Transactional, 인덱스 적용
  Repository 인터페이스 어노테이션 @Repository → @Component

STEP05  [+ 동시성]
  낙관적 락(@Version) / 비관적 락(@Lock) 도입
  Balance 큰 리팩토링: @OneToMany 끊고 ID 참조로 후퇴
  Service에 @Transactional 추가
  Rank 도메인 분리 (인기상품 배치 스케줄러)
  Filter/Interceptor 도입 (이슈 #31)

STEP06  [+ Redis]
  Redisson 분산락 (AOP 패턴, 11개 신규 파일)
  @Cacheable Read-Through 캐시
  인기상품 Redis Sorted Set, 40일 후 RDB 영속화
  쿠폰 발급 요청/완료 분리

STEP07  [- Facade, + EDA]
  Facade 클래스 통째 삭제
  ApplicationEventPublisher + @TransactionalEventListener
  BalanceClient 인터페이스 (도메인 격리, 헥사고날 한 발 근접)
  refund / REFUND enum / Saga 보상 트랜잭션
  @EnableAsync

STEP08  [+ Kafka]
  KafkaTemplate, Outbox 패턴
  인기상품 실시간 이벤트 전환 (배치 스케줄러 제거 — 5번째 진화)
  쿠폰 발급 카프카 직렬 처리 (단일 파티션)

STEP09  [+ 관측/부하]
  Prometheus + Grafana, K6 시나리오, 병목 분석/개선
  카프카 에러 로깅, Concurrency 옵션

(옵션) [멀티모듈 분리]
  common/service/support 모듈 분리 (헥사고날에 근접)
```

---

## 4. 핵심 원칙

1. STEP을 건너뛰지 마라 — 락 없이, 캐시 없이, 이벤트 없이 시작
2. 문서가 코드보다 먼저 — 보고서 작성 후 코드 변경
3. `[REFACTOR]`는 죄가 아니다 — 레퍼런스 274커밋 중 ~35%
4. 같은 코드를 여러 번 갈아엎는다 — 인기상품 5번 진화, Balance도 STEP04→05 큰 리팩토링
5. 학습 → 적용 → 회고 사이클 — WIL/Study/Report 누적
6. 라이트 DDD 4원칙 준수 — setter 없음, 정적 팩토리, 의도 있는 메서드, 도메인 내부 검증

---

## 5. GitHub 이슈 매핑 (39개)

### STEP01~02 (완료, 22 closed)

설계 문서 및 REST Docs 학습.

### STEP03 - 도메인 구현(잔액/쿠폰/상품) — 4 issues

|#|제목|상태|
|---|---|---|
|#3|[STEP03] 도메인 구현 - 잔액/쿠폰/상품|부모 트래커|
|#20|잔액 비즈니스 로직 구현 및 단위 테스트|완료 (PR 머지 예정)|
|#21|쿠폰 비즈니스 로직 구현 및 단위 테스트|**다음 작업**|
|#22|상품 비즈니스 로직 구현 및 단위 테스트|대기|

### STEP04 - 도메인 구현(주문/결제) — 6 issues

|#|제목|가이드 매핑|
|---|---|---|
|#4|[STEP04] 부모|-|
|#23|주문/결제 비즈니스 로직 구현 및 단위 테스트|OrderFacade|
|#24|인프라 레이어 구현체 작성|각 도메인 JPA Repository|
|#25|기능별 통합 테스트 작성|Testcontainers|
|#26|주요 기능별 동시성 실패 테스트 작성|STEP04 끝 미리 작성|
|#27|병목 예상 쿼리 분석 및 최적화 보고서 작성|`01.DBPerformanceOptimizationReport.md`|

### STEP05 - 동시성 — 6 issues

|#|제목|가이드 매핑|
|---|---|---|
|#5|[STEP05] 부모|-|
|#28|주요 기능별 동시성 테스트 작성|동시성 테스트 정식|
|#29|주요 기능 동시성 이슈 식별 및 해결|낙관적/비관적 락|
|#30|동시성 이슈 분석 및 해결 보고서 작성|`02.ConcurrencyReport.md`|
|#31|Filter/Interceptor/Scheduler 부가 로직 구현|Rank 도메인 + 배치 + Filter/Interceptor|
|#32|모든 API 정상 작동 및 가용성 확보|RestAssured E2E|

### STEP06 - DB 성능 + 캐시 — 8 issues

|#|제목|가이드 매핑|
|---|---|---|
|#6|[STEP06] 부모|-|
|#33|Redis 기반 분산락 구현 및 적용|Phase 1 Redisson|
|#34|Redis 분산락 동시성 보고서 추가|`02.ConcurrencyReport.md` 부록 또는 별도|
|#35|Redis 기반 캐싱 전략 설정 및 적용|Phase 2 @Cacheable|
|#36|캐싱 전략 및 성능 개선 보고서 작성|`03.CacheStrategyArchitectureReport.md`|
|#37|인기상품 Redis 기반 설계 및 구현|Phase 3 Sorted Set|
|#38|선착순 쿠폰발급 Redis 기반 설계 및 구현|쿠폰 요청/완료 분리|
|#39|Redis 디자인 설계 보고서 작성|`04.RedisDesignArchitectureReport.md`|

### STEP07 - EDA — 4 issues

|#|제목|가이드 매핑|
|---|---|---|
|#7|[STEP07] 부모|-|
|#43|주문/결제 완료 시 이벤트 기반 외부 데이터 플랫폼 전송|Phase 1 ApplicationEventPublisher|
|#44|파사드 제거 및 이벤트 기반 도메인 서비스 구현|refund + BalanceClient + REFUND enum 묻혀있음|
|#45|MSA 기반 이벤트 아키텍처 설계 문서 작성|`05.MsaEventDrivenArchitectureReport.md`|

### STEP08 - Kafka — 6 issues

|#|제목|가이드 매핑|
|---|---|---|
|#8|[STEP08] 부모|-|
|#46|카프카 기초 및 핵심 개념 문서 작성|`docs/study/01.Kafka.md`|
|#47|주문 완료 시 데이터 플랫폼으로 카프카 메시지 발행|주문 이벤트 카프카 발행|
|#48|대용량 트래픽 프로세스 카프카 활용 구현|쿠폰 발급 카프카 + 인기상품 실시간 이벤트 묻힘|
|#49|Outbox 패턴 적용|Outbox 구현|
|#50|카프카 기반 설계 문서 작성|`06.KafkaDesignArchitectureReport.md`|

### STEP09 - 부하테스트 — 5 issues

|#|제목|가이드 매핑|
|---|---|---|
|#9|[STEP09] 부모|-|
|#51|부하테스트 대상 선정 및 시나리오 계획 문서 작성|시나리오 문서|
|#52|부하테스트 스크립트 작성|K6|
|#53|부하테스트 결과 기반 병목 탐색 및 개선|후속 개선|
|#54|성능 테스트 및 장애대응 보고서 작성|`07.LoadTestReport.md`|

---

## 6. 현재 본인 프로젝트 상태

### 패키지 구조 (STEP03 #20 완료 시점)

```
com.github.gokid96.e_commerce
├── balance/
│   ├── interfaces/                    완료
│   │   ├── BalanceController.java
│   │   └── dto/
│   │       ├── BalanceRequest.java   (Charge 정적 내부 클래스)
│   │       └── BalanceResponse.java  (Balance 정적 내부 클래스)
│   ├── application/                   완료
│   │   ├── BalanceFacade.java
│   │   ├── BalanceCriteria.java
│   │   └── BalanceResult.java
│   ├── domain/                        완료
│   │   ├── Balance.java
│   │   ├── BalanceTransaction.java
│   │   ├── BalanceTransactionType.java
│   │   ├── BalanceCommand.java
│   │   ├── BalanceInfo.java
│   │   ├── BalanceRepository.java
│   │   └── BalanceService.java
│   └── infrastructure/
│       ├── jpa/
│       │   └── BalanceJpaRepository.java
│       └── BalanceCoreRepository.java
├── coupon/, order/, product/         interfaces/controller만 mock 상태
├── user/                              완료
│   ├── domain/
│   └── infrastructure/jpa/
├── common/                            완료
│   ├── ApiResponse.java
│   └── ApiControllerAdvice.java
└── ECommerceApplication.java

src/test/
├── java/.../balance/
│   ├── domain/
│   │   ├── BalanceTest.java           6 케이스
│   │   └── BalanceServiceTest.java    6 케이스
│   ├── application/
│   │   └── BalanceFacadeTest.java     2 케이스
│   └── interfaces/
│       └── BalanceControllerTest.java 5 케이스
├── java/.../support/
│   └── ControllerTestSupport.java     @WebMvcTest 공통 부모
└── resources/
    └── application.yaml               H2 테스트 설정
```

### 환경 설정 (STEP03 #20에서 추가됨)

- `build.gradle`: `testRuntimeOnly 'com.h2database:h2'` 추가
- `src/test/resources/application.yaml`: H2 MySQL 모드 + DataSourceAutoConfiguration 활성화
- Spring Boot 4.0.5, Java 21, `tools.jackson.databind.ObjectMapper` (Jackson 신규 패키지)
- 테스트 인프라: `@MockitoBean` (Spring Boot 3.4+ 신규 API)

### 산출물 현황

|카테고리|파일|상태|
|---|---|---|
|docs/architecture/|01~05 + 03-2|완료|
|docs/architecture/|06.SpringRestDocs.md|미작성 (선택)|
|docs/report/|01~07|미작성 (STEP04~09에서)|
|docs/study/|01.Kafka, 02.Cache|미작성 (STEP06, STEP08에서)|
|docs/devlog/|DTO패턴이유.md, 이커머스 아키텍처 설계.md|작성됨 (작업 메모)|
|docs/WIL/|2주차|작성됨|
|docs/WIL/|3~5주차|미작성 (STEP09 후 몰아쓰기 권장)|

---

## 7. 작업 방식 — STEP03 #20에서 확립된 패턴

### A. 한 PR 한 이슈

- 이슈 1개 = 브랜치 1개 = PR 1개
- 브랜치명: `feat/step03-{도메인}-{역할}` (예: `feat/step03-balance-service`)
- 사전 의존(User 도메인)은 별도 PR로 먼저 머지

### B. 작업 순서 (8 Step)

이 순서가 의존성 충돌 없이 가장 자연스러움:

```
1. 도메인 모델 (Entity, Enum)        ← 가장 안쪽부터
2. 도메인 DTO (Command, Info)
3. Repository (interface + Core + Jpa)
4. Service (Repository 의존)
5. Facade Layer (Criteria, Result, Facade)
6. Controller + Request/Response (interfaces 리네임 포함)
7. ApiControllerAdvice 예외 핸들러
8. 테스트 4종 (Domain / Service / Facade / Controller)
```

각 Step 끝나면 `./gradlew compileJava`로 컴파일 확인하고 다음 Step. 테스트는 8 Step에서 한꺼번에.

### C. 커밋 단위 — 작업과 동시에 add+commit (중요)

STEP03 #20에서 배운 것: 작업 다 끝내고 한 번에 add 하면 커밋이 압축됨. 작업하면서 단위마다 커밋.

권장 커밋 단위 (도메인당 7~8개):

```
[FEAT] {도메인} 도메인 모델 구현             ← Step 1
[FEAT] {도메인} 도메인 DTO(Command, Info) 추가  ← Step 2
[FEAT] {도메인} Repository 구현              ← Step 3
[FEAT] {도메인} Service + Facade 구현        ← Step 4, 5
[REFACTOR] {도메인} 인터페이스 패키지 구조 변경  ← Step 6 (controller → interfaces)
[TEST] {도메인} 도메인 단위/통합 테스트 작성  ← Step 8
[CHORE] 환경 설정 (필요 시)                  ← H2, yaml 등
[DOCS] 학습 메모 (선택)
```

다음 작업(#21)에서는 더 굵게 묶어도 OK. 레퍼런스는 도메인당 평균 2~3커밋. 본인 #20은 8커밋이라 좀 잘게 쪼갰음. 둘 다 정답.

### D. PR 본문 (짧게)

```markdown
## 작업 내용
- [FEAT] ...
- [REFACTOR] ...
- [TEST] ...

## 의식적으로 미룬 작업
- @Transactional on Service, @Version → STEP05
- refund, BalanceClient → STEP07
- Redis, Kafka, Testcontainers → 각 STEP

## 리뷰 포인트
- (해당 PR의 임시 구조나 STEP05 이후 변경될 부분 명시)

Close #N
```

회고(KPT)는 WIL로 분리. PR에 부담 X.

### E. 커밋 메시지 컨벤션

|태그|사용처|
|---|---|
|`[FEAT]`|새 기능|
|`[REFACTOR]`|개선/구조 변경|
|`[TEST]`|테스트|
|`[FIX]`|버그|
|`[CHORE]`|빌드/설정/환경|
|`[DOCS]`|문서|
|`[REVERT]`|되돌리기|

---

## 8. 테스트 작성 패턴 (STEP03 확립)

### 4종 분류

|종류|도구|Spring|Mock|무엇을 검증|
|---|---|---|---|---|
|도메인 단위|JUnit + AssertJ|안 띄움|없음|객체 자체의 행위|
|Service|JUnit + Mockito|안 띄움|Repository|분기 흐름|
|Facade|JUnit + Mockito + InOrder|안 띄움|UserService + BalanceService|협업 순서|
|Controller|JUnit + MockMvc|`@WebMvcTest` (부분)|Facade `@MockitoBean`|HTTP 라우팅 + JSON + 검증|

### 케이스 수 가이드 (레퍼런스 따라)

- 도메인 단위: 비즈니스 메서드당 happy + edge 짝지어 6개 정도
- Service: 분기마다(있음/없음) 짝지어 6개 정도
- Facade: InOrder 검증 핵심만 2~3개
- Controller: 정상 + 검증 실패 케이스 4~5개

지나치게 꼼꼼하면 STEP05 리팩토링 때 같이 깨짐 (transaction 컬렉션 검증 등). 핵심만.

### 환경 (Spring Boot 4 기준)

- `@MockBean` 대신 `@MockitoBean` 사용 (`org.springframework.test.context.bean.override.mockito.MockitoBean`)
- `@WebMvcTest` import: `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest` (Boot 4 신규 경로)
- ObjectMapper: `tools.jackson.databind.ObjectMapper` (Jackson 신규 패키지)

### 자주 쓰는 단축어

- `given()...willReturn()` — BDDMockito (한국 표준)
- `verify(repo, never())` / `verify(repo, times(1))` — 호출 검증
- `inOrder(A, B).verify(A); inOrder.verify(B)` — 순서 검증
- `assertThatThrownBy(() -> ...)` — 예외 검증
- `mockMvc.perform(...).andExpect(jsonPath("$.code").value(200))` — HTTP 검증

---

## 9. STEP03 신호등 (절대 하지 말 것)

### #21, #22 작업 시에도 동일

- `@Version` 낙관적 락 → STEP05
- `@Lock(PESSIMISTIC_WRITE)` → STEP05
- `@Transactional` on Service → STEP05 (낙관적 락과 함께)
- `refund` 메서드, `REFUND` enum → STEP07
- `@Cacheable` → STEP06 P2
- `Redisson` → STEP06 P1
- `ApplicationEventPublisher` → STEP07
- `@TransactionalEventListener` → STEP07
- `KafkaTemplate` → STEP08
- `BalanceClient` 인터페이스 → STEP07 (파사드 제거 시)
- `BaseEntity` / `createdAt` 필드 → 레퍼런스에 없음

### 추가 가능한 것 (STEP03 신호등 가능)

- 도메인 엔티티 비즈니스 메서드
- 4-Layer 클린 아키텍처
- Repository I/F + 구현체 분리
- 단위 테스트
- 파사드 클래스 도입
- `@OneToMany` cascade
- `@Enumerated(EnumType.STRING)`

---

## 10. 다음 작업: 이슈 #21 (Coupon 도메인)

### 진입 전 준비

```bash
# main 동기화
git checkout main
git pull

# 새 브랜치
git checkout -b feat/step03-coupon-domain
```

### 작업 방식 (3가지 시도)

이번엔 STEP03 #20 경험 살려 다음 시도:

1. 커밋 단위를 작업 시작 시점부터 의식 — Step 1 끝나면 바로 add+commit, Step 2 또 add+commit
2. AI 가이드 의존도 줄이기 — Balance 패턴 그대로 복사+이름 바꾸기. 막힐 때만 질문
3. 굵직한 커밋도 OK — 레퍼런스 스타일로 도메인 묶어서 3~4커밋으로 가도 됨

### Coupon 도메인 진입 시 미리 봐야 할 것

- 레퍼런스 `service/coupon/src/main/java/kr/hhplus/be/ecommerce/coupon/` 폴더 구조
- Balance와 다른 점: 쿠폰은 발급(create) + 사용(use) + 조회(list) 3가지 행위
- STEP05에서 비관적 락(`@Lock(PESSIMISTIC_WRITE)`) 들어갈 자리 미리 인지하되 STEP03엔 적용 X
- 쿠폰은 사용자별로 다수 보유 — `UserCoupon` 같은 매핑 엔티티 등장 가능성 (레퍼런스 확인)

### Coupon 작업 시 막힐 만한 지점

- 발급/사용 시점 분리: 발급=쿠폰 → 사용자 매핑 생성, 사용=상태 변경
- 만료 검증: 쿠폰 자체에 유효기간 필드
- "전체 발급 수량 - 발급된 수량" 같은 카운팅 — STEP05 비관적 락에서 다시 다룸. STEP03엔 검증만.

---

## 11. STEP04~09 미리보기

### STEP04 핵심 학습 포인트

1. Testcontainers 통합 테스트 (`@SpringBootTest` + MySQL 컨테이너)
2. JPA Repository 채우기 + Repository 인터페이스 `@Repository → @Component`
3. OrderFacade 진가 (`@Transactional` Facade에 적용)
4. `@Enumerated(EnumType.STRING)` 모든 enum
5. 인덱스 적용 — 보고서 먼저 → 코드 순서. 10만 건 + EXPLAIN ANALYZE
6. 동시성 실패 테스트 미리 작성 (STEP05 진입 준비)

산출물: `01.DBPerformanceOptimizationReport.md` (이슈 #27)

### STEP05 핵심 학습 포인트

락 전략:

|자원|전략|이유|
|---|---|---|
|잔액|낙관적 락 (`@Version`)|동일 사용자 중복 충전 = 의도 아님. 하나만 처리|
|쿠폰|비관적 락 (`@Lock(PESSIMISTIC_WRITE)`)|선착순 모두 처리|
|재고|비관적 락|동시 차감 시 음수 방지|

Balance 큰 리팩토링: `@OneToMany` cascade 끊고 ID 참조로. `Balance.create(userId)` 시그니처 변경. `BalanceService`에 `@Transactional` 추가.

기타: Rank 도메인 분리 (인기상품 배치), Filter/Interceptor 도입, API 메서드명 `updateBalance → chargeBalance`

산출물: `02.ConcurrencyReport.md` (이슈 #30)

### STEP06 핵심 (3 Phase)

- Phase 1: Redisson 분산락 AOP (11개 신규 파일, support/lock + infrastructure/lock)
- Phase 2: `@Cacheable` Read-Through, TTL 49시간
- Phase 3: Redis Sorted Set 인기상품, 40일 후 RDB 영속화

분산 락 + 트랜잭션 순서:

```
1. 분산 락 획득 (트랜잭션 밖)
2. 트랜잭션 시작
3. 비즈니스 로직
4. 트랜잭션 커밋
5. 분산 락 해제 (TransactionSynchronizationManager)
```

구현: `@Order(Ordered.HIGHEST_PRECEDENCE)` 락 AOP가 트랜잭션보다 먼저 실행

산출물 3개: 분산락 보고서(#34), 캐시 보고서(#36), Redis 디자인 보고서(#39)

### STEP07 핵심 (Facade 제거 + EDA)

이슈 #44 안에 3가지 큰 작업 묻혀있음:

1. Facade 제거 — `BalanceFacade`, `BalanceCriteria`, `BalanceResult` 삭제
2. `BalanceClient` 인터페이스 신규 (도메인 격리)
3. `refund` 메서드 구현 (+162줄) — `Balance.refund` / `BalanceCommand.Refund` / `BalanceTransaction.ofRefund` / `BalanceTransactionType.REFUND`

Saga 보상 트랜잭션: 결제 실패 → 잔액 환불 + 쿠폰 사용 취소 + 재고 복구

산출물: `05.MsaEventDrivenArchitectureReport.md` (이슈 #45)

### STEP08 핵심 (Kafka + Outbox)

- KafkaTemplate, Outbox 패턴
- 인기상품 실시간 이벤트 (배치 제거 — 5번째 진화)
- 쿠폰 발급 카프카 직렬 처리 (단일 파티션)

Outbox 흐름:

```
Auto 이벤트:
  BEFORE_COMMIT: Outbox 테이블 저장
  AFTER_COMMIT (@Async): Kafka 발행
Manual 이벤트 (트랜잭션 없는 로직):
  즉시 Kafka 발행
Consumer 처리 완료 시: eventId로 Outbox 삭제
```

인기상품 5번 진화:

1. DB 집계 (STEP03~04)
2. 배치 프로세스 (STEP05)
3. Redis 캐시 (STEP06)
4. Redis Sorted Set (STEP06)
5. 실시간 이벤트 (STEP08)

산출물: `01.Kafka.md` (study), `06.KafkaDesignArchitectureReport.md`

### STEP09 핵심 (부하테스트)

- Spring Actuator + Prometheus + Grafana
- K6 시나리오: 주문/결제 현실적 트래픽 + 선착순 쿠폰 Peak Test
- 사후 개선: 카프카 에러 로깅, Concurrency 옵션, CoreException 핸들링

산출물: `07.LoadTestReport.md`

---

## 12. STEP별 신호등

|작업|S03|S04|S05|S06|S07|S08|S09|
|---|---|---|---|---|---|---|---|
|도메인 엔티티 비즈니스 메서드|O|O|O|O|O|O|O|
|4-Layer 클린 아키텍처|O|O|O|O|O|O|O|
|Repository I/F + 구현체 분리|O|O|O|O|O|O|O|
|단위 테스트|O|O|O|O|O|O|O|
|파사드 클래스 도입|O|O|O|O|X(제거)|X|X|
|`@OneToMany` cascade|O|O|X(제거)|X|X|X|X|
|`@Transactional` on Facade|X|O|O|O|X(제거)|X|X|
|`@Transactional` on Service|X|X|O|O|O|O|O|
|통합 테스트 (Testcontainers)|X|O|O|O|O|O|O|
|인덱스 적용|X|O|O|O|O|O|O|
|`@Enumerated(EnumType.STRING)`|O|O|O|O|O|O|O|
|`@Version` 낙관적 락|X|X|O|O|O|O|O|
|`@Lock` 비관적 락|X|X|O|O|O|O|O|
|Rank 도메인 분리|X|X|O|O|O|O|O|
|배치 스케줄러|X|X|O|O|O|X(제거)|X|
|Filter/Interceptor|X|X|O|O|O|O|O|
|Redisson 분산 락|X|X|X|O|O|O|O|
|`@Cacheable` 캐시|X|X|X|O|O|O|O|
|Redis Sorted Set|X|X|X|O|O|O|O|
|`refund` 메서드|X|X|X|X|O|O|O|
|`BalanceClient` 인터페이스|X|X|X|X|O|O|O|
|`REFUND` enum|X|X|X|X|O|O|O|
|`ApplicationEventPublisher`|X|X|X|X|O|O|O|
|`@TransactionalEventListener`|X|X|X|X|O|O|O|
|`@EnableAsync`|X|X|X|X|O|O|O|
|Saga (보상 트랜잭션)|X|X|X|X|O|O|O|
|Kafka Producer/Consumer|X|X|X|X|X|O|O|
|Outbox 패턴|X|X|X|X|X|O|O|
|K6 부하테스트|X|X|X|X|X|X|O|
|Prometheus + Grafana|X|X|X|X|X|X|O|

---

## 13. Balance 도메인 진화 추적표

|항목|STEP03|STEP04|STEP05|STEP06|STEP07|STEP08|
|---|---|---|---|---|---|---|
|`Balance.create`|`(userId, amount)`|동일|`(userId)`|동일|동일|동일|
|`@OneToMany`|cascade 있음|있음|제거|없음|없음|없음|
|`BalanceTransaction` FK|`Balance` 참조|참조|`balanceId` Long|Long|Long|Long|
|`@Version`|없음|없음|추가|있음|있음|있음|
|`@Index`|없음|추가|있음|있음|있음|있음|
|`@Transactional` (Service)|없음|없음|추가|있음|있음|있음|
|`BalanceFacade`|있음|있음|있음|있음|삭제|없음|
|`BalanceClient`|없음|없음|없음|없음|추가|있음|
|`refund` 메서드|없음|없음|없음|없음|추가|있음|
|`REFUND` enum|없음|없음|없음|없음|추가|있음|
|`BalanceRepository.saveTransaction`|없음|없음|추가|있음|있음|있음|
|Controller 메서드명|`chargeBalance`|동일|동일|동일|동일|동일|

본인은 처음부터 `chargeBalance`로 시작해서 레퍼런스의 `updateBalance → chargeBalance` 리네임 없이 진행.

---

## 14. 산출물 체크리스트

### docs/architecture (완료)

|파일|상태|
|---|---|
|`01.Requirements.md`|완료|
|`02.Milestones.md`|완료|
|`03.SequenceDiagram.md`|완료|
|`03-2.StateDiagram.md`|완료|
|`04.ERD.md`|완료|
|`05.ApiDocument.md`|완료|
|`06.SpringRestDocs.md`|선택|

### docs/report (그 STEP 끝나면 즉시 작성)

|파일|작성 시점|이슈|
|---|---|---|
|`01.DBPerformanceOptimizationReport.md`|STEP04 끝|#27|
|`02.ConcurrencyReport.md`|STEP05 끝|#30|
|분산락 보고서 (#34)|STEP06 P1 끝|#34|
|`03.CacheStrategyArchitectureReport.md`|STEP06 P2 끝|#36|
|`04.RedisDesignArchitectureReport.md`|STEP06 P3 끝|#39|
|`05.MsaEventDrivenArchitectureReport.md`|STEP07 끝|#45|
|`06.KafkaDesignArchitectureReport.md`|STEP08 진행 중|#50|
|`07.LoadTestReport.md`|STEP09|#51, #54|

### docs/study

|파일|작성 시점|이슈|
|---|---|---|
|`01.Kafka.md`|STEP08 진행 중 동시 작성|#46|
|`02.Cache.md`|멀티모듈 분리 시점 (선택)|-|

### docs/WIL (몰아 작성 OK)

|파일|작성 시점|다루는 STEP|
|---|---|---|
|WIL 2주차|완료|STEP01~02|
|WIL 3주차|STEP05 끝 또는 STEP09 후|STEP03~04|
|WIL 4주차|STEP09 후|STEP04 트랜잭션 + 인덱스|
|WIL 5주차|STEP09 후|STEP05~06 동시성 + 분산락|

WIL 전략: 핵심 결정/고민만 그 시점에 단문 메모, 정식 회고는 STEP09 후 정리.

---

## 15. 환경 설정 (주의사항)

### Spring Boot 4 패키지 변경

|옛 (Boot 3)|새 (Boot 4)|
|---|---|
|`com.fasterxml.jackson.databind.ObjectMapper`|`tools.jackson.databind.ObjectMapper`|
|`org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest`|`org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`|
|`@MockBean`|`@MockitoBean`|

### 테스트 환경 분리 (STEP03에서 확립)

```
src/main/resources/application.yaml      ← 운영 (MySQL + DataSource exclude)
src/test/resources/application.yaml      ← 테스트 (H2 MySQL 모드 + DataSource 활성화)
```

테스트 yaml의 `autoconfigure.exclude:` 빈 항목으로 main의 exclude 덮어쓰기 → DataSourceAutoConfiguration 활성화.

### build.gradle 핵심 의존성

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
implementation 'org.springframework.boot:spring-boot-starter-webmvc'
implementation 'org.springframework.boot:spring-boot-starter-validation'
runtimeOnly 'com.mysql:mysql-connector-j'

testImplementation 'org.springframework.boot:spring-boot-starter-data-jpa-test'
testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'
testRuntimeOnly 'com.h2database:h2'                        // STEP03에서 추가
```

향후 추가 예정:

- STEP04: Testcontainers
- STEP06: Redisson + Redis
- STEP08: Spring Kafka
- STEP09: Micrometer Prometheus

---

## 16. 트러블슈팅 — STEP03 #20에서 겪은 것

### 1. `contextLoads` 테스트 실패

증상: `NoSuchBeanDefinitionException` — DataSource 빈 못 찾음 원인: main yaml에서 `DataSourceAutoConfiguration` exclude + JpaRepository 추가가 충돌 해결: test 전용 application.yaml + H2 의존성 추가 (위 환경 설정 참고)

### 2. `BalanceServiceTest.useBalance` 불안정 — 첫 실행 실패, 재실행 통과

증상: 단일 실행은 통과, 전체 실행 시 한 번씩 실패 원인: 테스트 메서드가 잘못 작성됨 — `useBalance` 검증해야 하는데 `getBalance` 호출 해결: 메서드 본문 수정. 테스트 대상 메서드를 정확히 호출하는지 확인 필수

### 3. `OrderControllerDocsTest` 컴파일 실패

증상: STEP02에서 만든 미완성 테스트가 옛 패키지 경로 참조 원인: STEP06에서 `controller/` → `interfaces/` 리네임 + `BalanceChargeRequest` 통합으로 import 깨짐 해결: 파일 통째 주석 처리 또는 삭제 (STEP09에서 REST Docs 다시 작성 시 새로 만들기)

### 4. 커밋 단위 압축

증상: 작업 다 끝내고 `git add .` 두 번으로 8단계가 2커밋으로 압축됨 해결: `git reset --soft`로 커밋만 취소 → 단위별로 다시 add+commit → `git push --force-with-lease` 예방: 작업 시작 시점부터 커밋 단위 의식 (다음 도메인부터 적용)

### 5. `git restore` 실수

증상: STEP02 잔존 테스트 파일을 의심해서 restore 했더니 주석 풀린 상태로 복원 교훈: `git restore`는 작업 디렉토리를 마지막 커밋 상태로 되돌림. 옛날 잔재 복원 가능. 의도 안 한 파일이면 그냥 두기.

---

## 17. 학습 자료 작성 전략

### Report (보고서) — 그 STEP 끝나면 즉시

측정 결과/스크린샷 휘발 전에 잡아둠. AS-IS / 문제 / 해결방안 / TO-BE 구조. 레퍼런스 패턴: 인덱스 적용한 날 보고서 작성, 동시성 락 적용한 날 보고서 작성.

### Study (개념 학습) — 그 STEP 진행 중 또는 직후

- Kafka 같이 큰 개념: 진행하면서 동시 정리 (이해도가 가장 깊을 때)
- Cache 같이 여러 STEP에 걸친 개념: 후반에 종합 정리

### WIL (회고) — 한 번에 몰아 작성 OK

레퍼런스 패턴: STEP 끝마다 즉시 안 씀. STEP09 후 몰아 작성.

추천 (C안 - 균형): 핵심 결정/고민만 그 시점에 단문 메모, 정식 회고는 STEP09 후 정리.

---

## 18. 막힐 때

레퍼런스 동일 시점 커밋 확인:

```bash
# 특정 커밋 시점의 파일 상태 보기
git --no-pager show <hash>:<path>

# 키워드로 커밋 검색
git --no-pager log --all --oneline --reverse --date=short \
  --pretty=format:"%h %ad %s" -S "<keyword>" -- "*.java"

# 본인 vs 레퍼런스 파일 비교
git diff --no-index <본인 파일> <레퍼런스 파일>
```

레퍼런스는 최종(멀티모듈) 상태라 본인 STEP과 시점 다를 수 있음. 가이드의 STEP별 커밋 해시 참고.

---

## 19. v7 → v8 변경 요약

- STEP03 #20 실제 작업 완료 상태 반영 (브랜치 머지 예정)
- 작업 방식 섹션 신설 (8 Step 순서, 커밋 단위, PR 본문 구조)
- 테스트 작성 패턴 섹션 신설 (4종 분류, 케이스 수 가이드, Boot 4 환경)
- 트러블슈팅 섹션 신설 (5가지 실제 발생 케이스)
- 환경 설정 섹션 신설 (Boot 4 패키지 변경, test yaml 분리)
- Spring Boot 4 명시 (`@MockitoBean`, `tools.jackson`, 신규 `WebMvcTest` 경로)
- 코드 가이드 섹션 제거 (이미 작업 완료, 채팅에서 제공)
- 이모티콘 제거
- 다음 작업 (#21 Coupon) 구체 가이드 추가

STEP04 진입 시 v9로 갱신 권장. 추가될 항목: Testcontainers 설정, `@Transactional` Facade 적용, 인덱스 보고서 작성 흐름.