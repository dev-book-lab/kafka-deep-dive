# 파티션 복제 — Leader/Follower 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Kafka의 복제(Replication)는 어떤 방식으로 동작하는가?
- Follower가 Leader를 복제할 때 Push가 아닌 Fetch(Pull) 방식을 쓰는 이유는?
- Leader와 Follower의 역할은 정확히 어떻게 나뉘는가?
- High Watermark(HW)가 Consumer에게 노출되는 offset을 어떻게 제어하는가?
- Leader Epoch는 무엇이고 왜 필요한가?
- 복제 팩터(Replication Factor)를 결정하는 기준은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka의 내구성은 복제(Replication)에서 나온다. 브로커 1대가 장애를 일으켜도 Follower가 데이터를 가지고 있기 때문에 서비스가 계속된다.

하지만 복제의 동작 방식을 모르면:
- "복제 팩터 3이면 항상 안전하다"고 오해 → ISR이 1개로 줄어도 모른 채 운영
- Follower가 Leader를 얼마나 따라가고 있는지 모니터링 안 함 → 장애 직전까지 인지 못함
- High Watermark 개념을 모르면 Consumer가 왜 최신 메시지를 즉시 못 받는지 설명 불가
- `replica.lag.time.max.ms` 설정이 ISR 이탈에 미치는 영향을 모르고 기본값 사용

복제 메커니즘을 이해하면 `acks=all`, `min.insync.replicas` 등의 설정이 어떻게 연동되는지 자연스럽게 이해된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 복제 팩터 = 단순 데이터 복사본 수로만 이해

  이해: "replication.factor=3이면 데이터가 3곳에 저장된다"
  누락:
    - 3개 중 누가 Leader이고 누가 Follower인지
    - Follower가 Leader를 어떻게 따라가는지
    - ISR이 3개에서 1개로 줄었을 때의 의미

  결과:
    ISR=1 경고를 무시하고 운영
    브로커 1대 장애 시 해당 파티션의 Follower가 없어서 복구 불가

실수 2: Consumer가 Leader와 Follower 모두에서 읽는다고 가정

  가정: "Follower가 있으니 읽기 트래픽을 분산하면 되겠다"
  실제(Kafka 2.3 이전): 모든 Fetch는 Leader에서만 가능
                         Follower는 Leader 복제용으로만 존재

  Kafka 2.4+: rack-aware Follower Fetching 가능하지만 기본은 Leader
              → 무조건 Follower에서 읽을 수 있다는 가정은 틀림

실수 3: High Watermark를 모른 채 실시간성 가정

  코드:
    producer.send(record); // 방금 발행
    // 바로 Consumer로 조회 시도 → 못 받을 수 있음

  이유: HW = ISR 전체가 복제 완료한 offset
        Follower 복제 완료 전까지 Consumer는 해당 메시지를 볼 수 없음
        acks=1이어도 HW가 올라가야 Consumer가 읽을 수 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
올바른 ISR 모니터링:
  kafka-topics --bootstrap-server localhost:9092 \
    --describe --topic orders
  # Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,2,3
  # Isr에서 브로커가 빠지면 → 즉시 알람 설정

  JMX 지표:
    kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
    → 0이 정상, 1 이상이면 ISR 이탈 발생 중

올바른 복제 팩터 선택:
  최소: replication.factor=3 (브로커 1대 장애 허용)
  권장: replication.factor <= 브로커 수
  비용: 복제 팩터 × 디스크 용량이 필요

Follower 복제 지연 모니터링:
  JMX: kafka.server:type=ReplicaFetcherManager,name=MaxLag
  → 높으면 Follower가 Leader를 못 따라가고 있음
  → 원인: 브로커 과부하, 네트워크 지연, 디스크 I/O 병목
```

---

## 🔬 내부 동작 원리

### 1. Leader/Follower 파티션 구조

```
토픽 "orders" (파티션 3개, 복제 팩터 3):

  Broker 1                  Broker 2                  Broker 3
  ┌────────────────┐        ┌────────────────┐        ┌────────────────┐
  │ Partition 0    │        │ Partition 0    │        │ Partition 0    │
  │ [LEADER]       │        │ [FOLLOWER]     │        │ [FOLLOWER]     │
  │ LEO: 101       │        │ LEO: 99        │        │ LEO: 101       │
  │ HW:  99        │        │ HW:  99        │        │ HW:  99        │
  ├────────────────┤        ├────────────────┤        ├────────────────┤
  │ Partition 1    │        │ Partition 1    │        │                │
  │ [FOLLOWER]     │        │ [LEADER]       │        │ [FOLLOWER]     │
  └────────────────┘        └────────────────┘        └────────────────┘

  - 각 파티션의 Leader는 서로 다른 브로커에 분산 → 부하 분산
  - Producer/Consumer 요청: Leader만 처리
  - Follower: Leader 복제 전용
  - LEO: Log End Offset (내가 가진 마지막 offset + 1)
  - HW: High Watermark (모든 ISR이 복제 완료한 offset)
