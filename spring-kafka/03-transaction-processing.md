# 트랜잭션 처리 — @Transactional과 KafkaTransactionManager

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- DB 트랜잭션(`@Transactional`)과 Kafka 트랜잭션의 경계가 다른 이유는?
- `KafkaTransactionManager`와 `JpaTransactionManager`를 함께 사용할 때 어떤 일이 발생하는가?
- `ChainedKafkaTransactionManager`의 Best Effort 방식이 완전한 원자성을 보장하지 못하는 이유는?
- `transactional.id` 설정이 Spring Kafka에서 어떻게 관리되는가?
- Kafka Consumer에서 처리 결과를 Kafka와 DB에 동시에 저장해야 할 때 어떤 패턴이 안전한가?
- `@TransactionalEventListener`와 Kafka 발행을 함께 쓸 때의 주의사항은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring의 `@Transactional`은 DB 트랜잭션을 관리한다. Kafka 발행은 별도의 네트워크 I/O다. 이 두 가지를 하나의 원자적 단위로 만들려는 시도에서 많은 버그가 발생한다.

"DB 저장 + Kafka 발행을 `@Transactional` 하나로 묶으면 되지 않나?"라고 생각하기 쉽지만, DB 트랜잭션과 Kafka 트랜잭션은 서로 다른 트랜잭션 매니저가 관리하며 두 개를 진정으로 원자적으로 묶을 방법은 없다.

이 한계를 알아야 올바른 설계(Outbox Pattern 등)를 선택할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: @Transactional 안에서 Kafka 발행이 DB 롤백에 함께 롤백된다고 가정

  코드:
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        kafkaTemplate.send("orders", order);  // Kafka 발행
        if (someCondition) throw new RuntimeException(); // DB 롤백
    }

  예상: DB 롤백 시 Kafka 발행도 취소됨
  실제:
    kafkaTemplate.send()는 기본 설정에서 트랜잭션 없이 즉시 전송
    DB rollback 후에도 Kafka 메시지는 이미 발행됨
    → Kafka에는 이벤트 있음, DB에는 주문 없음 → 데이터 불일치

실수 2: KafkaTransactionManager와 JpaTransactionManager를 ChainedTransactionManager로 묶으면 완전한 원자성 달성

  설정:
    @Transactional("chainedTransactionManager")  // DB + Kafka
    public void process(Order order) {
        orderRepository.save(order);        // JPA (DB)
        kafkaTemplate.send("orders", order); // Kafka
    }

  문제:
    DB commit 성공 → Kafka commit 실패 → DB 롤백 불가
    (이미 커밋된 DB 트랜잭션은 되돌릴 수 없음)
    → DB에는 주문, Kafka에는 이벤트 없음

  결론: ChainedTransactionManager = Best Effort 1PC
        완전한 원자성 보장 불가
        → Outbox Pattern이 근본 해결책

실수 3: 동일 transaction-id-prefix를 여러 인스턴스에서 공유

  설정:
    spring.kafka.producer.transaction-id-prefix: order-tx-

  Instance A: transactional.id = order-tx-0
  Instance B: transactional.id = order-tx-0  ← 동일!

  문제:
    Instance B 시작 → 동일 transactional.id → Epoch 증가
    → Instance A의 트랜잭션 강제 abort (Fenced)
    → ProducerFencedException 발생

  올바른 설정:
    Spring Kafka가 자동으로 인스턴스별 suffix 추가
    spring.kafka.producer.transaction-id-prefix: order-tx-
    → 실제 id = order-tx-0, order-tx-1, ... (자동 관리)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
시나리오별 올바른 패턴:

  케이스 1: Kafka → 처리 → Kafka (Streams 스타일)
    KafkaTransactionManager만 사용
    read_committed + 트랜잭션 Producer
    진정한 Exactly-Once 달성 가능

  케이스 2: Kafka → 처리 → DB 저장만 (Kafka 발행 없음)
    JpaTransactionManager만 사용
    @KafkaListener + @Transactional(JPA)
    DB 실패 시 재처리 (At-Least-Once + 멱등 처리)

  케이스 3: DB 저장 + Kafka 발행 원자성 필요
    → Outbox Pattern (Chapter 6 참조)
    DB 트랜잭션 내 Outbox 테이블 기록
    Debezium이 Kafka 발행 담당
    진정한 원자성 달성

  케이스 4: Best Effort (ChainedTransactionManager)
    DB commit 성공 + Kafka commit 성공 → 정상
    DB commit 성공 + Kafka commit 실패 → Kafka 재시도 (중복 가능)
    멱등 Consumer로 중복 처리
    적합: 낮은 정확성 요구, 단순 구현 선호

