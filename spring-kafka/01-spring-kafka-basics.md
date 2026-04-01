# Spring Kafka 기초 — KafkaTemplate과 @KafkaListener 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `KafkaTemplate.send()`가 내부에서 `ProducerFactory` → `KafkaProducer`로 연결되는 과정은?
- `@KafkaListener`가 `ConcurrentKafkaListenerContainerFactory`를 통해 컨테이너를 생성하는 방식은?
- `AckMode`(`BATCH`, `RECORD`, `MANUAL`, `MANUAL_IMMEDIATE`)의 차이는?
- `concurrency` 설정이 Consumer 스레드와 파티션 할당에 어떻게 영향을 주는가?
- `KafkaListenerContainerFactory` 빈을 커스텀해야 하는 이유는?
- `@KafkaListener` 의 `containerFactory` 속성이 필요한 시점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Kafka는 Kafka 클라이언트를 Spring 방식으로 추상화한다. 하지만 내부 구조를 모르면 설정만 바꿔가며 동작을 기대하게 되고, 예상과 다르게 동작할 때 원인을 알 수 없다.

`@KafkaListener`가 멀티 스레드로 동작하는 원리를 모르면 `concurrency=3`으로 설정했는데 파티션이 1개라 1개 스레드만 실제로 쓰이는 것을 모르고 서버 자원만 낭비한다. `AckMode`를 모르면 처리 완료 전 offset이 커밋되어 메시지가 유실된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: AckMode=BATCH로 설정했는데 중간에 예외 발생 시 동작 오해

  설정: AckMode.BATCH (기본값)
  동작: poll()로 받은 배치 전체 처리 완료 후 커밋

  오해: "예외가 발생하면 해당 배치 전체가 재처리된다"
  실제: 예외 핸들러(DefaultErrorHandler)가 없으면
        예외 발생 메시지에서 Seek → 재시도 → 소진 후 skip or DLT
        AckMode와 에러 핸들러가 독립적으로 동작

  올바른 이해:
    AckMode: 정상 처리 완료 후 커밋 방식
    에러 핸들러: 예외 발생 시 재시도/DLT 처리
    두 설정은 별개로 구성

실수 2: concurrency 설정이 무조건 병렬 처리를 보장한다고 오해

  설정: concurrency=6 (6개 스레드)
  토픽 파티션: 3개

  결과:
    KafkaMessageListenerContainer 6개 생성
    파티션 3개 → Consumer 3개만 파티션 할당받음
    나머지 3개 Consumer: IDLE (파티션 없음)

  비용: 스레드 6개 생성됐지만 3개만 실제 처리
  올바른 설정: concurrency = 파티션 수 (최적)

실수 3: KafkaTemplate.send()가 동기 블로킹이라고 오해

  코드:
    kafkaTemplate.send("orders", key, value);
    // "이 다음 줄 실행 시 브로커가 메시지를 받았겠지"

  실제:
    send()는 CompletableFuture<SendResult<K,V>> 반환
    즉시 반환 (비동기)
    브로커 응답 확인하려면 .get() 또는 콜백 필요

  올바른 사용:
    // 동기 확인이 필요한 경우
    kafkaTemplate.send("orders", key, value).get();

    // 비동기 콜백
    kafkaTemplate.send("orders", key, value)
                 .whenComplete((result, ex) -> {
                     if (ex != null) handleError(ex);
                     else log.info("offset: {}", result.getRecordMetadata().offset());
                 });
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
AckMode 선택 기준:
  BATCH (기본값):
    poll() 배치 전체 처리 후 1번 커밋
    처리량 높음, 구현 단순
    At-Least-Once (배치 중 실패 시 전체 배치 재처리 가능)

  RECORD:
    레코드마다 커밋
    처리 중 실패 시 해당 레코드부터 재처리
    처리량 낮음 (커밋 횟수 많음)

  MANUAL_IMMEDIATE:
    명시적 ack.acknowledge() 시 즉시 커밋
    세밀한 커밋 제어
    At-Least-Once 보장

  MANUAL:
    명시적 ack.acknowledge() 시 커밋 (다음 poll() 시)
    약간의 지연, MANUAL_IMMEDIATE보다 덜 빈번한 커밋

Spring Kafka 기본 설정:
  spring:
    kafka:
      consumer:
        group-id: order-group
        auto-offset-reset: earliest
        key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
        value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
        properties:
          spring.json.trusted.packages: "com.example.*"
      listener:
        ack-mode: MANUAL_IMMEDIATE
        concurrency: 3          # 파티션 수에 맞게
        poll-timeout: 3000      # 3초 poll 타임아웃
