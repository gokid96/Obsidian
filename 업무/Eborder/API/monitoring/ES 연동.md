****Handler > Messages (행 펼침):**

- MySQL에 있는 메시지( iapi-status-updater 파싱된것만 확인가능) → 관련 ES 로그 보기
- 정상 처리된 메시지의 상세 처리 과정 확인

**Handler > Logs:**

- ES 전체 검색
- **iapi-status-updater 파싱 에러 메시지도 검색 가능**
- MySQL에 없는 메시지도 찾을 수 있음

---
### 시나리오 1: 정상 메시지 추적

```
Messages2 → 행 펼침 → 처리 로그 확인
```

### 시나리오 2: 파싱 에러 메시지 찾기

```
Handler > Logs → "edifactParser" 또는 "Error" 검색
→ 파싱 실패한 메시지 원본 확인 가능
```

## ## ES 로그 시스템 구조

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              원격 서버 (iapp-test)                               │
│                              192.168.100.13                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   /home/eborder/iapp/log/                                                       │
│   ├── airlineMessageHandler_v3_UA1.log                                          │
│   ├── airlineMessageHandler_v3_1N.log                                           │
│   ├── airlineMessageHandler_v3_XS.MM1.log                                       │
│   └── airlineMessageHandler_v3_XS.ZG1.log                                       │
│                                                                                  │
│                          ↓ 파일 읽기                                             │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                        Vector (Nomad Job UI)                             │   │
│   │                        Port: 3110                                        │   │
│   ├─────────────────────────────────────────────────────────────────────────┤   │
│   │  [sources.handler_logs]     파일 수집 (멀티라인)                         │   │
│   │           ↓                                                              │   │
│   │  [transforms.parse_handler_logs]  필드 파싱                              │   │
│   │    - service_code (UA1, XS.MM1 등)                                       │   │
│   │    - direction (received/sent)                                           │   │
│   │    - message_type (CUSRES, Reject2, PAXLST)                             │   │
│   │    - request_id                                                          │   │
│   │    - log_level (INFO/WARN/ERROR)                                        │   │
│   │    - log_time                                                            │   │
│   │    - message_size                                                        │   │
│   │           ↓                                                              │   │
│   │  [sinks.to_elasticsearch]   ES로 전송                                    │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ HTTP (Port 9200)
                                       ↓
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Docker 서버 (mercury-dev)                              │
│                              192.168.14.31                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │              Elasticsearch (Docker)                                      │   │
│   │              Port: 9200                                                  │   │
│   ├─────────────────────────────────────────────────────────────────────────┤   │
│   │  Index: handler-logs-2026.01.14                                         │   │
│   │                                                                          │   │
│   │  Fields:                                                                 │   │
│   │    - timestamp (ES 저장 시간)                                            │   │
│   │    - log_time (로그 원본 시간) ← 정렬 기준                               │   │
│   │    - service_code.keyword                                                │   │
│   │    - direction.keyword                                                   │   │
│   │    - message_type.keyword                                                │   │
│   │    - request_id.keyword                                                  │   │
│   │    - log_level.keyword                                                   │   │
│   │    - message (전문 검색)                                                 │   │
│   │    - host_name.keyword                                                   │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                       │                                          │
│                                       │                                          │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │              Backend (Spring Boot, Docker)                               │   │
│   │              Port: 3200                                                  │   │
│   ├─────────────────────────────────────────────────────────────────────────┤   │
│   │  HandlerLogService.java                                                  │   │
│   │    - searchLogs() → ES 쿼리                                              │   │
│   │    - getServices() → 서비스 목록 (Aggregation)                           │   │
│   │    - getMessageTypes() → 타입 목록 (Aggregation)                         │   │
│   │                                                                          │   │
│   │  HandlerController.java                                                  │   │
│   │    - POST /api/handler/logs/search                                       │   │
│   │    - GET  /api/handler/logs/services/{server}                            │   │
│   │    - GET  /api/handler/logs/message-types                                │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                       │                                          │
│                                       │                                          │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │              Frontend (Vue.js, Docker)                                   │   │
│   │              Port: 5100                                                  │   │
│   ├─────────────────────────────────────────────────────────────────────────┤   │
│   │  views/menu/handler/Logs.vue                                             │   │
│   │    - 필터: Server, Service, Direction, Type, Level, RequestID           │   │
│   │    - 날짜 범위 선택                                                      │   │
│   │    - Live Tail (5초 갱신)                                                │   │
│   │    - 로그 펼치기 → Original Log + Parsed JSON                            │   │
│   │    - RequestID 클릭 → 관련 로그 검색                                     │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 데이터 흐름