Spring Kafka 트랜잭션 설정:
  spring:
    kafka:
      producer:
        transaction-id-prefix: order-tx-  # 트랜잭션 활성화
```

---

## 🔬 내부 동작 원리

### 1. Kafka 트랜잭션 Producer in Spring

```
transaction-id-prefix 설정 시 동작:

  DefaultKafkaProducerFactory 초기화:
    transactionIdPrefix = "order-tx-"
    → 트랜잭션 모드 활성화

  트랜잭션 범위 내 send():
    TransactionSynchronizationManager를 통해
    현재 스레드에 바인딩된 Producer 확인
    → 없으면 새 트랜잭션 Producer 생성
       transactional.id = "order-tx-" + threadId 또는 순번

  KafkaTransactionManager.doBegin():
    1. ProducerFactory.createProducer() (트랜잭션 Producer)
    2. producer.initTransactions()
    3. producer.beginTransaction()
    4. 스레드에 바인딩 (ThreadLocal)

  KafkaTransactionManager.doCommit():
    producer.commitTransaction()

  KafkaTransactionManager.doRollback():
    producer.abortTransaction()
```

### 2. @Transactional과 Kafka 발행의 경계

```
DB 트랜잭션 내 Kafka 발행 (transaction-id-prefix 없을 때):

  @Transactional  ← JpaTransactionManager 관리
  public void process(Order order) {
      orderRepository.save(order);      // DB 트랜잭션 내
      kafkaTemplate.send("orders", order); // ← Kafka는 트랜잭션 외!
  }

  Kafka send()는 즉시 브로커에 전송
  DB rollback 시 Kafka 메시지는 이미 전송됨 → 불일치!

@TransactionalEventListener(AFTER_COMMIT)으로 분리:

  @Service
  public class OrderService {
      @Transactional
      public void placeOrder(Order order) {
          orderRepository.save(order);
          eventPublisher.publishEvent(new OrderCreatedEvent(order));
          // 이벤트 발행 (Kafka 직접 전송 아님)
      }
  }

  @Component
  public class OrderEventHandler {
      @TransactionalEventListener(
          phase = TransactionPhase.AFTER_COMMIT)  // DB 커밋 후
      public void onOrderCreated(OrderCreatedEvent event) {
          kafkaTemplate.send("orders", event.getOrder());
          // DB 커밋 성공 후 Kafka 발행
      }
  }

  여전히 문제:
    DB 커밋 후 Kafka 발행 중 네트워크 오류 → Kafka 발행 유실
    → 완전한 원자성 아님, 하지만 DB에 데이터가 있어서 Outbox로 재발행 가능
```

### 3. ChainedKafkaTransactionManager Best Effort

```
ChainedKafkaTransactionManager 동작:

  설정:
    ChainedKafkaTransactionManager chained =
        new ChainedKafkaTransactionManager(
            kafkaTransactionManager, jpaTransactionManager);

  @Transactional("chainedTransactionManager")
  public void process(Order order) {
      orderRepository.save(order);        // JPA
      kafkaTemplate.send("orders", order); // Kafka
  }

  커밋 순서:
    1. Kafka.commitTransaction()  ← 먼저 커밋
    2. JPA.commit()               ← 나중 커밋

    실패 시나리오:
      Kafka commit 성공 → JPA commit 실패
      → JPA rollback 가능 (Kafka는 이미 커밋)
      → DB에 데이터 없음, Kafka에 이벤트 있음

      Kafka commit 실패 → JPA commit 불시도
      → Kafka rollback → JPA rollback
      → 양쪽 모두 없음 (비교적 안전)

  "Best Effort"의 의미:
    첫 번째 커밋이 성공하면 두 번째 실패 시 불일치 발생
    XA 트랜잭션이 아닌 이상 완전한 원자성 불가
    → 실무에서 Outbox Pattern이 권장되는 이유
```

### 4. Consumer에서 DB + Kafka 원자적 처리 (sendOffsetsToTransaction)

```
KafkaMessageListenerContainer + KafkaTransactionManager:

  설정:
    factory.setContainerTransactionManager(kafkaTransactionManager)
    // 각 poll() 처리를 트랜잭션으로 래핑

  처리 흐름:
    KafkaTransactionManager.beginTransaction()
    │
    ├─ Consumer.poll()
    ├─ 처리 로직 (DB 저장 + output 토픽 발행)
    ├─ producer.sendOffsetsToTransaction(consumerOffsets)
    │
    KafkaTransactionManager.commitTransaction()

  이 설정으로:
    Input 읽기 + 처리 + Output 발행 + Offset 커밋 = 원자적
    (Kafka 토픽 간만 원자적, DB는 별도)

  DB + Kafka 동시 원자성:
    ChainedTransactionManager: Best Effort
    Outbox Pattern: 진정한 원자성
    → 비즈니스 중요도에 따라 선택
