# Consumer Offset 관리 — 자동 커밋 vs 수동 커밋

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `enable.auto.commit=true`가 중복과 유실을 동시에 유발할 수 있는 시나리오는?
- `commitSync`와 `commitAsync`의 차이와 각각의 적합한 사용 시점은?
- `__consumer_offsets` 토픽에 offset이 어떤 형태로 저장되는가?
- 파티션별로 offset을 세밀하게 커밋하는 방법은?
- Consumer 재시작 후 offset이 어느 시점부터 재개되는가?
- `auto.offset.reset`과 수동 offset 리셋의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka Consumer는 메시지를 처리하고 나서 "어디까지 처리했다"는 offset을 Kafka에 기록한다. 이 커밋 타이밍이 At-Most-Once와 At-Least-Once의 경계를 결정한다.

자동 커밋이 활성화된 상태에서 아무 생각 없이 쓰면:
- 처리 중 크래시 → 재시작 후 자동 커밋된 offset 이후부터 읽음 → 유실
- 또는 자동 커밋 타이밍에 따라 이미 처리한 메시지를 다시 읽음 → 중복

Spring Kafka의 `@KafkaListener`에서 `ackMode`를 이해하지 못하면 의도치 않은 보장 단계로 동작한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 자동 커밋 + 오래 걸리는 처리

  설정:
    enable.auto.commit=true
    auto.commit.interval.ms=5000  (5초마다 자동 커밋)

  타임라인:
    t=0:    poll() → 100개 레코드 수신 (offset 200~299)
    t=2:    레코드 50개 처리 완료 (offset 200~249)
    t=5:    auto commit 발동 → offset 300으로 커밋
            (다음 poll에서 가져올 위치 = 300)
    t=7:    레코드 처리 중 크래시
    t=재시작: poll() → offset 300부터 시작
    → offset 250~299의 메시지는 커밋됐지만 처리 안 됨 → 유실!

실수 2: commitAsync만 사용 + 오류 무시

  코드:
    for (ConsumerRecord r : records) {
        process(r);
    }
    consumer.commitAsync();  // 비동기 커밋, 실패 무시

  문제:
    commitAsync 실패 시 콜백에서 처리하지 않으면
    커밋 없이 다음 poll() → 처리 완료했는데 offset 미커밋
    크래시 후 재시작 → 이미 처리한 메시지 재처리 (중복)
    
  올바른 사용:
    consumer.commitAsync((offsets, exception) -> {
        if (exception != null) {
            log.error("커밋 실패: {}", offsets, exception);
            // 재시도 또는 메트릭 기록
        }
    });

실수 3: 배치 처리에서 commitSync를 레코드마다 호출

  코드:
    for (ConsumerRecord r : records) {
        process(r);
        consumer.commitSync();  // 매번 커밋 (브로커 왕복)
    }

  문제:
    100개 레코드 → 100번 브로커 왕복 → 처리량 급감
  
  올바른 방법:
    배치 처리 완료 후 1번 commitSync
    또는 파티션별 마지막 offset만 commitSync
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
수동 커밋 패턴 (At-Least-Once 표준):

  // Spring Kafka: AckMode.MANUAL_IMMEDIATE
  @KafkaListener(topics = "orders")
  public void consume(ConsumerRecord<K,V> record, Acknowledgment ack) {
      try {
          processOrder(record.value());
          ack.acknowledge();   // 처리 후 커밋
      } catch (Exception e) {
          // 커밋 안 하면 재처리
          log.error("처리 실패: offset={}", record.offset(), e);
          // DLT로 보내거나 재시도 후 ack
      }
  }

배치 커밋 패턴 (처리량 최적화):

  // AckMode.BATCH: 배치 전체 처리 후 1번 커밋
  @KafkaListener(topics = "orders",
                 containerFactory = "batchFactory")
  public void consumeBatch(List<ConsumerRecord<K,V>> records,
                           Acknowledgment ack) {
      records.forEach(r -> processOrder(r.value()));
      ack.acknowledge();  // 배치 완료 후 1번
  }

파티션별 세밀한 커밋:

  Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
  for (ConsumerRecord<K,V> record : records) {
      processRecord(record);
      // 파티션별 최신 offset+1 업데이트
      offsets.put(
          new TopicPartition(record.topic(), record.partition()),
          new OffsetAndMetadata(record.offset() + 1)
      );
  }
  consumer.commitSync(offsets);  // 배치 완료 후 파티션별 커밋
