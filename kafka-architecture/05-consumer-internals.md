# Consumer 내부 동작 — Fetch 루프와 폴링 간격

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Kafka Consumer가 Push 대신 Pull(Fetch) 방식을 택한 이유는?
- `poll()` 을 호출하면 내부에서 어떤 일이 순서대로 일어나는가?
- `max.poll.records`와 `max.poll.interval.ms`는 어떻게 연결되어 리밸런싱을 유발하는가?
- `fetch.min.bytes`, `fetch.max.wait.ms`, `max.partition.fetch.bytes`의 차이는?
- `auto.offset.reset=earliest/latest`는 정확히 어떤 상황에 적용되는가?
- Consumer가 브로커에서 몇 개의 TCP 연결을 유지하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`poll()`의 타임아웃과 처리 시간을 잘못 관리하면 Kafka가 Consumer를 그룹에서 강제로 제거하고 리밸런싱이 발생한다.

```
consumer.poll(Duration.ofSeconds(1))을 호출해서 500개의 레코드를 받음
각 레코드를 DB에 저장하는 데 1ms → 500ms 소요
다음 poll()은 약 1.5초 후 호출됨

max.poll.interval.ms=300000(5분)이면 문제 없음
그런데 처리 로직이 외부 API를 호출하고 응답이 느려서
500개 처리에 400초가 걸리면 → max.poll.interval.ms 초과 → 리밸런싱!
```

이 문제를 모르면 "Kafka가 이유 없이 재할당된다"고 생각하고, 실제로는 처리 로직 최적화나 `max.poll.records` 감소가 필요한 상황을 놓친다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: poll() 루프에서 긴 처리 작업 실행

  코드:
    while(true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
        for (ConsumerRecord<String, String> record : records) {
            callSlowExternalAPI(record);  // 레코드당 1초씩 걸림
        }
    }

  문제: max.poll.records=500, 레코드당 1초 → 500초 처리
        max.poll.interval.ms=300000(5분) → 5분 초과
        → Consumer가 그룹에서 제거 → 리밸런싱
        → 처리 중인 메시지 재처리 (중복)

  해결 방법:
    1. max.poll.records=10 (한 번에 적게 가져와서 처리 시간 단축)
    2. max.poll.interval.ms=600000 (10분으로 늘리기)
    3. 처리 로직을 비동기로 분리하고 poll은 별도 스레드에서 계속 실행
       (단, 비동기 처리 완료 전에 offset 커밋하지 않도록 주의)

실수 2: auto.offset.reset 오해

  신규 Consumer Group으로 이미 메시지가 쌓인 토픽을 구독:
  auto.offset.reset=latest → 구독 시점 이후의 메시지만 받음
  
  기대: "이전에 쌓인 메시지도 처리할 것이다"
  실제: Consumer Group이 처음 파티션을 할당받을 때 offset이 없으면
        auto.offset.reset 기준으로 초기 offset 설정
        latest → 현재 로그 끝 offset 설정 → 이전 메시지 모두 건너뜀

  올바른 이해:
    auto.offset.reset은 "offset 정보가 없을 때만" 적용
    기존 Consumer Group은 마지막 커밋 offset부터 재개
    처음부터 읽으려면: --reset-offsets --to-earliest 로 offset 리셋 필요

실수 3: 여러 토픽 구독 시 파티션 할당 오해

  설정: consumer.subscribe(Arrays.asList("topicA", "topicB"))
  기대: "topicA 메시지와 topicB 메시지를 각각 다른 스레드에서 처리할 것이다"
  실제: 단일 Consumer 인스턴스가 topicA와 topicB의 파티션을 모두 할당받음
        하나의 poll() 루프에서 두 토픽 메시지를 섞어서 처리
        별도 스레드로 처리하려면 별도 Consumer 인스턴스 필요
        (KafkaConsumer는 스레드 안전하지 않음 — 멀티 스레드 접근 금지)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
안전한 poll() 루프 패턴:

  // 처리 시간 예측 가능한 경우
  while (running) {
      ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
      for (ConsumerRecord<String, String> record : records) {
          process(record);
          // 레코드당 처리 시간 << max.poll.interval.ms / max.poll.records
      }
      consumer.commitSync(); // 배치 완료 후 커밋
  }

  // 처리 시간 예측 불가 (외부 API 호출 등)
  // max.poll.records를 매우 작게 설정 (1~10)
  // 또는 비동기 처리 + pause/resume 패턴 사용
  while (running) {
      ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
      submitToAsyncProcessor(records);  // 비동기 처리 제출
      // 처리 완료는 콜백으로 offset 커밋
  }

