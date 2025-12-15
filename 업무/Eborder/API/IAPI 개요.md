
## IAPI란?
**Interactive Advance Passenger Information** - 실시간 사전 승객 정보 시스템

체크인 시점에 승객의 여권/비자 정보를 실시간으로 정부에 전송하고 응답을 받는 방식.

## PNR과의 차이

| 구분 | PNR | IAPI |
|------|-----|------|
| 데이터 | 예약 정보 | 신원 정보 (여권/비자) |
| 시점 | 예약~출발 전 | 체크인 시점 |
| 방식 | 일괄 전송 | 실시간 요청/응답 |
| 메시지 | PNRGOV | PAXLST/CUSRES |

## 시스템 구조
```
항공사
   │
   ▼ MATIP / MQ
┌─────────────────────────────────────┐
│      AirlineMessageHandler          │
│  ┌─────────────────────────────┐    │
│  │ AirlineServerWorker (수신)   │    │
│  │ AirlineMessageDispatcher    │    │
│  │ AirlineServerSender (송신)   │    │
│  │ AirlineMessageCleaner       │    │
│  └─────────────────────────────┘    │
└──────────────┬──────────────────────┘
               │ Redis
               ▼
┌─────────────────────────────────────┐
│       ImmigrationServer             │
└──────────────┬──────────────────────┘
               │
               ▼
           CIQ (법무부)
```

## 지원 프로토콜

| 프로토콜 | 설명 |
|----------|------|
| MATIP | TCP 소켓 직접 연결 |
| MQ | IBM MQ 메시지 큐 |
| HTH | Host-to-Host 헤더 방식 |

조합: `MATIP/HTH`, `MQ/HTH`, `MQ/NoHTH`

## 관련 문서
- [[AirlineMessageHandler]]
- [[프로토콜 (MATIP, MQ)]]
- [[메시지 타입]]