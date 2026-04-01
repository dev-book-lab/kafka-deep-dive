# 트랜잭션 Producer — 멀티 파티션 원자적 쓰기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `transactional.id`와 Transaction Coordinator의 역할은 무엇인가?
- `beginTransaction() → send() → commitTransaction()`의 내부 단계는?
- Two-Phase Commit이 Kafka 트랜잭션에서 어떻게 구현되는가?
- Producer Epoch가 좀비 Producer를 차단하는 메커니즘은?
- 트랜잭션이 abort되면 Consumer는 어떻게 동작하는가?
- `sendOffsetsToTransaction()`이 EOS Read-Process-Write를 가능하게 하는 원리는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

멱등성은 단일 파티션 내 단일 세션 중복을 막아준다. 하지만 "주문 완료 이벤트를 orders 파티션에 쓰고, 재고 감소 이벤트를 inventory 파티션에 쓰는" 작업이 원자적이어야 한다면 멱등성만으로는 부족하다.

트랜잭션 Producer는:
- 멀티 파티션에 원자적으로 쓰거나(모두 성공 or 모두 실패)
- Consumer offset 커밋을 output 발행과 원자적으로 묶어서

진정한 Exactly-Once를 달성한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 트랜잭션 없이 멀티 파티션 원자성 가정

  코드:
    producer.send(new ProducerRecord<>("orders", orderId, orderEvent));
    producer.send(new ProducerRecord<>("inventory", itemId, stockEvent));
    // "두 개 다 성공하겠지"

  문제:
    orders 파티션 성공, inventory 파티션 실패
    → 주문은 됐는데 재고 감소 안 됨 (데이터 불일치!)
    
  올바른 접근:
    producer.initTransactions();
    producer.beginTransaction();
    producer.send("orders", ...);
    producer.send("inventory", ...);
    producer.commitTransaction();  // 둘 다 성공해야 커밋
    // 실패 시 abortTransaction() → 둘 다 없던 일로

실수 2: transactional.id를 여러 인스턴스에서 공유

  설정:
    Instance A: transactional.id="order-processor"
    Instance B: transactional.id="order-processor" (동일!)

  문제:
    Instance A 실행 중
    Instance B 시작 → 동일 transactional.id → 새 Epoch 발급
    → Instance A는 좀비로 판단 → Instance A의 트랜잭션 강제 abort
    → Instance A가 commitTransaction() 호출 시 예외 발생
    
  올바른 접근:
    Instance A: transactional.id="order-processor-0"
    Instance B: transactional.id="order-processor-1"
    (각 인스턴스는 고유한 transactional.id 사용)

실수 3: commitTransaction 후에도 Consumer가 즉시 읽는다고 가정

  설정: consumer isolation.level=read_uncommitted (기본값)
  결과:
    트랜잭션 abort된 메시지도 Consumer가 읽음
    → EOS를 달성했다고 생각했는데 abort 메시지도 처리됨
  
  올바른 설정:
    isolation.level=read_committed
    → 커밋된 트랜잭션 메시지만 읽음
    → abort된 메시지 자동 건너뜀
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
트랜잭션 Producer 올바른 패턴:

  Properties props = new Properties();
  props.put("transactional.id", "order-processor-" + instanceId);
  // transactional.id는 인스턴스별로 고유하게
  
  KafkaProducer<String, String> producer = new KafkaProducer<>(props);
  producer.initTransactions();  // Transaction Coordinator에 등록
  
  try {
      producer.beginTransaction();
      producer.send(new ProducerRecord<>("orders", key, value));
      producer.send(new ProducerRecord<>("inventory", key, value));
      
      // EOS: Consumer offset도 트랜잭션에 포함
      producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);
      
      producer.commitTransaction();  // 원자적 커밋
  } catch (ProducerFencedException e) {
      // 좀비 Producer: 다른 인스턴스가 동일 transactional.id 사용 중
      producer.close();
  } catch (KafkaException e) {
      producer.abortTransaction();   // 원자적 롤백
  }

Spring Kafka 설정:
  spring:
    kafka:
      producer:
        transaction-id-prefix: order-processor-
        # 내부적으로 인스턴스별로 고유 ID 생성
```

---

## 🔬 내부 동작 원리

### 1. Transaction Coordinator의 역할

```
Transaction Coordinator:
  - Kafka 브로커 중 하나
  - __transaction_state 토픽의 파티션 Leader 브로커
  - 할당: hash(transactional.id) % 50 → 파티션 N → 해당 Leader

  관리 내용:
    transactional.id → {PID, Epoch, State, Partitions, Timeout}
    State: Empty → Ongoing → PrepareCommit → CompleteCommit
                           → PrepareAbort → CompleteAbort

  __transaction_state 토픽:
    50개 파티션, cleanup.policy=compact
    각 transactional.id의 최신 상태만 보존
