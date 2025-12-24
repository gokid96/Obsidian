# Cloudflare 프록시 설정 가이드

## 1. 프록시 ON을 사용하는 이유

- EC2에서 SSL 인증서 발급/갱신 관리가 필요 없음
- Cloudflare가 무료로 SSL 처리
- 백엔드 IP 숨김 및 DDoS 방어

> **참고: 일반적인 SSL 구성 방법**
> 
> **소규모 서비스**
> 
> - Certbot + Nginx 조합으로 Let's Encrypt 무료 인증서 자동 갱신
> 
> **대규모 서비스**
> 
> - AWS: ALB/ELB + ACM으로 로드밸런서에서 SSL 종료
> - Cloudflare Enterprise + Full (Strict) 모드로 전 구간 암호화
> - Kubernetes: Ingress + cert-manager로 자동 인증서 관리

---

## 2. 구성 및 요청 흐름

### 도메인 구성

|도메인|프록시|SSL 처리|
|---|---|---|
|astarchia.com|OFF|Vercel 자체 SSL|
|api.astarchia.com|ON|Cloudflare SSL|

### 요청 흐름

```
브라우저
    │ HTTPS
    ▼
Vercel (프론트, 자체 SSL)
    │ fetch('https://api.astarchia.com/...')
    ▼
Cloudflare (프록시 ON, SSL 종료)
    │ HTTP (백본 경유)
    ▼
EC2 (백엔드, SSL 없음)
```

### 프록시 ON vs OFF 비교

|        | 프록시 OFF       | 프록시 ON                  |
| ------ | ------------- | ----------------------- |
| DNS 응답 | EC2 실제 IP     | Cloudflare IP           |
| 트래픽 경로 | 브라우저 → EC2 직접 | 브라우저 → Cloudflare → EC2 |
| SSL    | EC2에서 직접 처리   | Cloudflare가 처리          |
| 부가 기능  | 없음            | DDoS 방어, 캐싱             |

---

## 3. 문제: 속도 지연

### 증상
클라이언트 대기시간 **521ms**

### 원인

````bash
curl -I https://api.astarchia.com
# CF-RAY: 9b2e0d9b4d6ec8a7-HKG ← 홍콩 서버로 라우팅
```

Cloudflare 무료 플랜에서 한국 트래픽이 홍콩으로 라우팅되어 불필요한 왕복 발생:
```
클라이언트(한국) → Cloudflare(홍콩) → EC2(서울) → Cloudflare(홍콩) → 클라이언트
````

---

## 4. 해결 방안

|방안|장점|단점|
|---|---|---|
|프록시 OFF + EC2 자체 SSL|빠름, 무료|IP 노출, DDoS 방어 없음|
|Argo Smart Routing|최적 경로, 프록시 유지|유료 (월 $5+)|
|AWS CloudFront|서울 엣지 보장|설정 복잡, 비용 발생|
|현 상태 유지|무료, IP 숨김, DDoS 방어|~500ms 지연|

프록시 OFF 설정 시 응답 속도: **~50ms**

---

## 5. 트레이드오프 요약

|     | 프록시 OFF + EC2 SSL | 프록시 ON           |
| --- | ----------------- | ---------------- |
| 속도  | ✅ ~50ms           | ❌ ~500ms         |
| 보안  | ❌ IP 노출           | ✅ IP 숨김, DDoS 방어 |
| 관리  | ❌ SSL 직접 관리       | ✅ 자동             |
| 비용  | ✅ 무료              | ✅ 무료             |
|     |                   |                  |

---

## 결론

- **소규모 프로젝트**: 속도 우선으로 **프록시 OFF** 권장
- **서비스 확장 또는 보안 우려 시**: **프록시 ON 유지** + Argo Smart Routing 고려