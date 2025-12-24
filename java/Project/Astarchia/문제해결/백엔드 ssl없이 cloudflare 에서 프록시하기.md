

1. **astarchia.com** → 프록시 OFF (Vercel이 알아서 SSL 처리)
2. **api.astarchia.com** → 프록시 ON (Cloudflare SSL로 HTTPS 제공)



### API 서버 도메인에 등록후 프록시 켜기

![[Pasted image 20251224114844.png]]


### 요청흐름
```
사용자 브라우저
    │
    │ 1. https://astarchia.com 접속
    ▼
┌─────────────────┐
│   Vercel Edge   │  ← 프론트엔드 (React/Next.js 등)
│   (자체 SSL)    │
└─────────────────┘
    │
    │ 2. 프론트 코드에서 fetch('https://api.astarchia.com/...')
    ▼
┌─────────────────┐
│   Cloudflare    │  ← 프록시 ON ()
│   Edge Server   │
│   (SSL 종료)    │  ← 여기서 HTTPS → HTTP 변환
└─────────────────┘
    │
    │ 3. HTTP로 EC2에 전달 (내부 통신)
    ▼
┌─────────────────┐
│   AWS EC2       │  ← 백엔드 서버 (3.37.191.88)
│   (SSL 없음)    │     포트 80 또는 3000 등
└─────────────────┘
    │
    │ 4. 응답 (JSON 등)
    ▼
  (역순으로 돌아감)
```
## 프록시 OFF (회색 구름)

```
브라우저
    │
    │ 1. DNS 조회: "api.astarchia.com?"
    ▼
┌─────────────────────┐
│   Cloudflare DNS    │  → 실제 IP 반환: "3.37.191.88"
└─────────────────────┘
    │
    │ 2. 브라우저가 EC2로 직접 요청
    ▼
┌─────────────────────┐
│   EC2 백엔드        │
│   (3.37.191.88)     │
└─────────────────────┘
```

**Cloudflare는 DNS 역할만 하고 끝.**

---

## 프록시 ON (주황 구름)

```
브라우저
    │
    │ 1. DNS 조회: "api.astarchia.com?"
    ▼
┌─────────────────────┐
│   Cloudflare DNS    │  → Cloudflare IP 반환: "104.21.x.x"
└─────────────────────┘
    │
    │ 2. 브라우저가 Cloudflare로 요청
    ▼
┌─────────────────────┐
│   Cloudflare        │  ← SSL 처리, DDoS 방어, 캐싱
│   프록시 서버        │
└─────────────────────┘
    │
    │ 3. Cloudflare가 EC2로 대신 요청
    ▼
┌─────────────────────┐
│   EC2 백엔드        │
│   (3.37.191.88)     │
└─────────────────────┘
```

**Cloudflare가 중간에서 모든 트래픽을 처리.**


## 단계별 상세 설명

**1단계: 사용자 → Vercel**
- 브라우저가 `https://astarchia.com` 요청
- DNS에서 Vercel IP (`76.76.21.21`) 반환
- Vercel이 SSL 처리 + 프론트 파일 전송

**2단계: 브라우저 → Cloudflare**
- 프론트 코드가 `https://api.astarchia.com/users` 호출
- DNS 조회 → Cloudflare 프록시 IP 반환 (실제 EC2 IP 숨겨짐)
- **브라우저 ↔ Cloudflare 구간: HTTPS (암호화)**

**3단계: Cloudflare → EC2**
- Cloudflare가 SSL 복호화
- EC2로 **HTTP** 요청 전달
- **Cloudflare ↔ EC2 구간: HTTP (비암호화)**

**4단계: 응답**
- EC2 → Cloudflare → 브라우저 (역순)



```
프론트 (Vercel HTTPS) ─암호화─▶ Cloudflare(DNS) ──HTTP(암호화X)──▶ EC2:8080 (Docker App)
```
