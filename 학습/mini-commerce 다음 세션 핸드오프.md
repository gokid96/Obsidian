

> 다음 세션에서 바로 이어 작업할 수 있도록 컨텍스트 + 작업 방식 + 진행 가이드 정리.

---

## 1. 프로젝트 기본 정보

| 항목 | 값 |
|---|---|
| 본인 레포 | https://github.com/gokid96/e-commerce |
| 레퍼런스 | https://github.com/discphy/e-commerce |
| 로컬 경로 | `C:\Users\eborder\sungmin\git\e-commerce` |
| 레퍼런스 로컬 | `C:\Users\eborder\sungmin\git\e-commerce-reference` |
| 패키지 루트 | `com.github.gokid96.e_commerce` |
| 환경 | Spring Boot 4.0.5, Java 21 |

### Spring Boot 4 주의사항

| 옛 (Boot 3) | 새 (Boot 4) |
|---|---|
| `com.fasterxml.jackson.databind.ObjectMapper` | `tools.jackson.databind.ObjectMapper` |
| `org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest` | `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest` |
| `@MockBean` | `@MockitoBean` (`org.springframework.test.context.bean.override.mockito.MockitoBean`) |

---

## 2. 현재 진행 상황

### 완료된 것

| STEP | 이슈/작업 | PR | 상태 |
|---|---|---|---|
| STEP01 | 설계 기본 | - | Done |
| STEP02 | 설계 심화 + Mock API | - | Done |
| STEP03 | User 도메인 | #67 | Done |
| STEP03 | #20 잔액 (Balance) | #68 | Done |
| STEP03 | #21 쿠폰 (Coupon) | #69 | Done |
| STEP03 | REST Docs 빈 파일 머지 사고 복구 (Balance/Coupon 본문 212줄) | - | Done |
| STEP03 | 쿠폰 버그 2건 수정 + 회귀테스트 (발급ID 오류, usedStatus 매핑 누락) | - | Done |

### 진행 중 (현재 작업)

**쿠폰/유저쿠폰 ERD 정렬 리팩토링** — 브랜치 `refactor/align-coupon-erd`

내 구현이 내 ERD/시퀀스 설계와 어긋난 것을 바로잡는 작업. 레퍼런스 전체 문서 검토로 방향 확정됨.

- Coupon: 상태값 `AVAILABLE/UNAVAILABLE` → `REGISTERED/PUBLISHABLE/CANCELED`, `discountAmount(long)` → `discountRate(double)`, `expiredAt` 추가, 수량모델 `totalQuantity+issuedQuantity` → `quantity` 하나
- Coupon.issue(): 상태 → 만료 → 수량 3종 검증 (시퀀스 다이어그램 근거)
- UserCoupon: `issuedAt`/`usedAt` 추가, `@NoArgsConstructor(PROTECTED)` 통일
- PK 컬럼명: `balance_id`, `transaction_id`, `coupon_id`, `user_coupon_id` (`@Column(name=...)`)
- User: `userName` → `nickname`
- 연쇄: CouponInfo/Result/Response(discountRate), UserInfo/UserService(getNickname) + 테스트 갱신

> 중복 발급 검증은 STEP05(동시성, 비관적락+유니크제약)로 미룸 — 레퍼런스 방식.
> UserCoupon의 EXPIRED/CANCELED 상태는 STEP07(환불)로 미룸.

### 다음 작업

**이슈 #22 — 상품(Product) 비즈니스 로직 구현 및 단위 테스트** (STEP03 마지막)

- 브랜치명: `feat/step03-product-domain`
- 정석 흐름: Docs 테스트 함께 작성, Always Green, 커밋 단위 쪼개기
- Product 엔티티(name/price/sellStatus) + Stock 도메인(조회 전용) 동시 구현
- 패키지: `product/domain/product/`, `product/domain/stock/` 분리 (A 방식 확정)
- 응답 `data.products[]` 래핑(stock 포함), Facade 생략, 인기상품(/ranks)은 STEP05+ 제외

---

## 3. 채택 아키텍처 — 라이트 DDD

### 4가지 원칙

| # | 원칙 | 코드 표현 |
|---|---|---|
| 1 | 불변 우선 | `@Setter` 없음, 필드 `private` |
| 2 | 생성은 정적 팩토리 | `Xxx.create(...)` + 검증 |
| 3 | 상태 변경은 의도 있는 메서드 | `charge()`, `use()` (setter 금지) |
| 4 | 검증은 도메인 내부 | 메서드 안에서 비즈니스 규칙 체크 |

### 레이어 구조

```
interfaces  →  application(Facade)  →  domain  ←  infrastructure
   ↑                  ↑                  ↑              ↑
Controller         Facade              Service     CoreRepository
Request/Response  Criteria/Result     Entity      JpaRepository
                                      Command/Info
                                      Repository(인터페이스)
```

### DTO 변환 흐름

```
HTTP JSON
  ↓
XxxRequest        (interfaces)
  ↓ toCriteria(userId)
XxxCriteria       (application)
  ↓ toCommand()
XxxCommand        (domain)
  ↓
[도메인 처리]
  ↓
XxxInfo           (domain)
  ↓ of()
XxxResult         (application)
  ↓ of()
XxxResponse       (interfaces)
  ↓
HTTP JSON
```

### 패키지 네이밍

| 레이어 | 폴더 | 클래스 | 입력 DTO | 출력 DTO |
|---|---|---|---|---|
| Presentation | `interfaces` | `XxxController` | `XxxRequest.Xxx` | `XxxResponse.Xxx` |
| Application | `application` | `XxxFacade` | `XxxCriteria.Xxx` | `XxxResult.Xxx` |
| Domain | `domain` | `XxxService` | `XxxCommand.Xxx` | `XxxInfo.Xxx` |
| Infrastructure | `infrastructure` + `infrastructure/jpa/` | `XxxCoreRepository` + `XxxJpaRepository` | - | - |

---

## 4. 프로젝트에 적용된 기법/패턴 (학습 자산)

### 아키텍처 패턴

- **라이트 DDD (Persistence-aware Domain Model)** — 핵심 아키텍처
- **Layered Architecture (4-Layer)** — interfaces / application / domain / infrastructure
- **Dependency Inversion Principle (DIP)** — Repository 인터페이스를 도메인이 정의
- **(예정) Hexagonal Architecture / Ports & Adapters** — STEP07에서 한 발 근접

### 디자인 패턴

