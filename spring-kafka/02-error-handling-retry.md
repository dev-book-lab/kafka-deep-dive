# 에러 처리와 재시도 — DefaultErrorHandler와 DLT

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `DefaultErrorHandler`(Spring Kafka 2.8+)가 `BackOff` 정책으로 재시도하는 원리는?
- `SeekToCurrentErrorHandler`(구버전)와 `DefaultErrorHandler`의 차이는?
- DLT(Dead Letter Topic)로 메시지가 라우팅되는 조건은?
- DLT 메시지를 재처리하는 두 가지 전략은?
- `@RetryableTopic`이 일반 DLT 패턴과 어떻게 다른가?
- 재시도 불가 예외(non-retryable)와 재시도 가능 예외를 어떻게 구분하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka Consumer 처리 중 예외가 발생했을 때 적절한 에러 처리가 없으면 두 가지 극단 중 하나가 된다. 재시도 없이 메시지를 건너뛰면 데이터 유실, 무한 재시도하면 파티션 전체가 멈춘다.

`DefaultErrorHandler`의 BackOff 정책과 DLT를 적절히 설정하면:
- 일시적 장애(DB 연결 오류): 재시도로 자연 복구
- 영구적 장애(잘못된 데이터 포맷): DLT로 격리 후 수동 처리
- 파티션 처리 중단 없이 다음 메시지 계속 처리

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 에러 핸들러 없이 운영 (무한 재처리)

  기본 동작 (DefaultErrorHandler 없음):
    예외 발생 → seek to current → 동일 메시지 재처리
    재시도 횟수 무제한 → 파티션 처리 중단!
    
  시나리오:
    offset 100 메시지 처리 중 DB 오류 (영구적)
    → offset 100 재처리 → DB 오류 → offset 100 재처리...
    → 파티션 전체가 offset 100에서 멈춤
    → Consumer Lag 무한 증가

  올바른 설정:
    DefaultErrorHandler 설정 → 최대 N회 재시도 후 DLT 전송

실수 2: DLT 없이 재시도 소진 후 skip

  설정:
    new DefaultErrorHandler(new FixedBackOff(1000L, 3))
    // DLT 없음 → 3회 실패 후 메시지 폐기

  문제:
    처리 실패한 메시지가 영구 유실
    나중에 "왜 이 주문이 처리 안 됐지?" → 원인 조사 불가
  
  올바른 설정:
    DeadLetterPublishingRecoverer로 DLT 전송
    DLT 메시지: 실패 원인, 원래 토픽, offset 정보 헤더 포함

실수 3: 모든 예외를 재시도

  설정: new FixedBackOff(1000L, Integer.MAX_VALUE)

  문제:
    JsonParseException (잘못된 JSON) → 재시도해도 항상 실패
    NullPointerException (버그) → 재시도해도 항상 실패
    무한 재시도 → 파티션 멈춤

  올바른 설정:
    // 재시도 불가 예외 지정
    DefaultErrorHandler handler = new DefaultErrorHandler(
        recoverer, new FixedBackOff(1000L, 3));
    handler.addNotRetryableExceptions(
        JsonParseException.class,
        NullPointerException.class);
    // → 즉시 DLT 전송 (재시도 없음)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
권장 에러 핸들러 설정:

  @Bean
  public DefaultErrorHandler errorHandler(
          KafkaTemplate<String, String> kafkaTemplate) {

      // DLT로 실패 메시지 전송
      DeadLetterPublishingRecoverer recoverer =
          new DeadLetterPublishingRecoverer(kafkaTemplate,
              (record, ex) -> new TopicPartition(
                  record.topic() + ".DLT",  // orders → orders.DLT
                  record.partition()));       // 동일 파티션 유지

      // 지수 백오프: 1초→2초→4초, 최대 3회
      ExponentialBackOff backOff =
          new ExponentialBackOff(1000L, 2.0);
      backOff.setMaxAttempts(3);

      DefaultErrorHandler handler =
          new DefaultErrorHandler(recoverer, backOff);

      // 재시도 불가 예외 (즉시 DLT)
      handler.addNotRetryableExceptions(
          JsonParseException.class,
          DeserializationException.class);

      return handler;
  }

  // ContainerFactory에 등록
  factory.setCommonErrorHandler(errorHandler(kafkaTemplate));

