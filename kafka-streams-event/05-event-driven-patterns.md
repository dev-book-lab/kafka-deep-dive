# 이벤트 기반 아키텍처 패턴 — Outbox Pattern

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- DB 트랜잭션 커밋 후 Kafka 발행 사이에 장애가 발생할 때 어떤 원자성 문제가 생기는가?
- Outbox Pattern이 이 원자성 문제를 해결하는 구조는?
- Debezium CDC(Change Data Capture)가 Outbox 테이블 변경을 Kafka로 전달하는 방식은?
- Saga Pattern의 Choreography와 Orchestration 방식은 어떻게 다른가?
- Outbox Pattern의 한계와 주의사항은 무엇인가?
- 이벤트 기반 마이크로서비스에서 보상 트랜잭션(Compensating Transaction)은 어떻게 구현하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이크로서비스에서 "주문 DB 저장 후 Kafka에 주문 완료 이벤트 발행"을 구현할 때 가장 흔한 실수는 두 작업을 순차적으로 실행하는 것이다.

```java
orderRepository.save(order);       // DB 저장 성공
kafkaTemplate.send("orders", event); // 여기서 장애 발생!
// → DB에는 주문이 있지만 Kafka에는 이벤트 없음 → 재고 감소 안 됨
```

반대도 문제다:
```java
kafkaTemplate.send("orders", event); // Kafka 발행 성공
orderRepository.save(order);         // 여기서 DB 오류
// → Kafka에는 이벤트 있지만 DB에는 주문 없음 → 데이터 불일치
```

이것이 **분산 시스템의 원자성 문제**다. Outbox Pattern은 이 문제를 DB의 트랜잭션 보장을 활용해서 해결한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: DB 저장 후 Kafka 발행을 별도 연산으로 처리

  코드:
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);      // 트랜잭션 내
        // 트랜잭션 커밋 후 Kafka 발행 (트랜잭션 밖)
    }
    
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onOrderCreated(OrderCreatedEvent event) {
        kafkaTemplate.send("orders", event);  // 여기서 오류 가능
    }

  문제:
    DB 커밋 성공 → Kafka 발행 실패 → 이벤트 유실
    재시도 로직 없으면 영구 유실

실수 2: @KafkaTransactionManager + DB 트랜잭션 혼용으로 완전한 원자성을 기대

  시도:
    @Transactional("chainedTransactionManager")  // DB + Kafka
    public void process() {
        orderRepository.save(order);   // DB 트랜잭션
        kafkaTemplate.send("orders");  // Kafka 트랜잭션
    }

  문제:
    DB 커밋 성공 + Kafka 커밋 실패 → 데이터 불일치
    X/Open XA 프로토콜이 아닌 이상 완전한 원자성 보장 불가

실수 3: Outbox 테이블 읽기를 폴링(Polling)으로 구현

  구현:
    @Scheduled(fixedDelay = 1000)
    public void publishOutboxEvents() {
        List<OutboxEvent> events = outboxRepository.findByPublished(false);
        for (OutboxEvent event : events) {
            kafkaTemplate.send(...);
            event.setPublished(true);
            outboxRepository.save(event);
        }
    }

  문제:
    kafkaTemplate.send() 성공 후 outboxRepository.save() 실패
    → 이벤트가 중복 발행될 수 있음
    → 폴링 주기 동안 지연 발생
    더 나은 방법: Debezium CDC로 Outbox 변경 즉시 감지
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Outbox Pattern with Debezium CDC:

  아키텍처:
    주문 서비스 → DB 트랜잭션(orders + outbox) → Debezium → Kafka

  DB 스키마:
    CREATE TABLE outbox_events (
        id UUID PRIMARY KEY,
        aggregate_type VARCHAR(255) NOT NULL,  -- "Order"
        aggregate_id VARCHAR(255) NOT NULL,    -- orderId
        event_type VARCHAR(255) NOT NULL,      -- "OrderCreated"
        payload JSONB NOT NULL,
        created_at TIMESTAMP DEFAULT NOW(),
        processed BOOLEAN DEFAULT FALSE
    );

  비즈니스 로직:
    @Transactional
    public void placeOrder(Order order) {
        // 1. 비즈니스 데이터 저장
        orderRepository.save(order);
        
        // 2. Outbox에 이벤트 기록 (같은 트랜잭션)
        outboxRepository.save(OutboxEvent.builder()
            .aggregateType("Order")
            .aggregateId(order.getId())
            .eventType("OrderCreated")
            .payload(toJson(order))
            .build());
        // 트랜잭션 커밋: 두 테이블 동시 커밋 (원자적)
    }

