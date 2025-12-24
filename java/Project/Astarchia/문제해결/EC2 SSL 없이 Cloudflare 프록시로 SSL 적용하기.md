## 1. 이유

- EC2에서 SSL 인증서 발급/갱신 관리 안 해도 됨
- Cloudflare가 무료로 SSL 처리
- 백엔드 IP 숨김 + DDoS 방어 등 보안 기능

> [!참고: 일반적인 SSL 구성 방법]
> 소규모 서비스 (보편적)
> - Certbot + Nginx 조합
> - Let's Encrypt 무료 인증서 자동 갱신
> - 설정 간단하고 자료 많음
> 
> 대규모 서비스
> - AWS: ALB/ELB + ACM (AWS Certificate Manager)
> - 로드밸런서에서 SSL 종료, 인증서 자동 갱신
> - Cloudflare Enterprise + Full (Strict) 모드
> - 전 구간 암호화 + 성능 최적화
> - Kubernetes: Ingress + cert-manager
> - 자동 인증서 발급/갱신
> 

---

## 2. 구성 및 연결 과정

### 구성

- **astarchia.com** → 프록시 OFF (Vercel 자체 SSL)
- **api.astarchia.com** → 프록시 ON (Cloudflare SSL)

![[Pasted image 20251224154912.png]]

### 요청 흐름

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
    │ HTTP (비암호화) ← 백본 경유, 탈취 가능성 적음
    ▼
┌─────────────────┐
│   EC2           │  ← 백엔드 (SSL 없음)
└─────────────────┘
```

**핵심:** 브라우저 ↔ Cloudflare는 HTTPS, Cloudflare ↔ EC2는 HTTP

### 프록시 ON vs OFF

|        | 프록시 OFF (회색)   | 프록시 ON (주황)             |
| ------ | -------------- | ----------------------- |
| DNS 응답 | EC2 실제 IP      | Cloudflare IP           |
| 트래픽 경로 | 브라우저 → EC2 직접  | 브라우저 → Cloudflare → EC2 |
| SSL    | EC2에서 직접 처리 필요 | Cloudflare가 처리          |
| 부가 기능  | 없음             | DDoS 방어, 캐싱 등           |

---

## 3. 문제: 속도 지연

### 증상

클라이언트 대기시간 521ms

![[Pasted image 20251224153232.png]]

### 원인 분석
```bash
curl -I https://api.astarchia.com
```

```
CF-RAY: 9b2e0d9b4d6ec8a7-HKG ← 홍콩 서버로 라우팅됨
```

Cloudflare 무료 플랜에서 한국 → 홍콩으로 라우팅되어 불필요한 왕복 발생
```
클라이언트(한국) → Cloudflare(홍콩) → EC2(서울) → Cloudflare(홍콩) → 클라이언트
````

---

## 4. 해결 방안

| 방안                   | 장점                           | 단점                           |
| -------------------- | ---------------------------- | ---------------------------- |
| 프록시 OFF + EC2 자체 SSL | 빠름, 무료                       | IP 노출, DDoS 방어 없음, SSL 직접 관리 |
| Argo Smart Routing   | 최적 경로 보장, 프록시 유지             | 유료 (월 $5 + 트래픽)              |
| AWS CloudFront       | 서울 엣지 확실, AWS 통합             | 설정 복잡, 비용 발생                 |
| 현 상태 유지              | 무료, 설정 변경 없음, IP 숨김, DDoS 방어 | 500ms 지연 감수                  |


프록시 OFF + EC2자체 SSL 설정시 응답 속도 확인

![[Pasted image 20251224162727.png]]

## 5. 트레이드오프

||프록시 OFF + EC2 SSL|프록시 ON (현재)|
|---|---|---|
|**속도**|✅ 빠름 (~50ms)|❌ 느림 (~500ms)|
|**보안**|❌ IP 노출, DDoS 취약|✅ IP 숨김, DDoS 방어|
|**관리**|❌ SSL 직접 관리 (Certbot)|✅ Cloudflare가 처리|
|**비용**|✅ 무료|✅ 무료|

**결론**: 소규모 개인 프로젝트면 속도 우선으로 **프록시 OFF**, 서비스가 커지거나 공격 우려 있으면 **프록시 ON 유지** 또는 **Argo 고려**