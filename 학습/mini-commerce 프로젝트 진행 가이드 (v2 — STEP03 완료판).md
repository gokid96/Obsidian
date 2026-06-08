# mini-commerce 진행 가이드 (v3)

> 학습용 이커머스 프로젝트. 본인이 직접 타이핑하며 배우는 게 목적. Claude는 **코드를 대신 수정하지 않고**, "① 경로 ② 코드 ③ 왜"로 제시 → 타이핑·git·gradlew는 본인이 실행 후 결과 공유.

---

## 0. 프로젝트 기본 정보

|항목|값|
|---|---|
|본인 레포|https://github.com/gokid96/e-commerce|
|로컬 경로|`C:\Users\eborder\sungmin\git\e-commerce`|
|레퍼런스|https://github.com/discphy/e-commerce (STEP09까지 완료된 멀티모듈 최종본)|
|레퍼런스 로컬|`C:\Users\eborder\sungmin\git\e-commerce-reference`|
|패키지 루트|`com.github.gokid96.e_commerce`|
|스택|Spring Boot 4.0.5, Java 21, H2(test), JPA/Hibernate|

### Boot 4 주의 (Boot 3과 다른 점)

- ObjectMapper: `tools.jackson.databind.ObjectMapper` (com.fasterxml 아님)
- WebMvcTest: `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`
- 목 빈: `@MockitoBean` (`@MockBean` 아님)

---

## 1. 작업 방식 (협업 규칙)

