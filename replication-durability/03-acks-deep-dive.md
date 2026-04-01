# acks 설정 완전 분해 — 보장하는 것과 보장하지 않는 것

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `acks=0`, `acks=1`, `acks=all`(-1) 각각이 정확히 무엇을 보장하는가?
- `acks=all`이어도 데이터가 유실될 수 있는 조건은?
- `acks=all`과 `enable.idempotence=true`는 어떻게 다른가?
- acks 설정이 Producer의 응답 지연(latency)에 미치는 영향은?
- `acks=1` 상태에서 Leader 장애 시 정확히 어떤 메시지가 유실되는가?
- 실무에서 acks를 어떤 기준으로 선택해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

acks 설정은 "Kafka에 메시지를 보냈을 때 성공 응답의 의미"를 결정한다. Producer의 `send()` Future가 complete 됐다는 것이 실제로 어디까지 보장하는지를 acks가 정의한다.

acks를 모르면:
- `acks=1`(기본값 Kafka 2.x)로 운영 중 Leader 장애 → "분명히 성공했는데 메시지가 없다" 상황
- `acks=all`이면 항상 안전하다고 오해 → ISR=1 상태에서도 성공 응답이 나오는 것을 모름
- 처리량이 낮아서 `acks=0`으로 바꿨더니 메시지 유실이 발생해도 인지 못함

acks의 정확한 의미를 알면 서비스 요구사항에 맞는 설정을 선택할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: acks=1이 기본값이니 안전하다고 가정

  Kafka 2.x 기본값: acks=1
  의미: Leader 브로커가 자신의 로그에 기록하면 성공 응답
  
  문제 시나리오:
    Producer → acks=1 → offset 100 발행 → Leader 성공 응답
    Follower들은 아직 offset 100 복제 안 함
    Leader 브로커 갑자기 장애
    새 Leader = Follower 중 1개 (offset 100 없음)
    → Producer는 성공이라고 알고 있는 offset 100이 새 Leader에 없음
    → 데이터 유실!
  
  인지: "Kafka에서 성공 응답 받은 메시지가 사라질 수 있다"는 것을 모름

실수 2: acks=all이면 무조건 안전하다는 오해

  상황: ISR=[1] (Follower 2, 3 이탈), acks=all
  Producer 발행 → Leader(1)만 기록 → 성공 응답
  (ISR이 [1]이므로 1개 기록이 acks=all 조건 충족)
  Broker 1 장애 → 데이터 유실

  해결: min.insync.replicas=2와 함께 사용해야
        ISR < 2이면 쓰기 거부 → 유실 방지

실수 3: acks=0이 처리량만 높이고 유실이 적다고 가정

  acks=0: Producer가 브로커 응답을 전혀 기다리지 않음
           네트워크 오류, 브로커 재시작, 버퍼 가득 참 등
           어떤 이유로든 메시지가 브로커에 도달 안 해도 감지 불가
           → 조용한 데이터 유실 (silent data loss)
  
  용도: 로그 수집처럼 일부 유실이 허용되는 경우에만 사용
        비즈니스 중요 메시지에 acks=0은 절대 금지
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
서비스 요구사항별 acks 선택:

  유실 불허 (결제, 주문, 이벤트 소싱):
    acks=all + min.insync.replicas=2 + enable.idempotence=true
    replication.factor=3
    → 브로커 1대 장애에도 데이터 유실 없음
    → 중복 없는 정확히 한 번 기록

  높은 처리량 + 낮은 지연 + 약간의 유실 허용 (실시간 모니터링 지표):
    acks=1 + replication.factor=3
    → Leader 장애 시 마지막 배치 유실 가능
    → 처리량 2~3배 증가

  유실 완전 허용 (디버그 로그, 실험적 데이터):
    acks=0
    → 최고 처리량, 최저 지연
    → 메시지 유실 가능성 상시 존재

권장 Spring Kafka 설정:
  # application.yml
  spring:
    kafka:
      producer:
        acks: all
        properties:
          enable.idempotence: true
          max.in.flight.requests.per.connection: 5
          retries: 2147483647