Debezium 설정:
  Debezium은 DB의 WAL(Write-Ahead Log)을 모니터링
  outbox_events 테이블에 새 레코드 INSERT 감지
  → Kafka의 outbox.events 토픽에 자동 발행
  → 별도 코드 없이 DB 변경 → Kafka 자동 전달
```

---

## 🔬 내부 동작 원리

### 1. 분산 트랜잭션 원자성 문제

```
마이크로서비스 주문 처리 흐름:

  주문 서비스:
    1. DB에 Order 저장
    2. Kafka에 OrderCreated 이벤트 발행
    3. 재고 서비스가 이벤트 수신 → 재고 감소

  원자성이 없을 때 장애 시나리오:

  시나리오 A: DB 저장 성공, Kafka 발행 실패
    DB: Order {id=1, status=CREATED} 있음
    Kafka: OrderCreated 이벤트 없음
    재고 서비스: 이벤트 못 받음 → 재고 감소 안 됨
    결과: 주문은 있지만 재고는 줄지 않음 → 데이터 불일치

  시나리오 B: Kafka 발행 성공, DB 저장 실패
    Kafka: OrderCreated 이벤트 있음
    재고 서비스: 이벤트 받아서 재고 감소
    DB: Order 없음 (롤백됨)
    결과: 재고는 줄었는데 주문은 없음 → 유령 재고 감소

  이것이 분산 트랜잭션 문제:
    DB 트랜잭션과 Kafka 발행을 원자적으로 묶을 수 없음
```

### 2. Outbox Pattern 구조

```
핵심 아이디어: "DB 트랜잭션 내에서 이벤트를 임시 저장하고, DB 변경을 감지해서 Kafka 발행"

  orders 테이블               outbox_events 테이블
  ┌─────────────────────┐     ┌─────────────────────────────────┐
  │ id     │ status     │     │ id │ aggregate_id │ event_type  │
  │ 1      │ CREATED    │     │ x1 │ 1            │ OrderCreated│
  │ 2      │ PAID       │     │ x2 │ 2            │ OrderPaid   │
  └─────────────────────┘     └─────────────────────────────────┘
                                        │
                              [같은 DB 트랜잭션으로 기록]
                                        │
                              Debezium (CDC 커넥터)
                                        │ WAL 모니터링
                                        ▼
                              Kafka Topic: "outbox.events"
                                        │
                              Consumer: 재고 서비스
                                        │
                              재고 감소 처리

  원자성 보장:
    orders + outbox_events = 하나의 DB 트랜잭션
    DB 커밋 → 두 테이블 동시 반영 (원자적)
    DB 롤백 → 두 테이블 모두 롤백 (원자적)
    → orders가 있으면 outbox_events도 있음 (불일치 없음)
```

### 3. Debezium CDC 동작 방식

```
Debezium = DB의 변경 이력(WAL)을 Kafka로 스트리밍하는 CDC 도구

  지원 DB: MySQL(binlog), PostgreSQL(replication slot), MongoDB(oplog)

  PostgreSQL WAL 기반 CDC:

    1. Debezium이 PostgreSQL에 replication slot 생성
    2. DB에 INSERT/UPDATE/DELETE 발생
    3. PostgreSQL이 WAL(Write-Ahead Log)에 변경 기록
    4. Debezium이 WAL에서 변경 이벤트 읽기
    5. Debezium → Kafka 토픽에 변경 이벤트 발행

  Outbox 테이블 INSERT 감지:
    DB INSERT: {id: x1, aggregate_id: 1, event_type: OrderCreated, ...}
    WAL: INSERT 이벤트 기록
    Debezium: WAL 읽기 → Kafka 발행
    Kafka 메시지: {before: null, after: {id:x1, ...}, op: "c"}

  Debezium 설정 (Docker):
    {
      "name": "outbox-connector",
      "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "postgres",
        "database.dbname": "orderdb",
        "table.include.list": "public.outbox_events",
        "transforms": "outbox",
        "transforms.outbox.type":
          "io.debezium.transforms.outbox.EventRouter",
        "transforms.outbox.route.by.field": "aggregate_type"
      }
    }
