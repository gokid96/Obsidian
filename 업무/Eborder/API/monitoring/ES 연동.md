****Handler > Messages (행 펼침):**

- MySQL에 있는 메시지( iapi-status-updater 파싱된것만 확인가능) → 관련 ES 로그 보기
- 정상 처리된 메시지의 상세 처리 과정 확인

**Handler > Logs:**

- ES 전체 검색
- **iapi-status-updater 파싱 에러 메시지도 검색 가능**
- MySQL에 없는 메시지도 찾을 수 있음

---

 두 가지 사용

### 시나리오 1: 정상 메시지 추적

```
Messages2 → 행 펼침 → 처리 로그 확인
```

### 시나리오 2: 파싱 에러 메시지 찾기

```
Handler > Logs → "edifactParser" 또는 "Error" 검색
→ 파싱 실패한 메시지 원본 확인 가능
```