```

---

## 💻 실전 실험

### 실험 1: Kafka 트랜잭션 Producer 설정

```yaml
# application.yml
spring:
  kafka:
    producer:
      transaction-id-prefix: order-tx-
      acks: all
      properties:
        enable.idempotence: true
```

```java
// 트랜잭션 발행
@Autowired
KafkaTemplate<String, String> kafkaTemplate;

// 방법 1: 선언적 트랜잭션
@Transactional("kafkaTransactionManager")
public void sendTransactional(String topic, String key, String value) {
    kafkaTemplate.send(topic, key, value);
    kafkaTemplate.send(topic + "-audit", key, "audit: " + value);
    // 두 전송이 하나의 트랜잭션 → 모두 성공 또는 모두 abort
}

// 방법 2: 프로그래밍 방식
kafkaTemplate.executeInTransaction(operations -> {
    operations.send("orders", "order-1", orderJson);
    operations.send("order-audit", "order-1", auditJson);
    return true;
});
```

### 실험 2: @TransactionalEventListener + Kafka

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(Order.from(request));
        // DB 커밋 전 이벤트 등록 (실제 발행은 커밋 후)
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order));
        return order;
    }
}

@Component
@RequiredArgsConstructor
public class KafkaEventPublisher {
    private final KafkaTemplate<String, Order> kafkaTemplate;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async  // 비동기로 발행 (메인 트랜잭션 블로킹 방지)
    public void handleOrderCreated(OrderCreatedEvent event) {
        kafkaTemplate.send("orders", event.getOrder().getId().toString(),
                          event.getOrder());
    }
}
```

### 실험 3: ChainedKafkaTransactionManager 설정

```java
@Bean
public ChainedKafkaTransactionManager<String, String>
        chainedTransactionManager(
            KafkaTransactionManager<String, String> kafkaTm,
            JpaTransactionManager jpaTm) {
    return new ChainedKafkaTransactionManager<>(kafkaTm, jpaTm);
}

@Transactional("chainedTransactionManager")
public void processBestEffort(Order order) {
    orderRepository.save(order);
    kafkaTemplate.send("orders", order.getId(), order);
    // Best Effort: DB + Kafka 동시 커밋 시도
    // 완전한 원자성 아님 (문서 참고)
}
```

---

## 📊 성능/비용 비교

### 트랜잭션 방식별 처리량 영향

```
방식별 1 KB 메시지 처리량:

  KafkaTemplate 일반 (트랜잭션 없음):
    처리량: ~300,000 msg/sec
    지연: 최소

  KafkaTransactionManager 단독:
    처리량: ~200,000 msg/sec (-33%)
    지연: Transaction Coordinator 왕복 추가

  ChainedTransactionManager (DB + Kafka):
    처리량: ~150,000 msg/sec (-50%)
    지연: DB commit + Kafka commit 순서 처리
    원자성: Best Effort (불완전)

  Outbox Pattern + Debezium:
    처리량: ~200,000 msg/sec (단일 DB 트랜잭션)
    지연: DB commit + Debezium 전파 (수백 ms)
    원자성: 완전 (DB 트랜잭션 활용)
```

---

## ⚖️ 트레이드오프

```
KafkaTransactionManager 단독:
  ✅ Kafka 토픽 간 원자적 발행
  ❌ DB 트랜잭션과 별도 (DB 실패 시 Kafka 발행 취소 안 됨)

ChainedKafkaTransactionManager:
  ✅ DB + Kafka 함께 시도 (간단한 설정)
  ❌ 완전한 원자성 보장 안 됨 (Best Effort)
  ❌ 첫 번째 커밋 성공 후 두 번째 실패 시 불일치

@TransactionalEventListener:
  ✅ DB 커밋 성공 후 Kafka 발행 (DB 롤백 시 발행 안 됨)
  ❌ Kafka 발행 실패 시 유실 (재시도 로직 필요)
  ❌ 발행 실패를 DB에서 추적하기 어려움

Outbox Pattern:
  ✅ 진정한 원자성 (DB 트랜잭션 활용)
  ❌ Debezium 인프라 필요
  ❌ 구현 복잡도 증가
  → 데이터 정확성이 핵심인 비즈니스에 권장
```