```

### 2. 트랜잭션 전체 흐름 (Two-Phase Commit)

```
initTransactions() 호출:
  Producer → Transaction Coordinator
  InitProducerId 요청 (transactional.id="order-processor-0")
  
  TC: transactional.id 조회
    신규: PID 발급, Epoch=0
    기존: Epoch++ (이전 인스턴스의 PID 무효화 = Fencing)
  응답: PID=100, Epoch=1

beginTransaction():
  Producer 내부 상태만 변경 (브로커 통신 없음)
  transactionInFlight = true

send() × N 회:
  각 배치에 PID + Epoch + Seq 포함
  배치를 받은 브로커들은 파티션 목록을
  Transaction Coordinator에 등록 요청
  → TC: 이 트랜잭션에 참여한 파티션 목록 기록

commitTransaction() — Phase 1 (Prepare):
  Producer → TC: EndTransaction(COMMIT)
  TC: 상태를 PrepareCommit으로 변경
  → __transaction_state에 기록 (장애 복구를 위해 퍼시스턴트)

commitTransaction() — Phase 2 (Commit):
  TC → 각 파티션 브로커: WriteTxnMarker(COMMIT)
  각 브로커: 해당 파티션에 COMMIT 마커 추가
  모든 브로커 응답 완료
  TC: 상태를 CompleteCommit으로 변경
  Producer에게 성공 응답

┌──────────────────────────────────────────────────────────────┐
│ 파티션 로그 (트랜잭션 완료 후):                                     │
│                                                              │
│ [Batch PID=100 Seq=0 타입=DATA] ← 실제 메시지                    │
│ [Batch PID=100 Seq=1 타입=DATA] ← 실제 메시지                    │
│ [ControlBatch PID=100 타입=COMMIT] ← 커밋 마커                  │
│                                                              │
│ isolation.level=read_committed Consumer는                     │
│ COMMIT 마커를 본 후에야 DATA 배치를 소비                            │
└──────────────────────────────────────────────────────────────┘
```

### 3. Producer Epoch와 좀비 차단

```
좀비 Producer 문제:
  Instance A (Epoch=1)가 장애 후 GC STW로 응답 없음
  Instance B (Epoch=2)로 새 인스턴스 시작
  Instance A가 GC 종료 후 트랜잭션 작업 재개 시도
  → 두 인스턴스가 동시에 동일 transactional.id로 쓰기 → 데이터 불일치!

Epoch 기반 Fencing:
  Instance B 시작 → TC에게 InitProducerId(transactional.id)
  TC: Epoch 1 → Epoch 2로 증가
  TC: Instance A의 PID(Epoch=1)를 무효화

  Instance A가 send/commit 시도:
  브로커: 수신 배치의 Epoch=1 < 현재 TC의 Epoch=2
  → ProducerFencedException 발생
  → Instance A가 강제 종료됨

  결과: 항상 최신 Epoch의 인스턴스만 활성 트랜잭션을 진행할 수 있음
        좀비 Producer 차단 완료
```

### 4. sendOffsetsToTransaction으로 EOS 달성

```
Read-Process-Write 패턴에서 EOS:

  일반 At-Least-Once 방식의 문제:
    Consumer: poll() → process() → 결과를 output 토픽에 produce()
    → output 토픽에 produce 성공
    → Consumer commit 실패 (크래시)
    → 재시작 후 같은 메시지 재처리 → output 토픽에 중복 발행

  EOS 방식:
    Consumer: poll() (isolation.level=read_committed)
    process()
    producer.beginTransaction()
    producer.send("output-topic", result)
    producer.sendOffsetsToTransaction(
        {TopicPartition → offset},    // 처리한 offset
        consumer.groupMetadata()      // Consumer Group 정보
    )
    producer.commitTransaction()
    // output 발행 + Consumer offset 커밋이 하나의 트랜잭션!

  sendOffsetsToTransaction 내부 동작:
    Producer → TC: 이 트랜잭션에 Consumer offset 커밋 포함 요청
    TC → __consumer_offsets 파티션 브로커: WriteTxnMarker(COMMIT)와 함께
    commitTransaction() 시 offset 커밋도 동시에 완료

  결과:
    처리 결과 발행 + Consumer offset 커밋 = 원자적
    크래시 후 재시작: offset이 커밋 안 됐으면 재처리 → 중복 발행 시도
    → 이전 트랜잭션이 abort → output 토픽의 중복 메시지는 abort 처리
    → Consumer(read_committed)는 abort 메시지 건너뜀 → Exactly-Once!
