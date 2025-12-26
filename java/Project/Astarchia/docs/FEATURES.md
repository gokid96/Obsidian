# 핵심 기능 설계

## 1. 워크스페이스 & 권한 관리

### 설계 의도

팀 협업 서비스에서 권한 관리는 핵심. Notion, Slack 등을 참고하여 역할 기반 접근 제어(RBAC)를 적용.

### 역할 계층

```java
public enum WorkspaceRole {
    OWNER,   // 0 - 모든 권한
    ADMIN,   // 1 - 멤버 관리, 설정 변경
    MEMBER,  // 2 - 문서 CRUD
    VIEWER   // 3 - 읽기 전용
}
```

### 권한 비교 방식

```java
private void checkPermission(WorkspaceMember member, WorkspaceRole requiredRole) {
    if (member.getRole().ordinal() > requiredRole.ordinal()) {
        throw new IllegalArgumentException("권한이 부족합니다.");
    }
}
```

**ordinal 사용 이유**
- 역할 간 계층이 명확함 (OWNER > ADMIN > MEMBER > VIEWER)
- 비교 로직이 단순해짐
- 새 역할 추가 시 enum 순서만 조정

**한계 & 대안**
- 역할 간 계층이 아닌 개별 권한이 필요하면 별도 Permission 테이블 필요
- 현재 요구사항에서는 과함

### 제약 조건과 이유

| 제약 | 이유 |
|------|------|
| OWNER 권한은 초대로 부여 불가 | 워크스페이스 소유권 보호 |
| OWNER는 나가기 불가 | 소유자 없는 워크스페이스 방지 |
| ADMIN은 OWNER 권한 변경 불가 | 권한 상승 공격 방지 |

### 향후 개선

- OWNER 양도 기능 (별도 확인 절차 포함)
- 초대 링크 기능 (만료 시간 포함)

---

## 2. 계층형 폴더 구조

### 설계 의도

무한 중첩 폴더 구조를 단순하게 구현하면서, 순환 참조 같은 데이터 무결성 문제를 방지.

### 자기 참조 관계

```java
@Entity
public class Folder {
    @Id @GeneratedValue
    private Long folderId;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Folder parent;
    
    // ...
}
```

**장점**
- 구현 단순
- 무한 중첩 가능

**단점**
- 깊은 트리 조회 시 N+1 또는 재귀 쿼리 필요
- 전체 트리 조회 비효율

### 순환 참조 방지

폴더 이동 시 자신의 하위 폴더로 이동하면 순환 발생:

```
A → B → C

C를 A의 부모로 이동하면?
C → A → B → C (순환!)
```

**해결: 이동 전 검증**

```java
public FolderResponseDTO moveFolder(Long folderId, Long newParentId) {
    // 자기 자신으로 이동 불가
    if (folder.getFolderId().equals(newParentId)) {
        throw new IllegalArgumentException("자기 자신으로 이동할 수 없습니다.");
    }
    
    // 하위 폴더로 이동 불가 (순환 방지)
    Folder current = newParent.getParent();
    while (current != null) {
        if (current.getFolderId().equals(folderId)) {
            throw new IllegalArgumentException("하위 폴더로 이동할 수 없습니다.");
        }
        current = current.getParent();
    }
    
    folder.moveToParent(newParent);
}
```

### 트리 조회 최적화 (향후)

현재는 깊이가 얕아서 문제없지만, 깊어지면:

| 방식 | 장점 | 단점 |
|------|------|------|
| Closure Table | 조회 빠름 | 삽입/이동 시 갱신 많음 |
| Materialized Path | 조회 빠름, 구현 단순 | 경로 길이 제한 |
| Nested Set | 조회 매우 빠름 | 삽입/이동 비용 높음 |

현재 요구사항에서는 자기 참조로 충분, 성능 이슈 발생 시 Materialized Path 검토.

---

## 3. 인증 시스템

### 세션 기반 선택 이유

JWT를 먼저 구현했다가 세션으로 전환함.

**전환 이유**
- 로그아웃 처리의 명확성 (JWT는 토큰 무효화 복잡)
- 현재 단일 서버에서는 세션이 더 단순
- 보안: 탈취된 세션은 서버에서 즉시 무효화 가능

**JWT가 나았을 상황**
- 다중 서버 (세션 공유 인프라 없이)
- 모바일 앱 지원
- MSA 환경

### 회원가입 흐름

```
1. 이메일 중복 확인
2. 인증번호 발송 (AWS SES)
3. 인증번호 검증
4. 회원 정보 입력 + 가입
5. 자동 로그인 (세션 생성)
6. 인증 데이터 삭제
```

**인증 데이터 삭제 이유**
- 일회용으로 설계
- 미사용 데이터 누적 방지
- 재가입 시 깨끗한 상태

### 확장 시 고려사항

| 상황 | 대응 |
|------|------|
| 다중 서버 | Redis 세션 클러스터링 |
| 모바일 앱 추가 | JWT 재검토 또는 세션 + 토큰 하이브리드 |
| 소셜 로그인 | OAuth2 + Spring Security OAuth |

---

## 4. 이메일 인증 (AWS SES)

### 설계

```
인증번호 발송 → DB 저장 (10분 만료) → 검증 → 인증 완료 마킹 → 회원가입 시 확인 → 삭제
```

### 만료 처리

```java
public boolean isExpired() {
    return LocalDateTime.now().isAfter(this.expiresAt);
}
```

**DB에서 삭제 vs 만료 플래그**
- 현재: 만료 여부를 조회 시 체크
- 스케줄러로 만료 데이터 주기적 삭제 고려 (데이터 누적 방지)

### SES 선택 이유

| 서비스 | 비용 | 설정 |
|--------|------|------|
| AWS SES | $0.10/1000건 | AWS 생태계 통합 |
| SendGrid | 무료 티어 있음 | 별도 설정 |
| 직접 SMTP | 무료 | 스팸 차단 위험 |

AWS 인프라 사용 중이라 SES 선택. 비용도 저렴.

---

## 5. 예외 처리 전략

### GlobalExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleIllegalArgument(IllegalArgumentException e) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse(e.getMessage()));
    }
    
    // ...
}
```

### 예외 타입 사용 기준

| 예외 | 용도 |
|------|------|
| IllegalArgumentException | 잘못된 입력값 |
| IllegalStateException | 잘못된 상태 (ex: 이메일 미인증 상태에서 가입 시도) |
| UsernameNotFoundException | Spring Security - 사용자 없음 |
| BadCredentialsException | Spring Security - 비밀번호 불일치 |

### 향후 개선

- 커스텀 예외 계층 구조 (BusinessException → 세부 예외)
- 에러 코드 체계화 (클라이언트 처리 용이)
