# Exactly-Once Semantics 전 과정 — Read-Process-Write

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `isolation.level=read_committed`로 Consumer가 커밋된 메시지만 읽는 원리는?
- Read-Process-Write 패턴에서 EOS를 달성하는 전체 흐름은?
- Kafka Streams의 `processing.guarantee=exactly_once_v2`와 일반 EOS의 차이는?
- `exactly_once_v2`가 `exactly_once`보다 나은 이유는?
- EOS 달성 후에도 Consumer 처리 로직이 멱등해야 하는 이유는?
- EOS의 실제 비용과 한계는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Exactly-Once를 달성했다고 착각하는 경우가 많다. "Producer 트랜잭션 + `isolation.level=read_committed`를 설정했으니 Exactly-Once다"라고 생각하지만, Consumer 처리 로직이 외부 DB에 쓰거나 외부 API를 호출하는 경우 Kafka 레벨의 EOS가 외부 시스템까지 보장하지 않는다.

Kafka EOS가 실제로 보장하는 범위와 보장하지 않는 범위를 정확히 알아야:
- 결제 처리에 EOS를 적용했는데 DB에는 중복 기록이 생기는 사고 예방
- Kafka Streams 파이프라인에서 EOS 활성화의 실제 효과 이해
- EOS 비용을 감수할 것인지 멱등 처리로 대체할 것인지 합리적 선택 가능

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Kafka EOS = 외부 DB 중복 없음이라는 오해

  설정:
    Producer: enable.idempotence=true + transactional.id
    Consumer: isolation.level=read_committed

  Consumer 처리:
    @KafkaListener
    public void process(OrderEvent event) {
        producer.beginTransaction();
        producer.send("notifications", event.userId(), "주문 완료");
        producer.sendOffsetsToTransaction(...);
        producer.commitTransaction();
        
        // ← 이 부분은 Kafka EOS 보장 밖
        orderRepository.save(event);  // DB에 저장
    }

  문제:
    Kafka 트랜잭션은 커밋됐지만
    orderRepository.save() 실패 후 재처리 시
    Kafka 레벨에서는 중복이 차단되지 않음
    (새 트랜잭션으로 다시 send() 가능)
    → DB에 중복 저장 발생

실수 2: exactly_once와 exactly_once_v2 혼용

  Kafka Streams 설정:
    processing.guarantee=exactly_once  ← 구버전
    또는
    processing.guarantee=exactly_once_v2  ← 권장

  exactly_once 문제:
    각 Task마다 트랜잭션 Producer 생성
    5개 Task = 5개 트랜잭션 코디네이터 연결
    → 오버헤드 큼

  exactly_once_v2 개선:
    브로커가 직접 Epoch 관리 (KIP-447)
    Task가 아닌 스레드당 1개 트랜잭션 Producer
    오버헤드 크게 감소 (Kafka 2.5+)

실수 3: EOS 활성화 후 처리 로직의 멱등성 검증 생략

  착각: "EOS 설정했으니 중복 처리 걱정 없다"
  현실:
    Kafka 레벨 EOS = Kafka 토픽 간 중복 없음
    외부 시스템(DB, 이메일, API) 호출 = EOS 밖
    처리 로직이 외부 시스템과 상호작용하면 별도 멱등성 설계 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
EOS가 적합한 시나리오:
  Kafka → Kafka 파이프라인 (입력 토픽 → 처리 → 출력 토픽)
  Kafka Streams 집계 연산
  이벤트 라우팅 (동일 이벤트를 여러 토픽에 분기)

EOS로 커버 안 되는 시나리오:
  → 추가 멱등 처리 필요
  Kafka → DB (UPSERT 또는 처리 여부 테이블)
  Kafka → 외부 API (멱등 API 설계 또는 중복 키 체크)
  Kafka → 이메일/SMS (발송 이력 테이블)

Kafka Streams EOS 설정:
  # application.properties
  spring.kafka.streams.properties.processing.guarantee=exactly_once_v2
  # Kafka 2.5 이상에서 사용 권장

