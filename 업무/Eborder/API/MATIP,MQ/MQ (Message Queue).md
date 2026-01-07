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