# 리밸런싱 중 중복 처리 — 멱등 처리 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 처리 완료 후 offset 커밋 전 리밸런싱이 발생하면 정확히 어떤 과정으로 중복이 발생하는가?
- `ConsumerRebalanceListener.onPartitionsRevoked()`에서 수동 커밋이 중복을 줄이는 원리는?
- `onPartitionsRevoked()`와 `onPartitionsAssigned()`는 각각 언제 호출되는가?
- DB에서 멱등 키(Idempotent Key)로 중복 처리를 무해하게 만드는 구체적 설계는?
- Cooperative 리밸런싱에서 중복이 줄어드는 이유는?
- Spring Kafka에서 리밸런싱 중복을 처리하는 실용적인 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

At-Least-Once를 선택한 이상 리밸런싱 중 중복 처리는 피할 수 없다. 하지만 중복이 발생해도 서비스가 올바르게 동작하도록 설계하면 문제가 없다.

"리밸런싱이 일어났는데 주문이 두 번 생성됐다"는 사고는 처리 로직이 멱등하지 않기 때문이다. 리밸런싱을 완전히 막을 수는 없지만, 중복 처리가 부작용을 일으키지 않도록 설계할 수 있다. 이것이 Kafka 기반 시스템 설계의 핵심 원칙이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: onPartitionsRevoked에서 커밋하지 않음

  코드:
    consumer.subscribe(topics);  // RebalanceListener 없음

  리밸런싱 시나리오:
    C1이 offset 200~299 처리 중 (offset 230까지 완료)
    리밸런싱 발생 → C1이 해당 파티션 반납
    onPartitionsRevoked 없음 → offset 230까지 처리했지만 커밋 없음
    C2가 해당 파티션 받음 → 마지막 커밋 offset(200)부터 읽기 시작
    → offset 200~230을 C2가 또 처리 (30개 중복!)

실수 2: 멱등하지 않은 처리 로직

  코드:
    @KafkaListener
    public void process(OrderEvent event) {
        orderRepository.save(event.toOrder());  // INSERT
        emailService.send(event.getUserEmail(), "주문 완료");
    }

  중복 시 문제:
    동일 주문 ID로 두 번 process() 호출
    → DB에 중복 주문 생성 (PRIMARY KEY 없으면)
    → 이메일 두 번 발송

실수 3: onPartitionsRevoked에서 긴 작업 실행

  public void onPartitionsRevoked(...) {
      waitForAllTasksComplete();  // 수 분 소요 가능
      consumer.commitSync(currentOffsets);
  }

  문제:
    onPartitionsRevoked가 오래 걸리면 rebalance.timeout.ms 초과
    → 해당 Consumer가 리밸런싱에서 제외 → 더 많은 리밸런싱
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
ConsumerRebalanceListener 패턴:

  Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();

  consumer.subscribe(topics, new ConsumerRebalanceListener() {
      @Override
      public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
          consumer.commitSync(currentOffsets);
          currentOffsets.clear();
      }

      @Override
      public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
          // 새 파티션 초기화 (상태 로드 등)
      }
  });

  // 처리 루프에서 offset 추적
  for (ConsumerRecord<K,V> record : records) {
      processRecord(record);
      currentOffsets.put(
          new TopicPartition(record.topic(), record.partition()),
          new OffsetAndMetadata(record.offset() + 1)
      );
  }

DB 멱등 설계 (UPSERT):
  INSERT INTO orders (order_id, user_id, amount, status)
  VALUES (?, ?, ?, ?)
  ON CONFLICT (order_id) DO UPDATE SET
    status = EXCLUDED.status,
    updated_at = NOW()
  WHERE orders.status = 'PENDING'

처리 이력 테이블:
  CREATE TABLE processed_messages (
    message_id VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP,
    group_id VARCHAR(255)
  );
  -- 처리 전 중복 체크
  if (processedRepo.existsById(record.key())) return;
