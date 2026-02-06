
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


**병목 1 발견 과정:** 
Stress 테스트(200명)에서 p(95) 1,210ms 발생 → 
서버 로그에서 매 요청마다 `[SLOW QUERY]` 50~60ms짜리가 반복되는 것을 확인 → 
서버 로그에서 매 요청마다 동일한 SELECT 3개가 반복되는 것을 확인 하고 있었음 → 
모든 API가 이걸 호출하므로 반복 1회에 불필요한 쿼리 9개 발생 → 
단일 쿼리로 최적화 (응답시간 55% 감소)

**병목 2 발견 과정:** 
쿼리 최적화 후에도 p(95) 538ms → 
Grafana JVM 대시보드에서 timed-waiting 스레드 109개 확인 →
HikariCP 대시보드 추가하여 확인 → 
Active 10(풀 전부 사용), Pending 55(대기 스레드), 
Acquire time 386ms(커넥션 대기) → 
쿼리 실행 122ms인데 커넥션 대기가 386ms로 응답시간의 75%를 차지 → 
RDS(db.t4g.micro) max_connections 40 확인 후 풀 사이즈 10 → 20으로 조정



커넥션풀 10 시작전

![[Pasted image 20260206132934.png]]

종료후
```

eborder@DESKTOP-49BM4V1:~/untitles-api$ k6 run -e BASE_URL=http://172.29.96.1:8070 -e SESSION_ID=FFB52020590D5E295C8E9A41FD575E92 -e WORKSPACE_ID=2 load-test/stressreal.js

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: load-test/stressreal.js
        output: -

     scenarios: (100.00%) 2 scenarios, 600 max VUs, 7m30s max duration (incl. graceful stop):
              * readers: Up to 300 looping VUs for 7m0s over 6 stages (gracefulRampDown: 30s, exec: readScenario, gracefulStop: 30s)
              * writers: Up to 300 looping VUs for 7m0s over 6 stages (gracefulRampDown: 30s, exec: writeScenario, gracefulStop: 30s)



  █ THRESHOLDS

    checks
    ✓ 'rate>0.6' rate=100.00%

    http_req_duration
    ✗ 'p(95)<5000' p(95)=5.64s


  █ TOTAL RESULTS

    checks_total.......: 33724   78.437909/s
    checks_succeeded...: 100.00% 33724 out of 33724
    checks_failed......: 0.00%   0 out of 33724

    ✓ detail: status 200
    ✓ tree: status 200
    ✓ edit: 200 or 409

    CUSTOM
    detail_duration................: avg=3.29s  min=104.25ms med=2.4s   max=8.15s  p(90)=5.57s  p(95)=5.63s
    edit_conflict..................: 91.90% 13774 out of 14988
    edit_duration..................: avg=3.18s  min=103.29ms med=2.14s  max=8s     p(90)=5.57s  p(95)=5.63s
    tree_duration..................: avg=3.33s  min=113.87ms med=2.59s  max=7.89s  p(90)=5.58s  p(95)=5.65s

    HTTP
    http_req_duration..............: avg=3.25s  min=103.29ms med=2.22s  max=8.15s  p(90)=5.57s  p(95)=5.64s
      { expected_response:true }...: avg=3.24s  min=104.25ms med=2.2s   max=8.15s  p(90)=5.57s  p(95)=5.64s
    http_req_failed................: 40.84% 13774 out of 33724
    http_reqs......................: 33724  78.437909/s

    EXECUTION
    iteration_duration.............: avg=14.72s min=2.78s    med=13.61s max=50.54s p(90)=27.78s p(95)=38.92s
    iterations.....................: 11029  25.652108/s
    vus............................: 1      min=1              max=600
    vus_max........................: 600    min=600            max=600

    NETWORK
    data_received..................: 30 MB  69 kB/s
    data_sent......................: 8.0 MB 19 kB/s

```

**Before (쿼리 3개)** — `FolderService.getRootFolders`에서 자격 확인 쿼리에서 3개 날리는 
```
select ... from workspace where workspace_id=?     ← getMemberOrThrow 쿼리 1
select ... from users where user_id=?               ← getMemberOrThrow 쿼리 2
select ... from workspace_member where ...           ← getMemberOrThrow 쿼리 3
select distinct ... from folder left join post ...   ← 비즈니스 쿼리
select ... from post where folder_id is null         ← 비즈니스 쿼리
```
→ 총 149ms
**After (단일 쿼리로 수정)** — 같은 API에서:
```
select ... from workspace_member left join workspace left join users where workspace_id=? and user_id=?  ← getMemberOrThrow 단일 쿼리
select distinct ... from folder left join post ...   ← 비즈니스 쿼리
select ... from post where folder_id is null         ← 비즈니스 쿼리
→ 총 46ms
```