```

---

## 🔬 내부 동작 원리

### 1. acks=0: 응답 없이 전송

```
Producer 동작:
  send(record) → RecordAccumulator에 추가
  Sender 스레드가 브로커에 전송
  브로커 응답을 기다리지 않음 → 즉시 다음 배치 전송

  ┌─────────────────────────────────────────────────────┐
  │ Producer                                            │
  │  send() → buffer → Sender ──────────────────────►   │
  │                     (응답 기다리지 않음)                │
  │                     ▼                               │
  │                    다음 배치 즉시 전송                  │
  └─────────────────────────────────────────────────────┘

  보장: 없음
  Future.get()의 RecordMetadata:
    offset = -1 (알 수 없음)
    timestamp = -1

  유실 시나리오:
    1. 네트워크 장애: 전송 패킷 유실 → 브로커가 받지 못함 → 감지 불가
    2. 브로커 버퍼 가득 참: 브로커가 패킷 버림 → 감지 불가
    3. 브로커 재시작 중: 연결 거부 → 재연결 후 해당 배치는 재시도 없음
```

### 2. acks=1: Leader 기록 완료

```
Producer 동작:
  send(record) → 브로커 전송
  Leader가 자신의 로그(.log 파일)에 기록 완료
  → Producer에게 성공 응답 (offset, timestamp 포함)
  Follower 복제 여부는 응답에 무관

  ┌─────────────────────────────────────────────────┐
  │ Producer          Leader         Follower 2, 3  │
  │  send() ─────────►write to log                  │
  │         ◄─────────ACK(offset=100)               │
  │                       │                         │
  │                       ▼ (비동기로 복제 진행)        │
  │                   Follower Fetch ◄────────────  │
  └─────────────────────────────────────────────────┘

  보장: Leader 브로커의 로그에 기록됨
  미보장: Follower 복제 완료

  유실 시나리오 (Leader 장애):
    offset 100 기록 후 ack 응답
    Follower는 아직 offset 100 복제 안 함
    Leader 장애 → ISR에서 Follower가 새 Leader로 선출
    새 Leader에는 offset 100 없음 → 유실

    Consumer 관점: offset 99까지 읽었던 Consumer가
                  재개 시 offset 100이 없음 → 건너뜀
```

### 3. acks=all(-1): ISR 전체 기록 완료

```
Producer 동작:
  send(record) → 브로커 전송
  Leader가 로그에 기록
  ISR의 모든 Follower가 복제 완료할 때까지 대기
  → 모든 ISR 멤버 복제 완료 → Producer에게 성공 응답

  ┌──────────────────────────────────────────────────────┐
  │ Producer       Leader         Follower 2   Follower 3│
  │  send() ──────►write to log                          │
  │                    │                                 │
  │                    ├──── Follower 2 Fetch 대기 ───────│
  │                    │                    ◄───Fetch────│
  │                    │                    ──FetchRsp──►│
  │                    ├──── Follower 3 Fetch 대기 ───────│
  │                    │                           ◄Fetch│
  │  ◄──────────── ACK(offset=100) (모든 ISR 완료 후)       │
  └──────────────────────────────────────────────────────┘

  보장: 현재 ISR의 모든 멤버에 기록됨
  미보장: ISR이 1개면 1개에만 기록해도 성공

  유실 방지 시나리오:
    ISR=[1,2,3], offset 100 기록
    Follower 2, 3 복제 완료 → ack 응답
    Leader 장애 → Follower 2 또는 3이 새 Leader
    새 Leader에는 offset 100 있음 → 유실 없음!

  유실 가능 시나리오 (ISR=1):
    ISR=[1] 상태에서 acks=all
    Leader만 기록 → ack 응답
    Leader 장애 → 데이터 유실 (위와 동일)
    → min.insync.replicas=2로 이 케이스 차단
```

### 4. acks=all + min.insync.replicas 조합의 완전한 보호

```
설정: acks=all, min.insync.replicas=2, replication.factor=3

  시나리오별 동작:

  ISR=[1,2,3]:
    쓰기: 3개 ISR 기록 완료 → 성공
    브로커 1대 장애: 나머지 2개에 데이터 있음 → 복구 가능

  ISR=[1,2]:
    쓰기: 2개 ISR 기록 완료 → 성공 (ISR 2 >= min.insync.replicas 2)
    브로커 1대 추가 장애: 나머지 1개에 데이터 있음 → 복구 가능

  ISR=[1]:
    쓰기: ISR 1 < min.insync.replicas 2
    → NotEnoughReplicasException 발생
    → Producer 쓰기 실패 (서비스 장애)
    의도: 내구성 보장이 불가한 상황에서 쓰기를 거부
          "불완전한 성공보다 명시적 실패가 낫다"

  결론: 브로커 1대까지 장애 → 쓰기 성공 + 내구성 보장
        브로커 2대 동시 장애 → 쓰기 실패 (서비스 중단이지만 데이터 유실 없음)