```

---

## 🔬 내부 동작 원리

### 1. __consumer_offsets 토픽 구조

```
__consumer_offsets 토픽:
  파티션 50개, cleanup.policy=compact

  저장 형태:
    key: [GroupId][Topic][Partition]
    value: OffsetAndMetadata {
        offset: 300,           ← 다음 poll에서 시작할 offset
        leaderEpoch: 5,        ← Leader Epoch
        metadata: "",          ← 사용자 메타데이터 (선택)
        commitTimestamp: ...,
        expireTimestamp: ...
    }

  예시:
    key=["order-group"]["orders"][0]
    value={offset: 300}
    → order-group의 orders 토픽 Partition 0은 offset 299까지 처리 완료
       다음 poll()은 offset 300부터 시작

  커밋 단위:
    commitSync() / commitAsync() → 현재 poll()에서 받은 모든 파티션의 마지막 offset+1
    commitSync(offsets) → 지정한 파티션-offset 쌍만 커밋

  Group Coordinator:
    각 Consumer Group은 특정 브로커의 __consumer_offsets 파티션이 담당
    hash(groupId) % 50 → 파티션 N → 해당 파티션 Leader = Group Coordinator
```

### 2. enable.auto.commit의 정확한 동작

```
자동 커밋 타이밍:
  poll() 호출 시 마지막 poll() 이후 auto.commit.interval.ms 경과했으면 커밋
  (별도 스레드가 아닌 poll() 내부에서 실행)

  타임라인:
    t=0:    poll() → 100개 수신 (internal: 커밋 시각 체크 → 최초라 스킵)
    t=5:    poll() → 이전 poll 이후 5초 경과 → offset 커밋 실행!
                  → 그 다음 새 레코드 Fetch

  핵심 주의:
    poll()을 호출해야만 자동 커밋이 일어남
    poll() 호출 없이 처리가 오래 걸리면 → 커밋도 안 됨
    → max.poll.interval.ms 초과 시 리밸런싱 + 자동 커밋도 지연

  enable.auto.commit=true의 위험:
    poll() 타이밍과 처리 완료 타이밍이 맞지 않으면
    유실(처리 전 커밋) 또는 중복(처리 후 미커밋)이 발생
    → 정밀한 보장이 필요하면 수동 커밋 필수
```

### 3. commitSync vs commitAsync

```
commitSync():
  동기 블로킹 커밋
  브로커에 커밋 요청 → 응답 대기 → 반환
  
  실패 시: 자동 재시도 (기본 동작)
  응답 시간: ~수 ms (브로커 왕복 1회)
  
  적합한 경우:
    배치 처리 완료 후 1번 커밋
    Consumer 종료 전 최종 커밋 (close() 전)
    재시도가 필요한 중요 커밋

commitAsync(callback):
  비동기 논블로킹 커밋
  브로커에 커밋 요청 → 즉시 반환 → 나중에 콜백 호출
  
  실패 시: 자동 재시도 없음 (콜백에서 처리 필요)
  이유: 재시도 중 더 최신 커밋이 성공한 경우 이전 커밋 재시도는 무의미
       (오래된 offset으로 덮어쓰기 → 더 많은 중복 발생)
  
  적합한 경우:
    처리 중간중간 빈번한 커밋 (처리 중단점 기록)
    처리량이 중요한 환경

조합 패턴 (권장):
  try {
      while (running) {
          records = consumer.poll(Duration.ofMillis(100));
          process(records);
          consumer.commitAsync();   // 평소에는 비동기
      }
  } finally {
      consumer.commitSync();        // 종료 시 동기 보장
  }
```

### 4. 리밸런싱 전 offset 커밋

```
ConsumerRebalanceListener 활용:

  class SaveOffsetOnRebalance implements ConsumerRebalanceListener {
      @Override
      public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
          // 파티션 반환 전 마지막 처리 offset 커밋
          consumer.commitSync(currentOffsets);
          // 현재까지 처리한 offset을 정확히 커밋
          // → 새 Consumer가 올바른 지점부터 시작
      }

      @Override
      public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
          // 새 파티션 할당 받음 → 초기화 작업
      }
  }

  consumer.subscribe(topics, new SaveOffsetOnRebalance());

  이것이 왜 중요한가:
    리밸런싱 직전에 처리한 메시지의 offset이 커밋 안 됐으면
    → 새 Consumer가 이미 처리된 offset부터 다시 읽음 → 중복
    onPartitionsRevoked에서 커밋하면 정확한 지점 전달 가능
