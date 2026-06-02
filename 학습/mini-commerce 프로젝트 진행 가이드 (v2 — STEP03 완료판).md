# mini-commerce 프로젝트 진행 가이드 (v2 — STEP03 완료판)

> 레퍼런스 문서 19종 + 도메인 코드를 **실제로 정독·검증**하여 작성한 핸드오프 문서. 다음 세션에서 바로 이어 작업할 수 있도록 컨텍스트 + 작업 방식 + 진행 가이드를 정리.
> 
> **검증 표기**: ✅ = 레퍼런스 문서/코드로 직접 확인함 · ⚠️ = 부분 확인 · 📌 = 결정사항

---

## 0. 이 문서의 검증 범위

**실제로 읽고 검증한 것 (이제 전부 ✅)**:

- ✅ architecture 전부: 01.Requirements / 02.Milestones / 03-1.SequenceDiagram / 03-2.StateDiagram / 04.ERD / 05.ApiDocument
- ✅ report 전부 7종: 01.DBPerformance / 02.Concurrency / 03.CacheStrategy / 04.RedisDesign / 05.MsaEventDriven / 06.KafkaDesign / 07.LoadTest
- ✅ WIL 전부 4종: week2(설계) / week3(아키텍처) / week4(트랜잭션·인덱스) / week5(동시성·분산락)
- ✅ study 2종: 01.Kafka / 02.Cache
- ✅ 도메인 코드 패턴: Balance, Coupon, Product, Stock (레퍼런스 최종본 + 본인 구현)

> v1에서 ⚠️로 남았던 week5/Kafka/Cache/week2를 이번 세션에 모두 정독 → 전부 ✅ 승격. 06.SpringRestDocs.md는 REST Docs 스크린샷 인덱스라 선택. 본인 06 산출물은 미작성.

> week5/Kafka/Cache의 개념 학습 노트는 부록 A에 정리 (해당 STEP 진입 시 참고).

---

## 1. 프로젝트 기본 정보

|항목|값|
|---|---|
|본인 레포|https://github.com/gokid96/e-commerce|
|레퍼런스|https://github.com/discphy/e-commerce|
|로컬 경로|`C:\Users\eborder\sungmin\git\e-commerce`|
|레퍼런스 로컬|`C:\Users\eborder\sungmin\git\e-commerce-reference`|
|패키지 루트|`com.github.gokid96.e_commerce`|
|환경|Spring Boot 4.0.5, Java 21|

### Spring Boot 4 주의사항

|옛 (Boot 3)|새 (Boot 4)|
|---|---|
|`com.fasterxml.jackson.databind.ObjectMapper`|`tools.jackson.databind.ObjectMapper`|
|`...web.servlet.WebMvcTest`|`org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`|
|`@MockBean`|`@MockitoBean` (`org.springframework.test.context.bean.override.mockito.MockitoBean`)|

### ⚠️ 레퍼런스는 멀티모듈 "최종본"이다 (중요)

레퍼런스는 STEP09까지 끝낸 **멀티모듈 최종본**. `service/{도메인}/...` 구조, 패키지 루트 `kr.hhplus.be.ecommerce`, Rank/Redis/Kafka/Querydsl/Scheduler가 전부 섞여 있음. → **문서는 "방향" 참고, 코드는 "현재 STEP"에 맞는 부분만** 잘라 쓴다. (쿠폰만 해도 STEP03→05→06 여러 번 재설계)

### ✅ 레퍼런스 발전 방식 (마일스톤 기준, 본인도 동일 경로)

```
STEP02(2주): 설계문서(요구사항→마일스톤→시퀀스→상태→ERD→API) → Mock API → REST Docs → E2E
STEP03(3주): Mock을 진짜 도메인으로 교체. 잔액/쿠폰/상품조회. 도메인→단위테스트→Service→Controller
STEP04(4주): 주문/결제 + 인기상품. 통합테스트(Testcontainers) + 인덱스 최적화(보고서)
STEP05(5주): 동시성(낙관/비관락) + Filter/Interceptor
STEP06(6주): 분산락 + 캐시 + Redis 자료구조
STEP07: Facade 제거 → EDA + Saga
STEP08: Kafka + Outbox
STEP09: 부하테스트(K6) + 병목 개선
```

핵심 원리: **Mock-First**(골격 먼저 → 살 채움) + **기능 구현 직후 보고서/문서 즉시**(측정값 휘발 전).

---

## 2. 현재 진행 상황

### ✅ STEP03 전체 완료 (User / Balance / Coupon / Product 4개 도메인)

