## 개요
정부가 항공사 시스템에 **직접 접속하여 PNR 데이터를 조회**하는 방식.

## 2단계 처리 흐름
```
    [1단계] FlightCollectingScheduler
        ┌─────────────────────────────────────┐
        │ scheduleCollect (항공편 스케줄 수집)  │
        │         ↓                           │
        │ scheduleParse (스케줄 파싱)          │
        │         ↓                           │
        │ scheduleUpdate (DB 등록)            │
        │         ↓                           │
        │ pnrCollectingSchedules 테이블       │
        └─────────────────────────────────────┘
                          ↓
    [2단계] PnrCollectingScheduler
        ┌─────────────────────────────────────┐
        │ pnrCollect (PNR 데이터 수집)         │
        │         ↓                           │
        │ pnrParse (PNRGOV 생성)              │
        │         ↓                           │
        │ pnrSend (CIQ 전송)                  │
        └─────────────────────────────────────┘
```

## 주요 코드

| 파일 | 역할 |
|------|------|
| `flightCollectingScheduler.rb` | 항공편 스케줄 수집 |
| `pnrCollectingScheduler.rb` | PNR 데이터 수집 |

## FlightCollectingScheduler

### 클래스 구성
- `FlightCollectingJobUpdater`: 수집 Job 스케줄 생성
- `FlightCollectingScheduler`: 실제 스케줄 수집/파싱

### 핵심 메서드

| 메서드 | 역할 |
|--------|------|
| `scheduleCollect` | 항공사에서 운항 스케줄 수집 |
| `scheduleParse` | 스케줄 데이터 파싱 |
| `scheduleUpdate` | pnrCollectingSchedules에 등록 |
| `getScheduleInfos` | GMT 계산, CollectingTime 설정 |

## PnrCollectingScheduler

### 핵심 메서드

| 메서드 | 역할 |
|--------|------|
| `pnrCollect` | 항공사 시스템에서 PNR 수집 |
| `pnrParse` | PNRGOV 포맷 변환 |
| `pnrSend` | CIQ 전송 |

### 특이사항
- **OZ(아시아나)**: 단일 스레드로 처리
- 다른 항공사: 멀티 스레드

## 로그 예시
```
11:00:17 Collector started   (OZ9723/20251215/KIX/PUS)
11:00:18 Collector succeeded
11:00:22 Parser started
11:00:22 Parser succeeded  
11:00:24 Sender started
11:00:24 Sender succeeded
```

## ReportDeadline 계산
- **입항**: 도착 1시간 전 (운항 3시간 미만 시 30분 전)
- **출항**: 출발 후 3시간 이내