```

### 5. acks와 처리량의 관계

```
acks 설정이 Producer 처리량에 미치는 영향:

  acks=0:
    응답 대기 없음 → max.in.flight 요청 계속 전송
    처리량: 최대 (네트워크 대역폭이 한계)
    지연: 최소 (~수 ms)

  acks=1:
    Leader 응답 대기 → 응답 후 다음 전송
    처리량: 높음 (~acks=0의 70~80%)
    지연: 중간 (~수 ms + Leader 응답 시간)

  acks=all, ISR=3:
    ISR 전체 복제 대기
    처리량: 중간 (~acks=0의 40~60%)
    지연: 높음 (~수 ms + 가장 느린 Follower 복제 시간)

  acks=all + linger.ms=20 + batch.size=65536:
    배치로 묶어서 전송 → acks=all 오버헤드 완화
    처리량: acks=1과 유사 수준까지 회복 가능
    지연: linger.ms만큼 추가 (20ms)
```

---

## 💻 실전 실험

### 실험 1: acks 설정별 처리량 비교

```bash
# acks=0 처리량
kafka-producer-perf-test \
  --topic orders --num-records 1000000 --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:19092 acks=0

# acks=1 처리량
kafka-producer-perf-test \
  --topic orders --num-records 1000000 --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:19092 acks=1

# acks=all 처리량
kafka-producer-perf-test \
  --topic orders --num-records 1000000 --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:19092 acks=all

# 결과 비교:
# acks=0:   ~600,000 records/sec
# acks=1:   ~400,000 records/sec
# acks=all: ~200,000 records/sec (ISR=3 기준)
```

### 실험 2: acks=1 상태에서 Leader 장애 시 유실 재현

```bash
# acks=1으로 메시지 발행 중 Leader 강제 종료
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic orders
# Leader가 Broker 1임을 확인

# 빠른 속도로 메시지 발행 시작 (백그라운드)
kafka-producer-perf-test \
  --topic orders --num-records 100000 --record-size 1024 \
  --throughput 10000 \
  --producer-props bootstrap.servers=localhost:19092 acks=1 &

# 발행 중 Leader 강제 종료
sleep 2 && docker kill --signal=SIGKILL kafka-1

# 발행 완료 후 총 발행 수 vs 실제 저장 수 비교
kafka-consumer-perf-test \
  --bootstrap-server localhost:19092 \
  --topic orders --messages 100000 --group verify-group
# messages.consumed가 발행 수보다 적으면 유실 발생
```

### 실험 3: acks=all + min.insync.replicas=2 검증

```bash
# min.insync.replicas=2 설정
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics --entity-name orders \
  --alter --add-config min.insync.replicas=2

# Broker 2, 3 중단 → ISR=[1]
docker stop kafka-2 kafka-3
sleep 35

# acks=all로 쓰기 시도 → 실패 확인
echo "test" | kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --producer-property acks=all
# ERROR: org.apache.kafka.common.errors.NotEnoughReplicasException
# → 데이터 유실 없이 명시적 실패
```

---

## 📊 성능/비용 비교

### acks 설정별 내구성 vs 처리량 vs 지연

```
기준: replication.factor=3, ISR=3, 1 KB 메시지

  acks=0:
    처리량:    ~600K msg/sec
    p99 지연:  ~2 ms
    내구성:    없음 (유실 가능)
    사용 케이스: 디버그 로그, 완전 유실 허용

  acks=1:
    처리량:    ~400K msg/sec
    p99 지연:  ~5 ms
    내구성:    Leader 기록 보장
               (Leader 장애 시 마지막 배치 유실 가능)
    사용 케이스: 실시간 지표, 일부 유실 허용 스트리밍

  acks=all (ISR=3):
    처리량:    ~150K msg/sec
    p99 지연:  ~15 ms
    내구성:    ISR 전체 기록 보장
    사용 케이스: 결제, 주문, 이벤트 소싱

  acks=all + linger.ms=20 + batch.size=64KB:
    처리량:    ~350K msg/sec (배치 효과)
    p99 지연:  ~25 ms (linger.ms 추가)
    내구성:    ISR 전체 기록 보장
    사용 케이스: 처리량도 중요한 운영 환경 권장
```

---

## ⚖️ 트레이드오프

```
acks=0:
  ✅ 최고 처리량, 최저 지연
  ❌ 유실 감지 불가 (silent data loss)
  ❌ 비즈니스 중요 데이터에 절대 금지

acks=1:
  ✅ 높은 처리량과 낮은 지연의 균형
  ✅ Leader 기록 보장
  ❌ Leader 장애 시 미복제 배치 유실 가능
  ❌ Follower 복제 완료 보장 없음

