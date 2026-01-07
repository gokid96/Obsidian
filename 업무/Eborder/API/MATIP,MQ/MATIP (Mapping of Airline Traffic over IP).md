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

