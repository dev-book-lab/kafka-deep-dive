# Exactly-Once in Kafka Streams — processing.guarantee

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `processing.guarantee=exactly_once_v2`가 내부에서 어떻게 동작하는가?
- EOS-V1(`exactly_once`)과 EOS-V2(`exactly_once_v2`)의 차이는?
- Kafka Streams EOS가 일반 Producer/Consumer EOS와 어떻게 다른가?
- EOS 활성화 시 처리량이 얼마나 감소하고 어떻게 최소화하는가?
- `at_least_once` vs `exactly_once_v2` 선택 기준은?
- EOS 상태에서 스트림 애플리케이션이 크래시하면 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka Streams로 주문 집계 파이프라인을 구축할 때 `at_least_once`(기본값)이면 중복 처리가 발생할 수 있다. 집계 카운터가 실제보다 더 많이 올라가거나, 이미 처리한 주문이 다시 집계될 수 있다.

`exactly_once_v2`를 설정하면 Kafka Streams가 내부적으로 트랜잭션 Producer와 `read_committed` Consumer를 자동으로 구성해서 처리 중 크래시가 발생해도 정확히 한 번만 처리됨을 보장한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: at_least_once로 재무 집계 파이프라인 운영

  설정: processing.guarantee=at_least_once (기본값)
  
  문제 시나리오:
    Task 0가 주문 10개를 처리하고 집계 결과를 출력 토픽에 발행
    결과 발행 직후 offset 커밋 전 크래시
    재시작 후 Task 0는 같은 10개 주문을 다시 처리
    → 집계 결과가 2배로 잘못됨

  처리: 매출 합산이 실제의 2배 → 재무 보고서 오류

  올바른 설정:
    processing.guarantee=exactly_once_v2
    → 집계 + 커밋이 원자적으로 처리

실수 2: exactly_once (V1)를 최신 Kafka에서 사용

  V1 문제:
    각 StreamTask마다 독립적인 트랜잭션 Producer 생성
    Task 100개 = Transaction Coordinator 연결 100개
    브로커 과부하, 높은 오버헤드

  올바른 설정:
    Kafka 2.5+ → processing.guarantee=exactly_once_v2 사용
    Kafka 2.5 미만 → exactly_once (V1) 사용 불가피

실수 3: EOS와 외부 시스템 연계에서 완전한 Exactly-Once를 기대

  착각: "exactly_once_v2 설정하면 DB 저장도 Exactly-Once"
  현실: Kafka Streams EOS는 Kafka 토픽 간에만 보장
        output 토픽 → DB 저장은 Kafka EOS 밖
        
  DB 저장 중복 방지: 별도 멱등 처리 (UPSERT, 처리 이력 테이블) 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
EOS 설정:
  spring:
    kafka:
      streams:
        properties:
          processing.guarantee: exactly_once_v2    # Kafka 2.5+
          # commit.interval.ms: 100               # EOS에서 자동 관리
          # 아래는 EOS가 자동 설정:
          # isolation.level=read_committed
          # enable.idempotence=true
          # transactional.id={appId}-{taskId}

EOS 선택 기준:
  exactly_once_v2 선택:
    ✅ 집계/카운터 결과의 정확성이 비즈니스 핵심
    ✅ 재무, 정산, 재고 집계
    ✅ 처리량 20~30% 감소를 감당 가능
    ✅ Kafka 2.5 이상 사용

  at_least_once 유지:
    ✅ 처리 로직이 멱등 (UPSERT, 카운트 오차 허용)
    ✅ 로그 분석, 비실시간 집계
    ✅ 처리량이 최우선
    ✅ 외부 시스템(DB) 연계 위주
