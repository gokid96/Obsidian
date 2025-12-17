### 필요한 경우
- 로그인 세션이 서버 메모리에 저장될 때
- WebSocket 연결 유지
### 방법
1. **Cookie 기반**: L7에서 쿠키로 서버 식별
2. **IP Hash**: 클라이언트 IP로 서버 고정
3. **Session Replication**: 서버 간 세션 공유 (Redis 등)

### 권장 방식
세션을 Redis/DB에 저장하여 Stateless 구조로 만들기 → Sticky Session 불필요, 어떤 서버로 가도 OK