설정 기준:
  max.poll.records = 처리 배치 크기
  max.poll.interval.ms > max.poll.records × 레코드당 최대 처리 시간

  예: 레코드당 최대 100ms, max.poll.records=500
      → max.poll.interval.ms > 500 × 100ms = 50,000ms (50초) → 60,000 (60초) 설정
```

---

## 🔬 내부 동작 원리

### 1. Consumer 초기화 시 연결

```
KafkaConsumer 생성 시:

  1. bootstrap.servers에 접속 → 클러스터 메타데이터 조회
     (모든 브로커 정보, 토픽 파티션 배치)

  2. subscribe(topic) 호출 → 실제 할당은 poll() 호출 시 발생

  3. 첫 poll() 호출:
     a. Group Coordinator 찾기
        → __consumer_offsets 토픽의 파티션 담당 브로커 찾기
        → hash(groupId) % 50 → __consumer_offsets 파티션 결정
        → 해당 파티션 Leader 브로커 = Group Coordinator

     b. JoinGroup 요청 (Group Coordinator에게)
        → Consumer의 기본 정보, 지원하는 할당 전략 전송

     c. 파티션 할당 (Group Leader Consumer가 수행)
        → Group Coordinator가 첫 번째 JoinGroup 응답자를 Group Leader로 선정
        → Group Leader가 할당 전략 실행 → 결과를 Group Coordinator에게 전달

     d. SyncGroup 요청/응답
        → 각 Consumer가 자신의 파티션 할당 정보 수신

     e. 각 파티션의 마지막 커밋 offset 조회 (Group Coordinator에서)
        → 없으면 auto.offset.reset 기준으로 결정

  연결 유지:
    Group Coordinator: 1개 (heartbeat, commit)
    각 파티션 Leader 브로커: 파티션당 1개 연결 (Fetch 요청)
    총 TCP 연결 수 = 1(GC) + N(파티션 Leader 브로커 수)
```

### 2. poll() 호출 내부 흐름

```
consumer.poll(Duration.ofMillis(1000)):

  ┌─────────────────────────────────────────────────────────────┐
  │ 1. ConsumerCoordinator.poll()                               │
  │    - heartbeat 전송 (필요 시)                                  │
  │    - 리밸런싱 진행 중이면 완료까지 대기                             │
  │    - auto.commit 활성화 시 주기적 커밋                           │
  │                                                             │
  │ 2. Fetcher.fetchedRecords() 반환 (이전 Fetch에서 받은 데이터)     │
  │    - 이미 Fetch해놓은 데이터가 있으면 바로 반환                      │
  │    - 없으면 다음 단계로                                         │
  │                                                             │
  │ 3. Fetcher.sendFetches()                                    │
  │    - 각 파티션의 현재 position(fetch offset) 기준으로             │
  │      브로커에 FetchRequest 전송                                │
  │    - max.partition.fetch.bytes 크기만큼 요청                   │
  │                                                             │
  │ 4. NetworkClient.poll(timeout)                              │
  │    - FetchResponse 대기 (fetch.max.wait.ms까지)               │
  │    - fetch.min.bytes 충족될 때까지 브로커가 응답 보류               │
  │                                                             │
  │ 5. FetchResponse 수신 → 역직렬화 → ConsumerRecords 변환          │
  │    - max.poll.records 개수만큼 잘라서 반환                       │
  │    - 나머지는 다음 poll()에서 반환 (Fetch 재요청 없음)              │
  └─────────────────────────────────────────────────────────────┘
  
  반환: ConsumerRecords (최대 max.poll.records개)
```

### 3. Fetch 설정의 상호작용

```
Fetch 관련 설정 동작:

  fetch.min.bytes=65536 (64 KB):
    Consumer가 Fetch 요청 → 브로커 응답 조건:
      현재 가용 데이터 >= 64 KB → 즉시 응답
      현재 가용 데이터 < 64 KB → 데이터가 64 KB 쌓일 때까지 대기
      단, fetch.max.wait.ms(기본 500ms) 지나면 데이터 부족해도 응답

  max.partition.fetch.bytes=1048576 (1 MB):
    파티션당 최대 1 MB를 한 번의 Fetch에서 가져옴
    파티션 10개 할당 → 최대 10 MB/Fetch

  max.poll.records=500:
    Fetch로 5000개가 왔어도 500개만 반환
    나머지 4500개는 내부 버퍼에 보관 → 다음 poll()에서 순차 반환
    → 브로커에 재Fetch 요청 없이 내부 버퍼에서 서빙

