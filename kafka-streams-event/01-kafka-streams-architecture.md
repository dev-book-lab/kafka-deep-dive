# Kafka Streams 아키텍처 — Topology와 상태 저장소

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Topology`와 `StreamTask`의 관계는 무엇인가?
- RocksDB 로컬 상태 저장소가 Changelog 토픽으로 내구성을 보장하는 원리는?
- Task 재배치 시 RocksDB 상태를 어떻게 복원하는가?
- Standby Replica란 무엇이고 왜 복구 시간을 단축시키는가?
- Kafka Streams 애플리케이션이 일반 Producer/Consumer 기반 애플리케이션과 어떻게 다른가?
- 상태 저장소의 크기가 커질 때 어떤 운영 문제가 발생하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka Streams는 Kafka 클라이언트 라이브러리만으로 스트림 처리를 구현한다. Flink나 Spark Streaming처럼 별도 클러스터 없이 Spring Boot 애플리케이션에 의존성 하나만 추가하면 된다.

하지만 내부 구조를 모르면:
- RocksDB 디스크가 가득 차는데 왜인지 모름
- Task 재배치 후 복구 시간이 길어지는 이유를 모름
- Changelog 토픽의 retention을 잘못 설정해서 복구 불가 상황 발생
- Standby Replica 없이 운영하다 장애 시 수십 분의 다운타임

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Changelog 토픽의 retention을 짧게 설정

  문제:
    상태 저장소의 Changelog 토픽은 compact + delete 정책으로 운영
    retention.ms를 1일로 설정하면
    → 1일 이상 된 데이터가 Changelog에서 삭제
    → Task 재배치 후 Changelog에서 상태 복원 시 불완전한 복원
    → 잘못된 집계 결과 (예: 누락된 이벤트로 인해 카운트 부족)

  올바른 설정:
    Changelog 토픽은 cleanup.policy=compact 유지
    delete.retention.ms는 충분히 길게 (기본값 유지 권장)
    상태 저장소가 너무 커지면 Compaction으로 크기 관리

실수 2: num.standby.replicas 설정 없이 운영

  상황: Kafka Streams 앱 3개 인스턴스 중 1개 장애
  num.standby.replicas=0 (기본값):
    장애 인스턴스의 Task → 다른 인스턴스로 재배치
    Changelog에서 RocksDB 복원 시작
    복원 시간: 수십 MB ~ 수 GB 데이터 재생 → 수십 초 ~ 수 분
    복원 완료 전까지 해당 Task 처리 중단

  num.standby.replicas=1 설정:
    각 Task의 Standby Replica가 Changelog를 미리 구독하여 RocksDB 최신 유지
    장애 시 Standby가 즉시 Active Task로 전환
    복구 시간: 수 초 (이미 최신 상태 유지 중)

실수 3: 로컬 상태 저장소 디스크 경로를 기본값으로 운영

  기본 경로: /tmp/kafka-streams (임시 디렉토리!)
  문제:
    시스템 재시작 시 /tmp 초기화 → 상태 저장소 전체 삭제
    → 다음 시작 시 Changelog에서 전체 복원 (긴 복구 시간)
  
  올바른 설정:
    state.dir=/var/kafka-streams (영구 디렉토리)
    서버 재시작 후 이전 상태 즉시 사용 가능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
운영 환경 권장 설정:
  # application.yml (Spring Kafka Streams)
  spring:
    kafka:
      streams:
        application-id: order-processor
        bootstrap-servers: localhost:9092
        properties:
          state.dir: /var/kafka-streams          # 영구 디렉토리
          num.standby.replicas: 1                # 빠른 장애 복구
          processing.guarantee: exactly_once_v2  # EOS
          commit.interval.ms: 100                # 100ms마다 상태 커밋
          cache.max.bytes.buffering: 10485760    # 10 MB 캐시

Changelog 토픽 설정 확인:
  # 자동 생성된 Changelog 토픽 확인
  kafka-topics --bootstrap-server localhost:9092 \
    --describe --topic order-processor-counts-changelog
  # cleanup.policy=compact 확인
  # retention.bytes 적절한지 확인 (상태 저장소 크기 기준)

상태 저장소 크기 모니터링:
  du -sh /var/kafka-streams/order-processor/
  # 너무 크면 Compaction 설정 조정 또는 파티션 재분배
```

