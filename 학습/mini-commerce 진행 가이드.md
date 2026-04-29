
> 레퍼런스 레포(discphy/e-commerce, 274 커밋)의 시간순 분석을 통해 도출한 단계별 진화 가이드

---

## 이 문서의 목적

레퍼런스는 **STEP01부터 STEP09까지 단계마다 코드를 다시 갈아엎으며** 진화해왔다. 처음부터 완성형으로 만든 게 아니라, **단순한 형태로 시작 → 문제 발견 → 리팩토링** 사이클을 반복했다.

이 문서는 그 진화 패턴을 STEP별로 정리하여, **"지금 단계에서 무엇을 하지 말아야 하는지"** 를 명확히 한다.

---

##  핵심 원칙 (절대 어기지 말 것)

### 1️⃣ STEP을 건너뛰지 마라

레퍼런스도 처음엔 락 없이, 캐시 없이, 이벤트 없이 구현했다. 조급해서 STEP05의 락을 STEP03에 넣거나, STEP07의 이벤트를 STEP04에 넣으면 학습 효과가 사라진다.

### 2️⃣ 문서가 코드보다 먼저다

레퍼런스 패턴: `[DOCS] 보고서 작성` → `[REFACTOR] 코드 변경` 순서. 보고서로 **AS-IS / 문제 / 해결방안 / TO-BE**를 정리한 뒤 코드를 바꾼다.

### 3️⃣ `[REFACTOR]`는 죄가 아니다

레퍼런스는 274 커밋 중 **약 35%가 `[REFACTOR]` 커밋**이다. 처음부터 완벽한 코드를 짜려 하지 말고, 일단 동작하게 만든 뒤 단계마다 개선한다.

### 4️⃣ 같은 코드를 여러 번 갈아엎는다

특히 **인기상품 조회 기능**은 레퍼런스에서 **5번** 갈아엎혔다. DB 집계 → 배치 → Redis 캐시 → Redis 자료구조 → 실시간 이벤트. 한 번에 끝낼 생각하지 말고, 단계마다 점진적으로 진화시킨다.

---

## STEP별 진행 가이드

### STEP01 - 설계 기본 ✅ 본인 완료

**레퍼런스 흐름** (2025-03-31 ~ 04-01)

```
[DOCS] 요구사항 분석 문서 작성
[DOCS] 마일스톤 문서 작성
[DOCS] 시퀀스 다이어그램 작성
[DOCS] ERD 작성
[DOCS] API 명세 작성
[DOCS] 요구사항 문서 수정     ← 후속 단계 진행하며 수차례 수정됨
[DOCS] ERD 수정
[DOCS] 시퀀스 다이어그램 수정
```

**핵심 학습 포인트**

- 설계 문서는 **STEP09까지 계속 수정**된다. 처음부터 완벽할 필요 없음
- 본인 docs/ 산출물은 레퍼런스 수준에 도달함

---

### STEP02 - 설계 심화 ✅ 본인 완료

**레퍼런스 흐름** (2025-04-02 ~ 04-03)

```
[FEAT] Mock API 작성                                  ← 컨트롤러 + 하드코딩 응답
[DOCS] Spring REST Docs 작성                          ← 문서화 자동화
[REFACTOR] OrderController 해피케이스 테스트
[DOCS] Spring REST Docs 문서화
[DOCS] 상태 다이어그램 추가
[DOCS] http 테스트 추가                               ← .http 파일로 수동 테스트
[DOCS] ERD 상태 및 타입 정의 추가                      ← 상태 enum 정리
[DOCS] ERD 설계 의도 작성                             ← "왜 이렇게 설계했는지"
```

**핵심 학습 포인트**

- Mock API는 **반드시 `[FEAT]` 커밋**, 진짜 구현이라고 간주
- REST Docs를 STEP02에 미리 세팅해두면 STEP03부터 자동으로 문서가 갱신됨
- 본인 `RestDocsSupport`, `ControllerTestSupport` 분리는 정확한 패턴

**본인 다음 액션**: 없음 (이미 완료)

---