```

### 5. abort 시 동작

```
abortTransaction() 호출 또는 Transaction Coordinator가 타임아웃 감지:

  TC → 각 파티션 브로커: WriteTxnMarker(ABORT)
  각 브로커: ABORT 마커 추가

  파티션 로그 (abort 후):
    [Batch PID=100 Seq=0 타입=DATA]     ← 기록됨
    [Batch PID=100 Seq=1 타입=DATA]     ← 기록됨
    [ControlBatch PID=100 타입=ABORT]   ← abort 마커

  isolation.level=read_committed Consumer:
    DATA 배치 발견 → COMMIT/ABORT 마커 기다림
    ABORT 마커 확인 → DATA 배치 건너뜀
    → Consumer는 이 메시지들을 받지 못함

  isolation.level=read_uncommitted Consumer (기본값):
    DATA 배치 발견 → 즉시 소비
    ABORT 마커 무시
    → abort된 메시지도 받음 (EOS 아님!)
```

---

## 💻 실전 실험

### 실험 1: 트랜잭션 커밋/abort 확인

```bash
# Spring Kafka Transaction 설정 (application.yml)
# spring.kafka.producer.transaction-id-prefix=tx-

# 두 파티션에 원자적 쓰기 확인
# 정상 커밋 후 Consumer에서 확인
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --isolation-level read_committed \
  --from-beginning
# abort된 메시지는 보이지 않음 확인
```

### 실험 2: 트랜잭션 상태 확인

```bash
# __transaction_state 토픽에서 트랜잭션 상태 확인
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic __transaction_state \
  --formatter "kafka.coordinator.transaction.TransactionLog\$TransactionLogMessageFormatter" \
  --from-beginning 2>/dev/null | grep "order-processor"
# 출력: order-processor-0::TransactionMetadata(state=CompleteCommit, ...)
```

### 실험 3: isolation.level 차이 확인

```bash
# abort된 메시지 포함: read_uncommitted
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --isolation-level read_uncommitted \
  --from-beginning

# abort된 메시지 제외: read_committed
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --isolation-level read_committed \
  --from-beginning
# → read_committed에서 메시지 수가 더 적을 수 있음
```

---

## 📊 성능/비용 비교

### 트랜잭션 사용에 따른 처리량 영향

```
조건: 파티션 3개, 복제 팩터 3, 1 KB 메시지

트랜잭션 없음 (acks=all):
  처리량: ~200,000 msg/sec
  지연(p99): ~15 ms
  TC 통신: 없음

트랜잭션 사용:
  처리량: ~130,000 msg/sec (~35% 감소)
  지연(p99): ~30 ms
  TC 통신: beginTx(1회) + 파티션 등록(배치당) + commit(2회)

트랜잭션 배치 크기 최적화:
  배치당 메시지 수 많을수록 TC 통신 고정 비용이 분산
  linger.ms=100, batch.size=1MB로 배치 크게 → 오버헤드 감소
  처리량 회복: ~170,000 msg/sec
```

### transaction.timeout.ms 설정 영향

```
transaction.timeout.ms=60000 (1분, 기본값):
  1분 동안 commitTransaction/abortTransaction 없으면 TC가 자동 abort
  처리 시간이 긴 작업에 여유 있게 설정

너무 짧게 (예: 1000ms):
  처리 시간 1초 초과 시 TC가 자동 abort
  → abort된 트랜잭션의 메시지는 Consumer에게 보이지 않음 (read_committed)
  → 데이터 유실처럼 보이는 현상

너무 길게 (예: 3600000ms = 1시간):
  좀비 트랜잭션이 1시간 동안 살아있음
  → read_committed Consumer가 해당 파티션 HW(High Watermark)를 LSO(Last Stable Offset)까지만 읽음
  → 미완료 트랜잭션 동안 read_committed Consumer 지연 증가
```

---

## ⚖️ 트레이드오프

```
트랜잭션 사용:
  ✅ 멀티 파티션 원자적 쓰기
  ✅ EOS Read-Process-Write 달성
  ✅ 좀비 Producer 차단 (Epoch 기반)
  ❌ 처리량 ~35% 감소
  ❌ Transaction Coordinator 추가 통신 오버헤드
  ❌ transactional.id 관리 필요 (인스턴스별 고유)
  ❌ read_committed Consumer에서 LSO 지연 발생 가능
  ❌ 외부 DB 연계 원자성 불가 (Kafka 내부만)

트랜잭션 없이 멱등성만:
  ✅ 단일 파티션 재시도 중복 제거
  ✅ 처리량 영향 미미 (~1%)
  ❌ 멀티 파티션 원자성 없음
  ❌ Producer 재시작 후 중복 가능
