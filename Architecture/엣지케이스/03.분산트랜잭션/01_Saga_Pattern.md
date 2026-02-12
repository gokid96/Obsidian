## 1. 보상 트랜잭션 자체가 실패

| 항목        | 내용                                       |
| --------- | ---------------------------------------- |
| **문제**    | 보상 로직도 네트워크 호출이므로 실패 가능                  |
| **왜 발생**  | 보상 대상 서비스가 장애이거나 네트워크 불안정                |
| **금융 예시** | 출금 성공 → 입금 실패 → 출금 환불(보상) 실패 → 돈이 사라진 상태 |

### 해결: 3단계 방어

**1단계 - 재시도:** Exponential Backoff + Jitter로 최대 N회 재시도. 보상은 반드시 성공해야 하므로 넉넉하게 잡는다.

**2단계 - 멱등성 보장:** 재시도 시 같은 보상이 두 번 실행되면 안 됨. sagaId로 이미 처리된 보상인지 체크.

```java
public void refund(String sagaId, String userId, Long amount) {
    if (compensationRepository.existsBySagaId(sagaId)) return;  // 이미 처리됨
    account.addBalance(amount);
    compensationRepository.save(sagaId);
}
```

**3단계 - DLQ + 수동 개입:** 재시도 전부 실패하면 Dead Letter Queue에 저장 + 운영자 알림(Slack/PagerDuty). 운영자가 원인 파악 후 수동 환불 처리.

---

## 2. Isolation 부재 — 중간 상태 dirty read

|항목|내용|
|---|---|
|**문제**|보상 실행 전 중간 상태 데이터를 다른 트랜잭션이 읽어버림|
|**왜 발생**|Saga는 ACID 중 Isolation을 보장하지 않음|
|**금융 예시**|송금 중 잔액 조회 → "출금은 됐는데 입금 전" 상태의 잔액이 보임|

### 해결 1: 시맨틱 락

계좌 테이블에 status 필드 추가. 거래 시작 시 TRANSFER_IN_PROGRESS로 변경하여 다른 거래를 차단.

```sql
UPDATE account SET status = 'TRANSFER_IN_PROGRESS'
WHERE user_id = 'A' AND status = 'AVAILABLE';
-- 0건이면 이미 다른 거래 진행 중 → 거부
-- 조회 시 status 보고 "송금 처리 중" 표시
-- 완료 후 AVAILABLE로 복구
-- 서비스 죽을 경우 대비 TTL 스케줄러로 5분 초과 시 강제 해제
```

### 해결 2: 상태 머신

Saga의 각 단계를 상태로 정의하고 DB에 저장. 현재 상태를 보고 허용 가능한 액션만 실행.

```
INITIATED → 출금 성공 → DEBITED → 입금 성공 → CREDITED → 락 해제 → COMPLETED
            DEBITED → 입금 실패 → DEBIT_COMPENSATING → 보상 성공 → COMPENSATED
```

잔액 조회 시 활성 Saga가 있으면 "송금 처리 중 (반영 대기)" 표시.

### 해결 3: 클라이언트 낙관적 업데이트 (UX 레벨)

서버 처리와 별개로 클라이언트에서 즉시 차감 표시 + "처리 중" 안내. Saga 완료/실패 시 확정 통보.

---

## 3. 오케스트레이터 장애 시 Saga 복구

|항목|내용|
|---|---|
|**문제**|오케스트레이터가 죽으면 진행 중이던 Saga 상태 유실|
|**왜 발생**|오케스트레이터가 상태를 메모리에만 들고 있는 경우|
|**트레이드오프**|매 스텝마다 DB 쓰기 → 성능 오버헤드, 하지만 금융에서는 필수|

### 해결 1: Saga 상태를 매 스텝마다 DB에 저장

```sql
CREATE TABLE saga_state (
    saga_id VARCHAR(50) PRIMARY KEY,
    current_state VARCHAR(30),    -- INITIATED, DEBITED, CREDITED...
    payload JSON,                 -- { userId, targetId, amount }
    updated_at TIMESTAMP
);
```

매 단계 완료 시 상태 업데이트. 어디서 죽어도 마지막 상태를 알 수 있음.

### 해결 2: 재시작 시 미완료 Saga 스캔

오케스트레이터가 살아나면 COMPLETED/COMPENSATED/FAILED가 아닌 Saga를 찾아서 마지막 상태부터 이어서 처리.

```
DEBITED 상태 → 출금은 했고 입금은 아직 → 입금부터 이어서 진행
CREDITED 상태 → 입금까지 했고 락 해제만 남음 → 락 해제 실행
DEBIT_COMPENSATING 상태 → 보상 중이었음 → 보상 재시도
```

### 해결 3: 오케스트레이터 다중화

오케스트레이터를 여러 인스턴스로 띄움. 같은 Saga를 두 인스턴스가 동시에 처리하면 안 되므로 Redis 분산 락으로 Saga별 처리 권한 관리.