```

### 4. Saga Pattern: Choreography vs Orchestration

```
분산 트랜잭션의 대안 = Saga: 로컬 트랜잭션의 체인 + 보상 트랜잭션

  Choreography (이벤트 기반 조율):
    각 서비스가 이벤트를 구독하고 독립적으로 반응

    주문 서비스 → OrderCreated
    재고 서비스 ← OrderCreated → 재고 감소 → InventoryReserved
    결제 서비스 ← InventoryReserved → 결제 처리 → PaymentCompleted
    배송 서비스 ← PaymentCompleted → 배송 준비

    장점:
      서비스 간 직접 의존 없음 (느슨한 결합)
      각 서비스가 독립적으로 확장
    단점:
      전체 흐름 파악 어려움 (각 서비스의 로직 분산)
      디버깅 어려움 (이벤트 체인 추적 필요)

  Orchestration (중앙 조율):
    오케스트레이터(Saga Manager)가 각 서비스에 명령

    Saga Manager → Reserve(inventory)
    재고 서비스 → Reserved → Saga Manager
    Saga Manager → Charge(payment)
    결제 서비스 → Charged → Saga Manager
    Saga Manager → Ship(shipping)
    배송 서비스 → Shipped

    장점:
      전체 흐름이 한 곳에 집중 → 파악 용이
      복잡한 분기 처리 쉬움
    단점:
      Saga Manager 자체가 단일 장애점 가능
      Saga Manager와 각 서비스 간 결합

보상 트랜잭션:
  결제 실패 시 이미 완료된 재고 예약 취소 필요

  Choreography:
    PaymentFailed 이벤트
    재고 서비스 ← PaymentFailed → 재고 예약 취소 (보상)

  Orchestration:
    Saga Manager가 직접 Cancel(inventory) 명령
```

### 5. Outbox Pattern 주의사항

```
멱등 Consumer 필요:
  Debezium이 네트워크 오류로 동일 WAL 이벤트를 중복 발행 가능
  → Consumer는 idempotent key (outbox_events.id)로 중복 처리 방지

  처리 전:
    SELECT COUNT(*) FROM processed WHERE outbox_id = ?
  처리 후:
    INSERT INTO processed (outbox_id) VALUES (?)

Outbox 테이블 정리:
  processed=true인 오래된 이벤트 주기적 삭제 필요
  무한 증가 방지:
    DELETE FROM outbox_events
    WHERE processed = true AND created_at < NOW() - INTERVAL '7 days'

이벤트 순서 보장:
  Outbox 이벤트를 aggregate_id를 Kafka 파티션 키로 사용
  → 동일 aggregate의 이벤트가 동일 파티션 → 순서 보장

  Debezium 설정:
    transforms.outbox.route.topic.replacement=${routedByValue}
    message.key.columns: public.outbox_events:aggregate_id

지연 이슈:
  Debezium이 WAL을 폴링하는 주기 → 수백 ms ~ 수 초 지연
  실시간성이 매우 중요하면 직접 Kafka 발행 + 멱등 처리 고려
```

---

## 💻 실전 실험

### 실험 1: Outbox 테이블 기반 패턴 구현

```sql
-- PostgreSQL Outbox 테이블 생성
CREATE TABLE outbox_events (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processed BOOLEAN DEFAULT FALSE
);

-- 인덱스 (미처리 이벤트 조회용)
CREATE INDEX idx_outbox_unprocessed
    ON outbox_events(created_at)
    WHERE processed = FALSE;

-- 주문 생성 + Outbox 기록 (단일 트랜잭션)
BEGIN;
INSERT INTO orders (id, user_id, amount, status)
    VALUES ('order-1', 'user-1', 100, 'CREATED');
INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload)
    VALUES ('Order', 'order-1', 'OrderCreated', '{"amount": 100}');