- **Facade Pattern** — BalanceFacade, CouponFacade (STEP07에서 제거 예정)
- **Repository Pattern** — 도메인이 추상화된 저장소 인터페이스만 알도록
- **Static Factory Method** — `Xxx.create()` + 검증 강제
- **Builder Pattern** — Lombok `@Builder`로 자동 생성
- **DTO Pattern** — 레이어 경계마다 객체 변환
- **AOP** — `@Transactional`, `@RestControllerAdvice`
- **(예정) Strategy Pattern** — STEP06 분산락 LockTemplate
- **(예정) Saga Pattern** — STEP07 보상 트랜잭션
- **(예정) Outbox Pattern** — STEP08

### 개발 방법론

- **Test-After Development (TAD)** — 본인 방식. 코드 먼저 → 테스트 나중
- **Mock-First Development** — mock으로 골격 만들고 실제 도메인 채워가기
- **Walking Skeleton** — 전체 골격 먼저 세우고 살 붙이기
- **Iterative Development** — STEP 단위 점진적 진화
- **Continuous Refactoring** — 여러 번 다듬음 (레퍼런스 ~35%가 REFACTOR)
- **Always Green** — 빌드/테스트 항상 통과 유지 (이슈 #22부터 적용)
- **Living Documentation** — Spring REST Docs로 테스트=문서

### 동시성/성능 패턴 (예정)

- **Optimistic Locking** — STEP05, 잔액
- **Pessimistic Locking** — STEP05, 쿠폰/재고
- **Distributed Locking** — STEP06, Redisson
- **Cache-Aside / Read-Through** — STEP06, `@Cacheable`

### 이벤트/메시징 (예정)

- **Publish-Subscribe** — STEP07
- **Event-Driven Architecture (EDA)** — STEP07
- **Message Queue (Kafka)** — STEP08
- **Outbox Pattern** — STEP08

### 테스트 패턴

- **Given-When-Then (BDD 스타일)** — 모든 테스트 메서드
- **Test Doubles (Mock, Stub)** — Mockito
- **Test Fixture** — `ControllerTestSupport`, `RestDocsSupport` 공통 부모
- **(예정) Integration Testing with Testcontainers** — STEP04
- **(예정) E2E Testing** — STEP05 RestAssured
- **(예정) Load Testing** — STEP09 K6

### 문서화/협업

- **Living Documentation** — Spring REST Docs
- **Conventional Commits 변형** — `[FEAT]`, `[REFACTOR]` 등
- **Issue-Driven Development** — GitHub Issues로 작업 단위 추적

### 코딩 스타일

- **Lombok 활용** (`@Getter`, `@Builder`, `@RequiredArgsConstructor`)
- **Static Inner Class for DTOs** (네임스페이스 효과)
- **Immutable Objects** (final 필드, setter 없음)

### 안 쓰는 것 (오해 주의)

- **TDD** — 아님. Test-After
- **순수 DDD** — 아님. Persistence-aware DDD
- **순수 헥사고날** — 아님. 용어 일부 차용
- **마이크로서비스** — 아님. 단일 모듈 (옵션으로 멀티모듈 분리 가능)

### TDD vs 본인 방식 (Test-After + Always Green)

**TDD 핵심 조건**: 1) 빨강(실패 테스트) → 2) 초록(통과시키려 본체) → 3) 리팩토링. = 테스트 → 본체 순서.

**본인 방식**: 본체 먼저 → 테스트 나중 → 본체 변경 시 테스트 같이 갱신(Always Green). = 본체 → 테스트 순서. TDD 아님.

판별 질문 3개:
1. 테스트 처음 작성 시 컴파일 안 됐나? (TDD: 네 / 본인: 아니오)
2. 테스트 통과시키려 본체 작성? (TDD: 네 / 본인: 아니오)
3. 테스트가 본체 설계 이끌었나? (TDD: 네 / 본인: 아니오)

본인 방식은 현업 다수 방식. 학습 단계에 적합. STEP09 후 여유 생기면 TDD 시도 가치 있음.

---

## 5. 작업 방식 (확립된 패턴)

### A. 한 PR 한 이슈

- 이슈 1개 = 브랜치 1개 = PR 1개
- 브랜치명: `feat/step{NN}-{도메인}-{역할}`
- 사전 의존(User 도메인 등)은 별도 PR로 먼저 머지

### B. 도메인 작업 8 Step

```
1. 도메인 모델 (Entity, Enum)
2. 도메인 DTO (Command, Info)
3. Repository (interface + Core + Jpa)
4. Service
5. Facade Layer (Criteria, Result, Facade)
6. Controller + Request/Response (interfaces 리네임)
7. ApiControllerAdvice (이미 IllegalArgumentException 핸들러 있음)
8. 테스트 4종 (Domain / Service / Facade / Controller) + Docs 테스트
```

각 Step 후 `./gradlew compileJava`로 컴파일 확인.

### C. 정석 흐름 (이슈 #22부터 적용)

```
- 도메인 만들면서 Docs 테스트도 같이 작성
- 항상 모든 테스트 통과 상태 유지
- 미완성 테스트는 @Disabled 명시 (그냥 두지 말고)
- STEP02 잔존 mock 테스트는 정식 구현하며 갱신
```

### D. 커밋 단위

작업하면서 단위마다 add+commit (한 번에 압축 금지). 권장 커밋 수: 도메인당 7~8개

```
[FEAT] {도메인} 도메인 모델 구현
[FEAT] {도메인} 도메인 DTO(Command, Info) 추가
[FEAT] {도메인} Repository 구현
[FEAT] {도메인} Service 구현
[FEAT] {도메인} Facade 구현
[REFACTOR] {도메인} 인터페이스 패키지 구조 변경
[TEST] {도메인} 도메인 단위/통합 테스트 작성
[CHORE] 환경 설정 (필요 시)
```

### E. 커밋 메시지 컨벤션

| 태그 | 사용처 |
|---|---|
| `[FEAT]` | 새 기능 |
| `[REFACTOR]` | 개선/구조 변경 |
| `[TEST]` | 테스트 |
| `[FIX]` | 버그 또는 정정 |
| `[CHORE]` | 빌드/설정/환경 |
| `[DOCS]` | 문서 |
| `[REVERT]` | 되돌리기 |

메시지 형식: 한 줄, 짧게. 상세 설명은 PR 본문에.

### F. 커밋 메시지가 잘못 적혔을 때 (rebase)

```
git rebase -i {기준 커밋}   # 예: main 머지 시점 62ae902
# 에디터에서 고칠 커밋의 pick 을 reword 로 변경 → 저장 종료
# 메시지 편집 창에서 새 메시지로 변경 후 저장
```

