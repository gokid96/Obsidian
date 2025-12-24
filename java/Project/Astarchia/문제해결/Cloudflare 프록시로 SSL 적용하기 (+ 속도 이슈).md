#### EC2 SSL 없이 Cloudflare 프록시로  SSL 적용하기

## 구성
- **astarchia.com** → 프록시 OFF (Vercel 자체 SSL)
- **api.astarchia.com** → 프록시 ON (Cloudflare SSL)

![[Pasted image 20251224114844.png]]

---

## 요청 흐름

```
브라우저
    │
    │ https://astarchia.com
    ▼
┌─────────────────┐
│   Vercel        │  ← 프론트 (자체 SSL)
└─────────────────┘
    │
    │ fetch('https://api.astarchia.com/...')
    ▼
┌─────────────────┐
│   Cloudflare    │  ← 프록시 ON, SSL 종료
└─────────────────┘
    │
    │ HTTP (비암호화)
    ▼
┌─────────────────┐
│   EC2           │  ← 백엔드 (SSL 없음)
└─────────────────┘
```

**핵심:** 브라우저 ↔ Cloudflare는 HTTPS, Cloudflare ↔ EC2는 HTTP

---

## 프록시 ON vs OFF

|        | 프록시 OFF (회색)   | 프록시 ON (주황)             |
| ------ | -------------- | ----------------------- |
| DNS 응답 | EC2 실제 IP      | Cloudflare IP           |
| 트래픽 경로 | 브라우저 → EC2 직접  | 브라우저 → Cloudflare → EC2 |
| SSL    | EC2에서 직접 처리 필요 | Cloudflare가 처리          |
| 부가 기능  | 없음             | DDoS 방어, 캐싱 등           |


---

## 속도 이슈

### 증상

클라이언트 대기시간 521ms로 느림

![[Pasted image 20251224153232.png]]

### 원인 curl -I https://api.astarchia.com
```
HTTP/1.1 401 Unauthorized
Date: Wed, 24 Dec 2025 06:30:59 GMT
Content-Type: application/json;charset=UTF-8
Content-Length: 172
Connection: keep-alive
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Set-Cookie: JSESSIONID=F1088C5029124DBEC530CBF7C230C11D; Path=/; HttpOnly
X-Content-Type-Options: nosniff
X-XSS-Protection: 0
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Nel: {"report_to":"cf-nel","success_fraction":0.0,"max_age":604800}
cf-cache-status: DYNAMIC
Report-To: {"group":"cf-nel","max_age":604800,"endpoints":[{"url":"https://a.nel.cloudflare.com/report/v4?s=UT6T80U0HEveb4dv5bZwR7P8W8aOdl23yL1QPSf9lDZoM%2F%2BzgGjzzblYSbu8CyFuEDwRXpNmKqOF9xjQfMegJJAqsUXcYGJg%2BUz6128EyA4c"}]}
Server: cloudflare
CF-RAY: 9b2e0d9b4d6ec8a7-HKG
alt-svc: h3=":443"; ma=86400
```

```
CF-RAY: 9b2e0d9b4d6ec8a7-HKG ← 홍콩 서버로 라우팅됨
```

```
클라이언트(한국) → Cloudflare(홍콩) → EC2(서울) → Cloudflare(홍콩) → 클라이언트
```

서울 리전인데 홍콩을 경유해서 왕복 지연 발생

### 해결 방안

1. **프록시 OFF + EC2 자체 SSL** → 빠름, 단 IP 노출
2. **Argo Smart Routing (유료)** → 최적 경로 보장
3. **AWS CloudFront** → 서울 엣지 확실
4. **현 상태 유지** → 500ms 감수