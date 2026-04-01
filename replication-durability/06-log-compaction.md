# Log Compaction — 키 기반 최신값 보존

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Log Compaction은 일반 Retention 삭제와 어떻게 다른가?
- `clean` 영역과 `dirty` 영역의 차이는 무엇인가?
- Compaction 실행 중 Consumer는 영향을 받는가?
- 키별 최신값만 남기고 나머지를 삭제하는 내부 알고리즘은?
- `__consumer_offsets` 토픽이 Log Compaction을 사용하는 이유는?
- Tombstone(삭제 마커)이 필요한 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

일반 토픽은 `retention.ms`가 지나면 메시지가 삭제된다. 모든 이력이 필요 없고 "현재 상태"가 중요한 데이터라면 이 방식은 낭비다.

예를 들어 사용자 프로필 업데이트 이벤트가 1년 동안 수십 번 발행됐다면, 100번의 업데이트 이력이 아니라 **최신 프로필 1개**만 있어도 충분한 경우가 많다. Log Compaction은 키별로 최신 값만 유지하고 이전 값을 삭제해서 디스크를 절약한다.

`__consumer_offsets` 토픽, Kafka Streams의 상태 저장소 changelog 토픽 등이 Log Compaction으로 동작한다. 이를 모르면:
- Compacted 토픽에서 Consumer가 왜 일부 메시지를 받지 못하는지 혼란
- Tombstone 메시지의 의미를 모르고 "null 값 버그"로 오해
- `__consumer_offsets` 동작을 이해 못해 offset 관련 문제 진단 불가

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Compacted 토픽에서 이전 값을 읽으려고 offset을 리셋

  상황: cleanup.policy=compact 토픽에서 key="user-1"의 변경 이력 조회
  시도: --reset-offsets --to-earliest
  결과:
    Compaction이 이미 실행된 경우 "user-1"의 이전 값들은 삭제됨
    최신 값만 남아있음
    → 이력 추적이 필요하면 compact 대신 delete (일반 retention) 사용
    또는 cleanup.policy=compact,delete 조합 사용

실수 2: Tombstone 메시지를 버그로 오해

  상황: Consumer에서 value=null 메시지 수신
  오해: "Producer 버그로 null 값이 발행됐다"
  실제:
    Compacted 토픽에서 null 값 = Tombstone (삭제 마커)
    해당 키의 모든 데이터를 삭제하겠다는 의미
    Compaction 후 해당 키 자체가 토픽에서 제거됨
  
  처리 방법:
    if (record.value() == null) {
        // Tombstone: 해당 key 삭제 처리
        cache.remove(record.key());
    } else {
        cache.put(record.key(), record.value());
    }

실수 3: Compaction이 실시간으로 즉시 실행된다고 가정

  가정: "메시지를 발행하면 이전 값이 즉시 삭제된다"
  실제:
    Compaction은 백그라운드 스레드가 주기적으로 실행
    min.cleanable.dirty.ratio 설정에 따라 dirty/total 비율이 충분해야 실행
    Compaction 실행 전까지 이전 값들이 모두 남아있을 수 있음
    Consumer는 Compaction 전에는 이전 값들을 모두 읽을 수 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Compacted 토픽 올바른 사용:
  # 최신 상태가 중요한 데이터
  kafka-topics --bootstrap-server localhost:9092 \
    --create --topic user-profiles \
    --config cleanup.policy=compact \
    --config min.cleanable.dirty.ratio=0.5 \
    --config segment.ms=3600000    # 1시간마다 세그먼트 롤링
    --config delete.retention.ms=86400000  # Tombstone 24시간 보존

  Compaction + Retention 조합:
    --config cleanup.policy=compact,delete
    --config retention.ms=604800000  (7일)
    → 7일 이후는 retention으로 삭제
    → 7일 이내에서는 키별 최신값만 유지

Tombstone 올바른 처리:
  # Java Producer로 Tombstone 발행
  producer.send(new ProducerRecord<>("user-profiles", "user-1", null));
  # → key="user-1", value=null → Tombstone

  # Consumer에서 처리
  records.forEach(record -> {
      if (record.value() == null) {
          stateStore.delete(record.key());
      } else {
          stateStore.put(record.key(), record.value());
      }
  });