1. **Claude는 직접 코드 수정 X.** 경로 + 코드 + 이유를 제시하고, 타이핑은 본인이.
2. **git / gradlew는 본인이 실행**하고 결과(콘솔 출력)를 공유. Claude가 그걸 보고 다음 안내.
3. **커밋 전 항상 `git status`로 staged 확인** (트러블 #16 — staged/unstaged 섞임 방지).
4. Claude는 세션 시작 시 **실제 코드를 먼저 읽고** 검증 (기억/추측으로 안내 금지 — 트러블 #13).
5. 각 코드 작성 후 **`./gradlew compileJava`**, 테스트 작성 후 **`./gradlew test`**로 그린 확인.

---

## 2. 아키텍처

### 2-1. 패키지 구조: Feature-by-package + 라이트 DDD

최상위가 **도메인**(`balance/`, `order/`, `payment/`, `coupon/`, `product/`), 그 안에 레이어:

```
order/
├── application/      ← Facade (여러 도메인 조율)
├── domain/           ← 엔티티, DTO(Command/Info), Repository(인터페이스), Service, enum
├── infrastructure/   ← Repository 구현체(CoreRepository) + jpa/(JpaRepository)
└── interfaces/       ← Controller, Request, Response
```

> ⚠️ 레퍼런스는 **layer-by-package**(반대 구조: `domain/order`, `service/order` …). import 경로 번역 필요.

### 2-2. supporting 도메인은 일부 레이어만

- `payment/` — Controller/Facade 없음. `domain/` + `infrastructure/`만 (결제는 OrderFacade가 조율).
- `product/domain/stock`, `product/domain/rank` — product 하위 별도 폴더.

### 2-3. 애그리거트 단위로 묶기

- 같은 생명주기 = 같은 폴더: `order/`(Order+OrderProduct), `coupon/`(Coupon+UserCoupon)
- 느슨한 연결(`xxxId: Long` 참조) = 별도 폴더/도메인: `payment/`(orderId 참조), `stock`(productId 참조)

---

## 3. 코드 컨벤션 (balance/coupon/product 실측으로 확정)

### 3-1. 엔티티

```java
@Getter
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {

    @Id
    @Column(name = "order_id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long userId;          // FK는 plain Long (@Column 없음)

    @Enumerated(EnumType.STRING)  // enum은 항상 STRING
    private OrderStatus orderStatus;

    @Builder
    private Order(...) { ... }            // private 빌더 생성자

    public static Order create(...) {     // 정적 팩토리
        // 검증 → builder().build()
    }

    public void paid() { ... }            // 상태 변경은 의도 있는 메서드
}
```

- `@Table` **안 씀** (네이밍 전략이 `OrderProduct`→`order_product` 자동 변환). id에만 `@Column(name="xxx_id")`.
- **`@Table` 예외 = `Order`만** → `@Table(name = "orders")` (`order`가 SQL 예약어).
- import는 개별 import (와일드카드 `*` 금지).
- price/amount는 primitive `long`.

### 3-2. enum (설명 + 헬퍼)

```java
@Getter
@RequiredArgsConstructor
public enum PaymentStatus {
    READY("결제 준비"),
    COMPLETED("결제 완료"),
    FAILED("결제 실패"),
    CANCELED("결제 취소");

    private final String description;

    private static final List<PaymentStatus> CANNOT_PAYABLE_STATUSES =
            List.of(COMPLETED, FAILED, CANCELED);

    public boolean cannotPayable() {
        return CANNOT_PAYABLE_STATUSES.contains(this);
    }
}
```

- `상수("설명")` + `description` 필드.
- IN-list 헬퍼(`cannotXxx`)는 **양방향 테스트** (true/false 둘 다 — 트러블 #14).

### 3-3. DTO (Command / Info / Criteria / Result)

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)   // outer
public class OrderCommand {

    @Getter
    public static class Create {                      // nested: @Getter + final
        private final Long userId;
        ...
        @Builder
        private Create(...) { ... }
        public static Create of(...) { return Create.builder()...build(); }
    }
}
```

- **Command/Criteria**: `@Builder` + `of()`.
- **Info/Result**: `of()`가 **엔티티를 받음**. 중첩 클래스명이 엔티티명과 충돌하면 **엔티티를 풀패키지명**으로 표기.
    
    ```java
    public static Order of(com.github.gokid96.e_commerce.order.domain.Order order) {    return Order.builder().orderId(order.getId())...build();}
    ```
    
- 단일 필드(리스트 래퍼 등)는 `@Builder` 대신 `new` + `of()`로 충분.

### 3-4. Repository 3단 구조

```
domain/OrderRepository                 ← 인터페이스 (@Repository), 메서드 선언만
   ↑ implements
infrastructure/OrderCoreRepository     ← @Component, 도메인 인터페이스 구현 + Jpa에 위임
   │  (생성자 주입은 OrderJpaRepository!)   ⚠️ 도메인 인터페이스 주입 금지(자기참조 순환)
   ↓ 위임
infrastructure/jpa/OrderJpaRepository  ← extends JpaRepository<Order, Long>
```

```java
@Component
@RequiredArgsConstructor
public class OrderCoreRepository implements OrderRepository {
    private final OrderJpaRepository orderJpaRepository;   // ← Jpa 주입
    public Order save(Order o) { return orderJpaRepository.save(o); }
    public Optional<Order> findById(Long id) { return orderJpaRepository.findById(id); }
}
```

- 조회는 `Optional` 반환 → Service에서 `orElseThrow`.
- 레퍼런스는 RepositoryImpl 1개(2단). 본인은 3단으로 분리.

### 3-5. Service

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;

    public void paidOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("주문이 존재하지 않습니다."));
        order.paid();
    }
}
```

- `@Transactional` **없음** (Service 레벨 트랜잭션은 STEP05). 단, **Facade의 `@Transactional`은 #25에서 바로** 붙임 (트랜잭션 경계 = Facade).
- 조회 `Optional` + `orElseThrow`.
- **예외는 전부 `IllegalArgumentException`으로 통일** (트러블 #8 — 레퍼런스는 IllegalState 섞음).
- DTO→엔티티 변환은 Service의 일 (도메인이 Command를 모르게).

### 3-6. 변환 흐름

```
Request → Criteria.toCommand() → Command → [도메인] → Info.of(entity) → Result.of(info) → Response
```

### 3-7. 테스트

- **도메인 단위 테스트**: plain 클래스(어노테이션 없음) + `@DisplayName`(한글) + given/when/then + AssertJ.
- **Service 테스트**: `@ExtendWith(MockitoExtension.class)` + `@Mock`/`@InjectMocks` + `given().willReturn()` + `verify(repo, times/never)`.
- **`@ParameterizedTest` 안 씀** — plain `@Test` 한 메서드에서 여러 상태를 직접 assert (ProductTest의 `cannotSelling_true` 방식).
- 계산 검증 시 상품 수량·가격을 **서로 다른 값**으로 (합산 오류 잡기 — 트러블 #12).

---

## 4. 도메인 설계 메모 (STEP04 확정)

### Order / OrderProduct

- `Order.totalPrice = 합계 − 할인` (net, 최종결제액). `discountPrice` = 할인액. (레퍼런스 테스트가 net 검증)
- `OrderStatus`: CREATED, PAID (CANCELED는 STEP07).
- `OrderProduct` → Order만 `@ManyToOne`(같은 애그리거트). 백참조 `setOrder`는 package-private.
- `@OneToMany cascade`로 Order 저장 시 OrderProduct 같이 저장 (OrderProductRepository 없음). STEP05에서 cascade 정리.
- ⚠️ **빈 주문 검증 누락** — `Order.create`에 `if (products == null || products.isEmpty()) throw ...` **추가 필요** (레퍼런스엔 있음, 본인 4755dfb 스냅샷엔 없었음). → 다음 작업/#25에서 보완.

### Payment

- **userId 없음**, `create(orderId, amount)` — 결제는 잠깐 받는 입력(orderId로 충분). 레퍼런스 최종본도 userId 제거.
- `PaymentStatus`: READY/COMPLETED/FAILED/CANCELED (WAITING 제외 — 동기결제). `cannotPayable = {COMPLETED, FAILED, CANCELED}`.
- `PaymentMethod`: BALANCE(기본)/KAKAO_PAY/TOSS_PAY/NAVER_PAY.
- `paidAt` 보유 (인기상품 쿼리에 필요).
- PaymentInfo 없음 (결제는 반환값 없이 처리만).

### Stock

- `Stock.deduct(int)` — 부족 시 예외(`this.quantity < quantity`). 비관적 락 X (STEP05 #26 동시성용).
- `StockCommand.OrderProducts`(리스트 래퍼) + `OrderProduct`(productId, quantity).

### 잔액 한도

- 현재 `Balance.charge`에 최대 한도 검증 없음. 요구사항에 있으면 STEP05 #29 리팩토링 때 함께.

---

## 5. 진행 현황 — 전체 이슈/서브이슈 트리

> 주당 1 STEP 페이스. 부모 이슈(STEPxx) 아래 서브이슈가 달린 구조.

### STEP01 — 설계 기본과제 (#1) ✅

- #10 요구사항 분석 문서 작성
- #11 마일스톤 문서화 및 GitHub 연동
- #12 시퀀스 다이어그램 작성
- #13 ERD 설계 및 작성
- #14 API 명세 작성

### STEP02 — 설계 심화과제 (#2) ✅

- #15 Mock API 구현
- #16 Spring REST Docs 문서화
- #17 E2E 테스트 작성
- #18 API Request http 파일 작성
- #19 상태 다이어그램 작성

### STEP03 — 도메인 구현: 잔액/쿠폰/상품 (#3) ✅

- #20 잔액 비즈니스 로직 구현 및 단위 테스트
- #21 쿠폰 비즈니스 로직 구현 및 단위 테스트
- #22 상품 비즈니스 로직 구현 및 단위 테스트

### STEP04 — 도메인 구현: 주문/결제 (#4) ★ 진행 중

| 이슈  | 내용                            | 상태                     |
| --- | ----------------------------- | ---------------------- |
| #23 | 주문/결제 비즈니스 로직 + 단위 테스트        | ✅ 머지 (PR #75)          |
| #24 | 인프라 레이어 구현체                   | ✅ 머지 (#23과 통합, PR #75) |
| #25 | 기능별 통합 테스트 (← OrderFacade 포함) | ⬜ **다음**               |
| #26 | 주요 기능별 동시성 실패 테스트 작성          | ⬜                      |
| #27 | 병목 예상 쿼리 분석 및 최적화 보고서 작성      | ⬜                      |

### STEP05 — 동시성 이슈 해결 (#5)

- #28 주요 기능별 동시성 테스트 작성
- #29 주요 기능 동시성 이슈 식별 및 해결
- #30 동시성 이슈 분석 및 해결 보고서 작성
- #31 Filter/Interceptor/Scheduler 부가 로직 구현
- #32 모든 API 정상 작동 및 가용성 확보

### STEP06 — DB 성능 최적화 / Redis (#6)

- #33 Redis 기반 분산락 구현 및 적용
- #34 Redis 분산락 동시성 보고서 추가
- #35 Redis 기반 캐싱 전략 설정 및 적용
- #36 캐싱 전략 및 성능 개선 보고서 작성
- #37 인기상품 Redis 기반 설계 및 구현
- #38 선착순 쿠폰발급 Redis 기반 설계 및 구현
- #39 Redis 디자인 설계 보고서 작성

### STEP07 — MSA 기반 이벤트 아키텍처 (#7)

- #43 주문/결제 완료 시 이벤트 기반 외부 데이터 플랫폼 전송
- #44 파사드 클래스 제거 및 이벤트 기반 도메인 서비스 구현
- #45 MSA 기반 이벤트 아키텍처 설계 문서 작성

### STEP08 — 카프카 활용 (#8)

- #46 카프카 기초 및 핵심 개념 문서 작성
- #47 주문 완료 시 데이터 플랫폼으로 카프카 메시지 발행
- #48 대용량 트래픽 프로세스 카프카 활용 구현
- #49 Outbox 패턴 적용
- #50 카프카 기반 설계 문서 작성

### STEP09 — 부하테스트 및 장애대응 (#9)

- #51 부하테스트 대상 선정 및 시나리오 계획 문서 작성
- #52 부하테스트 스크립트 작성
- #53 부하테스트 결과 기반 병목 탐색 및 개선
- #54 성능 테스트 및 장애대응 보고서 작성

> ⚠️ STEP05~09 서브이슈에도 #24처럼 "도메인/인프라 분리"식으로 쪼개진 게 있으면 6장 교훈 적용(한 PR로 묶기). 특히 STEP05 #28/#29(동시성 테스트/해결)는 같은 코드를 건드리므로 함께 진행 고려.

### #25에서 할 일 (가장 큰 덩어리)

1. **OrderFacade** 작성 — 상품조회→재고차감→쿠폰사용→잔액차감→결제→주문저장, **`@Transactional`로 한 트랜잭션**.
2. Facade가 호출할 **미구현 메서드** 추가:
    - `ProductService.getOrderProducts` (주문용 상품 조회 + 판매검증 + name/price 스냅샷)
    - `CouponService` 할인율 조회 (현재 useCoupon만, discountRate 가져오는 것 없음)
    - 주문은 `userCouponId`를 받는 게 본인 useCoupon과 정합
3. **interfaces** 재작성 — OrderController, OrderRequest, OrderResponse.
4. **통합 테스트** — `@SpringBootTest`로 전체 흐름 검증.
5. (잊지 말 것) **빈 주문 검증** Order.create에 추가 + 테스트.

---

## 6. 이슈 분할 교훈 ★ (STEP04에서 배움)

- **도메인 / 인프라를 별도 이슈로 쪼개지 말 것.**
    - 인터페이스만 만들고 구현체(빈)를 안 만들면 → 빈 부재로 **`contextLoads()` 실패** (앱이 안 뜸).
    - STEP03(잔액/쿠폰/상품)은 "도메인+인프라"를 한 묶음으로 해서 이 문제가 없었음.
    - 레퍼런스도 도메인+인프라를 **한 PR**에 했음(인터페이스+스텁 구현체를 같이 둬서 빈이 항상 존재).
- **STEP05~09: 도메인+인프라를 한 이슈/한 PR로** 묶어 진행. 쪼갠다면 최소한 같은 브랜치/PR에서 처리.
- 이미 쪼갠 이슈는 PR 본문에 `Close #A` `Close #B`로 함께 닫고, 통합 이유 한 줄 기록.

---

## 7. 커밋 / PR 규칙

### 커밋 태그

`[FEAT]` 기능 / `[FIX]` 정정 / `[TEST]` 테스트 / `[REFACTOR]` 리팩토링 / `[DOCS]` 문서. **"건드린 게 뭐냐"로 태그 결정.** 한 작업에 여러 성격이 섞이면 주된 성격으로.

### 커밋 순서 (도메인 한 덩어리 예시)

```
[FEAT] 도메인 모델 → [FEAT] DTO → [FEAT] Repository 인터페이스
→ [FEAT] Service → [FEAT] 인프라 구현체 → [TEST] 도메인 단위 → [TEST] Service
```

### PR

- 본문에 작업 요약 + `Close #xx` (PR 본문/커밋 메시지에서만 자동 닫힘; 이미 닫힌 다른 이슈에 써도 무효).
- 첫 push: `git push -u origin <branch>` (이후 `git push`).

---

## 8. 트러블슈팅 누적

|#|증상|원인 / 해결|
|---|---|---|
|#8|예외 타입 제각각|전부 `IllegalArgumentException` 통일 (레퍼런스는 IllegalState 섞음)|
|#11|빈 파일이 머지됨|add 후 추가 수정분 누락. 커밋 전 `git status` 필수|
|#12|계산 테스트가 못 잡음|상품 수량·가격을 다른 값으로 둬야 합산 오류 검출|
|#13|기억 기반 안내가 어긋남|세션 시작 시 실제 코드 먼저 읽기|
|#14|enum 헬퍼 한쪽만 테스트|IN-list 헬퍼는 true/false 양방향|
|#16|staged/unstaged 섞임|add 후 또 수정 → 옛 버전 커밋 위험. 매 add 뒤 `git status`|
|**#17**|**컴파일 에러 `cannot find symbol getProductPrice()`**|**DTO 필드명 오타 (`quantityPrice`→`productPrice`). compileJava가 잡음**|
|**#18**|**`BeanCurrentlyInCreationException` (순환참조)**|**CoreRepository가 도메인 인터페이스(자기 자신)를 주입. → `JpaRepository`를 주입해야**|
|**#19**|**`NoSuchBeanDefinitionException` contextLoads 실패**|**인터페이스만 있고 구현체(빈) 없음. 인프라 구현체 추가 또는 도메인+인프라 한 묶음**|
|**#20**|**인프라가 `interfaces/`에 들어감**|**인프라 구현체는 `infrastructure/`(+`jpa/`). `interfaces/`는 Controller/Request/Response. `git mv` + package 선언 수정**|

### 참고: 무관한 경고

- 테스트 로그의 `drop/create table user` H2 syntax error → `user`가 H2 예약어라 나는 경고. 현재 실패 원인 아님(User 엔티티 테이블명은 추후 별건으로).

---

## 9. devlog / 회고 작성 규칙

- STEP(또는 PR) 단위로 회고 작성: 한 일 / 막힌 점(트러블 번호 연결) / 결정과 이유 / 배운 것.
- "왜 이렇게 설계했나"는 **코드 주석이 아니라 PR 본문·회고에** (코드는 이름·도메인 메서드로 말함).
- 비직관적 결정(예: 특정 상수값)만 코드 주석으로.