### STEP03 - 도메인 구현 (잔액/쿠폰/상품)  **다음 작업**

**레퍼런스 흐름** (2025-04-10 ~ 04-12)

```
[FEAT] Mock 테스트 지원 클래스 추가
[FEAT] 선착순 쿠폰 발급 어플리케이션 구현              ← Service 레이어 (락 없음!)
[FEAT] 잔액 충전 어플리케이션 구현
[FEAT] 주문 및 결제 어플리케이션 구현
[FEAT] 잔액 조회 어플리케이션 구현
[FEAT] 상품 조회 어플리케이션 구현
[FEAT] 보유 쿠폰 목록 조회 어플리케이션 구현
[FEAT] 상위 상품 조회 어플리케이션 구현
[REFACTOR] 인터페이스 패키지 구조 변경
[REFACTOR] JPA 구현체 추가                            ← 인프라 레이어
[REFACTOR] 정적 팩토리 메서드 추가
[REFACTOR] 잔액 도메인 리팩토링
[FEAT] 도메인 클래스 검증 추가                        ← 엔티티에 비즈니스 규칙
[REFACTOR] 주문 도메인 리팩토링
[REFACTOR] 메서드 추상화
[REFACTOR] 주문 서비스 테스트 코드 작성
[REFACTOR] 주문 서비스 리팩토링
```

**핵심 학습 포인트 — 절대 하지 말 것**

- ❌ `@Version` 낙관적 락 추가 금지 (STEP05에서 추가)
- ❌ `@Lock(PESSIMISTIC_WRITE)` 비관적 락 추가 금지 (STEP05에서 추가)
- ❌ `@Cacheable` 캐시 추가 금지 (STEP06에서 추가)
- ❌ `ApplicationEventPublisher` 이벤트 발행 금지 (STEP07에서 추가)
- ❌ Redis, Kafka 의존성 추가 금지

**핵심 학습 포인트 — 반드시 할 것**

- ✅ 엔티티에 **비즈니스 메서드** 작성 (`balance.charge()`, `balance.use()`)
- ✅ 엔티티 메서드 안에서 **검증 로직 캡슐화** (`MAX_BALANCE_AMOUNT` 초과 등)
- ✅ Repository는 **인터페이스(domain/) + 구현체(infrastructure/)** 분리
- ✅ 정적 팩토리 메서드(`Balance.create(userId)`, `BalanceTransaction.ofCharge(...)`) 사용
- ✅ 단위 테스트 → 구현 → 통합 테스트 순서

**본인 첫 PR 추천**

브랜치: `feat/step03-balance-domain`

체크리스트:

- [ ] `Balance` 엔티티 + `charge/use/refund` 비즈니스 메서드
- [ ] `BalanceTransaction` 엔티티 + `transaction_type` enum
- [ ] `BalanceRepository` 인터페이스 (도메인 레이어)
- [ ] `BalanceCoreRepository` 구현체 (인프라 레이어, JPA 위임)
- [ ] `BalanceJpaRepository`, `BalanceTransactionJpaRepository` (Spring Data JPA)
- [ ] `BalanceService` (chargeBalance, useBalance, getBalance)
- [ ] `BalanceCommand`, `BalanceInfo` (도메인 입출력 객체)
- [ ] `BalanceController` 에서 mock 응답 제거 → Service 호출
- [ ] `Balance` 단위 테스트 (충전 한도, 차감 부족 등)
- [ ] `BalanceService` 단위 테스트 (Mockito)
- [ ] `BalanceIntegrationTest` (`@SpringBootTest` + Testcontainers MySQL)
- [ ] `ApiControllerAdvice`에 `IllegalArgumentException` 핸들러 추가
- [ ] `BalanceControllerDocsTest` 갱신 (실제 Service 동작)

**작업 순서**

1. 단위 테스트 먼저 작성 (`BalanceTest`)
2. 엔티티 구현 (테스트 통과시키기)
3. Service 단위 테스트 작성 (`BalanceServiceTest`, Repository는 Mock)
4. Service 구현
5. Repository 인터페이스 + JPA 구현체
6. 통합 테스트 작성
7. Controller mock 제거 + Service 연결
8. REST Docs 테스트 갱신
9. PR → 머지 → 이슈 #3-1 close

