## VPN이란?

- 공용 네트워크에서 암호화된 사설 연결
- 마치 같은 네트워크에 있는 것처럼 통신

## VPN 유형

### Site-to-Site VPN
```
[본사 네트워크] ←── VPN 터널 ──→ [지사 네트워크]
(192.168.1.0/24)   (인터넷)     (192.168.2.0/24)
```

### Remote Access VPN
```
[재택 PC] ←── VPN 터널 ──→ [회사 VPN Gateway] → [내부망]
```

## 주요 VPN 프로토콜

|프로토콜|특징|
|---|---|
|IPSec|표준, L3 레벨, IKE로 키 교환|
|OpenVPN|오픈소스, SSL 기반, 유연함|
|WireGuard|최신, 간단, 빠름|
|L2TP/IPSec|L2TP + IPSec 조합|

## IPSec 구성 요소
```
IKE (Internet Key Exchange): 키 교환 프로토콜
├── Phase 1: SA 설정, 인증
└── Phase 2: IPSec SA 설정

ESP (Encapsulating Security Payload): 암호화, 인증
AH (Authentication Header): 인증만 (암호화 X)
```

## WireGuard 설정 예시
```ini
# /etc/wireguard/wg0.conf (서버)
[Interface]
PrivateKey = <서버 개인키>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <클라이언트 공개키>
AllowedIPs = 10.0.0.2/32
```

```bash
# 활성화
wg-quick up wg0

# 상태 확인
wg show
```

## VPN 트러블슈팅
```bash
# 1. VPN 연결 상태 확인
wg show
# 또는
ipsec status

# 2. 라우팅 테이블 확인
ip route

# 3. 방화벽 규칙 확인
iptables -L -n -v | grep vpn

# 4. MTU 문제 확인
ping -M do -s 1400 [대상]
```

## 자주 발생하는 문제

|증상|원인|해결|
|---|---|---|
|VPN 연결됐는데 통신 안됨|라우팅 누락|ip route 확인, 추가|
|특정 사이트만 안됨|MTU 문제|MTU 낮춤 (1400)|
|연결 자체 안됨|방화벽 차단|UDP 500, 4500 열기|