흐름 예시 (메시지가 활발히 들어오는 경우):
  t=0ms:   Fetch 요청 (파티션 0: offset 100부터)
  t=2ms:   브로커 응답 (64 KB 데이터, 레코드 300개)
           내부 버퍼: 300개
  t=2ms:   poll() 반환: 300개 (max.poll.records=500이므로 전부)
           처리 중에 다음 Fetch 요청 자동 시작
  t=5ms:   처리 완료 → 다음 poll() 호출
           내부 버퍼: 비어있음 → 새 Fetch 응답 대기
```

### 4. heartbeat와 max.poll.interval.ms

```
두 가지 독립적인 타이머:

  1. heartbeat 타이머 (별도 HeartbeatThread):
     heartbeat.interval.ms(기본 3000ms)마다 Group Coordinator에 heartbeat 전송
     session.timeout.ms(기본 10000ms) 동안 heartbeat 없으면 → Consumer 장애로 판단
     → 리밸런싱 발생

  2. max.poll.interval.ms 타이머:
     poll() 호출 간격이 max.poll.interval.ms(기본 300000ms) 초과 시
     → Consumer가 처리 능력을 잃었다고 판단 → Group Coordinator에 LeaveGroup 요청
     → 리밸런싱 발생

  두 타이머의 독립성:
    heartbeat 스레드는 별도로 동작 → poll()을 오래 안 해도 heartbeat는 전송
    하지만 max.poll.interval.ms 체크는 poll() 호출 시 내부에서 검사
    → poll()을 오래 안 하면 두 타이머 중 max.poll.interval.ms가 먼저 문제

  실수 패턴:
    session.timeout.ms=10000ms, heartbeat.interval.ms=3000ms
    poll() 루프에서 처리 시간이 10초 넘어도
    → HeartbeatThread가 3초마다 heartbeat 보내므로 session 유지됨
    → 하지만 max.poll.interval.ms(5분) 초과 시 LeaveGroup 발생
    → "heartbeat는 정상인데 왜 리밸런싱이?" 로 혼란
```

### 5. auto.offset.reset 적용 조건

```
auto.offset.reset이 적용되는 경우:
  1. 새 Consumer Group이 토픽을 처음 구독 (offset 기록 없음)
  2. 기존 Consumer Group이지만 파티션의 offset이 만료/삭제된 경우
     (retention.ms 지나 해당 offset의 데이터가 없는 경우)

  earliest: 파티션의 첫 번째 offset으로 이동 (처음부터 읽기)
  latest:   파티션의 마지막 offset으로 이동 (이후 메시지만 읽기)
  none:     offset 없으면 예외 발생 (직접 offset 관리하는 경우)

auto.offset.reset이 적용되지 않는 경우:
  기존 Consumer Group이 마지막 커밋 offset이 있는 경우
  → auto.offset.reset 무시, 마지막 커밋 offset부터 재개

  → 이 차이를 모르면 새 Consumer Group과 기존 Consumer Group의
    동작이 다른 이유를 설명할 수 없음
```

---

## 💻 실전 실험

### 실험 1: max.poll.interval.ms 초과 리밸런싱 재현

```bash
# 파티션 3개 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic slow-test --partitions 3 --replication-factor 1

# 메시지 미리 발행
for i in $(seq 1 30); do
  echo "msg-$i" | kafka-console-producer \
    --bootstrap-server localhost:19092 \
    --topic slow-test
done

# max.poll.interval.ms를 5초로 짧게 설정한 Consumer 시뮬레이션
# (실제 Spring Kafka 코드로 테스트)
# consumer.props:
#   max.poll.interval.ms=5000
#   max.poll.records=10
# 처리 루프에서 Thread.sleep(6000) → 6초 후 리밸런싱 발생

# 리밸런싱 발생 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group slow-group
# State: PreparingRebalance 또는 CompletingRebalance 확인
```

### 실험 2: Fetch 설정별 지연 측정

```bash
# Consumer 성능 테스트 (기본 Fetch 설정)
kafka-consumer-perf-test \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --messages 1000000 \
  --group perf-test-group \
  --consumer-props \
    fetch.min.bytes=1 \
    fetch.max.wait.ms=500 \
    max.poll.records=500

# Consumer 성능 테스트 (최적화된 Fetch 설정)
kafka-consumer-perf-test \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --messages 1000000 \
  --group perf-test-group2 \
  --consumer-props \
    fetch.min.bytes=65536 \
    fetch.max.wait.ms=100 \
    max.poll.records=1000

# 결과 비교: data.consumed.in.MB/s
```

### 실험 3: auto.offset.reset 동작 확인

```bash
# 새 토픽에 메시지 미리 발행
kafka-console-producer --bootstrap-server localhost:19092 \
  --topic offset-test