---

### STEP04 - 도메인 구현 (주문/결제) ⏳

**레퍼런스 흐름** (2025-04-13 ~ 04-15)

```
[FEAT] 통합테스트 위한 설정 추가 및 변경              ← Testcontainers 설정
[FEAT] 사용자 도메인 Infra Layer 구현 및 통합테스트
[FEAT] 잔액 도메인 Infra Layer 구현 및 통합테스트
[FEAT] 쿠폰, 사용자 쿠폰 도메인 Infra Layer 구현
[FEAT] 상품 도메인 Infra Layer 구현 및 통합테스트
[FEAT] 재고 도메인 Infra Layer 구현 및 통합테스트
[FEAT] 주문 도메인 Infra Layer 구현 및 통합테스트
[FEAT] 결제 도메인 Infra Layer 구현 및 통합테스트
[REFACTOR] 파사드 트랜잭션 적용                       ← OrderFacade 등장!
[FEAT] 주문/결제 파사드 통합 테스트 작성
[FEAT] 상품 파사드 통합 테스트 작성
[REFACTOR] @Enumerated(EnumType.STRING) 적용
[DOCS] DB 성능 최적화 보고서 작성                     ← STEP04 마지막 산출물
[REFACTOR] 엔티티 클래스 인덱스 적용                  ← 보고서 결과 반영
```

**핵심 학습 포인트**

- ✅ `OrderFacade` 패턴 도입: `BalanceService`, `CouponService`, `StockService`를 조합
    - 파사드 = "여러 도메인 서비스를 호출하는 트랜잭션 경계"
    - **STEP07에서 이 파사드를 제거하고 이벤트로 대체**할 것이므로, 지금은 단순하게 작성
- ✅ `@Enumerated(EnumType.STRING)` 필수 (ORDINAL 사용 금지)
- ✅ STEP04 끝물에 **DB 성능 보고서 작성** + 인덱스 적용

**미리 알아두기 — STEP07 대비 설계 팁**

- 파사드의 메서드를 트랜잭션 단위로 명확히 분리해두면 나중에 이벤트 분리가 쉽다
- `OrderFacade.createOrder()` 안에서 한 번에 다 하지 말고, 단계별 메서드로 추출

---

### STEP05 - 동시성 ⏳

**레퍼런스 흐름** (2025-04-17 ~ 04-25)

```
[TEST] 동시성 테스트 작성                             ← 실패 테스트 먼저!
[TEST] 데이터 클렌징 추가 및 동시성 리팩토링
[REFACTOR] 재고 비관적 락 동시성 제어 구현
[REFACTOR] 쿠폰 비관적 락 동시성 제어 구현
[REFACTOR] 잔액 낙관적 락 동시성 제어 구현
[REFACTOR] 사용자 쿠폰 유니크 제약조건 추가 및 검증 로직 추가
[REFACTOR] "XxxWithLock" postfix 로 일관성 있게 수정
[FEAT] 로깅 필터 적용
[REFACTOR] 예외 처리 추가
[DOCS] 동시성 이슈 분석 및 해결 보고서 작성           ← 핵심 산출물
[TEST] RestAssured E2E Test 추가
[REFACTOR] Order 인덱스 적용
[REFACTOR] Rank 인덱스 적용
```

**핵심 학습 포인트 — 락 전략 (검증 완료)**

|자원|전략|이유|
|---|---|---|
|잔액|**낙관적 락** (`@Version`)|동일 사용자가 동시 충전 = 의도치 않은 중복. 하나만 처리|
|쿠폰|**비관적 락** (`@Lock(PESSIMISTIC_WRITE)`)|선착순은 모두 처리되어야 함. 충돌 잦음|
|재고|**비관적 락**|동시 차감 시 음수 방지. 정확성 최우선|

**작업 순서 (반드시 이 순서로)**