```

---

## 🔬 내부 동작 원리

### 1. 리밸런싱 중 중복 발생 상세 과정

```
시나리오: C1이 파티션 0 처리 중, 리밸런싱 발생

  초기 상태:
    파티션 0 마지막 커밋 offset: 200
    C1 처리 완료: offset 200~249 (50개)
    C1 처리 중: offset 250~299 (50개)

  onPartitionsRevoked 없는 경우:
    C1: 처리 중단, 파티션 0 반납, 커밋 없음
    C2: offset 200부터 읽기 → offset 200~249 재처리 (50개 중복!)

  onPartitionsRevoked에서 commitSync(offset=250):
    C1: currentOffsets(250) commitSync → 반납
    C2: offset 250부터 읽기 → offset 250~299 처리
    → C1의 미처리 부분을 C2가 처리 (중복 없음)
```

### 2. onPartitionsRevoked 호출 타이밍 (Eager vs Cooperative)

```
Eager 리밸런싱:
  onPartitionsRevoked([P0, P1]) → 유지될 P0 포함!
  JoinGroup → SyncGroup
  onPartitionsAssigned([P0])  (P0 재할당)

  주의: revoked에는 유지될 파티션도 포함
        P0, P1 모두에 대해 커밋 실행

Cooperative 리밸런싱:
  Round 1: onPartitionsRevoked([P1])  ← 이동 대상만
            P0은 revoke/assign 없이 계속 처리
  Round 2: C2 onPartitionsAssigned([P1])

  → 유지 파티션(P0)은 전체 과정에서 중단 없음
  → revoke된 파티션(P1)만 중복 가능 범위
```

### 3. 멱등 처리 패턴 3단계

```
Level 1: DB UPSERT (단순, 데이터 저장 케이스)

  대상: 단순 데이터 저장 (주문, 사용자 정보 등)
  구현: ON CONFLICT DO UPDATE / INSERT IGNORE
  한계: 이메일 발송, 외부 API처럼 부작용 있는 경우 불가

Level 2: 처리 이력 테이블 (외부 부작용 방지)

  흐름:
    1. processed_messages에서 message_id 조회
    2. 없으면 처리 실행
    3. DB 트랜잭션으로 처리 결과 + 이력 동시 기록
    4. 있으면 skip

  BEGIN;
    INSERT INTO orders ...;            -- 처리
    INSERT INTO processed_messages ...; -- 이력
  COMMIT;

Level 3: Kafka 트랜잭션 EOS (완전한 보장)

  Producer transactional.id + Consumer isolation.level=read_committed
  처리 + offset 커밋 원자적 → Kafka 레벨 중복 완전 차단
  단, 외부 시스템은 여전히 Level 1/2 필요
```

### 4. Spring Kafka ConsumerSeekAware 패턴

```java
@Component
public class OrderConsumer implements ConsumerSeekAware {
    private final Map<TopicPartition, Long> offsets = new ConcurrentHashMap<>();

    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        Map<TopicPartition, OffsetAndMetadata> toCommit = new HashMap<>();
        for (TopicPartition tp : partitions) {
            Long offset = offsets.get(tp);
            if (offset != null) {
                toCommit.put(tp, new OffsetAndMetadata(offset));
            }
        }
        // 반납 전 커밋
        if (!toCommit.isEmpty()) {
            // Spring Kafka는 내부 consumer 접근 제공
            // KafkaListenerEndpointRegistry로 접근
        }
    }

    @KafkaListener(topics = "orders")
    public void consume(ConsumerRecord<String, String> record,
                        Acknowledgment ack) {
        processOrder(record.value());  // 멱등 처리
        offsets.put(
            new TopicPartition(record.topic(), record.partition()),
            record.offset() + 1
        );
        ack.acknowledge();
    }
}
```

### 5. 중복 처리가 어려운 케이스별 전략

```
이메일 발송:
  email_sent_log 테이블 (message_id, email_type) → 중복 체크
  발송 성공 후 이력 기록 (트랜잭션)
  멱등 API 지원 서비스(SendGrid 등) → Idempotency-Key 헤더 활용

외부 결제 API:
  요청 시 고유 멱등 키 포함 (order_id + attempt_count)
  동일 키로 재요청 시 결제 API가 중복 처리 방지

상태 전이 (주문 상태 변경):
  현재 상태 체크 후 조건부 업데이트
  UPDATE orders SET status='PAID'
  WHERE order_id=? AND status='PENDING'
  → status가 이미 PAID이면 UPDATE 0건 (안전)