DLT 재처리 전략:
  전략 A: 수동 재처리
    orders.DLT 토픽의 메시지를 사람이 확인 후
    원인 수정 → orders 토픽으로 재발행

  전략 B: @RetryableTopic (자동 재시도 토픽)
    @RetryableTopic(
        backoff = @Backoff(delay = 1000, multiplier = 2),
        attempts = "4",
        topicSuffixingStrategy = SUFFIX_WITH_INDEX_VALUE
    )
    @KafkaListener(topics = "orders")
    → orders → orders-retry-0 → orders-retry-1 → orders.DLT
    자동으로 재시도 토픽 생성 및 처리
```

---

## 🔬 내부 동작 원리

### 1. DefaultErrorHandler 재시도 메커니즘

```
DefaultErrorHandler 처리 흐름:

  @KafkaListener 메서드에서 예외 발생
       │
       ▼
  DefaultErrorHandler.handleRemaining() 호출
       │
       ├─ 재시도 불가 예외? (addNotRetryableExceptions)
       │    YES → 즉시 recoverer.accept() (DLT 전송)
       │    NO  → 다음 단계
       │
       ├─ BackOff 소진됨?
       │    YES → recoverer.accept() (DLT 전송)
       │    NO  → backOff.nextBackOff() 대기 후 seek
       │          → 동일 오프셋부터 재처리 (재시도)
       │
       └─ Seek: consumer.seek(partition, failedOffset)
                → 다음 poll() 시 failedOffset부터 다시 읽기

  BackOff 정책:
    FixedBackOff(1000L, 3):
      1000ms 대기, 최대 3회 재시도 (총 4회 처리 시도)

    ExponentialBackOff(1000L, 2.0):
      1000ms → 2000ms → 4000ms → 소진
      maxAttempts=3이면 3회 재시도

  재시도 시 같은 파티션의 이후 메시지는:
    seek 후 재처리하는 동안 대기
    → 실패한 메시지가 재처리되는 동안 파티션 처리 일시 중단
    → 단, 다른 파티션은 계속 처리
```

### 2. DeadLetterPublishingRecoverer

```
DLT 전송 시 포함되는 헤더:

  kafka_dlt-original-topic:     원래 토픽 이름
  kafka_dlt-original-partition: 원래 파티션
  kafka_dlt-original-offset:    원래 offset
  kafka_dlt-original-timestamp: 원래 타임스탬프
  kafka_dlt-exception-fqcn:     예외 클래스 전체 이름
  kafka_dlt-exception-message:  예외 메시지
  kafka_dlt-exception-stacktrace: 스택 트레이스 (설정에 따라)

  DLT 토픽 기본 이름 규칙:
    원래 토픽 + ".DLT"
    예: orders → orders.DLT

  DeadLetterPublishingRecoverer 커스텀:
    new DeadLetterPublishingRecoverer(kafkaTemplate,
        (record, ex) -> {
            if (ex instanceof BusinessException) {
                return new TopicPartition("business-errors", -1);
            }
            return new TopicPartition(record.topic() + ".DLT", record.partition());
        });
    // 예외 유형에 따라 다른 토픽으로 라우팅