1. **실패하는 동시성 테스트 작성** (락 적용 전, `ExecutorService` + `CompletableFuture`)
2. 테스트 실패 확인 (의도된 실패)
3. 락 적용
4. 테스트 성공 확인
5. 보고서 작성 (AS-IS 결과 / TO-BE 결과 스크린샷 포함)

**미리 알아두기**

- 비관적 락은 **공정성을 보장하지 않는다**. 진짜 선착순은 STEP08에서 Kafka로 해결
- DB 락 + Redis 분산 락 **2중 적용**은 STEP06에서 추가됨 (지금은 DB 락만)

---

### STEP06 - DB 성능 + 캐시 

**레퍼런스 흐름** (2025-04-23 ~ 05-15)

**Phase 1: Redis 분산 락 도입** (2025-04-30)

```
[CHORE] Redis 컨테이너 및 Redisson 추가
[REFACTOR] Test Container 리팩토링
[FEAT] Redis 프로덕션 코드 설정
[FEAT] 분산 락 AOP 적용             ← @DistributedLock 어노테이션
[FEAT] 선착순 쿠폰 분산 락 적용       ← STEP05 비관적 락에 추가
```

**Phase 2: 인기상품 캐시** (2025-05-04 ~ 05-07)

```
[FEAT] Redis 캐시 관련 클래스 작성
[TEST] Redis 캐시 클렌징 코드 추가
[REFACTOR] Product 도메인에서 "인기 상품 조회" 기능 제거
[FEAT] Rank 도메인 "인기 상품 조회" 기능 추가
[REFACTOR] 인기 상품 조회 배치 프로세스 전환 (4개 커밋)
[REFACTOR] 인기 상품 조회 스케줄러 적용
[TEST] 캐시 성능 테스트 위한 K6 스크립트 작성
[DOCS] 캐시 전략 설계 보고서 작성                     ← TTL 49h 설계
```

**Phase 3: Redis 자료구조 전환** (2025-05-15)

```
[FEAT] Redis 키 관련 인터페이스 추가
[FEAT] 인기상품 Redis 자료구조 전환                   ← Sorted Set
[FEAT] Redis 자료구조 키 클렌징 클래스 추가
[REFACTOR] 쿠폰 도메인 - 발급 가능 목록 조회 및 발급 종료 기능
[DOCS] Redis 디자인 아키텍처 보고서 작성
[FEAT] 인기상품 - Redis에서 40일 이후 DB 영속화 스케줄러
```

**핵심 학습 포인트**

**캐시 전략 (검증 완료)**

- **읽기**: Read-Through (`@Cacheable`)
- **쓰기**: Write-Through (`@CachePut` + 매일 00:05 스케줄러)
- **TTL**: 49시간 (24시간이면 배치 시각과 만료 겹침 + hotfix 여유 확보)

**작업 순서**

1. 분산 락 도입 (쿠폰 발급에 적용 — STEP05 비관적 락 위에 **추가**, 대체 X)
2. 인기상품 도메인을 Product에서 Rank로 분리
3. 배치로 인기상품 집계 → DB 저장
4. Redis 캐시 적용 (`@Cacheable` + TTL 49h)
5. K6로 캐시 성능 측정
6. Redis Sorted Set으로 자료구조 전환

**미리 알아두기**

- 분산 락은 **DB 락을 대체하지 않는다**. Redis 장애 시에도 정합성 보장 위해 둘 다 유지
- 인기상품은 STEP06에서 **배치**, STEP07에서 **실시간 이벤트**로 또 갈아엎힘

---

### STEP07 - EDA (이벤트 기반) 

**레퍼런스 흐름** (2025-05-21 ~ 05-22)

**Phase 1: 외부 데이터 플랫폼 전송 분리** (2025-05-21)

```
[REFACTOR] Order interfaces 레이어 api 패키지 하위로 이동
[REFACTOR] Message 도메인 개념 추출
[REFACTOR] 주문 결제시, 외부 플랫폼 데이터 전송 방식 이벤트 기반으로 변경
```

**Phase 2: 도메인 이벤트 작성 + 파사드 제거** (2025-05-22, **하루 만에 폭풍 작업**)