```

---

## 🔬 내부 동작 원리

### 1. KafkaTemplate 내부 구조

```
KafkaTemplate 빈 생성 흐름:

  ProducerFactory<K,V>
    └─ DefaultKafkaProducerFactory (기본 구현)
         └─ KafkaProducer 인스턴스 관리
              (스레드당 또는 공유 인스턴스)

  KafkaTemplate<K,V>
    └─ ProducerFactory를 통해 KafkaProducer 획득
    └─ send() 호출 시:
         1. ProducerFactory.createProducer()
         2. KafkaProducer.send(ProducerRecord)
         3. 결과를 SettableListenableFuture에 래핑

  트랜잭션 없을 때:
    DefaultKafkaProducerFactory가 Producer를 캐시/공유
    스레드마다 새 Producer 생성 (기본) 또는 공유 (설정에 따라)

  트랜잭션 있을 때:
    TransactionSynchronizationManager와 연동
    트랜잭션 범위 내에서 동일 Producer 재사용

  send() 반환값:
    CompletableFuture<SendResult<K,V>>
    (Spring Kafka 3.0+, 이전: ListenableFuture)
    SendResult: RecordMetadata + ProducerRecord
```

### 2. @KafkaListener 내부 동작

```
@KafkaListener 어노테이션 처리 과정:

  1. KafkaListenerAnnotationBeanPostProcessor
     애플리케이션 컨텍스트 초기화 시 모든 @KafkaListener 탐색

  2. KafkaListenerContainerFactory
     각 @KafkaListener에 대해 MessageListenerContainer 생성

  3. ConcurrentKafkaListenerContainerFactory
     (기본 팩토리)
     concurrency 수만큼 KafkaMessageListenerContainer 생성
     각 컨테이너가 별도 스레드에서 poll() 루프 실행

  4. KafkaMessageListenerContainer 내부:
     ├─ ListenerConsumer (Runnable)
     │    └─ KafkaConsumer.poll() 루프
     │    └─ 레코드를 @KafkaListener 메서드로 디스패치
     └─ HeartbeatThread (별도 스레드)

  @KafkaListener 예시:
    @KafkaListener(
        topics = "orders",
        groupId = "order-group",
        concurrency = "3",          // 이 리스너만 3개 스레드
        containerFactory = "batchFactory"  // 커스텀 팩토리
    )
    public void consume(ConsumerRecord<String, Order> record) {
        processOrder(record.value());
    }
```

### 3. AckMode별 동작 상세

```
AckMode.BATCH:
  poll() → [record1, record2, ..., recordN] 수신
  → 모든 레코드 처리 완료
  → commitSync/commitAsync (1번)
  예외 발생 시: ErrorHandler가 처리 (재시도, DLT 등)

AckMode.RECORD:
  poll() → record1 처리 → commitSync
  → record2 처리 → commitSync
  → ...
  각 레코드마다 브로커 왕복 → 처리량 낮음
  개별 레코드 실패가 다음 레코드에 영향 없음

AckMode.MANUAL_IMMEDIATE:
  @KafkaListener 메서드에 Acknowledgment 파라미터 추가
  
  @KafkaListener(topics = "orders")
  public void consume(ConsumerRecord<K,V> record,
                      Acknowledgment ack) {
      process(record);
      ack.acknowledge();  // 즉시 commitSync
  }
  
  ack.acknowledge() 호출 전 크래시 → 재처리 (At-Least-Once)
  ack.acknowledge() 호출 후 크래시 → 커밋됨, 재처리 없음

AckMode.MANUAL:
  ack.acknowledge() 호출 시 커밋 예약
  다음 poll() 호출 시 실제 커밋
  → MANUAL_IMMEDIATE보다 브로커 왕복 감소
```

### 4. concurrency와 파티션 할당

```
concurrency 동작:

  설정: concurrency=3, 토픽 파티션=6개

  생성되는 컨테이너:
    KafkaMessageListenerContainer-0 (Consumer 스레드 1)
    KafkaMessageListenerContainer-1 (Consumer 스레드 2)
    KafkaMessageListenerContainer-2 (Consumer 스레드 3)

  파티션 할당:
    Consumer-0 → P0, P1
    Consumer-1 → P2, P3
    Consumer-2 → P4, P5

  concurrency=6, 파티션=3개:
    Consumer-0 → P0
    Consumer-1 → P1
    Consumer-2 → P2
    Consumer-3, 4, 5 → IDLE (파티션 없음)
    → 스레드 3개 낭비

  최적: concurrency = 파티션 수
        또는 concurrency <= 파티션 수 (파티션:Consumer = N:1 가능)