```

---

## 🔬 내부 동작 원리

### 1. EOS-V1 (exactly_once)

```
구조: Task당 독립적인 트랜잭션 Producer

  StreamTask 0 → TransactionalProducer(id=appId-0)
  StreamTask 1 → TransactionalProducer(id=appId-1)
  StreamTask 2 → TransactionalProducer(id=appId-2)
  ...
  StreamTask N → TransactionalProducer(id=appId-N)

  Task 100개 = Transaction Coordinator 연결 100개

  각 Task 처리 사이클:
    1. beginTransaction()
    2. Consumer.poll() (read_committed)
    3. 처리 + output 토픽 produce()
    4. sendOffsetsToTransaction(consumer offsets)
    5. commitTransaction()
    → 처리 + output + offset 커밋 원자적

  문제:
    Task 수가 많으면 Transaction Coordinator 부하 × Task 수
    Kafka 2.5 이전의 방식
```

### 2. EOS-V2 (exactly_once_v2)

```
구조: Thread당 1개 트랜잭션 Producer (KIP-447)

  StreamThread-1 (Task 0, 1, 2 처리):
    공유 TransactionalProducer(id=appId-thread-1)

  StreamThread-2 (Task 3, 4, 5 처리):
    공유 TransactionalProducer(id=appId-thread-2)

  인스턴스당 Transaction Coordinator 연결 = 스레드 수 (보통 1~2개)
  Task 100개여도 TC 연결은 2개!

  처리 사이클:
    StreamThread-1이 Task 0, 1, 2를 순차 처리:
    1. beginTransaction()
    2. Task 0: poll + process + produce
    3. Task 1: poll + process + produce
    4. Task 2: poll + process + produce
    5. sendOffsetsToTransaction(Task 0,1,2 offsets)
    6. commitTransaction()
    → 여러 Task의 처리가 하나의 트랜잭션에 묶임

  V1 대비 개선:
    TC 연결 수 극감 (Task 수 → 스레드 수)
    브로커 TC 부하 감소
    처리량 오버헤드 감소 (~40% → ~20%)
```

### 3. 크래시 후 복구 시나리오

```
EOS 활성화 상태에서 크래시:

  t=1: Task 0, 처리 중 (주문 10개, 집계 결과 발행)
  t=2: output 토픽에 미커밋 트랜잭션 데이터 기록됨
  t=3: commitTransaction() 호출 전 크래시!

  복구:
    새 인스턴스가 시작, 동일 transactional.id로 initTransactions()
    Transaction Coordinator: "이전 트랜잭션 미완료 감지"
    → 이전 트랜잭션 자동 abort
    → output 토픽의 미커밋 데이터: ABORT 마커 추가

  Consumer (read_committed):
    abort된 데이터 무시
    → 마치 처리가 안 된 것처럼 동작

  새 인스턴스:
    마지막 커밋된 Consumer offset부터 재처리
    → 주문 10개 다시 처리 → 집계 결과 다시 발행 → commitTransaction()
    → output 토픽: COMMIT 마커 → Consumer가 읽을 수 있음

  결과: 주문 10개가 정확히 1번만 집계됨 (Exactly-Once)
```

### 4. EOS가 자동 구성하는 설정들

```
processing.guarantee=exactly_once_v2 설정 시 자동 설정:

  Consumer 설정:
    isolation.level=read_committed
    → 커밋된 트랜잭션 메시지만 읽음
    → abort된 이전 실행의 output 무시

  Producer 설정:
    enable.idempotence=true
    transactional.id={application.id}-{taskId 또는 threadId}
    acks=all
    retries=Integer.MAX_VALUE

  commit.interval.ms:
    EOS에서는 트랜잭션 단위로 커밋
    commit.interval.ms 설정이 트랜잭션 빈도 결정
    낮게: 더 자주 커밋 → 재처리 범위 감소 → 처리량 감소
    높게: 드물게 커밋 → 더 큰 트랜잭션 배치 → 처리량 증가

  주의: EOS 설정 후 브로커 버전 확인
    exactly_once_v2: Kafka 2.5 브로커 이상 필요
    이전 버전 브로커: 자동으로 V1으로 폴백
```

### 5. EOS 성능 영향

```
처리량 비교:

  at_least_once:
    처리량: ~300,000 msg/sec
    지연(p99): ~10 ms

  exactly_once (V1):
    처리량: ~180,000 msg/sec (-40%)
    지연(p99): ~25 ms
    TC 연결 수: Task 수에 비례

  exactly_once_v2:
    처리량: ~240,000 msg/sec (-20%)
    지연(p99): ~18 ms
    TC 연결 수: 스레드 수 (고정, 소수)

  최적화 방법:
    commit.interval.ms 증가 → 배치 트랜잭션 → 처리량 회복
    linger.ms + batch.size 최적화 (일반 Producer 최적화와 동일)