```
[FEAT] 비동기 설정 추가                               ← @EnableAsync
[FEAT] Order 이벤트 작성
[FEAT] Balance 이벤트 작성
[FEAT] Coupon 이벤트 작성
[FEAT] Payment 이벤트 작성
[FEAT] Stock 이벤트 작성
[FEAT] Message 이벤트 작성
[FEAT] Rank 이벤트 작성
[REFACTOR] MSA 기반 Balance 파사드 클래스 제거
[REFACTOR] MSA 기반 Product 파사드 클래스 제거
[REFACTOR] MSA 기반 Rank 파사드 클래스 제거
[REFACTOR] MSA 기반 Coupon 파사드 클래스 제거
[REFACTOR] MSA 기반 Order 파사드 클래스 제거
[REFACTOR] Balance 잔액 환불 구현                     ← 보상 트랜잭션
[REFACTOR] Coupon 사용 취소 구현
[REFACTOR] Payment 결제 취소 구현
[REFACTOR] Stock 재고 복구 구현
[REFACTOR] @Transactional 누락 추가
[DOCS] MSA 기반 이벤트 아키텍처 설계 보고서 작성
```

**핵심 학습 포인트**

**이벤트 패턴 (검증 완료)**

```java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(OrderEvent.Created event) {
    // 트랜잭션 커밋 후 비동기 처리
}
```

**작업 순서**

1. 가장 단순한 케이스(외부 데이터 플랫폼 전송)부터 이벤트로 분리
2. 도메인별 이벤트 객체 정의 (`OrderEvent.Created`, `BalanceEvent.Used` 등)
3. 이벤트 리스너 작성 (각 도메인 패키지에 `XxxEventListener`)
4. 파사드 클래스를 도메인별로 하나씩 제거
5. 보상 트랜잭션 메서드 추가 (refund, cancel, restore)

**미리 알아두기**

- 보상 트랜잭션(Saga 패턴)이 STEP07에서 등장. 결제 실패 시 잔액 환불, 재고 복구
- `@TransactionalEventListener`의 `AFTER_COMMIT`이 핵심. 트랜잭션 커밋된 후에만 발행
- STEP08에서 이 이벤트 발행처를 Kafka로 교체

---

### STEP08 - Kafka 

**레퍼런스 흐름** (2025-05-29 ~ 06-02)

```
[FEAT] Kafka 설정
[REFACTOR] 기존 Message 도메인 제거
[FEAT] 데이터 역직렬화 클래스 작성
[FEAT] Message - Kafka 인터페이스 작성
[FEAT] Event 객체 생성 및 토픽, 컨슈머 그룹 정의
[FEAT] Outbox 구현                                    ← 트랜잭션-메시지 정합성
[REFACTOR] 주문 완료 시 외부 데이터 플랫폼 전송 → Kafka
[DOCS] 카프카 기초 및 핵심 개념 문서 작성
[REFACTOR] Kafka를 활용한 쿠폰 발급 프로세스 변경    ← 선착순 순서 보장!
[FEAT] 쿠폰 이벤트 리스너 작성
[TEST] 쿠폰 발급 요청 및 발급 완료 이벤트 테스트
[REFACTOR] 결제 시, 포인트 차감 및 쿠폰 사용 로직 추가
[FEAT] 주문/결제 관련 이벤트 리스너 작성
[DOCS] 쿠폰 발급 프로세스 카프카 기반 설계 문서 작성
[REFACTOR] @TransactionalEventListener AFTER_COMMIT 변경
[REFACTOR] 인기상품 배치 제거 및 실시간 이벤트 변경  ← 인기상품 5번째 진화
[REFACTOR] Outbox 수동 제거, 카프카 메시지 발행 수정
```

**핵심 학습 포인트**

- ApplicationEvent 발행처 → Kafka 발행처로 교체 (리스너 코드는 거의 그대로)
- **Outbox 패턴**: 트랜잭션 + 메시지 발행 정합성 보장
- 쿠폰 발급을 Kafka로 직렬 처리 → STEP05에서 못 풀었던 "공정성" 문제 해결
- 인기상품을 배치 → 실시간 이벤트로 전환 (5번째 진화)