```
1. Handler가 로그 파일에 기록
   ↓
2. Vector가 파일 변경 감지 → 멀티라인으로 묶음 → 필드 파싱
   ↓
3. ES에 문서 저장 (인덱스: handler-logs-YYYY.MM.DD)
   ↓
4. Backend가 ES 쿼리 (log_time 기준 정렬)
   ↓
5. Frontend에서 테이블 표시
```

---

## 주요 컴포넌트

|컴포넌트|위치|역할|
|---|---|---|
|**Vector**|iapp-test (Nomad)|로그 수집 + 파싱 + ES 전송|
|**Elasticsearch**|mercury-dev (Docker)|로그 저장 + 검색|
|**Backend**|mercury-dev (Docker)|API 서버|
|**Frontend**|mercury-dev (Docker)|웹 UI|

---

## 설정 파일 위치

|설정|위치|
|---|---|
|Vector|Nomad UI → Jobs → vector-handler-test|
|ES|Docker Compose (`elasticsearch` 서비스)|
|Backend|`application.yml` → `elasticsearch.url`|
|Frontend|`Logs.vue` → `handlerLogAPI`|





## ES 자주 쓰는 명령어

### 인덱스 관련
```bash
# 전체 인덱스 목록
curl "http://192.168.14.31:9200/_cat/indices?v"

# 특정 패턴 인덱스만
curl "http://192.168.14.31:9200/_cat/indices/handler-logs-*?v"

# 인덱스 용량 순 정렬
curl "http://192.168.14.31:9200/_cat/indices?v&s=store.size:desc"
```

### ILM 관련
```bash
# ILM 정책 목록 (어떤 정책들이 있는지 파일 로테이션 기간 등)
curl "http://192.168.14.31:9200/_ilm/policy?pretty"

# 인덱스별 ILM 상태 (인덱스에 정책이 적용됐는지 + 몇일남았는지)
curl "http://192.168.14.31:9200/handler-logs-*/_ilm/explain?pretty"

```

### 클러스터 상태
```bash
# 클러스터 헬스
curl "http://192.168.14.31:9200/_cluster/health?pretty"

# 노드 상태
curl "http://192.168.14.31:9200/_cat/nodes?v"

# 디스크 사용량
curl "http://192.168.14.31:9200/_cat/allocation?v"
```

### 로그 검색
```bash
# 최근 로그 10건
curl "http://192.168.14.31:9200/handler-logs-*/_search?pretty&size=10"

# 키워드 검색
curl "http://192.168.14.31:9200/handler-logs-*/_search?pretty" \
-H "Content-Type: application/json" \
-d '{"query":{"match":{"message":"error"}}}'

# 특정 호스트 로그
curl "http://192.168.14.31:9200/handler-logs-*/_search?pretty" \
-H "Content-Type: application/json" \
-d '{"query":{"term":{"host_name":"iapp-test"}}}'
```

### 문서 수 확인
```bash
# 인덱스별 문서 수
curl "http://192.168.14.31:9200/_cat/count/handler-logs-*?v"
```

### 인덱스 삭제 (수동)
```bash
# 특정 인덱스 삭제
curl -X DELETE "http://192.168.14.31:9200/handler-logs-2026.01.14"
```