중복 커밋 합치기는 squash. 막히면 `git rebase --abort`로 원상복구.

---

## 6. PR 본문 형식 (확립됨)

### 기본 템플릿

```
작업 내용

* {커밋해시7자리} [FEAT] {도메인} 도메인 모델 구현
* {커밋해시7자리} [FEAT] {도메인} 도메인 DTO(Command, Info) 추가
* {커밋해시7자리} [FEAT] {도메인} Repository 구현
* {커밋해시7자리} [FEAT] {도메인} Service 구현
* {커밋해시7자리} [FEAT] {도메인} Facade 구현
* {커밋해시7자리} [REFACTOR] {도메인} 인터페이스 패키지 구조 변경
* {커밋해시7자리} [TEST] {도메인} 도메인 단위/통합 테스트 작성

Close #{이슈번호}
```

### 해시 확보 방법

```
git push -u origin {브랜치명}
git log --oneline -10
```

→ 각 커밋 7자리 해시 복사 → PR 본문에 붙여넣기. GitHub가 자동 링크 변환.

### 옵션 — 의식적으로 미룬 작업 / 리뷰 포인트

```
## 의식적으로 미룬 작업
- @Transactional on Service, @Version 낙관적 락 → STEP05
- refund, BalanceClient → STEP07
- Redis, Kafka, Testcontainers → 각 STEP

## 리뷰 포인트
- {해당 PR의 임시 구조나 향후 변경될 부분 명시}
```

---

## 7. 테스트 작성 패턴

### 5종 분류 (Docs 추가)

| 종류 | 도구 | Spring | Mock | 무엇을 검증 |
|---|---|---|---|---|
| 도메인 단위 | JUnit + AssertJ | 안 띄움 | 없음 | 객체 자체의 행위 |
| Service | JUnit + Mockito | 안 띄움 | Repository | 분기 흐름 |
| Facade | JUnit + Mockito + InOrder | 안 띄움 | UserService + Service | 협업 순서 |
| Controller | JUnit + MockMvc | `@WebMvcTest` (부분) | Facade `@MockitoBean` | HTTP 라우팅 + JSON + 검증 |
| Docs | JUnit + MockMvc + REST Docs | 안 띄움 (standalone) | Facade Mockito.mock() | API 문서 자동 생성 |

### 케이스 수 가이드 (레퍼런스 따라)

- 도메인 단위: 비즈니스 메서드당 happy + edge 짝지어 5~7개
- Service: 분기마다(있음/없음) 짝지어 6~8개
- Facade: InOrder 검증 핵심만 2~3개
- Controller: 정상 + 검증 실패 4~6개
- Docs: 정상 시나리오만 (검증 실패는 Controller Test에서 이미 다룸)

지나치게 꼼꼼 X (transaction 컬렉션 검증 등은 STEP05 리팩토링 때 깨짐).

### 자주 쓰는 단축어

- `given(...).willReturn(...)` — BDDMockito
- `verify(repo, never())` / `verify(repo, times(1))` — 호출 검증
- `inOrder(A, B).verify(A); inOrder.verify(B)` — 순서 검증
- `assertThatThrownBy(() -> ...)` — 예외 검증
- `mockMvc.perform(...).andExpect(jsonPath("$.code").value(200))` — HTTP 검증

### Controller 테스트 환경 (@WebMvcTest 방식)

```java
@WebMvcTest(controllers = {
    BalanceController.class,
    CouponController.class,
    ApiControllerAdvice.class
    // ← 새 Controller 추가 시 여기 추가
})
public abstract class ControllerTestSupport {
    @Autowired protected MockMvc mockMvc;
    @Autowired protected ObjectMapper objectMapper;
    @MockitoBean protected BalanceFacade balanceFacade;
    @MockitoBean protected CouponFacade couponFacade;
    // ← 새 Facade 추가 시 여기에
}
```

### Docs 테스트 환경 (standalone 방식)

`RestDocsSupport.java`:

```java
@ExtendWith(RestDocumentationExtension.class)
public abstract class RestDocsSupport {
    protected MockMvc mockMvc;
    protected ObjectMapper objectMapper = new ObjectMapper();

    @BeforeEach
    void setUp(RestDocumentationContextProvider provider) {
        this.mockMvc = MockMvcBuilders.standaloneSetup(initController())
                .apply(documentationConfiguration(provider))
                .setControllerAdvice(new ApiControllerAdvice())
                .build();
    }

    protected abstract Object initController();
}
```

**Controller Test vs Docs Test 결정적 차이**:
- Controller Test = `@WebMvcTest` + `@MockitoBean` (Spring 부분 컨텍스트)
- Docs Test = `MockMvcBuilders.standaloneSetup()` + `Mockito.mock()` (Spring 안 띄움)
- → Docs Test는 `@MockitoBean` 못 씀. 자식이 Mock 직접 만들어 Controller 생성자에 주입.

```java
class BalanceControllerDocsTest extends RestDocsSupport {
    private final BalanceFacade balanceFacade = Mockito.mock(BalanceFacade.class);

    @Override
    protected Object initController() {
        return new BalanceController(balanceFacade);
    }
}
```

### 환경 설정

- `src/main/resources/application.yaml` — 운영 (MySQL + DataSource exclude)
- `src/test/resources/application.yaml` — 테스트 (H2 MySQL 모드)
- `build.gradle`:
    - `testRuntimeOnly 'com.h2database:h2'`
    - `testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'`
    - `asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor'`

---

## 8. 다음 작업 — 이슈 #22 (Product 도메인)

### 진입 전 준비

```
git checkout main
git pull
git checkout -b feat/step03-product-domain
```

### 정석 흐름 체크리스트

- [ ] Product 도메인 정식 구현 (8 Step)
- [ ] `ProductControllerTest`의 `@Disabled` 제거 → 정식 테스트로 갱신
- [ ] `ControllerTestSupport`에 `ProductController` + `ProductFacade` 추가
- [ ] ProductControllerDocsTest 정식 구현 (Balance/Coupon Docs 참고)
- [ ] 모든 테스트 통과 (`./gradlew test --rerun-tasks`)
- [ ] PR 머지

### Product 도메인 진입 시 미리 봐야 할 것

```
Get-ChildItem -Recurse "C:\Users\eborder\sungmin\git\e-commerce-reference\service\product\src\main\java\kr\hhplus\be\ecommerce\product" | Select-Object FullName
```