일반 Producer/Consumer EOS:
  Properties props = new Properties();
  props.put("transactional.id", "processor-" + instanceId);

  producer.initTransactions();
  
  while (running) {
      ConsumerRecords<K,V> records = consumer.poll(Duration.ofMillis(100));
      
      producer.beginTransaction();
      try {
          for (ConsumerRecord<K,V> record : records) {
              String result = transform(record.value());
              producer.send(new ProducerRecord<>("output", record.key(), result));
          }
          producer.sendOffsetsToTransaction(
              getCurrentOffsets(records),
              consumer.groupMetadata()
          );
          producer.commitTransaction();
      } catch (Exception e) {
          producer.abortTransaction();
      }
  }
```

---

## 🔬 내부 동작 원리

### 1. isolation.level=read_committed의 LSO 기반 읽기

```
LSO (Last Stable Offset):
  = 가장 오래된 미완료 트랜잭션의 시작 offset
  = Consumer가 read_committed로 읽을 수 있는 최대 offset

  HW(High Watermark) vs LSO:
    HW: ISR 전체가 복제 완료한 offset (일반 Consumer 읽기 한계)
    LSO: 미완료 트랜잭션이 없을 때 = HW, 있을 때 < HW

  예시:
    offset 100: 트랜잭션 시작 (아직 commit/abort 없음)
    offset 150: 일반 메시지 (트랜잭션 외)
    offset 200: HW

    read_uncommitted Consumer: offset 200까지 읽기 가능
    read_committed Consumer:   LSO = 100 → offset 99까지만 읽기 가능
    → 트랜잭션이 commit/abort되면 LSO 상승 → 150, 200 읽기 가능

  LSO 지연 영향:
    미완료 트랜잭션이 오래 열려있으면
    read_committed Consumer 전체가 지연
    → transaction.timeout.ms를 적절히 설정
    → 처리 실패 시 빠르게 abortTransaction() 호출
```

### 2. EOS Read-Process-Write 전체 흐름

```
완전한 EOS 파이프라인:

  입력: "input-topic" (partition=0, offset=200~299)
  처리: 메시지 변환 + 집계
  출력: "output-topic" (결과 발행)

  Step 1. Consumer Fetch
    isolation.level=read_committed
    → 미커밋 트랜잭션 메시지 건너뜀
    → 커밋된 메시지만 받음

  Step 2. beginTransaction()
    Producer 트랜잭션 시작
    Transaction Coordinator에 PID + Epoch 확인

  Step 3. 처리 + output 발행
    for record in records:
        result = transform(record)
        producer.send("output-topic", result)
        // UNCOMMITTED 상태로 output-topic에 기록됨
        // read_committed Consumer는 아직 못 읽음

  Step 4. sendOffsetsToTransaction()
    Producer → Transaction Coordinator:
    "input-topic partition=0 offset=300도 이 트랜잭션에 포함"
    // Consumer offset 커밋도 트랜잭션의 일부가 됨

  Step 5. commitTransaction()
    TC: PrepareCommit 기록 → __transaction_state
    TC → output-topic 브로커: COMMIT 마커
    TC → __consumer_offsets 브로커: COMMIT 마커 + offset=300 커밋
    모든 완료 → Producer에게 성공 응답

  Step 6. 성공 후:
    output-topic의 result 메시지: COMMITTED (Consumer가 읽을 수 있음)
    input-topic의 offset: 300으로 커밋 완료
    → 다음 poll()은 offset 300부터 시작

  크래시 시나리오 (Step 5 전):
    트랜잭션 abort (timeout 또는 명시적 abort)
    output-topic의 result 메시지: ABORTED (read_committed Consumer 건너뜀)
    input-topic의 offset: 커밋 안 됨 → 재시작 후 offset 200부터 재처리
    output-topic에 중복 발행 시도 → read_committed Consumer는 못 봄
    → Exactly-Once 달성!