```

### 5. ContainerProperties 핵심 설정

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String>
        kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setConcurrency(3);

    ContainerProperties props = factory.getContainerProperties();
    // AckMode 설정
    props.setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
    // poll() 타임아웃
    props.setPollTimeout(3000);
    // 리밸런싱 리스너
    props.setConsumerRebalanceListener(new ConsumerAwareRebalanceListener() {
        @Override
        public void onPartitionsRevokedBeforeCommit(
                Consumer<?, ?> consumer,
                Collection<TopicPartition> partitions) {
            // 파티션 반납 전 커밋
        }
    });
    // 에러 핸들러
    factory.setCommonErrorHandler(new DefaultErrorHandler(
        new FixedBackOff(1000L, 3)));  // 1초 간격 3회 재시도

    return factory;
}
```

---

## 💻 실전 실험

### 실험 1: KafkaTemplate.send() 동기/비동기 비교

```java
// 비동기 (기본)
kafkaTemplate.send("orders", "order-1", orderJson)
             .whenComplete((result, ex) -> {
                 if (ex != null) {
                     log.error("전송 실패", ex);
                 } else {
                     log.info("전송 완료: offset={}",
                         result.getRecordMetadata().offset());
                 }
             });

// 동기 (블로킹)
try {
    SendResult<String, String> result =
        kafkaTemplate.send("orders", "order-1", orderJson).get();
    log.info("전송 완료: offset={}",
        result.getRecordMetadata().offset());
} catch (ExecutionException e) {
    log.error("전송 실패", e.getCause());
}
```

### 실험 2: AckMode.MANUAL_IMMEDIATE와 At-Least-Once

```java
@KafkaListener(topics = "orders", groupId = "test-group")
public void consume(ConsumerRecord<String, String> record,
                    Acknowledgment ack) {
    log.info("처리 시작: offset={}", record.offset());

    try {
        processOrder(record.value());
        ack.acknowledge();  // 성공 시 커밋
        log.info("처리 완료 및 커밋: offset={}", record.offset());
    } catch (Exception e) {
        log.error("처리 실패: offset={}", record.offset(), e);
        // ack 하지 않음 → 재처리 (에러 핸들러에게 위임)
    }
}
```

### 실험 3: 멀티 컨테이너 팩토리 설정

```java
// 배치 처리용 팩토리
@Bean("batchFactory")
public ConcurrentKafkaListenerContainerFactory<String, String> batchFactory() {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
    factory.setConsumerFactory(consumerFactory());
    factory.setBatchListener(true);  // 배치 모드
    factory.setConcurrency(3);
    factory.getContainerProperties().setAckMode(
        ContainerProperties.AckMode.BATCH);
    return factory;
}

// 실시간 처리용 팩토리
@Bean("realtimeFactory")
public ConcurrentKafkaListenerContainerFactory<String, String> realtimeFactory() {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
    factory.setConsumerFactory(consumerFactory());
    factory.setBatchListener(false);  // 개별 레코드 모드
    factory.setConcurrency(1);
    factory.getContainerProperties().setAckMode(
        ContainerProperties.AckMode.MANUAL_IMMEDIATE);
    return factory;
}

// 사용
@KafkaListener(topics = "orders", containerFactory = "batchFactory")
public void consumeBatch(List<ConsumerRecord<String, String>> records,
                         Acknowledgment ack) {
    records.forEach(r -> processOrder(r.value()));
    ack.acknowledge();
}
```

---

## 📊 성능/비용 비교

### AckMode별 처리량 비교

```
조건: 1 KB 메시지, 파티션 3개, Consumer 3개

  AckMode.BATCH:
    처리량: ~200,000 msg/sec
    브로커 커밋 요청: 배치당 1회
    At-Least-Once 보장

  AckMode.RECORD:
    처리량: ~50,000 msg/sec (-75%)
    브로커 커밋 요청: 레코드당 1회
    세밀한 커밋 제어

  AckMode.MANUAL_IMMEDIATE:
    처리량: RECORD와 유사 또는 낮음
    (ack.acknowledge() 호출 빈도에 따라)
    커밋 타이밍 완전 제어

  AckMode.MANUAL (배치 처리 완료 후 1회):
    처리량: BATCH와 유사
    커밋 타이밍 완전 제어 + 효율적

결론:
  처리량 우선: BATCH
  세밀한 제어: MANUAL_IMMEDIATE
  배치 완료 후 1번 커밋: MANUAL
  특별한 이유 없으면 BATCH 사용
```

---

## ⚖️ 트레이드오프