Balance/Coupon과 다른 점:
- Product는 비교적 단순 (Product 엔티티 + Stock 도메인 분리)
- STEP05에서 비관적 락(`@Lock(PESSIMISTIC_WRITE)`) 들어갈 자리 (재고 차감)
- 인기상품 조회는 STEP05~08에 걸쳐 진화 → STEP03엔 단순 조회만

### 기존 본인 Product 상태 확인

```
Get-ChildItem -Recurse "C:\Users\eborder\sungmin\git\e-commerce\src\main\java\com\github\gokid96\e_commerce\product" | Select-Object FullName
```

### Docs 테스트 작성 시 참고

이미 작성된 `BalanceControllerDocsTest`, `CouponControllerDocsTest` 패턴 따라가기:
- `private final ProductFacade productFacade = Mockito.mock(ProductFacade.class);`
- `initController()`에서 `new ProductController(productFacade)`
- `given(productFacade.xxx(...)).willReturn(...)` 으로 Mock 응답 설정
- `document("product-list", responseFields(...))` 로 문서화

---

## 9. STEP03 신호등 (절대 하지 말 것)

- `@Version` 낙관적 락 → STEP05
- `@Lock(PESSIMISTIC_WRITE)` → STEP05 (재고 차감용)
- `@Transactional` on Service → STEP05
- `@Cacheable` → STEP06
- `Redisson` → STEP06
- `ApplicationEventPublisher` → STEP07
- `@TransactionalEventListener` → STEP07
- `KafkaTemplate` → STEP08
- `BaseEntity` / `createdAt` 필드 → 레퍼런스에 없음

---

## 10. 전체 남은 작업 목록

### 남은 GitHub 이슈 (30개)

| STEP | 이슈 # | 제목 |
|---|---|---|
| STEP03 | 22 | 상품 비즈니스 로직 구현 및 단위 테스트 (다음 작업) |
| STEP04 | 23 | 주문/결제 비즈니스 로직 구현 및 단위 테스트 |
| STEP04 | 24 | 인프라 레이어 구현체 작성 |
| STEP04 | 25 | 기능별 통합 테스트 작성 |
| STEP04 | 26 | 주요 기능별 동시성 실패 테스트 작성 |
| STEP04 | 27 | 병목 예상 쿼리 분석 및 최적화 보고서 작성 |
| STEP05 | 28 | 주요 기능별 동시성 테스트 작성 |
| STEP05 | 29 | 주요 기능 동시성 이슈 식별 및 해결 |
| STEP05 | 30 | 동시성 이슈 분석 및 해결 보고서 작성 |
| STEP05 | 31 | Filter/Interceptor/Scheduler 부가 로직 구현 |
| STEP05 | 32 | 모든 API 정상 작동 및 가용성 확보 |
| STEP06 | 33 | Redis 기반 분산락 구현 및 적용 |
| STEP06 | 34 | Redis 분산락 동시성 보고서 추가 |
| STEP06 | 35 | Redis 기반 캐싱 전략 설정 및 적용 |
| STEP06 | 36 | 캐싱 전략 및 성능 개선 보고서 작성 |
| STEP06 | 37 | 인기상품 Redis 기반 설계 및 구현 |
| STEP06 | 38 | 선착순 쿠폰발급 Redis 기반 설계 및 구현 |
| STEP06 | 39 | Redis 디자인 설계 보고서 작성 |
| STEP07 | 43 | 주문/결제 완료 시 이벤트 기반 외부 데이터 플랫폼 전송 |
| STEP07 | 44 | 파사드 제거 및 이벤트 기반 도메인 서비스 구현 |
| STEP07 | 45 | MSA 기반 이벤트 아키텍처 설계 문서 작성 |
| STEP08 | 46 | 카프카 기초 및 핵심 개념 문서 작성 |
| STEP08 | 47 | 주문 완료 시 데이터 플랫폼으로 카프카 메시지 발행 |
| STEP08 | 48 | 대용량 트래픽 프로세스 카프카 활용 구현 |
| STEP08 | 49 | Outbox 패턴 적용 |
| STEP08 | 50 | 카프카 기반 설계 문서 작성 |
| STEP09 | 51 | 부하테스트 대상 선정 및 시나리오 계획 문서 작성 |
| STEP09 | 52 | 부하테스트 스크립트 작성 |
| STEP09 | 53 | 부하테스트 결과 기반 병목 탐색 및 개선 |
| STEP09 | 54 | 성능 테스트 및 장애대응 보고서 작성 |

### 이슈에 묻혀있는 큰 작업들 (중요)

각 STEP 진입 시 가이드 확인 필수. 이슈 제목만 보면 작업 범위 과소평가됨.

| 이슈 | 묻혀있는 실제 작업 |
|---|---|
| #29 (동시성 해결) | **Balance 큰 리팩토링** — `@OneToMany` 제거, `balanceId Long` 참조, `Balance.create(userId)` 시그니처 변경, `saveTransaction` 명시 호출 |
| #31 (Filter/Interceptor) | **Rank 도메인 분리**, 배치 스케줄러 도입 |
| #44 (Facade 제거) | **3가지**: Facade 제거 / `BalanceClient` 인터페이스 / `refund` + `REFUND` enum + Saga 보상 트랜잭션 |
| #48 (카프카 트래픽) | **인기상품 실시간 이벤트** (5번째 진화, 배치 제거) |

### 이슈로 안 잡혀있는 항목 (놓치기 쉬움)

| 항목 | 시점 | 처리 방법 |
|---|---|---|
| WIL 3주차 (STEP03~04 회고) | STEP05 끝 또는 STEP09 후 | C안 — 핵심 메모만 그 시점에, 정식 회고는 STEP09 후 |
| WIL 4주차 (STEP04 트랜잭션/인덱스) | STEP09 후 | 몰아 작성 |
| WIL 5주차 (STEP05~06 동시성/분산락) | STEP09 후 | 몰아 작성 |
| STEP02 산출물 `06.SpringRestDocs.md` | 선택사항 | 작성 또는 생략 |
| OrderControllerTest @Disabled 제거 | STEP04 #23 작업 시 | 정식 테스트로 갱신 |
| ProductControllerTest @Disabled 제거 | STEP03 #22 작업 시 | 정식 테스트로 갱신 |
| OrderControllerDocsTest 재작성 | STEP04 #23 | 정석 흐름으로 함께 작성 |
| ProductControllerDocsTest 정식 작성 | STEP03 #22 | 정석 흐름으로 함께 작성 |
| Balance 도메인 진화 작업들 | STEP05 #29, STEP07 #44 | 가이드 13번 진화 추적표 참조 |
| 인기상품 5번 진화 | STEP05 #31, STEP06 #37, STEP08 #48 | DB집계 → 배치 → Redis캐시 → Sorted Set → 실시간 이벤트 |
| API 메서드명 정정 (`updateBalance → chargeBalance`) | 본인은 처음부터 `chargeBalance`라 생략 가능 | - |

