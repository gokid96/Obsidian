

## 시스템 개요

```
항공사 ←→ [AirlineMessageHandler] ←→ Redis ←→ [ImmigrationServer] ←→ 법무부(CIQ)
```

## 지원 프로토콜 (3가지)

|프로토콜|설명|
|---|---|
|**MATIP**|TCP 소켓 + MATIP 헤더 (직접 연결)|
|**MQ**|IBM MQ 등 메시지 큐 방식|
|**HTH (Host-to-Host)**|Layer 4/5/6 헤더가 있는 방식|

조합: `MATIP/HTH`, `MQ/HTH`, `MQ/NoHTH`

---

## 주요 클래스

|클래스|역할|
|---|---|
|`AirlineServerWorker`|항공사로부터 메시지 수신 (MATIP 패킷 파싱)|
|`AirlineMessageDispatcher`|메시지 타입 분류 및 검증 (PAXLST, USM Reply 등)|
|`AirlineServerSender`|항공사로 응답 전송 (CUSRES, Reject, USM)|
|`AirlineMessageCleaner`|Timeout 처리 (RequestTimeout 초과 시 Reject2)|
|`HandlerStatusUpdater`|Redis에 상태/카운트 업데이트|
|`ConfigurationReload`|설정 변경 시 재시작 없이 적용|

---

## 메시지 타입

### 수신 (항공사 → E-border)

|타입|QRI|설명|
|---|---|---|
|**PAXLST**|H|승객 정보 (API 메시지)|
|**USM Reply**|D|USM에 대한 응답|
|**Query Rejected**|J|쿼리 거부|
|**Reply Rejected**|F|응답 거부|

### 송신 (E-border → 항공사)

|타입|설명|
|---|---|
|**CUSRES**|PAXLST에 대한 정상 응답|
|**Reject1**|즉시 거부 (1D: 사이즈 초과, 1J: 항공사 미등록, 1K: 주소 불일치)|
|**Reject2**|Timeout 거부 (1F)|
|**USM**|Unsolicited Message (정부 → 항공사 요청)|

---

## MATIP 프로토콜 상세

ruby

````ruby
MATIPHeaderSize = 4  # 헤더 크기

# 메시지 타입
MATIPSessionOpen = 254        # 세션 시작
MATIPSessionOpenConfirm = 253 # 세션 확인
MATIPSessionClose = 252       # 세션 종료
MATIPData = 0                 # 데이터
MATIPSSQ = 251                # Status Query
MATIPSSR = 250                # Status Response
```

---

## 처리 흐름
```
[PAXLST 수신 흐름]

항공사 → MATIP 패킷 수신 (AirlineServerWorker)
       ↓
HTH 헤더 파싱 (parseHeader)
       ↓
메시지 타입 분류 (AirlineMessageDispatcher)
       ↓
검증 (항공사 코드, 사이즈, Layer5 주소)
       ↓
Redis에 저장 → ImmigrationServer로 전달
       ↓
응답 수신 → 항공사로 전송 (AirlineServerSender)
````

---

## Redis 키 구조

|키|용도|
|---|---|
|`AirlineRequestMessages`|메시지 정보 저장 (Hash)|
|`AirlineRequestIDs:Request:{ServiceName}`|처리 대기 큐|
|`AirlineRequestIDs:Response:{ServiceName}`|응답 대기 큐|
|`AirlineRequestIDs:Request:{ServiceName}:Pending`|Timeout 체크용|
|`PaxlstResponseSentCheck`|중복 응답 방지|
|`AirlineMessageHandlerStatus`|상태/카운트|

---

## AutoReply 기능

특정 항공사/편명에 대해 **법무부 거치지 않고 자동 응답** 생성:

ruby

````ruby
$AutoReplyEnabled = true
$AutoCarrierCodes = ["OZ", "KE"]  # 자동 응답 대상 항공사
$AutoFlightNumbers = ["OZ123"]    # 자동 응답 대상 편명
```

---

## 전체 E-border 시스템 구조 (업데이트)
```
┌─────────────────────────────────────────────────────────────────────┐
│                        E-border 시스템                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  [Push 방식]                                                        │
│  항공사 → MATIP → pushPnrManager → PNRGOV 생성 → CIQ              │
│                                                                     │
│  [Pull 방식]                                                        │
│  FlightCollectingScheduler → PnrCollectingScheduler → CIQ          │
│                                                                     │
│  [실시간 PAXLST]                                                    │
│  항공사 ←→ AirlineMessageHandler ←→ Redis ←→ ImmigrationServer    │
│            (MATIP/MQ/HTH)                      ↓                   │
│                                              CIQ(법무부)            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
````

이제 **Push PNR**, **Pull PNR**, **실시간 PAXLST(MQ/MATIP)**