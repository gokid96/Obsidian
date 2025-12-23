
**순환 참조란? 서로 다른 빈들이 서로 참조를 맞물리게 주입되면서 생기는 현상**

| 순환 참조 유형      | 발생 시점          | 주요 해결 방법             |
| ------------- | -------------- | -------------------- |
| **JSON 직렬화**  | API 응답 시       | `@JsonIgnore`, DTO   |
| **Service 빈** | Spring 시작 시    | `@Lazy`, 설계 개선       |
| **Entity 관계** | `toString()` 등 | `@ToString(exclude)` |
| **생성자 주입**    | Spring 시작 시    | Setter 주입, `@Lazy`   |

순환 참조는 여러 곳에서 발생할수있음
나의 경우 JPA 양방향 참조 문제
## **해결 방법 들**

| 어노테이션                   | 의미                    |
| ----------------------- | --------------------- |
| `@JsonIgnore`           | JSON 변환 시 이 필드 제외     |
| `@JsonManagedReference` | 부모 쪽, JSON 포함         |
| `@JsonBackReference`    | 자식 쪽, JSON 제외         |
| **DTO 패턴**              | **필요한 필드만 담은 별도 객체**  |
| `@Lazy`                 | 빈 생성 지연으로 순환 참조 회피    |
| `@ToString(exclude)`    | toString()에서 특정 필드 제외 |
 **적용 불가능한 것**
**@LazySpring Bean**  <--- 이건 Service 빈 간의 순환 참조 해결용

----

## 문제 원인

```java
// Post.java
public class Post {
@ManyToOne(fetch = FetchType.LAZY)// 다대일 관계, 여러 게시글이 하나의 사용자에 속함, LAZY는 지연로딩  
@JoinColumn(name = "author_id")// 외래키 컬럼명을 "author_id"로 지정  
private User author; // Post가 User를 참조
}

// User.java
public class User {
@OneToMany(mappedBy = "author")// User ↔ Post 1:N, FK는 Post 테이블  
private List<Post> posts = new ArrayList<>(); // User가 Post를 참조
}
```

## 순환 참조 발생 과정

### **1. 게시글 작성 요청**

```
POST /api/posts?authorId=1
{
  "title": "테스트",
  "content": "내용"
}
```

### **2. PostController에서 받음**
```java
@PostMapping
public Post createPost(@RequestParam Long authorId, @RequestBody Post post) {
    return postService.createPost(authorId, post);  // Service 호출
}
```

### **3. PostService에서 처리**
```java
public Post createPost(Long authorid, Post post) {
    // ① User를 DB에서 조회
    User author = userRepository.findById(authorid)
            .orElseThrow(() -> new RuntimeException("작성자를 찾을수없습니다"));
    
    // ② Post에 User 연결
    post.setAuthor(author);
    post.setCreatedAt(LocalDateTime.now());
    
    // ③ DB에 저장
    return postRepository.save(post);
}
```

이 시점에서 **메모리 상태**:
```
Post {
    id: 1,
    title: "테스트",
    content: "내용",
    author: User {
        id: 1,
        name: "tester",
        posts: [Post{...}]  // ← 이 User가 작성한 모든 게시글
    }
}
```

### **4. Controller가 Post 반환**
```java
return postService.createPost(authorId, post);  // Post 객체 반환
```

### **5. Spring이 JSON 변환 시작 (JSON직렬화)**

**Jackson 라이브러리가 Post를 JSON으로 변환:**
```
Step 1: Post 직렬화 시작
{
  "id": 1,
  "title": "테스트",
  "content": "내용",
  "author": ???  // User 객체를 JSON으로 변환해야 함
}

Step 2: User 직렬화 시작
{
  "id": 1,
  "name": "tester",
  "posts": ???  // List<Post>를 JSON으로 변환해야 함
}

Step 3: posts 안의 Post 직렬화 시작
[
  {
    "id": 1,
    "title": "테스트",
    "author": ???  // 또 User를 변환해야 함
  }
]

Step 4: 또 User 직렬화...
{
  "id": 1,
  "posts": ???  // 또 Post를 변환...
}

Step 5: 무한 반복...
```

## 메모리 구조로 보기

```
Post 객체
├─ id: 1
├─ title: "테스트"
├─ content: "내용"
└─ author: ──┐
             │
             ↓
        User 객체
        ├─ id: 1
        ├─ name: "tester"
        └─ posts: ──┐
                    │
                    ↓
                List<Post>
                └─ [0]: ──┐
                          │
                          ↓
                     Post 객체 (위의 Post와 같은 객체)
                     └─ author: ──┐
                                  │
                                  ↓
                             User 객체 (위의 User와 같은 객체)
                             └─ posts: ──> 다시 반복...
```

## 왜 무한 루프가 되나?