### 산출물 체크리스트 (STEP별 즉시 작성)

| 파일 | 작성 시점 | 이슈 |
|---|---|---|
| `01.DBPerformanceOptimizationReport.md` | STEP04 끝 | #27 |
| `02.ConcurrencyReport.md` | STEP05 끝 | #30 |
| 분산락 보고서 (#34) | STEP06 P1 끝 | #34 |
| `03.CacheStrategyArchitectureReport.md` | STEP06 P2 끝 | #36 |
| `04.RedisDesignArchitectureReport.md` | STEP06 P3 끝 | #39 |
| `05.MsaEventDrivenArchitectureReport.md` | STEP07 끝 | #45 |
| `06.KafkaDesignArchitectureReport.md` | STEP08 진행 중 | #50 |
| `07.LoadTestReport.md` | STEP09 | #51, #54 |
| `docs/study/01.Kafka.md` | STEP08 진행 중 동시 | #46 |
| `docs/study/02.Cache.md` | 멀티모듈 분리 시점 (선택) | - |

---

## 11. STEP04~09 핵심 미리보기

### STEP04 (다음다음 작업)

1. Testcontainers 통합 테스트 (`@SpringBootTest` + MySQL 컨테이너)
2. JPA Repository 채우기 + Repository 인터페이스 `@Repository → @Component`
3. **OrderFacade에 `@Transactional` 적용** (Service 아님!)
4. `@Enumerated(EnumType.STRING)` 모든 enum
5. 인덱스 적용 — **보고서 먼저 → 코드** 순서. 10만 건 + EXPLAIN ANALYZE
6. 동시성 실패 테스트 미리 작성 (STEP05 진입 준비)

### STEP05

락 전략:

| 자원 | 전략 | 이유 |
|---|---|---|
| 잔액 | 낙관적 락 (`@Version`) | 동일 사용자 중복 충전 = 의도 아님 |
| 쿠폰 | 비관적 락 (`@Lock(PESSIMISTIC_WRITE)`) | 선착순 모두 처리 |
| 재고 | 비관적 락 | 동시 차감 시 음수 방지 |

**Balance 큰 리팩토링** (이슈 #29에 묻힘):
- `@OneToMany` cascade 제거 → `balanceId Long` 참조
- `Balance.create(userId)` 시그니처 변경 (amount 제거)
- `BalanceService`에 `@Transactional` 추가
- `saveTransaction` 메서드 명시 호출

기타: Rank 도메인 분리, Filter/Interceptor 도입

### STEP06 (3 Phase)

- Phase 1: Redisson 분산락 AOP (11개 신규 파일, `support/lock` + `infrastructure/lock`)
- Phase 2: `@Cacheable` Read-Through, TTL 49시간
- Phase 3: Redis Sorted Set 인기상품, 40일 후 RDB 영속화

**분산 락 + 트랜잭션 순서** (매우 중요):

```
1. 분산 락 획득 (트랜잭션 밖)
2. 트랜잭션 시작
3. 비즈니스 로직
4. 트랜잭션 커밋
5. 분산 락 해제 (TransactionSynchronizationManager)
```

구현: `@Order(Ordered.HIGHEST_PRECEDENCE)` 락 AOP가 트랜잭션보다 먼저 실행

### STEP07 (Facade 제거 + EDA)

이슈 #44 안에 3가지 큰 작업:
1. **Facade 제거** — `BalanceFacade`, `BalanceCriteria`, `BalanceResult` 삭제
2. **`BalanceClient` 인터페이스** 신규 (도메인 격리)
3. **`refund` 메서드** 구현 (+162줄): `Balance.refund(amount)`, `BalanceCommand.Refund`, `BalanceTransaction.ofRefund()`, `BalanceTransactionType.REFUND`

Saga 보상 트랜잭션: 결제 실패 → 잔액 환불 + 쿠폰 사용 취소 + 재고 복구

### STEP08 (Kafka + Outbox)

- KafkaTemplate, Outbox 패턴
- 인기상품 실시간 이벤트 (배치 제거 — 5번째 진화)
- 쿠폰 발급 카프카 직렬 처리 (파티션키=쿠폰ID)

**Outbox 흐름**:

```
Auto 이벤트 (트랜잭션 있음):
  BEFORE_COMMIT: Outbox 테이블 저장
  AFTER_COMMIT (@Async): Kafka 발행
Manual 이벤트 (트랜잭션 없는 로직):
  즉시 Outbox 저장 후 Kafka 발행
Consumer 처리 완료 시: eventId로 Outbox 삭제
```

**인기상품 5번 진화**: 1) DB 집계(STEP03~04) → 2) 배치(STEP05) → 3) Redis 캐시(STEP06) → 4) Redis Sorted Set(STEP06) → 5) 실시간 이벤트(STEP08)

### STEP09 (부하테스트)

- Spring Actuator + Prometheus + Grafana
- K6 시나리오: 주문/결제 현실적 트래픽(최대 300VU) + 선착순 쿠폰 Peak Test(최대 1000VU)
- SLA: p99 < 1s, 실패율 < 1%(주문) / 5%(쿠폰)
- 사후 개선: 인기상품 캐싱 + Kafka 파티션/컨슈머 추가, CoreException→ack로 Lag 방지

---

## 12. STEP별 신호등 (전체)