# 메시지 10개 입력 후 종료

# auto.offset.reset=latest로 새 그룹 시작
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic offset-test \
  --group new-group-latest \
  --consumer-property auto.offset.reset=latest
# → 이전 10개 메시지 안 보임 (latest = 지금 이후 메시지만)

# 새 메시지 발행
echo "new-msg" | kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic offset-test
# → 위 Consumer에서 "new-msg" 수신 확인

# auto.offset.reset=earliest로 새 그룹 시작
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic offset-test \
  --group new-group-earliest \
  --consumer-property auto.offset.reset=earliest
# → 이전 10개 + "new-msg" 모두 수신
```

---

## 📊 성능/비용 비교

### Fetch 설정별 처리량 vs 지연 비교

```
조건: 1 KB 메시지, 파티션 10개, Consumer 10개

  설정 A: fetch.min.bytes=1, fetch.max.wait.ms=500, max.poll.records=500
    처리량: ~200,000 msg/sec
    평균 지연(end-to-end): ~600 ms
    특징: 메시지 도착 즉시 Fetch (지연 낮음) 하지만 소량 Fetch 빈도 높음

  설정 B: fetch.min.bytes=65536, fetch.max.wait.ms=100, max.poll.records=1000
    처리량: ~500,000 msg/sec
    평균 지연(end-to-end): ~150 ms
    특징: 64 KB 모아서 Fetch (처리량 높음) 메시지 적을 때는 100ms 대기

  설정 C: fetch.min.bytes=1048576, fetch.max.wait.ms=500, max.poll.records=5000
    처리량: ~800,000 msg/sec
    평균 지연(end-to-end): ~700 ms
    특징: 1 MB 모아서 Fetch (처리량 극대화) 지연은 높음

결론:
  실시간 지연이 중요: 설정 A (fetch.min.bytes=1)
  처리량이 중요:      설정 B~C (fetch.min.bytes 높이기)
```

### max.poll.records 설정과 리밸런싱 빈도

```
처리 로직: 레코드당 DB INSERT 1ms

  max.poll.records=500:
    500개 처리 = 500ms
    max.poll.interval.ms=5분 → 안전 (500ms << 5분)

  max.poll.records=5000:
    5000개 처리 = 5000ms (5초)
    max.poll.interval.ms=5분 → 안전

  max.poll.records=5000, DB 느려서 레코드당 100ms:
    5000개 처리 = 500,000ms (500초 ≈ 8분)
    max.poll.interval.ms=5분 → 리밸런싱 발생!

  해결: max.poll.records를 현실적인 처리 시간에 맞게 조정
    DB 느릴 때(100ms/record): max.poll.records = 5분 / 100ms = 3000
    하지만 DB 부하도 고려 → max.poll.records=300 정도로 보수적 설정
```

---

## ⚖️ 트레이드오프

```
fetch.min.bytes 증가:
  ✅ 대량 Fetch → 브로커 요청 횟수 감소 → 처리량 증가
  ❌ 메시지가 적을 때 fetch.max.wait.ms까지 대기 → 지연 증가
  → 처리량이 높은 토픽에 적합, 실시간 알림 등에는 부적합

max.poll.records 증가:
  ✅ poll()당 더 많은 레코드 → 배치 처리 효율 증가
  ❌ 처리 시간 증가 → max.poll.interval.ms 초과 위험
  → 레코드당 처리 시간 × max.poll.records < max.poll.interval.ms 보장 필수

enable.auto.commit vs 수동 커밋:
  자동: ✅ 구현 단순 ❌ 처리 완료 전 커밋 → 장애 시 메시지 유실
  수동: ✅ 처리 완료 후 커밋 → At-Least-Once 보장 ❌ 구현 복잡

별도 스레드 처리:
  ✅ poll() 루프를 빠르게 유지 → 리밸런싱 위험 감소
  ❌ 비동기 처리 완료 확인 + offset 커밋 관리 복잡
  ❌ KafkaConsumer는 스레드 안전하지 않음 → consumer.pause/resume 패턴 필요