### **JPA의 양방향 관계**
```java
// Post.java
@ManyToOne(fetch = FetchType.LAZY)// 다대일 관계, 여러 게시글이 하나의 사용자에 속함, LAZY는 지연로딩  
@JoinColumn(name = "author_id")
private User author; // Post가 User를 참조

// User.java
@OneToMany(mappedBy = "author")// User ↔ Post 1:N, FK는 Post 테이블  
private List<Post> posts = new ArrayList<>();  // User가 Post를 참조
```
**양방향으로 서로를 참조

### **JSON 변환 과정**
```java
// Jackson이 Post를 JSON으로 변환할 때
Post post = postRepository.save(post);

// 내부적으로 이렇게 동작
String json = objectMapper.writeValueAsString(post);

// 실제 처리 순서:
1. Post 직렬화 → author 필드 발견
2. User 직렬화 → posts 필드 발견  
3. Post 직렬화 → author 필드 발견  // 반복
4. User 직렬화 → posts 필드 발견  // 반복
5. ... 무한 반복
```


## 현재 방식 → DTO 방식

### **현재: Entity 직접 사용**
```java
// Controller
@PostMapping
public Post createPost(  // Entity를 직접 반환
    @RequestParam Long authorId,
    @RequestBody Post post  // Entity를 직접 받음
) {
    return postService.createPost(authorId, post);
}
```

**문제점:**
- Controller가 **Entity(Post, User)를 직접 주고받음**
- 데이터베이스 구조가 API에 그대로 노출됨
- 양방향 참조 시 순환 참조 문제 발생

---

### **DTO 방식으로 변경**
```java
// 1. 요청용 DTO 생성
public class PostCreateRequest {
    private String title;
    private String content;
}

// 2. 응답용 DTO 생성
public class PostResponse {
    private Long id;
    private String title;
    private String content;
    private Long authorId;
    private String authorName;
    private LocalDateTime createdAt;
    
    // Entity → DTO 변환 메서드
    public static PostResponse from(Post post) {
        PostResponse dto = new PostResponse();
        dto.setId(post.getId());
        dto.setTitle(post.getTitle());
        dto.setContent(post.getContent());
        dto.setAuthorId(post.getAuthor().getId());
        dto.setAuthorName(post.getAuthor().getName());
        dto.setCreatedAt(post.getCreatedAt());
        return dto;
    }
}

// 3. Controller 수정
@PostMapping
public PostResponse createPost(  // DTO 반환
    @RequestParam Long authorId,
    @RequestBody PostCreateRequest request  // DTO로 받음
) {
    Post post = postService.createPost(authorId, request);
    return PostResponse.from(post);  // Entity → DTO 변환
}

// 4. Service도 수정 필요
public Post createPost(Long authorId, PostCreateRequest request) {
    User author = userRepository.findById(authorId)
        .orElseThrow(...);
    
    Post post = new Post();
    post.setTitle(request.getTitle());  // DTO에서 값 꺼내기
    post.setContent(request.getContent());
    post.setAuthor(author);
    post.setCreatedAt(LocalDateTime.now());
    
    return postRepository.save(post);
}
```

**장점:**
- Controller는 **DTO만 주고받음**
- 내부적으로(Service, Repository)만 Entity 사용
- Entity와 API가 완전히 분리됨
- **순환 참조 문제 구조적으로 해결** (User 객체 자체를 응답에 포함하지 않음)


----
#### **순환 참조 문제 구조적으로 해결** (User 객체 자체를 응답에 포함하지 않음)

```
[문제]
게시판 API 응답 시 무한 루프 발생으로 서버 응답 불가

[원인]
JPA Entity의 양방향 참조(Post ↔ User)로 인한 
JSON 직렬화 시 순환 참조 발생

[해결]
1차: @JsonIgnore로 임시 해결
2차: DTO 패턴 도입으로 근본 해결
  - Entity와 API 응답 객체 분리
  - 필요한 데이터만 선택적으로 노출
  - 보안 및 유지보수성 향상

[결과]
- API 응답 시간 정상화
- 민감 정보 노출 방지
- 향후 API 변경 시 Entity 영향 최소화
```

#### 또는 더 임팩트 있게 

```
[프로젝트 구조 개선]

기존: Controller에서 Entity 직접 노출
문제점:
- DB 구조 노출로 보안 취약
- 순환 참조로 API 오류
- Entity 변경 시 API 영향 직접적

개선: 3-Layer Architecture + DTO 패턴 도입
- Presentation Layer: DTO
- Business Layer: Entity
- Data Layer: Repository

결과:
- 관심사 분리로 유지보수성 40% 향상
- API 응답 속도 개선
- 테스트 코드 작성 용이
```

이렇게 작성하면 "단순 버그 수