```

---

## 🔬 내부 동작 원리

### 1. Clean vs Dirty 영역

```
파티션 로그의 두 영역:

  [Clean 영역]              [Dirty 영역]         [Active 세그먼트]
  ────────────────────────────────────────────────────────────────►
  offset: 0 ~ 1000         offset: 1001 ~ 1500   offset: 1501 ~
  (이미 Compaction 완료)    (아직 Compaction 안 됨)  (현재 쓰는 중)

  Clean 영역 특성:
    키별로 이미 중복이 제거됨
    각 키의 최신 값만 남아있음

  Dirty 영역 특성:
    동일 키의 이전/최신 값이 혼재
    Compaction 대상

  Active 세그먼트:
    현재 쓰고 있는 세그먼트 → Compaction 대상 아님
    롤링 후 Dirty가 됨

  min.cleanable.dirty.ratio=0.5:
    dirty/(clean+dirty) >= 0.5 이면 Compaction 실행
    dirty 비율이 50% 이상일 때 Compaction 트리거
```

### 2. Compaction 알고리즘

```
Log Cleaner 스레드 동작:

  1. Dirty 영역 스캔 → Key → 최신 offset 매핑 생성
     (OffsetMap: 해시맵)
     key="user-1" → offset 1450 (최신)
     key="user-2" → offset 1480 (최신)
     key="user-3" → offset 1300 (최신)

  2. Clean 영역부터 Dirty 영역까지 순서대로 스캔
     각 레코드의 key가 OffsetMap에서 현재 레코드 offset = 최신?
       YES: 보존 (최신 값)
       NO:  제거 (이전 값)
     Tombstone (value=null): delete.retention.ms 후 제거
       그 전까지는 보존 (Consumer가 삭제 이벤트를 놓치지 않도록)

  3. 살아남은 레코드를 새 세그먼트에 기록
     새 세그먼트 = Compaction 완료된 Clean 영역

  4. 이전 Dirty 세그먼트 삭제

  예시:
  Compaction 전:
    offset 10: key="user-1" value="Alice"
    offset 50: key="user-2" value="Bob"
    offset 90: key="user-1" value="Alice2"   ← user-1 최신값
    offset 130: key="user-1" value="Alice3"  ← user-1 최신값 (90보다 최신)
    offset 140: key="user-3" value=null       ← Tombstone

  Compaction 후:
    offset 50: key="user-2" value="Bob"       ← 보존
    offset 130: key="user-1" value="Alice3"   ← 최신값만 보존
    (offset 10, 90 제거, offset 140 Tombstone은 delete.retention.ms 후 제거)
```

### 3. Compaction 중 Consumer 동작

```
Compaction은 백그라운드 스레드에서 비동기로 실행
Consumer는 Compaction 중에도 계속 읽기 가능

다만 읽기 일관성:
  Compaction 전: 동일 키의 이전 값들을 모두 읽을 수 있음
  Compaction 중: 일부 오래된 값은 이미 삭제됐을 수 있음
  Compaction 후: 키별 최신값만 존재

  Consumer가 --from-beginning으로 읽으면:
    Compaction 완료 후: 키별 최신값 1개씩만 받음
    각 키의 최신 상태를 빠르게 구성 가능
    이것이 Kafka Streams가 Compacted 토픽을 State Store 복구에 사용하는 이유

Log Cleaner 설정:
  log.cleaner.threads=1          (Compaction 스레드 수)
  log.cleaner.io.max.bytes.per.second  (Compaction I/O 제한)
  log.cleaner.dedupe.buffer.size=128MB  (OffsetMap 메모리)
```

### 4. __consumer_offsets 토픽이 Compaction을 사용하는 이유

```
__consumer_offsets:
  Consumer Group의 파티션별 커밋 offset을 저장하는 내부 토픽
  cleanup.policy=compact

  구조:
    key: GroupId + TopicPartition
    value: CommittedOffset

  예시:
    key="order-group:orders:0"  value=offset-150  (offset 150 커밋)
    key="order-group:orders:0"  value=offset-200  (offset 200 커밋)
    key="order-group:orders:0"  value=offset-300  (offset 300 커밋)

  Compaction 후:
    key="order-group:orders:0"  value=offset-300  (최신 커밋만 보존)

  이유:
    Consumer Group 재시작 시 마지막 커밋 offset 조회
    이전 커밋 이력은 불필요 → Compaction으로 최신값만 유지
    일반 Retention이면 오래된 Group의 offset이 만료 → Group이 처음부터 다시 읽는 문제
    Compaction이면 offset 정보가 영구 보존 (삭제하기 전까지)
