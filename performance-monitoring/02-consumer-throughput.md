# Consumer 처리량 최적화 — Fetch 튜닝

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `fetch.min.bytes`와 `fetch.max.wait.ms`가 소량 Fetch 왕복을 어떻게 줄이는가?
- `max.partition.fetch.bytes`를 줄이면 처리량이 감소하는 이유는?
- Consumer 병렬화에서 스레드 풀 방식과 인스턴스 확장의 차이는?
- `fetch.min.bytes`를 높이면 실시간성이 낮아지는 이유는?
- Consumer를 병렬화할 때 offset 커밋을 어떻게 안전하게 처리하는가?
- `max.poll.records` 증가가 처리량과 리밸런싱 위험에 미치는 영향은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Consumer 처리량은 두 구간에서 결정된다. 첫 번째는 Kafka 브로커에서 메시지를 얼마나 효율적으로 가져오는지(Fetch 효율), 두 번째는 가져온 메시지를 얼마나 빠르게 처리하는지(처리 로직 효율)다.

Fetch 설정을 모르면:
- 메시지가 적을 때 Consumer가 빈 Fetch를 초당 수백 번 날려 브로커 부하
- 반대로 Fetch를 너무 크게 설정해서 Consumer 메모리 OOM
- 처리 로직 병목을 Fetch 설정 탓으로 잘못 진단

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: fetch.min.bytes=1 (기본값)으로 고처리량 운영

  결과:
    브로커에 1바이트만 있어도 Fetch 응답
    초당 수백~수천 번 소량 Fetch
    브로커 CPU: 요청 처리 오버헤드
    Consumer: 소량 배치 처리 반복 → 처리량 저하

  올바른 설정 (배치 처리 환경):
    fetch.min.bytes=65536  (64 KB, 64개/1KB 메시지)
    fetch.max.wait.ms=500  (500ms 대기 후 강제 전송)

실수 2: max.partition.fetch.bytes를 너무 크게 설정

  설정: max.partition.fetch.bytes=104857600 (100 MB)
  Consumer 10개 파티션 할당 → 1회 poll() 최대 1 GB 수신
  결과:
    Consumer JVM 힙 OOM
    처리 지연 증가 (1 GB 데이터 처리)
  
  올바른 설정:
    max.partition.fetch.bytes=1048576 (1 MB, 기본값)
    필요하면 파티션 수 × 1MB ≤ 가용 메모리 확인

실수 3: 스레드 풀 사용 중 offset 커밋 타이밍 오류

  코드:
    records = consumer.poll(100ms);
    executor.submit(() -> processAll(records));  // 비동기 처리
    consumer.commitSync();  // 즉시 커밋 (처리 완료 전!)
  
  문제:
    비동기 처리 완료 전 커밋 → 크래시 시 유실 (At-Most-Once)
  
  올바른 방법:
    비동기 처리 완료 후 커밋
    CountDownLatch나 CompletableFuture로 완료 대기
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Fetch 최적화 설정:

  # 처리량 우선 (배치 파이프라인)
  spring:
    kafka:
      consumer:
        fetch-min-size: 65536        # 64 KB 쌓이면 Fetch
        fetch-max-wait: 500          # 최대 500ms 대기
        max-poll-records: 500        # 한 번에 500개
        properties:
          max.partition.fetch.bytes: 1048576  # 파티션당 1 MB

  # 실시간 처리 (낮은 지연)
  consumer:
    fetch-min-size: 1                # 즉시 Fetch
    fetch-max-wait: 100              # 최대 100ms 대기
    max-poll-records: 100            # 적게 받아서 빠르게 처리

병렬화 전략 선택 기준:
  처리 로직이 CPU 바운드:
    → Consumer 인스턴스 증가 (파티션 수 범위 내)
    → 각 Consumer가 독립 JVM에서 병렬 실행

  처리 로직이 I/O 바운드 (DB, API 호출):
    → 스레드 풀 (1 Consumer + N Worker 스레드)
    → Consumer poll 유지 + 처리 병렬화
    → 파티션 수 제한 없이 병렬도 향상