COMMIT;
-- orders와 outbox_events 동시 커밋 (원자적)
```

### 실험 2: Debezium 커넥터 설정

```bash
# Debezium PostgreSQL 커넥터 등록
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "outbox-source-connector",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "postgres",
      "database.port": "5432",
      "database.user": "debezium",
      "database.password": "debezium",
      "database.dbname": "orderdb",
      "table.include.list": "public.outbox_events",
      "plugin.name": "pgoutput",
      "slot.name": "outbox_slot"
    }
  }'

# 커넥터 상태 확인
curl http://localhost:8083/connectors/outbox-source-connector/status

# Outbox 이벤트가 Kafka 토픽으로 전달되는지 확인
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic postgres.public.outbox_events \
  --from-beginning --property print.key=true
```

### 실험 3: Choreography Saga 패턴 시뮬레이션

```bash
# 이벤트 체인 시뮬레이션
# 1. 주문 생성 이벤트
echo "order-1:{type:OrderCreated,userId:user-1,amount:100}" | \
  kafka-console-producer --bootstrap-server localhost:9092 \
  --topic order-events --property "parse.key=true" --property "key.separator=:"

# 2. 재고 예약 이벤트 (재고 서비스가 발행)
echo "order-1:{type:InventoryReserved,itemId:item-1,qty:1}" | \
  kafka-console-producer --bootstrap-server localhost:9092 \
  --topic inventory-events --property "parse.key=true" --property "key.separator=:"

# 3. 결제 실패 이벤트 (결제 서비스가 발행)
echo "order-1:{type:PaymentFailed,reason:InsufficientFunds}" | \
  kafka-console-producer --bootstrap-server localhost:9092 \
  --topic payment-events --property "parse.key=true" --property "key.separator=:"

# 재고 서비스: PaymentFailed 구독 → 보상 트랜잭션 (재고 예약 취소)
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic payment-events --group inventory-compensation-consumer
```

---

## 📊 성능/비용 비교

### Outbox Pattern vs 직접 발행 비교

```
방식 1: 직접 Kafka 발행 (트랜잭션 없음)
  지연: 최소 (DB 저장 후 즉시 발행)
  구현: 단순
  원자성: 없음 (장애 시 불일치 가능)
  적합: 유실 허용, 멱등 처리가 완벽한 경우

방식 2: Spring ChainedTransactionManager
  지연: 최소
  구현: 중간
  원자성: 불완전 (Best-Effort 1PC)
  적합: 간단한 시스템, 낮은 정확성 요구

방식 3: Outbox Pattern + Debezium CDC
  지연: ~수백 ms (Debezium 폴링 주기)
  구현: 복잡 (Debezium 설치 및 운영)
  원자성: 완전 (DB 트랜잭션 활용)
  적합: 마이크로서비스, 높은 정확성 요구

방식 4: Outbox Pattern + Polling
  지연: ~폴링 주기 (1초)
  구현: 중간 (스케줄러 + 멱등 처리)
  원자성: 완전 (DB 트랜잭션 활용)
  적합: 인프라 단순화 선호, 낮은 실시간성 요구
```

---

## ⚖️ 트레이드오프

```
Outbox + Debezium:
  ✅ 완전한 원자성 (DB 트랜잭션 활용)
  ✅ 이벤트 유실 없음
  ✅ 비즈니스 로직 변경 없음
  ❌ Debezium 인프라 운영 비용 (추가 컴포넌트)
  ❌ DB WAL 설정 필요 (wal_level=logical)
  ❌ 수백 ms ~ 수 초 지연

Choreography Saga:
  ✅ 서비스 간 느슨한 결합
  ✅ 각 서비스 독립 확장
  ❌ 전체 흐름 파악 어려움
  ❌ 장애 진단 복잡 (이벤트 체인 추적)

Orchestration Saga:
  ✅ 전체 흐름 중앙 집중 관리
  ✅ 복잡한 분기/보상 처리 용이
  ❌ Saga Manager 단일 장애점 가능
  ❌ 서비스 간 결합 증가

보상 트랜잭션:
  ✅ 분산 트랜잭션 없이 일관성 달성
  ❌ 보상 로직 구현 복잡
  ❌ 보상 실패 시 수동 개입 필요
```

---

## 📌 핵심 정리

```
이벤트 기반 아키텍처 패턴 핵심:

1. 원자성 문제:
   DB 저장 + Kafka 발행은 원자적 보장 불가
   → Outbox Pattern으로 해결