수정 후 다시 테스트 
![[Pasted image 20260206135949.png]]


| 지표              | 1차-Before (쿼리3+풀10) | 1차-After (쿼리1+풀10) | 변화     |
| --------------- | ------------------- | ------------------ | ------ |
| p(95)           | 5.64s               | **4.36s**          | 23% 감소 |
| 평균              | 3.25s               | **2.45s**          | 25% 감소 |
| 처리량             | 78.4/s              | **94.1/s**         | 20% 향상 |
| Pending threads | 189                 | **189**            | 동일     |

쿼리 최적화만으로 응답시간은 개선됐지만, Pending threads가 여전히 189인 게 핵심 커넥션 점유 시간이 줄어서 회전은 빨라짐,
쿼리 최적화만으로는 부족, 커넥션 풀 조정 필요

커넥션 풀 20 으로 수정


eborder@DESKTOP-49BM4V1:~/untitles-api$ k6 run -e BASE_URL=http://172.29.96.1:8070 -e SESSION_ID=6742252BB8F8EA70D7A221FA3AC25E9D -e WORKSPACE_ID=2 load-test/stressreal.js

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: load-test/stressreal.js
        output: -

     scenarios: (100.00%) 2 scenarios, 600 max VUs, 7m30s max duration (incl. graceful stop):
              * readers: Up to 300 looping VUs for 7m0s over 6 stages (gracefulRampDown: 30s, exec: readScenario, gracefulStop: 30s)
              * writers: Up to 300 looping VUs for 7m0s over 6 stages (gracefulRampDown: 30s, exec: writeScenario, gracefulStop: 30s)

WARN[0216] Request Failed                                error="Put \"http://172.29.96.1:8070/api/v1/workspaces/2/posts/6\": dial: i/o timeout"
WARN[0254] Request Failed                                error="Get \"http://172.29.96.1:8070/api/v1/workspaces/2/posts/9\": dial: i/o timeout"
WARN[0254] Request Failed                                error="Get \"http://172.29.96.1:8070/api/v1/workspaces/2/folders\": dial: i/o timeout"


  █ THRESHOLDS

    checks
    ✓ 'rate>0.6' rate=99.99%

    http_req_duration
    ✓ 'p(95)<5000' p(95)=1.44s


  █ TOTAL RESULTS

    checks_total.......: 71663  167.622907/s
    checks_succeeded...: 99.99% 71660 out of 71663
    checks_failed......: 0.00%  3 out of 71663

    ✗ tree: status 200
      ↳  99% — ✓ 15590 / ✗ 1
    ✗ detail: status 200
      ↳  99% — ✓ 22933 / ✗ 1
    ✗ edit: 200 or 409
      ↳  99% — ✓ 33137 / ✗ 1

    CUSTOM
    detail_duration................: avg=700.32ms min=0s     med=473.76ms max=3.1s   p(90)=1.39s  p(95)=1.45s
    edit_conflict..................: 92.60% 30686 out of 33138
    edit_duration..................: avg=684.23ms min=0s     med=371.43ms max=3.31s  p(90)=1.39s  p(95)=1.45s
    error_count....................: 3      0.007017/s
    tree_duration..................: avg=632.26ms min=0s     med=433.1ms  max=3.12s  p(90)=1.31s  p(95)=1.37s

    HTTP
    http_req_duration..............: avg=678.07ms min=0s     med=410.77ms max=3.31s  p(90)=1.38s  p(95)=1.44s
      { expected_response:true }...: avg=662.39ms min=26.7ms med=388.94ms max=3.12s  p(90)=1.38s  p(95)=1.42s
    http_req_failed................: 42.82% 30689 out of 71663
    http_reqs......................: 71663  167.622907/s

    EXECUTION
    iteration_duration.............: avg=6.95s    min=2.66s  med=5.99s    max=33.35s p(90)=12.72s p(95)=15.9s
    iterations.....................: 22935  53.645973/s
    vus............................: 1      min=1              max=600
    vus_max........................: 600    min=600            max=600

    NETWORK
    data_received..................: 62 MB  144 kB/s
    data_sent......................: 17 MB  40 kB/s




running (7m07.5s), 000/600 VUs, 22935 complete and 0 interrupted iterations
readers ✓ [======================================] 000/300 VUs  7m0s
writers ✓ [======================================] 000/300 VUs  7m0s
