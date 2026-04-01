# Producer 멱등성 — Producer ID와 Sequence Number

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `enable.idempotence=true` 활성화 시 브로커가 중복을 감지하는 원리는?
- ProducerID(PID)와 Sequence Number가 각각 무엇을 식별하는가?
- 재시도(retry)로 인한 중복이 멱등성으로 제거되는 정확한 과정은?
- 멱등성이 보장하는 범위 — 단일 파티션, 단일 세션의 의미는?
- Producer 재시작 후 새 PID를 받으면 멱등성 보장이 어떻게 달라지는가?
- `max.in.flight.requests.per.connection=5`와 순서 보장의 관계는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

네트워크 오류로 Producer가 메시지 전송을 재시도할 때, 브로커가 이미 메시지를 받았는데 응답이 유실된 경우 동일 메시지가 두 번 저장될 수 있다. 이것이 At-Least-Once의 "중복 가능"이다.

`enable.idempotence=true`는 이 재시도 중복을 브로커 레벨에서 차단한다. 추가 비용은 미미하고(처리량 ~1% 미만 감소), 얻는 보장은 크다. Kafka 3.0부터는 기본값이 `true`로 변경됐다.

멱등성을 모르면:
- retries 설정 시 중복 방지를 위해 `max.in.flight.requests.per.connection=1`을 설정해서 처리량을 낮추는 불필요한 조치
- "브로커가 중복을 잡아줄 텐데 왜 UPSERT를 써야 하나?" → 멱등성 범위(단일 세션)를 모른 채 과신
- Producer 재시작 후 멱등성이 리셋되는 것을 모르고 두 세션 간 중복 발생

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 순서 보장을 위해 max.in.flight=1 설정

  배경: retries>0이면 네트워크 오류 후 재시도 중
        다른 배치가 먼저 도착해서 순서가 역전될 수 있음
  
  잘못된 해결책:
    max.in.flight.requests.per.connection=1
    → 동시에 1개 요청만 전송 → 파이프라이닝 없음 → 처리량 급감

  올바른 해결책:
    enable.idempotence=true
    → 브로커가 Sequence Number로 순서를 보장
    → max.in.flight.requests.per.connection=5 유지 가능
    → 처리량 유지 + 순서 보장 + 중복 제거 동시 달성

실수 2: 멱등성 = 완전한 Exactly-Once라는 오해

  현실:
    멱등성은 "단일 파티션, 단일 Producer 세션" 내에서만 중복 제거
    
    보장 안 되는 경우:
    - Producer 재시작 → 새 PID → 이전 세션 메시지와 구분 불가
    - 다른 파티션에 보낸 메시지 간의 원자성 → 트랜잭션 필요
    - Consumer 쪽 중복 처리 → 별도 처리 필요

실수 3: 멱등성 활성화 시 retries를 0으로 설정

  설정: enable.idempotence=true, retries=0
  문제:
    브로커 오류 시 재시도 없음 → 메시지 유실
    멱등성의 의미가 없어짐 (재시도가 없으니 중복도 없지만 유실 발생)
  
  올바른 설정:
    enable.idempotence=true → 자동으로 설정됨:
      retries=Integer.MAX_VALUE
      max.in.flight.requests.per.connection=5 (최대)
      acks=all
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
멱등성 활성화 (Kafka 3.0+ 기본값):
  spring:
    kafka:
      producer:
        properties:
          enable.idempotence: true
          # 자동 설정됨:
          # retries=Integer.MAX_VALUE
          # max.in.flight.requests.per.connection=5
          # acks=all

보장 범위 명확히 이해:
  [보장 O] 동일 Producer 세션 내 재시도 중복
           동일 파티션에 대한 순서 역전

  [보장 X] Producer 재시작 후 중복 (새 PID)
           다른 파티션 간의 원자성 (트랜잭션 필요)
           Consumer 처리 중복 (멱등 처리 로직 필요)

Producer 재시작 중복 방지:
  transactional.id 설정 → 재시작 후 동일 ID로 인식
  이전 미완료 트랜잭션 abort → 중복 방지
  단순 멱등성이 아닌 트랜잭션 영역 (Chapter 3.03 참고)
