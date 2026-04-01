# 전달 보장 3단계 — At-Most-Once / At-Least-Once / Exactly-Once

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- At-Most-Once, At-Least-Once, Exactly-Once 각각의 구현 방식은 무엇인가?
- Producer 쪽 보장과 Consumer 쪽 보장이 어떻게 결합되어 최종 전달 보장이 결정되는가?
- 실무에서 At-Least-Once가 기본이 되는 이유는?
- Exactly-Once가 필요한 시나리오와 그 비용은 얼마인가?
- "처리 후 커밋"이 At-Least-Once를, "커밋 후 처리"가 At-Most-Once를 만드는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka 메시지 처리에서 장애가 발생했을 때 어떤 메시지가 어떻게 처리되는지는 세 가지 보장 단계에 따라 완전히 다르다.

이를 모르면:
- "Kafka를 쓰면 메시지가 정확히 한 번 처리된다"고 오해 → 기본은 At-Least-Once(중복 가능)
- 결제 처리에 At-Least-Once 코드를 그대로 써서 중복 결제 발생
- Exactly-Once를 위해 Kafka 트랜잭션을 썼지만 Consumer 쪽 처리가 멱등하지 않아 중복 발생

전달 보장 3단계의 정확한 의미와 구현 방식을 이해해야 서비스 요구사항에 맞는 전략을 선택할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Kafka = 메시지 정확히 한 번 처리라는 오해

  현실:
    기본 설정(enable.auto.commit=true, retries>0)
    = At-Least-Once (중복 가능)
    
    At-Most-Once(유실 가능) 달성도 잘못된 구현으로 쉽게 발생:
      Consumer가 fetch하자마자 커밋 → 처리 전 크래시 → 유실

  Exactly-Once는 명시적으로 설정해야:
    Producer: enable.idempotence=true + transactional.id
    Consumer: isolation.level=read_committed + 수동 커밋

실수 2: 처리 전에 offset 커밋 (의도치 않은 At-Most-Once)

  코드:
    ConsumerRecords<K,V> records = consumer.poll(Duration.ofMillis(100));
    consumer.commitSync();         // ← 처리 전에 커밋!
    for (ConsumerRecord<K,V> r : records) {
        processMessage(r);         // 여기서 크래시
    }
  
  결과:
    offset은 커밋됐지만 processMessage 미실행
    재시작 후 해당 메시지 다시 읽지 않음 → 유실

실수 3: At-Least-Once에서 중복 처리 로직 없음

  코드:
    for (ConsumerRecord<K,V> r : records) {
        dbInsert(r.key(), r.value());  // 멱등하지 않은 INSERT
    }
    consumer.commitSync();

  문제:
    처리 후 commitSync 전에 크래시
    → 재시작 후 같은 메시지 재처리
    → dbInsert가 INSERT면 중복 레코드 생성
    해결: INSERT → UPSERT (ON CONFLICT DO UPDATE)로 멱등하게
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
보장 단계별 선택 기준:

  At-Most-Once (유실 허용, 중복 없음):
    사용: 실시간 알림, 광고 노출 카운터, 오류 허용 메트릭
    구현: 처리 전 offset 커밋 (또는 acks=0)
    비용: 최소

  At-Least-Once (중복 허용, 유실 없음):
    사용: 이메일 발송, 재처리 가능한 비즈니스 로직
    구현:
      Producer: acks=all + retries
      Consumer: 처리 후 수동 커밋 + 멱등 처리 로직
    비용: 중간
    실무 기본: 대부분의 서비스에 적합

  Exactly-Once (중복 없음, 유실 없음):
    사용: 결제, 잔액 변경, 이벤트 소싱
    구현:
      Producer: enable.idempotence=true + transactional.id
      Consumer: isolation.level=read_committed + 트랜잭션 내 커밋
    비용: 높음 (처리량 약 20~30% 감소, 구현 복잡성)

멱등 처리로 At-Least-Once를 Exactly-Once처럼 만들기:
  중복 메시지가 와도 결과가 같으면 사실상 Exactly-Once와 동일
  구현: UPSERT / 중복 키 체크 / 처리 여부 테이블
  → Kafka 트랜잭션 없이도 달성 가능 (구현이 더 단순할 수 있음)
