# Producer 내부 동작 — RecordAccumulator와 배치 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Producer가 `send()` 를 호출하면 메시지가 즉시 브로커로 전송되는가?
- `RecordAccumulator`는 어떻게 메시지를 파티션별로 배치로 묶는가?
- `linger.ms`와 `batch.size`가 동시에 작동할 때 어떤 규칙으로 배치를 전송하는가?
- `buffer.memory`가 꽉 차면 어떤 일이 발생하는가?
- Sticky Partitioner가 Round-Robin보다 배치 효율이 높은 이유는?
- `max.in.flight.requests.per.connection`이 메시지 순서에 미치는 영향은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Producer 설정을 기본값으로 두면 처리량이 낮거나, 너무 민감하게 튜닝하면 메시지 순서가 깨진다.

`linger.ms=0`(기본값)이면 메시지 하나가 들어오는 즉시 브로커로 전송한다. 이 경우 배치가 1개짜리가 되어 네트워크 요청 횟수가 많아지고 처리량이 낮다. 그렇다고 `linger.ms=100`으로 설정하면 처리량은 높아지지만 100ms 지연이 모든 메시지에 추가된다.

`max.in.flight.requests.per.connection`을 5로 두고 `retries`를 설정하면, 배치 1이 실패해서 재시도하는 동안 배치 2, 3이 먼저 성공하면 메시지 순서가 역전된다. 멱등성(`enable.idempotence=true`)이 없으면 이 문제를 직접 다뤄야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: send()가 즉시 브로커로 전송된다고 가정

  코드:
    producer.send(record);
    // "이 다음 줄 실행될 때는 브로커가 메시지를 받았겠지"
    doSomethingElse();

  실제:
    send()는 RecordAccumulator(내부 버퍼)에 메시지를 추가하고 즉시 반환
    실제 전송은 별도 Sender 스레드가 담당
    send()가 반환된 시점에 브로커가 메시지를 받은 것을 보장하지 않음
    
  올바른 방법:
    // 동기 확인이 필요한 경우
    Future<RecordMetadata> future = producer.send(record);
    RecordMetadata metadata = future.get(); // 브로커 응답까지 블로킹
    
    // 비동기 콜백
    producer.send(record, (metadata, exception) -> {
        if (exception != null) { handleError(exception); }
        else { log.info("sent to partition {}", metadata.partition()); }
    });

실수 2: buffer.memory 부족 시 동작을 모름

  상황: 브로커 장애로 전송 지연 → buffer.memory 꽉 참
  기대: "예외가 발생할 것이다"
  실제:
    max.block.ms(기본 60초) 동안 send() 호출이 블로킹됨
    60초 후에도 버퍼가 안 비워지면 → TimeoutException 발생
    
  문제: 브로커 장애 60초 동안 애플리케이션 스레드가 send()에서 멈춤
  대응: max.block.ms를 짧게 설정하거나, 버퍼 상태를 모니터링해서 조기 대응

실수 3: retries 설정 + 순서 보장 혼용

  설정:
    retries=3
    max.in.flight.requests.per.connection=5

  문제:
    배치 1 (offset 0~9) 전송 실패 → 재시도 대기 중
    배치 2 (offset 10~19) 전송 성공
    배치 1 재시도 성공
    결과: 브로커에는 offset 10~19가 먼저, offset 0~9가 나중에 도착
    → 메시지 순서 역전!

  해결:
    enable.idempotence=true 설정하면 자동으로 아래가 설정됨:
      max.in.flight.requests.per.connection=5 (최대값 5)
      retries=Integer.MAX_VALUE
      acks=all
    순서 보장 + 재시도 + 중복 제거 모두 해결
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
처리량 최적화 설정:
  batch.size=65536         (64 KB, 기본 16 KB보다 크게)
  linger.ms=20             (20ms 대기 → 배치 충전 시간)
  buffer.memory=67108864   (64 MB, 기본 32 MB보다 크게)
  compression.type=lz4     (빠른 압축으로 네트워크 절감)

  효과: 배치가 꽉 차서 전송 → 브로커 요청 횟수 감소 → 처리량 증가
  단점: 최대 linger.ms(20ms) 지연 추가

