
##목표 아키텍처

```
┌─────────────────┐
│  order-service  │
│    주문 생성     │
│   Port: 8081    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│                      Kafka                          │
│                  (order-topic)                      │
│                    Port: 9092                       │
└────────┬──────────────┬──────────────┬─────────────┘
         │              │              │
         ▼              ▼              ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   stock     │  │notification │  │  analytics  │
│  service    │  │  service    │  │  service    │
│  재고 차감   │  │  알림 발송   │  │  통계 수집   │
│ Port: 8082  │  │ Port: 8083  │  │ Port: 8084  │
└─────────────┘  └─────────────┘  └─────────────┘
```

---

## 인프라 구성

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│    Kafka    │  │  Zookeeper  │  │    Redis    │
│ 메시지 브로커 │  │ Kafka 관리   │  │  캐시/락    │
│ Port: 9092  │  │ Port: 2181  │  │ Port: 6379  │
└─────────────┘  └─────────────┘  └─────────────┘
```

### 각 인프라 역할

| 컴포넌트          | 역할                            | 개발자가 직접 사용?     |
| ------------- | ----------------------------- | --------------- |
| **Kafka**     | 서비스 간 이벤트 전달                  | 직접 사용           |
| **Zookeeper** | Kafka 클러스터 관리 (리더 선출, 브로커 감시) | Kafka가 내부적으로 사용 |
| **Redis**     | 캐시, 중복 요청 방지, 분산락             | 직접 사용 (선택)      |

---

##  진행 상황

|서비스|역할|상태|
|---|---|---|
|order-service|Producer - 주문 생성 → Kafka 발행|✅ 완성|
|stock-service|Consumer - 재고 차감|⬜ 미완성|
|notification-service|Consumer - 알림 발송|⬜ 미완성|
|analytics-service|Consumer - 통계 수집|⬜ 미완성|

---

##  이벤트 흐름

```
1. POST /orders 호출 (order-service)
         │
         ▼
2. Kafka에 "주문 생성" 이벤트 발행
         │
         ▼
3. 동시에 3개 서비스가 각자 메시지 수신
   ├── stock-service: 재고 차감
   ├── notification-service: 알림 발송
   └── analytics-service: 통계 기록
```

### 이벤트 드리븐의 장점

|Kafka 없이 (강결합)|Kafka 있으면 (약결합)|
|---|---|
|주문 서비스가 모든 서비스 직접 호출|주문 서비스는 Kafka에 던지고 끝|
|하나 죽으면 전체 실패|하나 죽어도 나머지 정상|
|새 서비스 추가 = 코드 수정|새 Consumer만 추가|

---

##  Spring Boot 내부 동작

```
[HTTP 요청]
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                     order-service                        │
│                                                          │
│  Controller → Producer → KafkaTemplate → Kafka Broker   │
│                                                          │
│  1. @PostMapping /orders                                 │
│  2. Order 객체 → JSON 변환                               │
│  3. kafkaTemplate.send("order-topic", message)          │
└─────────────────────────────────────────────────────────┘
```

### 파일 구조

```
order-service/
└── src/main/kotlin/com/kafkapractice/orderservice/
    ├── OrderServiceApplication.kt   # 메인 클래스
    ├── Order.kt                     # 데이터 클래스 (DTO)
    ├── OrderController.kt           # REST API 엔드포인트
    ├── OrderProducer.kt             # Kafka 메시지 발행
    └── KafkaConfig.kt               # Kafka 설정
```

### 각 파일 역할

|파일|역할|Java 비유|
|---|---|---|
|Order.kt|주문 데이터 담는 객체|DTO|
|OrderController.kt|API 엔드포인트|@RestController|
|OrderProducer.kt|Kafka로 메시지 전송|@Service|
|KafkaConfig.kt|KafkaTemplate Bean 설정|@Configuration|

---

## 테스트 방법

### 1. Docker 컨테이너 실행 (WSL)

```bash
cd ~/kafka-practice  # 또는 docker-compose.yml 위치
docker compose up -d
docker compose ps    # 상태 확인
```

### 2. Spring Boot 실행

IntelliJ에서 Run 버튼 클릭

### 3. API 호출 테스트 (Windows 터미널)

```bash
curl -X POST http://localhost:8081/orders ^
  -H "Content-Type: application/json" ^
  -d "{\"orderId\":\"1\",\"productId\":\"A001\",\"quantity\":2}"
```

### 4. Kafka 메시지 확인 (WSL)

```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic order-topic \
  --from-beginning
```

---

## 핵심 개념 정리

### Kafka 용어

|용어|설명|
|---|---|
|**Producer**|메시지를 Kafka에 보내는 쪽 (order-service)|
|**Consumer**|메시지를 Kafka에서 가져가는 쪽 (stock-service 등)|
|**Topic**|메시지 카테고리 (order-topic)|
|**Broker**|Kafka 서버|
|**Partition**|Topic을 나눈 단위 (병렬 처리용)|

### Kafka vs 일반 큐

|구분|일반 큐 (Redis List 등)|Kafka|
|---|---|---|
|읽으면|메시지 사라짐|남아있음 (설정 기간 동안)|
|여러 Consumer|한 명만 가져감|각자 다 가져갈 수 있음|
|재처리|불가능|가능 (오프셋 되돌리기)|
|저장|메모리|디스크 (영속성)|

### Spring Kafka 주요 클래스

|클래스|역할|
|---|---|
|`KafkaTemplate`|메시지 전송 도구|
|`ProducerFactory`|KafkaTemplate 생성용 설정|
|`@KafkaListener`|메시지 수신 어노테이션 (Consumer용)|

---

## Docker 컨테이너 관리

### 시작

```bash
docker compose up -d
```

### 상태 확인

```bash
docker compose ps
```

### 중지

```bash
docker compose down
```

### 로그 확인

```bash
docker logs kafka
docker logs zookeeper
docker logs redis
```

---

## 참고: Redis 활용 (추후 적용)

```
[주문 요청]
     │
     ▼
┌─────────────────────────────────────┐
│ order-service                       │
│                                     │
│  1. Redis에서 중복 주문 체크         │  ← 멱등성 보장
│  2. Redis에서 재고 캐시 조회         │  ← 성능 향상
│  3. Kafka에 이벤트 발행             │
└─────────────────────────────────────┘
```

---


- Kafka + Zookeeper + Redis Docker 환경 구성
- order-service (Producer) 완성
- (예정) stock-service (Consumer) 구현
- (예정) notification-service (Consumer) 구현
- (예정) analytics-service (Consumer) 구현