

### 필요성

- 장애 서버로 트래픽 보내면 안 됨
- 자동으로 장애 감지 및 제외

### 종류

#### TCP Health Check

```
LB → Server:Port 연결 시도
성공: Healthy
실패: Unhealthy
```

#### HTTP Health Check

```
LB → GET /health HTTP/1.1
200 OK: Healthy
5xx or timeout: Unhealthy
```

### 주요 설정값

|설정|설명|예시|
|---|---|---|
|interval|체크 주기|5초|
|timeout|응답 대기 시간|3초|
|healthy_threshold|정상 판정 횟수|2회 연속 성공|
|unhealthy_threshold|비정상 판정 횟수|3회 연속 실패|

### 헬스체크 엔드포인트 구현 예시 (Spring Boot)

java

```java
@RestController
public class HealthController {
    
    @GetMapping("/health")
    public ResponseEntity<String> health() {
        // DB 연결, 필수 서비스 체크 등
        if (isHealthy()) {
            return ResponseEntity.ok("OK");
        }
        return ResponseEntity.status(503).body("Service Unavailable");
    }
}
```