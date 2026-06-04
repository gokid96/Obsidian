> 아래 두 섹션은 진행 가이드에 추가용. **3번(채택 아키텍처) 바로 뒤**에 끼우는 걸 권장 (3-A / 3-B). 검증 표기(✅/⚠️/📌)·간결체 유지.

---

## 3-A. 코드 컨벤션 — 실제 코드 검증 (✅ balance 모듈 전체 정독)

> 추측 아님. `balance/{domain,application,infrastructure}` 실물을 읽어 확정. Coupon/Product/UserCoupon/enum도 대조 완료.

### 아키텍처 한 줄 정의

**Feature-by-package 레이어드 + 라이트 DDD(Persistence-aware).** 최상위가 도메인(`balance/`, `order/`...), 그 안에 `application/domain/infrastructure/interfaces`. (레퍼런스는 layer-by-package — `domain/`, `application/` 안에 도메인. 구조가 반대라 import 경로는 못 베끼고 번역.)

### 레이어별 컨벤션 (✅ 확정)

|레이어|클래스|규칙|
|---|---|---|
|domain|`Xxx`(엔티티)|`@Getter @Entity @NoArgsConstructor(PROTECTED)`, `@Table` **없음**(예약어만 예외), id에 `@Column(name="xxx_id")`, `@Builder` 생성자 + `static create()`, enum 필드엔 `@Enumerated(STRING)`|
|domain|`XxxStatus`(enum)|`@Getter @RequiredArgsConstructor`, `상수("설명")` + `description`, `CANNOT_XXX_STATUSES` 리스트 + `cannotXxx()`|
|domain|`XxxCommand`|outer `@NoArgsConstructor(PROTECTED)`, nested `@Getter` + `final` 필드 + `@Builder` 생성자 + `of()`|
|domain|`XxxInfo`|동일 구조. **`of()`가 엔티티를 받음** → `Info.of(entity)`|
|domain|`XxxRepository`|`@Repository` 인터페이스(메서드 선언만)|
|domain|`XxxService`|`@Service @RequiredArgsConstructor`, `@Transactional` 없음(STEP05), `Info.of(entity)` 반환|
|application|`XxxCriteria`|nested + `of()` + **`toCommand()`**|
|application|`XxxResult`|nested + **`of(info)`**|
|application|`XxxFacade`|`@Service @RequiredArgsConstructor`, `userService.getUser()` 검증 + 도메인 조합|
|infra|`XxxCoreRepository`|`@Component @RequiredArgsConstructor implements XxxRepository`, Jpa에 위임|
|infra|`jpa/XxxJpaRepository`|`extends JpaRepository<Xxx, Long>`|

