## 테스트 환경

- **DB**: MySQL on RDS (db.t4g.micro — 2 vCPU, 1GB RAM, max_connections 40)
- **부하 도구**: k6
- **모니터링**: Prometheus + Grafana (JVM 대시보드 4701, HikariCP 대시보드 6083)

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

"이 사용자가 이 워크스페이스의 멤버인가?"를 확인하는 권한 체크 메서드. PostService, FolderService의 **모든 API가 호출**.

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

|API|getMemberOrThrow|비즈니스 쿼리|
|---|---|---|
|트리 조회 `GET /folders`|3개|2개|
|상세 조회 `GET /posts/{id}`|3개|1개|
|수정 `PUT /posts/{id}`|3개|2개|
|**합계**|**9개**|**5개**|

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

|API|getMemberOrThrow|비즈니스 쿼리|
|---|---|---|
|트리 조회|1개|2개|
|상세 조회|1개|1개|
|수정|1개|2개|
|**합계**|**3개**|**5개**|

반복당 쿼리 14개 → 8개

### Stress 테스트 — After (VU 200명, 쿼리 최적화 후)

```
scenarios: (100.00%) 1 scenario, 200 max VUs, 7m30s max duration (incl. graceful stop):
          * default: Up to 200 looping VUs for 7m0s over 6 stages (gracefulRampDown: 30s, gracefulStop: 30s)

█ THRESHOLDS
  checks
  ✓ 'rate>0.6' rate=100.00%
  http_req_duration
  ✓ 'p(95)<5000' p(95)=538.63ms

█ TOTAL RESULTS
  checks_total.......: 35601   83.950891/s
  checks_succeeded...: 100.00% 35601 out of 35601
  checks_failed......: 0.00%   0 out of 35601
  ✓ tree: status 200
  ✓ detail: status 200
  ✓ edit: 200 or 409

  CUSTOM
  detail_duration................: avg=283.05ms min=101.33ms med=182.98ms max=1.1s  p(90)=514.69ms p(95)=549.89ms
  edit_conflict..................: 97.65% 11589 out of 11867
  edit_duration..................: avg=286.31ms min=102.83ms med=186.06ms max=1.59s p(90)=518.38ms p(95)=558.47ms
  tree_duration..................: avg=201.15ms min=25.26ms  med=103.66ms max=1.43s p(90)=436.34ms p(95)=471.08ms

  HTTP
  http_req_duration..............: avg=256.84ms min=25.26ms  med=164.4ms  max=1.59s p(90)=500.06ms p(95)=538.63ms
    { expected_response:true }...: avg=242.27ms min=25.26ms  med=152.89ms max=1.43s p(90)=488.57ms p(95)=528.86ms
  http_req_failed................: 32.55% 11589 out of 35601
  http_reqs......................: 35601  83.950891/s

  EXECUTION
  iteration_duration.............: avg=4.51s    min=2.8s     med=4.48s    max=7.2s  p(90)=5.46s    p(95)=5.67s
  iterations.....................: 11867  27.98363/s

running (7m04.1s), 000/200 VUs, 11867 complete and 0 interrupted iterations
default ✓ [======================================] 000/200 VUs  7m0s
```

| 지표          | Before  | After      |
| ----------- | ------- | ---------- |
| p(95) 응답시간  | 1,210ms | **538ms**  |
| 평균 응답시간     | 572ms   | **256ms**  |
| i/o timeout | 5건      | **0건**     |
| check 통과율   | 99.98%  | **100%**   |
| 총 반복 수      | 9,802   | **11,867** |
| 초당 요청       | 69.3/s  | **83.9/s** |

---

## 6. HikariCP 커넥션 풀 분석

Grafana HikariCP 대시보드에서 확인한 수치 (VU 200 Stress Before 기준)

|지표|값|의미|
|---|---|---|
|Max connections|10|HikariCP 기본값|
|Active connections (최대)|10|전부 사용 중|
|Pending threads (최대)|61|커넥션 대기 스레드|
|Acquire time (최대)|386ms|커넥션 획득 대기시간|
|Usage time (최대)|122ms|실제 쿼리 실행시간|

커넥션 획득 대기가 386ms — **응답시간의 75%가 커넥션 대기**.

**발견 과정:** 쿼리 최적화 후에도 p(95) 538ms
→ Grafana JVM 대시보드에서 timed-waiting 스레드 109개 확인 
→ HikariCP 대시보드 추가하여 확인 
→ Active 10(풀 전부 사용), Pending 55(대기 스레드), Acquire time 386ms(커넥션 대기) 
→ 쿼리 실행 122ms인데 커넥션 대기가 386ms로 응답시간의 75%를 차지 
→ RDS(db.t4g.micro) max_connections 40 확인 후 풀 사이즈 10 → 20으로 조정 예정

---

## 7. VU 600 Stress 테스트 — 커넥션 풀 튜닝

VU 200에서 쿼리 최적화로 개선했으나, 커넥션 풀 병목이 여전히 있음. 더 높은 부하에서 커넥션 풀 조정 효과를 확인하기 위해 VU 600으로 추가 Stress 테스트를 진행

> 이하 결과는 VU 600 기준이며, 앞선 VU 200 결과와 직접 비교할 수 없음

### 7-1. 쿼리 최적화 효과 확인 (풀 10 유지)

**Before (쿼리 3개)** — `FolderService.getRootFolders` 기준:

```
select ... from workspace where workspace_id=?     ← getMemberOrThrow 쿼리 1
select ... from users where user_id=?               ← getMemberOrThrow 쿼리 2
select ... from workspace_member where ...           ← getMemberOrThrow 쿼리 3
select distinct ... from folder left join post ...   ← 비즈니스 쿼리
select ... from post where folder_id is null         ← 비즈니스 쿼리
```

