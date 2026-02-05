# 부하 테스트 — getMemberOrThrow 쿼리 최적화

## 테스트 환경

- **서버**: Spring Boot 3.5.7, Java 21
- **DB**: MySQL (HikariCP 기본값, 커넥션 풀 10개)
- **부하 도구**: k6
- **모니터링**: Prometheus + Grafana (JVM 대시보드 4701, HikariCP 대시보드 6083)
- **Rate Limiter**: 테스트를 위해 비활성화 (서버 순수 성능 측정 목적)

---

## 1. Smoke 테스트

VU 1명으로 API 흐름이 정상 동작하는지 확인.

|항목|결과|
|---|---|
|트리 조회 `GET /folders`|✅ 200|
|상세 조회 `GET /posts/{id}`|✅ 200, version 정상 반환|
|수정 `PUT /posts/{id}`|✅ 200|
|check 통과율|100% (5/5)|
|평균 응답시간|175ms|

---

## 2. Load 테스트 (VU 50명)

10명 → 50명 → 50명 유지 → 0명, 총 3분 30초.

|지표|결과|
|---|---|
|p(95) 응답시간|164ms|
|평균 응답시간|145ms|
|check 통과율|100%|
|초당 요청|26.6/s|
|edit_conflict (409)|93.33%|

50명에서 안정적. 서버가 여유 있음을 확인.

---

## 3. Stress 테스트 — Before (VU 200명)

50명 → 100명 유지 → 200명 유지 → 0명, 총 7분.

|지표|결과|
|---|---|
|p(95) 응답시간|**1,210ms**|
|평균 응답시간|572ms|
|i/o timeout|5건|
|check 통과율|99.98%|
|초당 요청|69.3/s|

**200명 구간에서 응답시간이 급격히 증가.** timeout 발생 시작.

---

## 4. 병목 분석

### 원인: `getMemberOrThrow` — 매 요청마다 쿼리 3개

"이 사용자가 이 워크스페이스의 멤버인가?"를 확인하는 권한 체크 메서드
PostService, FolderService의 **모든 API가 호출**한다.

```java
// Before: 쿼리 3개
private WorkspaceMember getMemberOrThrow(Long userId, Long workspaceId) {
    Workspace workspace = workspaceRepository.findById(workspaceId);       
    // 쿼리 1
    Users user = userRepository.findById(userId);                          
    // 쿼리 2
    return workspaceMemberRepository.findByWorkspaceAndUser(workspace, user); 
    // 쿼리 3
}
```

### 반복 1회(API 3개)당 쿼리 수

|API|getMemberOrThrow|비즈니스 쿼리|합계|
|---|---|---|---|
|트리 조회 `GET /folders`|3개|2개|**5개**|
|상세 조회 `GET /posts/{id}`|3개|1개|**4개**|
|수정 `PUT /posts/{id}`|3개|2개|**5개**|
|**합계**|**9개**|**5개**|**14개**|

반복 1회에 14개 쿼리, 이 중 getMemberOrThrow가 9개 (64%).

---

## 5. 개선

workspaceId와 userId만으로 member를 바로 조회하도록 변경.

```java
// After: 쿼리 1개
private WorkspaceMember getMemberOrThrow(Long userId, Long workspaceId) {
    return workspaceMemberRepository
        .findByWorkspaceWorkspaceIdAndUserUserId(workspaceId, userId)
        .orElseThrow(() -> new BusinessException(ErrorCode.ACCESS_DENIED));
}
```

### 개선 후 반복 1회당 쿼리 수

|API|getMemberOrThrow|비즈니스 쿼리|합계|
|---|---|---|---|
|트리 조회|1개|2개|**3개**|
|상세 조회|1개|1개|**2개**|
|수정|1개|2개|**3개**|
|**합계**|**3개**|**5개**|**8개**|

반복당 쿼리 14개 → 8개, **43% 감소**.

---

## 6. Stress 테스트 — Before vs After 비교 (VU 200명)



| 지표          | Before  | After      | 변화         |
| ----------- | ------- | ---------- | ---------- |
| p(95) 응답시간  | 1,210ms | **538ms**  | **55% 감소** |
| 평균 응답시간     | 572ms   | **256ms**  | **55% 감소** |
| i/o timeout | 5건      | **0건**     | 해소         |
| check 통과율   | 99.98%  | **100%**   | 에러 제거      |
| 총 반복 수      | 9,802   | **11,867** | **21% 증가** |
| 초당 요청       | 69.3/s  | **83.9/s** | **21% 향상** |
"Stress 테스트에서 p(95) 1,210ms가 나왔고, 
서버 로그에서 매 요청마다 동일한 SELECT 3개가 반복되는 것을 확인. g
etMemberOrThrow를 단일 쿼리로 최적화하여 p(95) 538ms로 55% 개선."

---

## 7. HikariCP 커넥션 풀 분석 (추가 확인 필요)

Grafana HikariCP 대시보드에서 확인한 수치 (Stress Before 기준)

| 지표                      | 값     | 의미           |
| ----------------------- | ----- | ------------ |
| Max connections         | 10    | HikariCP 기본값 |
| Active connections (최대) | 10    | 전부 사용 중      |
| Pending threads (최대)    | 61    | 커넥션 대기 스레드   |
| Acquire time (최대)       | 386ms | 커넥션 획득 대기시간  |
| Usage time (최대)         | 122ms | 실제 쿼리 실행시간   |
쿼리 실행은 122ms인데 커넥션 획득 대기가 386ms — **응답시간의 75%가 커넥션 대기**.

> ⚠️ 이 수치는 getMemberOrThrow 최적화 전 데이터. 쿼리 최적화로 커넥션 점유 시간이 줄었으므로, 현재 상태에서 다시 측정 필요. 결과에 따라 커넥션 풀 사이즈 조정 여부 판단 예정.