- **예외**: 전부 `IllegalArgumentException` 통일 (레퍼런스는 IllegalState 섞음 — 트러블 #8 재발 방지).
- **변환 흐름**: `Request → Criteria.toCommand() → Command → [도메인] → Info.of(entity) → Result.of(info) → Response`.
- **@Table 규칙**: 안 쓰는 게 기본(네이밍 전략이 클래스명→snake_case). **`Order`만** `@Table(name="orders")` — `order`가 SQL 예약어라서. 그 외(OrderProduct/Payment 등) 전부 `@Table` 없음.
- **@Enumerated(STRING)**: 전 enum 적용(트레이드오프 아님 — ORDINAL은 enum 추가 시 기존 데이터 깨짐).

### 재사용 템플릿

Command/Info/Criteria/Result (nested + Builder + of):

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class XxxCommand {
    @Getter
    public static class Create {
        private final Long userId;
        @Builder
        private Create(Long userId) { this.userId = userId; }
        public static Create of(Long userId) { return Create.builder().userId(userId).build(); }
    }
}
```

Repository 3단 (레퍼런스의 `XxxRepositoryImpl` 1개와 다름):

```
domain/XxxRepository (인터페이스)
  ↑ implements
infrastructure/XxxCoreRepository (@Component, Jpa 위임 — 캐시/Redis 끼울 자리)
  → infrastructure/jpa/XxxJpaRepository (extends JpaRepository)
```

---

## 3-B. 레퍼런스와의 차이 + 트레이드오프 (✅ 확정)

> 레퍼런스 STEP04 시점이 단순했던 건 "아직 거기까지 안 갔을 뿐". 레퍼런스도 매주 자기 코드를 갈아엎으며 진행(git log: `41016ce 주문 도메인 리팩토링`, week4에 `@Enumerated` 추가, week8에 MSA 모듈 분리). → **고정된 정답 없음. 도메인 로직·계산식만 참고, 패키지·네이밍·예외·매핑은 본인 컨벤션.**

### 차이 요약

|항목|본인|레퍼런스|
|---|---|---|
|패키지|feature-by-package|layer-by-package|
|Repository|인터페이스 + Core + Jpa 3단|RepositoryImpl 1개(스텁→인프라단계에 JPA로 채움)|
|`Info.of()`|엔티티 받음|필드 나열|
|예외|IllegalArgumentException 통일|IllegalState 섞음|
|발급 메서드|`issue()`|`publish()`|
|@Enumerated|전 enum STRING|STEP04 일부 누락(week4 보완)|
|@Table|안 씀(`Order`만 예약어 예외)|OrderProduct 등에도 붙음|

### 트레이드오프 (양면)

|항목|본인 선택이 얻는 것|치르는 비용|
|---|---|---|
|feature-by-package|도메인 응집, MSA 분리 쉬움(종착지와 일관)|레이어 횡단 조망 어려움, 도메인 적을 땐 폴더만 많음|
|Repository 3단|도메인이 인터페이스만 의존(DIP 진짜), 캐시/Redis 끼울 자리|단순 위임 Core가 보일러플레이트|
|`Info.of(엔티티)`|필드 추가 시 of() 한 줄, 매핑 한 곳(트러블 #16 방어)|Info가 엔티티에 결합, 엔티티 없이 생성 번거로움|
|예외 통일|핸들러 하나(400), #8 재발 방지, 단순|"잘못된 값" vs "상태 위반" 의미 손실(409 세분화 시 재분리)|
|@Enumerated STRING|순서 바뀌어도 안전, DB 값 가독|약간의 저장/비교 비용(사실상 일방적 우위)|
|@Table 안 씀|간결, 클래스명=테이블명 직관|네이밍 전략 의존(명시적 아님), 예약어 수동 처리|

> 요약: 본인 선택들은 대체로 _지금 살짝 더 쓰고(보일러플레이트·결합·의미단순화) 나중 확장성·안전·일관성을 사는_ 방향. 종착지(MSA·캐시·분산락)를 알기에 미리 정렬. **면접에선 이 트레이드오프를 양면으로 말하는 게 최강 답** — 베낀 사람은 설명 못 함.

### 📌 `of()` 네이밍 노트 (선택 개선)

- 현업 관례: `from(Entity)` = 엔티티 1개→변환 / `of(값, 값...)` = 값 모아 생성. 본인의 `of(entity)`는 동작은 맞지만 관례상 **`from(entity)`** 가 더 정확(선택 사항).
- 자리(레이어 거리)에 따라 섞는 게 정석: 엔티티 바로 옆=엔티티 받기(`from`), 여러 소스 조합/응답 경계=값 받기(`of`). 규모 커지면 **MapStruct**로 변환 자동화(매핑 누락 방지)도 흔함.
- 지금 단계(엔티티→Info 1:1)는 본인 방식이 적합. 일관성만 유지.

> 아래 두 섹션은 진행 가이드에 추가용. **3번(채택 아키텍처) 바로 뒤**에 끼우는 걸 권장 (3-A / 3-B). 검증 표기(✅/⚠️/📌)·간결체 유지.

---

## 3-A. 코드 컨벤션 — 실제 코드 검증 (✅ balance 모듈 전체 정독)

> 추측 아님. `balance/{domain,application,infrastructure}` 실물을 읽어 확정. Coupon/Product/UserCoupon/enum도 대조 완료.

### 아키텍처 한 줄 정의

**Feature-by-package 레이어드 + 라이트 DDD(Persistence-aware).** 최상위가 도메인(`balance/`, `order/`...), 그 안에 `application/domain/infrastructure/interfaces`. (레퍼런스는 layer-by-package — `domain/`, `application/` 안에 도메인. 구조가 반대라 import 경로는 못 베끼고 번역.)

### 레이어별 컨벤션 (✅ 확정)

|레이어|클래스|규칙|
|---|---|---|
|domain|`Xxx`(엔티티)|`@Getter @Entity @NoArgsConstructor(PROTECTED)`, `@Table` **없음**(예약어만 예외), id에 `@Column(name="xxx_id")`, `@Builder` 생성자 + `static create()`, enum 필드엔 `@Enumerated(STRING)`|
|domain|`XxxStatus`(enum)|`@Getter @RequiredArgsConstructor`, `상수("설명")` + `description`, `CANNOT_XXX_STATUSES` 리스트 + `cannotXxx()`|
|domain|`XxxCommand`|outer `@NoArgsConstructor(PROTECTED)`, nested `@Getter` + `final` 필드 + `@Builder` 생성자 + `of()`|
|domain|`XxxInfo`|동일 구조. **`of()`가 엔티티를 받음** → `Info.of(entity)`|
|domain|`XxxRepository`|`@Repository` 인터페이스(메서드 선언만)|
|domain|`XxxService`|`@Service @RequiredArgsConstructor`, `@Transactional` 없음(STEP05), `Info.of(entity)` 반환|
|application|`XxxCriteria`|nested + `of()` + **`toCommand()`**|
|application|`XxxResult`|nested + **`of(info)`**|
|application|`XxxFacade`|`@Service @RequiredArgsConstructor`, `userService.getUser()` 검증 + 도메인 조합|
|infra|`XxxCoreRepository`|`@Component @RequiredArgsConstructor implements XxxRepository`, Jpa에 위임|
|infra|`jpa/XxxJpaRepository`|`extends JpaRepository<Xxx, Long>`|

- **예외**: 전부 `IllegalArgumentException` 통일 (레퍼런스는 IllegalState 섞음 — 트러블 #8 재발 방지).
- **변환 흐름**: `Request → Criteria.toCommand() → Command → [도메인] → Info.of(entity) → Result.of(info) → Response`.
- **@Table 규칙**: 안 쓰는 게 기본(네이밍 전략이 클래스명→snake_case). **`Order`만** `@Table(name="orders")` — `order`가 SQL 예약어라서. 그 외(OrderProduct/Payment 등) 전부 `@Table` 없음.
- **@Enumerated(STRING)**: 전 enum 적용(트레이드오프 아님 — ORDINAL은 enum 추가 시 기존 데이터 깨짐).

### 재사용 템플릿

Command/Info/Criteria/Result (nested + Builder + of):

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class XxxCommand {
    @Getter
    public static class Create {
        private final Long userId;
        @Builder
        private Create(Long userId) { this.userId = userId; }
        public static Create of(Long userId) { return Create.builder().userId(userId).build(); }
    }
}
```

Repository 3단 (레퍼런스의 `XxxRepositoryImpl` 1개와 다름):

```
domain/XxxRepository (인터페이스)
  ↑ implements
infrastructure/XxxCoreRepository (@Component, Jpa 위임 — 캐시/Redis 끼울 자리)
  → infrastructure/jpa/XxxJpaRepository (extends JpaRepository)
```

---

## 3-B. 레퍼런스와의 차이 + 트레이드오프 (✅ 확정)

> 레퍼런스 STEP04 시점이 단순했던 건 "아직 거기까지 안 갔을 뿐". 레퍼런스도 매주 자기 코드를 갈아엎으며 진행(git log: `41016ce 주문 도메인 리팩토링`, week4에 `@Enumerated` 추가, week8에 MSA 모듈 분리). → **고정된 정답 없음. 도메인 로직·계산식만 참고, 패키지·네이밍·예외·매핑은 본인 컨벤션.**

### 차이 요약

|항목|본인|레퍼런스|
|---|---|---|
|패키지|feature-by-package|layer-by-package|
|Repository|인터페이스 + Core + Jpa 3단|RepositoryImpl 1개(스텁→인프라단계에 JPA로 채움)|
|`Info.of()`|엔티티 받음|필드 나열|
|예외|IllegalArgumentException 통일|IllegalState 섞음|
|발급 메서드|`issue()`|`publish()`|
|@Enumerated|전 enum STRING|STEP04 일부 누락(week4 보완)|
|@Table|안 씀(`Order`만 예약어 예외)|OrderProduct 등에도 붙음|

### 트레이드오프 (양면)

|항목|본인 선택이 얻는 것|치르는 비용|
|---|---|---|
|feature-by-package|도메인 응집, MSA 분리 쉬움(종착지와 일관)|레이어 횡단 조망 어려움, 도메인 적을 땐 폴더만 많음|
|Repository 3단|도메인이 인터페이스만 의존(DIP 진짜), 캐시/Redis 끼울 자리|단순 위임 Core가 보일러플레이트|
|`Info.of(엔티티)`|필드 추가 시 of() 한 줄, 매핑 한 곳(트러블 #16 방어)|Info가 엔티티에 결합, 엔티티 없이 생성 번거로움|
|예외 통일|핸들러 하나(400), #8 재발 방지, 단순|"잘못된 값" vs "상태 위반" 의미 손실(409 세분화 시 재분리)|
|@Enumerated STRING|순서 바뀌어도 안전, DB 값 가독|약간의 저장/비교 비용(사실상 일방적 우위)|
|@Table 안 씀|간결, 클래스명=테이블명 직관|네이밍 전략 의존(명시적 아님), 예약어 수동 처리|

> 요약: 본인 선택들은 대체로 _지금 살짝 더 쓰고(보일러플레이트·결합·의미단순화) 나중 확장성·안전·일관성을 사는_ 방향. 종착지(MSA·캐시·분산락)를 알기에 미리 정렬. **면접에선 이 트레이드오프를 양면으로 말하는 게 최강 답** — 베낀 사람은 설명 못 함.

### 📌 `of()` 네이밍 노트 (선택 개선)

- 현업 관례: `from(Entity)` = 엔티티 1개→변환 / `of(값, 값...)` = 값 모아 생성. 본인의 `of(entity)`는 동작은 맞지만 관례상 **`from(entity)`** 가 더 정확(선택 사항).
- 자리(레이어 거리)에 따라 섞는 게 정석: 엔티티 바로 옆=엔티티 받기(`from`), 여러 소스 조합/응답 경계=값 받기(`of`). 규모 커지면 **MapStruct**로 변환 자동화(매핑 누락 방지)도 흔함.
- 지금 단계(엔티티→Info 1:1)는 본인 방식이 적합. 일관성만 유지.