```

### 2. Follower의 Fetch 기반 복제

```
Follower가 Leader를 복제하는 방식 = Consumer Fetch와 동일 메커니즘

  Follower (Broker 2)                    Leader (Broker 1)
       │                                         │
       │── FetchRequest(partition=0, offset=99)─►│
       │                                         │ .log 파일에서 offset 99 이후 읽기
       │◄── FetchResponse(records: 99,100) ──────│
       │                                         │
       │ 로컬 .log에 기록                           │
       │ 자신의 LEO = 101로 업데이트                  │
       │                                         │
       │── FetchRequest(offset=101) ───────────► │
       │   (데이터 없으면 fetch.wait.max.ms 대기) │

  왜 Pull(Fetch) 방식인가?
    Push: Leader가 각 Follower로 전송
          → Follower마다 전송 속도 관리 복잡
          → 느린 Follower가 Leader 쓰기 스레드를 블로킹
    Pull: Follower가 자신의 속도로 요청
          → 느린 Follower가 Leader 성능에 영향 없음
          → Consumer Fetch 코드 재사용 (코드 단순성)

  관련 설정:
    replica.fetch.max.bytes=1048576  (Follower가 한 번에 가져오는 최대 크기)
    num.replica.fetchers=1           (복제 스레드 수, 고처리량 시 증가 권장)
    replica.fetch.wait.max.ms=500    (데이터 없을 때 대기 시간)
```

### 3. High Watermark(HW)와 Log End Offset(LEO)

```
시간 흐름에 따른 HW/LEO 변화:

  t0: 초기 상태
      Leader LEO=100, HW=100
      Follower2 LEO=100, Follower3 LEO=100
      Consumer가 읽을 수 있는 최대: offset 99

  t1: Producer → offset 100 발행
      Leader LEO=101
      Follower2 LEO=100 (아직 Fetch 안 함)
      Follower3 LEO=100
      HW=100 (min(101,100,100) = 100)
      → Consumer는 여전히 offset 99까지만 읽을 수 있음

  t2: Follower3가 Fetch → offset 100 복제 완료
      Leader LEO=101, Follower2 LEO=100, Follower3 LEO=101
      HW=100 (min(101,100,101) = 100, Follower2가 병목)

  t3: Follower2도 Fetch → offset 100 복제 완료
      모든 ISR LEO=101
      HW=101
      → Consumer가 offset 100을 읽을 수 있게 됨

  HW = min(ISR의 모든 LEO)
  Consumer는 HW 이하의 offset만 읽을 수 있음
  → "모든 ISR이 안전하게 저장했다고 보장된 offset"

  HW가 없으면:
    Consumer가 offset 100을 읽었는데
    Follower 복제 완료 전에 Leader 장애 → 새 Leader는 offset 100 없음
    → 이미 Consumer가 읽은 메시지가 데이터상 사라지는 모순 발생
```

### 4. Leader Epoch와 데이터 정합성

```
Leader Epoch = 리더십이 교체될 때마다 단조 증가하는 번호

문제 상황 (Leader Epoch 없을 때):
  Epoch 0: Broker 1 = Leader, offset 0~100 기록
  Follower Broker 2: offset 0~98까지만 복제 완료
  Broker 1 장애 → Broker 2가 새 Leader (offset 0~98)
  Broker 1 복구 → Follower로 복귀 시도
    Broker 1에는 offset 99~100 있음, 새 Leader(Broker2)에는 없음
    → 데이터 불일치!

Leader Epoch로 해결:
  Epoch 0: Broker 1 Leader → offset 0~100 기록
  Broker 1 장애 → Broker 2가 Epoch 1 시작
  Broker 1 복구 시:
    "Epoch 0 데이터 중 Epoch 1 Leader가 인정하는 offset은 몇까지?"
    → Broker 2에게 OffsetForLeaderEpoch 요청
    → "Epoch 0는 offset 98까지만 유효" 응답
    → Broker 1: offset 99~100 제거(Truncation) 후 Follower로 합류

  결과: 새 Leader와 복귀 Follower 간 데이터 일관성 보장