---

## 🔬 내부 동작 원리

### 1. Topology: 처리 그래프

```
Kafka Streams 애플리케이션 = Topology (방향성 비순환 그래프)

  Source Node → Processor Node(s) → Sink Node

  예시: 주문 금액 합산 파이프라인

  [Source Node]
    토픽: "orders"
    역할: Kafka 파티션에서 레코드 읽기
         │
         ▼
  [Processor Node: Filter]
    조건: amount > 0 인 주문만 통과
         │
         ▼
  [Processor Node: Aggregate]
    그룹화: userId 기준
    집계: 합산 (total amount per user)
    상태 저장소: RocksDB ("user-totals")
         │
         ▼
  [Sink Node]
    토픽: "user-order-totals"
    역할: 집계 결과를 Kafka에 발행

  코드:
    StreamsBuilder builder = new StreamsBuilder();
    KStream<String, Order> orders = builder.stream("orders");
    orders
      .filter((k, v) -> v.getAmount() > 0)
      .groupByKey()
      .aggregate(
          () -> 0L,
          (userId, order, total) -> total + order.getAmount(),
          Materialized.as("user-totals")
      )
      .toStream()
      .to("user-order-totals");
    Topology topology = builder.build();
```

### 2. StreamTask: 파티션과 처리의 1:1 대응

```
Kafka Streams 파티션 할당:

  입력 토픽 "orders"에 파티션 4개
  → StreamTask 4개 생성 (Task 0, 1, 2, 3)
  → 각 Task는 정확히 하나의 파티션을 처리

  Kafka Streams 인스턴스 2개:
    Instance 1: Task 0, Task 1 담당
    Instance 2: Task 2, Task 3 담당

  각 Task의 상태 저장소:
    Task 0: RocksDB 인스턴스 0 (/var/kafka-streams/.../0/)
    Task 1: RocksDB 인스턴스 1 (/var/kafka-streams/.../1/)
    ...

  핵심: 동일한 userId의 이벤트는 항상 동일 파티션 → 동일 Task
        → 해당 Task의 RocksDB에 누적 상태가 쌓임
        → 다른 Task와 상태 공유 없음 (로컬 상태, 분산 처리)
```

### 3. RocksDB + Changelog 토픽: 내구성 보장

```
상태 저장소 업데이트 흐름:

  레코드 처리 → RocksDB 업데이트
                     │
                     ├─→ 즉시: 로컬 RocksDB에 쓰기
                     └─→ 비동기: Changelog 토픽에 변경사항 발행

  Changelog 토픽:
    이름: {application-id}-{store-name}-changelog
    예:   order-processor-user-totals-changelog
    형식: key=userId, value=현재 합산값
    정책: cleanup.policy=compact (키별 최신값만 보존)

  복구 시나리오:
    Task 0이 Instance 1에서 Instance 2로 이동
    Instance 2: Task 0의 RocksDB 없음
    → Changelog 토픽 구독 시작
    → offset 0부터 읽어서 RocksDB 재구성
    → Compaction된 Changelog라면 키별 최신값만 재생
    → 복구 완료 → Task 0 처리 재개
```

### 4. Standby Replica: 빠른 장애 복구

```
num.standby.replicas=1 설정 시:

  Active Task 0 (Instance 1)      Standby Task 0 (Instance 2)
       │                                    │
       │ Changelog 발행              Changelog 구독 (실시간)
       ▼                                    ▼
  RocksDB 최신 상태            RocksDB 복사본 (거의 최신)
  + 처리 중                    + 대기 중 (처리 안 함)

  Instance 1 장애 시:
    Standby Task 0 → Active Task 0 즉시 승격
    RocksDB: 이미 최신 상태에 가까움 (약간의 gap만 Changelog에서 보충)
    복구 시간: 수 초 (Standby 없을 때 수십 초 ~ 수 분 대비)

  비용:
    각 Standby는 Changelog를 구독하여 RocksDB 유지
    → 추가 디스크 + CPU 비용
    → 인스턴스당 충분한 리소스 필요
```

### 5. 상태 저장소 스냅샷