```

### 3. Kafka Streams exactly_once_v2

```
exactly_once와 exactly_once_v2 차이:

  exactly_once (Kafka 0.11):
    각 StreamTask마다 독립적 트랜잭션 Producer
    Task 수 = 트랜잭션 Producer 수
    → Task 100개 → 브로커와 100개 트랜잭션 연결 유지
    → 오버헤드 큼, 브로커 부하 증가

  exactly_once_v2 (Kafka 2.5+, KIP-447):
    스레드당 1개 트랜잭션 Producer (Task 공유)
    여러 Task가 동일 Producer를 공유
    Producer Epoch를 브로커가 직접 관리 (TC 오버헤드 감소)
    → 연결 수 급감 → 오버헤드 크게 감소

  성능 차이:
    exactly_once:   처리량 ~40% 감소 (대규모 Task)
    exactly_once_v2: 처리량 ~20% 감소

  설정:
    processing.guarantee=exactly_once_v2
    (Kafka 2.5 이상 필수, 이하는 exactly_once 사용)
```

### 4. EOS의 실제 보장 범위

```
EOS가 보장하는 것:
  ✅ Kafka 토픽 → Kafka 토픽 파이프라인에서 중복 없음
     (input 읽기 + output 발행 + offset 커밋 원자적)
  ✅ Kafka Streams 집계/변환 연산의 상태 일관성
  ✅ 동일 Kafka 클러스터 내 멀티 파티션 원자적 쓰기

EOS가 보장하지 않는 것:
  ❌ Kafka → 외부 DB 중복 없음
  ❌ Kafka → 외부 API 중복 없음
  ❌ 처리 로직 내 사이드 이펙트 (이메일 발송 등)
  ❌ 다른 Kafka 클러스터 간 원자성 (MirrorMaker 사용 시)

외부 시스템 연계 시 추가 전략:
  DB: UPSERT (ON CONFLICT DO UPDATE) + message_id 기반 중복 체크
  API: 멱등 API 설계 (PUT with resource ID) + 처리 이력 테이블
  이메일: 발송 이력 테이블 + 중복 체크 후 발송
```

---

## 💻 실전 실험

### 실험 1: EOS 파이프라인 구현

```java
// Spring Kafka EOS 설정
@Configuration
public class EosConfig {

    @Bean
    public KafkaTransactionManager<String, String> transactionManager(
            ProducerFactory<String, String> pf) {
        return new KafkaTransactionManager<>(pf);
    }

    @Bean
    @Transactional("transactionManager")
    public KafkaListenerContainerFactory<?> eosFactory() {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.getContainerProperties().setEosMode(EosMode.V2);
        return factory;
    }
}

@KafkaListener(topics = "input-topic",
               containerFactory = "eosFactory")
@Transactional("transactionManager")
public void process(ConsumerRecord<String, String> record) {
    String result = transform(record.value());
    // 트랜잭션 내에서 output 토픽 발행 + offset 커밋
    kafkaTemplate.send("output-topic", record.key(), result);
    // Spring이 자동으로 sendOffsetsToTransaction() 호출
}
```

### 실험 2: LSO와 read_committed 동작 확인

```bash
# 트랜잭션 시작 후 abort 메시지를 read_uncommitted vs read_committed로 비교

# read_uncommitted: abort 메시지 보임
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic output-topic \
  --isolation-level read_uncommitted \
  --from-beginning

# read_committed: abort 메시지 안 보임
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic output-topic \
  --isolation-level read_committed \
  --from-beginning
# 메시지 수 비교
```

### 실험 3: Kafka Streams EOS 처리량 비교

```bash
# EOS 없이 (기본 at_least_once)
# spring.kafka.streams.properties.processing.guarantee=at_least_once

# EOS v2 활성화
# spring.kafka.streams.properties.processing.guarantee=exactly_once_v2

# 동일 토폴로지에서 처리량 측정
kafka-producer-perf-test \
  --topic streams-input \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:19092