```

---

## 🔬 내부 동작 원리

### 1. Fetch 설정 3개의 상호작용

```
FetchRequest 전송 결정 흐름:

  Consumer → 브로커: FetchRequest
    (fetch offset, max.partition.fetch.bytes=1MB 포함)

  브로커 처리 (Long Polling):
    현재 가용 데이터 크기 확인
    
    가용 데이터 >= fetch.min.bytes(예: 64KB)?
      YES: 즉시 FetchResponse 반환
      NO:  대기...
           fetch.max.wait.ms(예: 500ms) 경과?
             YES: 데이터 부족해도 FetchResponse 반환 (빈 응답 가능)
             NO:  계속 대기

  Consumer 수신:
    FetchResponse에서 최대 max.poll.records(예: 500개) 추출
    나머지는 내부 버퍼 → 다음 poll()에서 브로커 재요청 없이 반환

시각화:
  메시지 발행 속도 낮을 때 (fetch.min.bytes=64KB):
    t=0ms:   Fetch 요청
    t=50ms:  10 KB 수신 (64 KB 미달)
    t=100ms: 20 KB 수신 (64 KB 미달)
    t=500ms: fetch.max.wait.ms 만료 → 20 KB로 응답
    Consumer: 20 KB만 처리하고 다음 Fetch

  메시지 발행 속도 높을 때:
    t=0ms:   Fetch 요청
    t=5ms:   64 KB 도달 → 즉시 응답
    Consumer: 64 KB 처리 → 즉시 다음 Fetch
    → 지연 없이 고처리량
```

### 2. max.partition.fetch.bytes와 메모리 계획

```
메모리 사용량 계산:

  Consumer가 한 번의 poll()에서 받을 수 있는 최대 데이터:
    max.partition.fetch.bytes × 할당된 파티션 수

  예시:
    max.partition.fetch.bytes=1MB, 파티션 10개 할당
    최대 수신: 1MB × 10 = 10 MB/poll()

  JVM 힙 계획:
    Consumer 처리 중 10 MB를 메모리에 보유
    + 처리 중인 배치 (파싱, 역직렬화 후 객체)
    + 처리 결과 버퍼
    총 힙: 10 MB × 3~5배 = 30~50 MB 최소

  대규모 파티션 할당 시:
    파티션 100개, max.partition.fetch.bytes=1MB
    최대 수신: 100 MB/poll()
    → Consumer 힙 OOM 위험
    → max.partition.fetch.bytes 줄이거나 파티션 수 분산

  max.fetch.bytes (전체 제한):
    max.partition.fetch.bytes × 파티션 수 합계가 이 값을 초과하지 않도록
    기본값: 50 MB → 50MB 이상 받지 않음
```

### 3. Consumer 병렬화: 인스턴스 vs 스레드 풀

```
방식 1: Consumer 인스턴스 증가 (수평 확장)

  구조:
    Consumer1 [Thread] → Poll P0 → Process P0
    Consumer2 [Thread] → Poll P1 → Process P1
    Consumer3 [Thread] → Poll P2 → Process P2

  장점:
    구현 단순 (각 Consumer 독립)
    offset 커밋 단순 (각 Consumer 자신의 파티션만)
    리밸런싱으로 자동 부하 분산

  단점:
    파티션 수가 한계 (Consumer > 파티션 → IDLE)
    Consumer당 독립 JVM/프로세스 → 리소스 오버헤드

