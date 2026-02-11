# Distributed Tracing 엣지케이스

## 1. 비동기 이벤트에서 trace context 끊김

| 항목 | 내용 |
|------|------|
| **문제** | Kafka 등 메시지 브로커를 거치면 trace context가 자연스럽게 전파되지 않음 |
| **왜 발생** | HTTP 헤더 기반 trace 전파가 메시지 브로커에서는 자동으로 이뤄지지 않음 |
| **해결** | 메시지 헤더에 traceId/spanId 명시적 삽입, OpenTelemetry 같은 표준 사용 |
| **트레이드오프** | 모든 producer/consumer에 계측 코드 필요 → 개발 부담 증가, 메시지 크기 미세 증가 |