# Streams 앱 처리량을 JMX로 측정
# kafka.streams:type=stream-processor-node-metrics,
#   thread-id=*,task-id=*,processor-node-id=*
# process-rate 지표 비교
```

---

## 📊 성능/비용 비교

### EOS 활성화 단계별 처리량

```
조건: Kafka Streams 파이프라인, 파티션 12개, 1 KB 메시지

at_least_once:
  처리량: ~300,000 msg/sec
  지연(p99): ~10 ms

exactly_once (구버전):
  처리량: ~180,000 msg/sec (-40%)
  지연(p99): ~25 ms
  Task당 트랜잭션 Producer 오버헤드

exactly_once_v2:
  처리량: ~240,000 msg/sec (-20%)
  지연(p99): ~18 ms
  스레드당 트랜잭션 공유 → 오버헤드 감소

일반 Producer/Consumer EOS:
  처리량: ~130,000 msg/sec (-35%)
  지연(p99): ~30 ms
  TC 통신 오버헤드
```

### EOS vs 멱등 처리 선택 기준

```
EOS (Kafka 트랜잭션) 선택:
  처리 결과가 Kafka 토픽에만 쓰여지는 경우
  처리 로직을 멱등하게 만들기 어렵거나 비용이 큰 경우
  Kafka Streams 파이프라인에서 상태 일관성이 필수인 경우

멱등 처리 선택:
  처리 결과가 외부 DB, API로 나가는 경우
  DB UPSERT 등으로 쉽게 멱등성 보장 가능
  EOS 처리량 감소가 허용되지 않는 경우
  구현 단순성이 중요한 경우

"처리량 감소 20~35% + 구현 복잡도 vs 멱등 처리 로직 추가 설계"
→ 대부분의 실무에서 멱등 처리가 더 실용적
→ EOS는 Kafka 내부 파이프라인에서 진가를 발휘
```

---

## ⚖️ 트레이드오프

```
EOS 사용:
  ✅ Kafka 토픽 간 완전한 중복 없음
  ✅ 크래시 후 재처리해도 output 중복 없음 (read_committed 조합)
  ✅ Kafka Streams에서 상태 일관성
  ❌ 처리량 20~40% 감소
  ❌ 구현 복잡도 (트랜잭션 관리)
  ❌ 외부 시스템 보장 불가 (DB, API)
  ❌ LSO 지연으로 read_committed Consumer 처리 지연 가능

At-Least-Once + 멱등 처리:
  ✅ 높은 처리량
  ✅ 구현 단순 (UPSERT 등)
  ✅ 외부 시스템도 커버 가능
  ❌ 멱등 처리 로직 설계 필요
  ❌ 외부 API 호출처럼 멱등화 어려운 케이스 존재
  → 가장 실용적인 선택 (대부분 서비스)
```

---

## 📌 핵심 정리

```
Exactly-Once Semantics 핵심:

1. Kafka EOS = Kafka 토픽 간에만 보장
   외부 DB, API는 별도 멱등성 설계 필요

2. EOS 달성 조건:
   Producer: transactional.id + beginTransaction/commitTransaction
   Consumer: isolation.level=read_committed
   핵심 API: sendOffsetsToTransaction() (offset 커밋 원자적 포함)

3. isolation.level=read_committed = LSO 기반 읽기
   미완료 트랜잭션 있으면 LSO 이하만 읽음 → Consumer 지연 가능

4. Kafka Streams EOS:
   exactly_once_v2 (Kafka 2.5+) 권장
   exactly_once 대비 오버헤드 절반

5. 실무 선택:
   Kafka → Kafka 파이프라인 → EOS 고려
   Kafka → 외부 시스템 → At-Least-Once + 멱등 처리
   대부분: At-Least-Once + UPSERT가 가장 실용적