---

## 📌 핵심 정리

```
트랜잭션 처리 핵심:

1. DB 트랜잭션과 Kafka 트랜잭션은 별개
   @Transactional(JPA)로 Kafka 발행 원자성 보장 불가

2. ChainedKafkaTransactionManager = Best Effort 1PC
   완전한 원자성 아님
   첫 번째 커밋 성공 후 두 번째 실패 시 불일치 가능

3. Kafka 토픽 간 원자성:
   KafkaTransactionManager + sendOffsetsToTransaction
   Kafka 레벨에서만 완전한 원자성

4. DB + Kafka 진정한 원자성:
   Outbox Pattern + Debezium CDC
   DB 트랜잭션의 원자성을 Kafka 발행에 활용

5. transaction-id-prefix:
   Spring Kafka가 인스턴스별 고유 suffix 자동 관리
   수동으로 suffix 관리 불필요
```

---

## 🤔 생각해볼 문제

**Q1. `@Transactional`과 `KafkaTransactionManager`를 함께 쓸 때 트랜잭션 전파(Propagation)는 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

Spring의 트랜잭션 전파는 동일한 트랜잭션 매니저 내에서만 동작합니다. `JpaTransactionManager`로 시작한 트랜잭션 내에서 `KafkaTransactionManager`가 관리하는 코드를 호출하면, Kafka 트랜잭션은 별도로 시작됩니다(중첩이 아닌 독립).

`ChainedKafkaTransactionManager`를 사용하면 두 트랜잭션 매니저를 체인으로 묶어서 시작/커밋/롤백을 순서대로 처리합니다. 하지만 내부적으로는 여전히 두 개의 독립적인 트랜잭션입니다.

이 차이를 이해하려면 `AbstractPlatformTransactionManager`가 `TransactionSynchronizationManager`를 통해 스레드에 트랜잭션을 바인딩하는 방식을 이해해야 합니다. 각 트랜잭션 매니저는 자신의 리소스(Connection, KafkaProducer)를 독립적으로 관리합니다.

</details>

---

**Q2. `kafkaTemplate.executeInTransaction()` 안에서 예외가 발생하면 어떻게 처리되나요?**

<details>
<summary>해설 보기</summary>

예외가 발생하면 트랜잭션이 abort됩니다. `executeInTransaction()`은 내부적으로 `try-catch`로 예외를 잡아서 `producer.abortTransaction()`을 호출합니다. 람다 내에서 발생한 예외는 래핑되어 호출자에게 전파됩니다.

```java
kafkaTemplate.executeInTransaction(ops -> {
    ops.send("orders", key, value);    // 성공
    ops.send("inventory", key, value); // 예외 발생
    return null;
});
// → 두 번째 send에서 예외 → abortTransaction()
// → orders 토픽의 첫 번째 send도 abort 됨
// → read_committed Consumer는 두 메시지 모두 읽지 못함
```

이것이 멀티 토픽 원자적 발행을 `executeInTransaction()`으로 구현하는 방법입니다. 모두 성공하거나 모두 abort됩니다.

</details>

---

**Q3. Spring Boot 자동 설정에서 `transaction-id-prefix`를 설정하면 모든 `KafkaTemplate`이 트랜잭션 모드가 되나요?**

<details>
<summary>해설 보기</summary>

`spring.kafka.producer.transaction-id-prefix`를 설정하면 Spring Boot 자동 설정이 생성하는 `KafkaTemplate` 빈이 트랜잭션 모드가 됩니다. 

트랜잭션 모드의 `KafkaTemplate`은 트랜잭션 컨텍스트 외부에서 `send()`를 호출하면 `IllegalStateException`이 발생합니다. 명시적인 트랜잭션(`@Transactional("kafkaTransactionManager")` 또는 `executeInTransaction()`) 내에서만 사용해야 합니다.

비트랜잭션 `KafkaTemplate`도 필요하다면 별도 빈으로 생성합니다:

```java
@Bean("nonTxKafkaTemplate")
public KafkaTemplate<String, String> nonTxTemplate(
        DefaultKafkaProducerFactory<String, String> pf) {
    return new KafkaTemplate<>(pf);  // transaction-id-prefix 없는 팩토리 사용
}
```

</details>

---

<div align="center">

**[⬅️ 이전: 에러 처리와 재시도](./02-error-handling-retry.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Cloud Stream ➡️](./04-spring-cloud-stream.md)**

</div>
