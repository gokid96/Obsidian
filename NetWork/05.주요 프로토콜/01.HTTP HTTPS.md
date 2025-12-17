
## HTTP (HyperText Transfer Protocol)

- **포트**: 80
- **특징**: 비암호화, Stateless

## 버전 변화

|버전|특징|
|---|---|
|HTTP/1.0|요청마다 연결 생성/종료|
|HTTP/1.1|Keep-Alive (연결 재사용), 파이프라이닝|
|HTTP/2|멀티플렉싱, 헤더 압축, 서버 푸시|
|HTTP/3|QUIC(UDP 기반), 더 빠른 연결|

## HTTPS

- **포트**: 443
- **특징**: HTTP + TLS/SSL 암호화
- **보안 제공**: 기밀성, 무결성, 인증

## TLS Handshake (간략)

```
Client                              Server
   │                                   │
   │──── Client Hello ────────────────▶│  (지원 암호화 목록)
   │◀─── Server Hello + 인증서 ────────│  (선택된 암호화 + 인증서)
   │──── 키 교환 ─────────────────────▶│  (Pre-master secret)
   │◀─── Finished ─────────────────────│
   │──── Finished ────────────────────▶│
   │                                   │
   │        암호화 통신 시작            │
```

## HTTP 메서드

|메서드|용도|멱등성|안전|
|---|---|---|---|
|GET|조회|O|O|
|POST|생성|X|X|
|PUT|전체 수정|O|X|
|PATCH|부분 수정|X|X|
|DELETE|삭제|O|X|

## HTTP 상태 코드

|범위|의미|예시|
|---|---|---|
|1xx|정보|100 Continue|
|2xx|성공|200 OK, 201 Created|
|3xx|리다이렉션|301 Moved, 304 Not Modified|
|4xx|클라이언트 오류|400 Bad Request, 404 Not Found|
|5xx|서버 오류|500 Internal Error, 502 Bad Gateway|