| 작업 | S03 | S04 | S05 | S06 | S07 | S08 | S09 |
|---|---|---|---|---|---|---|---|
| 도메인 엔티티 비즈니스 메서드 | O | O | O | O | O | O | O |
| 4-Layer 클린 아키텍처 | O | O | O | O | O | O | O |
| Repository I/F + 구현체 분리 | O | O | O | O | O | O | O |
| 단위 테스트 | O | O | O | O | O | O | O |
| Docs 테스트 | O | O | O | O | O | O | O |
| 파사드 클래스 도입 | O | O | O | O | X(제거) | X | X |
| `@OneToMany` cascade | O | O | X(제거) | X | X | X | X |
| `@Transactional` on Facade | X | O | O | O | X(제거) | X | X |
| `@Transactional` on Service | X | X | O | O | O | O | O |
| 통합 테스트 (Testcontainers) | X | O | O | O | O | O | O |
| 인덱스 적용 | X | O | O | O | O | O | O |
| `@Enumerated(EnumType.STRING)` | O | O | O | O | O | O | O |
| `@Version` 낙관적 락 | X | X | O | O | O | O | O |
| `@Lock` 비관적 락 | X | X | O | O | O | O | O |
| Rank 도메인 분리 | X | X | O | O | O | O | O |
| 배치 스케줄러 | X | X | O | O | O | X(제거) | X |
| Filter/Interceptor | X | X | O | O | O | O | O |
| Redisson 분산 락 | X | X | X | O | O | O | O |
| `@Cacheable` 캐시 | X | X | X | O | O | O | O |
| Redis Sorted Set | X | X | X | O | O | O | O |
| `refund` 메서드 | X | X | X | X | O | O | O |
| `BalanceClient` 인터페이스 | X | X | X | X | O | O | O |
| `REFUND` enum | X | X | X | X | O | O | O |
| `ApplicationEventPublisher` | X | X | X | X | O | O | O |
| `@TransactionalEventListener` | X | X | X | X | O | O | O |
| `@EnableAsync` | X | X | X | X | O | O | O |
| Saga (보상 트랜잭션) | X | X | X | X | O | O | O |
| Kafka Producer/Consumer | X | X | X | X | X | O | O |
| Outbox 패턴 | X | X | X | X | X | O | O |
| K6 부하테스트 | X | X | X | X | X | X | O |
| Prometheus + Grafana | X | X | X | X | X | X | O |

---

## 13. Balance 도메인 진화 추적표

| 항목 | STEP03 | STEP04 | STEP05 | STEP06 | STEP07 | STEP08 |
|---|---|---|---|---|---|---|
| `Balance.create` | `(userId, amount)` | 동일 | `(userId)` | 동일 | 동일 | 동일 |
| `@OneToMany` | cascade | 있음 | **제거** | 없음 | 없음 | 없음 |
| `BalanceTransaction` FK | `Balance` 참조 | 참조 | **`balanceId` Long** | Long | Long | Long |
| `@Version` | 없음 | 없음 | **추가** | 있음 | 있음 | 있음 |
| `@Index` | 없음 | **추가** | 있음 | 있음 | 있음 | 있음 |
| `@Transactional` (Service) | 없음 | 없음 | **추가** | 있음 | 있음 | 있음 |
| `BalanceFacade` | 있음 | 있음 | 있음 | 있음 | **삭제** | 없음 |
| `BalanceClient` | 없음 | 없음 | 없음 | 없음 | **추가** | 있음 |
| `refund` 메서드 | 없음 | 없음 | 없음 | 없음 | **추가** | 있음 |
| `REFUND` enum | 없음 | 없음 | 없음 | 없음 | **추가** | 있음 |
| `BalanceRepository.saveTransaction` | 없음 | 없음 | **추가** | 있음 | 있음 | 있음 |

---

## 14. Coupon 도메인 진화 추적표

ERD 정렬 리팩토링으로 STEP03 기준 확정. 이후 진화 예정.

| 항목 | STEP03 (정렬 후) | STEP05 | STEP06 | STEP08 |
|---|---|---|---|---|
| Coupon 상태값 | `REGISTERED/PUBLISHABLE/CANCELED` | 동일 | +`FINISHED`(발급종료) | 동일 |
| `discountRate` | double | 동일 | 동일 | 동일 |
| `quantity` | int 하나 (`quantity--`) | 동일 | 동일 | 동일 |
| `expiredAt` | 있음 | 있음 | 있음 | 있음 |
| 발급 검증 | 상태+만료+수량 | **+중복발급** (유니크제약) | 동일 | 동일 |
| 발급 동시성 | 없음 | **비관적 락** | **+분산 락** | **Kafka 직렬**(파티션키=쿠폰ID) |
| UserCoupon 상태 | `UNUSED/USED` | 동일 | 동일 | 동일 |
| UserCoupon `EXPIRED/CANCELED` | 없음 | 없음 | 없음 | (STEP07 환불 영역) |
| 중복발급 방지 | 없음 | UK(user_id,coupon_id) | Redis addIfAbsent | Kafka 멱등성 |

---

## 15. 트러블슈팅 — 이미 겪은 것 (재발 방지)

### 1. `contextLoads` 테스트 실패
- 증상: `NoSuchBeanDefinitionException` — DataSource 빈 못 찾음
- 원인: main yaml에서 `DataSourceAutoConfiguration` exclude + JpaRepository 추가 충돌
- 해결: test 전용 application.yaml + H2 의존성 추가 — 이미 적용됨

### 2. Service 테스트 메서드 잘못 호출
- 증상: 단일 실행 통과, 전체 실행 시 한 번씩 실패
- 원인: 테스트가 잘못된 메서드 호출 (예: `useBalance` 검증해야 하는데 `getBalance` 호출)
- 예방: "테스트 대상 메서드를 정확히 호출하는가" 확인 필수

