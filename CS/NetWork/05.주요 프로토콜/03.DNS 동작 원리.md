## DNS (Domain Name System)

- **포트**: 53 (UDP 주로, TCP도 사용)
- **역할**: 도메인 → IP 주소 변환

## DNS 레코드 타입

| 타입    | 의미                   |
| ----- | -------------------- |
| A     | IPv4 주소              |
| AAAA  | IPv6 주소              |
| CNAME | 별칭 (다른 도메인으로 연결)     |
| MX    | 메일 서버                |
| TXT   | 텍스트 정보 (SPF, DKIM 등) |
| NS    | 네임서버                 |
| PTR   | 역방향 조회 (IP → 도메인)    |

## Recursive vs Iterative 쿼리
```
Recursive (재귀적):
클라이언트 → Local DNS → 최종 답을 받아서 반환
(Local DNS가 대신 모든 조회 수행)

Iterative (반복적):
Local DNS → Root NS: ".com은 이쪽으로"
Local DNS → .com NS: "google.com은 이쪽으로"
Local DNS → google.com NS: "142.250.x.x"
(Local DNS가 직접 단계별 조회)
```

## DNS 조회 과정
```
브라우저 캐시 → OS 캐시 → /etc/hosts → Local DNS → Root → TLD → 권한 NS
```

## DNS 명령어
```bash
# 기본 조회
nslookup google.com
dig google.com

# 특정 타입 조회
dig google.com MX
dig google.com TXT

# 전체 조회 과정 추적
dig +trace google.com

# 역방향 조회
dig -x 8.8.8.8
```

## DNS TTL
- Time To Live: 캐시 유지 시간
- 짧으면: 변경 빠르게 반영, DNS 부하 증가
- 길면: 캐시 효율 좋음, 변경 반영 느림