# 시스템 아키텍처

## 현재 구성

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client (Browser)                         │
└─────────────────────────────────────────────────────────────────┘
                                │ HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Vercel (Frontend)                           │
│                      - Next.js                                   │
│                      - Vercel 자체 SSL                           │
└─────────────────────────────────────────────────────────────────┘
                                │ API 요청
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Cloudflare (DNS Only)                         │
│                    - 프록시 OFF (속도 이슈로 비활성화)            │
└─────────────────────────────────────────────────────────────────┘
                                │ HTTPS (Let's Encrypt)
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AWS EC2 (Backend)                           │
│                      - Spring Boot 3.5                           │
│                      - Nginx + Certbot                           │
│                      - t3.micro                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┴───────────────┐
                ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────────┐
│       AWS RDS (MySQL)      │   │          AWS SES              │
│       - db.t3.micro        │   │          - 이메일 인증        │
└───────────────────────────┘   └───────────────────────────────┘
```

---

## 기술 선택 근거

### 인증: 세션 vs JWT

| 구분 | JWT | Session |
|------|-----|---------|
| 서버 부담 | Stateless | 세션 저장 필요 |
| 로그아웃 | 토큰 무효화 복잡 | 세션 삭제로 즉시 처리 |
| 보안 | 탈취 시 만료까지 유효 | 서버에서 즉시 무효화 가능 |
| 확장성 | 좋음 | Redis 필요 |

**선택: Session**

현재 단일 서버 구성에서는 세션이 구현/운영 모두 간편함. JWT 구현 후 세션으로 전환한 이유는 로그아웃 처리의 명확성 때문.

**확장 시**: Redis 세션 클러스터링 또는 JWT + Refresh Token 재검토

---

### DB: MySQL (RDS)

**선택 이유**
- 익숙함과 안정성
- AWS 생태계 통합 (백업, 모니터링)
- 추후 Read Replica 확장 용이

**고려했던 대안**
- PostgreSQL: JSON 지원 우수하나, 현재 요구사항에서는 MySQL로 충분
- Aurora: 비용 대비 현재 규모에서는 과함

---

### 인프라: EC2 + Cloudflare

**현재 구성을 선택한 이유**
- 비용: 프리티어 + 무료 DNS
- 단순함: 관리 포인트 최소화
- 충분함: 현재 트래픽에서는 오버엔지니어링 불필요

**Cloudflare 프록시 OFF 선택**

초기에는 프록시 ON으로 SSL 관리 편의성을 얻으려 했으나, 한국에서 홍콩 PoP로 라우팅되어 ~500ms 지연 발생. 속도 우선으로 프록시 OFF + 자체 SSL 선택.

→ 상세 내용: [TROUBLESHOOTING.md](./TROUBLESHOOTING.md#1-api-응답-속도-지연-cloudflare-프록시)

---

## 확장 시나리오

### 트래픽 증가 시

```
현재: Client → EC2 (단일)

확장: Client → ALB → EC2 (Auto Scaling Group)
                         │
                         ▼
                    Redis (세션)
```

| 변경 사항 | 이유 |
|-----------|------|
| ALB 추가 | 로드 밸런싱, Health Check |
| Auto Scaling | 트래픽에 따른 자동 확장/축소 |
| Redis | 세션 공유 (다중 서버) |
| ACM | ALB에서 SSL 종료 (Certbot 관리 불필요) |

### DB 부하 증가 시

```
현재: EC2 → RDS (단일)

확장: EC2 → RDS Primary (쓰기)
        └→ RDS Read Replica (읽기)
```

- 읽기/쓰기 분리
- Spring의 @Transactional(readOnly = true)로 라우팅

### 글로벌 서비스 시

```
현재: Cloudflare (DNS Only)

확장: CloudFront 또는 Cloudflare Pro
      - 서울 PoP 보장
      - 정적 자원 캐싱
```

---

## SSL 구성

### 현재: Certbot + Nginx

```bash
# 인증서 발급
sudo certbot --nginx -d api.astarchia.com

# 자동 갱신 (systemd timer)
sudo certbot renew --dry-run
```

**장점**: 무료, 자동 갱신
**단점**: 서버에서 직접 관리 필요

### 확장 시: ALB + ACM

- ACM에서 무료 인증서 발급
- ALB에서 SSL 종료
- EC2는 HTTP만 처리 (내부 통신)

---

## 배포

### 현재: 수동 배포

```bash
ssh ec2-user@api.astarchia.com
cd ~/astarchia
git pull origin main
./gradlew bootJar
sudo systemctl restart astarchia
```

### 계획: GitHub Actions CI/CD

```yaml
# 예정
- main 브랜치 push 시 자동 배포
- 테스트 통과 후 배포
- 롤백 전략 수립
```