```

---

## 🔬 내부 동작 원리

### 1. ProducerID(PID)와 Sequence Number 구조

```
enable.idempotence=true 활성화 시:

  Producer 초기화:
    브로커(Transaction Coordinator 또는 일반 브로커)에게
    InitProducerId 요청 → PID(ProducerID) 발급

  PID: 클러스터에서 고유한 Producer 식별자
    Producer 인스턴스 생성마다 새 PID 발급
    재시작하면 새 PID (= 새 세션)

  Sequence Number: 파티션별 단조 증가 번호
    ProducerRecord 전송 시 파티션별 시퀀스 번호 부여
    (PID=100, Partition=0, Seq=0)  ← 첫 번째 배치
    (PID=100, Partition=0, Seq=1)  ← 두 번째 배치
    (PID=100, Partition=1, Seq=0)  ← Partition 1의 첫 번째 배치

  브로커가 유지하는 상태:
    각 파티션에 대해: lastSeq[PID] = 마지막 성공한 Sequence Number
```

### 2. 재시도 중복 제거 과정

```
중복이 발생하는 시나리오:

  t1: Producer → Batch(PID=100, Seq=5) 전송
  t2: 브로커가 Seq=5 기록 완료
  t3: 네트워크 오류 → Producer가 응답 못 받음
  t4: Producer 재시도 → 동일 Batch(PID=100, Seq=5) 재전송

  멱등성 없을 때:
    브로커: Seq=5를 다시 받음 → 그냥 기록 → 중복!

  멱등성 있을 때:
    브로커: lastSeq[PID=100, Partition=0] = 5
            수신한 Seq = 5
            Seq == lastSeq → "이미 받은 배치" → 무시
            Producer에게 DuplicateSequenceNumber 응답
    Producer: DuplicateSequenceNumber = 성공으로 처리
              "브로커가 이미 받았음"

  ┌────────────────────────────────────────────────────────┐
  │ 브로커 시퀀스 검사 로직:                                     │
  │                                                        │
  │ if (seq == lastSeq + 1) → 정상 순서 → 기록                │
  │ if (seq == lastSeq)     → 중복      → 무시 (OK 응답)      │
  │ if (seq < lastSeq)      → 오래된 중복 → 무시 (OK 응답)      │
  │ if (seq > lastSeq + 1)  → 순서 역전 → OutOfOrder에러      │
  └────────────────────────────────────────────────────────┘
```

### 3. 순서 보장 메커니즘

```
max.in.flight.requests.per.connection=5 + 멱등성:

  정상 전송 (순서대로):
    Batch-0 (Seq=0) ────────────────────► 브로커 기록 완료
    Batch-1 (Seq=1) ──────────────────────► 기록 완료
    Batch-2 (Seq=2) ────────────────────────► 기록 완료

  Batch-0 실패, Batch-1, 2 성공 시나리오:
    Batch-0 (Seq=0) ─── 실패 (재시도 대기)
    Batch-1 (Seq=1) ────────────────► 브로커 수신
    Batch-2 (Seq=2) ──────────────────── 브로커 수신

    브로커: Seq=1 수신 → lastSeq=0이어야 하는데 Seq=1
            OutOfOrderSequenceException!
            Batch-1, Batch-2 거부

    Batch-0 재시도 성공 → lastSeq=0 → Seq=1 허용
    Batch-1 재시도 → Seq=1 허용
    Batch-2 재시도 → Seq=2 허용

  결과: 순서가 역전되지 않음
        멱등성이 없으면 max.in.flight=1이 필요했지만
        멱등성 + Seq 번호로 max.in.flight=5 유지 가능

  주의: max.in.flight.requests.per.connection > 5이면
        멱등성이 있어도 순서 보장 불가
        (Kafka 내부 버퍼링 구조상 5가 한계)
```

### 4. 멱등성의 보장 범위 한계

```
[보장 범위 내]

  1. 동일 Producer 인스턴스의 재시도 중복
     → PID + Seq로 브로커가 감지

  2. 동일 파티션에서의 순서 역전
     → Seq 번호 순서 검사

[보장 범위 외]

  3. Producer 재시작 후 중복
     PID-1 세션: offset 100에 "order-1" 기록
     PID-1 크래시 → PID-2로 재시작
     PID-2 세션: 동일 메시지 재발행 시도
     → 브로커: PID-2는 PID-1과 다른 Producer
       → 중복 감지 불가 → 중복 기록
     해결: transactional.id 사용 (동일 ID로 재시작 인식)

  4. 다른 파티션 간 원자성
     Partition 0에 기록 성공, Partition 1에 기록 실패
     → 두 파티션에 걸친 원자적 쓰기 불가
     해결: 트랜잭션 Producer 사용

  5. Consumer 쪽 중복 처리
     멱등성은 Producer→브로커 구간만 보호
     Consumer가 처리 중 크래시 → 재처리 중복
     해결: 멱등 처리 로직 또는 EOS
