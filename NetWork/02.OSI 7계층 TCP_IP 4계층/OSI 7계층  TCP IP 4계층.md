## 계층 비교

```
OSI 7계층              TCP/IP 4계층           데이터 단위      주요 장비
─────────────────────────────────────────────────────────────────────
7. 응용 (Application)  ┐
8. 표현 (Presentation) ├─ 응용 (Application)    Data(메시지)    -
9. 세션 (Session)      ┘
10. 전송 (Transport)    ─  전송 (Transport)      Segment        방화벽(L4)
11. 네트워크 (Network)   ─  인터넷 (Internet)     Packet         라우터
12. 데이터링크 (Data Link) ┬ 네트워크 접근        Frame          스위치
13. 물리 (Physical)       ┘ (Network Access)    Bit            허브, 케이블
```

---

## OSI 각 계층 상세

### L1. 물리 계층 (Physical)

- **역할**: 비트를 전기/광 신호로 변환
- **장비**: 허브, 리피터, 케이블
- **단위**: Bit
- **예시**: 이더넷 케이블, 광케이블

### L2. 데이터링크 계층 (Data Link)

- **역할**: 같은 네트워크 내 노드 간 통신, 오류 검출
- **장비**: 스위치, 브릿지
- **단위**: Frame
- **주소**: MAC 주소
- **프로토콜**: Ethernet, ARP

### L3. 네트워크 계층 (Network)

- **역할**: 다른 네트워크 간 경로 설정 (라우팅)
- **장비**: 라우터, L3 스위치
- **단위**: Packet
- **주소**: IP 주소
- **프로토콜**: IP, ICMP, OSPF, BGP

### L4. 전송 계층 (Transport)

- **역할**: 종단 간 신뢰성 있는 데이터 전송
- **장비**: 방화벽 (L4)
- **단위**: Segment (TCP) / Datagram (UDP)
- **주소**: Port 번호
- **프로토콜**: TCP, UDP

### L5. 세션 계층 (Session)

- **역할**: 연결 설정, 유지, 종료 관리
- **예시**: 로그인 세션, RPC

### L6. 표현 계층 (Presentation)

- **역할**: 데이터 형식 변환, 암호화, 압축
- **예시**: SSL/TLS, JPEG, ASCII

### L7. 응용 계층 (Application)

- **역할**: 사용자와 직접 상호작용
- **프로토콜**: HTTP, FTP, SMTP, DNS, SSH
