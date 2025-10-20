Comment 엔티티가 Post나 User와 연관관계를 맺고 있을 때, Hibernate 프록시 객체를 JSON으로 변환하려다 발생하는 오류

### **핵심 원인 Lazy Loading + JSON 직렬화 충돌**

```java
public class Comment {
@ManyToOne (fetch = FetchType.LAZY)  // ← 이게 문제!
@JoinColumn(name = "post_id")
private Post post;
}
```

## 상세 설명
```
댓글 생성 → DB 저장 ✅
↓
Comment 객체 반환 시도
↓
Jackson이 JSON 변환 시도
↓
post 필드 접근 → Hibernate Proxy 객체 발견 ❌
↓
에러 발생!
```

###  **왜 Proxy 객체가 문제인가?**
`LAZY` 로딩 시:
```java
public class CommentService {
public Comment createComment(Long postId, Long authorId, String content) {
Comment comment = new Comment();
comment.setPost(???)  // 실제 Post 객체가 아니라
                      // Hibernate의 ByteBuddyInterceptor 프록시!
}
```

Jackson이 이 프록시를 JSON으로 변환하려다 실패합니다.

###  **로그 확인**
```
✅ 댓글 작성 완료 - commentId: 1  (DB 저장은 성공!)
❌ HttpMessageConversionException    (JSON 변환 실패!)
```

----
## ByteBuddyInterceptor 프록시란?

### **실제 객체 vs 프록시 객체**
```java
// EAGER 로딩 (즉시 로딩)
Comment comment = commentRepository.findById(1L);
comment.getPost(); // 👉 실제 Post 객체
// Post{id=1, title="게시글", content="내용"}

// LAZY 로딩 (지연 로딩)
Comment comment = commentRepository.findById(1L);
comment.getPost(); // 👉 프록시 객체 (가짜!)
// Post$ByteBuddyInterceptor$abcd1234
```

### **프록시는 왜 만들어지나?**
```java
public class Comment {
@ManyToOne(fetch = FetchType.LAZY)
private Post post;
}
```

이렇게 설정하면:
- Comment만 조회할 때는 Post를 DB에서 안 가져옴 (성능 최적화)
- 대신 **빈 껍데기(프록시)**를 넣어둠 프록시 객체
- 나중에 `comment.getPost().getTitle()` 호출할 때 그때 DB 조회

```java
// Hibernate의 작동 방식
Comment comment = new Comment();
comment.id = 1;
comment.content = "댓글내용";
comment.post = new Post$ByteBuddyInterceptor(); // 👈 프록시!
                                                  // 아직 DB 조회 안 함
```

## 2. JSON 변환 실패하면 어떻게 되는가?

### **결과: 클라이언트(Postman)에 에러 응답이 감**
```json
// ❌ 이렇게 응답됨 (500 Internal Server Error)
{
    "timestamp": "2025-09-29T17:42:43.657+09:00",
    "status": 500,
    "error": "Internal Server Error",
    "message": "Type definition error",
    "path": "/api/comments"
}
```

### **단계별로 보면:**

```
1. CommentService.createComment() 실행
   ✅ DB에 댓글 저장 성공!
   
2. Controller가 Comment 객체 반환
   @PostMapping
   public Comment createComment(...) {
       return commentService.createComment(...); // ← 여기까지 성공
   }
   
3. Spring이 Comment를 JSON으로 변환 시도 (클라이언트에 응답용 JSON)
   Comment → JSON 변환 (Jackson 라이브러리 사용)
   ❌ 프록시 객체 발견 → 변환 실패!
   
4. 클라이언트에게 500 에러 응답
   Postman에 에러 메시지 표시
```
----
요약 
1. 엔티티에 @ManyToOne (fetch = FetchType.LAZY)  사용 시 Comment만 조회할 때는 Post를 DB에서 안 가져오고 대신 **빈 껍데기(프록시)**를 넣어둠 프록시 객체
2. 댓글 작성 DB 저장 후 return sevedComment; 에서 클라이언트에 응답(JSON 직렬화)을 보내줘야 하는데 빈 껍데기 (프록시 객체) 이기 때문에 org.hibernate.proxy.pojo.bytebuddy.ByteBuddyInterceptor 에러 발생
3. 빈 껍데기를 무시하거나 제외하기 또는 근본적인 문제해결을 위해 DTO패턴으로 수정