```

---

## 🤔 생각해볼 문제

**Q1. EOS 환경에서 Consumer가 `read_committed`로 설정돼 있는데 메시지를 전혀 못 받는 경우가 있었습니다. 원인이 뭘까요?**

<details>
<summary>해설 보기</summary>

LSO(Last Stable Offset)가 오래된 미완료 트랜잭션 때문에 상승하지 않는 상황입니다.

어떤 Producer가 트랜잭션을 시작(`beginTransaction`)하고 `commitTransaction` 또는 `abortTransaction` 없이 멈춰있다면, 해당 트랜잭션의 시작 offset에서 LSO가 멈춥니다. `read_committed` Consumer는 LSO 이하만 읽을 수 있으므로 그 이후에 기록된 메시지(커밋된 트랜잭션 포함)도 읽지 못합니다.

진단 방법: JMX에서 `records-lag`가 계속 쌓이는지 확인하고, `kafka-consumer-groups --describe`로 LAG 확인. `__transaction_state` 토픽에서 오래된 미완료 트랜잭션 확인.

해결: `transaction.timeout.ms`가 지나면 TC가 자동 abort합니다. 만약 Producer가 좀비 상태로 실행 중이라면 해당 인스턴스를 종료하거나 동일 `transactional.id`로 새 인스턴스를 시작해서 Epoch Fencing으로 기존 트랜잭션을 강제 abort합니다.

</details>

---

**Q2. `sendOffsetsToTransaction()`을 호출하면 Consumer Group의 offset 커밋도 트랜잭션에 포함된다고 했는데, Consumer가 직접 `commitSync()`를 호출하면 안 되나요?**

<details>
<summary>해설 보기</summary>

EOS 패턴에서는 Consumer가 직접 `commitSync()`를 호출하면 안 됩니다.

`sendOffsetsToTransaction()`은 Consumer의 offset 커밋을 Producer 트랜잭션의 일부로 포함합니다. 즉, `commitTransaction()`이 성공할 때만 offset도 커밋됩니다. `abortTransaction()`이면 offset도 커밋되지 않습니다.

만약 Consumer가 별도로 `commitSync()`를 호출하면 트랜잭션과 독립적으로 offset이 커밋됩니다. 이렇게 되면 트랜잭션이 abort되더라도 offset은 이미 커밋됐기 때문에 재처리가 발생하지 않고 메시지가 유실됩니다.

Spring Kafka의 `EosMode.V2`를 사용하면 `sendOffsetsToTransaction()` 호출을 자동으로 처리하고 Consumer의 직접 커밋을 방지합니다.

</details>

---

**Q3. 결제 처리 서비스에서 EOS를 적용한다면 어떤 구조가 올바른가요?**

<details>
<summary>해설 보기</summary>

결제는 외부 시스템(DB, 결제 게이트웨이)을 포함하므로 순수 Kafka EOS만으로는 불충분합니다. 다음과 같은 복합 구조를 권장합니다.

**1단계 (이벤트 수신)**: `isolation.level=read_committed`로 결제 이벤트 Fetch. 이미 처리된 이벤트인지 DB에서 확인(멱등성 체크).

**2단계 (처리)**: DB 트랜잭션 내에서 결제 처리 + payment_events 테이블에 processed 마킹 + Outbox 테이블에 결과 이벤트 기록. DB 트랜잭션 커밋.

**3단계 (발행)**: CDC(Debezium)가 Outbox 테이블 변경사항을 감지해서 Kafka에 발행. DB 트랜잭션과 Kafka 발행의 원자성을 Outbox Pattern으로 보장.

**4단계 (Offset 커밋)**: 결제 처리 완료 확인 후 Consumer offset 커밋.

이 구조에서 핵심 보장은 Kafka EOS가 아닌 **DB 트랜잭션 + Outbox Pattern + 멱등 체크**입니다. Kafka EOS는 Kafka 내부 파이프라인에서만 사용하고, 외부 시스템 연계는 애플리케이션 레벨에서 멱등성을 보장합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Consumer Offset 관리](./04-consumer-offset-management.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — Consumer Group 내부 동작 ➡️](../consumer-rebalancing/01-consumer-group-internals.md)**

</div>
