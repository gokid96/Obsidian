## 전체 구조

```
[사용자]
    ↓ HTTPS
[Cloudflare DNS]
    ↓
[AWS ALB] (ap-northeast-2a / ap-northeast-2c)
    ↓ Round Robin
┌───────────────┬───────────────┐
│   EC2-1       │   EC2-2       │
│ 3.37.79.4     │ 43.200.44.47  │
│               │               │
│ Docker        │ Docker        │
│ untitles-api  │ untitles-api  │
│ :8070         │ :8070         │
│               │               │
│ Caffeine      │ Caffeine      │
│ Cache (로컬)  │ Cache (로컬)  │
└───────┬───────┴───────┬───────┘
        └───────┬───────┘
                ↓
           [AWS RDS]
           MySQL 8.0
           (ap-northeast-2a)
```

---

## 인프라

|구성 요소|상세|
|---|---|
|**ALB**|Application Load Balancer, Round Robin, AZ-A/C|
|**EC2-1**|탄력적 IP `3.37.79.4`, 인스턴스 `i-010bcef68fdb077c0`|
|**EC2-2**|탄력적 IP `43.200.44.47`, 인스턴스 `i-06380465df597885a`|
|**RDS**|MySQL 8.0, `untitles-db.c7si8i2ao8v3.ap-northeast-2.rds.amazonaws.com`|
|**Cloudflare R2**|이미지 스토리지, `images.untitles.net`|
|**AWS SES**|이메일 발송 (이메일 인증)|

---

## 애플리케이션 스택

|구성 요소|상세|
|---|---|
|**Language**|Java 21|
|**Framework**|Spring Boot 3.5.7|
|**Build**|Gradle 8.5|
|**Container**|Docker (eclipse-temurin:21-jre)|

---

## 인증

```
일반 로그인
[클라이언트] → POST /auth/login → JWT 발급 (AccessToken 30분 / RefreshToken 7일)
               → localStorage에 토큰 저장

OAuth2 로그인
[클라이언트] → Google/Kakao/Naver → OAuth2 콜백
             → JWT 발급 → /oauth/callback?accessToken=xxx&refreshToken=xxx
             → localStorage에 토큰 저장

인증 방식: Stateless JWT (서버에 세션 없음)
→ ALB Round Robin 환경에서도 어느 서버로 가든 독립적으로 검증 가능
```

---

## 캐시 (현재 문제)

```
EC2-1 Caffeine Cache          EC2-2 Caffeine Cache
┌─────────────────────┐       ┌─────────────────────┐
│ workspaceTree:2014  │       │ workspaceTree:2014  │
│ → [folder1, 2, 3]  │       │ → [folder1, 2]      │ ← 불일치!
└─────────────────────┘       └─────────────────────┘

문제: EC2-1에서 폴더 생성 시 EC2-1 캐시만 삭제됨
      EC2-2는 이전 캐시를 그대로 유지 → 캐시 불일치 발생
해결: Redis 분산 캐시 도입 필요
```

---

## 배포 파이프라인

```
[GitHub main 브랜치 push]
    ↓
[GitHub Actions]
    ↓ Build & Push
[GHCR (GitHub Container Registry)]
    ↓ SSH 배포
┌───────────────┬───────────────┐
│   EC2-1       │   EC2-2       │
│ docker pull   │ docker pull   │
│ docker run    │ docker run    │
└───────────────┴───────────────┘
```

---

## 모니터링

### CloudWatch

|메트릭|설명|
|---|---|
|EC2-1 CPUUtilization|EC2-1 CPU 사용률|
|EC2-2 CPUUtilization|EC2-2 CPU 사용률|
|RDS CPUUtilization|DB CPU 사용률|
|RDS DatabaseConnections|DB 커넥션 수 (현재 약 32개)|
|RDS FreeStorageSpace|DB 디스크 여유 공간 (현재 약 19.22G)|
|ALB HealthyHostCount|정상 서버 수 (정상: 2)|
|ALB UnHealthyHostCount|비정상 서버 수 (정상: 0)|

### Grafana + Prometheus

- WSL Ubuntu에서 Docker Compose로 실행
- EC2-1, EC2-2 각각 직접 스크랩 (`http://IP:8070/actuator/prometheus`)
- Spring Boot JVM, HTTP 응답시간, DB 커넥션 등 상세 메트릭

---

## 프론트엔드

|구성 요소|상세|
|---|---|
|**Framework**|Vue 3|
|**Build Tool**|Vite|
|**배포**|Vercel (`untitles.net`, `www.untitles.net`)|
|**API 연동**|`api.untitles.net` (ALB DNS)|

---

## 현재 이슈 및 개선 계획

| 구분          | 내용                                    | 상태        |
| ----------- | ------------------------------------- | --------- |
| 캐시 불일치      | Caffeine 로컬 캐시로 서버 간 캐시 불일치 발생        | ❌ 미해결     |
| Redis 도입    | 분산 캐시로 캐시 불일치 해결                      | 🔜 예정     |
| RDS 퍼블릭 액세스 | RDS에 불필요한 탄력적 IP 연결                   | 🔜 정리 예정  |
| prod SQL 로그 | application-prod.yml에 DEBUG 레벨 SQL 로그 | 🔜 정리 예정  |
| RateLimit   | RateLimitFilter @Component 주석 처리 상태   | 🔜 활성화 예정 |


---