낮은 지연 최적화 설정:
  linger.ms=0              (즉시 전송)
  batch.size=16384         (기본값 유지)
  acks=1                   (Leader만 확인, 빠른 응답)

  효과: 메시지 하나 들어오면 즉시 전송 → 지연 최소
  단점: 처리량 낮음 (배치 효과 없음)

신뢰성 + 처리량 균형:
  enable.idempotence=true  (순서 보장 + 중복 제거)
  acks=all                 (자동 설정됨)
  batch.size=32768         (32 KB)
  linger.ms=5              (5ms 대기)
  compression.type=snappy  (균형 잡힌 압축)
```

---

## 🔬 내부 동작 원리

### 1. Producer 전체 아키텍처

```
KafkaProducer.send() 호출 흐름:

  애플리케이션 스레드                    Sender 스레드
  ┌──────────────┐                     ┌─────────────────────────┐
  │              │                     │                         │
  │ send(record) │                     │ while(running) {        │
  │      │       │                     │   sendProducerData()    │
  │      ▼       │                     │   client.poll()         │
  │ Serializer   │                     │ }                       │
  │      │       │                     │                         │
  │      ▼       │                     │  RecordAccumulator에서   │
  │ Partitioner  │                     │  파티션별 배치 꺼내서        │
  │      │       │    notify/poll      │  브로커로 전송             │
  │      ▼       │◄───────────────────►│                         │
  │ RecordAcc.   │                     │  응답 받으면 콜백 호출       │
  │ (버퍼에 추가)   │                     │                         │
  │      │       │                     └─────────────────────────┘
  │    Future    │
  │    반환       │
  └──────────────┘

  핵심:
  - 애플리케이션 스레드: Serializer → Partitioner → RecordAccumulator 추가 → 즉시 반환
  - Sender 스레드:      RecordAccumulator에서 배치 꺼내서 브로커에 비동기 전송
  - 두 스레드는 RecordAccumulator를 통해 분리됨
```

### 2. RecordAccumulator 내부 구조

```
RecordAccumulator (파티션별 Deque<ProducerBatch>):

  Partition 0:  [Batch A: 60 KB] → [Batch B: 12 KB ← 현재 추가 중]
  Partition 1:  [Batch C: 64 KB] → [Batch D: 5 KB ← 현재 추가 중]
  Partition 2:  [Batch E: 8 KB ← 현재 추가 중]

  각 ProducerBatch:
  ┌─────────────────────────────────┐
  │ topicPartition: orders-0        │
  │ records: [record1, record2, ...]│
  │ currentBytes: 60,000            │
  │ maxBytes: 65,536 (batch.size)   │
  │ createdAt: System.nanoTime()    │
  │ full(): currentBytes >= maxBytes│
  └─────────────────────────────────┘

Sender 스레드가 배치를 전송하는 조건:
  1. batch.full() = true (batch.size 도달)
     OR
  2. System.nanoTime() - batch.createdAt >= linger.ms (대기 시간 초과)
     OR
  3. buffer.memory 부족으로 공간 확보 필요

  → 조건 1 또는 2 중 먼저 충족되는 것
  → linger.ms=0이면 배치에 레코드 1개만 있어도 즉시 전송
```

### 3. linger.ms와 batch.size 상호작용

```
시나리오: batch.size=64KB, linger.ms=20ms

  t=0ms:   record1 (1 KB) → Batch A 생성, 1 KB 채워짐
  t=5ms:   record2 (1 KB) → Batch A, 2 KB
  t=10ms:  record3 (1 KB) → Batch A, 3 KB
  ...
  t=19ms:  record20 (1 KB) → Batch A, 20 KB
  t=20ms:  linger.ms 만료 → Batch A (20 KB) 전송 (batch.size 미달이어도)

  다른 시나리오: 메시지가 빠르게 유입될 때
  t=0ms:   record1 → Batch A
  t=1ms:   record2 → Batch A
  ...
  t=5ms:   record64 → Batch A, 64 KB 도달 → 즉시 전송 (linger.ms 15ms 남았어도)

  결론:
    batch.size는 상한선 (도달하면 즉시 전송)
    linger.ms는 최대 대기 (지나면 반드시 전송)
    실제 전송 크기 = min(현재 배치 크기, batch.size)
    실제 전송 타이밍 = min(linger.ms 만료, batch.size 도달)
