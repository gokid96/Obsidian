

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