---

### STEP09 - 부하테스트 

**레퍼런스 흐름** (2025-06-05 ~ 06-06)

```
[CHORE] 도커 및 어플리케이션 부하 테스트 환경 구축
[FEAT] 주문 상세 조회 API 구현                        ← 부하 테스트 시나리오용
[FEAT] 상품 조회 커서 페이징 적용
[FEAT] 부하 테스트 픽스쳐 작성
[FEAT] 부하 테스트 스크립트 작성
[DOCS] 부하 테스트 대상 선정 및 목적, 시나리오 보고서
[REFACTOR] 카프카 에러 로깅 추가 작성
[REFACTOR] 비동기 에러 로깅 추가 작성
[REFACTOR] 카프카 리스너 Concurrency 옵션 추가        ← 부하테스트 결과 반영
[REFACTOR] 카프카 리스너 CoreException 핸들링
[DOCS] 부하 테스트 성능 지표 분석, 병목 탐색 및 개선
```

**핵심 학습 포인트**

- 부하테스트 대상은 **STEP06에서 만든 K6 스크립트 확장**
- 측정 → 병목 발견 → 개선 → 재측정 사이클
- 결과는 보고서로 (성능 지표, 병목 원인, 개선 전후 비교)

---

###  마지막: 멀티모듈 분리 (옵션, STEP08~09 이후)

**레퍼런스 흐름** (2025-07-07, **하루 만에**)

```
[REFACTOR] 공통:캐시 모듈 분리
[REFACTOR] 공통:클라이언트 모듈 분리
[REFACTOR] 공통:락 모듈 분리
[REFACTOR] 공통:메시지 모듈 분리
[REFACTOR] 공통:아웃박스 모듈 분리
[REFACTOR] 공통:직렬화 모듈 분리
[REFACTOR] 공통:스토리지 모듈 분리
[REFACTOR] 서비스:잔액 모듈 분리
[REFACTOR] 서비스:쿠폰 모듈 분리
[REFACTOR] 서비스:주문 모듈 분리
[REFACTOR] 서비스:결제 모듈 분리
[REFACTOR] 서비스:상품 모듈 분리
[REFACTOR] 서비스:유저 모듈 분리
[REFACTOR] 지원:RestDocs 모듈 분리
[REFACTOR] 기존 모놀리식 코드 삭제
```

**핵심 학습 포인트**

- 단일 모듈로 STEP09까지 완성 후, 멀티모듈로 분리
- 본인 의도(레퍼런스 따라가기)에 맞음: 큰 리팩이 어떻게 일어나는지 직접 체험

---

## 📐 작업 표준

### 브랜치 명명

```
docs/step01-requirements           ← 요구사항 분석 문서 작성
feat/step03-balance-domain         ← Balance 도메인 구현
test/step05-concurrency            ← 동시성 테스트 추가
refactor/step07-order-event        ← 주문 이벤트 변환
```

### 커밋 메시지

| 태그           | 사용처       | 예시                                 |
| ------------ | --------- | ---------------------------------- |
| `[DOCS]`     | 문서 작성/수정  | `[DOCS] 동시성 이슈 분석 및 해결 보고서 작성`     |
| `[FEAT]`     | 새 기능 추가   | `[FEAT] Balance 도메인 비즈니스 메서드 구현`   |
| `[TEST]`     | 테스트 코드 작성 | `[TEST] BalanceService 단위 테스트 작성`  |
| `[REFACTOR]` | 기존 코드 개선  | `[REFACTOR] 잔액 낙관적 락 동시성 제어 구현`    |
| `[FIX]`      | 버그 수정     | `[FIX] 분산락, 트랜잭션 해제 순서 보장 문제 수정`   |
| `[CHORE]`    | 빌드/설정     | `[CHORE] Redis 컨테이너 및 Redisson 추가` |
| `[REVERT]`   | 되돌리기      | `[REVERT] 캐시 delimiter 수정`         |

### PR 사이클