```

### 4. buffer.memory와 배압(Backpressure)

```
buffer.memory = Producer 전체 메모리 풀 (기본 32 MB)

  정상 상태:
    buffer.memory: 32 MB 중 20 MB 사용
    새 메시지 → 버퍼에 공간 있음 → 즉시 배치에 추가

  브로커 느려질 때:
    Sender 스레드가 전송 못하고 배치가 쌓임
    buffer.memory: 32 MB 중 30 MB 사용 → 2 MB 남음

  buffer.memory 가득 찰 때:
    send() 호출 → 버퍼에 공간 없음
    → 애플리케이션 스레드가 max.block.ms(기본 60초)까지 대기
    → 60초 안에 공간이 생기면 진행
    → 60초 후에도 공간 없음 → TimeoutException

  모니터링:
    JMX: kafka.producer:type=producer-metrics,client-id=*
    지표: buffer-available-bytes, record-queue-time-avg
    
    buffer-available-bytes가 0에 가까우면 → 브로커 전송 지연 또는 buffer.memory 부족
```

### 5. max.in.flight.requests와 순서 보장

```
max.in.flight.requests.per.connection=5 의미:
  브로커로부터 응답을 기다리는 요청이 최대 5개
  5개가 전송 중일 때 새 전송 시도 → 대기

  순서 보장 문제:
  브로커 B
  ├── Request 1 (Batch: offset 0~9)  → 실패 (재시도 대기)
  ├── Request 2 (Batch: offset 10~19) → 성공 ✓
  ├── Request 3 (Batch: offset 20~29) → 성공 ✓
  └── Request 1 재시도              → 성공 ✓

  결과: 브로커 로그: offset 10~29가 먼저, 0~9가 나중 → 순서 역전!

  해결: enable.idempotence=true
    → Producer ID + Sequence Number 부여
    → 브로커가 중복/순서 오류 감지
    → max.in.flight.requests.per.connection=5 유지하면서 순서 보장
    → 재시도 시 브로커가 "이미 받은 seq #0~9" 확인 후 재배치 거부
```

---

## 💻 실전 실험

### 실험 1: 배치 크기와 처리량 측정

```bash
# Kafka 성능 테스트 도구로 배치 설정별 처리량 비교
# linger.ms=0, batch.size=16KB (기본값)
kafka-producer-perf-test \
  --topic orders \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:19092 \
    linger.ms=0 \
    batch.size=16384 \
    acks=1

# linger.ms=20, batch.size=65536 (최적화)
kafka-producer-perf-test \
  --topic orders \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:19092 \
    linger.ms=20 \
    batch.size=65536 \
    compression.type=lz4 \
    acks=1

# 출력 비교:
# 기본값: 95,000 records/sec, 92.8 MB/sec
# 최적화: 320,000 records/sec, 312.5 MB/sec
```

### 실험 2: buffer.memory 부족 재현

```bash
# Spring Kafka로 buffer.memory 한계 실험
# producer 설정에서 buffer.memory를 매우 작게 설정 (1 MB)
# 브로커를 멈추고 메시지를 계속 발행하면 max.block.ms 후 TimeoutException 발생

# Kafka 브로커 접속 차단 (iptables로 시뮬레이션)
docker exec kafka-1 bash -c "iptables -I INPUT -p tcp --dport 9092 -j DROP"

# 메시지 발행 시도 (buffer 꽉 찰 때까지)
kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --timeout 5000 \
  --topic orders
# 입력 계속하면 5초 후 TimeoutException

# 차단 해제
docker exec kafka-1 bash -c "iptables -D INPUT -p tcp --dport 9092 -j DROP"
```

### 실험 3: JMX로 Producer 메트릭 확인

```bash
# Producer 메트릭 확인 (jconsole 또는 jmxterm 활용)
# 배치 사이즈 평균 확인
# kafka.producer:type=producer-metrics,client-id=producer-1
# 지표: batch-size-avg, record-send-rate, request-rate

