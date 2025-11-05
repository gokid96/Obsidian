### 1단계: 개인 블로그 (메모+발행)

```
✅ 사용자 인증 (JWT)
✅ 게시글 CRUD (마크다운)
✅ 임시저장 / 발행 시스템
✅ 카테고리, 태그
✅ 시리즈
✅ 검색
✅ 공개 블로그 페이지
✅ 이미지 업로드
```

### 2단계: 소셜 기능

```
👥 팔로우 / 팔로워
💬 댓글 시스템
👍 좋아요 / 반응
🔔 알림 (팔로잉 새 글, 댓글)
📰 피드 (팔로잉한 사람 글)
```

### 3단계: 네트워킹 (커피챗)

```
☕ 커피챗 신청/관리
💬 실시간 1:1 채팅 (WebSocket)
📅 일정 관리
🔔 채팅 알림
✍️ 후기 시스템
```


## 🗄️ Astarchia 1단계 ERD

```
┌─────────────────────────────────────────────────────────────┐
│                          User (회원)                          │
├─────────────────────────────────────────────────────────────┤
│ PK  userId             BIGINT                                │
│     email              VARCHAR(100)  UNIQUE, NOT NULL       │
│     login_id           VARCHAR(50)   NOT NULL
	  password           VARCHAR(255)  NOT NULL               │
│     username           VARCHAR(50)   UNIQUE, NOT NULL       │

│     updated_at         TIMESTAMP     NOT NULL               │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ 1:N
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Post (게시글)                            │
├─────────────────────────────────────────────────────────────┤
│ PK  id                 BIGINT                                │
│ FK  user_id            BIGINT        NOT NULL               │
│ FK  category_id        BIGINT        NULL                   │
│ FK  series_id          BIGINT        NULL                   │
│     title              VARCHAR(200)  NOT NULL               │
│     content            TEXT          NOT NULL               │
│     summary            VARCHAR(300)  NULL                   │
│     thumbnail_url      VARCHAR(500)  NULL                   │
│     status             ENUM          NOT NULL (DRAFT/PUBLISHED) │
│     visibility         ENUM          NOT NULL (PUBLIC/PRIVATE)  │
│     view_count         INT           DEFAULT 0              │
│     slug               VARCHAR(200)  UNIQUE                 │
│     published_at       TIMESTAMP     NULL                   │
│     created_at         TIMESTAMP     NOT NULL               │
│     updated_at         TIMESTAMP     NOT NULL               │
└─────────────────────────────────────────────────────────────┘
              │                           │
              │ N:M                       │ 1:N
              ▼                           ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│   PostTag (중간 테이블)    │    │    Comment (댓글) *2단계*  │
├──────────────────────────┤    ├──────────────────────────┤
│ PK  id         BIGINT    │    │ 나중에 추가 예정          │
│ FK  post_id    BIGINT    │    └──────────────────────────┘
│ FK  tag_id     BIGINT    │
│     created_at TIMESTAMP │
└──────────────────────────┘
              │
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│                        Tag (태그)                             │
├─────────────────────────────────────────────────────────────┤
│ PK  id                 BIGINT                                │
│     name               VARCHAR(50)   UNIQUE, NOT NULL       │
│     slug               VARCHAR(50)   UNIQUE, NOT NULL       │
│     created_at         TIMESTAMP     NOT NULL               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     Category (카테고리)                        │
├─────────────────────────────────────────────────────────────┤
│ PK  id                 BIGINT                                │
│ FK  user_id            BIGINT        NOT NULL               │
│     name               VARCHAR(50)   NOT NULL               │
│     slug               VARCHAR(50)   NOT NULL               │
│     display_order      INT           DEFAULT 0              │
│     created_at         TIMESTAMP     NOT NULL               │
│     updated_at         TIMESTAMP     NOT NULL               │
│                                                              │
│     UNIQUE(user_id, name)                                   │
│     UNIQUE(user_id, slug)                                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Series (시리즈)                          │
├─────────────────────────────────────────────────────────────┤
│ PK  id                 BIGINT                                │
│ FK  user_id            BIGINT        NOT NULL               │
│     name               VARCHAR(100)  NOT NULL               │
│     description        TEXT          NULL                   │
│     slug               VARCHAR(100)  NOT NULL               │
│     thumbnail_url      VARCHAR(500)  NULL                   │
│     created_at         TIMESTAMP     NOT NULL               │
│     updated_at         TIMESTAMP     NOT NULL               │
│                                                              │
│     UNIQUE(user_id, name)                                   │
│     UNIQUE(user_id, slug)                                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Image (이미지)                             │
├─────────────────────────────────────────────────────────────┤
│ PK  id                 BIGINT                                │
│ FK  user_id            BIGINT        NOT NULL               │
│     original_name      VARCHAR(255)  NOT NULL               │
│     stored_name        VARCHAR(255)  UNIQUE, NOT NULL       │
│     url                VARCHAR(500)  NOT NULL               │
│     size               BIGINT        NOT NULL               │
│     mime_type          VARCHAR(50)   NOT NULL               │
│     created_at         TIMESTAMP     NOT NULL               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 주요 설계 포인트

### 1. **User (회원)**

- `blog_url`: 개인 블로그 주소 (`astarchia.com/@username`)
- `username`: URL용 고유 식별자
- `display_name`: 화면에 표시되는 이름

### 2. **Post (게시글)**

- `status`: `DRAFT`(임시저장) / `PUBLISHED`(발행)
- `visibility`: `PUBLIC`(공개) / `PRIVATE`(비공개)
- `slug`: SEO 친화적 URL (`/posts/my-first-post`)
- `published_at`: 발행 시점 기록 (임시저장 시 NULL)

### 3. **Category (카테고리)**

- 사용자별로 생성 (user_id 필수)
- `display_order`: 순서 조정 가능
- UNIQUE 제약: 같은 사용자는 중복 카테고리명 불가

### 4. **Series (시리즈)**

- 연재물 관리
- 썸네일 지원
- 사용자별 고유

### 5. **Tag (태그)**

- 전역 태그 (모든 사용자가 공유)
- `slug`: URL용 (`/tags/javascript`)
- N:M 관계 (PostTag 중간 테이블)

### 6. **Image (이미지)**

- 업로드한 이미지 메타데이터 관리
- `stored_name`: 서버 저장용 고유 이름
- `original_name`: 사용자가 올린 원본 파일명