```

---

## 🔬 내부 동작 원리

### 1. At-Most-Once: 유실 가능, 중복 없음

```
Producer 쪽 At-Most-Once:
  acks=0 설정
  → 응답 확인 없이 전송 → 브로커 미수신 시 유실
  → 재시도 없음 → 중복 없음

Consumer 쪽 At-Most-Once:
  poll() → commitSync() → 처리()

  타임라인:
  t1: record = poll()           [offset 100 수신]
  t2: commitSync()              [offset 101 커밋 완료]
  t3: processMessage() → crash  [처리 중 장애]
  t4: 재시작 → poll()            [offset 101부터 시작]
  → offset 100 메시지는 영영 처리되지 않음 (유실)

  적합한 케이스:
    로그 스트리밍 집계 (일부 유실 허용)
    실시간 대시보드 (오래된 데이터는 무의미)
```

### 2. At-Least-Once: 유실 없음, 중복 가능

```
Producer 쪽 At-Least-Once:
  acks=all + retries=Integer.MAX_VALUE
  → 성공 응답 전까지 재시도 → 유실 없음
  → 재시도로 인한 중복 가능 (멱등성 없으면)

Consumer 쪽 At-Least-Once:
  poll() → 처리() → commitSync()

  타임라인:
  t1: record = poll()           [offset 100 수신]
  t2: processMessage()          [처리 완료]
  t3: commitSync() → crash 전  [crash!]
  t4: 재시작 → poll()            [offset 100부터 다시 시작]
  t5: processMessage()          [offset 100 다시 처리 → 중복!]

  중복 제거 전략:
    1. 멱등 처리: UPSERT, 조건부 업데이트
    2. 중복 키 테이블: processed_message_id
    3. Kafka 트랜잭션 (Exactly-Once로 격상)

  실무 기본:
    대부분의 비즈니스 로직은 멱등하게 설계 가능
    → At-Least-Once + 멱등 처리가 가장 실용적 선택
```

### 3. Exactly-Once: 유실 없음, 중복 없음

```
Exactly-Once Semantics(EOS) 달성 조건:

  Producer 쪽:
    enable.idempotence=true   → 단일 파티션 중복 제거
    transactional.id=...      → 멀티 파티션 원자적 쓰기

  Consumer 쪽:
    isolation.level=read_committed → 커밋된 메시지만 읽기

  Read-Process-Write 전체 패턴:
    1. Consumer: isolation.level=read_committed로 Fetch
    2. 처리 로직 실행
    3. Producer 트랜잭션 시작
    4. 결과를 output 파티션에 send()
    5. 해당 Consumer의 offset을 트랜잭션에 포함
       (sendOffsetsToTransaction())
    6. commitTransaction()
    → 처리 + 결과 발행 + offset 커밋이 원자적으로 완료
    → 중간에 크래시해도 트랜잭션 abort → 재시작 후 재처리 → 중복 없음

비용:
  Producer: 트랜잭션 코디네이터 왕복 ~2~3배 지연
  Consumer: read_committed = abort된 트랜잭션 메시지 건너뜀
  처리량: ~20~30% 감소 (워크로드에 따라 다름)
```

### 4. 보장 단계 결정 매트릭스

```
최종 전달 보장 = min(Producer 보장, Consumer 보장)

Producer 쪽:
  acks=0                 → At-Most-Once 가능성 있음
  acks=1, retries=0      → At-Most-Once (재시도 없음)
  acks=all, retries>0    → At-Least-Once
  idempotence=true       → At-Least-Once (단일 파티션 중복 제거)
  transactional.id       → Exactly-Once (멀티 파티션 원자적)

Consumer 쪽:
  커밋 후 처리            → At-Most-Once
  처리 후 커밋 (자동/수동) → At-Least-Once (재처리 시 중복)
  트랜잭션 내 커밋         → Exactly-Once (처리+커밋 원자적)

예시 조합:
  acks=all + 처리 후 수동 커밋 + 멱등 처리
  = At-Least-Once + 멱등 = 실질적 Exactly-Once 효과
  = 가장 실용적인 운영 패턴