```

---

## 💻 실전 실험

### 실험 1: EOS 설정 전후 처리량 비교

```bash
# at_least_once 처리량 측정
# 토픽에 100만 메시지 발행 후 Streams 처리 시간 측정
kafka-producer-perf-test \
  --topic input-topic \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092

# Kafka Streams 처리 시간 측정 (at_least_once)
# → processing.guarantee=at_least_once 설정 후 측정

# Kafka Streams 처리 시간 측정 (exactly_once_v2)
# → processing.guarantee=exactly_once_v2 설정 후 측정

# 결과 비교: Consumer Lag 소진 시간 차이
watch -n 1 'kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group streams-app'
```

### 실험 2: 크래시 후 EOS 복구 확인

```bash
# 출력 토픽에 메시지 수 모니터링
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group output-consumer

# Kafka Streams 앱 강제 종료 (처리 중)
kill -9 $(pgrep -f "streams-app")

# 자동 재시작 후 output 토픽 메시지 수 확인
# at_least_once: 중복 메시지 가능 (메시지 수 > 입력 수)
# exactly_once_v2: 정확한 메시지 수 (중복 없음)
kafka-run-class kafka.tools.GetOffsetShell \
  --bootstrap-server localhost:9092 \
  --topic output-topic --time -1
```

### 실험 3: 트랜잭션 상태 모니터링

```bash
# __transaction_state 토픽에서 Streams 트랜잭션 확인
kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic __transaction_state \
  --formatter "kafka.coordinator.transaction.TransactionLog\$TransactionLogMessageFormatter" \
  --from-beginning 2>/dev/null | grep "streams-app"
# 트랜잭션 상태: CompleteCommit / CompleteAbort / Ongoing
```

---

## 📊 성능/비용 비교

### EOS 설정별 성능 영향 요약

```
Kafka Streams + 1 KB 메시지 + 파티션 12개:

  at_least_once:
    처리량:     ~300,000 msg/sec
    지연 p99:   ~10 ms
    TC 연결:    0개 (트랜잭션 없음)

  exactly_once (V1):
    처리량:     ~180,000 msg/sec (-40%)
    지연 p99:   ~25 ms
    TC 연결:    Task 수 (12개)

  exactly_once_v2:
    처리량:     ~240,000 msg/sec (-20%)
    지연 p99:   ~18 ms
    TC 연결:    스레드 수 (1~2개)

  exactly_once_v2 + commit.interval.ms=500:
    처리량:     ~270,000 msg/sec (-10%)
    지연 p99:   ~520 ms (커밋 간격)
    → 처리량 회복, 지연 약간 증가
```

---

## ⚖️ 트레이드오프

```
exactly_once_v2:
  ✅ 정확한 처리 보장 (재무, 집계, 정산)
  ✅ V1 대비 오버헤드 절반
  ❌ 처리량 ~20% 감소
  ❌ Kafka 2.5 이상 필요
  ❌ 외부 시스템 연계는 별도 멱등 처리 필요

at_least_once:
  ✅ 최고 처리량
  ✅ 구현 단순
  ❌ 중복 처리 가능
  → 처리 로직이 멱등하면 사실상 Exactly-Once와 동일

EOS commit.interval.ms:
  짧게 (100ms): 재처리 범위 최소 ↔ TC 트랜잭션 빈도 증가 → 처리량 감소
  길게 (1000ms): 처리량 향상 ↔ 장애 시 더 많은 재처리 범위
```

---

## 📌 핵심 정리

```
Kafka Streams EOS 핵심:

1. processing.guarantee=exactly_once_v2 (Kafka 2.5+)
   스레드당 1개 트랜잭션 Producer (V1의 Task당 1개 대비 개선)
   자동으로 read_committed Consumer + idempotent Producer 구성