방식 2: 스레드 풀 (단일 Consumer + Worker 스레드)

  구조:
    Consumer [Thread]
      → poll() → records → submit to ExecutorService
      → Worker1: process(record1, record4, ...)
      → Worker2: process(record2, record5, ...)
      → Worker3: process(record3, record6, ...)
      → poll() 계속 실행 (heartbeat 유지)

  장점:
    I/O 바운드 처리에서 파티션 수 제한 없이 병렬도 향상
    단일 Consumer로 여러 파티션의 메시지를 병렬 처리
    파티션이 3개여도 Worker 스레드 10개 활용 가능

  단점:
    offset 커밋 복잡 (처리 완료된 레코드만 추적해야 함)
    Worker 실패 시 처리 중인 레코드 추적 어려움
    consumer.pause()/resume()으로 배압 제어 필요

  offset 추적 구현:
    ConcurrentHashMap<TopicPartition, Long> completedOffsets
    각 Worker가 완료 시 completedOffsets 업데이트
    poll() 루프에서 주기적으로 commitSync(completedOffsets)
```

### 4. max.poll.records와 처리량 균형

```
max.poll.records 설정 기준:

  처리량 관점:
    클수록 배치 처리 효율 향상 (DB batchInsert, 배치 API 호출)
    예: 1개씩 INSERT vs 500개 batchInsert → 처리량 차이 큼

  리밸런싱 안전 관점:
    레코드당 처리 시간 × max.poll.records < max.poll.interval.ms
    
    예: 레코드당 10ms, max.poll.records=500
        500 × 10ms = 5,000ms (5초) < 300,000ms (5분) → 안전

    예: 레코드당 100ms, max.poll.records=500
        500 × 100ms = 50,000ms (50초) < 300,000ms → 안전

    예: 레코드당 1,000ms (외부 API), max.poll.records=500
        500 × 1,000ms = 500,000ms (8.3분) > 300,000ms → 리밸런싱!
        해결: max.poll.records=200, max.poll.interval.ms=300,000ms 이하 유지

  권장 계산:
    max.poll.records = max.poll.interval.ms / (레코드당 최대 처리 시간)
    예: 5분 / 100ms = 3,000개 → 여유 보정 후 2,000개
```

---

## 💻 실전 실험

### 실험 1: fetch.min.bytes 설정별 처리량 비교

```bash
# fetch.min.bytes=1 (기본값, 소량 Fetch 빈발)
kafka-consumer-perf-test \
  --bootstrap-server localhost:19092 \
  --topic perf-topic \
  --messages 1000000 \
  --group fetch-test-1 \
  --consumer-props \
    fetch.min.bytes=1 \
    fetch.max.wait.ms=500 \
    max.poll.records=500

# fetch.min.bytes=65536 (대량 Fetch)
kafka-consumer-perf-test \
  --bootstrap-server localhost:19092 \
  --topic perf-topic \
  --messages 1000000 \
  --group fetch-test-2 \
  --consumer-props \
    fetch.min.bytes=65536 \
    fetch.max.wait.ms=500 \
    max.poll.records=1000

# 결과 비교: data.consumed.in.MB/s
```

### 실험 2: 스레드 풀 Consumer 구현

```java
// 스레드 풀 Consumer (I/O 바운드 처리 최적화)
ExecutorService workers = Executors.newFixedThreadPool(10);
ConcurrentHashMap<TopicPartition, Long> pendingOffsets = new ConcurrentHashMap<>();
AtomicLong pendingCount = new AtomicLong(0);

while (running) {
    ConsumerRecords<K,V> records = consumer.poll(Duration.ofMillis(100));

    // 처리 속도 > 수신 속도 조절
    if (pendingCount.get() > 5000) {
        consumer.pause(consumer.assignment());
    } else {
        consumer.resume(consumer.assignment());
    }

    for (ConsumerRecord<K,V> record : records) {
        pendingCount.incrementAndGet();
        workers.submit(() -> {
            try {
                processRecord(record);  // I/O 바운드 처리
                pendingOffsets.merge(
                    new TopicPartition(record.topic(), record.partition()),
                    record.offset() + 1,
                    Math::max
                );
            } finally {
                pendingCount.decrementAndGet();
            }
        });
    }

    // 완료된 offset 커밋 (주기적)
    if (!pendingOffsets.isEmpty()) {
        Map<TopicPartition, OffsetAndMetadata> toCommit = new HashMap<>();
        pendingOffsets.forEach((tp, offset) ->
            toCommit.put(tp, new OffsetAndMetadata(offset)));
        consumer.commitAsync(toCommit, null);
        pendingOffsets.clear();
    }
}
```

### 실험 3: Consumer 처리량 JMX 모니터링

```bash
# Consumer 성능 지표 확인
kafka-consumer-perf-test \
  --bootstrap-server localhost:19092 \
  --topic perf-topic \
  --messages 1000000 \
  --group monitor-group \
  --print-metrics 2>&1 | grep -E "fetch-rate|records-consumed|fetch-size"