```
RocksDB 스냅샷 (commit.interval.ms마다):

  commit.interval.ms=100 (100ms마다 커밋):
    현재 처리 위치(offset) + RocksDB 상태를 원자적으로 커밋
    Changelog 토픽에 Kafka offset 체크포인트 기록
    장애 후 복구 시 체크포인트부터 재생 (처음부터 재생 불필요)

  state.dir 디렉토리 구조:
    /var/kafka-streams/{application-id}/
      {taskId}/
        rocksdb/
          {store-name}/        ← RocksDB 파일들
        .checkpoint            ← 마지막 커밋 offset

  스냅샷 덕분에:
    같은 서버 재시작: .checkpoint 기준으로 소량만 Changelog 재생
    서버 이전: Changelog 전체 재생 필요 (Standby로 완화)
```

---

## 💻 실전 실험

### 실험 1: Kafka Streams 기본 파이프라인 실행

```java
// Spring Boot + Kafka Streams 기본 설정
@Configuration
@EnableKafkaStreams
public class StreamsConfig {
    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public KafkaStreamsConfiguration streamsConfig() {
        Map<String, Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "order-processor");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.STATE_DIR_CONFIG, "/var/kafka-streams");
        props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);
        return new KafkaStreamsConfiguration(props);
    }
}

@Component
public class OrderAggregator {
    @Autowired
    void buildTopology(StreamsBuilder builder) {
        builder.stream("orders", Consumed.with(Serdes.String(), orderSerde))
               .groupByKey()
               .count(Materialized.as("order-counts"))
               .toStream()
               .to("order-count-results", Produced.with(Serdes.String(), Serdes.Long()));
    }
}
```

### 실험 2: Changelog 토픽과 상태 저장소 확인

```bash
# 자동 생성된 내부 토픽 확인
kafka-topics --bootstrap-server localhost:9092 --list | grep "order-processor"
# order-processor-order-counts-changelog
# order-processor-order-counts-repartition (있을 경우)

# Changelog 토픽 상세 확인
kafka-topics --bootstrap-server localhost:9092 \
  --describe --topic order-processor-order-counts-changelog
# cleanup.policy=compact 확인

# Changelog 내용 확인 (상태 저장소 현재값)
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic order-processor-order-counts-changelog \
  --from-beginning \
  --property print.key=true \
  --max-messages 10
```

### 실험 3: Task 재배치 후 복구 시간 측정

```bash
# Kafka Streams 앱 인스턴스 1개 강제 종료
kill -9 $(pgrep -f "order-processor")

# 다른 인스턴스의 Changelog 복원 로그 확인
grep -E "Restoring|Restored|changelog" /var/log/streams.log

# 복구 중 처리 지연 확인 (Consumer Lag 모니터링)
watch -n 1 'kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group order-processor'
```

---

## 📊 성능/비용 비교

### num.standby.replicas 설정별 복구 시간

```
상태 저장소 크기 1 GB 가정:

  num.standby.replicas=0 (기본값):
    Task 재배치 후 복구: Changelog 1 GB 재생
    복구 속도: ~100 MB/s → ~10초
    처리 공백: 10초 (Lag 쌓임)
    추가 비용: 없음

  num.standby.replicas=1:
    Task 재배치 후 복구: 최근 몇 초치 gap만 보충
    복구 시간: 수 초
    처리 공백: 거의 없음
    추가 비용: 상태 저장소 크기만큼 디스크 × 인스턴스 수

  상태 저장소 100 GB:
    Standby=0: 복구 ~17분 (심각한 다운타임)
    Standby=1: 복구 수 초 → 대규모 상태에서 Standby 필수

commit.interval.ms 설정별 처리량 영향:
  1ms:   높은 내구성, 처리량 ~20% 감소
  100ms: 균형 (권장)
  1000ms: 높은 처리량, 장애 시 최대 1초치 재처리
```

---

## ⚖️ 트레이드오프

```
RocksDB 로컬 상태:
  ✅ 네트워크 없이 로컬 디스크 접근 → 낮은 지연
  ✅ 상태 크기 무제한 (디스크 용량 범위 내)
  ❌ Task 재배치 시 상태 복원 비용 (Changelog 재생)
  ❌ 디스크 관리 필요

Standby Replica:
  ✅ 장애 복구 시간 대폭 단축
  ❌ 추가 디스크 + Changelog 구독 CPU 비용
  ❌ 인스턴스당 더 많은 리소스 필요

state.dir 영구 디렉토리:
  ✅ 재시작 후 즉시 이전 상태 사용
  ❌ 디스크 공간 관리 필요
  ❌ 서버 이전 시 상태 이식 불가 (Changelog에서 재구성)
```