```

### 5. 브로커의 PID 상태 관리

```
브로커가 유지하는 ProducerStateManager:

  파티션별로:
    Map<PID, ProducerStateEntry> producerStateMap

  ProducerStateEntry:
    lastSeq: 마지막으로 처리한 Sequence Number
    lastOffset: 마지막으로 기록한 offset
    lastTimestamp: 마지막 배치 타임스탬프
    epoch: Producer Epoch (트랜잭션용)

  PID 상태 보존 기간:
    transactional.id.expiration.ms=604800000 (7일, 기본값)
    7일 이상 비활성 PID는 브로커에서 정리
    → 7일 이후 동일 PID로 발행 시 브로커가 상태 없음 → 새 Seq 시작
    → 중복 감지 불가능 → 장기 비활성 Producer는 재시작 권장

  상태 저장 위치:
    메모리 + 파티션의 .snapshot 파일 (재시작 시 복구용)
```

---

## 💻 실전 실험

### 실험 1: 멱등성 활성화 전후 중복 발생 비교

```java
// 멱등성 없이 재시도 중복 시뮬레이션
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:19092");
props.put("enable.idempotence", "false");
props.put("retries", "3");
props.put("acks", "all");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// 의도적으로 응답 지연 브로커 시뮬레이션
// (실제로는 request.timeout.ms를 짧게 설정해서 재시도 유발)
props.put("request.timeout.ms", "100");  // 100ms 내 응답 없으면 재시도
```

### 실험 2: PID와 Sequence Number 확인

```bash
# 메시지 발행
kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --producer-property enable.idempotence=true \
  --producer-property client.id=test-producer

# .log 파일에서 PID와 Seq 확인
docker exec kafka-1 bash -c "
  kafka-dump-log \
    --files /var/kafka-logs/orders-0/00000000000000000000.log \
    --print-data-log 2>&1 | grep -E 'producerId|baseSequence'
"
# 출력:
# producerId: 12345          ← PID
# baseSequence: 0            ← Seq=0 (첫 번째 배치)
# producerId: 12345
# baseSequence: 1            ← Seq=1 (두 번째 배치)
```

### 실험 3: 멱등성 메트릭 확인

```bash
# Producer JMX 메트릭으로 중복 제거 확인
kafka-producer-perf-test \
  --topic orders \
  --num-records 100000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:19092 \
    enable.idempotence=true \
  --print-metrics 2>&1 | grep -E "record-error|retry"
# record-error-rate: 0.0       ← 오류 없음
# record-retry-rate: 0.0       ← 재시도 없음 (정상 환경)
```

---

## 📊 성능/비용 비교

### 멱등성 활성화 전후 처리량

```
조건: 파티션 3개, 복제 팩터 3, 1 KB 메시지, acks=all

enable.idempotence=false:
  처리량: ~200,000 msg/sec
  p99 지연: ~15 ms
  추가 동작: 없음

enable.idempotence=true:
  처리량: ~198,000 msg/sec (~1% 감소)
  p99 지연: ~15 ms
  추가 동작:
    InitProducerId 요청 (초기화 1회)
    각 배치에 PID + Seq 헤더 추가
    브로커의 Seq 검증

결론: 처리량 감소 ~1% 미만, 실질적으로 무시 가능
      Kafka 3.0+에서 기본값 true로 변경된 이유
```

### max.in.flight.requests 설정별 처리량

```
멱등성 없이 순서 보장을 위해 max.in.flight=1 설정:
  처리량: ~50,000 msg/sec (파이프라이닝 없음)
  순서: 보장
  중복: 재시도 시 발생 가능

멱등성 + max.in.flight=5 (권장):
  처리량: ~200,000 msg/sec (파이프라이닝 활용)
  순서: 보장 (Seq 번호 기반)
  중복: 동일 세션 재시도 중복 없음

→ 멱등성 활성화로 처리량 4배 향상이면서 순서 + 중복 제거 동시 달성
```

---

## ⚖️ 트레이드오프

```
enable.idempotence=true:
  ✅ 재시도 중복 자동 제거
  ✅ 단일 파티션 순서 보장
  ✅ max.in.flight=5로 높은 처리량 유지
  ✅ 처리량 감소 ~1% (무시 가능)
  ❌ 단일 세션 내에서만 보장
  ❌ Producer 재시작 후 중복 방지 불가 (transactional.id 필요)
  ❌ 다른 파티션 간 원자성 없음