→ 총 149ms

**After (단일 쿼리)** — 같은 API:

```
select ... from workspace_member left join workspace left join users where workspace_id=? and user_id=?  ← 단일 쿼리
select distinct ... from folder left join post ...   ← 비즈니스 쿼리
select ... from post where folder_id is null         ← 비즈니스 쿼리
```

→ 총 46ms

| 지표              | 쿼리3 + 풀10 | 쿼리1 + 풀10  |
| --------------- | --------- | ---------- |
| p(95)           | 5.64s     | **4.36s**  |
| 평균              | 3.25s     | **2.45s**  |
| 처리량             | 78.4/s    | **94.1/s** |
| Pending threads | 189       | **189**    |

쿼리 최적화만으로 응답시간은 개선됐지만, Pending threads가 여전히 189로 동일. 커넥션 점유 시간이 줄어서 회전은 빨라졌으나, 풀 자체가 부족한 상태. 커넥션 풀 조정이 필요하다는 것을 확인.

**Grafana — 쿼리3 + 풀10:**

![[Pasted image 20260206133818.png]]

Active 10(풀 전부 사용), Pending 189, Acquire time 최대 2초. 커넥션 풀이 완전히 포화된 상태.

**Grafana — 쿼리1 + 풀10:**

![[Pasted image 20260206135949.png]]

쿼리 최적화 후에도 Active 10, Pending 189로 동일. 다만 Acquire time이 2초 → 1초로 감소. 커넥션 점유 시간이 줄어 회전이 빨라졌지만, 풀 크기 자체가 부족하여 대기 스레드 수는 변하지 않음

### 7-2. 커넥션 풀 10 → 20 조정

RDS(db.t4g.micro) max_connections 40을 확인하고, 풀 사이즈를 10 → 20으로 변경.

| 지표          | 쿼리1 + 풀10 | 쿼리1 + 풀20   | 변화         |
| ----------- | --------- | ----------- | ---------- |
| p(95)       | 4.36s     | **1.46s**   | **67% 감소** |
| 평균          | 2.45s     | **710ms**   | **71% 감소** |
| 처리량         | 94.1/s    | **164.9/s** | **75% 향상** |
| 총 반복 수      | —         | **22,644**  | —          |
| i/o timeout | 0건        | **7건**      | 발생         |
| check 통과율   | 100%      | **99.99%**  | 미세 하락      |

커넥션 풀 확장으로 대폭 개선. 다만 i/o timeout 7건이 새로 발생했으며, `min=0s`인 요청이 있는 것으로 보아 TCP 레벨에서 연결 자체가 안 된 케이스로 네트워크 일시 중단 원인 가능성 있으나, 재현되지 않음

> 참고: `http_req_failed: 42.76%`는 대부분 409 Conflict(낙관적 락 충돌)이며, edit_conflict 92.61%로 의도된 동작

**Grafana — 쿼리1 + 풀20:**

![[Pasted image 20260206153746.png]]

Active 20(풀 전부 사용), Pending **179**, Acquire time 최대 946ms. 풀을 2배로 늘렸지만 Pending이 189 → 179로 거의 줄지 않았다 이는 애플리케이션 쪽 커넥션 대기는 해소되었으나, RDS(db.t4g.micro, 2 vCPU, 1GB RAM)의 처리 능력이 한계에 도달했기 때문 커넥션 20개가 동시에 쿼리를 보내도 vCPU 2개로는 병렬 처리에 한계가 있어, 병목이 애플리케이션 커넥션 풀 → DB 서버 처리 능력으로 이동한 것으로 판단

|지표|쿼리3 + 풀10|쿼리1 + 풀10|쿼리1 + 풀20|
|---|---|---|---|
|Pending threads|189|189|**179**|
|Acquire time (최대)|2s|1s|**946ms**|
|Usage time (최대)|154ms|114ms|**135ms**|

Acquire time은 단계마다 줄었지만, Usage time(실제 쿼리 실행 시간)은 비슷하게 유지. DB 자체의 처리 속도는 변하지 않았음을 확인.

### 7-3. VU 600 전체 비교 (3단계)

|지표|쿼리3 + 풀10|쿼리1 + 풀10|쿼리1 + 풀20|
|---|---|---|---|
|p(95)|5.64s|4.36s|**1.46s**|
|평균|3.25s|2.45s|**710ms**|
|처리량|78.4/s|94.1/s|**164.9/s**|

---

## 8. 정리

### 개선

|단계|내용|핵심 효과|
|---|---|---|
|병목 1|getMemberOrThrow 쿼리 3개 → 1개|반복당 쿼리 43% 감소|
|병목 2|HikariCP 풀 10 → 20|커넥션 대기 해소|

### 결과

|기준|Before (최초)|After (최종)|변화|
|---|---|---|---|
|VU 200 p(95)|1,210ms|538ms|**55% 감소**|
|VU 600 p(95)|5.64s|1.46s|**74% 감소**|
|VU 600 처리량|78.4/s|164.9/s|**110% 향상**|

### 병목 이동 경로

```
쿼리 과다 (getMemberOrThrow 3개)
  → 쿼리 최적화로 해소
    → 커넥션 풀 부족 (풀 10, Pending 189)
      → 풀 확장으로 해소
        → RDS 처리 능력 한계 (db.t4g.micro, 2 vCPU)  ← 현재 병목
```

- RDS 인스턴스 스펙 업그레이드 검토 (db.t4g.micro → db.t4g.small 이상)
- 또는 읽기 전용 쿼리에 대한 캐싱 도입 검토 (getMemberOrThrow 결과 등)