|STEP|작업|상태|
|---|---|---|
|STEP01|설계 기본|Done|
|STEP02|설계 심화 + Mock API|Done|
|STEP03|User 도메인 (#67)|Done|
|STEP03|잔액 Balance (#20, #68)|Done|
|STEP03|쿠폰 Coupon (#21, #69)|Done|
|STEP03|쿠폰/유저쿠폰 ERD 정렬 리팩토링 (#73 머지)|✅ Done|
|STEP03|**상품 Product 도메인 (#22 머지)**|✅ Done|

### ✅ 방금 완료: 상품(Product) 도메인 (`feat/step03-product-domain` → main 머지)

커밋 10개, 테스트 그린 후 PR #22 머지 (`Close #22`):

```
[FEAT] Product 도메인 모델 구현
[FEAT] Stock 도메인 모델 구현
[FEAT] Product/Stock 도메인 DTO(Info) 추가
[FEAT] Product/Stock Repository 구현
[FEAT] Product/Stock Service 구현
[FEAT] Product Controller/Response 구현 및 mock 정리
[FIX] Product 접근 제어 PROTECTED로 수정
[FEAT] ProductInfo.of 시그니처를 엔티티 기반으로 변경
[FIX] ProductSellingStatus 판매불가 상태 목록 정정
[TEST] Product/Stock 도메인 단위/통합 테스트 작성
```

구현 요약 (📌 결정):

- `Product`(id/name/price/sellStatus) + `Stock`(productId/quantity) 별도 도메인 (`product/domain/product`, `product/domain/stock`)
- `ProductSellingStatus`: HOLD/SELLING/STOP_SELLING + `cannotSelling()` + `forSelling()`
- Stock은 **조회 전용** (deduct/restore는 STEP04~05). Product `@Column(name="product_id")`, Stock `@Column(name="stock_id")`
- 응답 `data.products[]`(stock 포함). **Facade 생략** — ProductService가 StockService 주입해 직접 조합(단일 도메인, WIL week3 근거)
- `/ranks`(인기상품)는 제거 → STEP04로 (레퍼런스 마일스톤상 4주차부터)
- Service `@Transactional`은 STEP05 신호등이라 생략 (레퍼런스엔 `@Transactional(readOnly=true)` 있음)

### 다음 작업: 이슈 #23 — 주문/결제 (STEP04 시작)

상세는 9번 섹션(STEP04 미리보기). 진입 전 `git checkout main && git pull && git checkout -b feat/step04-order-payment`.

> ⚠️ STEP05 진입 시 1순위 처리: **Balance 큰 리팩토링** (`@Column(name=balance_id)` 누락 + `@OneToMany` 제거 + `@Version` + `Balance.create(userId)` 등). 가이드 12번 추적표. Balance는 STEP03 초반 코드라 아직 안 다듬어짐 — STEP05 #29에서 한 방에. EOF echo "1부 완료"

---

## 3. 채택 아키텍처 — 라이트 DDD (✅ WIL week3)

### 4가지 원칙

|#|원칙|코드 표현|
|---|---|---|
|1|불변 우선|`@Setter` 없음, 필드 `private`|
|2|생성은 정적 팩토리|`Xxx.create(...)` + 검증, `@NoArgsConstructor(PROTECTED)`|
|3|상태 변경은 의도 있는 메서드|`charge()`, `use()`, `issue()` (setter 금지)|
|4|검증은 도메인 내부|메서드 안에서 비즈니스 규칙 체크|

> ✅ WIL week3: "검증 로직이 코드의 80%. 도메인 객체 내부에 두는 것을 선호." ✅ WIL week2: "설계가 명확하면 코드는 목표 달성의 수단, 불명확하면 불필요한 노동" — 요구사항 구체화 + 확장성 고려가 설계의 핵심.

### 레이어 구조 (✅ WIL week3)

```
interfaces  →  application(Facade)  →  domain  ←  infrastructure
   ↑                  ↑                  ↑              ↑
Controller         Facade              Service     CoreRepository
Request/Response  Criteria/Result     Entity      JpaRepository
                                      Command/Info
                                      Repository(인터페이스)
```

- 도메인 레이어는 외부 레이어에 의존하지 않음 (DIP). Repository 인터페이스를 도메인이 정의, 구현체(`XxxCoreRepository`)는 infrastructure.

### DTO 변환 흐름

```
HTTP JSON → XxxRequest(interfaces) → [toCriteria] → XxxCriteria(application)
→ [toCommand] → XxxCommand(domain) → [도메인 처리] → XxxInfo(domain)
→ [of] → XxxResult(application) → [of] → XxxResponse(interfaces) → HTTP JSON
```

> Facade 생략 도메인(Product 등)은 application 레이어(Criteria/Result)를 건너뜀: Info → Response 직결.

### 패키지 네이밍 (✅ WIL week3)

|레이어|폴더|클래스|입력 DTO|출력 DTO|
|---|---|---|---|---|
|Presentation|`interfaces`|`XxxController`|`XxxRequest.Xxx`|`XxxResponse.Xxx`|
|Application|`application`|`XxxFacade`|`XxxCriteria.Xxx`|`XxxResult.Xxx`|
|Domain|`domain`|`XxxService`|`XxxCommand.Xxx`|`XxxInfo.Xxx`|
|Infrastructure|`infrastructure` + `/jpa/`|`XxxCoreRepository` + `XxxJpaRepository`|-|-|

### Facade 패턴 — "울며 겨자먹기" (✅ WIL week3)

- 여러 도메인 서비스 조합 시에만 조율자로 사용. **단일 도메인이면 생략**.
- Balance/Coupon은 userService 검증이 끼어 Facade 있음. **Product 조회는 단일이라 생략**(ProductService가 StockService 직접 주입).
- STEP07에서 EDA 전환하며 Facade 전면 제거.

### 도메인 vs JPA 엔티티 (✅ WIL week3)

- 이론상 분리가 맞으나 과제 제약으로 **엔티티=도메인 겸용**(Persistence-aware). 순수 DDD 아님.
- JPA 연관관계 최소화 — Stock이 Product를 `@ManyToOne` 안 걸고 `productId Long`으로 참조하는 게 그 예.

---

## 4. 작업 방식 (확립된 워크플로)

### 📌 세션 워크플로 (이슈 1개 = 1사이클)

```
브랜치 생성 → [경로+코드+설명] 단계별 (커밋 쪼갬) → compileJava/test 확인(본인 실행)
→ push → PR(작업내용+커밋해시) → 머지 → git checkout main → git pull → 다음 브랜치 → 반복
```

### 📌 Claude 작업 규칙

- **Claude는 코드를 직접 수정하지 않는다.** 읽기만 하고 "① 경로 → ② 코드 → ③ 왜 바꾸는지 설명" 제시. **타이핑은 본인이.** (직접 짜며 배우는 프로젝트)
- Claude는 git/gradlew 실행 불가. → 컴파일/테스트는 본인이 돌려 결과 공유, Claude가 분석.
- 커밋 전 `git status`로 staged 상태 확인 (트러블 #16 교훈).

### A. 한 PR 한 이슈

이슈1 = 브랜치1 = PR1. 브랜치명 `feat/step{NN}-{도메인}-{역할}` 또는 `refactor/...`. 사전 의존은 별도 PR 먼저.

### B. 도메인 작업 Step (Facade 생략 시 7단계)

```
1. 도메인 모델 (Entity, Enum)
2. 도메인 DTO (Command, Info)   ※조회만이면 Command 생략 가능
3. Repository (interface + Core + Jpa)
4. Service
5. Facade (Criteria, Result, Facade)  ※단일 도메인이면 생략
6. Controller + Request/Response (interfaces)
7. ApiControllerAdvice (IllegalArgumentException 핸들러 이미 있음)
8. 테스트 (Domain/Service/Facade/Controller) + Docs 테스트
```

각 단계 후 `./gradlew compileJava`.

### C. 정석 흐름 (#22부터 적용 중)

도메인 만들며 Docs 테스트 함께 / Always Green / 미완성은 @Disabled 명시 / mock 테스트는 정식 구현하며 갱신.

### D. 커밋 단위

작업 단위마다 add+commit (압축 금지). 도메인당 7~10개. **태그는 "건드린 게 뭐냐"로 가른다** — 본체 기능=FEAT, 본체 정정=FIX, 테스트=TEST, 구조변경=REFACTOR.

### E. 커밋 컨벤션

`[FEAT]` 새 기능 / `[REFACTOR]` 구조변경 / `[TEST]` 테스트 / `[FIX]` 버그·정정 / `[CHORE]` 빌드·설정 / `[DOCS]` 문서 / `[REVERT]` 되돌리기. 한 줄, 짧게. 상세는 PR 본문.

### F. 커밋 메시지 정정 (rebase)

`git rebase -i {기준}` → pick→reword. 합치기 squash. 막히면 `--abort`. 단독 브랜치면 `--force-with-lease`.

### 📌 PR 본문 형식 (확정 — 간결)

```
작업 내용

* {해시7자리} [태그] 설명
* ...

Close #{이슈}    ← 이슈 닫는 PR만
```

해시: `git push -u origin {브랜치}` → `git log --oneline -N` → 복사. GitHub 자동 링크.

---

## 5. 테스트 작성 패턴

### 5종 분류

|종류|도구|Spring|Mock|검증|
|---|---|---|---|---|
|도메인 단위|JUnit + AssertJ|안 띄움|없음|객체 행위|
|Service|JUnit + Mockito|안 띄움|Repository(+협력Service)|분기 흐름|
|Facade|JUnit + Mockito + InOrder|안 띄움|UserService + Service|협업 순서|
|Controller|JUnit + MockMvc|`@WebMvcTest`|Facade/Service `@MockitoBean`|HTTP+JSON+검증|
|Docs|JUnit + MockMvc + REST Docs|standalone|`Mockito.mock()`|API 문서|

### 케이스 수 가이드

도메인 5~7 / Service 6~8 / Facade 2~3 / Controller 4~6(검증 없으면 1~2) / Docs 정상 시나리오만. 지나치게 꼼꼼 X.

### 자주 쓰는 단축어

- `given(...).willReturn(...)` / `verify(repo, never()|times(1))` / `inOrder(A,B).verify(A)...`
- `assertThatThrownBy(() -> ...).isInstanceOf(...).hasMessage(...)`
- `mockMvc.perform(...).andExpect(jsonPath("$.code").value(200))`

### Controller 테스트 환경 (@WebMvcTest) — 현재 상태

```java
@WebMvcTest(controllers = {
    BalanceController.class, CouponController.class,
    ProductController.class,                       // ← #22에서 추가됨
    ApiControllerAdvice.class
})
public abstract class ControllerTestSupport {
    @Autowired protected MockMvc mockMvc;
    @Autowired protected ObjectMapper objectMapper;
    @MockitoBean protected BalanceFacade balanceFacade;
    @MockitoBean protected CouponFacade couponFacade;
    @MockitoBean protected ProductService productService;   // ← Product는 Facade 없어 Service
    // STEP04: OrderController/OrderFacade 추가 예정
}
```

### Docs 테스트 환경 (standalone)

```java
@ExtendWith(RestDocumentationExtension.class)
public abstract class RestDocsSupport {
    protected MockMvc mockMvc;
    protected ObjectMapper objectMapper = new ObjectMapper();
    @BeforeEach void setUp(RestDocumentationContextProvider provider) {
        this.mockMvc = MockMvcBuilders.standaloneSetup(initController())
            .apply(documentationConfiguration(provider))
            .setControllerAdvice(new ApiControllerAdvice()).build();
    }
    protected abstract Object initController();
}
```

- Controller Test = `@WebMvcTest` + `@MockitoBean` (Spring 부분 컨텍스트)
- Docs Test = `standaloneSetup()` + `Mockito.mock()` (Spring 안 띄움) → 자식이 Mock 직접 만들어 생성자 주입

```java
class ProductControllerDocsTest extends RestDocsSupport {
    private final ProductService productService = Mockito.mock(ProductService.class);
    @Override protected Object initController() { return new ProductController(productService); }
}
```

### 환경 설정

- `src/main/resources/application.yaml` 운영(MySQL+DataSource exclude) / `src/test/resources/application.yaml` 테스트(H2 MySQL 모드)
- build.gradle: `testRuntimeOnly h2`, `testImplementation spring-restdocs-mockmvc`, `asciidoctorExt spring-restdocs-asciidoctor`

---

## 6. 남은 GitHub 이슈

|STEP|이슈#|제목|
|---|---|---|
|STEP04|23|주문/결제 비즈니스 로직 구현 및 단위 테스트 ← **다음**|
|STEP04|24|인프라 레이어 구현체 작성|
|STEP04|25|기능별 통합 테스트 작성|
|STEP04|26|주요 기능별 동시성 실패 테스트 작성|
|STEP04|27|병목 예상 쿼리 분석 및 최적화 보고서 작성|
|STEP05|28~32|동시성 테스트/해결/보고서, Filter·Interceptor·Scheduler, 가용성|
|STEP06|33~39|분산락, 캐싱, 인기상품/선착순쿠폰 Redis, 각 보고서|
|STEP07|43~45|이벤트 기반 외부전송, 파사드 제거+EDA, MSA 설계문서|
|STEP08|46~50|카프카 개념문서, 메시지 발행, 대용량 트래픽, Outbox, 설계문서|
|STEP09|51~54|부하테스트 계획/스크립트/병목개선/보고서|

### 이슈 제목에 묻힌 큰 작업 (과소평가 주의)

|이슈|실제 작업|
|---|---|
|#29|**Balance 큰 리팩토링** (@Column 누락 + @OneToMany 제거 + balanceId Long + Balance.create(userId) + saveTransaction + @Version)|
|#31|**Rank 도메인 분리** + 배치 스케줄러|
|#44|Facade 제거 / BalanceClient 인터페이스 / refund+REFUND enum+Saga 보상|
|#48|인기상품 실시간 이벤트 (5번째 진화, 배치 제거)|

### 이슈로 안 잡힌 항목

|항목|시점|
|---|---|
|OrderControllerTest @Disabled 제거 + DocsTest 재작성|STEP04 #23 (지금)|
|WIL 회고|핵심메모는 그 시점, 정식은 STEP09 후|

---

## 7. STEP03 신호등 (참고 — STEP03은 완료)

STEP03에서 하지 말 것이었던 것들 (다음 STEP들에서 순차 도입): `@Version`/`@Lock`(STEP05), Service `@Transactional`(STEP05), `@Cacheable`/Redisson(STEP06), 이벤트(STEP07), Kafka(STEP08), `@Index`/커서페이징(STEP04), Rank/`/ranks`(STEP04~), Stock deduct/restore(STEP04~05). `BaseEntity`/`createdAt`은 레퍼런스에 아예 없음.

---

## 8. STEP04 상세 — 이슈 #23~27 (✅ report 01 + SequenceDiagram + WIL week4)

> ⚠️ STEP04 진입 시 정독: **01.DBPerformanceOptimizationReport** + **WIL week4** + **03-1.SequenceDiagram(주문결제 30단계)**.

### 작업 순서 (이슈 흐름)

1. **#23 주문/결제 도메인** — Order/Payment 도메인 모델 + 단위테스트
2. **#24 인프라 구현체** — JPA Repository 채우기, Repository 인터페이스 `@Repository`/`@Component`
3. **#25 통합테스트** — Testcontainers(`@SpringBootTest` + MySQL 컨테이너)
4. **#26 동시성 실패 테스트** — 락 없는 상태에서 깨지는 것 먼저 작성 (STEP05 준비)
5. **#27 DB 최적화 보고서** — 인덱스 + EXPLAIN ANALYZE, **보고서 먼저 → 코드** 순서

### 주문/결제 핵심 (✅ SequenceDiagram, WIL week2)

- 📌 **주문/결제 도메인 통합** 결정 (WIL week2 — 분리는 오버엔지니어링 판단). 단 차후 분리 가능성 열어둠.
- 주문 상태: CREATED → PAID (StateDiagram). 결제: READY/COMPLETED/FAILED.
- 주문 흐름: 잔액 확인 → 쿠폰 적용 → 재고 차감 → 주문 생성 → 결제 → 완료. (시퀀스 30단계가 청사진)
- **OrderFacade가 여러 도메인(Balance/Coupon/Product/Stock) 조합** → Facade 필수 (단일 아님).
- 📌 **`@Transactional`은 OrderFacade에** (Service 아님 — STEP04 신호등). Service @Transactional은 STEP05.
- Stock에 `deduct()`/`restore()` 추가 (주문이 재고 차감). 단 비관적 락은 STEP05.
- 모든 enum `@Enumerated(EnumType.STRING)`.

### ✅ report 01 인덱스 교훈 (그대로 베끼지 말 것 — 측정은 본인 환경에서)

- 잔액/재고/쿠폰: 단일·복합 인덱스로 99%+ 개선 — `user_id`, `product_id`, `(user_id,used_status)`, `(user_id,coupon_id)`
- **상품 조회는 인덱스 역효과** — `sell_status='SELLING'`이 90% → 카디널리티 낮아 풀스캔이 빠름 → **커서 페이징**(`product_id > cursor`)으로 0.14ms
- 인기상품: `(payment_status, paid_at)` 복합 + `order_id` 단일
- 측정값/스크린샷은 STEP04 끝나자마자 보고서로 (#27)

---

## 9. STEP05~09 미리보기 (✅ report + WIL + study 정독 기반)

### STEP05 — 동시성 (report 02, WIL week5)

✅ **자원별 일관 락 전략** (같은 자원엔 같은 전략 — WIL week5):

|자원|전략|이유|
|---|---|---|
|잔액|낙관적(`@Version`)|중복 충전/차감은 의도 아님|
|쿠폰|비관적(+STEP06 분산락)|선착순 모두 성공해야|
|재고|비관적|동시 차감 음수 방지|

- 락 기준: **반드시 성공해야 하면 비관적**, 아니면 낙관적. 범위는 최소.
- 동시성 유형: Race Condition(순서 의존), Lost Update(덮어쓰기). Java 해결책: synchronized(간단·공정성X), ReentrantLock(공정성 설정 가능), Atomic(CAS·락 없음·단순연산만).
- **Balance 큰 리팩토링** 여기서 (#29). Rank 도메인 분리, Filter/Interceptor 도입.
- 동시성 테스트: `ExecutorService`+`CompletableFuture`, `AtomicInteger`로 성공/실패 집계.
- 한계: 비관적 락도 공정성(순서) 보장 X → 선착순은 Kafka로 (STEP08).

### STEP06 — 분산락+캐시+Redis (report 02/03/04, WIL week5, study Cache)

**P1 분산락**: `@DistributedLock` + `DistributedLockAspect`(`@Order(HIGHEST_PRECEDENCE)` — 락 AOP가 트랜잭션보다 먼저). 전략패턴 `LockTemplate`(Simple/Spin/PubSub). PubSub=Redisson 기반.

- ✅ **분산락+트랜잭션 순서**: ①락획득(트랜잭션 밖) ②트랜잭션 시작 ③로직 ④커밋 ⑤락해제. `TransactionSynchronizationManager`로 afterCompletion에 해제.
- ✅ 어기면: (문제1) 트랜잭션 먼저→락이면 조회 시 락 미적용 Race + 커넥션 낭비. (문제2) 락 먼저 해제→커밋이면 미커밋 데이터 조회 정합성 깨짐.

**P2 캐시**: 인기상품 `@Cacheable`(Read Through), 배치 `@CachePut`(Write Through) 매일 00:05.

- ✅ **TTL 49h** 이유: 24h면 00:05 배치와 만료 겹침 + 배치 실패 시 hotfix 여유.
- ✅ 캐시 전략(study Cache): 읽기=Look Aside(상품상세)/Read Through(인기상품, 캐시웜업 필요). 쓰기=Write Around/Back/Through. 스탬피드 방지=웜업+Read Through.
- K6: 평균 54ms→2.1ms (~25배).

**P3 Redis 자료구조**: 인기상품 ZSET(`ZINCRBY rank:sell:{yyyyMMdd}`, `ZUNIONSTORE` 3일 합산, 5분 스케줄러 TOP5). 선착순쿠폰 ZSET(value=userId, score=요청시각, `addIfAbsent` 중복방지+순서, 1분 배치 DB insert, 5분 배치 FINISHED 처리). → CouponStatus에 `FINISHED` 추가.

### STEP07 — Facade 제거 + EDA (report 05)

✅ #44 = 3작업: ①Facade 제거(BalanceFacade/Criteria/Result 삭제) ②`BalanceClient` 인터페이스(도메인 격리) ③`refund`(+BalanceCommand.Refund, BalanceTransaction.ofRefund, REFUND enum).

- 주문 커밋 후 `OrderEvent.Created` 발행 → 잔액/쿠폰/재고 리스너 `@Async @TransactionalEventListener(AFTER_COMMIT)` 병렬 처리 → 결제 → 성공/실패.
- **Saga 보상**: 실패 시 잔액환불/쿠폰취소/재고복구 + 주문취소. Redis Hash로 프로세스 상태(PENDING/SUCCESS/FAILED). orderId 분산락으로 중복발행 방지.
- ⚠️ Spring ApplicationEvent는 단일 인스턴스 한계 → STEP08 Kafka.

### STEP08 — Kafka + Outbox (report 06, study Kafka)

- 선착순 쿠폰 **즉시 발급**으로 개선(배치 지연 제거). ✅ **파티션키=쿠폰ID**(동일 쿠폰 순차보장). 토픽 버저닝(v1→v2).
- **Outbox**: Auto(트랜잭션)=BEFORE_COMMIT 저장→AFTER_COMMIT(@Async) 발행. Manual(트랜잭션 없음)=즉시 저장+발행. Consumer 처리완료=eventId로 삭제, ack 수동커밋.
- 멱등성: 수신측 중복/발급개수/만료 재검증 (Inbox 패턴).
- ✅ study Kafka 개념: Broker/Producer/Consumer/Topic/Partition/ConsumerGroup/Rebalancing/Replication. **파티션-컨슈머 비율 3케이스**: 파티션>컨슈머(1컨슈머 다수파티션, throttling), =(1:1 최대성능), <(유휴 컨슈머, 장애복구 여유). DLQ는 `-dlt` 접미사.
- ✅ **인기상품 5번 진화**: ①DB집계(S03~04)→②배치(S05)→③Redis캐시(S06P2)→④ZSET(S06P3)→⑤실시간이벤트(S08).

### STEP09 — 부하테스트 (report 07)

- Actuator+Prometheus+Grafana+K6(+InfluxDB). 주문 Load 300VU(충전20%→주문10%), 선착순쿠폰 Peak 1000VU.
- SLA: p99<1s, 실패율 <1%(주문)/<5%(쿠폰). ✅ **Lag 방지**: `CoreException`(중복발급/재고부족 등 재시도 무의미) 시 `ack.acknowledge()`로 재처리 차단.
- 결과: 주문 p99 2.19s→405ms, 쿠폰 6.92s→531ms.

---

## 10. 문서/산출물 작성 타이밍

> 원리: **기능 구현 → 그 기능 보고서 즉시**(측정·스크린샷 휘발 전). 보고서 구조: 배경 → 대상 선정 → 문제분석(AS-IS) → 해결방안 → 측정/테스트(TO-BE) → 한계 → 결론.

|문서|트리거|STEP/이슈|
|---|---|---|
|01.DBPerformanceReport|인덱스+EXPLAIN ANALYZE 측정 후|STEP04 #27|
|02.ConcurrencyReport|락으로 동시성 해결+테스트 후|STEP05 #30|
|(분산락 보고서)|Redisson 분산락+테스트 후|STEP06 P1 #34|
|03.CacheStrategyReport|@Cacheable+K6 측정 후|STEP06 P2 #36|
|04.RedisDesignReport|ZSET 재설계 후|STEP06 P3 #39|
|05.MsaEventDrivenReport|Facade제거+EDA+Saga 후|STEP07 #45|
|06.KafkaDesignReport|Kafka+Outbox 후|STEP08 #50|
|07.LoadTestReport|K6+병목개선 후|STEP09 #51·54|
|study 01.Kafka.md|Kafka 도입 시작 시|STEP08 #46|
|WIL week2~5|핵심메모 그 시점 / 정식 STEP09 후|-|

### 📌 학습 회고 (devlog) 작성 규칙 — 본인 방식

> 레퍼런스는 `docs/WIL/weekN/README.md`에 **주차별로 1파일**(그 주 배운 것 전부)을 몰아 씀. 본인은 `docs/devlog/`에 **주제별로 1파일**을 씀 (주차가 아니라 주제 단위). 둘 다 유효 — 본인 방식이 검색·재사용에 유리.

**작성 타이밍** (핵심): 각 STEP **코드 머지 직후**, 그 STEP에서 새로 배운 개념·결정을 그 시점에 작성. 시간 지나면 "왜 그렇게 했는지"가 휘발됨. 코드 머지 + devlog 작성까지가 한 STEP의 마무리. (정식 종합 회고는 STEP09 후 몰아서.)

**기존 devlog 파일**:

- `이커머스 아키텍처 설계.md` — STEP02 설계 회고 (회고형)
- `DTO패턴이유.md` — STEP03 DTO 설계 결정 (결정+이유 표 형식, 짧고 담백)

**STEP03 devlog (작성 예정 — 2파일)**:

- `라이트DDD와4Layer.md` — 라이트 DDD 4원칙, 4-Layer 책임, DTO 변환 흐름, Facade 단일도메인 생략
- `도메인설계원칙.md` — 정적 팩토리+PROTECTED 생성자(생성 강제), 검증은 도메인 내부, `cannotXxx()` IN-list 헬퍼, ERD 정렬에서 배운 것(discountRate 의미/issue 3종 검증), Product-Stock 분리 이유

**작성 방식**: 회고는 본인 생각이 들어가야 의미 있으므로, Claude는 **다룰 주제 + 핵심 포인트(뼈대)만** 제시하고 본인이 본인 언어로 채움. (면접에서 본인 말로 설명할 수 있게.) `DTO패턴이유.md`처럼 **표(결정·이유) + 짧은 산문** 형식이 본인 톤.

**STEP별 devlog 트리거 요약**:

|STEP|머지 직후 쓸 devlog 주제(예시)|
|---|---|
|STEP03|라이트DDD/4Layer, 도메인설계원칙 (위 2파일)|
|STEP04|트랜잭션(@Transactional 위치/ACID), 인덱스(카디널리티/커서페이징), 주문결제 통합 결정|
|STEP05|동시성(낙관vs비관 기준), 분산락 순서, Balance 리팩토링 회고|
|STEP06|캐시 전략(Read Through/TTL 49h), Redis ZSET|
|STEP07|Facade→EDA 전환, Saga 보상|
|STEP08|Kafka(파티션키/Outbox/멱등성)|
|STEP09|부하테스트 회고 + 전체 종합 회고|

---

## 11. STEP별 신호등 (전체)

|작업|S03✓|S04|S05|S06|S07|S08|S09|
|---|---|---|---|---|---|---|---|
|도메인 비즈니스 메서드 / 4-Layer / Repo I/F분리 / 단위·Docs 테스트|O|O|O|O|O|O|O|
|파사드 클래스|O|O|O|O|X(제거)|X|X|
|`@OneToMany` cascade|O|O|X(제거)|X|X|X|X|
|`@Transactional` on Facade|X|O|O|O|X(제거)|X|X|
|`@Transactional` on Service|X|X|O|O|O|O|O|
|통합 테스트(Testcontainers) / 인덱스·커서페이징|X|O|O|O|O|O|O|
|Stock deduct/restore|X|O|O|O|O|O|O|
|Rank 도메인 분리 / `/ranks`|X|O|O|O|O|O|O|
|`@Version` 낙관 / `@Lock` 비관|X|X|O|O|O|O|O|
|Filter/Interceptor|X|X|O|O|O|O|O|
|배치 스케줄러|X|X(집계)|O|O|O|X(제거)|X|
|Redisson 분산락 / `@Cacheable` / ZSET|X|X|X|O|O|O|O|
|refund/BalanceClient/REFUND / Event / Saga|X|X|X|X|O|O|O|
|Kafka / Outbox|X|X|X|X|X|O|O|
|K6 / Prometheus+Grafana|X|X|X|X|X|X|O|
|`@Enumerated(STRING)`|O|O|O|O|O|O|O|

---

## 12. Balance 도메인 진화 추적표 (✅ report 02/05, WIL week5)

|항목|S03(현재)|S04|S05|S06|S07|S08|
|---|---|---|---|---|---|---|
|`@Column(name=balance_id)`|**없음(빚)**|없음|**추가**|있음|있음|있음|
|`Balance.create`|`(userId, amount)`|동일|`(userId)`|동일|동일|동일|
|`@OneToMany`|cascade|있음|**제거→balanceId Long**|없음|없음|없음|
|`@Version`|없음|없음|**추가**|있음|있음|있음|
|`@Index(user_id)`|없음|**추가**|있음|있음|있음|있음|
|`@Transactional`(Service)|없음|없음|**추가**|있음|있음|있음|
|`BalanceFacade`|있음|있음|있음|있음|**삭제**|없음|
|`BalanceClient` / `refund` / `REFUND`|없음|없음|없음|없음|**추가**|있음|
|`saveTransaction`|없음|없음|**추가**|있음|있음|있음|

> ✅ 레퍼런스 최종 Balance: MAX=10_000_000, INIT=0, charge/use/refund 검증, @Version, @Index. BalanceService는 balanceClient.getUser() + 모든 메서드 @Transactional. ⚠️ 현재 본인 Balance엔 `@Column(name=balance_id)`도 없음(STEP03 초반 코드). STEP05 #29에서 한 번에 정리.

## 13. Coupon 도메인 진화 추적표 (✅ ERD 정렬로 S03 확정)

|항목|S03(정렬 후)|S05|S06|S08|
|---|---|---|---|---|
|상태값|REGISTERED/PUBLISHABLE/CANCELED|동일|+FINISHED|동일|
|discountRate|double(0~1)|동일|동일|동일|
|quantity|int 하나(`quantity--`)|동일|동일|동일|
|발급 검증|상태+만료+수량|+중복발급(UK)|동일|동일|
|발급 동시성|없음|비관적 락|+분산 락|Kafka 직렬(파티션키=쿠폰ID)|
|중복발급 방지|없음|UK(user_id,coupon_id)|Redis addIfAbsent|Kafka 멱등성|

> ✅ 레퍼런스는 `publish()`+`CoreException`. 본인은 `issue()`+`IllegalArgumentException`(의도된 차이).

## 14. Product/Stock 도메인 진화 추적표 (✅ #22로 S03 확정, NEW)

|항목|S03(현재)|S04|S05|S06|
|---|---|---|---|---|
|Product sellStatus|HOLD/SELLING/STOP_SELLING|동일|동일|동일|
|`cannotSelling()`/`forSelling()`|있음|있음|있음|있음|
|Stock deduct/restore|**없음(조회전용)**|**추가**(주문 차감)|+비관적 락|동일|
|Stock 비관적 락|없음|없음|**`findByProductIdWithLock`**|있음|
|Product 조회|전체(findBySellStatusIn)|**커서 페이징**|동일|+캐시|
|`@Index`|없음|**추가**(product_id 등)|있음|있음|
|Rank(인기상품)|**없음**|**별도 Rank 도메인 신설**(/ranks)|배치|Redis ZSET|
|Service `@Transactional`|없음|(readOnly 검토)|추가|있음|

> 인기상품 5번 진화의 출발점이 STEP04 Rank 도메인. Product/Stock 자체는 비교적 안정적, Stock만 락이 붙으며 진화.

---

## 15. 트러블슈팅 — 이미 겪은 것

1. **contextLoads 실패** — DataSource 빈 못 찾음. 해결: test 전용 yaml + H2 (적용됨)
2. **Service 테스트 메서드 오호출** — 단일 통과/전체 실패. 테스트 대상 메서드 정확히 호출 확인
3. **DocsTest 컴파일 실패** — 본체에 생성자 의존성 추가됐는데 Docs는 옛 `new XxxController()`. 본체 바꿀 때 테스트 동기화
4. **커밋 압축** — `git add .` 두 번으로 단계 뭉침. `git reset --soft`로 재커밋. 작업 시작부터 단위 의식
5. **git restore 실수** — 마지막 커밋 상태로 되돌림. 의도 안 한 파일이면 두기
6. **@WebMvcTest controllers에서 빼면 상속 테스트 깨짐** — @Disabled + 사유 명시
7. **검증 `<` vs `<=`** — "양수/0보다 커야"는 `<= 0`(0 막음), "0 이상"은 `< 0`(0 허용). Product 가격=`<=0`, Stock 수량=`<0`이 그 예
8. **예외 타입 불일치** — `IllegalStateException`→500. ApiControllerAdvice는 `IllegalArgumentException`만 잡음. 도메인 검증은 IllegalArgumentException로 통일
9. **Rebase로 메시지 정정** — `rebase -i`→reword. 합치기 squash. `--abort`/`--force-with-lease`
10. **Docs Mock 주입 차이** — `@MockitoBean`이 standalone에서 안 통함. `Mockito.mock()` 직접 생성자 주입
11. **빈 파일 머지 사고** — Docs가 0줄 머지(본문은 working tree만). `git diff --cached`로 커밋 전 확인
12. **테스트가 못 잡은 버그** — userId==couponId(둘 다 1L)라 ID 바꿔치기 못 잡음 / of() 매핑 누락인데 assert 없어 못 잡음. 데이터 서로 다른 값, 모든 필드 assert
13. **핸드오프 전제와 실제 코드 불일치** — 문서가 "엔티티 반영 후"라 했으나 미반영. 세션 시작 시 **실제 코드 먼저 확인**. ✅/⚠️ 표기로 추측·사실 구분
14. **(NEW) enum 상태 목록 뒤집힘** — `cannotSelling()`의 IN-list에 SELLING을 넣고 STOP_SELLING 누락 → 로직 정반대. 도메인 단위 테스트(true/false 양쪽)가 잡음. 교훈: IN-list 헬퍼는 **올바른 원소** 검증을 양방향으로
15. **(NEW) 새 엔티티 @NoArgsConstructor PROTECTED 누락** — 타이핑 중 `(access=PROTECTED)`+import 빠진 채 커밋. public도 컴파일돼서 안 걸림. 교훈: 새 엔티티는 기존(Coupon 등)과 생성자 접근제어 대조
16. **(NEW) DTO of() 시그니처와 Service 호출 불일치** — Info.of()를 필드나열형으로 만들었는데 Service는 엔티티 받는 형태로 호출. staged/unstaged 섞여 추적 어려움. 교훈: 커밋 전 `git status`로 staged 확인

---

## 16. 막힐 때 명령

```
git --no-pager show <hash>:<path>                                # 특정 커밋 시점 파일
git --no-pager log --all --oneline --reverse -S "<keyword>" -- "*.java"   # 키워드 검색
git diff --no-index <본인> <레퍼런스>                            # 비교
Get-ChildItem -Recurse "...e-commerce-reference\service\{도메인}\..." | Select-Object FullName
Get-ChildItem -Recurse "...e-commerce\src\main\java\...\{도메인}" | Select-Object FullName
```

> 레퍼런스는 멀티모듈 최종본. 문서=방향, 코드=해당 STEP 커밋 시점.

---

## 17. 면접/이력서 어필 자산

라이트 DDD(Persistence-aware) / 4-Layer+DIP / Facade→EDA 전환(S03→S07) / Repository로 영속성 격리 / Static Factory+Builder 캡슐화 / DTO 변환 흐름 / Test-After+Always Green / Given-When-Then+Living Docs(REST Docs) / 낙관·비관·분산락(순서: 락→트랜잭션→커밋→락해제) / 캐시(TTL 49h)·ZSET / Kafka·Saga·Outbox·멱등성 / 인기상품 5번 진화 / 부하테스트 병목개선(주문 p99 2.19s→405ms).

강한 무기: ①라이트 DDD ②Facade→EDA 전환 ③분산락+트랜잭션 순서 ④인기상품 5번 진화 ⑤Outbox+Kafka 멱등성.

### TDD vs 본인 방식

- TDD: 빨강→초록→리팩토링, 테스트가 본체 설계 이끔.
- 본인: 본체 먼저→테스트 나중→본체 변경 시 테스트 같이 갱신(Always Green). **TDD 아님, 현업 다수 방식.**

---

## 부록 A. 개념 학습 노트 (✅ WIL week5 / study Kafka·Cache 정독분)

> 해당 STEP 진입 시 보고서 쓸 때 활용. 깊은 디테일은 원문 재정독.

### A-1. 동시성 (WIL week5)

- 유형: Race Condition(순서 의존 결과 달라짐), Lost Update(덮어쓰기). 격리수준 문제: Dirty/Non-Repeatable/Phantom Read.
- Java: synchronized(간단·무한대기·공정성X) / ReentrantLock(공정성·타임아웃·세밀제어) / Atomic(CAS·락없음·단순연산).
- DB락: 쓰기=배타락(X-Lock). MVCC로 쓰기 중에도 이전 버전 읽기.
- 낙관(@Version, 충돌 시 OptimisticLockingFailureException→@Retryable 재시도) / 비관(@Lock(PESSIMISTIC_WRITE), 조회 시점부터 락).
- 기준: 반드시 성공해야 하면 비관, 아니면 낙관. 범위 최소. 같은 자원 같은 전략.
- 분산락: Redis SET NX 원자성. 전용 인스턴스 권장. Simple/Spin/PubSub. Spin=setIfAbsent 루프+Lua unlock. PubSub=Redisson tryLock. 해제는 TransactionSynchronizationManager afterCompletion.

### A-2. Kafka (study Kafka)

- Broker(서버), Producer(발행, key 라우팅), Consumer(소비, offset 관리: Current/Committed), Topic(논리), Partition(물리·순서보장·병렬), Consumer Group(파티션 1개=컨슈머 1개, 부하분산), Rebalancing(소유권 변경), Cluster, Replication(Leader/Follower).
- 파티션-컨슈머 비율: >(1컨슈머 다수파티션, throttling로 DB부하 조절) / =(1:1 최대성능) / <(유휴 컨슈머, 장애복구 여유).
- 트랜잭셔널 메시징: Outbox(이벤트를 테이블에 먼저 저장 후 별도 프로세스가 발행) / CDC. 중복→멱등성(Inbox). 실패→DLQ(`-dlt`).

### A-3. 캐시 (study Cache)

- 읽기: Look Aside(캐시 먼저, 없으면 DB→캐시 저장, 장애 유연, 상품상세) / Read Through(캐시만, 미스 시 장애 위험, 웜업 필요, 인기상품).
- 쓰기: Write Around(DB만, 미스 시 캐시, 정합성 약) / Write Back(캐시 먼저 후 배치 DB, 쓰기부하↓ 유실위험) / Write Through(동시 저장, 정합성 보장 리소스↑).
- 캐시 웜업(미리 적재)으로 스탬피드(미스 시 DB 부하 급증) 방지. 무효화: TTL / 명시적. 만료정책 LRU/LFU/FIFO. 로컬 vs 글로벌(Redis).

---

## 부록 B. 다음 세션 시작 안내 문구

```
mini-commerce 프로젝트 진행할거야.

STEP03 전부 끝나서 main에 머지됐고(User/Balance/Coupon/Product),
지금은 STEP04 이슈 #23 (주문/결제) 차례야.
git checkout main && git pull 후 feat/step04-order-payment 브랜치 생성.

가이드 8번 섹션(STEP04 상세)에 작업순서/주문결제 핵심/인덱스 교훈 있어.
STEP04 진입이니 먼저 정독: 01.DBPerformanceReport + WIL week4 + 03-1.SequenceDiagram(주문결제 30단계).

정석 흐름(Docs 테스트 함께, Always Green, 커밋 쪼개기) + 경로+코드+설명 방식.
(Claude는 코드 직접 수정 X, 타이핑은 내가. 커밋 전 git status 확인.)

⚠️ Order/Payment 도메인 구조 + OrderFacade @Transactional 위치부터 합의하고 시작하자.
먼저 레퍼런스 order 구조랑 본인 현재 order mock 상태 확인부터.
```

> 이 문서는 살아있는 문서. STEP 진행하며 발견하는 것 추가/갱신. 검증 표기(✅/⚠️/📌) 유지.