```

---

## 📌 핵심 정리

```
트랜잭션 Producer 핵심:

1. transactional.id → Transaction Coordinator에서 PID + Epoch 발급
   Epoch++가 이전 좀비 Producer를 Fencing

2. Two-Phase Commit:
   Phase 1: TC에 PrepareCommit 기록 (내구성)
   Phase 2: 각 파티션에 COMMIT 마커 기록

3. abort 시 각 파티션에 ABORT 마커
   read_committed Consumer는 ABORT 마커 메시지 건너뜀

4. sendOffsetsToTransaction()
   Consumer offset 커밋을 트랜잭션에 포함
   결과 발행 + offset 커밋 = 원자적 → EOS 달성

5. transactional.id는 인스턴스별 고유
   공유 시 Fencing으로 기존 트랜잭션 강제 abort
```

---

## 🤔 생각해볼 문제

**Q1. `commitTransaction()` 중 Transaction Coordinator가 장애나면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

Phase 1(PrepareCommit 기록)이 완료됐다면 안전하게 복구됩니다. `__transaction_state` 토픽에 PrepareCommit 상태가 기록됐으므로, 새 Transaction Coordinator가 선출된 후 이 상태를 읽고 Phase 2를 완료합니다(Coordinator 재선출 후 자동으로 미완료 트랜잭션 복구).

Phase 1 전에 TC가 장애나면 트랜잭션은 commit/abort되지 않은 상태입니다. `transaction.timeout.ms` 초과 후 새 TC가 자동으로 abort 처리합니다.

Producer 입장에서는 `commitTransaction()`에서 타임아웃이 발생하거나 예외가 던져집니다. 이 경우 `abortTransaction()`을 호출하거나 예외를 다시 던져서 상위 로직에서 재시도하게 합니다.

</details>

---

**Q2. `read_committed` Consumer가 현재 진행 중인(미완료) 트랜잭션 메시지를 기다리는 동안 그 파티션의 다른 메시지도 못 받나요?**

<details>
<summary>해설 보기</summary>

네, 그렇습니다. 이것이 LSO(Last Stable Offset) 개념입니다.

`read_committed` Consumer는 HW(High Watermark)가 아닌 LSO(Last Stable Offset)까지만 읽습니다. LSO는 "가장 오래된 미완료 트랜잭션의 시작 offset"입니다.

예를 들어 offset 100에서 트랜잭션이 시작되고 아직 commit/abort가 안 됐다면, offset 200에 일반 메시지가 있어도 Consumer는 offset 99까지만 읽습니다. offset 100~200은 트랜잭션 완료(commit 또는 abort) 후에야 읽을 수 있습니다.

이 때문에 `transaction.timeout.ms`를 너무 길게 설정하거나 오래 실행되는 트랜잭션은 `read_committed` Consumer의 지연을 유발합니다. 트랜잭션은 가능한 한 짧게 유지하는 것이 중요합니다.

</details>

---

**Q3. Spring Kafka에서 `@Transactional`과 `KafkaTransactionManager`를 사용할 때 DB 트랜잭션과 Kafka 트랜잭션은 어떻게 연동되나요?**

<details>
<summary>해설 보기</summary>

Spring의 `@Transactional`은 단일 트랜잭션 매니저를 기준으로 동작합니다. `KafkaTransactionManager`를 사용하면 Kafka 트랜잭션만 관리하고, DB 트랜잭션은 별도입니다.

**Best-Effort 1PC(단일 단계 커밋)**: Spring의 `ChainedKafkaTransactionManager`를 사용하면 DB 트랜잭션 커밋 후 Kafka 트랜잭션을 커밋합니다. DB 커밋 성공 후 Kafka 커밋 전 장애나면 Kafka 메시지가 abort → 재시도 → DB에 중복 커밋 시도. DB에 멱등성(UPSERT)이 있으면 안전합니다.

**진정한 원자성**: 외부 DB와 Kafka 간 완전한 원자성은 Outbox Pattern이 가장 실용적입니다. DB 트랜잭션 내에서 비즈니스 데이터와 outbox 테이블에 동시 기록 → CDC(Debezium)로 outbox 변경사항을 Kafka에 발행 → DB 트랜잭션과 Kafka 발행의 원자성 보장. 이 내용은 Chapter 6에서 자세히 다룹니다.

</details>

---

<div align="center">

**[⬅️ 이전: Producer 멱등성](./02-producer-idempotence.md)** | **[홈으로 🏠](../README.md)** | **[다음: Consumer Offset 관리 ➡️](./04-consumer-offset-management.md)**

</div>
