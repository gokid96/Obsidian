
## 문제 상황

Cloudflare 프록시를 활성화한 상태에서 API 응답 속도가 비정상적으로 느림

| 항목    | 값                         |
| ----- | ------------------------- |
| 측정 구간 | 클라이언트 → api.astarchia.com |
| 응답 시간 | **521ms**                 |
| 기대 수준 | ~50ms                     |

## 원인 분석

### 1. CF-RAY 헤더 확인

````bash
curl -I https://api.astarchia.com
# CF-RAY: 9b2e0d9b4d6ec8a7-HKG ← 홍콩 서버로 라우팅
```

### 2. 실제 트래픽 경로

Cloudflare 무료 플랜에서 한국 트래픽이 홍콩 PoP로 라우팅됨
```
클라이언트(한국) → Cloudflare(홍콩) → EC2(서울) → Cloudflare(홍콩) → 클라이언트
````

불필요한 국제 왕복으로 인해 ~450ms 추가 지연 발생

## 대안 비교

|방안|속도|보안|비용|비고|
|---|---|---|---|---|
|프록시 OFF + EC2 자체 SSL|✅ ~50ms|❌ IP 노출|무료|Certbot 관리 필요|
|Argo Smart Routing|✅ 빠름|✅ IP 숨김|월 $5+|최적 경로 보장|
|AWS CloudFront|✅ 서울 엣지|✅|종량제|설정 복잡|
|현 상태 유지|❌ ~500ms|✅ IP 숨김|무료|DDoS 방어|

## 의사결정

**선택: 프록시 OFF + EC2 자체 SSL**

현재 서비스 규모(소규모, 트래픽 적음)를 고려했을 때 보안보다 **사용자 경험(속도)**을 우선함

향후 트래픽 증가 또는 보안 이슈 발생 시 Argo Smart Routing 도입 검토

## 적용

### 1. Cloudflare 프록시 OFF

DNS 설정에서 api.astarchia.com 프록시 비활성화 (주황 → 회색)

### 2. EC2에 nginx ssl 자동갱신

```bash
# Certbot으로 Let's Encrypt 인증서 발급
sudo certbot --nginx -d api.astarchia.com

# 자동 갱신 확인
sudo certbot renew --dry-run
```

자동 갱신은 systemd timer로 처리

## 결과

|항목|Before|After|개선율|
|---|---|---|---|
|API 응답 시간|521ms|**70ms**|**86% 개선**|

## 배운 점

Cloudflare 무료 플랜은 지역에 따라 가까운 PoP가 아닌 원거리 서버로 라우팅될 수 있음. 한국에서는 홍콩/싱가포르로 우회되는 경우가 많아, **속도가 중요한 API 서버**에는 주의가 필요함.