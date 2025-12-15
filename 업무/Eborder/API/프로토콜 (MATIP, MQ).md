## MATIP (Mapping of Airline Traffic over IP)

### 개요
항공 업계 Type B 메시지를 TCP/IP로 전송하기 위한 프로토콜.

### 패킷 구조
```
┌─────────┬─────────┬─────────┬─────────────┐
│ Version │  Type   │ Length  │   Payload   │
│ (1byte) │ (1byte) │ (2byte) │  (가변)     │
└─────────┴─────────┴─────────┴─────────────┘
```

### 메시지 타입

| 코드 | 타입 | 설명 |
|------|------|------|
| 254 | SessionOpen | 세션 시작 요청 |
| 253 | SessionOpenConfirm | 세션 시작 확인 |
| 252 | SessionClose | 세션 종료 |
| 0 | Data | 데이터 전송 |
| 251 | SSQ | Status Query |
| 250 | SSR | Status Response |

### 세션 흐름
```
항공사                    E-border
   │                        │
   │──── SessionOpen ──────▶│
   │◀── SessionOpenConfirm ─│
   │                        │
   │◀────── SSQ ────────────│ (주기적)
   │─────── SSR ───────────▶│
   │                        │
   │──── Data (PAXLST) ────▶│
   │◀─── Data (CUSRES) ─────│
   │                        │
   │──── SessionClose ─────▶│
```

## MQ (Message Queue)

### 개요
IBM MQ 등 메시지 큐를 통한 비동기 메시지 전송.

### 특징
- 비동기 처리
- 메시지 영속성
- HTH 헤더 유/무 선택 가능

### MQ 메시지 구조
```json
{
  "message": {
    "messageId": "...",
    "correlationId": "...",
    "messageBody": "PAXLST 메시지 내용"
  },
  "amrEntryUnixTime": 1702627200000,
  "amrLastUnixTime": 1702627200100,
  "amrSentUnixTime": 1702627200200
}
```

## HTH (Host-to-Host) 헤더

### 구조
```
Layer4\r
Layer5\r
Layer6\r
Payload
```

### Layer5 상세
```
V{QRI}/E1{SourceAddress}/I1{DestinationAddress}/P{TransactionID}

예: VHDG.WA/E11ASEL01/I1KRSEL01/P123456
```

### QRI 코드

| Octet1 | 의미 |
|--------|------|
| H | Query (PAXLST) |
| D | Reply (CUSRES, USM Reply) |
| J | Query Rejected |
| F | Reply Rejected |