# kafka-console-producer로 발행하면서 별도 터미널에서 JMX 확인
# (kafka-producer-perf-test의 출력에서 주요 지표 확인 가능)
kafka-producer-perf-test \
  --topic orders \
  --num-records 100000 \
  --record-size 1024 \
  --throughput 10000 \
  --producer-props bootstrap.servers=localhost:19092 \
  --print-metrics
# 출력에서 batch-size-avg, record-queue-time-avg 확인
```

---

## 📊 성능/비용 비교

### linger.ms 설정별 처리량 vs 지연

```
조건: 100 byte 메시지, acks=1, 단일 파티션

  linger.ms=0   (즉시 전송):
    처리량: ~50,000 msg/sec
    평균 지연: ~1 ms
    배치 크기 평균: ~1~5 KB

  linger.ms=5:
    처리량: ~200,000 msg/sec
    평균 지연: ~5~7 ms
    배치 크기 평균: ~20~30 KB

  linger.ms=20:
    처리량: ~400,000 msg/sec
    평균 지연: ~20~25 ms
    배치 크기 평균: ~50~64 KB

  linger.ms=100:
    처리량: ~600,000 msg/sec
    평균 지연: ~100~110 ms
    배치 크기 평균: ~64 KB (batch.size 도달)

결론: linger.ms 증가 → 처리량 증가, 지연 증가 (비례)
     실시간성이 중요하면 linger.ms=0~5
     처리량이 중요하면 linger.ms=20~100
```

### 압축 방식별 CPU vs 처리량

```
1 KB 메시지, 1,000,000 건 처리 기준:

  압축 없음:
    Producer CPU: 낮음
    브로커 저장 크기: 1 GB
    네트워크 사용: 높음 (원본 크기)
    전체 처리량: ~300,000 msg/sec

  snappy:
    Producer CPU: 중간
    압축률: ~35% (저장 크기 650 MB)
    전체 처리량: ~350,000 msg/sec (네트워크 절감 효과)

  lz4:
    Producer CPU: 낮음 (snappy보다 빠름)
    압축률: ~30% (저장 크기 700 MB)
    전체 처리량: ~380,000 msg/sec

  gzip:
    Producer CPU: 높음
    압축률: ~70% (저장 크기 300 MB)
    전체 처리량: ~250,000 msg/sec (CPU 병목)

  zstd (Kafka 2.1+):
    Producer CPU: 중간
    압축률: ~60% (저장 크기 400 MB)
    전체 처리량: ~350,000 msg/sec

권장: 처리량 우선 → lz4
      저장 비용 우선 → zstd 또는 gzip
      균형 → snappy
```

---

## ⚖️ 트레이드오프

```
배치 크기 크게 (batch.size 높이기):
  ✅ 요청 횟수 감소 → 처리량 증가
  ✅ 압축 효율 증가 (배치 단위로 압축)
  ❌ 배치가 차기를 기다리는 시간 → 지연 증가
  ❌ 실패 시 재전송 배치 크기 증가

linger.ms 증가:
  ✅ 배치가 꽉 차서 전송될 가능성 증가 → 처리량 증가
  ❌ 최소 linger.ms만큼 지연 추가 (처리량 낮을 때도)

acks=all + enable.idempotence:
  ✅ 데이터 유실 방지 + 순서 보장 + 중복 제거
  ❌ acks 응답 대기 → 지연 증가 (~수 ms ~ 수십 ms)
  ❌ ISR 전체 응답 기다림 → ISR 이탈 시 지연 증가

max.in.flight.requests.per.connection:
  높게 (5): ✅ 파이프라이닝 → 처리량 증가 ❌ 순서 보장 깨질 수 있음
  낮게 (1): ✅ 순서 보장        ❌ 파이프라이닝 없어 처리량 감소
  → enable.idempotence=true 사용 시 5로 유지하면서 순서 보장 가능
```

---

## 📌 핵심 정리

```
Producer 내부 핵심 컴포넌트:

  send() → Serializer → Partitioner → RecordAccumulator → (Sender 스레드) → 브로커