```

### 5. 파티션별 독립 커밋

```
파티션별 세밀한 offset 추적이 필요한 경우:

  // 파티션별 처리된 마지막 offset 추적
  Map<TopicPartition, OffsetAndMetadata> processed = new HashMap<>();

  for (ConsumerRecord<K,V> record : records) {
      TopicPartition tp = new TopicPartition(record.topic(), record.partition());
      
      try {
          process(record);
          // 처리 성공 시 해당 파티션 offset 업데이트
          processed.put(tp, new OffsetAndMetadata(record.offset() + 1));
      } catch (Exception e) {
          // 처리 실패 시 해당 파티션 건너뜀
          log.error("파티션 {}, offset {} 처리 실패", tp, record.offset());
          // 실패한 파티션은 현재 offset 유지 → 재처리 예정
      }
  }

  // 성공한 파티션만 커밋
  if (!processed.isEmpty()) {
      consumer.commitSync(processed);
  }

  활용: 파티션 A는 성공, 파티션 B는 실패 → A만 커밋
        재시작 후 B의 실패 offset부터 재처리 (A의 성공분은 중복 없음)
```

---

## 💻 실전 실험

### 실험 1: 자동 커밋과 수동 커밋 동작 비교

```bash
# 토픽 생성 및 메시지 발행
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic commit-test --partitions 1 --replication-factor 1

for i in $(seq 1 30); do echo "msg-$i"; done | \
  kafka-console-producer --bootstrap-server localhost:19092 \
  --topic commit-test

# 자동 커밋 Consumer (auto.commit.interval.ms=1000)
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic commit-test \
  --group auto-group \
  --consumer-property enable.auto.commit=true \
  --consumer-property auto.commit.interval.ms=1000 \
  --max-messages 10

# 5초 대기 (자동 커밋 발생)
sleep 5

# 현재 커밋 상태 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group auto-group
# CURRENT-OFFSET vs LOG-END-OFFSET 비교
```

### 실험 2: 수동 커밋 offset 직접 확인

```bash
# offset 수동 리셋 (재처리 시뮬레이션)
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --group auto-group \
  --topic commit-test \
  --reset-offsets --to-earliest --execute

# 처음부터 재처리
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group auto-group
# CURRENT-OFFSET=0 확인
```

### 실험 3: __consumer_offsets 내용 직접 확인

```bash
# 특정 group의 offset 저장 내용 확인
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic __consumer_offsets \
  --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" \
  --from-beginning 2>/dev/null | grep "auto-group"
# [auto-group,commit-test,0]::OffsetAndMetadata(offset=10, ...)
```

---

## 📊 성능/비용 비교

### 커밋 전략별 처리량 영향

```
조건: 1000개 레코드 배치 처리

레코드마다 commitSync:
  1000번 브로커 왕복
  처리량: ~10,000 msg/sec (커밋 오버헤드 지배적)
  지연: 높음

배치 완료 후 commitSync (권장):
  1번 브로커 왕복
  처리량: ~200,000 msg/sec
  지연: 낮음 (브로커 1회 왕복)

commitAsync (빈번한 중간 커밋):
  비동기 → 처리 블로킹 없음
  처리량: ~200,000 msg/sec
  지연: 최소 (응답 기다리지 않음)
  리스크: 실패 시 재시도 없음 → 콜백 처리 필수

auto.commit (기본):
  poll() 호출 시 주기적 커밋
  처리량: 높음 (비동기와 유사)
  리스크: 처리 완료 전 커밋 가능 → 유실/중복 위험
```

---

## ⚖️ 트레이드오프

```
enable.auto.commit=true:
  ✅ 구현 단순
  ❌ 커밋 타이밍 제어 불가 → 유실/중복 위험
  ❌ 처리 완료와 커밋이 분리됨

commitSync (수동):
  ✅ 처리 완료 후 정확한 커밋 → At-Least-Once 보장
  ✅ 실패 시 자동 재시도
  ❌ 브로커 응답 대기 → 약간의 처리량 감소
  → 배치 단위로 호출하면 영향 최소화

commitAsync (수동):
  ✅ 블로킹 없음 → 최고 처리량
  ❌ 실패 시 자동 재시도 없음 → 콜백 필수
  ❌ 재시도 로직 구현 복잡
  → 종료 전 commitSync와 조합 필수

파티션별 커밋:
  ✅ 실패한 파티션만 재처리 → 중복 최소화
  ❌ 구현 복잡도 증가
  → 파티션별 처리 실패 가능성 있는 경우 필요