2. EOS 동작:
   poll + process + output produce + offset 커밋 = 하나의 트랜잭션
   크래시 → 트랜잭션 abort → output 무시 → offset 재처리
   → 정확히 한 번 처리 보장

3. V1 vs V2:
   V1: Task당 TC 연결 → 대규모 Task에서 오버헤드 큼 (-40%)
   V2: 스레드당 TC 연결 → 오버헤드 감소 (-20%)

4. EOS 범위:
   Kafka 토픽 간에만 보장
   외부 DB, API는 별도 멱등 처리 필요

5. 선택 기준:
   정확성 우선 (재무, 정산): exactly_once_v2
   처리량 우선 + 멱등 로직: at_least_once
```

---

## 🤔 생각해볼 문제

**Q1. `exactly_once_v2`를 사용하는데 출력 토픽의 Consumer가 `read_uncommitted`를 사용한다면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

EOS 보장이 깨집니다. `read_uncommitted` Consumer는 트랜잭션이 commit됐는지 abort됐는지 확인하지 않고 데이터를 읽습니다. 따라서 크래시 후 abort된 트랜잭션의 데이터도 읽게 됩니다.

이것은 Kafka Streams의 EOS가 "Producer는 트랜잭션으로 쓰고, Consumer는 `read_committed`로 읽어야" 완성된다는 의미입니다. 양쪽이 모두 맞아야 합니다.

실무에서 Kafka Streams의 출력 토픽을 소비하는 Consumer가 Spring Kafka로 구현됐다면, 반드시 `isolation.level=read_committed`를 설정해야 합니다. 이 설정을 누락하면 EOS 파이프라인을 구축했어도 최종 Consumer에서 중복이 발생합니다.

</details>

---

**Q2. Kafka Streams 애플리케이션이 EOS 모드에서 처리하는 도중 Kafka 브로커가 재시작되면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

트랜잭션 코디네이터가 있는 브로커가 재시작되면 미완료 트랜잭션은 `transaction.timeout.ms`(기본 1분) 후 자동 abort됩니다. 트랜잭션 코디네이터 역할이 새 브로커로 넘어가면 이전 미완료 트랜잭션을 확인하고 처리합니다.

Kafka Streams 애플리케이션은 `TransactionAbortedException`이나 `ProducerFencedException`을 받아서 현재 트랜잭션을 abort하고 재시도합니다. `retries=Integer.MAX_VALUE`(EOS 기본값)이므로 재시도 후 정상화됩니다.

브로커 재시작 중 일시적으로 처리 지연이 발생하지만 데이터 유실이나 중복 없이 복구됩니다. 이것이 EOS의 핵심 가치입니다.

</details>

---

**Q3. `at_least_once`를 사용하는 기존 Kafka Streams 앱을 `exactly_once_v2`로 전환하면 어떤 작업이 필요한가요?**

<details>
<summary>해설 보기</summary>

설정 변경만으로 전환이 가능합니다. 코드 변경은 필요 없습니다.

`processing.guarantee=exactly_once_v2`를 설정하고 애플리케이션을 재시작하면 Kafka Streams가 자동으로 내부 트랜잭션 설정을 구성합니다.

단, 전환 시 주의사항:

1. **출력 토픽의 Consumer 확인**: 출력 토픽을 소비하는 모든 Consumer에 `isolation.level=read_committed`를 설정해야 EOS가 완전히 작동합니다.

2. **브로커 버전 확인**: Kafka 2.5 이상 브로커가 필요합니다. 이하 버전이면 V1으로 자동 폴백되어 성능 오버헤드가 더 큽니다.

3. **처리량 계획**: 약 20% 처리량 감소를 예상하고 Consumer 수나 파티션 수를 미리 조정합니다.

4. **상태 저장소 초기화 필요 없음**: 기존 RocksDB 상태는 그대로 유지됩니다. EOS 전환이 상태를 초기화하지는 않습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 윈도우 연산](./03-window-operations.md)** | **[홈으로 🏠](../README.md)** | **[다음: 이벤트 기반 아키텍처 패턴 ➡️](./05-event-driven-patterns.md)**

</div>