```

### 3. @RetryableTopic 자동 재시도 토픽

```
@RetryableTopic 동작 원리:

  설정:
    @RetryableTopic(
        backoff = @Backoff(delay = 1000, multiplier = 2),
        attempts = "4"  // 1 원본 + 3 재시도
    )
    @KafkaListener(topics = "orders")
    public void consume(Order order) { ... }

  자동 생성 토픽:
    orders             ← 원본
    orders-retry-0     ← 1차 재시도 (1초 후)
    orders-retry-1     ← 2차 재시도 (2초 후)
    orders-retry-2     ← 3차 재시도 (4초 후)
    orders-dlt         ← 최종 DLT

  흐름:
    orders 처리 실패 → orders-retry-0에 발행 (1초 후 재처리)
    orders-retry-0 처리 실패 → orders-retry-1에 발행 (2초 후 재처리)
    orders-retry-1 처리 실패 → orders-retry-2에 발행 (4초 후 재처리)
    orders-retry-2 처리 실패 → orders-dlt에 발행 (최종)

  장점:
    각 재시도 단계가 별도 토픽 → 재처리 중 원본 토픽 계속 소비 가능
    재시도 지연이 토픽 기반 → 브로커 스케줄링

  단점:
    토픽 수 증가 (시도 수만큼)
    메시지 순서 역전 가능 (재시도 메시지가 원본 처리 흐름에서 분리)
```

### 4. 재시도 가능 vs 재시도 불가 예외 분류

```
재시도가 의미 있는 예외 (일시적 오류):
  ConnectException       - DB/외부 서비스 일시 연결 불가
  TransientDataException - 일시적 DB 오류
  OptimisticLockException - 낙관적 잠금 충돌
  (재시도하면 성공할 가능성 있음)

재시도가 의미 없는 예외 (영구적 오류):
  JsonParseException        - 메시지 포맷 오류 (재시도해도 동일 오류)
  DeserializationException  - 역직렬화 실패
  ConstraintViolationException - DB 유니크 제약 위반
  BusinessRuleException     - 비즈니스 규칙 위반 (재시도 의미 없음)
  NullPointerException      - 버그 (재시도해도 동일 오류)

Spring Kafka 설정:
  handler.addNotRetryableExceptions(
      JsonParseException.class,
      DeserializationException.class,
      IllegalArgumentException.class);
  
  // 예외 계층 구조 처리:
  handler.addNotRetryableExceptions(RuntimeException.class);
  handler.removeNotRetryableException(ConnectException.class);
  // RuntimeException은 재시도 불가, 그 중 ConnectException만 재시도 가능
```

---

## 💻 실전 실험

### 실험 1: DefaultErrorHandler with DLT 설정

```java
@Configuration
public class KafkaErrorConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            kafkaListenerContainerFactory(
                    ConsumerFactory<String, String> cf,
                    KafkaTemplate<String, String> kt) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.setConsumerFactory(cf);

        var recoverer = new DeadLetterPublishingRecoverer(kt,
            (r, e) -> new TopicPartition(r.topic() + ".DLT", r.partition()));

        var backOff = new ExponentialBackOff(1_000L, 2.0);
        backOff.setMaxAttempts(3);

        var handler = new DefaultErrorHandler(recoverer, backOff);
        handler.addNotRetryableExceptions(JsonParseException.class);

        factory.setCommonErrorHandler(handler);
        return factory;
    }

    // DLT 소비자
    @KafkaListener(topics = "orders.DLT", groupId = "dlt-monitor")
    public void consumeDlt(ConsumerRecord<String, String> record) {
        String originalTopic = new String(
            record.headers().lastHeader("kafka_dlt-original-topic").value());
        String exceptionMsg = new String(
            record.headers().lastHeader("kafka_dlt-exception-message").value());
        log.error("DLT 수신: originalTopic={}, error={}", originalTopic, exceptionMsg);
        // 알림 발송, 모니터링 기록 등
    }
}
```

### 실험 2: @RetryableTopic 설정

```java
@Component
public class OrderConsumer {