# 주요 지표:
# fetch-rate:          초당 Fetch 요청 수 (낮을수록 효율적)
# records-consumed-rate: 초당 처리 레코드 수
# fetch-size-avg:       평균 Fetch 크기 (높을수록 효율적)
```

---

## 📊 성능/비용 비교

### Fetch 설정별 처리량 비교 (1 KB 메시지)

```
fetch.min.bytes=1 (기본값):
  처리량: ~180,000 msg/sec
  Fetch 요청 수/초: ~360
  평균 Fetch 크기: ~500 KB

fetch.min.bytes=65536 (64 KB):
  처리량: ~350,000 msg/sec (+94%)
  Fetch 요청 수/초: ~180 (절반 감소)
  평균 Fetch 크기: ~1.9 MB

fetch.min.bytes=1048576 (1 MB):
  처리량: ~420,000 msg/sec (+133%)
  Fetch 요청 수/초: ~50
  평균 Fetch 크기: ~8 MB
  단점: 메시지 적을 때 500ms 대기 지연

스레드 풀 10개 + fetch.min.bytes=65536:
  처리량: ~800,000 msg/sec (I/O 바운드 처리 시)
  조건: 처리 로직이 비동기 I/O 가능한 경우
```

---

## ⚖️ 트레이드오프

```
fetch.min.bytes 증가:
  ✅ Fetch 요청 횟수 감소 → 브로커/네트워크 부하 감소
  ✅ 배치 처리 효율 향상
  ❌ 메시지 적을 때 fetch.max.wait.ms까지 대기 → 지연 증가
  → 처리량 중요한 배치 파이프라인에 적합
  → 실시간 알림처럼 지연이 중요하면 1로 유지

max.partition.fetch.bytes 증가:
  ✅ 더 많은 데이터를 한 번에 가져옴 → Fetch 효율
  ❌ Consumer 메모리 사용 증가
  ❌ 파티션 수 많으면 OOM 위험

스레드 풀 병렬화:
  ✅ I/O 바운드 처리에서 처리량 대폭 향상
  ✅ 파티션 수 제한 없이 병렬도 조정 가능
  ❌ offset 커밋 복잡도 증가
  ❌ 처리 중 Consumer 종료 시 미커밋 레코드 추적 어려움

인스턴스 확장:
  ✅ 구현 단순, offset 관리 단순
  ✅ 파티션 단위로 격리 → 장애 격리
  ❌ 파티션 수가 상한선
  ❌ 인스턴스당 리소스 오버헤드
```

---

## 📌 핵심 정리

```
Consumer 처리량 최적화 핵심:

1. Fetch 튜닝:
   fetch.min.bytes: 한 번에 가져올 최소 데이터 (기본 1 → 64KB+)
   fetch.max.wait.ms: 데이터 없을 때 최대 대기 (기본 500ms)
   두 설정의 조합이 Fetch 효율 결정

2. max.partition.fetch.bytes:
   파티션당 최대 Fetch 크기 (기본 1 MB)
   Consumer 메모리 = 파티션 수 × 이 값

3. max.poll.records:
   처리 배치 크기 결정
   = max.poll.interval.ms / 레코드당 최대 처리 시간

