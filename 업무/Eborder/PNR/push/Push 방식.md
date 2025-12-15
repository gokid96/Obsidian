## 개요
항공사가 정부에 **능동적으로 PNR 데이터를 전송**하는 방식.

## 처리 흐름
```
항공사
   │
   ▼ MATIP
┌─────────────────┐
│ .RCV 파일 수신   │
└────────┬────────┘
         ▼
┌─────────────────┐
│ EDIFACT 파싱    │  ← getEDIFACTMessageInfo()
│ (UNA/UNB/UNH)   │
└────────┬────────┘
         ▼
┌─────────────────┐
│ PNRGOV 변환     │  ← CUSMANToPNRGOV / PNRGOVToPNRGOV
└────────┬────────┘
         ▼
┌─────────────────┐
│ DB 저장         │  ← pushPnrStatus
└────────┬────────┘
         ▼
┌─────────────────┐
│ CIQ 전송        │  ← pnrSend()
└─────────────────┘
```

## 주요 코드
- **파일**: `pushPnrManager.rb`
- **클래스**: `PushPnrManager`

## 핵심 메서드

| 메서드 | 역할 |
|--------|------|
| `pnrParse` | .RCV 파일 파싱 |
| `getEDIFACTMessageInfo` | EDIFACT 세그먼트 분석 |
| `pushInfoUpdate` | DB 상태 업데이트 |
| `seriesInfoUpdate` | 분할 메시지 처리 |
| `pnrPartialMessage` | 미완성 메시지 강제 처리 |

## Series 메시지 처리
대용량 데이터는 여러 메시지로 분할 전송됨.
```
UNH+1+PNRGOV:11:1:IA+08064348195007+01:C'  ← Continue
UNH+1+PNRGOV:11:1:IA+08064348195007+02'    ← 중간
UNH+1+PNRGOV:11:1:IA+08064348195007+03:F'  ← Final
```

- `govInfo_*.dat` 파일에 임시 저장
- Final(F) 메시지 수신 시 합쳐서 처리
- 5분 이상 미완성 시 강제 처리

## ResultCode

| 코드 | 의미 |
|------|------|
| 0 | 성공 |
| 1 | 파싱 에러 |
| 2 | 승객 없음 |
| 3 | 일부 PNR 스킵 |
| 4 | Series 미완성 |
| 5 | Series 미완성 + 스킵 |
| 6 | 미허용 항공사 코드 |