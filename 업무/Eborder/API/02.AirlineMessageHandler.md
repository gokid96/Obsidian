## 개요
항공사와의 실시간 메시지 통신을 담당하는 핵심 모듈.

## 클래스 구조

| 클래스 | 역할 |
|--------|------|
| `AirlineServerWorker` | 메시지 수신, MATIP 패킷 파싱 |
| `AirlineMessageDispatcher` | 메시지 타입 분류 및 검증 |
| `AirlineServerSender` | 응답 전송 (CUSRES, Reject, USM) |
| `AirlineMessageCleaner` | Timeout 처리 |
| `HandlerStatusUpdater` | Redis 상태 업데이트 |
| `ConfigurationReload` | 설정 핫 리로드 |

## 처리 흐름
```
[수신]
항공사 → MATIP 패킷 수신 (AirlineServerWorker)
       ↓
HTH 헤더 파싱 (parseHeader)
       ↓
메시지 타입 분류 (AirlineMessageDispatcher)
  ├─ PAXLST (QRI: H) → 검증 → Redis 저장
  ├─ USM Reply (QRI: D) → USM 매칭
  ├─ Query Rejected (QRI: J) → 삭제
  └─ Reply Rejected (QRI: F) → 삭제
       ↓
ImmigrationServer로 전달

[송신]
ImmigrationServer → Redis → AirlineServerSender
       ↓
HTH 헤더 추가
       ↓
MATIP 패킷으로 전송 → 항공사
```

## Redis 키 구조

| 키 | 용도 |
|----|------|
| `AirlineRequestMessages` | 메시지 정보 (Hash) |
| `AirlineRequestIDs:Request:{ServiceName}` | 처리 대기 큐 |
| `AirlineRequestIDs:Response:{ServiceName}` | 응답 대기 큐 |
| `AirlineRequestIDs:Request:{ServiceName}:Pending` | Timeout 체크용 |
| `PaxlstResponseSentCheck` | 중복 응답 방지 |
| `AirlineMessageHandlerStatus` | 상태/카운트 |

## 검증 항목

| 검증 | WhyCode | 설명 |
|------|---------|------|
| Layer5 주소 | 1K | 소스/목적지 주소 불일치 |
| 항공사 코드 | 1J | 미등록 항공사 |
| 메시지 크기 | 1D | 최대 크기 초과 |
| Timeout | 1F | 응답 시간 초과 |

## AutoReply 기능
특정 항공사/편명에 대해 법무부 거치지 않고 자동 응답:
```ruby
$AutoReplyEnabled = true
$AutoCarrierCodes = ["OZ", "KE"]
$AutoFlightNumbers = ["OZ123"]
```