```

---

## 💻 실전 실험

### 실험 1: onPartitionsRevoked 커밋 효과 확인

```bash
# 메시지 발행
for i in $(seq 1 100); do
  echo "order-$i"
done | kafka-console-producer \
  --bootstrap-server localhost:19092 --topic orders

# Consumer 실행 중 강제 리밸런싱 (Consumer 2 추가)
# 1) RebalanceListener 없음: 마지막 커밋 offset부터 중복
# 2) onPartitionsRevoked에서 commitSync: 중복 0

kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group test-group
# 리밸런싱 전후 CURRENT-OFFSET 비교
```

### 실험 2: DB UPSERT 중복 방지 확인

```sql
-- 중복 방지 테이블
CREATE TABLE orders (
    order_id VARCHAR(255) PRIMARY KEY,
    amount DECIMAL(10,2),
    status VARCHAR(50)
);

-- 첫 번째 처리
INSERT INTO orders VALUES ('order-123', 100.00, 'CREATED')
ON CONFLICT (order_id) DO NOTHING;
-- 1 row inserted

-- 중복 처리 (리밸런싱 후 재처리)
INSERT INTO orders VALUES ('order-123', 100.00, 'CREATED')
ON CONFLICT (order_id) DO NOTHING;
-- 0 rows inserted (안전하게 무시)
```

### 실험 3: 리밸런싱 중 처리 흐름 로그 확인

```bash
# Consumer 로그에서 revoke/assign 패턴 확인
grep -E "Revoked|Assigned|commitSync|Rebalance" consumer.log

# 예상 패턴:
# Rebalance triggered
# Revoked partitions: [orders-0, orders-1]
# commitSync: {orders-0=offset250, orders-1=offset180}
# Assigned partitions: [orders-0]
# Fetching from offset 250
```

---

## 📊 성능/비용 비교

### 리밸런싱 발생 시 중복 처리 수 비교

```
조건: 파티션 1개, 100개 레코드 처리 중 (50개 완료, 50개 미완료)
리밸런싱 발생으로 파티션 재배정

  onPartitionsRevoked 커밋 없음:
    마지막 커밋 offset부터 재처리
    중복: 처리 완료했지만 커밋 안 된 50개

  onPartitionsRevoked에서 commitSync:
    커밋된 다음 offset부터 재처리
    중복: 0개

  Cooperative + onPartitionsRevoked 커밋:
    이동 파티션만 중단 + 커밋
    비이동 파티션: 중단 없이 계속 처리
    중복: 이동 파티션의 미완료 레코드만 (범위 최소화)
```

### 멱등 처리 방식별 성능 영향

```
UPSERT (ON CONFLICT):
  INSERT 대비 오버헤드: ~5~10% (CONFLICT 체크)
  처리량 영향: 미미

처리 이력 테이블:
  SELECT + INSERT 추가 → 레코드당 DB 왕복 1회 추가
  처리량 영향: DB 응답 시간 × 처리량 (5ms 왕복 → 200/sec 한계)
  최적화: 배치 체크 (IN 절로 여러 ID 한번에)

Kafka 트랜잭션 EOS:
  처리량: ~30% 감소
  지연: ~2배
```

---

## ⚖️ 트레이드오프

```
onPartitionsRevoked 커밋:
  ✅ 리밸런싱 중 중복 최소화
  ❌ 커밋 실패 시 중복 여전히 발생
  ❌ commitSync 시간만큼 리밸런싱 지연

DB UPSERT 멱등 처리:
  ✅ 중복이 와도 결과 일관성
  ✅ 구현 단순
  ❌ 모든 케이스 적용 불가 (이메일 발송 등)

처리 이력 테이블:
  ✅ 외부 부작용도 방지
  ✅ 감사 로그 겸용
  ❌ 조회 1회 추가 → 처리량 영향
  ❌ 이력 테이블 정리 필요

Exactly-Once:
  ✅ Kafka 레벨 완전 보장
  ❌ 처리량 30% 감소
  ❌ 외부 시스템은 별도 멱등 처리 필요
