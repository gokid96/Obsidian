
> 레퍼런스 문서 19종 + 도메인 코드를 **실제로 정독·검증**하여 재작성한 핸드오프 문서. 다음 세션에서 바로 이어 작업할 수 있도록 컨텍스트 + 작업 방식 + 진행 가이드를 정리.
> 
> **검증 표기**: ✅ = 레퍼런스 문서/코드로 직접 확인함 · ⚠️ = 부분 확인(다음 세션 보강 권장) · 📌 = 결정사항


| 도메인            | 의미              | 핵심 필드/역할                                               |
| -------------- | --------------- | ------------------------------------------------------ |
| **User**       | 유저              | nickname                                               |
| **Balance**    | 유저의 **잔액(돈)**   | amount(잔액) + BalanceTransaction(충전/사용 이력)              |
| **Coupon**     | **쿠폰 정책(원본)**   | 할인율, 수량, 만료일, 상태. "어떤 쿠폰이 존재하는가" (관리자가 등록)             |
| **UserCoupon** | **유저가 발급받은 쿠폰** | userId + couponId + 사용여부(UNUSED/USED). "누가 어떤 쿠폰을 가졌나" |
| **Product**    | **상품**          | 이름/가격/판매상태 (수량은 여기 없음!)                                |
| **Stock**      | **재고**          | productId + quantity. Product와 분리된 별도 테이블              |
| **Order**      | 주문              | (STEP04에서 구현)                                          |

---

## 0. 이 문서의 검증 범위 (재작성 세션 기록)

이번 재작성에서 **실제로 읽고 검증한 것**:

- ✅ architecture 전부: 01.Requirements / 02.Milestones / 03-1.SequenceDiagram / 03-2.StateDiagram / 04.ERD / 05.ApiDocument
- ✅ WIL week3 (클린 레이어드 아키텍처 — 라이트 DDD의 직접 근거)
- ✅ WIL week4 (트랜잭션/인덱스 학습노트)
- ✅ report 전부 7종: 01.DBPerformance / 02.Concurrency / 03.CacheStrategy / 04.RedisDesign / 05.MsaEventDriven / 06.KafkaDesign / 07.LoadTest
- ✅ 도메인 코드 패턴: Balance, Coupon, Product, Stock (레퍼런스 최종본 + 본인 현재 상태)

**아직 정독 못 해 다음 세션 보강 필요**:

- ⚠️ WIL week5 (동시성/분산락 학습노트) — 단, report 02에서 상당 부분 커버됨
- ⚠️ study 01.Kafka.md — 단, report 06에서 상당 부분 커버됨
- ⚠️ study 02.Cache.md — 단, report 03에서 상당 부분 커버됨
- ⚠️ WIL week2 (설계 회고)

> 06.SpringRestDocs.md는 REST Docs 스크린샷 인덱스라 선택. 본인 06 산출물은 미작성.

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
|`org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest`|`org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`|
|`@MockBean`|`@MockitoBean` (`org.springframework.test.context.bean.override.mockito.MockitoBean`)|

### ⚠️ 레퍼런스는 멀티모듈 "최종본"이다 (중요)

레퍼런스(`e-commerce-reference`)는 STEP09까지 다 끝낸 **멀티모듈 최종본**이다.

- `service/{도메인}/...` 구조 (본인은 단일 모듈 `src/main/java/...`)
- 패키지 루트가 `kr.hhplus.be.ecommerce`
- Rank/Redis/Kafka/Querydsl/Scheduler 등 후속 STEP 코드가 전부 섞여 있음

따라서 **문서는 "방향" 참고, 코드는 "현재 STEP"에 맞는 부분만** 잘라 써야 한다. (쿠폰만 해도 STEP03 → STEP05 → STEP06에서 여러 번 재설계됨)

---

## 2. 현재 진행 상황

### 완료된 것

