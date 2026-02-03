
Apache Kafka는 LinkedIn에서 개발한 **분산 이벤트 스트리밍 플랫폼**

---

## 핵심 구성요소

### Kafka 외부 (클라이언트)

- **Producer:** 메시지를 Kafka에 보내는 역할
- **Consumer:** 메시지를 Kafka에서 읽어가는 역할
- **Consumer Group:** 같은 그룹의 Consumer들이 Partition을 나눠서 처리

### Kafka 내부

- **Broker:** Kafka 서버 인스턴스. 여러 대가 모여 클러스터 구성
- **Topic:** 메시지가 저장되는 논리적 채널 (예: `order-topic`, `user-topic`)
- **Partition:** Topic을 쪼갠 조각. 병렬 처리와 확장성 제공

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Cluster                            │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Broker 1   │  │  Broker 2   │  │  Broker 3   │         │
│  │  Partition 0│  │  Partition 1│  │  Partition 2│         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

---

## 아키텍처 예시

```
┌─────────────────┐
│  order-service  │ ← Producer
│    주문 생성     │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│                      Kafka                          │
│                  (order-topic)                      │
└────────┬──────────────┬──────────────┬─────────────┘
         │              │              │
         ▼              ▼              ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   stock     │  │notification │  │  analytics  │ ← Consumer
│  service    │  │  service    │  │  service    │
└─────────────┘  └─────────────┘  └─────────────┘
```

- Producer(order-service)는 누가 메시지를 읽는지 모름
- Consumer들은 서로 독립적으로 동작
- 새 Consumer 추가 시 기존 서비스 수정 불필요

---

## 주요 특징

- **높은 처리량:** 초당 수백만 건 메시지 처리 가능
- **내구성:** 디스크 저장 + 복제로 데이터 손실 방지
- **확장성:** Broker, Partition 추가로 수평 확장
- **메시지 보존:** 소비 후에도 설정 기간 동안 보관

---

## 사용 사례

- 마이크로서비스 간 비동기 통신
- 실시간 로그 수집 및 분석
- 이벤트 소싱 아키텍처
- 데이터 파이프라인 (ETL)

---

## 코드 예시 (Spring 없이 Kafka 라이브러리만 사용)

### Producer
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("my-topic", "key", "Hello Kafka!"));
producer.close();
```

### Consumer
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("my-topic"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        System.out.println(record.value());
    }
}
```

### Spring Kafka
```java
// Producer— 메시지를 보내는 쪽
kafkaTemplate.send("order-topic", orderEvent);
```

```java
// Consumer — 메시지를 받는 쪽
@KafkaListener(topics = "order-topic")
public void handleOrder(OrderEvent event) {
    // 처리 로직
}
```

---

## 관련 개념