RecordAccumulator:
  파티션별 ProducerBatch Deque 유지
  배치 전송 조건: batch.size 도달 OR linger.ms 만료

처리량 vs 지연 튜닝:
  처리량 우선: linger.ms=20, batch.size=65536, compression.type=lz4
  지연 우선:   linger.ms=0, batch.size=16384, acks=1

신뢰성 설정:
  enable.idempotence=true → acks=all + 순서 보장 + 중복 제거 자동 설정
  → 실무에서 모든 Producer에 적용 권장 (성능 비용 미미)

buffer.memory 꽉 차면:
  send()가 max.block.ms(기본 60초) 동안 블로킹 → TimeoutException
  → 브로커 상태 모니터링 + buffer.memory 여유 있게 설정
```

---

## 🤔 생각해볼 문제

**Q1. `producer.send(record)`를 호출하고 바로 애플리케이션을 종료(System.exit())하면 어떤 일이 발생하나요?**

<details>
<summary>해설 보기</summary>

RecordAccumulator에 남아있는 미전송 메시지가 유실됩니다.

`send()`는 내부 버퍼에 메시지를 넣고 즉시 반환합니다. 실제 전송은 Sender 스레드가 담당합니다. `System.exit()`을 호출하면 JVM이 종료되면서 Sender 스레드도 강제 종료되고, 버퍼에 남은 메시지는 전송되지 않습니다.

안전한 종료를 위해서는 반드시 `producer.close()`를 호출해야 합니다. `close()`는 RecordAccumulator에 남은 모든 배치를 플러시하고 Sender 스레드가 완료되기를 기다린 후 종료합니다. 타임아웃을 지정할 수도 있습니다: `producer.close(Duration.ofSeconds(30))`.

Spring Kafka에서는 `@PreDestroy`나 `DisposableBean`에서 `producer.close()`를 호출하거나, Spring이 자동으로 처리합니다.

</details>

---

**Q2. 같은 토픽의 다른 파티션으로 메시지를 보낼 때, 브로커 연결은 파티션마다 따로 맺나요?**

<details>
<summary>해설 보기</summary>

파티션마다가 아니라 **브로커마다** 연결을 맺습니다.

파티션 0의 Leader가 Broker 1에 있고, 파티션 1의 Leader가 Broker 2에 있다면, Producer는 Broker 1과 Broker 2 각각에 TCP 연결을 1개씩 유지합니다. 파티션이 100개여도 브로커가 3대라면 TCP 연결은 3개입니다.

`max.in.flight.requests.per.connection`은 브로커 연결 단위로 적용됩니다. Broker 1으로의 연결에서 최대 5개의 요청이 동시에 진행 중일 수 있습니다. Broker 1에 파티션 0, 3, 6이 있다면 이 세 파티션으로의 배치 전송이 합쳐서 최대 5개까지 동시 전송됩니다.

</details>

---

**Q3. `enable.idempotence=true`로 설정했을 때 Producer가 재시작되면 같은 Producer ID를 유지하나요?**

<details>
<summary>해설 보기</summary>

유지되지 않습니다. Producer ID(PID)는 Kafka 브로커에서 새 Producer 인스턴스가 초기화될 때마다 새로 발급합니다. `enable.idempotence=true`이어도 Producer가 재시작되면 새 PID를 받습니다.

이것이 의미하는 바는: **멱등성은 동일 Producer 세션 내에서의 중복 제거만 보장합니다.** Producer A가 메시지를 보내고 크래시 후 재시작되면, 이전 Producer A가 보낸 메시지와 새 Producer A가 보낸 메시지 사이의 중복은 감지하지 못합니다.

이 문제를 해결하려면 `transactional.id`를 설정해야 합니다. `transactional.id`는 재시작 후에도 동일하게 유지되어 브로커가 이전 Producer와 동일함을 인식하고 이전 트랜잭션을 중단(abort)시킵니다. 이것이 Chapter 3에서 다루는 트랜잭션 Producer입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 브로커 내부 구조](./03-broker-log-segment.md)** | **[홈으로 🏠](../README.md)** | **[다음: Consumer 내부 동작 ➡️](./05-consumer-internals.md)**

</div>