2. Outbox Pattern:
   DB 트랜잭션 내에 비즈니스 데이터 + Outbox 이벤트 동시 기록
   Debezium CDC가 DB 변경을 Kafka로 전달
   → DB 원자성으로 Kafka 발행 원자성 보장

3. Debezium CDC:
   DB WAL 모니터링 → 변경 이벤트를 Kafka 발행
   별도 코드 없이 DB 변경 → Kafka 자동 전달

4. Saga Pattern:
   분산 트랜잭션 없이 일관성 달성
   Choreography: 이벤트 기반 (느슨한 결합)
   Orchestration: 중앙 조율 (전체 흐름 명확)

5. 보상 트랜잭션:
   실패 시 이미 완료된 작업을 되돌리는 역방향 작업
   Kafka를 통해 보상 이벤트 전파
```

---

## 🤔 생각해볼 문제

**Q1. Outbox Pattern에서 Debezium이 다운되면 이벤트는 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

Outbox 테이블에 미처리 이벤트가 쌓입니다. Debezium이 복구되면 마지막으로 처리한 WAL 위치부터 재시작하여 미처리 이벤트를 Kafka에 발행합니다.

PostgreSQL의 replication slot 덕분에 Debezium이 오프라인인 동안의 WAL이 보존됩니다. 단, replication slot이 있는 동안 PostgreSQL이 WAL을 삭제하지 않으므로 Debezium이 오랫동안 다운되면 WAL이 무제한 증가할 수 있습니다. 이를 방지하기 위해 Debezium 커넥터의 헬스체크와 알람 설정이 필요합니다.

Debezium 재시작 후 중복 이벤트가 발행될 수 있으므로 Consumer는 반드시 멱등 처리를 구현해야 합니다.

</details>

---

**Q2. Choreography와 Orchestration을 혼합해서 사용할 수 있나요?**

<details>
<summary>해설 보기</summary>

네, 실무에서는 혼합 사용이 일반적입니다.

복잡한 비즈니스 프로세스의 핵심 흐름은 Orchestration(Saga Manager)으로 관리하고, 그 과정에서 발생하는 부가적인 이벤트(알림, 로그, 분석 등)는 Choreography로 처리합니다.

예: 주문 처리 Saga
- Orchestration: 주문→재고예약→결제→배송 핵심 흐름 (Saga Manager 관리)
- Choreography: 주문 완료 이벤트 → 알림 서비스, 포인트 적립 서비스, 분석 서비스 (각 서비스가 이벤트 구독)

핵심 프로세스의 일관성은 Orchestration으로 보장하고, 부가 기능은 Choreography로 확장성 있게 처리하는 패턴이 실용적입니다.

</details>

---

**Q3. Outbox 테이블을 폴링 방식으로 구현할 때 중복 발행을 방지하는 방법은?**

<details>
<summary>해설 보기</summary>

트랜잭션과 낙관적 잠금을 활용합니다.

```sql
-- 미처리 이벤트를 선택 + 처리 중 표시 (원자적)
UPDATE outbox_events
SET status = 'PROCESSING', updated_at = NOW()
WHERE id IN (
    SELECT id FROM outbox_events
    WHERE status = 'PENDING'
    ORDER BY created_at
    LIMIT 100
    FOR UPDATE SKIP LOCKED  -- 다른 인스턴스가 처리 중인 것 건너뜀
)
RETURNING *;
```

Kafka 발행 성공 후:
```sql
UPDATE outbox_events
SET status = 'PUBLISHED', updated_at = NOW()
WHERE id IN (...);
```

타임아웃 처리 (PROCESSING 상태로 너무 오래 있는 이벤트 재처리):
```sql
UPDATE outbox_events
SET status = 'PENDING'
WHERE status = 'PROCESSING'
AND updated_at < NOW() - INTERVAL '5 minutes';
```

`FOR UPDATE SKIP LOCKED`가 핵심입니다. 여러 폴링 인스턴스가 동시에 실행될 때 중복 처리를 방지합니다. 이 방식은 Debezium보다 단순하지만 DB에 폴링 부하가 추가됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Exactly-Once in Kafka Streams](./04-exactly-once-streams.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — Spring Kafka 기초 ➡️](../spring-kafka/01-spring-kafka-basics.md)**

</div>