```

### 5. 복제 경로와 Zero-Copy

```
브로커 내부 복제 성능:

  Producer → Leader Partition
                    │
                    ▼ write() → OS PageCache
                    │
           ┌────────┴────────┐
           │                 │
           ▼                 ▼
     Consumer Fetch       Follower Fetch
     sendfile()           sendfile()
     PageCache → NIC      PageCache → NIC
     (Zero-Copy)          (Zero-Copy)

  Follower가 Leader에서 읽는 것 = Consumer Fetch와 동일한 경로
  → sendfile()로 PageCache → Follower로 Zero-Copy 전달
  → 추가 디스크 I/O 없음 (PageCache 히트 시)
  → Follower 복제가 Producer 쓰기 성능에 영향을 최소화
```

---

## 💻 실전 실험

### 실험 1: 파티션 복제 상태 확인

```bash
# 복제 팩터 3인 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic replication-test \
  --partitions 3 \
  --replication-factor 3

# 복제 상태 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic replication-test

# 출력:
# Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,2,3
# Partition: 1  Leader: 2  Replicas: 2,3,1  Isr: 2,3,1
# Partition: 2  Leader: 3  Replicas: 3,1,2  Isr: 3,1,2
#
# Replicas: 복제본이 있는 브로커 목록 (첫 번째 = Preferred Leader)
# Isr: 현재 In-Sync Replica 목록
```

### 실험 2: Follower 장애 시 ISR 변화 관찰

```bash
# 메시지 발행
for i in $(seq 1 20); do
  echo "msg-$i"
done | kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic replication-test

# 정상 상태 ISR 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic replication-test
# Isr: 1,2,3

# Broker 3 컨테이너 중단
docker stop kafka-3

# replica.lag.time.max.ms(30초) 경과 후 ISR 변화 확인
sleep 35
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic replication-test
# Isr: 1,2  ← Broker 3이 ISR에서 이탈

# Broker 3 재시작 → 복제 동기화 → ISR 복귀
docker start kafka-3
sleep 20
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic replication-test
# Isr: 1,2,3  ← 복귀 완료
```

### 실험 3: Under-Replicated Partitions 모니터링

```bash
# Under-Replicated 파티션 목록 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe \
  --under-replicated-partitions
# 출력 없음 = 모든 파티션 정상 복제 중
# Broker 장애 시 해당 파티션이 출력됨

# Under-Min-ISR 파티션 확인 (min.insync.replicas 미달)
kafka-topics --bootstrap-server localhost:19092 \
  --describe \
  --under-min-isr-partitions
```

---

## 📊 성능/비용 비교

### 복제 팩터별 내구성 vs 비용

```
replication.factor=1:
  내구성: 브로커 장애 시 데이터 유실
  비용:   디스크 1배
  용도:   개발/테스트, 유실 허용 로그

replication.factor=2:
  내구성: 브로커 1대 장애 허용
  비용:   디스크 2배
  주의:   ISR이 1개로 줄면 사실상 복제 없음

replication.factor=3:
  내구성: 브로커 2대 동시 장애까지 허용 (ISR 기준)
  비용:   디스크 3배
  용도:   운영 환경 표준

acks=all + replication.factor=3일 때 쓰기 지연:
  추가 지연 = Leader가 ISR Follower 복제 확인 대기
  실제: 수 ms ~ 수십 ms (Follower 복제 속도에 의존)
  ISR 중 가장 느린 Follower가 병목
```

### num.replica.fetchers 설정 효과

```
num.replica.fetchers=1 (기본):
  Follower가 단일 스레드로 모든 파티션 복제
  파티션 수 많으면 복제 지연 발생

num.replica.fetchers=4:
  4스레드 병렬 복제 → 복제 지연 감소
  CPU 사용량 증가

권장: 브로커당 파티션 수 > 100이고 복제 지연 발생 시 2~4로 증가
```

---

## ⚖️ 트레이드오프

```
복제 팩터 높이면:
  ✅ 내구성 향상 (더 많은 브로커 장애 허용)
  ✅ Consumer 읽기 가용성 향상 (Leader 재선출 가속)
  ❌ 디스크 비용 복제 팩터 배수
  ❌ acks=all 시 지연 증가 (더 많은 Follower 복제 대기)
  ❌ 네트워크 트래픽 증가 (Leader → Follower 복제)