---

## 📌 핵심 정리

```
Kafka Streams 아키텍처 핵심:

1. Topology = 처리 그래프
   Source → Processor(s) → Sink 노드 체인

2. StreamTask = 파티션의 처리 단위
   파티션 수 = Task 수
   각 Task는 독립 RocksDB 인스턴스 보유

3. 내구성 보장: RocksDB + Changelog 토픽
   변경사항 → Changelog 토픽(compact) → 재배치 시 복원

4. Standby Replica:
   Changelog 실시간 구독 → 장애 시 즉시 승격
   대규모 상태 운영 시 필수 (num.standby.replicas >= 1)

5. 운영 핵심 설정:
   state.dir: 영구 디렉토리 (not /tmp)
   num.standby.replicas: 1 이상
   commit.interval.ms: 100 (균형)
   cleanup.policy=compact: Changelog 토픽 유지
```

---

## 🤔 생각해볼 문제

**Q1. Kafka Streams Task 수를 늘리고 싶으면 어떻게 해야 하나요?**

<details>
<summary>해설 보기</summary>

입력 토픽의 파티션 수를 늘려야 합니다. Kafka Streams의 Task 수는 입력 토픽의 최대 파티션 수로 결정됩니다. 파티션 3개이면 최대 Task 3개 → 최대 병렬 처리 인스턴스 3개입니다.

파티션을 늘리면(`kafka-topics --alter --partitions`) Task 수가 자동으로 증가하고, Kafka Streams 인스턴스가 더 있다면 새 Task가 자동으로 배정됩니다.

단, 파티션 수 변경 시 키 해시 분포가 바뀌어 기존 상태 저장소의 데이터가 새 파티션에는 없을 수 있습니다. 상태가 있는 스트림 애플리케이션에서 파티션 수를 변경하면 상태 저장소를 리셋하거나 재처리가 필요할 수 있습니다.

</details>

---

**Q2. 동일한 Kafka Streams 애플리케이션 인스턴스를 10개 실행하는데 파티션이 3개라면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

파티션 3개이면 Task도 3개입니다. 10개 인스턴스 중 3개만 Task를 할당받고 처리합니다. 나머지 7개는 Standby Replica 역할을 하거나 완전히 IDLE 상태가 됩니다.

`num.standby.replicas=2`이면 각 Task마다 2개의 Standby가 있어서 최대 Task 3개 × (1 Active + 2 Standby) = 9개 인스턴스까지 활용됩니다. 10번째는 완전히 IDLE입니다.

인스턴스 수가 Task 수보다 많으면 추가 비용만 발생하고 처리량 향상은 없습니다. 최적: 인스턴스 수 = Task 수 + (Task 수 × num.standby.replicas).

</details>

---

**Q3. Changelog 토픽의 데이터가 쌓여서 복구 시간이 길어질 때 어떻게 해결하나요?**

<details>
<summary>해설 보기</summary>

Log Compaction이 주기적으로 실행되어 키별 최신값만 보존하지만, Compaction이 충분히 빠르게 실행되지 않으면 Changelog가 커질 수 있습니다. 해결 방법:

**Compaction 가속**: `min.cleanable.dirty.ratio`를 낮추거나 `log.cleaner.threads`를 늘려서 Compaction을 더 자주 실행합니다.

**Standby Replica 사용**: 복구 시 Changelog 전체를 재생하지 않고 Standby에서 gap만 보충합니다.

**상태 저장소 크기 제한**: 집계 키의 카디널리티가 너무 크면(수억 개의 고유 키) 상태가 무제한 증가합니다. Windowed 상태 저장소를 사용하면 시간이 지난 윈도우는 자동으로 삭제됩니다.

**RocksDB 튜닝**: `rocksdb.config.setter`로 RocksDB Compaction 설정을 최적화합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: KTable과 KStream ➡️](./02-ktable-kstream.md)**

</div>