```
AckMode.BATCH:
  ✅ 처리량 최고, 구현 단순
  ❌ 배치 중 실패 시 전체 배치 재처리 가능 (중복)
  ❌ 세밀한 커밋 제어 불가

AckMode.RECORD:
  ✅ 레코드별 독립 커밋 (장애 격리)
  ❌ 처리량 대폭 감소 (레코드당 브로커 왕복)

AckMode.MANUAL_IMMEDIATE:
  ✅ 처리 완료 시점에 정확한 커밋
  ✅ 에러 처리와 커밋을 분리 제어
  ❌ 코드 복잡도 증가 (Acknowledgment 파라미터)
  ❌ ack.acknowledge() 잊어버리면 무한 재처리

concurrency 높게:
  ✅ 처리 병렬도 향상 (파티션 수 범위 내)
  ❌ 파티션보다 많으면 스레드 낭비
  ❌ 리밸런싱 시 더 많은 Consumer 재조정 오버헤드
```

---

## 📌 핵심 정리

```
Spring Kafka 기초 핵심:

1. KafkaTemplate.send() = 비동기
   CompletableFuture 반환
   동기 확인: .get() 또는 .whenComplete()

2. @KafkaListener = ConcurrentKafkaListenerContainerFactory
   concurrency만큼 KafkaMessageListenerContainer 생성
   각 컨테이너 = 별도 스레드 + KafkaConsumer

3. AckMode:
   BATCH: 배치 처리 후 1회 커밋 (기본, 처리량 우선)
   RECORD: 레코드마다 커밋 (처리량 낮음)
   MANUAL_IMMEDIATE: 명시적 ack.acknowledge() 시 즉시 커밋
   MANUAL: ack.acknowledge() 예약 후 다음 poll() 시 커밋

4. concurrency = 파티션 수에 맞게 설정
   파티션보다 많으면 IDLE Consumer 발생

5. 커스텀 ContainerFactory로 리스너별 설정 분리
   @KafkaListener(containerFactory = "customFactory")
```

---

## 🤔 생각해볼 문제

**Q1. `@KafkaListener` 메서드에서 예외가 발생했는데 `Acknowledgment.acknowledge()`를 호출하면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

예외가 발생한 후 `ack.acknowledge()`를 명시적으로 호출하면 해당 offset이 커밋됩니다. 예외가 발생했어도 커밋은 됩니다. 즉, 처리에 실패한 메시지의 offset이 커밋되어 재처리 기회가 없어집니다.

올바른 패턴은 처리 성공 시에만 `ack.acknowledge()`를 호출하는 것입니다. 예외 발생 시에는 `acknowledge()`를 호출하지 말고, 에러 핸들러(`DefaultErrorHandler`)가 재시도 또는 DLT 전송을 담당하도록 합니다.

`AckMode.MANUAL_IMMEDIATE`에서 에러 핸들러와 수동 ACK를 함께 사용할 때는 에러 핸들러가 `acknowledge()`를 호출하지 않도록 주의합니다. 에러 핸들러가 DLT에 전송하고 `acknowledge()`까지 호출하는 구조가 일반적입니다.

</details>

---

**Q2. `@KafkaListener`에 `topics`와 `topicPattern`을 동시에 지정할 수 있나요?**

<details>
<summary>해설 보기</summary>

아니요, 둘 중 하나만 지정해야 합니다. `topics`, `topicPattern`, `topicPartitions` 중 하나만 사용할 수 있습니다.

`topicPattern`은 정규식으로 매칭되는 모든 토픽을 구독합니다. 예: `topicPattern = "order.*"` → `orders`, `order-events`, `order-completed` 토픽 모두 구독.

새 토픽이 생성되고 패턴에 매칭되면 자동으로 구독 목록에 추가됩니다(리밸런싱 발생). 동적으로 생성되는 토픽을 구독해야 할 때 유용하지만, 의도하지 않은 토픽까지 구독될 수 있어 패턴 설계에 주의가 필요합니다.

</details>

---

**Q3. `DefaultKafkaProducerFactory`는 스레드당 `KafkaProducer` 인스턴스를 따로 생성하나요?**

<details>
<summary>해설 보기</summary>

기본 설정에서는 공유(shared) 인스턴스를 사용합니다. `DefaultKafkaProducerFactory`는 기본적으로 하나의 `KafkaProducer` 인스턴스를 캐시하고 모든 스레드가 공유합니다.

`KafkaProducer`는 내부적으로 스레드 안전(thread-safe)하게 설계되어 있어 여러 스레드에서 동시에 `send()`를 호출해도 안전합니다.

단, 트랜잭션을 사용하는 경우에는 스레드마다 별도의 `KafkaProducer` 인스턴스가 필요합니다. `transactional.id` 설정 시 `DefaultKafkaProducerFactory`가 트랜잭션 스코프(thread-bound) 인스턴스를 관리합니다. `spring.kafka.producer.transaction-id-prefix` 설정 시 인스턴스별 고유 suffix가 붙어서 관리됩니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 에러 처리와 재시도 ➡️](./02-error-handling-retry.md)**

</div>