4. 병렬화 전략:
   CPU 바운드 → Consumer 인스턴스 증가 (파티션 수 범위)
   I/O 바운드 → 스레드 풀 (offset 커밋 주의)

5. 처리 병목 먼저 식별:
   Fetch 크기 확인 → 처리 로직 프로파일링 → 설정 조정
   (Fetch 설정보다 처리 로직 최적화가 더 효과적인 경우 많음)
```

---

## 🤔 생각해볼 문제

**Q1. `fetch.min.bytes=1MB`로 설정했는데 토픽에 메시지가 없을 때 Consumer는 계속 Fetch 요청을 보내나요?**

<details>
<summary>해설 보기</summary>

아니요. `fetch.max.wait.ms` 덕분에 Long Polling으로 동작합니다. Consumer가 Fetch 요청을 보내면 브로커는 `fetch.min.bytes`(1MB) 조건이 충족될 때까지 응답을 보류합니다. 데이터가 없으면 `fetch.max.wait.ms`(기본 500ms) 후에 빈 응답을 반환합니다.

즉 메시지가 없을 때: Fetch 요청 → 500ms 대기 → 빈 응답 → Fetch 요청 → 500ms 대기 → ... 패턴으로 초당 2번 정도만 브로커에 요청합니다. `fetch.min.bytes=1`이면 즉시 빈 응답을 받아서 더 자주 요청하게 됩니다. 메시지가 없는 환경에서는 `fetch.min.bytes`를 올리는 것이 브로커 부하 측면에서도 유리합니다.

</details>

---

**Q2. 스레드 풀 방식에서 Worker 스레드가 10개인데 파티션이 3개라면 어떤 이점이 있나요?**

<details>
<summary>해설 보기</summary>

파티션 수와 무관하게 처리 병렬도를 높일 수 있습니다. 파티션 3개이면 Consumer 인스턴스를 3개 이상 늘려도 처리 효율이 없습니다. 하지만 스레드 풀을 사용하면 파티션 3개에서 온 메시지를 10개 스레드로 동시에 처리할 수 있습니다.

예를 들어 파티션 0에서 받은 10개 레코드를 Worker 1~10이 동시에 처리하면, 순차 처리 대비 10배 빠릅니다. 특히 레코드당 처리가 DB 쿼리나 외부 API 호출처럼 I/O 대기가 많을 때 효과적입니다.

단, 같은 파티션의 레코드를 여러 스레드에서 동시 처리하면 순서 보장이 깨집니다. 순서가 중요한 경우 파티션별로 독립 큐를 만들고 파티션당 스레드 1개를 배정하는 방식을 사용합니다.

</details>

---

**Q3. Consumer가 처리 속도보다 빠르게 메시지를 받아서 OOM이 발생했습니다. 어떻게 해결하나요?**

<details>
<summary>해설 보기</summary>

`consumer.pause()`와 `consumer.resume()`을 사용해서 배압(Backpressure)을 구현합니다.

처리 대기 중인 레코드 수를 모니터링하다가 임계값을 초과하면 `consumer.pause(assignedPartitions)`를 호출합니다. `pause` 상태에서 `poll()`은 빈 응답을 반환해서 새 데이터를 가져오지 않습니다. 처리 대기 레코드가 줄어들면 `consumer.resume(assignedPartitions)`로 다시 Fetch를 시작합니다.

```java
if (pendingQueue.size() > MAX_QUEUE_SIZE) {
    consumer.pause(consumer.assignment());
} else {
    consumer.resume(consumer.assignment());
}
ConsumerRecords<K,V> records = consumer.poll(Duration.ofMillis(100));
```

이 방식으로 처리 속도에 맞게 Fetch 속도를 자동 조절합니다. `max.poll.records`를 줄이는 것도 간단한 해결책이지만, 스레드 풀 방식에서는 `pause`/`resume`이 더 정밀한 제어를 제공합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Producer 처리량 최적화](./01-producer-throughput.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파티션 핫스팟 ➡️](./03-partition-hotspot.md)**

</div>
