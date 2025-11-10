# 기본 구조

쿼리 메서드는 메서드 이름으로 쿼리를 자동 생성합니다. 기본 패턴은 다음과 같습니다:

```
find/read/get/query/search/stream + By + 조건
```

## 주요 키워드

**조회 키워드:**

- `findBy`, `readBy`, `getBy`, `queryBy`, `searchBy` - 모두 같은 의미로 조회
- `existsBy` - 존재 여부 확인 (boolean 반환)
- `countBy` - 개수 세기 (long 반환)
- `deleteBy`, `removeBy` - 삭제

**조건 키워드:**

- `And` - 그리고 (여러 조건 연결)
- `Or` - 또는
- `Is`, `Equals` - 같음 (생략 가능)
- `Not` - 같지 않음
- `IsNull`, `IsNotNull` - null 체크
- `Like`, `NotLike` - 패턴 매칭
- `StartingWith`, `EndingWith`, `Containing` - 문자열 검색
- `LessThan`, `LessThanEqual` - 미만, 이하
- `GreaterThan`, `GreaterThanEqual` - 초과, 이상
- `Between` - 범위
- `In`, `NotIn` - 포함 여부
- `OrderBy` - 정렬 (Asc/Desc)
- `True`, `False` - boolean 값

## 실전 예제

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // 이름으로 찾기
    List<User> findByName(String name);
    
    // 이메일로 찾기 (Optional)
    Optional<User> findByEmail(String email);
    
    // 나이가 특정 값보다 큰 사용자
    List<User> findByAgeGreaterThan(int age);
    
    // 나이 범위로 찾기
    List<User> findByAgeBetween(int startAge, int endAge);
    
    // 이름과 이메일로 찾기 (AND)
    User findByNameAndEmail(String name, String email);
    
    // 이름 또는 이메일로 찾기 (OR)
    List<User> findByNameOrEmail(String name, String email);
    
    // 이름에 특정 문자 포함
    List<User> findByNameContaining(String keyword);
    
    // 이름으로 시작
    List<User> findByNameStartingWith(String prefix);
    
    // 정렬
    List<User> findByAgeOrderByNameAsc(int age);
    List<User> findByNameOrderByAgeDesc(String name);
    
    // 존재 여부
    boolean existsByEmail(String email);
    
    // 개수 세기
    long countByAge(int age);
    
    // 삭제
    void deleteByName(String name);
    
    // 특정 목록에 포함
    List<User> findByNameIn(List<String> names);
    
    // null 체크
    List<User> findByEmailIsNull();
    List<User> findByEmailIsNotNull();
    
    // boolean 필드
    List<User> findByActiveTrue();
    List<User> findByActiveFalse();
    
    // 복잡한 조건
    List<User> findByNameAndAgeGreaterThanOrderByNameAsc(String name, int age);
    
    // 페이징
    Page<User> findByAge(int age, Pageable pageable);
    
    // Top/First (상위 N개)
    List<User> findTop3ByOrderByAgeDesc();
    User findFirstByOrderByNameAsc();
}
```

## 특별한 매개변수

- `Pageable` - 페이징 처리
- `Sort` - 정렬 처리
- `Limit` - 결과 개수 제한

```java
// 페이징 예제
Page<User> findByName(String name, Pageable pageable);

// 정렬 예제
List<User> findByName(String name, Sort sort);
```