```

---

## 📌 핵심 정리

```
Consumer 내부 핵심:

  poll() = heartbeat + Fetch 요청 + 레코드 반환
  
  Fetch 흐름:
    → 각 파티션 Leader 브로커에 FetchRequest
    → fetch.min.bytes 충족 시 또는 fetch.max.wait.ms 경과 시 응답
    → max.poll.records개 반환 (나머지는 내부 버퍼)

  리밸런싱 트리거:
    session.timeout.ms 동안 heartbeat 없음 (Consumer 장애)
    max.poll.interval.ms 동안 poll() 없음 (처리 지연)
    → max.poll.records × 레코드당 처리 시간 < max.poll.interval.ms 보장

  auto.offset.reset:
    offset 정보가 없을 때만 적용 (처음 구독 또는 offset 만료)
    기존 그룹은 마지막 커밋 offset부터 재개

  TCP 연결:
    Group Coordinator 브로커: 1개
    각 파티션 Leader 브로커: 브로커당 1개
    파티션 10개 / 브로커 3대 = 최대 4개 TCP 연결
```

---

## 🤔 생각해볼 문제

**Q1. `KafkaConsumer`는 스레드 안전하지 않다고 했는데, 멀티 스레드로 Consumer를 사용하고 싶으면 어떻게 해야 하나요?**

<details>
<summary>해설 보기</summary>

두 가지 패턴이 있습니다.

**패턴 1: Consumer 인스턴스 per 스레드**
가장 단순한 방법입니다. 스레드마다 KafkaConsumer 인스턴스를 1개씩 생성합니다. 파티션 수 = 스레드 수로 맞추면 최적입니다. Spring Kafka의 `concurrency` 설정이 이 방식입니다.

```java
// ConcurrentKafkaListenerContainerFactory 설정
factory.setConcurrency(3); // 스레드 3개 = KafkaConsumer 3개 = 파티션 3개
```

**패턴 2: Consumer 스레드 + Worker 스레드 풀**
poll() 루프는 단일 Consumer 스레드에서 실행하고, 처리 로직은 별도 ExecutorService에 제출합니다. poll() 스레드는 빠르게 유지되어 heartbeat와 Fetch를 계속합니다. 처리 완료 후 해당 파티션의 offset을 커밋합니다.

이 패턴은 구현이 복잡하지만 처리 로직이 오래 걸릴 때 리밸런싱 없이 고처리량을 달성할 수 있습니다. 단, offset 커밋 타이밍을 처리 완료와 정확히 동기화해야 합니다.

</details>

---

**Q2. Consumer가 100개의 파티션을 할당받았을 때 100개의 파티션 모두에서 동시에 Fetch를 하나요?**

<details>
<summary>해설 보기</summary>

같은 브로커에 있는 파티션은 한 번의 FetchRequest에 묶어서 보냅니다. Consumer는 브로커별로 FetchRequest를 1개씩 보내고, 각 요청에 해당 브로커가 담당하는 모든 파티션의 Fetch 정보를 포함합니다.

예를 들어 파티션 100개가 브로커 3대에 분산되어 있다면:
- Broker 1 → FetchRequest (33개 파티션 요청 포함)
- Broker 2 → FetchRequest (33개 파티션 요청 포함)  
- Broker 3 → FetchRequest (34개 파티션 요청 포함)

즉 TCP 연결은 3개, FetchRequest는 3개입니다. 이것이 Kafka Consumer가 파티션 수가 많아도 네트워크 연결 관리 오버헤드가 크지 않은 이유입니다.

응답에서 각 파티션의 데이터가 포함되고, 전체 응답 크기는 `max.partition.fetch.bytes × 파티션 수`가 상한입니다. 이 크기가 크면 Consumer 메모리 부하가 될 수 있습니다.

</details>

---

**Q3. Consumer가 `commitSync()`를 호출하기 전에 JVM이 크래시되면 어떤 일이 발생하나요?**

<details>
<summary>해설 보기</summary>

처리한 레코드의 offset이 커밋되지 않아, 다음에 그 파티션을 담당하는 Consumer는 마지막으로 커밋된 offset부터 다시 읽습니다. 이미 처리한 레코드를 다시 처리하게 됩니다. 이것이 Kafka의 기본 보장인 **At-Least-Once** 입니다.

이 중복 처리를 무해하게 만들려면 처리 로직이 **멱등(idempotent)**해야 합니다. 예를 들어:
- DB INSERT 대신 UPSERT (같은 키로 다시 쓰면 덮어쓰기)
- 처리 전에 "이미 처리한 레코드인지" 확인하는 중복 필터 테이블
- Kafka offset을 DB 트랜잭션과 함께 저장해서 정확히 한 번 처리 보장

완전한 Exactly-Once를 원하면 Kafka 트랜잭션을 사용해야 합니다. 이것은 Chapter 3에서 자세히 다룹니다.

</details>

---

<div align="center">

**[⬅️ 이전: Producer 내부 동작](./04-producer-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: ZooKeeper vs KRaft ➡️](./06-zookeeper-vs-kraft.md)**

</div>