```

---

## 📌 핵심 정리

```
리밸런싱 중 중복 처리 핵심:

1. 중복 발생 원인:
   처리 완료 → offset 커밋 전 리밸런싱
   → 새 Consumer가 마지막 커밋 offset부터 재처리

2. onPartitionsRevoked에서 commitSync로 중복 최소화
   반납 전 현재까지 처리한 offset 커밋
   → 새 Consumer가 커밋된 다음 offset부터 시작

3. Cooperative 리밸런싱 + onPartitionsRevoked 조합
   이동 파티션만 revoke → 중복 가능 범위 최소화
   비이동 파티션은 전체 과정에서 중단 없음

4. 처리 로직 멱등 설계 3단계:
   Level 1: DB UPSERT (단순, 데이터 저장)
   Level 2: 처리 이력 테이블 (외부 부작용 방지)
   Level 3: Kafka 트랜잭션 EOS (높은 비용)

5. onPartitionsRevoked는 빠르게:
   commitSync만 수행 → rebalance.timeout.ms 초과 방지
```

---

## 🤔 생각해볼 문제

**Q1. `onPartitionsRevoked()`에서 `commitSync()`를 호출했는데도 중복이 발생했습니다. 어떤 경우가 가능한가요?**

<details>
<summary>해설 보기</summary>

몇 가지 경우가 있습니다.

타이밍 문제: `currentOffsets`가 실제 처리 완료한 offset보다 오래됐을 수 있습니다. 처리 루프에서 `currentOffsets`를 업데이트하지 않거나, 배치 처리 중간에 리밸런싱이 발생해서 일부 레코드의 offset이 반영되지 않았을 때 발생합니다.

commitSync 실패: 네트워크 오류로 `commitSync()`가 실패했지만 파티션은 반납됩니다. 예외 처리가 없으면 커밋 실패를 인지하지 못합니다.

Auto-commit 간섭: `enable.auto.commit=true`와 수동 커밋을 혼용하면 잘못된 offset이 커밋될 수 있습니다. 수동 커밋 시 반드시 `enable.auto.commit=false`로 설정하세요.

</details>

---

**Q2. 이메일 발송 처리에서 리밸런싱 중복을 방지하려면 어떻게 설계해야 하나요?**

<details>
<summary>해설 보기</summary>

이메일 발송은 외부 부작용이라 Kafka 레벨에서 중복을 막을 수 없습니다. 처리 이력 테이블 패턴을 사용합니다.

```
email_sent_log (message_id VARCHAR PK, email_type VARCHAR, recipient VARCHAR, sent_at TIMESTAMP)
```

처리 로직: (1) `email_sent_log`에서 `(message_id, email_type)` 존재 확인, (2) 없으면 이메일 발송 → DB 트랜잭션으로 발송 기록, (3) 있으면 건너뜀.

이메일 서비스 API가 멱등 키를 지원한다면(SendGrid 등) 해당 API의 Idempotency-Key 기능을 활용하는 것이 더 안전합니다.

</details>

---

**Q3. `onPartitionsRevoked()`에서 커밋이 느려서 `rebalance.timeout.ms`를 초과할 것 같습니다. 어떻게 처리해야 하나요?**

<details>
<summary>해설 보기</summary>

`onPartitionsRevoked()`에서는 복잡한 작업 없이 빠른 커밋만 수행하고, 처리 중인 작업은 중단 플래그로 취소합니다.

```java
private volatile boolean shouldStop = false;

@Override
public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
    shouldStop = true;
    consumer.commitSync(currentOffsets);  // 현재까지만 커밋
    shouldStop = false;
}

// 처리 루프:
for (ConsumerRecord r : records) {
    if (shouldStop) break;  // 중단 신호 감지
    processRecord(r);
    currentOffsets.put(...);
}
```

중단 시점까지 처리된 레코드의 offset만 커밋하고, 나머지는 새 Consumer에게 재처리를 맡깁니다. 처리 로직이 멱등하다면 재처리해도 문제없습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 리밸런싱 전략 비교](./03-rebalancing-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: Consumer Lag ➡️](./05-consumer-lag.md)**

</div>