```

---

## 📌 핵심 정리

```
Consumer Offset 관리 핵심:

1. offset 커밋 = "여기까지 처리 완료" 선언
   처리 전 커밋 → At-Most-Once (유실)
   처리 후 커밋 → At-Least-Once (중복)

2. __consumer_offsets 토픽에 저장
   key=[GroupId][Topic][Partition], value=offset+1
   cleanup.policy=compact → 최신 커밋만 보존

3. 자동 커밋 함정:
   처리 완료와 커밋 타이밍 불일치 → 유실/중복 가능
   정밀한 보장이 필요하면 수동 커밋 필수

4. commitSync vs commitAsync:
   commitSync: 응답 대기, 실패 시 재시도 → 안전, 약간 느림
   commitAsync: 응답 안 기다림, 실패 콜백 필요 → 빠름
   조합: 평소 commitAsync + 종료 시 commitSync

5. Spring Kafka AckMode:
   RECORD: 레코드마다 커밋 (느림)
   BATCH: 배치 완료 후 커밋 (권장)
   MANUAL: 명시적 ack.acknowledge() 필요
   MANUAL_IMMEDIATE: 즉시 커밋 (세밀한 제어)
```

---

## 🤔 생각해볼 문제

**Q1. `commitSync()`가 실패하면 어떻게 되나요? 무한 재시도하나요?**

<details>
<summary>해설 보기</summary>

`commitSync()`는 재시도 가능한 오류(예: 일시적 네트워크 오류)는 내부적으로 재시도합니다. 재시도 불가한 오류(예: `CommitFailedException` - 리밸런싱으로 해당 파티션을 더 이상 소유하지 않음)는 즉시 예외를 던집니다.

가장 흔한 실패 원인은 `CommitFailedException`입니다. 이것은 처리 시간이 너무 오래 걸려서 `max.poll.interval.ms`를 초과하고 리밸런싱이 발생했을 때 나타납니다. 파티션이 다른 Consumer에게 넘어간 상태에서 커밋을 시도하면 실패합니다.

해결: `max.poll.records`를 줄이거나 `max.poll.interval.ms`를 늘려서 처리 시간 내에 커밋이 가능하게 합니다.

</details>

---

**Q2. Consumer가 처리한 레코드가 1000개인데 첫 500개는 성공, 나머지 500개는 실패했습니다. offset 커밋을 어떻게 해야 할까요?**

<details>
<summary>해설 보기</summary>

파티션 구조에 따라 다르지만, 가장 안전한 방법은 **500번째 레코드까지의 파티션-offset만 커밋**하는 것입니다.

파티션별로 처리된 마지막 성공 offset을 추적하고 `commitSync(Map<TopicPartition, OffsetAndMetadata>)`로 성공한 부분만 커밋합니다. 실패한 레코드의 파티션은 커밋하지 않으면 재시작 후 해당 파티션의 마지막 커밋 offset부터 재처리합니다.

단, 파티션 내에서는 순서가 있기 때문에 **같은 파티션에서 100번 성공, 101번 실패, 102~200번 성공했다면** 101번을 건너뛰고 커밋할 수 없습니다. 파티션 내에서는 offset 100까지만 커밋하고, 101번부터 재처리해야 합니다. 101번이 반복적으로 실패하면 DLT(Dead Letter Topic)로 보내고 넘어가는 패턴을 사용합니다.

</details>

---

**Q3. `auto.offset.reset=latest`로 설정한 Consumer Group이 며칠 동안 중단됐다가 재시작하면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

`auto.offset.reset`은 offset 정보가 없을 때만 적용됩니다. 이미 커밋된 offset 정보가 있으면 `auto.offset.reset` 설정은 무시하고 마지막 커밋 offset부터 재개합니다.

단, `__consumer_offsets`의 offset 정보는 `offsets.retention.minutes`(기본 7일) 동안만 보존됩니다. 7일 이상 Consumer Group이 중단됐다면 offset 정보가 삭제됩니다. 이 경우 `auto.offset.reset=latest`이면 재시작 시점부터만 읽게 되어 7일치 메시지를 건너뜁니다.

장기 중단 가능성이 있는 Consumer는 `offsets.retention.minutes`를 높이거나, 재시작 전 명시적으로 offset을 설정하는 절차가 필요합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 트랜잭션 Producer](./03-transactional-producer.md)** | **[홈으로 🏠](../README.md)** | **[다음: Exactly-Once Semantics ➡️](./05-exactly-once-semantics.md)**

</div>