enable.idempotence=false:
  ✅ 약간 높은 처리량 (~1%)
  ❌ 재시도 중복 발생 가능
  ❌ 순서 보장 위해 max.in.flight=1 필요 → 처리량 급감

결론: 특별한 이유가 없다면 항상 true
      Kafka 3.0+에서 기본값이 true인 이유
```

---

## 📌 핵심 정리

```
Producer 멱등성 핵심:

1. PID(ProducerID) + Seq(Sequence Number)로 브로커가 중복 감지
   (PID, Partition, Seq) 삼중쌍이 고유한 배치 식별자

2. 재시도 → 동일 Seq → 브로커가 무시 → 중복 없음

3. 보장 범위: 단일 파티션, 단일 Producer 세션
   Producer 재시작 = 새 PID = 새 세션 = 이전 세션과 구분 불가

4. max.in.flight.requests.per.connection=5 유지 가능
   Seq 번호 순서 검사로 순서 역전도 방지
   (5 초과 시 순서 보장 불가)

5. enable.idempotence=true 설정 시 자동으로:
   acks=all
   retries=Integer.MAX_VALUE
   max.in.flight.requests.per.connection=5
```

---

## 🤔 생각해볼 문제

**Q1. 브로커가 Sequence Number가 예상보다 크게 건너뛴 배치를 받으면 어떻게 처리하나요?**

<details>
<summary>해설 보기</summary>

`OutOfOrderSequenceException`을 발생시킵니다. 예를 들어 lastSeq=5인데 Seq=8이 들어오면 "6, 7은 어디 갔나?"라고 판단하고 에러를 반환합니다.

이 에러는 Producer가 중간 배치를 건너뛰고 전송했거나, 네트워크 패킷이 비정상적으로 재정렬된 경우 발생합니다. 멱등성이 활성화된 경우 `max.in.flight.requests.per.connection=5` 이하에서는 Seq 번호가 순서대로 브로커에 도달하도록 Kafka가 내부에서 관리합니다.

`OutOfOrderSequenceException`은 치명적 오류로 처리되어 Producer 인스턴스를 닫고 재시작해야 합니다. `enable.idempotence=true` 상태에서 이 오류는 매우 드물게 발생하며, 발생 시 버그 리포트 대상입니다.

</details>

---

**Q2. 멱등성이 활성화된 상태에서 브로커가 재시작되면 PID 상태가 유지되나요?**

<details>
<summary>해설 보기</summary>

네, 유지됩니다. 브로커는 PID 상태를 메모리와 함께 파티션 로그 디렉토리의 `.snapshot` 파일에 주기적으로 저장합니다. 브로커 재시작 시 이 파일을 읽어 PID 상태를 복원합니다.

따라서 브로커 재시작 후에도 동일 PID의 Producer가 재시도를 보내면 브로커가 이전 Seq 번호를 기억하고 중복을 감지합니다. 단, `.snapshot` 파일이 손상됐거나 없는 경우 PID 상태가 초기화될 수 있으며, 이 경우 Producer 재시작으로 새 PID를 받는 것이 안전합니다.

</details>

---

**Q3. 멱등성과 트랜잭션을 함께 사용할 수 있나요? 어떤 차이가 있나요?**

<details>
<summary>해설 보기</summary>

`transactional.id`를 설정하면 자동으로 멱등성이 활성화됩니다. 트랜잭션 Producer는 멱등성의 상위 집합입니다.

**멱등성만**: 단일 파티션 내 재시도 중복 제거 + 순서 보장. Producer 세션 범위.

**트랜잭션 포함**: 멱등성 기능 전부 + 멀티 파티션 원자적 쓰기 + Consumer offset 커밋 원자적 포함 + Producer 재시작 후에도 동일 transactional.id로 인식 (이전 미완료 트랜잭션 abort).

결론: 단순 재시도 중복만 방지하면 `enable.idempotence=true`로 충분. 멀티 파티션 원자성이나 Producer 재시작 후 보장이 필요하면 `transactional.id`를 추가로 설정합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 전달 보장 3단계](./01-delivery-semantics.md)** | **[홈으로 🏠](../README.md)** | **[다음: 트랜잭션 Producer ➡️](./03-transactional-producer.md)**

</div>