```
1. 이슈 #N 확인 (sub-issue 단위)
2. 브랜치 생성 (feat/step0X-xxx)
3. TDD: 실패 테스트 → 구현 → 통과 테스트
4. 커밋 (메시지 컨벤션 준수)
5. Push → PR 생성 → 본인 셀프 리뷰
6. main 머지 → 이슈 close → 칸반 Done
```

### TDD 사이클 (STEP03~04에서 반드시)

```
1. 단위 테스트 작성 (실패)
2. 엔티티/서비스 구현 (테스트 통과)
3. 리팩토링
4. 통합 테스트 작성
5. REST Docs 테스트 갱신
```

---

## 🚦 단계별 신호등 — "지금 이걸 해도 되는가?"

| 작업                            | STEP03 | STEP04 | STEP05 | STEP06 | STEP07 | STEP08 |
| ----------------------------- | ------ | ------ | ------ | ------ | ------ | ------ |
| 도메인 엔티티 비즈니스 메서드              | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     |
| JPA Repository 구현             | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     |
| `@Transactional`              | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     |
| 단위/통합 테스트                     | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     |
| 파사드 클래스 도입                    | 🔴     | 🟢     | 🟢     | 🟢     | 🔴(제거) | 🔴     |
| 인덱스 적용                        | 🔴     | 🟢     | 🟢     | 🟢     | 🟢     | 🟢     |
| `@Version` 낙관적 락              | 🔴     | 🔴     | 🟢     | 🟢     | 🟢     | 🟢     |
| `@Lock` 비관적 락                 | 🔴     | 🔴     | 🟢     | 🟢     | 🟢     | 🟢     |
| Redis 분산 락                    | 🔴     | 🔴     | 🔴     | 🟢     | 🟢     | 🟢     |
| `@Cacheable` 캐시               | 🔴     | 🔴     | 🔴     | 🟢     | 🟢     | 🟢     |
| 배치 스케줄러                       | 🔴     | 🔴     | 🔴     | 🟢     | 🔴(제거) | 🔴     |
| `ApplicationEventPublisher`   | 🔴     | 🔴     | 🔴     | 🔴     | 🟢     | 🟢     |
| `@TransactionalEventListener` | 🔴     | 🔴     | 🔴     | 🔴     | 🟢     | 🟢     |
| Kafka Producer/Consumer       | 🔴     | 🔴     | 🔴     | 🔴     | 🔴     | 🟢     |
| Outbox 패턴                     | 🔴     | 🔴     | 🔴     | 🔴     | 🔴     | 🟢     |

🟢 = 사용 가능 / 🔴 = 사용 금지 (STEP를 먼저 완료할 것)

---

##  레퍼런스 통계 (참고)

- **총 커밋**: 274
- **개발 기간**: 2024-12-27 ~ 2025-07-09 (약 6.5개월)
- **STEP당 평균 PR**: 2~3개 (week2/1, week2/2 식)
- **보고서 7개**: DB 성능, 동시성, 캐시 전략, Redis 디자인, MSA EDA, Kafka 디자인, 부하테스트
- **커밋 태그 분포** (대략):
    - `[REFACTOR]` 35%
    - `[FEAT]` 30%
    - `[DOCS]` 25%
    - `[TEST]` 8%
    - `[FIX]/[CHORE]/[REVERT]` 2%

**시사점**: `[REFACTOR]`가 가장 많다. 즉 **점진적 개선이 핵심**. 한 번에 완벽하게 만들려 하지 말 것.

---

##  이 문서 사용법

1. STEP 시작 전 → 해당 STEP의 "레퍼런스 흐름" 섹션 정독
2. "절대 하지 말 것" 체크 → 다음 STEP의 도구를 미리 끌어다 쓰지 않기
3. 작업 중 → "신호등 표"로 현재 STEP에서 허용된 도구 확인
4. STEP 완료 → 보고서 작성 (단계마다 산출물 누적)
5. 막히면 → 레퍼런스 동일 시점 커밋 확인 (`git log --grep="키워드"`)

---

> 📌 이 문서는 살아있는 문서다. STEP 진행 중 새로운 깨달음이나 함정이 발견되면 갱신할 것.