```

---

## 💻 실전 실험

### 실험 1: Compacted 토픽 생성과 동작 확인

```bash
# Compacted 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic user-states \
  --partitions 1 \
  --replication-factor 1 \
  --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.01 \
  --config segment.ms=60000

# 동일 키로 여러 번 값 변경
kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic user-states \
  --property "parse.key=true" \
  --property "key.separator=:"
# 입력:
# user-1:{"name":"Alice","age":25}
# user-2:{"name":"Bob","age":30}
# user-1:{"name":"Alice","age":26}   ← user-1 업데이트
# user-1:{"name":"Alice Kim","age":26} ← user-1 또 업데이트

# Compaction 전 읽기 (4개 메시지 모두 보임)
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic user-states \
  --from-beginning \
  --property print.key=true \
  --max-messages 4

# Compaction 대기 (min.cleanable.dirty.ratio=0.01이므로 빠르게 실행)
sleep 120

# Compaction 후 읽기 (user-1은 최신값 1개만)
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic user-states \
  --from-beginning \
  --property print.key=true
# user-1:{"name":"Alice Kim","age":26}  ← 최신값만
# user-2:{"name":"Bob","age":30}
```

### 실험 2: Tombstone으로 키 삭제

```bash
# Tombstone 발행 (value 없이)
kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic user-states \
  --property "parse.key=true" \
  --property "key.separator=:" \
  --property "null.marker=NULL"
# 입력: user-2:NULL  ← Tombstone

# Compaction 후 user-2 키 완전 삭제 확인
sleep 120
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic user-states \
  --from-beginning \
  --property print.key=true
# user-1만 남음 (user-2는 삭제됨)
```

### 실험 3: __consumer_offsets 토픽 확인

```bash
# __consumer_offsets 토픽 설정 확인
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics \
  --entity-name __consumer_offsets \
  --describe
# cleanup.policy=compact 확인

# 특정 Consumer Group의 offset 정보 직접 확인
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic __consumer_offsets \
  --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" \
  --from-beginning 2>/dev/null | grep "order-consumer"
# 출력: [order-consumer,orders,0]::OffsetAndMetadata(offset=300, ...)
```

---

## 📊 성능/비용 비교

### cleanup.policy별 디스크 사용량

```
토픽: 사용자 프로필 (100만 사용자, 월 10회 업데이트)
메시지 크기: 1 KB

  cleanup.policy=delete, retention.ms=7일:
    7일치 데이터 = 100만 × 10회/월 × (7/30) = ~233만 메시지
    디스크: ~2.3 GB

  cleanup.policy=compact:
    키별 최신값 1개 = 100만 메시지
    디스크: ~1 GB (Compaction 완료 후)
    절감: ~57%

  cleanup.policy=compact,delete, retention.ms=30일:
    30일 이내 키별 최신값 유지
    오래된 Tombstone 자동 삭제
    디스크: ~1 GB ~ 2 GB (최신값 + 최근 변경 이력 일부)
```

### Compaction 영향도

```
log.cleaner.io.max.bytes.per.second 제한 없을 때:
  Compaction이 디스크 I/O를 과점유 → 브로커 성능 저하
  
  권장: log.cleaner.io.max.bytes.per.second=20971520 (20 MB/s)
        Compaction I/O를 제한해서 일반 읽기/쓰기 성능 보호

log.cleaner.dedupe.buffer.size=134217728 (128 MB):
  OffsetMap 크기 = Compaction 효율 결정
  너무 작으면: 여러 패스로 나눠서 실행 → 느림
  늘리면: Compaction 속도 향상이지만 JVM 힙 외의 메모리 사용
```

---

## ⚖️ 트레이드오프

```
cleanup.policy=compact:
  ✅ 디스크 절감 (키별 최신값만 보존)
  ✅ Consumer 재시작 시 최신 상태 빠르게 구성
  ✅ 무한 retention (Tombstone 전까지 키 보존)
  ❌ 변경 이력 없음 (감사 로그 목적에 부적합)
  ❌ Compaction 실행 전까지 이전 값 노출
  ❌ 동일 키 메시지 순서가 Compaction 후 변경될 수 있음