Follower Pull 방식:
  ✅ 느린 Follower가 Leader 성능에 영향 없음
  ❌ 복제 지연 발생 가능 (Follower가 느리면 HW 상승 지연)
  ❌ HW 지연 = Consumer 읽기 지연 (미복제 데이터 읽기 방지)

High Watermark:
  ✅ Consumer가 미복제 데이터 읽는 것 방지 (일관성 보장)
  ❌ ISR 복제 지연만큼 Consumer 읽기 지연 발생
```

---

## 📌 핵심 정리

```
Kafka 복제 핵심:

1. Leader만 읽기/쓰기 처리
   Follower는 Leader를 Fetch로 복제하는 역할만

2. Follower는 Consumer처럼 Leader에서 Fetch (Pull)
   느린 Follower가 Leader 성능에 영향 없음

3. High Watermark = 안전하게 읽을 수 있는 최대 offset
   = min(ISR의 모든 LEO)
   Consumer는 HW 이하 offset만 읽기 가능

4. Leader Epoch = 리더십 교체 횟수
   복귀 Follower가 데이터 정합성 확인에 사용
   잘못된 offset Truncation 방지

5. ISR 모니터링 필수
   UnderReplicatedPartitions JMX 지표 = 0이 정상
   replication.factor=3, min.insync.replicas=2가 운영 표준
```

---

## 🤔 생각해볼 문제

**Q1. 복제 팩터가 3인데 브로커가 2대뿐이라면 어떤 일이 발생하나요?**

<details>
<summary>해설 보기</summary>

토픽 생성 시 에러가 발생합니다. `replication.factor=3`은 3개의 서로 다른 브로커에 복제본을 배치해야 하는데, 브로커가 2대뿐이면 3번째 복제본을 배치할 수 없습니다. `InvalidReplicationFactorException: Replication factor: 3 larger than available brokers: 2` 에러가 발생합니다.

운영 환경에서 브로커 장애 후 임시로 2대만 남은 경우에도, 기존 복제 팩터 3 토픽의 일부 파티션 복제본이 없어진 브로커에 있었다면 해당 파티션은 Under-Replicated 상태가 됩니다. 브로커가 복구되면 자동으로 복제본이 다시 동기화됩니다.

</details>

---

**Q2. Follower가 Leader를 따라가는 도중 Leader가 장애나면, 미복제된 offset의 메시지는 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

`acks=all` 설정 여부에 따라 다릅니다.

`acks=all`이면: Leader는 ISR 전체가 복제 완료한 후에야 Producer에게 성공 응답을 보냅니다. 미복제 메시지는 Producer 성공 응답 전이므로, Producer는 실패로 인식하고 재시도합니다. 데이터 유실 없음.

`acks=1`이면: Leader가 자신에게 기록한 즉시 성공 응답합니다. Follower가 복제 완료 전에 Leader 장애 → 새 Leader(Follower)는 해당 메시지 없음 → Producer는 이미 성공 응답 받았으므로 재시도 안 함 → 데이터 유실.

이것이 `acks=all`을 권장하는 핵심 이유입니다.

</details>

---

**Q3. Kafka 2.4의 Follower Fetching(rack-aware)은 어떤 경우에 활성화하면 유용한가요?**

<details>
<summary>해설 보기</summary>

멀티 AZ(Availability Zone) 또는 멀티 리전 배포에서 유용합니다. 각 브로커를 다른 AZ에 배치하고, Consumer도 특정 AZ에서 실행 중이라면, Consumer가 항상 같은 AZ의 Leader를 찾아 다른 AZ로 네트워크 요청을 보내는 Cross-AZ 트래픽 비용이 발생합니다.

Follower Fetching(`replica.selector.class=RackAwareReplicaSelector`)을 활성화하면 Consumer가 동일 rack(AZ)에 있는 Follower에서 읽을 수 있습니다. Cross-AZ 네트워크 비용이 줄고 지연도 낮아집니다. 단, Follower는 HW까지만 노출하므로 최신성은 Leader와 동일합니다.

Cloud 환경(AWS, GCP)에서 AZ 간 트래픽 비용이 부담될 때 매우 효과적인 최적화입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: ISR(In-Sync Replicas) ➡️](./02-isr-in-sync-replicas.md)**

</div>
