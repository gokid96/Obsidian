# 트러블슈팅

## 1. API 응답 속도 지연 (Cloudflare 프록시)

### 문제 상황

Cloudflare 프록시 ON 상태에서 API 응답이 비정상적으로 느림

| 항목 | 값 |
|------|-----|
| 측정 구간 | 클라이언트 → api.astarchia.com |
| 응답 시간 | **521ms** |
| 기대 수준 | ~50ms |

### 원인 분석

**CF-RAY 헤더로 라우팅 위치 확인**

```bash
curl -I https://api.astarchia.com
# CF-RAY: 9b2e0d9b4d6ec8a7-HKG ← 홍콩
```

Cloudflare 무료 플랜에서 한국 트래픽이 홍콩 PoP로 라우팅됨.

```
클라이언트(한국) → Cloudflare(홍콩) → EC2(서울) → Cloudflare(홍콩) → 클라이언트
```

### 대안 비교

| 방안 | 속도 | 보안 | 비용 |
|------|------|------|------|
| 프록시 OFF + 자체 SSL | ~50ms | IP 노출 | 무료 |
| Argo Smart Routing | 빠름 | IP 숨김 | 월 $5+ |
| AWS CloudFront | 서울 보장 | IP 숨김 | 종량제 |
| 현 상태 유지 | ~500ms | IP 숨김 | 무료 |

### 의사결정

**프록시 OFF + EC2 자체 SSL 선택**

- 현재 트래픽 규모에서 DDoS 방어보다 사용자 경험(속도) 우선
- Certbot으로 SSL 관리 부담은 자동 갱신으로 최소화
- 향후 트래픽/보안 요구 증가 시 Argo 또는 CloudFront 전환

### 적용

```bash
# Certbot으로 Let's Encrypt 인증서 발급
sudo certbot --nginx -d api.astarchia.com

# 자동 갱신 확인
sudo certbot renew --dry-run
```

### 결과

| 항목 | Before | After | 개선 |
|------|--------|-------|------|
| API 응답 시간 | 521ms | 70ms | **86%** |

### 교훈

Cloudflare 무료 플랜은 지역에 따라 원거리 PoP로 라우팅될 수 있음. API 서버처럼 지연에 민감한 경우 프록시 사용 전 실제 라우팅 경로 확인 필요.

---

## 2. 폴더 이동 시 순환 참조 위험

### 문제 상황

자기 참조 관계로 폴더 계층을 구현했는데, 이동 시 순환 참조 가능성 발견

```
폴더 A
└── 폴더 B
    └── 폴더 C

A를 C의 하위로 이동하면?
C → A → B → C → A → ... (무한 순환)
```

### 위험성

- 조회 시 무한 루프
- 데이터 무결성 파괴
- 서비스 장애

### 해결

**이동 전 순환 여부 검증**

```java
@Transactional
public FolderResponseDTO moveFolder(Long folderId, Long newParentId) {
    Folder folder = folderRepository.findById(folderId).orElseThrow();
    Folder newParent = folderRepository.findById(newParentId).orElseThrow();
    
    // 1. 자기 자신으로 이동 차단
    if (folder.getFolderId().equals(newParentId)) {
        throw new IllegalArgumentException("자기 자신으로 이동할 수 없습니다.");
    }
    
    // 2. 하위 폴더로 이동 차단 (순환 방지)
    Folder current = newParent.getParent();
    while (current != null) {
        if (current.getFolderId().equals(folderId)) {
            throw new IllegalArgumentException("하위 폴더로 이동할 수 없습니다.");
        }
        current = current.getParent();
    }
    
    folder.moveToParent(newParent);
    return FolderResponseDTO.from(folder);
}
```

### 시간 복잡도

O(depth) - 폴더 깊이만큼 순회. 현재 깊이 제한 없으나, 실사용에서는 수십 레벨 이내로 예상되어 문제없음.

### 대안 검토

| 방식 | 장점 | 단점 |
|------|------|------|
| 현재 (이동 시 검증) | 구현 단순 | 깊은 트리에서 검증 비용 |
| DB 트리거 | 애플리케이션 무관하게 보장 | DB 종속, 디버깅 어려움 |
| Closure Table | 순환 자체 불가능한 구조 | 구현 복잡, 이동 시 갱신 많음 |

현재 요구사항에서는 검증 방식으로 충분.

### 교훈

자기 참조 관계에서는 순환 참조 가능성을 항상 고려해야 함. 특히 사용자 입력으로 관계가 변경되는 경우 반드시 검증 필요.

---

## 3. 세션 vs JWT 전환

### 상황

초기에 JWT로 구현 후, 세션으로 전환

### JWT 구현 시 겪은 문제

**로그아웃 처리의 복잡함**

JWT는 Stateless라 서버에서 토큰을 무효화할 방법이 마땅치 않음.

- 토큰 블랙리스트: Redis 등 별도 저장소 필요 → Stateless 장점 상실
- 짧은 만료 시간: UX 저하 (자주 재로그인)
- Refresh Token: 구현 복잡도 증가

**보안 우려**

토큰 탈취 시 만료까지 유효. 세션은 서버에서 즉시 무효화 가능.

### 전환 결정

| 고려 사항 | JWT | Session |
|-----------|-----|---------|
| 현재 서버 구성 | 단일 서버라 장점 없음 | 단순 |
| 로그아웃 | 복잡 | session.invalidate() |
| 보안 | 탈취 시 대응 어려움 | 즉시 무효화 |
| 향후 확장 | 좋음 | Redis 필요 |

현재 단일 서버에서는 세션이 더 적합하다고 판단.

### 적용

```java
// 로그인
session.setAttribute(
    HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
    SecurityContextHolder.getContext()
);

// 로그아웃
SecurityContextHolder.clearContext();
session.invalidate();
```

### 확장 시 계획

다중 서버 전환 시:
1. **Redis 세션**: 기존 코드 변경 최소화
2. **JWT 재도입**: 모바일 앱 등 추가 시 검토

### 교훈

기술 선택은 현재 상황에 맞게. "JWT가 트렌드니까" 보다 실제 요구사항과 운영 환경 고려.