acks=all:
  ✅ 가장 강한 내구성 보장 (ISR 기준)
  ✅ enable.idempotence와 조합 시 중복 제거도 가능
  ❌ 가장 낮은 처리량
  ❌ 가장 높은 지연
  ❌ ISR=1이면 보장 약화 → min.insync.replicas 필수
```

---

## 📌 핵심 정리

```
acks 핵심:

  acks=0: 브로커 응답 안 기다림 → 유실 감지 불가
  acks=1: Leader 기록 완료 → Leader 장애 시 미복제 유실 가능
  acks=all: ISR 전체 기록 완료 → ISR=1이면 보장 약화

acks=all의 맹점:
  ISR이 줄어도 acks=all 성공 응답이 나옴
  → min.insync.replicas와 반드시 함께 설정

운영 표준:
  replication.factor=3
  acks=all
  min.insync.replicas=2
  enable.idempotence=true
  → 브로커 1대 장애 허용 + 중복 없는 정확한 기록

처리량이 필요하면:
  linger.ms=10~20, batch.size=65536으로 배치 최적화
  → acks=all이어도 충분한 처리량 달성 가능
```

---

## 🤔 생각해볼 문제

**Q1. acks=all이고 ISR=[1,2,3]인데 Follower 2가 GC로 인해 응답이 느리다면 Producer 쓰기 지연에 어떤 영향을 주나요?**

<details>
<summary>해설 보기</summary>

acks=all은 ISR의 모든 멤버가 복제 완료해야 성공 응답을 보냅니다. Follower 2가 GC로 500ms 동안 Fetch를 못 하면, Leader는 Follower 2의 복제 완료를 500ms 동안 기다려야 하고, Producer 쓰기 응답도 500ms 지연됩니다.

이것이 acks=all의 "가장 느린 ISR 멤버가 병목"이 되는 이유입니다. Follower 2가 replica.lag.time.max.ms(30초) 이상 응답 없으면 ISR에서 이탈하고, 그 이후부터는 기다리지 않습니다.

이를 완화하는 방법: JVM GC 최적화(G1GC, STW 최소화), num.replica.fetchers 증가, 복제에 전용 네트워크 인터페이스 할당 등이 있습니다.

</details>

---

**Q2. Producer가 acks=all로 100개의 배치를 보냈는데 브로커가 재시작됩니다. 이미 성공 응답받은 배치는 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

성공 응답을 받은 배치는 ISR의 모든 멤버에 기록이 완료된 것입니다. 브로커가 재시작하더라도 디스크에 기록된 데이터는 유지됩니다. 재시작 후 해당 브로커는 로그 파일을 재로드하고 정상 서비스를 재개합니다.

Kafka 브로커는 기본적으로 `log.flush.interval.messages`나 `log.flush.interval.ms` 설정에 따라 OS 페이지 캐시를 디스크에 flush합니다. 기본값은 OS에 flush를 맡기는 방식이지만, OS는 브로커 종료 시 더티 페이지를 flush합니다. 비정상 종료(kill -9, 전원 차단) 시에는 미flush 데이터가 유실될 수 있지만, 이 경우 acks=all이어도 다른 ISR 멤버에서 복구됩니다.

</details>

---

**Q3. acks=all + enable.idempotence=true + transactional.id 조합은 무엇을 추가로 보장하나요?**

<details>
<summary>해설 보기</summary>

각 설정의 보장 범위:

**acks=all**: ISR 전체에 기록 완료 보장 (내구성)

**enable.idempotence=true**: Producer ID + Sequence Number로 브로커가 중복 레코드를 감지하고 거부. 네트워크 오류로 인한 재시도 시 중복 저장 방지. `exactly-once per producer session` 보장.

**transactional.id**: Producer 재시작 후에도 동일 transactional.id를 통해 이전 미완료 트랜잭션을 중단(abort)하고 새 트랜잭션 시작. 멀티 파티션에 원자적 쓰기 가능. Consumer는 `isolation.level=read_committed`로 커밋된 트랜잭션만 읽기 가능.

세 가지를 모두 조합하면 Read-Process-Write 패턴에서 Exactly-Once Semantics(EOS)를 달성할 수 있습니다. 이는 Chapter 3에서 자세히 다룹니다.

</details>

---

<div align="center">

**[⬅️ 이전: ISR(In-Sync Replicas)](./02-isr-in-sync-replicas.md)** | **[홈으로 🏠](../README.md)** | **[다음: min.insync.replicas ➡️](./04-min-insync-replicas.md)**

</div>