### 3. ControllerDocsTest 컴파일 실패 — 본질은 미동기화
- 증상: STEP02 Docs 테스트가 STEP03 본체 갈아엎으면서 깨짐
- 원인: 본체에 의존성 추가됐는데(`new XxxController(facade)`) Docs 테스트는 옛 코드(`new XxxController()`) 그대로
- 본질: "미완성"이 아니라 "본체 변경 시 테스트 동기화 안 함"
- 해결(정석): 본체 갈아엎을 때 같이 갱신 (이슈 #22부터 적용)
- 레퍼런스 방식: Docs 테스트 삭제 안 함. 본체 진화에 맞춰 같이 갱신해 끝까지 살림

### 4. 커밋 단위 압축
- 증상: `git add .` 두 번으로 8단계가 2커밋으로 압축됨
- 해결: `git reset --soft`로 커밋만 취소 → 단위별 다시 add+commit → `git push --force-with-lease`
- 예방: 작업 시작부터 커밋 단위 의식, Step 끝나면 바로 add+commit

### 5. `git restore` 실수
- 증상: STEP02 잔존 테스트 파일 의심해 restore 했더니 주석 풀린 상태로 복원
- 교훈: `git restore`는 작업 디렉토리를 마지막 커밋 상태로 되돌림. 의도 안 한 파일이면 그냥 두기.

### 6. Order/Product mock 테스트 깨짐
- 증상: `ControllerTestSupport`의 `@WebMvcTest(controllers={...})`에서 Order/Product 빼면 상속 테스트 깨짐
- 해결: `@Disabled` 처리 + 사유 명시
- 예방: Support controllers 배열에서 뭘 뺄 때 상속 테스트 영향 확인

### 7. 도메인 검증 조건 (`<` vs `<=`)
- 증상: "양수여야 한다" 메시지인데 0이 통과됨
- 원인: `if (amount < 0)` → 0 통과
- 해결: `if (amount <= 0)` — 0도 막음. "양수"는 0 초과.

### 8. 예외 타입 불일치
- 증상: `IllegalStateException` 던졌는데 ApiControllerAdvice가 500 응답
- 원인: ApiControllerAdvice는 `IllegalArgumentException`만 잡음
- 해결: 도메인 검증 실패는 `IllegalArgumentException`로 통일

### 9. Rebase 활용 — 커밋 메시지 잘못 적힘
- 증상: Coupon 작업 중인데 커밋 메시지가 "Balance 도메인 모델 구현"으로 적힘 (복붙 실수)
- 해결: `git rebase -i {기준 커밋}` → `reword`로 메시지 변경
- 추가: 중복 커밋 합치기는 `squash`. 안전판: `git rebase --abort`. 강제 push: `git push --force-with-lease` (단독 브랜치라 안전)

### 10. Docs 테스트의 Mock 주입 방식 차이
- 증상: Controller Test에서 쓰던 `@MockitoBean`이 Docs Test에서 안 통함
- 원인: Docs Test는 `MockMvcBuilders.standaloneSetup()` 방식이라 Spring 컨텍스트 안 띄움
- 해결: 자식 클래스에서 `Mockito.mock()`으로 직접 Mock 생성 → Controller 생성자에 주입

### 11. 빈 파일 머지 사고 (REST Docs)
- 증상: PR이 Docs 테스트를 빈 파일(0줄)로 커밋·머지. 실제 본문 212줄은 working tree에만 있었음
- 해결: `git stash` → main 최신화 → 새 브랜치 → `stash pop` → 본문 커밋 → PR 머지
- 예방: 커밋 전 `git diff --cached`로 실제 내용 들어갔는지 확인

### 12. 테스트가 못 잡은 버그 (쿠폰)
- 증상1: `issueCoupon`이 `findCouponById(userId)` 호출 (couponId여야 함) — 테스트가 userId==couponId(둘 다 1L)라 못 잡음
- 증상2: `UserCoupon.of()`에 `usedStatus` 매핑 누락 → 응답 항상 null — assert 없어 못 잡음
- 해결: 테스트 데이터를 서로 다른 값으로(userId=1, couponId=5), 모든 필드 assert 추가
- 교훈: 테스트 데이터에 같은 값 쓰면 ID 바꿔치기 버그 못 잡음. 응답 필드는 빠짐없이 assert.

---

## 16. 막힐 때 활용할 명령

레퍼런스 동일 시점 커밋 확인:

```
# 특정 커밋 시점의 파일 상태 보기
git --no-pager show <hash>:<path>

# 키워드로 커밋 검색
git --no-pager log --all --oneline --reverse --date=short --pretty=format:"%h %ad %s" -S "<keyword>" -- "*.java"

# 본인 vs 레퍼런스 파일 비교
git diff --no-index <본인 파일> <레퍼런스 파일>
```

레퍼런스는 최종(멀티모듈) 상태라 본인 STEP과 시점 다를 수 있음.

레퍼런스 폴더 둘러보기:

```
Get-ChildItem -Recurse "C:\Users\eborder\sungmin\git\e-commerce-reference\service\{도메인}\src\main\java\kr\hhplus\be\ecommerce\{도메인}" | Select-Object FullName
```

---

## 17. 학습 자료 작성 전략

### Report (보고서) — 그 STEP 끝나면 즉시
측정 결과/스크린샷 휘발 전에 잡아둠. AS-IS / 문제 / 해결방안 / TO-BE 구조.

### Study (개념 학습)
- Kafka 같이 큰 개념: 진행하면서 동시 정리 (이해도 깊을 때)
- Cache 같이 여러 STEP에 걸친 개념: 후반에 종합 정리

### WIL (회고) — 몰아 작성 OK
- 핵심 결정/고민만 그 시점에 단문 메모
- 정식 회고는 STEP09 후 정리

### 본인 어필 자산 (면접/이력서)

```
- 라이트 DDD (Persistence-aware Domain Model) 적용
- 4-Layer 클린 아키텍처 + DIP
- Facade Pattern → EDA 전환 경험 (STEP03 → STEP07)
- Repository Pattern으로 영속성 격리
- Static Factory Method + Builder Pattern으로 도메인 캡슐화
- DTO 변환 흐름으로 레이어 간 의존 분리
- Test-After Development + Always Green 원칙
- Given-When-Then 테스트 작성 + Living Documentation (Spring REST Docs)
- (예정) 낙관적/비관적 락, 분산락, Redis 캐시, Kafka, Saga, Outbox 패턴
- (예정) 인기상품 5번 진화 사례 (DB → 배치 → 캐시 → SortedSet → 실시간 이벤트)
```

특히 강한 면접 무기:
1. 라이트 DDD — 한국 백엔드 면접 매우 흔함
2. Facade → EDA 전환 경험 (STEP07 후) — 진화 사례
3. 분산락 + 트랜잭션 순서 (STEP06 후) — 실무 깊이
4. 인기상품 5번 진화 (STEP08 후) — 진화하는 설계 사례
5. Outbox + Kafka (STEP08 후) — 메시지 신뢰성

---

## 18. 레퍼런스 학습 인덱스 (전체 문서 학습 맵)

레퍼런스 README에 걸린 **19개 문서 전부**를 학습 대상으로 정리.
원칙: **미리 다 읽되, 구현은 신호등대로.** 지금 STEP에 맞는 건 깊게, 나머지는 해당 STEP 진입 시 그 문서 1개를 읽고 시작.

> ⚠️ 레퍼런스는 멀티모듈 **최종본**이라 STEP03 시점과 코드가 다름. 문서는 "방향" 참고용, 코드는 해당 STEP 커밋 시점 기준으로 봐야 함. (쿠폰만 해도 STEP03 → STEP05 → STEP06에서 여러 번 재설계됨)
>
> 로컬 경로: `C:\Users\eborder\sungmin\git\e-commerce-reference\docs\`

### A. 설계 문서 (docs/architecture) — STEP02 산출물, 지금 다 봐도 안전

| 파일 | 핵심 내용 | 학습 시점 |
|---|---|---|
| 01.Requirements.md | 기능/비기능 요구사항. 쿠폰 발급 검증 6단계(사용자→존재→중복→상태→수량→만료), 잔액 최대 1000만원 | 지금 (반영됨) |
| 02.Milestones.md | 주차별 로드맵(2주 설계→3주 도메인→4주 주문/통계→5주 동시성→6주 캐시/인덱스→7주 Kafka) | 지금 |
| 03-1.SequenceDiagram.md | 잔액/상품/쿠폰/주문결제/랭킹 시퀀스. 주문결제 30단계가 STEP04 구현 청사진 | 지금~STEP04 |
| 03-2.StateDiagram.md | 주문(CREATED/PAID), 결제(READY/COMPLETED/FAILED), UserCoupon(ISSUED/USED/EXPIRED/CANCELED) | 지금 (EXPIRED/CANCELED는 STEP07) |
| 04.ERD.md | 전체 테이블. PK 별도 ID 전략, 동시성 컬럼(amount/quantity), 상태컬럼 의도 | 지금 (쿠폰 정렬 기준) |
| 05.ApiDocument.md | API 명세. 쿠폰 응답 `discountRate`, 상품 `data.products[]` 래핑 | 지금~STEP04 |
| 06.SpringRestDocs.md | REST Docs 스크린샷 인덱스 | 선택 (본인 06 산출물 미작성) |

### B. 기술 보고서 (docs/report) — STEP04~09, 해당 STEP 진입 시 깊게

| 파일 | STEP/이슈 | 핵심 학습 포인트 |
|---|---|---|
| 01.DBPerformanceOptimizationReport.md | STEP04 #27 | 인덱스 전후 EXPLAIN ANALYZE. 교훈: 카디널리티 낮은 컬럼(sell_status)은 인덱스 역효과 → 커서 페이징. 복합인덱스 (user_id,used_status)/(user_id,coupon_id) |
| 02.ConcurrencyReport.md | STEP05 #30 | 낙관/비관/분산락. 자원별 일관 전략 — 잔액=낙관, 쿠폰=비관+분산, 재고=비관. 공정성 한계 → Kafka |
| 03.CacheStrategyArchitectureReport.md | STEP06 #36 | @Cacheable Read-Through, @CachePut, TTL 49h 이유(배치 겹침 회피), 캐시 스탬피드 |
| 04.RedisDesignArchitectureReport.md | STEP06 #39 | ZSET 인기상품(ZINCRBY/ZUNIONSTORE), 선착순쿠폰(addIfAbsent 중복방지), 배치 발급 |
| 05.MsaEventDrivenArchitectureReport.md | STEP07 #45 | Facade 제거, @TransactionalEventListener AFTER_COMMIT, Saga 보상 트랜잭션, Redis Hash 프로세스 상태 |
| 06.KafkaDesignArchitectureReport.md | STEP08 #50 | Outbox 패턴, 파티션키=쿠폰ID(순차보장), 멱등성, 토픽 버저닝 |
| 07.LoadTestReport.md | STEP09 #54 | K6+Prometheus+Grafana, SLA(p99<1s), 개선 전후 비교, Lag 방지(CoreException→ack) |

### C. 스터디 (docs/study)

| 파일 | STEP/이슈 | 핵심 학습 포인트 |
|---|---|---|
| 01.Kafka.md | STEP08 #46 | Broker/Producer/Consumer/Topic/Partition/ConsumerGroup, 파티션-컨슈머 비율 3케이스, Outbox/Inbox/DLQ, Rebalancing, Replication |
| (02.Cache.md) | 선택 | 멀티모듈 분리 시점에 종합 정리 |

### D. WIL (회고) — 개념 학습 노트

| 파일 | 다루는 STEP | 핵심 학습 포인트 |
|---|---|---|
| week2 | STEP02 회고 | 설계 중요성, 요구사항 "구체화", 최소스펙 vs 확장성, 주문/결제 통합 결정 |
| week3 ⭐ | STEP03 회고 | 가장 중요. 클린 레이어드 아키텍처, DTO 3종(Command/Criteria/Request), Facade "울며 겨자먹기"(단일도메인이면 생략), 검증은 도메인 객체 내부에, 도메인-엔티티 분리, JPA 연관관계 최소화(애그리거트 내부만 허용) → 본인 라이트 DDD의 직접 근거 |
| week4 | STEP04 학습노트 | 트랜잭션 ACID/격리수준/MVCC, @Transactional 주의(자기호출/private/readOnly), 인덱스 B-Tree/복합/커버링/카디널리티 |
| week5 (5·6주) | STEP05~06 학습노트 | 동시성 유형(Race/Lost Update), synchronized/ReentrantLock/Atomic, 낙관/비관락, 분산락 순서(락획득→트랜잭션→커밋→락해제), Spin/PubSub Lock |

### 학습 순서 원칙 (신호등 연동)

```
지금 (STEP03 쿠폰 정렬 ~ #22 Product):
  → A 설계문서 전체 + WIL week2,3 까지만 깊게 학습
  → 나머지(보고서/Kafka/WIL4,5)는 "있다"만 알고 넘어감

각 STEP 진입 시:
  → 그 STEP의 보고서/WIL 1개를 읽고 시작 (B/C/D 표의 STEP 매핑 참고)
  → 예: STEP05 진입 → 02.ConcurrencyReport + WIL week5 정독 후 코드
```

> 보고서는 "내가 그 STEP을 끝내고 직접 쓸 산출물"이기도 함 (섹션 17 참조).
> 레퍼런스 보고서는 목차/구조/AS-IS·TO-BE 형식을 베끼되, 측정값·코드는 내 것으로 채울 것.

---

## 19. 다음 세션 시작 시 안내 문구

```
mini-commerce 프로젝트 진행할거야.

지금은 쿠폰/유저쿠폰 ERD 정렬 리팩토링 중 (refactor/align-coupon-erd).
엔티티(CouponStatus/Coupon/UserCoupon + PK컬럼명 + User nickname) 반영 후
./gradlew compileJava 돌려서 깨지는 DTO/테스트 정리하는 단계.

ERD 정렬 끝나면 STEP03 이슈 #22 (Product 도메인) 진행.
정석 흐름(Docs 테스트 함께, Always Green, 커밋 쪼개기)으로.

먼저 레퍼런스 Product 구조랑 본인 현재 Product 상태 확인하고 시작하자.
```

---

> 이 문서는 살아있는 문서. STEP 진행하며 발견하는 것 추가/갱신.