|STEP|작업|상태|
|---|---|---|
|STEP01|설계 기본|Done|
|STEP02|설계 심화 + Mock API|Done|
|STEP03|User 도메인 (#67)|Done|
|STEP03|잔액 Balance (#20, #68)|Done|
|STEP03|쿠폰 Coupon (#21, #69)|Done|
|STEP03|REST Docs 빈 파일 머지 사고 복구|Done|
|STEP03|쿠폰 버그 2건 수정 + 회귀테스트|Done|
|STEP03|**쿠폰/유저쿠폰 ERD 정렬 리팩토링** (PR 머지 완료)|✅ Done|

### ✅ 방금 완료: 쿠폰/유저쿠폰 ERD 정렬 (`refactor/align-coupon-erd` → main 머지)

커밋 6개로 진행, `BUILD SUCCESSFUL` + 테스트 통과 확인 후 PR 머지:

1. `[REFACTOR]` CouponStatus 상태값: `AVAILABLE/UNAVAILABLE` → `REGISTERED/PUBLISHABLE/CANCELED` + `cannotPublishable()` 헬퍼
2. `[REFACTOR]` Coupon 엔티티: `discountAmount(long)` → `discountRate(double, 0~1)`, 수량모델 `totalQuantity+issuedQuantity` → `quantity` 하나, `expiredAt` 추가, `@Column(name="coupon_id")`. `issue()` = 상태→만료→수량 3종 검증
3. `[REFACTOR]` UserCoupon 엔티티: `issuedAt`/`usedAt` 추가, `@Column(name="user_coupon_id")`, `@NoArgsConstructor(PROTECTED)`. `use()`에서 `usedAt` 세팅
4. `[REFACTOR]` User: `userName` → `nickname` (User/UserInfo/UserService 연쇄)
5. `[REFACTOR]` Coupon DTO 연쇄: CouponInfo/CouponResult/CouponResponse `discountAmount` → `discountRate`
6. `[TEST]` 위 변경에 맞춰 테스트 갱신 (CouponTest 재작성, ServiceTest/FacadeTest/ControllerTest/DocsTest 갱신)

> 📌 할인율은 **0~1** 방식 (10% = 0.1). 레퍼런스 따름. 검증은 `discountRate < 0 || > 1`. 📌 메서드명 `issue()` 유지(레퍼런스는 `publish()`), 예외는 전부 `IllegalArgumentException`(본인 ApiControllerAdvice가 이것만 잡음, 트러블 #8). 📌 중복 발급 검증 → STEP05 / UserCoupon EXPIRED·CANCELED → STEP07 / CouponStatus FINISHED → STEP06 으로 미룸.

### 다음 작업: 이슈 #22 — 상품(Product) 도메인 (STEP03 마지막)

- 브랜치명: `feat/step03-product-domain` (이미 생성됨, main pull 후 분기)
- ✅ 레퍼런스 마일스톤으로 범위 확정: **3주차(STEP03)는 "상품 조회"만**. 인기상품(`/ranks`)은 **4주차(STEP04)부터** ("상위 상품 조회 및 배치 스케줄러").
- 정석 흐름: Docs 테스트 함께 작성, Always Green, 커밋 단위 쪼개기
- Product 엔티티(name/price/sellStatus) + Stock 도메인(조회 전용) 동시 구현
- 패키지: `product/domain/product/`, `product/domain/stock/` 분리 (A 방식)
- 응답 `data.products[]` 래핑(stock 포함), Facade 생략, 인기상품(/ranks)은 제외
- 자세한 구현 계획은 8번 섹션 참조.

---

## 3. 채택 아키텍처 — 라이트 DDD (✅ WIL week3 근거)

### 4가지 원칙

|#|원칙|코드 표현|
|---|---|---|
|1|불변 우선|`@Setter` 없음, 필드 `private`|
|2|생성은 정적 팩토리|`Xxx.create(...)` + 검증|
|3|상태 변경은 의도 있는 메서드|`charge()`, `use()`, `issue()` (setter 금지)|
|4|검증은 도메인 내부|메서드 안에서 비즈니스 규칙 체크|

> ✅ WIL week3: "검증 로직이 코드의 80%. 도메인 객체 내부에 두는 것을 선호" — 원칙 4의 직접 근거.

### 레이어 구조 (✅ WIL week3)

```
interfaces  →  application(Facade)  →  domain  ←  infrastructure
   ↑                  ↑                  ↑              ↑
Controller         Facade              Service     CoreRepository
Request/Response  Criteria/Result     Entity      JpaRepository
                                      Command/Info
                                      Repository(인터페이스)
```

- 도메인 레이어는 **어떤 외부 레이어도 의존하지 않는다** (DIP, 핵심).
- Repository 인터페이스를 **도메인이 정의**, 구현체(`XxxCoreRepository`)는 infrastructure.

### DTO 변환 흐름

```
HTTP JSON → XxxRequest(interfaces) → [toCriteria] → XxxCriteria(application)
→ [toCommand] → XxxCommand(domain) → [도메인 처리] → XxxInfo(domain)
→ [of] → XxxResult(application) → [of] → XxxResponse(interfaces) → HTTP JSON
```

> ✅ WIL week3: "레이어별 DTO 분리는 오버엔지니어링처럼 보이나, 레이어 간 결합도를 낮추는 완충제. API 스펙 변경이 도메인에 전파되지 않게 한다."

### 패키지 네이밍 (✅ WIL week3)

|레이어|폴더|클래스|입력 DTO|출력 DTO|
|---|---|---|---|---|
|Presentation|`interfaces`|`XxxController`|`XxxRequest.Xxx`|`XxxResponse.Xxx`|
|Application|`application`|`XxxFacade`|`XxxCriteria.Xxx`|`XxxResult.Xxx`|
|Domain|`domain`|`XxxService`|`XxxCommand.Xxx`|`XxxInfo.Xxx`|
|Infrastructure|`infrastructure` + `infrastructure/jpa/`|`XxxCoreRepository` + `XxxJpaRepository`|-|-|

### Facade 패턴 — "울며 겨자먹기" (✅ WIL week3)

- 여러 도메인 서비스를 조합해야 할 때만 중간 조율자로 사용.
- **단일 도메인 서비스만 쓰면 Facade를 만들지 않는다** (불필요한 파일만 늘어남).
- → Product 조회는 단일 도메인이라 **Facade 생략** (Q3 결정의 근거).
- STEP07에서 EDA 전환하며 Facade 제거 예정.

### 도메인 클래스 vs JPA 엔티티 (✅ WIL week3)

- 이론상 분리가 바람직하나, 레퍼런스는 과제 제약("코치보다 코드 잘 짜야 분리 가능")으로 **엔티티=도메인 겸용**.
- → 본인 라이트 DDD도 **Persistence-aware (겸용)** 방식. 순수 DDD 아님.
- JPA 연관관계는 최소화 (애그리거트 내부 / 생명주기 동일 / 루트 애그리거트 간만 허용).

---

## 4. 작업 방식 (확립된 워크플로)

### 📌 세션 워크플로 (이슈 1개 = 1사이클)

```
브랜치 생성
→ [경로 + 코드 + 설명] 8 Step (커밋 단위로 쪼갬)
→ compileJava / test 확인 (본인이 실행, 결과 공유)
→ push
→ PR (작업 내용 + 커밋 해시 목록)
→ 머지
→ git checkout main → git pull
→ 다음 브랜치 생성 → 반복
```

### 📌 Claude 작업 규칙 (이 세션에서 확립)

- **Claude는 코드를 직접 수정하지 않는다.** 파일은 읽기만 하고, "경로 + 코드 + 왜 바꾸는지 설명"을 제시. **타이핑은 본인이.** (이유: 직접 짜면서 배우는 프로젝트. Claude가 고치면 학습 과정이 사라짐.)
- Claude는 `git`/`./gradlew`를 실행할 수 없음. → 컴파일/테스트는 본인이 돌려 결과 공유, Claude가 분석.
- 제안 형식: **① 파일 경로 명시 → ② 코드 블록 → ③ 코드 아래에 "뭘 왜 바꿨는지" 설명.**

### A. 한 PR 한 이슈

- 이슈 1개 = 브랜치 1개 = PR 1개
- 브랜치명: `feat/step{NN}-{도메인}-{역할}` 또는 `refactor/...`
- 사전 의존(User 도메인 등)은 별도 PR로 먼저 머지

### B. 도메인 작업 8 Step

```
1. 도메인 모델 (Entity, Enum)
2. 도메인 DTO (Command, Info)
3. Repository (interface + Core + Jpa)
4. Service
5. Facade Layer (Criteria, Result, Facade)  ※단일 도메인이면 생략
6. Controller + Request/Response (interfaces)
7. ApiControllerAdvice (IllegalArgumentException 핸들러 이미 있음)
8. 테스트 (Domain / Service / Facade / Controller) + Docs 테스트
```

각 Step 후 `./gradlew compileJava`로 컴파일 확인.

### C. 정석 흐름 (이슈 #22부터 적용)

```
- 도메인 만들면서 Docs 테스트도 같이 작성
- 항상 모든 테스트 통과 상태 유지 (Always Green)
- 미완성 테스트는 @Disabled 명시
- STEP02 잔존 mock 테스트는 정식 구현하며 갱신
```

### D. 커밋 단위

작업 단위마다 add+commit (한 번에 압축 금지). 도메인당 7~8개 권장.

```
[FEAT] {도메인} 도메인 모델 구현
[FEAT] {도메인} 도메인 DTO(Command, Info) 추가
[FEAT] {도메인} Repository 구현
[FEAT] {도메인} Service 구현
[FEAT] {도메인} Facade 구현          ※단일 도메인이면 생략
[REFACTOR] {도메인} 인터페이스 패키지 구조 변경
[TEST] {도메인} 도메인 단위/통합 테스트 작성
```

### E. 커밋 메시지 컨벤션

|태그|사용처|
|---|---|
|`[FEAT]`|새 기능|
|`[REFACTOR]`|개선/구조 변경|
|`[TEST]`|테스트|
|`[FIX]`|버그/정정|
|`[CHORE]`|빌드/설정/환경|
|`[DOCS]`|문서|
|`[REVERT]`|되돌리기|

메시지는 한 줄, 짧게. 상세는 PR 본문에.

### F. 커밋 메시지 정정 (rebase)

```
git rebase -i {기준 커밋}   # pick → reword 로 변경
# 중복 커밋 합치기는 squash. 막히면 git rebase --abort
# 단독 브랜치면 git push --force-with-lease 안전
```

### 📌 PR 본문 형식 (확정 — 간결하게)

```
작업 내용

* {해시7자리} [FEAT] {도메인} 도메인 모델 구현
* {해시7자리} [FEAT] {도메인} 도메인 DTO 추가
* ... (커밋별로)

Close #{이슈번호}    ← 이슈 닫는 PR만
```

해시 확보: `git push -u origin {브랜치}` → `git log --oneline -N` → 7자리 복사. GitHub가 자동 링크.

---

## 5. 테스트 작성 패턴

### 5종 분류

|종류|도구|Spring|Mock|검증 대상|
|---|---|---|---|---|
|도메인 단위|JUnit + AssertJ|안 띄움|없음|객체 자체의 행위|
|Service|JUnit + Mockito|안 띄움|Repository|분기 흐름|
|Facade|JUnit + Mockito + InOrder|안 띄움|UserService + Service|협업 순서|
|Controller|JUnit + MockMvc|`@WebMvcTest`|Facade `@MockitoBean`|HTTP 라우팅 + JSON + 검증|
|Docs|JUnit + MockMvc + REST Docs|standalone|Facade `Mockito.mock()`|API 문서 자동 생성|

### 케이스 수 가이드

- 도메인 단위: 비즈니스 메서드당 happy + edge 짝지어 5~7개
- Service: 분기마다(있음/없음) 짝지어 6~8개
- Facade: InOrder 검증 핵심만 2~3개
- Controller: 정상 + 검증 실패 4~6개
- Docs: 정상 시나리오만 (검증 실패는 Controller Test에서 다룸)

지나치게 꼼꼼 X (collection 검증 등은 후속 STEP 리팩토링 때 깨짐).

### 자주 쓰는 단축어

- `given(...).willReturn(...)` — BDDMockito
- `verify(repo, never())` / `verify(repo, times(1))` — 호출 검증
- `inOrder(A, B).verify(A); inOrder.verify(B)` — 순서 검증
- `assertThatThrownBy(() -> ...).isInstanceOf(...).hasMessage(...)` — 예외 검증
- `mockMvc.perform(...).andExpect(jsonPath("$.code").value(200))` — HTTP 검증

### Controller 테스트 환경 (@WebMvcTest)

```java
@WebMvcTest(controllers = {
    BalanceController.class,
    CouponController.class,
    ApiControllerAdvice.class
    // ← 새 Controller 추가 시 여기 추가 (Product 작업 시 ProductController 추가)
})
public abstract class ControllerTestSupport {
    @Autowired protected MockMvc mockMvc;
    @Autowired protected ObjectMapper objectMapper;   // tools.jackson.databind (Boot4)
    @MockitoBean protected BalanceFacade balanceFacade;
    @MockitoBean protected CouponFacade couponFacade;
    // ← 새 Facade 추가 시 여기에 (단, Product는 Facade 없음 → ProductService를 MockitoBean)
}
```

### Docs 테스트 환경 (standalone)

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
    @Override protected Object initController() {
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

## 6. 다음 작업 — 이슈 #22 (Product 도메인) 상세

### ✅ 범위 확정 (레퍼런스 마일스톤 + 요구사항으로 검증)

- ✅ Requirements: "상품 조회 = 상품 정보(ID/이름/가격/재고수) 목록 조회. 재고는 재고 테이블에서. **판매가능 상태만** 조회."
- ✅ Milestones: 3주차 = "상품 조회"만. 인기상품은 4주차.
- ✅ SequenceDiagram(상품 조회): 클라이언트 → 상품 조회 → 재고 조회 → 판매가능 상품 반환.
- ✅ ApiDocument: `GET /api/v1/products` → `data.products[]` (id/name/price/stock).
- ✅ StateDiagram: 상품 판매상태 = `HOLD`(판매보류) / `SELLING`(판매중) / `STOP_SELLING`(판매중지).

### 📌 결정사항

- Q1 = (a) **Stock 조회 전용** (deduct/restore는 STEP04~05에서)
- Q2 = **예** 응답 `data.products[]`에 재고수량(stock) 포함
- Q3 = **예** Facade 생략, Controller가 ProductService 직접 호출

### 진입 전 준비

```
git checkout main
git pull
git checkout -b feat/step03-product-domain   # 이미 만들었다면 생략
```

### 만들 최종 파일 트리

```
product/
├── domain/
│   ├── product/
│   │   ├── Product.java               # 엔티티: id/name/price/sellStatus + create검증 + cannotSelling()
│   │   ├── ProductSellingStatus.java  # HOLD/SELLING/STOP_SELLING + cannotSelling() + forSelling()
│   │   ├── ProductInfo.java           # Products / Product (productId/name/price/quantity)
│   │   ├── ProductRepository.java     # 인터페이스: save / findBySellStatusIn
│   │   └── ProductService.java        # getSellingProducts → stock 묶어서 반환
│   └── stock/
│       ├── Stock.java                 # 엔티티: id/productId/quantity + create검증 (deduct/restore 제외)
│       ├── StockInfo.java             # Stock (stockId/quantity)
│       ├── StockRepository.java       # 인터페이스: save / findByProductId
│       └── StockService.java          # getStock 조회 전용
├── infrastructure/
│   ├── ProductCoreRepository.java
│   ├── StockCoreRepository.java
│   └── jpa/
│       ├── ProductJpaRepository.java
│       └── StockJpaRepository.java
└── interfaces/
    ├── ProductController.java         # GET /api/v1/products → getSellingProducts
    └── dto/
        └── ProductResponse.java       # Products / Product (id/name/price/stock)

[삭제/이동] product/controller/ProductController.java (옛 mock 위치 → interfaces로)
[삭제/이동] product/dto/response/ProductResponse.java (옛 위치 → interfaces/dto로)
[삭제] product/dto/response/ProductRankResponse.java (ranks = STEP04로 미룸)
```

> ⚠️ ranks(인기상품) 관련(엔드포인트 + ProductRankResponse + 테스트 2개)은 제거. STEP04~05에서 Rank 도메인으로 정식 구현. 지금 두면 어차피 갈아엎음 (신호등).

### 도메인 간 협력 (📌 (가) 방식)

응답에 stock 포함이므로 `ProductService.getSellingProducts()`가 product + stock 조합.

- (가) `ProductService`가 `StockRepository`(or StockService)를 주입받아 직접 합침 → Facade 없이 단일 Service에서 끝.
- 같은 `product` 패키지 하위 도메인 간 의존이라 허용 범위.
- 레퍼런스는 상위(주문 facade)에서 조합하지만 그건 STEP04 주문 생기면서. STEP03 단독 조회는 (가)가 간결.

### 레퍼런스 Product 엔티티 형태 (✅ STEP03용으로 잘라낸 것)

```java
@Getter @Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Product {
    @Id @Column(name = "product_id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private long price;
    @Enumerated(EnumType.STRING)
    private ProductSellingStatus sellStatus;

    @Builder private Product(...) { ... }

    public static Product create(String name, long price, ProductSellingStatus sellStatus) {
        // name 공백 검증, price <= 0 검증, sellStatus null 검증
    }
    public boolean cannotSelling() { return sellStatus.cannotSelling(); }
}
```

`ProductSellingStatus`:

```java
public enum ProductSellingStatus {
    HOLD("판매 보류"), SELLING("판매 중"), STOP_SELLING("판매 중지");
    private final String description;
    private static final List<ProductSellingStatus> CANNOT_SELLING_STATUSES = List.of(HOLD, STOP_SELLING);
    public boolean cannotSelling() { return CANNOT_SELLING_STATUSES.contains(this); }
    public static List<ProductSellingStatus> forSelling() { return List.of(SELLING); }
}
```

`Stock` (STEP03 = 조회 전용, deduct/restore 제외):

```java
@Getter @Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Stock {
    @Id @Column(name = "stock_id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long productId;
    private int quantity;
    // create(productId, quantity) + quantity < 0 검증
    // ⚠️ deduct()/restore()는 STEP04~05에서 추가 (비관적 락과 함께)
    // ⚠️ 레퍼런스 Stock엔 @Table(indexes=@Index(product_id))가 있으나 인덱스는 STEP04
}
```

### 정석 흐름 체크리스트

- [ ] Product 도메인 정식 구현 (8 Step)
- [ ] `ProductControllerTest`의 `@Disabled` 제거 → 정식 테스트로. ranks 테스트는 삭제.
- [ ] `ControllerTestSupport`에 `ProductController` + (Facade 없으니) `ProductService` `@MockitoBean` 추가
- [ ] `ProductControllerDocsTest` 정식 구현 (현재 `new ProductController()` → `new ProductController(productService)`로). ranks docs 삭제.
- [ ] 모든 테스트 통과 (`./gradlew test --rerun-tasks`)
- [ ] PR 머지

### 본인 현재 Product 상태 (✅ 확인함)

- `product/controller/ProductController.java`: mock. `List.of(ProductResponse.of(1L,"상품명",30000L,100L))` 하드코딩 + `/ranks`도 하드코딩.
- `product/dto/response/ProductResponse.java`: `id/name/price/stock` (위치만 interfaces/dto로 이동, 내용은 거의 유지)
- `product/dto/response/ProductRankResponse.java`: `id/name/price/saleCount` → 삭제
- `ProductControllerTest`: `@Disabled("상품 도메인 정식 구현 시 재작성 예정")`, getProducts + getRanks 2개
- `ProductControllerDocsTest`: `initController()`가 `new ProductController()` (인자 없음) → 본체 바뀌면 깨짐
- order/product mock은 쿠폰 할인/유저 nickname 참조 안 함 (ERD 정렬 때 확인)

---

## 7. STEP03 신호등 (절대 하지 말 것)

- `@Version` 낙관적 락 → STEP05
- `@Lock(PESSIMISTIC_WRITE)` → STEP05 (재고 차감용)
- `@Transactional` on Service → STEP05 (⚠️ 단, 레퍼런스 ProductService엔 `@Transactional(readOnly=true)`가 있음. 본인 신호등은 STEP05부터라 STEP03에선 생략)
- `@Cacheable` → STEP06
- `Redisson` → STEP06
- `ApplicationEventPublisher` / `@TransactionalEventListener` → STEP07
- `KafkaTemplate` → STEP08
- `@Index` / 커서 페이징 → STEP04
- `Rank` 도메인 / `/ranks` → STEP04~
- Stock `deduct()`/`restore()` → STEP04~05
- `BaseEntity` / `createdAt` 필드 → 레퍼런스에 없음

---

## 8. 전체 남은 작업 목록 (GitHub 이슈)

|STEP|이슈#|제목|
|---|---|---|
|STEP03|22|상품 비즈니스 로직 구현 및 단위 테스트 (다음 작업)|
|STEP04|23|주문/결제 비즈니스 로직 구현 및 단위 테스트|
|STEP04|24|인프라 레이어 구현체 작성|
|STEP04|25|기능별 통합 테스트 작성|
|STEP04|26|주요 기능별 동시성 실패 테스트 작성|
|STEP04|27|병목 예상 쿼리 분석 및 최적화 보고서 작성|
|STEP05|28|주요 기능별 동시성 테스트 작성|
|STEP05|29|주요 기능 동시성 이슈 식별 및 해결|
|STEP05|30|동시성 이슈 분석 및 해결 보고서 작성|
|STEP05|31|Filter/Interceptor/Scheduler 부가 로직 구현|
|STEP05|32|모든 API 정상 작동 및 가용성 확보|
|STEP06|33|Redis 기반 분산락 구현 및 적용|
|STEP06|34|Redis 분산락 동시성 보고서 추가|
|STEP06|35|Redis 기반 캐싱 전략 설정 및 적용|
|STEP06|36|캐싱 전략 및 성능 개선 보고서 작성|
|STEP06|37|인기상품 Redis 기반 설계 및 구현|
|STEP06|38|선착순 쿠폰발급 Redis 기반 설계 및 구현|
|STEP06|39|Redis 디자인 설계 보고서 작성|
|STEP07|43|주문/결제 완료 시 이벤트 기반 외부 데이터 플랫폼 전송|
|STEP07|44|파사드 제거 및 이벤트 기반 도메인 서비스 구현|
|STEP07|45|MSA 기반 이벤트 아키텍처 설계 문서 작성|
|STEP08|46|카프카 기초 및 핵심 개념 문서 작성|
|STEP08|47|주문 완료 시 데이터 플랫폼으로 카프카 메시지 발행|
|STEP08|48|대용량 트래픽 프로세스 카프카 활용 구현|
|STEP08|49|Outbox 패턴 적용|
|STEP08|50|카프카 기반 설계 문서 작성|
|STEP09|51|부하테스트 대상 선정 및 시나리오 계획 문서 작성|
|STEP09|52|부하테스트 스크립트 작성|
|STEP09|53|부하테스트 결과 기반 병목 탐색 및 개선|
|STEP09|54|성능 테스트 및 장애대응 보고서 작성|

### 이슈 제목에 묻혀있는 큰 작업들 (과소평가 주의)

|이슈|묻혀있는 실제 작업|
|---|---|
|#29 동시성 해결|**Balance 큰 리팩토링** — `@OneToMany` 제거, `balanceId Long` 참조, `Balance.create(userId)` 시그니처 변경, `saveTransaction` 명시 호출 (13번 추적표)|
|#31 Filter/Interceptor|**Rank 도메인 분리**, 배치 스케줄러 도입|
|#44 Facade 제거|**3가지**: Facade 제거 / `BalanceClient` 인터페이스 / `refund` + `REFUND` enum + Saga 보상|
|#48 카프카 트래픽|**인기상품 실시간 이벤트** (5번째 진화, 배치 제거)|

### 이슈로 안 잡힌 항목 (놓치기 쉬움)

|항목|시점|처리|
|---|---|---|
|OrderControllerTest @Disabled 제거|STEP04 #23|정식 테스트로 갱신|
|ProductControllerTest @Disabled 제거|STEP03 #22|정식 테스트로 갱신 (지금)|
|ProductControllerDocsTest 정식 작성|STEP03 #22|정석 흐름으로 함께 (지금)|
|OrderControllerDocsTest 재작성|STEP04 #23|정석 흐름으로 함께|
|WIL 회고|STEP05 끝 / STEP09 후|핵심 메모는 그 시점, 정식은 몰아서|

---

## 9. STEP04~09 미리보기 (✅ report 7종 정독 기반)

> 각 STEP 진입 시 해당 report/WIL을 그 STEP에서 다시 정독 후 코드. 아래는 검증된 핵심 요약.

### STEP04 — 주문/결제 + DB 최적화 (report 01)

1. Testcontainers 통합 테스트 (`@SpringBootTest` + MySQL 컨테이너)
2. JPA Repository 채우기 + Repository 인터페이스 `@Repository`/`@Component`
3. **OrderFacade에 `@Transactional`** (Service 아님 — STEP04 신호등). Service @Transactional은 STEP05.
4. `@Enumerated(EnumType.STRING)` 모든 enum
5. 인덱스 적용 — **보고서 먼저 → 코드** 순서. 10만 건 + `EXPLAIN ANALYZE`
6. 동시성 실패 테스트 미리 작성 (STEP05 준비)
7. Stock에 `deduct()`/`restore()` 추가 (주문이 재고 차감하므로)

✅ **report 01 결정적 교훈** (그대로 베끼지 말 것):

- 잔액/재고/쿠폰: 단일·복합 인덱스로 99%+ 개선 (`user_id`, `product_id`, `(user_id,used_status)`, `(user_id,coupon_id)`)
- **상품 조회는 인덱스 역효과** — `sell_status='SELLING'`이 전체의 90% → 카디널리티 낮아 풀스캔이 더 빠름. **커서 페이징**(`product_id > cursor`)으로 해결 (0.14ms)
- 인기상품: `(payment_status, paid_at)` 복합 + `order_id` 단일
- 측정값/스크린샷은 그 STEP 끝나자마자 보고서로 (재현 번거로움)

### STEP05 — 동시성 (report 02)

✅ **자원별 일관된 락 전략** (핵심: 같은 자원엔 같은 전략):

|자원|전략|이유|
|---|---|---|
|잔액|낙관적 락 (`@Version`)|중복 충전/차감은 의도 아님 → 하나만 성공|
|쿠폰|비관적 락 (+STEP06 분산락)|선착순 모두 성공해야|
|재고|비관적 락|동시 차감 음수 방지|

- 락 결정 기준: **반드시 성공해야 하는 요청 → 비관적**, 아니면 낙관적. 락 범위는 최소(사용자/상품 단위).
- **Balance 큰 리팩토링** (#29에 묻힘): `@OneToMany` cascade 제거 → `balanceId Long` 참조, `Balance.create(userId)` (amount 제거), `BalanceService`에 `@Transactional`, `saveTransaction` 명시 호출, `@Version` 추가
- 한계: 비관적 락도 **공정성(순서) 보장 안 됨** → 선착순은 Kafka 직렬처리로 (STEP08)
- 기타: Rank 도메인 분리, Filter/Interceptor 도입
- 테스트: `ExecutorService` + `CompletableFuture`, 스레드 2개, `AtomicInteger`로 성공/실패 집계 (AS-IS 락없음 → TO-BE 락적용)

### STEP06 — 분산락 + 캐시 + Redis (report 02 분산락 / 03 캐시 / 04 Redis)

**Phase 1 — Redisson 분산락 (report 02)**:

- `@DistributedLock` 어노테이션 + `DistributedLockAspect` (AOP)
- `@Order(Ordered.HIGHEST_PRECEDENCE)` — **락 AOP가 트랜잭션보다 먼저**
- 전략 패턴: `LockTemplate` 인터페이스 + SpinLock / PubSubLock(Redisson) 구현
- ✅ **분산락 + 트랜잭션 순서** (매우 중요):
    
    ```
    1. 분산 락 획득 (트랜잭션 밖)2. 트랜잭션 시작3. 비즈니스 로직4. 트랜잭션 커밋5. 분산 락 해제
    ```
    
- 쿠폰: 비관적 락 + 분산락 병행 (Redis 선점 제어 + DB 최종 정합성)

**Phase 2 — 캐시 (report 03)**:

- 인기상품 조회에 `@Cacheable` (Read-Through). 캐시 미스 시 DB 조회 후 자동 저장
- 배치 갱신은 `@CachePut` (Write Through), 매일 00:05 스케줄러
- ✅ **TTL 49시간** 이유: 24h면 매일 00:05 배치와 만료가 겹쳐 미스 리스크 + 배치 실패 시 hotfix 여유. eviction 정책 없음(일 단위 갱신)
- 캐시 스탬피드 주의 (배치 2회 연속 실패/TTL 만료 시 대량 DB 유입)
- K6 측정: 평균 54ms → 2.1ms (약 25배), p95 31배 개선

**Phase 3 — Redis 자료구조 (report 04)**:

- 인기상품: **Sorted Set(ZSET)**. value=상품ID, score=판매량. `ZINCRBY rank:sell:{yyyyMMdd}` 주문완료 시 집계, `ZUNIONSTORE`로 최근 3일 합산, 5분 스케줄러로 TOP5 갱신
- 선착순쿠폰: **ZSET**. value=사용자ID, score=발급요청시각(Epoch). `addIfAbsent`로 중복방지+순서보장. 1분 배치로 DB BatchInsert, 5분 배치로 `FINISHED` 처리
- → CouponStatus에 `FINISHED`(발급종료) 추가되는 시점

### STEP07 — Facade 제거 + EDA (report 05)

✅ 이슈 #44 안에 3가지 큰 작업:

1. **Facade 제거** — `BalanceFacade`, `BalanceCriteria`, `BalanceResult` 삭제
2. **`BalanceClient` 인터페이스** 신규 (도메인 격리. BalanceService가 balanceClient.getUser() 호출)
3. **`refund` 메서드** — `Balance.refund(amount)`, `BalanceCommand.Refund`, `BalanceTransaction.ofRefund()`, `BalanceTransactionType.REFUND`

EDA 구조:

- 주문 생성 후 트랜잭션 커밋 → `OrderEvent.Created` 발행
- 잔액/쿠폰/재고 리스너가 `@Async @TransactionalEventListener(AFTER_COMMIT)`로 **병렬** 처리, 각자 결과 이벤트 발행
- 모든 프로세스 수신 후 결제 대기 이벤트 → 결제 서비스 → 성공/실패 이벤트
- **Saga 보상 트랜잭션**: 하나라도 실패 → 주문 실패 이벤트 → 각 서비스 보상(잔액 환불/쿠폰 취소/재고 복구) + 주문 취소
- Redis Hash로 주문 프로세스 상태(PENDING/SUCCESS/FAILED) 관리
- `orderId` 분산락으로 이벤트 중복 발행 방지
- ⚠️ Spring ApplicationEvent라 단일 인스턴스 한계 → STEP08 Kafka

### STEP08 — Kafka + Outbox (report 06)

- 선착순 쿠폰을 **즉시 발급**으로 개선 (배치 지연 제거 — UX)
- ✅ **파티션키 = 쿠폰ID** 채택 (토픽 분리 방식 대비 단순. 동일 쿠폰 순차처리 보장. 단 병렬 한계)
- 토픽 버저닝 (`...v1` → `...v2`) — 파티션 수 변경 불가 대응
- **Outbox 패턴**:
    
    ```
    Auto 이벤트(트랜잭션 있음): BEFORE_COMMIT에 Outbox 저장 → AFTER_COMMIT(@Async)에 Kafka 발행Manual 이벤트(트랜잭션 없는 로직): 즉시 Outbox 저장 후 Kafka 발행 (@Async @EventListener)Consumer 처리 완료: eventId로 Outbox 삭제 / ack 수동커밋
    ```
    
- 멱등성: 수신 측에서 중복여부/발급개수/만료일 재검증
- Redis는 "발급 가능 여부 조회"로만 축소 (`coupon:available:{couponId}`)
- ✅ **인기상품 5번 진화**: ①DB집계(STEP03~04) → ②배치(STEP05) → ③Redis캐시(STEP06 P2) → ④Redis ZSET(STEP06 P3) → ⑤실시간 이벤트(STEP08)

### STEP09 — 부하테스트 (report 07)

- Spring Actuator + Prometheus + Grafana + K6(+InfluxDB)
- ✅ 시나리오: 주문/결제 Load Test 최대 300VU (충전 20%→주문 10% 확률), 선착순 쿠폰 Peak Test 최대 1000VU
- ✅ SLA: **p99 < 1s**, 실패율 < 1%(주문) / < 5%(쿠폰)
- 개선: 인기상품 Redis 캐싱 + Kafka 파티션/컨슈머 추가
- ✅ **Lag 방지**: `@KafkaListener`에서 `CoreException`(재시도 무의미한 의도된 에러: 중복발급/재고부족) 시 `ack.acknowledge()` 호출해 재처리 방지
- 결과: 주문 p99 2.19s→405ms, 쿠폰 p99 6.92s→531ms

---

## 10. 문서/산출물 작성 타이밍

> 원리: **기능 구현 → 그 기능 보고서 즉시 작성**. 측정값·스크린샷 휘발 전에. 보고서 공통 구조: **배경 → 대상 선정 → 문제분석(AS-IS) → 해결방안 → 측정/테스트(TO-BE) → 한계 → 결론**.

### 설계 문서 (docs/architecture) — STEP02 완료

01.Requirements / 02.Milestones / 03-1.Sequence / 03-2.State / 04.ERD / 05.ApiDocument (06.SpringRestDocs 선택, 본인 미작성)

### 기술 보고서 (docs/report) — 해당 STEP "끝낸 직후"

|문서|트리거|STEP/이슈|
|---|---|---|
|01.DBPerformanceOptimizationReport|인덱스 + EXPLAIN ANALYZE 측정 후|STEP04 #27|
|02.ConcurrencyReport|락으로 동시성 해결 + 테스트 통과 후|STEP05 #30|
|(분산락 보고서)|Redisson 분산락 + 테스트 후|STEP06 P1 #34|
|03.CacheStrategyArchitectureReport|@Cacheable + K6 전후 측정 후|STEP06 P2 #36|
|04.RedisDesignArchitectureReport|ZSET 인기상품/선착순쿠폰 재설계 후|STEP06 P3 #39|
|05.MsaEventDrivenArchitectureReport|Facade 제거 + EDA + Saga 후|STEP07 #45|
|06.KafkaDesignArchitectureReport|Kafka 발행/소비 + Outbox 후|STEP08 #50|
|07.LoadTestReport|K6 부하테스트 + 병목 개선 후|STEP09 #51·#54|

### 스터디 (docs/study)

- 01.Kafka.md: Kafka 도입 시작 시 동시 정리 (STEP08 #46)
- 02.Cache.md: 캐시 여러 STEP 겪은 후 종합 (선택)

### WIL — 핵심 메모는 그 시점, 정식은 STEP09 후 몰아서

week2(STEP02 설계) / week3(STEP03 아키텍처) / week4(STEP04 트랜잭션·인덱스) / week5(STEP05~06 동시성·분산락)

### 작성 순서 한눈에

```
STEP02(완료): 01~05 설계문서 → REST Docs
STEP03(지금): 문서 산출물 없음. 도메인 구현 + Docs 테스트만. WIL week3 핵심 메모.
STEP04 끝:   01.DBPerformanceReport (#27) + WIL week4 메모
STEP05 끝:   02.ConcurrencyReport (#30) + WIL week3 정식/week5 메모
STEP06 P1:   분산락 보고서 (#34)
STEP06 P2:   03.CacheStrategyReport (#36)
STEP06 P3:   04.RedisDesignReport (#39)
STEP07 끝:   05.MsaEventDrivenReport (#45)
STEP08 중/끝: 01.Kafka.md (#46) + 06.KafkaDesignReport (#50)
STEP09 끝:   07.LoadTestReport (#51·#54) + WIL 전체 정식 회고
```

---

## 11. STEP별 신호등 (전체)

|작업|S03|S04|S05|S06|S07|S08|S09|
|---|---|---|---|---|---|---|---|
|도메인 엔티티 비즈니스 메서드|O|O|O|O|O|O|O|
|4-Layer 클린 아키텍처|O|O|O|O|O|O|O|
|Repository I/F + 구현체 분리|O|O|O|O|O|O|O|
|단위 테스트 / Docs 테스트|O|O|O|O|O|O|O|
|파사드 클래스|O|O|O|O|X(제거)|X|X|
|`@OneToMany` cascade|O|O|X(제거)|X|X|X|X|
|`@Transactional` on Facade|X|O|O|O|X(제거)|X|X|
|`@Transactional` on Service|X|X|O|O|O|O|O|
|통합 테스트 (Testcontainers)|X|O|O|O|O|O|O|
|인덱스 / 커서 페이징|X|O|O|O|O|O|O|
|`@Enumerated(EnumType.STRING)`|O|O|O|O|O|O|O|
|`@Version` 낙관적 락|X|X|O|O|O|O|O|
|`@Lock` 비관적 락|X|X|O|O|O|O|O|
|Stock deduct/restore|X|O|O|O|O|O|O|
|Rank 도메인 분리 / `/ranks`|X|O|O|O|O|O|O|
|배치 스케줄러|X|X(집계)|O|O|O|X(제거)|X|
|Filter/Interceptor|X|X|O|O|O|O|O|
|Redisson 분산 락|X|X|X|O|O|O|O|
|`@Cacheable` 캐시|X|X|X|O|O|O|O|
|Redis Sorted Set|X|X|X|O|O|O|O|
|`refund` / `BalanceClient` / `REFUND`|X|X|X|X|O|O|O|
|`ApplicationEventPublisher`|X|X|X|X|O|O|O|
|`@TransactionalEventListener` / `@EnableAsync`|X|X|X|X|O|O|O|
|Saga (보상 트랜잭션)|X|X|X|X|O|O|O|
|Kafka Producer/Consumer / Outbox|X|X|X|X|X|O|O|
|K6 / Prometheus + Grafana|X|X|X|X|X|X|O|

> ⚠️ 레퍼런스 ProductService엔 `@Transactional(readOnly=true)`가 STEP03 시점부터 있으나, 본인 신호등은 Service `@Transactional`을 STEP05로 잡음. STEP03 Product에선 생략.

---

## 12. Balance 도메인 진화 추적표 (✅ report 02/05 검증)

|항목|S03|S04|S05|S06|S07|S08|
|---|---|---|---|---|---|---|
|`Balance.create`|`(userId, amount)`|동일|`(userId)`|동일|동일|동일|
|`@OneToMany`|cascade|있음|**제거**|없음|없음|없음|
|`BalanceTransaction` FK|`Balance` 참조|참조|**`balanceId` Long**|Long|Long|Long|
|`@Version`|없음|없음|**추가**|있음|있음|있음|
|`@Index(user_id)`|없음|**추가**|있음|있음|있음|있음|
|`@Transactional`(Service)|없음|없음|**추가**|있음|있음|있음|
|`BalanceFacade`|있음|있음|있음|있음|**삭제**|없음|
|`BalanceClient`|없음|없음|없음|없음|**추가**|있음|
|`refund` / `REFUND` enum|없음|없음|없음|없음|**추가**|있음|
|`saveTransaction`|없음|없음|**추가**|있음|있음|있음|

> ✅ 레퍼런스 최종 Balance: `MAX_BALANCE_AMOUNT=10_000_000`, `INIT=0`, charge/use/refund 각각 검증, `@Version`, `@Table(indexes=@Index(idx_user_id, user_id))`. BalanceService는 balanceClient.getUser() 호출 + 모든 메서드 @Transactional + saveTransaction.

---

## 13. Coupon 도메인 진화 추적표 (✅ ERD 정렬로 STEP03 확정)

|항목|S03(정렬 후)|S05|S06|S08|
|---|---|---|---|---|
|Coupon 상태값|`REGISTERED/PUBLISHABLE/CANCELED`|동일|+`FINISHED`(발급종료)|동일|
|`discountRate`|double (0~1)|동일|동일|동일|
|`quantity`|int 하나 (`quantity--`)|동일|동일|동일|
|`expiredAt`|있음|있음|있음|있음|
|발급 검증|상태+만료+수량|**+중복발급**(UK)|동일|동일|
|발급 동시성|없음|**비관적 락**|**+분산 락**|**Kafka 직렬**(파티션키=쿠폰ID)|
|UserCoupon 상태|`UNUSED/USED`|동일|동일|동일|
|UserCoupon `EXPIRED/CANCELED`|없음|없음|없음|(STEP07 환불 영역)|
|중복발급 방지|없음|UK(user_id,coupon_id)|Redis addIfAbsent|Kafka 멱등성|

> ⚠️ State Diagram엔 쿠폰 EXPIRED가 별도 상태로 그려져 있으나, 본인 구현은 expiredAt 필드로 만료 판정(별도 상태 안 둠). 레퍼런스 코드도 동일. ✅ 레퍼런스 최종 Coupon: `publish()` 메서드명 + `CoreException`. 본인은 `issue()` + `IllegalArgumentException` (의도된 차이).

---

## 14. 트러블슈팅 — 이미 겪은 것 (재발 방지)

1. **`contextLoads` 실패** — `NoSuchBeanDefinitionException`(DataSource). 원인: main yaml DataSourceAutoConfiguration exclude + JpaRepository 충돌. 해결: test 전용 yaml + H2 의존성 (적용됨)
2. **Service 테스트 메서드 잘못 호출** — 단일 통과/전체 실패. "테스트 대상 메서드를 정확히 호출하는가" 확인 필수
3. **ControllerDocsTest 컴파일 실패** — 본질은 미동기화. 본체에 `new XxxController(facade)` 의존성 추가됐는데 Docs 테스트는 옛 `new XxxController()` 그대로. 해결: 본체 바꿀 때 테스트 같이 갱신 (#22부터 적용). 레퍼런스는 Docs 테스트 끝까지 살림
4. **커밋 단위 압축** — `git add .` 두 번으로 8단계가 2커밋. 해결: `git reset --soft` → 단위별 재커밋 → `git push --force-with-lease`. 예방: 작업 시작부터 커밋 단위 의식
5. **`git restore` 실수** — 주석 풀린 상태로 복원됨. 교훈: restore는 마지막 커밋 상태로 되돌림. 의도 안 한 파일이면 그냥 두기
6. **Order/Product mock 테스트 깨짐** — `@WebMvcTest(controllers={...})`에서 빼면 상속 테스트 깨짐. 해결: `@Disabled` + 사유 명시
7. **도메인 검증 `<` vs `<=`** — "양수" 메시지인데 0 통과. `if(amount < 0)` → `<= 0`. "양수"는 0 초과
8. **예외 타입 불일치** — `IllegalStateException` 던지니 500. ApiControllerAdvice는 `IllegalArgumentException`만 잡음. 도메인 검증 실패는 `IllegalArgumentException`로 통일
9. **Rebase로 커밋 메시지 정정** — `git rebase -i {기준}` → `reword`. 합치기는 `squash`. 안전판 `--abort`. 강제푸시 `--force-with-lease`
10. **Docs 테스트 Mock 주입 차이** — `@MockitoBean`이 Docs Test(standalone)에서 안 통함. 자식이 `Mockito.mock()`으로 생성 → 생성자 주입
11. **빈 파일 머지 사고** — Docs 테스트가 0줄로 커밋·머지(본문은 working tree에만). 해결: `git stash`→main 최신화→새 브랜치→`stash pop`→커밋. 예방: 커밋 전 `git diff --cached` 확인
12. **테스트가 못 잡은 버그(쿠폰)** — ①`findCouponById(userId)`(couponId여야)인데 테스트가 userId==couponId(둘 다 1L)라 못 잡음 ②`UserCoupon.of()`에 usedStatus 매핑 누락인데 assert 없어 못 잡음. 해결: 테스트 데이터 서로 다른 값(userId=1,couponId=5), 모든 필드 assert. 교훈: 같은 값 쓰면 ID 바꿔치기 버그 못 잡음

### 13. (이번 세션) 핸드오프 전제와 실제 코드 불일치

- 증상: 핸드오프 시작 문구가 "엔티티 반영 후 compileJava 단계"라 했으나, 실제론 엔티티 미반영 상태였음
- 교훈: 세션 시작 시 **문서 말고 실제 코드를 먼저 확인**. 이 가이드도 "검증 표기(✅/⚠️)"로 추측과 사실을 구분.

---

## 15. 막힐 때 활용할 명령

레퍼런스 동일 시점 확인:

```
git --no-pager show <hash>:<path>                          # 특정 커밋 시점 파일
git --no-pager log --all --oneline --reverse -S "<keyword>" -- "*.java"   # 키워드로 커밋 검색
git diff --no-index <본인파일> <레퍼런스파일>               # 비교
```

레퍼런스 폴더 둘러보기:

```
Get-ChildItem -Recurse "C:\Users\eborder\sungmin\git\e-commerce-reference\service\{도메인}\src\main\java\kr\hhplus\be\ecommerce\{도메인}" | Select-Object FullName
```

본인 현재 상태:

```
Get-ChildItem -Recurse "C:\Users\eborder\sungmin\git\e-commerce\src\main\java\com\github\gokid96\e_commerce\{도메인}" | Select-Object FullName
```

> 레퍼런스는 멀티모듈 최종본이라 STEP 시점과 코드가 다름. 문서=방향, 코드=해당 STEP 커밋 시점.

---

## 16. 면접/이력서 어필 자산

```
- 라이트 DDD (Persistence-aware Domain Model) 적용
- 4-Layer 클린 아키텍처 + DIP
- Facade Pattern → EDA 전환 경험 (STEP03 → STEP07)
- Repository Pattern으로 영속성 격리
- Static Factory Method + Builder Pattern으로 도메인 캡슐화
- DTO 변환 흐름으로 레이어 간 의존 분리
- Test-After Development + Always Green
- Given-When-Then + Living Documentation (Spring REST Docs)
- 낙관/비관/분산락, Redis 캐시(TTL 49h), ZSET, Kafka, Saga, Outbox
- 인기상품 5번 진화 (DB→배치→캐시→SortedSet→실시간 이벤트)
- 부하테스트로 병목 식별·개선 (주문 p99 2.19s→405ms, 쿠폰 6.92s→531ms)
```

강한 면접 무기: ①라이트 DDD ②Facade→EDA 전환 ③분산락+트랜잭션 순서 ④인기상품 5번 진화 ⑤Outbox+Kafka 멱등성

### TDD vs 본인 방식 (Test-After + Always Green)

- TDD: 빨강(실패 테스트) → 초록(본체) → 리팩토링. 테스트가 본체 설계를 이끔.
- 본인: 본체 먼저 → 테스트 나중 → 본체 변경 시 테스트 같이 갱신(Always Green). **TDD 아님, 현업 다수 방식.**
- 판별: ①첫 테스트 작성 시 컴파일 안 됐나(TDD:네/본인:아니오) ②테스트 통과시키려 본체 작성(TDD:네/본인:아니오) ③테스트가 본체 설계 이끔(TDD:네/본인:아니오)

---

## 17. 다음 세션 시작 시 안내 문구

```
mini-commerce 프로젝트 진행할거야.

쿠폰/유저쿠폰 ERD 정렬 리팩토링은 끝나서 main에 머지됐고,
지금은 STEP03 이슈 #22 (Product 도메인) 차례야.
브랜치 feat/step03-product-domain 에서 시작 (없으면 main pull 후 생성).

이 가이드 6번 섹션(이슈 #22 상세)에 파일 트리/엔티티 형태/체크리스트 다 있어.
정석 흐름(Docs 테스트 함께, Always Green, 커밋 쪼개기)으로,
경로+코드+설명 방식으로 진행. (Claude는 코드 직접 수정 X, 타이핑은 내가)

먼저 본인 현재 Product mock 상태 다시 확인하고 8 Step 첫 커밋부터 가자.
```

### ⚠️ 다음 세션 우선 보강 (이번에 못 읽은 것)

가이드 정확도를 더 높이려면 다음을 정독해 이 문서를 보강:

- WIL week5 (동시성/분산락 개념 — synchronized/ReentrantLock/Atomic, Spin/PubSub Lock 상세)
- study 01.Kafka.md (Broker/Partition/ConsumerGroup, 파티션-컨슈머 비율, Rebalancing, DLQ)
- study 02.Cache.md
- WIL week2 (설계 회고)

> 단, week5↔report02, Kafka↔report06, Cache↔report03 은 내용이 상당히 겹쳐 이미 핵심은 반영됨.

---

> 이 문서는 살아있는 문서. STEP 진행하며 발견하는 것 추가/갱신. 검증 표기(✅/⚠️/📌)를 유지해 추측과 사실을 구분할 것.