    @RetryableTopic(
        backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 30000),
        attempts = "4",
        autoCreateTopics = "true",
        dltTopicSuffix = ".DLT",
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
        include = {ConnectException.class, TransientDataException.class},
        exclude = {JsonParseException.class}
    )
    @KafkaListener(topics = "orders")
    public void consume(Order order) {
        orderService.process(order);
    }

    @DltHandler
    public void handleDlt(Order order, ConsumerRecord<?, ?> record) {
        log.error("최종 DLT 도달: order={}, partition={}, offset={}",
            order.getId(), record.partition(), record.offset());
        alertService.notifyDltArrival(order);
    }
}
```

### 실험 3: DLT 메시지 수동 재처리

```bash
# DLT 토픽 내용 확인
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic orders.DLT \
  --from-beginning \
  --property print.key=true \
  --property print.headers=true \
  --max-messages 10

# 원인 파악 후 수정된 메시지를 원본 토픽으로 재발행
# 또는 DLT Consumer 그룹의 offset을 earliest로 리셋 후 재처리
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group dlt-reprocess \
  --topic orders.DLT \
  --reset-offsets --to-earliest --execute
```

---

## 📊 성능/비용 비교

### BackOff 정책별 재시도 시간

```
FixedBackOff(1000L, 3) - 고정 간격:
  1회: 즉시 실패 → 1초 대기
  2회: 1초 후 재시도 → 실패 → 1초 대기
  3회: 1초 후 재시도 → 실패 → 1초 대기
  4회: 1초 후 재시도 → 실패 → DLT 전송
  총 재처리 시간: ~3초
  처리 중단: ~3초

ExponentialBackOff(1000L, 2.0) maxAttempts=3:
  1회: 실패 → 1초 대기
  2회: 실패 → 2초 대기
  3회: 실패 → 4초 대기
  4회: 실패 → DLT 전송
  총 재처리 시간: ~7초
  처리 중단: ~7초

ExponentialBackOff + 외부 서비스 장애 (1분 지속):
  재시도 소진 후 DLT → 외부 서비스 복구 후 DLT 재처리 필요
  vs
  FixedBackOff(10000L, 6): 10초 × 6회 = 1분 → 자연 복구 가능
  선택: 외부 서비스 장애 지속 시간 예측 기반으로 BackOff 설정
```

---

## ⚖️ 트레이드오프

```
DefaultErrorHandler + DLT:
  ✅ 재시도 횟수 제한 → 파티션 처리 무한 중단 방지
  ✅ DLT로 실패 메시지 보존 → 데이터 유실 없음
  ✅ 재시도 중 다른 파티션 계속 처리
  ❌ DLT 재처리 운영 오버헤드
  ❌ 재시도 기간 동안 파티션 처리 지연

@RetryableTopic:
  ✅ 재시도 중 원본 토픽 계속 소비 (처리 중단 없음)
  ✅ 재시도 지연을 Kafka 스케줄링으로 처리
  ❌ 재시도 토픽 수 증가
  ❌ 메시지 순서 역전 가능

즉시 DLT (재시도 없음):
  ✅ 파티션 처리 지연 없음
  ❌ 일시적 오류도 재시도 기회 없음
  → 재시도 불가 예외에만 적용
```

---

## 📌 핵심 정리

```
에러 처리와 재시도 핵심:

1. DefaultErrorHandler = 재시도 + DLT의 통합 에러 핸들러
   BackOff: 재시도 간격/횟수 정책
   DeadLetterPublishingRecoverer: DLT 전송

2. 재시도 불가 예외 지정 필수
   addNotRetryableExceptions()로 영구 오류 즉시 DLT 격리

3. DLT 전략:
   수동 재처리: 운영자가 원인 파악 후 재발행
   @RetryableTopic: 자동 단계별 재시도 토픽

4. BackOff 선택:
   일시적 오류(DB 연결): ExponentialBackOff (빠른 재시도 후 간격 증가)
   예측 가능한 장애: FixedBackOff (예상 복구 시간 기반)

5. DLT 헤더:
   원본 토픽, 파티션, offset, 예외 정보 자동 포함
   → 원인 추적과 재처리에 활용
