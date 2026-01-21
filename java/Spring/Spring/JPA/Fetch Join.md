### 1. JPA 기본 제공 쿼리

- `findAll()`, `findById()` 등
- 연관 필드는 **Proxy(가짜 객체)** 로 가져옴
- 연관 필드 접근 시 추가 쿼리 발생

### 2. JPQL 일반 JOIN

```java
SELECT m FROM Member m JOIN m.team t
```

- 연관 필드에 값이 있는 객체만 조회
- 연관 필드는 **Proxy**로 가져옴
- 조건 필터링 용도로 사용

### 3. JPQL Fetch JOIN

```java
SELECT m FROM Member m JOIN FETCH m.team
```

- 연관 필드에 값이 있는 객체만 조회
- 연관 필드를 **실제 객체**로 가져옴
- N+1 문제 해결 용도로 사용

### 비교표

|구분|조회 대상|연관 필드|
|---|---|---|
|기본 쿼리|전체|Proxy|
|일반 JOIN|연관 있는 것만|Proxy|
|Fetch JOIN|연관 있는 것만|실제 객체|

### N+1 문제

```java
List<Member> members = memberRepository.findAll();  // 쿼리 1번

for (Member m : members) {
    m.getTeam().getName();  // 회원 수만큼 쿼리 추가 발생
}
```

- 회원 100명이면 쿼리 101번 실행
- Fetch JOIN 사용하면 **쿼리 1번**으로 해결