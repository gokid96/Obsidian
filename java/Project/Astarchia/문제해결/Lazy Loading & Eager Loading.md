### 1. 개념 정의

#### Eager Loading (즉시 로딩)

> 엔티티 조회 시 **연관된 엔티티도 즉시 함께** 조회하는 방식

#### Lazy Loading (지연 로딩)

> 엔티티 조회 시 **연관된 엔티티는 실제 사용할 때** 조회하는 방식

---
### 3. 상황 설정

```java
@Entity
public class Member {
    @Id
    private Long id;
    private String name;
    
    @ManyToOne
    private Team team;  // Member는 Team에 소속
}
```

**질문**: Member를 조회할 때, Team도 같이 가져올까?

---

### 4. 적용 방법

#### Eager Loading

```java
@ManyToOne(fetch = FetchType.EAGER)
private Team team;
```

```java
Member member = memberRepository.findById(1L);
// Member + Team 둘 다 조회 완료
```


```sql
SELECT m.*, t.* 
FROM member m 
JOIN team t ON m.team_id = t.id
WHERE m.id = 1;
```

#### Lazy Loading

```java
@ManyToOne(fetch = FetchType.LAZY)
private Team team;
```

```java
Member member = memberRepository.findById(1L);
// Member만 조회됨, Team은 프록시 객체

member.getTeam().getName();
// 이 순간 Team 조회 쿼리 발생!
```

```sql
-- findById 시점
SELECT * FROM member WHERE id = 1;

-- getTeam().getName() 시점
SELECT * FROM team WHERE id = 10;
```

---

### 5. 프록시 객체란?

Lazy Loading 시 JPA가 만드는 **가짜 객체**

```java
Member member = memberRepository.findById(1L);

System.out.println(member.getTeam().getClass());
// 출력: Team$HibernateProxy$xxx (진짜 Team이 아님!)

member.getTeam().getName();  // 이 순간 진짜 Team 조회
```

|시점|team 상태|
|---|---|
|findById 직후|프록시 객체 (빈 껍데기)|
|getName() 호출|실제 DB 조회 → 진짜 Team|

---

### 6. 왜 Lazy를 쓰나?
```java
public String getMemberName(Long id) {
    Member member = memberRepository.findById(id);
    return member.getName();  // Team 안 씀!
}
```

|전략|결과|
|---|---|
|Eager|Team도 조회함 → **낭비**|
|Lazy|Team 조회 안 함 → **효율적**|

---

### 7. JPA 기본값

|관계|기본 전략|
|---|---|
|`@ManyToOne`|EAGER|
|`@OneToOne`|EAGER|
|`@OneToMany`|LAZY|
|`@ManyToMany`|LAZY|

**실무에서는 전부 LAZY로 설정 권장!**

---

### 8. Lazy의 함정 → N+1 문제
```java
List<Member> members = memberRepository.findAll();  // 1번 쿼리

for (Member member : members) {
    member.getTeam().getName();  // N번 쿼리 발생!
}
```

**Lazy 자체가 문제가 아니라, 반복문에서 연관 객체 접근 시 N+1 발생!**

---

### 9. 정리

|구분|Eager|Lazy|
|---|---|---|
|조회 시점|즉시|접근할 때|
|장점|한 번에 가져옴|필요한 것만 가져옴|
|단점|불필요한 조회|N+1 문제 가능|
|실무|거의 안 씀|**권장**|

**결론**: Lazy 사용 + Fetch Join으로 N+1 해결