```

---

## 🤔 생각해볼 문제

**Q1. 재시도 중 동일 메시지를 N번 처리했는데 N번째에 성공했습니다. DLT에는 메시지가 가나요?**

<details>
<summary>해설 보기</summary>

재시도 중 성공하면 DLT에 가지 않습니다. `DefaultErrorHandler`는 재시도 횟수(`BackOff`)를 소진한 경우에만 `recoverer.accept()`(DLT 전송)를 호출합니다. 재시도 중 성공하면 정상 처리 완료로 간주하고 offset이 커밋됩니다.

재시도 1회: 실패 → 1초 대기
재시도 2회: 실패 → 2초 대기
재시도 3회: 성공 → DLT 전송 없음, offset 커밋

이것이 일시적 오류(DB 연결 순간 끊김, 네트워크 히컵 등)에 재시도가 효과적인 이유입니다. 일시적 오류는 재시도하면 성공하므로 DLT 없이 처리됩니다.

</details>

---

**Q2. `DefaultErrorHandler`의 재시도는 동기적으로 동작하나요? 재시도 중 해당 파티션이 멈추나요?**

<details>
<summary>해설 보기</summary>

재시도 중 해당 파티션은 처리가 멈추지만 다른 파티션은 계속 처리됩니다.

`DefaultErrorHandler`가 `BackOff` 대기 중에는 `ListenerConsumer`가 대기 상태입니다. 이 컨테이너가 담당하는 파티션들(concurrency=1이면 여러 파티션)은 처리가 중단됩니다. 단, 다른 `KafkaMessageListenerContainer`(다른 concurrency 스레드)가 담당하는 파티션은 영향받지 않습니다.

`@RetryableTopic`이 이 문제를 해결하는 이유가 여기에 있습니다. 실패한 메시지를 별도 재시도 토픽으로 보내고 원본 토픽은 계속 처리합니다. 재시도 토픽의 메시지는 지정 시간 후 별도로 처리됩니다.

</details>

---

**Q3. DLT 메시지를 재처리할 때 같은 에러 핸들러가 또 DLT로 보내는 무한 루프가 생기지 않나요?**

<details>
<summary>해설 보기</summary>

`@RetryableTopic`에서 `@DltHandler`를 사용하면 DLT 처리 중 예외가 발생해도 추가 DLT 전송이 발생하지 않습니다. `@DltHandler` 어노테이션이 붙은 메서드는 자동으로 에러 핸들러의 DLT 재전송에서 제외됩니다.

일반 `@KafkaListener`로 DLT를 소비할 때는 해당 리스너의 `containerFactory`에 에러 핸들러를 별도로 설정해야 합니다. DLT Consumer용 팩토리는 `DefaultErrorHandler`를 설정하되 `recoverer`를 단순 로깅으로만 처리하거나, `addNotRetryableExceptions(Exception.class)`로 모든 예외를 재시도 불가로 설정해서 무한 루프를 방지합니다.

```java
@KafkaListener(topics = "orders.DLT",
               containerFactory = "dltContainerFactory")
public void consumeDlt(ConsumerRecord<String, String> record) {
    log.error("DLT 처리 중 예외 발생해도 추가 DLT 전송 없음");
}

@Bean("dltContainerFactory")
public ConcurrentKafkaListenerContainerFactory<?,?> dltFactory(...) {
    // 모든 예외를 재시도 불가로 설정 (무한 루프 방지)
    var handler = new DefaultErrorHandler(new FixedBackOff(0L, 0));
    handler.addNotRetryableExceptions(Exception.class);
    factory.setCommonErrorHandler(handler);
    return factory;
}
```

</details>

---

<div align="center">

**[⬅️ 이전: Spring Kafka 기초](./01-spring-kafka-basics.md)** | **[홈으로 🏠](../README.md)** | **[다음: 트랜잭션 처리 ➡️](./03-transaction-processing.md)**

</div>