```

---

## 💻 실전 실험

### 실험 1: At-Least-Once 중복 재현

```bash
# 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic at-least-once-test \
  --partitions 1 --replication-factor 1

# 메시지 10개 발행
for i in $(seq 1 10); do
  echo "msg-$i"
done | kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic at-least-once-test

# Consumer 실행 후 강제 종료 (처리 후 커밋 전 시뮬레이션)
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic at-least-once-test \
  --group test-group \
  --max-messages 5    # 5개 처리 후 종료 (커밋은 됐을 수도 있음)

# offset 상태 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group test-group
# CURRENT-OFFSET vs LOG-END-OFFSET 확인

# 재시작 → 중복 처리되는 메시지 범위 확인
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic at-least-once-test \
  --group test-group
```

### 실험 2: 수동 커밋으로 At-Least-Once 구현

```java
// Spring Kafka 수동 커밋 설정
@Configuration
public class KafkaConsumerConfig {
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> factory() {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}

@KafkaListener(topics = "orders")
public void consume(ConsumerRecord<String, String> record, Acknowledgment ack) {
    try {
        processOrder(record.value());  // 처리 먼저
        ack.acknowledge();             // 성공 후 커밋
    } catch (Exception e) {
        // 커밋 안 함 → 재처리
        log.error("처리 실패, 재처리 예정: {}", record.offset());
    }
}
```

### 실험 3: offset 커밋 타이밍별 동작 비교

```bash
# auto.commit.interval.ms=100ms로 설정 (빈번한 자동 커밋)
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic at-least-once-test \
  --group auto-commit-group \
  --consumer-property enable.auto.commit=true \
  --consumer-property auto.commit.interval.ms=100

# 중간에 Ctrl+C로 종료 후 재시작
# → 자동 커밋 시점에 따라 중복 또는 유실 발생
```

---

## 📊 성능/비용 비교

### 전달 보장 단계별 처리량 비교

```
토픽: 파티션 3개, 복제 팩터 3, 1 KB 메시지

At-Most-Once:
  Producer acks=0, Consumer 커밋 후 처리
  처리량: ~600,000 msg/sec
  지연(p99): ~2 ms
  비용: 최소 / 유실 위험 상시

At-Least-Once:
  Producer acks=all + retries, Consumer 처리 후 수동 커밋
  처리량: ~200,000 msg/sec
  지연(p99): ~15 ms
  비용: 중간 / 멱등 처리 로직 필요

Exactly-Once:
  Producer transactional + Consumer read_committed
  처리량: ~130,000 msg/sec
  지연(p99): ~30 ms
  비용: 높음 / 구현 복잡성 증가

결론:
  At-Most-Once 대비 At-Least-Once: 처리량 ~67% 감소
  At-Least-Once 대비 Exactly-Once: 처리량 추가 ~35% 감소
  대부분 At-Least-Once + 멱등 처리로 충분
```

---

## ⚖️ 트레이드오프

```
At-Most-Once:
  ✅ 최고 처리량, 최저 지연
  ✅ 구현 단순
  ❌ 유실 가능 → 비즈니스 중요 데이터 부적합

At-Least-Once:
  ✅ 유실 없음
  ✅ 구현 비교적 단순 (처리 후 커밋)
  ✅ 멱등 처리 조합 시 Exactly-Once 효과
  ❌ 중복 처리 가능 → 처리 로직이 멱등해야 함
  → 실무 기본 전략

Exactly-Once:
  ✅ 유실 없음 + 중복 없음
  ❌ 처리량 감소 (~30%)
  ❌ 구현 복잡 (트랜잭션 관리)
  ❌ 트랜잭션 코디네이터 단일 장애점
  ❌ Consumer와 Producer가 같은 Kafka 클러스터여야 함 (외부 DB 연계 불가)
  → 처리 로직을 멱등하게 만들기 어려운 경우에만 선택
```

---

## 📌 핵심 정리

```
전달 보장 3단계:

  At-Most-Once  = 처리 전 커밋 → 크래시 시 유실, 중복 없음
  At-Least-Once = 처리 후 커밋 → 크래시 시 재처리, 중복 가능
  Exactly-Once  = 처리 + 커밋 원자적 → 유실 없음, 중복 없음

최종 보장 = Producer 보장 × Consumer 보장
  acks=all + 처리 후 커밋 = At-Least-Once
  + 멱등 처리 = 실질적 Exactly-Once
  + 트랜잭션 = 공식 Exactly-Once

실무 선택:
  대부분: At-Least-Once + 멱등 처리 (비용 대비 최적)
  결제/잔액: Exactly-Once (트랜잭션 사용)
  로그/메트릭: At-Most-Once (유실 허용)
```

---

## 🤔 생각해볼 문제

**Q1. "처리 후 커밋"이 At-Least-Once인 이유는 무엇인가요? "커밋 후 처리"와 무엇이 다른가요?**

<details>
<summary>해설 보기</summary>

커밋은 "이 offset까지 처리했음을 Kafka에 알리는 행위"입니다.

**처리 후 커밋(At-Least-Once)**: 처리 완료 → 커밋 순서. 처리는 완료됐지만 커밋 전 크래시 → 재시작 후 같은 offset부터 다시 읽음 → 이미 처리한 메시지를 또 처리 → **중복**. 단, 커밋 전에 처리가 완료됐으므로 **유실 없음**.

**커밋 후 처리(At-Most-Once)**: 커밋 → 처리 순서. 커밋은 됐지만 처리 전 크래시 → 재시작 후 커밋된 다음 offset부터 읽음 → 커밋은 됐지만 처리 안 된 메시지는 영영 처리되지 않음 → **유실**. 단, 재처리가 없으므로 **중복 없음**.

간단히: "커밋 = 처리 완료 선언"이기 때문에 커밋 전에 처리가 완료돼야 안전합니다.

</details>

---

**Q2. At-Least-Once에서 멱등 처리를 구현했다면 Exactly-Once와 같은가요?**

<details>
<summary>해설 보기</summary>

결과적으로는 같지만 보장의 메커니즘이 다릅니다.

**멱등 처리**: 애플리케이션 레벨에서 "같은 메시지가 여러 번 와도 결과가 같다"를 구현. 예를 들어 DB UPSERT를 사용하면 같은 key로 여러 번 처리해도 최종 상태는 동일합니다.

**Exactly-Once(Kafka 트랜잭션)**: Kafka 레벨에서 "메시지가 정확히 한 번만 처리됨"을 보장. 트랜잭션 내에서 처리 결과 발행과 offset 커밋이 원자적으로 이루어집니다.

차이점: 멱등 처리는 "여러 번 처리해도 괜찮다"이고, Exactly-Once는 "한 번만 처리된다"입니다. 처리 비용이 비싼 연산(외부 API 호출, 이메일 발송 등)은 멱등화가 어렵거나 부작용이 있을 수 있어 이 경우 진짜 Exactly-Once가 필요합니다.

</details>

---

**Q3. Kafka Exactly-Once는 Kafka 내부에서만 보장됩니다. 외부 DB에 저장하는 경우는 어떻게 해야 하나요?**

<details>
<summary>해설 보기</summary>

Kafka 트랜잭션은 Kafka 토픽 간의 원자성만 보장합니다. 외부 DB(MySQL, PostgreSQL 등)에 저장하는 경우 Kafka 트랜잭션이 DB 트랜잭션을 포함하지 않으므로 완전한 Exactly-Once가 보장되지 않습니다.

이 경우 사용하는 패턴:
1. **멱등 저장**: DB UPSERT + 메시지 ID 기반 중복 체크 테이블
2. **Outbox Pattern**: DB 트랜잭션 내에서 비즈니스 데이터 + outbox 테이블에 함께 저장 → CDC(Debezium)로 Kafka 발행 → 원자성 보장
3. **Two-Phase Commit (XA)**: DB와 Kafka 모두 XA 트랜잭션 지원 필요 (복잡하고 성능 저하)

실무에서는 **멱등 저장(UPSERT + 처리 여부 테이블)**이 가장 실용적인 선택입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Producer 멱등성 ➡️](./02-producer-idempotence.md)**

</div>