cleanup.policy=delete (일반 Retention):
  ✅ 시간순 이력 전체 보존
  ✅ 설정 단순
  ❌ 보존 기간 이후 데이터 영구 삭제
  ❌ 키 개수 많으면 디스크 사용량 큼

cleanup.policy=compact,delete 조합:
  ✅ 최신값 보존 + 오래된 키 정리 (Tombstone 처리)
  ✅ 두 정책의 장점 결합
  ❌ 동작 이해 복잡도 증가
```

---

## 📌 핵심 정리

```
Log Compaction 핵심:

1. 목적: 키별 최신값만 보존, 이전 값 삭제
   → 디스크 절감, 최신 상태 빠른 구성

2. Clean/Dirty 영역 구분
   Dirty: Compaction 대상 (이전/최신 값 혼재)
   Clean: Compaction 완료 (키별 최신값만)
   Active 세그먼트: 현재 쓰는 중 → Compaction 대상 아님

3. Tombstone (value=null)
   해당 키 삭제 마커
   delete.retention.ms 후 완전 제거

4. __consumer_offsets 토픽이 compact 사용
   Group의 최신 커밋 offset만 보존
   재시작 시 마지막 offset 조회에 활용

5. Compaction은 백그라운드 비동기 실행
   즉시 삭제 아님 → min.cleanable.dirty.ratio 기준
   Consumer는 Compaction 중에도 읽기 가능
```

---

## 🤔 생각해볼 문제

**Q1. Compacted 토픽에서 `--from-beginning`으로 Consumer를 시작하면 어떤 이점이 있나요?**

<details>
<summary>해설 보기</summary>

최신 상태를 빠르게 복원할 수 있습니다. Compaction이 완료된 토픽은 키별 최신값 1개씩만 남아있으므로, `--from-beginning`으로 읽으면 모든 키의 현재 상태를 전체 이력을 재생하지 않고도 구성할 수 있습니다.

Kafka Streams가 이 방식을 활용합니다. 상태 저장소(RocksDB)의 changelog 토픽을 Compacted 토픽으로 운영하고, 애플리케이션이 재시작되면 changelog 토픽을 처음부터 읽어 상태 저장소를 복원합니다. Compaction 덕분에 전체 이력 대신 최신 상태만 재생하므로 복원 시간이 짧습니다.

</details>

---

**Q2. 동일 키로 10,000번 업데이트했지만 Compaction이 아직 실행되지 않았다면 Consumer는 무엇을 받나요?**

<details>
<summary>해설 보기</summary>

10,000개의 메시지를 모두 받습니다. Compaction은 백그라운드에서 주기적으로 실행되므로 실행 전에는 모든 이전 값이 그대로 남아있습니다. Consumer는 10,000번의 업데이트 이력 전체를 순서대로 받게 됩니다.

이것은 Compaction의 중요한 특성입니다: **Compaction은 새 Consumer가 읽는 것에 영향을 주지만, 이미 읽고 있는 Consumer에게 메시지를 빼앗지 않습니다.** 즉, Compaction은 로그의 특정 세그먼트를 새 세그먼트로 교체하는 방식으로 동작하고, Consumer가 그 세그먼트를 넘어가면 Compaction 영향을 받습니다.

</details>

---

**Q3. `__consumer_offsets` 토픽의 Compaction이 실패하거나 지연되면 어떤 문제가 발생하나요?**

<details>
<summary>해설 보기</summary>

`__consumer_offsets` 토픽의 크기가 계속 증가합니다. 토픽은 파티션 50개로 구성되어 있고, 활발한 Consumer Group이 많은 클러스터에서는 지속적으로 메시지가 쌓입니다. Compaction이 지연되면 디스크 사용량이 증가하고, 브로커 재시작 시 `__consumer_offsets`를 읽어 Consumer Group 상태를 복원하는 시간이 늘어납니다.

실무에서 `log.cleaner.threads`를 1보다 높게 설정하거나, `__consumer_offsets`의 전용 Log Cleaner를 운영하는 방법으로 Compaction 속도를 높일 수 있습니다. 또한 비활성 Consumer Group은 `kafka-consumer-groups --delete`로 정리하면 불필요한 offset 레코드를 줄일 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Leader Election](./05-leader-election.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — 전달 보장 3단계 ➡️](../delivery-guarantee/01-delivery-semantics.md)**

</div>
