<div align="center">

# 🚀 Kafka Deep Dive

**"메시지를 발행하는 것과, Kafka가 어떻게 순서와 내구성을 보장하는지 아는 것은 다르다"**

<br/>

> *"`@KafkaListener` 붙이면 메시지를 받겠지 — 와 — 리밸런싱이 왜 발생하고 처리 중 리밸런싱이 데이터 중복을 일으키는지, `acks=all`이 실제로 무엇을 보장하는지 아는 것의 차이를 만드는 레포"*

파티션이 왜 병렬성의 기본 단위인지, 로그 세그먼트 `.log/.index/.timeindex` 파일이 순차 I/O로 빠른 이유, ISR이 이탈할 때 `min.insync.replicas`가 가용성과 내구성을 어떻게 트레이드오프하는지, Exactly-Once가 Producer ID + Sequence Number + Two-Phase Commit으로 어떻게 실제 구현되는지까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Kafka 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Kafka](https://img.shields.io/badge/Kafka-3.x-231F20?style=flat-square&logo=apachekafka&logoColor=white)](https://kafka.apache.org/documentation/)
[![Spring](https://img.shields.io/badge/Spring_Kafka-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-kafka/docs/current/reference/html/)
[![Docs](https://img.shields.io/badge/Docs-37개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Kafka에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`@KafkaListener`를 붙이면 메시지를 받습니다" | `ContainerFactory` → `KafkaMessageListenerContainer` → Fetch 루프 → Offset 커밋 전 과정, `max.poll.records`가 리밸런싱 위험에 미치는 영향 |
| "`acks=all`로 설정하면 안전합니다" | `acks=all`이어도 ISR이 1개뿐이면 단일 브로커 장애로 데이터 유실이 발생하는 이유, `min.insync.replicas`와의 조합이 왜 필수인가 |
| "파티션을 늘리면 처리량이 증가합니다" | 파티션 수가 Consumer 스레드 수와 어떻게 연결되는지, Leader 선출 비용과 Fetch 병렬성 트레이드오프 |
| "Exactly-Once를 사용하세요" | `enable.idempotence=true`의 Producer ID + Sequence Number 중복 제거 원리, `transactional.id`로 멀티 파티션 Two-Phase Commit이 동작하는 방식 |
| "리밸런싱이 발생하면 잠깐 멈춥니다" | Eager 리밸런싱의 Stop-The-World 구간, `max.poll.interval.ms` 초과로 그룹에서 제거될 때 미커밋 오프셋이 재처리되는 이유, Cooperative 리밸런싱이 이를 개선하는 방식 |
| "Consumer Lag을 모니터링하세요" | Lag이 쌓이는 원인 트리(파티션 수 부족 / 처리 로직 병목 / 리밸런싱 반복 / DB 왕복), `kafka-consumer-groups`로 진단하는 방법 |
| 이론 나열 | 실행 가능한 `kafka-console-producer/consumer`, `kafka-topics`, `kafka-consumer-groups`, `kafka-log-dirs` CLI 실험 + 3-브로커 Docker Compose 환경 + Spring 연결 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Architecture](https://img.shields.io/badge/🔹_Architecture-Kafka_설계_철학-231F20?style=for-the-badge&logo=apachekafka&logoColor=white)](./kafka-architecture/01-kafka-design-philosophy.md)
[![Replication](https://img.shields.io/badge/🔹_Replication-파티션_복제_구조-231F20?style=for-the-badge&logo=apachekafka&logoColor=white)](./replication-durability/01-partition-replication.md)
[![Delivery](https://img.shields.io/badge/🔹_Delivery-전달_보장_3단계-231F20?style=for-the-badge&logo=apachekafka&logoColor=white)](./delivery-guarantee/01-delivery-semantics.md)
[![Consumer](https://img.shields.io/badge/🔹_Consumer-Consumer_Group_내부-231F20?style=for-the-badge&logo=apachekafka&logoColor=white)](./consumer-rebalancing/01-consumer-group-internals.md)
[![Performance](https://img.shields.io/badge/🔹_Performance-Producer_처리량_최적화-231F20?style=for-the-badge&logo=apachekafka&logoColor=white)](./performance-monitoring/01-producer-throughput.md)
[![Streams](https://img.shields.io/badge/🔹_Streams-Kafka_Streams_아키텍처-231F20?style=for-the-badge&logo=apachekafka&logoColor=white)](./kafka-streams-event/01-kafka-streams-architecture.md)
[![Spring](https://img.shields.io/badge/🔹_Spring-Spring_Kafka_기초-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./spring-kafka/01-spring-kafka-basics.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Kafka 핵심 아키텍처

> **핵심 질문:** Kafka는 왜 메시지 큐가 아닌 분산 로그인가? 파티션이 왜 병렬성의 기본 단위이고, 브로커는 어떻게 순차 I/O로 고처리량을 달성하며, Producer와 Consumer는 내부에서 어떻게 동작하는가?

<details>
<summary><b>Kafka 설계 철학부터 KRaft 아키텍처까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Kafka 설계 철학 — 왜 메시지 큐가 아닌 분산 로그인가](./kafka-architecture/01-kafka-design-philosophy.md) | RabbitMQ 등 전통적 MQ와 Kafka의 근본적 차이(Push vs Pull, 메시지 삭제 시점, Consumer 독립성), 분산 커밋 로그(Distributed Commit Log)로 설계된 이유, 토픽을 여러 Consumer Group이 독립적으로 소비할 수 있는 원리 |
| [02. 토픽과 파티션 — 병렬성의 기본 단위](./kafka-architecture/02-topic-partition.md) | 파티션이 병렬 처리의 단위인 이유(Consumer 1개 = 파티션 1개 할당), 파티션 수가 처리량과 지연에 미치는 영향(Leader 선출 비용, Fetch 병렬성), 파티션 키로 메시지 순서를 보장하는 방식과 한계, 파티션 수 선택 기준 |
| [03. 브로커 내부 구조 — 로그 세그먼트와 순차 I/O](./kafka-architecture/03-broker-log-segment.md) | `.log` / `.index` / `.timeindex` 파일 구조와 각각의 역할, 순차 I/O(Sequential Write)가 랜덤 I/O보다 빠른 이유(디스크 헤드 이동 최소화), 페이지 캐시(Page Cache)를 적극 활용하는 설계의 의도, `kafka-log-dirs`로 세그먼트 실제 확인 |
| [04. Producer 내부 동작 — RecordAccumulator와 배치 전략](./kafka-architecture/04-producer-internals.md) | `RecordAccumulator`가 메시지를 파티션별로 배치로 묶는 구조, `linger.ms`(대기 시간)와 `batch.size`(최대 배치 크기)의 상호작용, 파티션 선택 전략(Key 해시 / Round-Robin / Sticky Partitioner) 비교, `buffer.memory` 초과 시 블로킹 동작 |
| [05. Consumer 내부 동작 — Fetch 루프와 폴링 간격](./kafka-architecture/05-consumer-internals.md) | Kafka가 Push 대신 Pull(Fetch) 방식을 택한 이유(Consumer 속도 독립성), `max.poll.records`로 한 번에 가져오는 레코드 수 제어, `max.poll.interval.ms` 초과가 리밸런싱을 유발하는 메커니즘, `fetch.min.bytes` / `fetch.max.wait.ms`로 Fetch 효율 조정 |
| [06. ZooKeeper vs KRaft — 메타데이터 관리의 진화](./kafka-architecture/06-zookeeper-vs-kraft.md) | ZooKeeper가 Kafka 메타데이터(브로커 등록, 파티션 리더, 컨트롤러 선출)를 관리해온 방식, ZooKeeper 의존성의 운영 복잡성 문제, KRaft(Kafka Raft)에서 Controller 쿼럼이 메타데이터를 직접 관리하는 구조, 마이그레이션 고려사항 |

</details>

<br/>

### 🔹 Chapter 2: 복제와 내구성 보장

> **핵심 질문:** ISR은 어떤 조건에서 이탈하고, `acks=all`이어도 데이터가 유실될 수 있는 조건은 무엇이며, `min.insync.replicas`는 가용성과 내구성을 어떻게 트레이드오프하는가?

<details>
<summary><b>파티션 복제부터 Log Compaction까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 파티션 복제 — Leader/Follower 구조](./replication-durability/01-partition-replication.md) | Leader 파티션이 모든 읽기/쓰기를 처리하는 이유, Follower가 Leader를 Fetch 방식으로 따라가는 과정(Pull 기반 복제), High Watermark(HW)와 Log End Offset(LEO)의 차이, `kafka-topics --describe`로 복제 상태 확인 |
| [02. ISR(In-Sync Replicas) — 조건과 이탈/복귀](./replication-durability/02-isr-in-sync-replicas.md) | ISR 멤버 조건(`replica.lag.time.max.ms` 내에 Leader LEO 따라잡기), Follower가 느려질 때 ISR에서 이탈하는 과정, ISR 이탈이 `acks=all`의 동작에 미치는 영향, ISR 복귀 조건과 복귀 중 Fetch 동기화 과정 |
| [03. acks 설정 완전 분해 — 보장하는 것과 보장하지 않는 것](./replication-durability/03-acks-deep-dive.md) | `acks=0`(응답 안 기다림) / `acks=1`(Leader만) / `acks=all`(ISR 전체) 각각이 실제로 보장하는 범위, `acks=all`이어도 ISR=1이면 단일 브로커 장애로 유실 가능한 이유, 설정별 처리량과 내구성 트레이드오프 수치 비교 |
| [04. min.insync.replicas — 가용성과 내구성의 트레이드오프](./replication-durability/04-min-insync-replicas.md) | `min.insync.replicas`가 쓰기 요청을 거부하는 조건(`NotEnoughReplicasException`), `replication.factor=3, min.insync.replicas=2, acks=all` 황금 조합의 근거, 브로커 장애 시 시나리오별 가용성 vs 내구성 계산 |
| [05. Leader Election — 파티션 리더 선출과 Unclean Election](./replication-durability/05-leader-election.md) | 파티션 리더 브로커 장애 시 새 리더 선출 과정(Controller가 ISR에서 선택), Unclean Leader Election(`unclean.leader.election.enable=true`)이 데이터 유실을 감수하는 이유, Preferred Leader Election으로 리더 분산을 재조정하는 방법 |
| [06. Log Compaction — 키 기반 최신값 보존](./replication-durability/06-log-compaction.md) | Compaction 대상 세그먼트(`clean` 영역 vs `dirty` 영역) 선택 기준, 키별 최신 값만 남기는 Compaction 과정, `__consumer_offsets` 토픽이 Log Compaction으로 운영되는 이유, Consumer가 Compacted 토픽을 읽을 때의 동작 |

</details>

<br/>

### 🔹 Chapter 3: 전달 보장과 멱등성

> **핵심 질문:** At-Least-Once와 Exactly-Once는 실제로 어떻게 구현이 다른가? Producer 멱등성은 내부에서 어떻게 중복을 제거하고, Consumer Offset 커밋 타이밍이 왜 중복/유실을 결정하는가?

<details>
<summary><b>전달 보장 3단계부터 EOS 전 과정까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 전달 보장 3단계 — At-Most-Once / At-Least-Once / Exactly-Once](./delivery-guarantee/01-delivery-semantics.md) | 각 보장 단계의 구현 방법(At-Most-Once: 커밋 후 처리 / At-Least-Once: 처리 후 커밋 / Exactly-Once: 멱등성 + 트랜잭션), 실무에서 At-Least-Once가 기본인 이유, Exactly-Once가 필요한 시나리오와 비용 |
| [02. Producer 멱등성 — Producer ID와 Sequence Number](./delivery-guarantee/02-producer-idempotence.md) | `enable.idempotence=true` 활성화 시 브로커가 `(ProducerID, PartitionID, SequenceNumber)` 조합으로 중복 메시지를 감지하는 원리, Producer 재시도(`retries`)로 인한 중복이 멱등성으로 제거되는 방식, 멱등성이 보장하는 범위(단일 파티션, 단일 세션) |
| [03. 트랜잭션 Producer — 멀티 파티션 원자적 쓰기](./delivery-guarantee/03-transactional-producer.md) | `transactional.id`와 Transaction Coordinator의 역할, Two-Phase Commit으로 여러 파티션에 원자적으로 쓰는 과정(`beginTransaction` → `send` → `commitTransaction`), Epoch 번호로 좀비 Producer를 차단하는 메커니즘 |
| [04. Consumer Offset 관리 — 자동 커밋 vs 수동 커밋](./delivery-guarantee/04-consumer-offset-management.md) | `enable.auto.commit=true`의 자동 커밋이 중복/유실을 발생시키는 시나리오, `commitSync`(처리 완료 후 블로킹) vs `commitAsync`(비블로킹, 콜백)의 차이, `__consumer_offsets` 토픽에 오프셋이 저장되는 구조, 수동 커밋 패턴 구현 |
| [05. Exactly-Once Semantics 전 과정 — Read-Process-Write](./delivery-guarantee/05-exactly-once-semantics.md) | `isolation.level=read_committed`로 Consumer가 커밋된 메시지만 읽는 방식, Read-Process-Write 패턴에서 EOS 달성 전체 흐름, Kafka Streams의 `processing.guarantee=exactly_once_v2`와 일반 Producer/Consumer EOS의 차이 |

</details>

<br/>

### 🔹 Chapter 4: Consumer Group과 리밸런싱

> **핵심 질문:** 리밸런싱은 정확히 무엇이 방아쇠를 당기고, Stop-The-World가 발생하는 구간은 어디이며, 처리 중 리밸런싱이 왜 중복 메시지를 만드는가?

<details>
<summary><b>Consumer Group 상태 머신부터 Consumer Lag까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Consumer Group 내부 동작 — Group Coordinator와 상태 머신](./consumer-rebalancing/01-consumer-group-internals.md) | Group Coordinator 브로커가 Consumer Group의 멤버십과 오프셋을 관리하는 방식, Consumer Group 상태 머신(`Empty` → `PreparingRebalance` → `CompletingRebalance` → `Stable`), Group Leader Consumer가 파티션 할당을 결정하는 역할 분리 |
| [02. 리밸런싱 발생 조건 — heartbeat와 poll 간격](./consumer-rebalancing/02-rebalancing-trigger.md) | `heartbeat.interval.ms`(생존 신호)와 `session.timeout.ms`(장애 판단 임계값)의 차이, `max.poll.interval.ms` 초과(처리 지연)가 그룹 이탈을 유발하는 과정, Consumer 추가/제거 외에 리밸런싱을 유발하는 숨겨진 원인들 |
| [03. 리밸런싱 전략 비교 — Eager vs Cooperative](./consumer-rebalancing/03-rebalancing-strategy.md) | Eager(기존) 리밸런싱의 Stop-The-World: 모든 Consumer가 파티션을 반납하고 재할당받는 과정, Cooperative(Incremental) 리밸런싱이 이동이 필요한 파티션만 재할당하는 방식, `partition.assignment.strategy=CooperativeStickyAssignor` 적용 방법 |
| [04. 리밸런싱 중 중복 처리 — 멱등 처리 패턴](./consumer-rebalancing/04-rebalancing-duplicate.md) | 처리 완료 후 Offset 커밋 전 리밸런싱이 발생하면 다음 Consumer가 동일 메시지를 재처리하는 과정, `ConsumerRebalanceListener.onPartitionsRevoked()`에서 수동 커밋으로 중복을 최소화하는 패턴, DB 멱등 키(Idempotent Key)로 중복 처리를 무해하게 만드는 설계 |
| [05. Consumer Lag — 원인 분석과 처리 속도 향상](./consumer-rebalancing/05-consumer-lag.md) | Consumer Lag이 쌓이는 원인 트리(파티션 수 < Consumer 수 → 유휴 Consumer 발생 / 처리 로직 병목 / DB 왕복 / 반복 리밸런싱), `kafka-consumer-groups --describe`로 파티션별 Lag 진단, 병렬 처리와 배치 처리로 소비 속도 높이는 전략 |

</details>

<br/>

### 🔹 Chapter 5: 성능 튜닝과 모니터링

> **핵심 질문:** 처리량을 높이려면 Producer와 Consumer의 어느 설정을 건드려야 하고, 파티션 핫스팟은 어떻게 진단하며, 운영 중 Under-Replicated Partitions가 발생하면 어떻게 대응하는가?

<details>
<summary><b>Producer/Consumer 튜닝부터 운영 장애 패턴까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Producer 처리량 최적화 — 배치와 압축](./performance-monitoring/01-producer-throughput.md) | `batch.size`(배치 최대 크기)와 `linger.ms`(전송 대기 시간) 조합으로 처리량을 높이는 원리, `buffer.memory` 설정이 배압(Backpressure)에 미치는 영향, 압축 방식(`gzip` / `snappy` / `lz4` / `zstd`) 별 CPU 비용 vs 네트워크 절감 비교 |
| [02. Consumer 처리량 최적화 — Fetch 튜닝](./performance-monitoring/02-consumer-throughput.md) | `fetch.min.bytes`(최소 Fetch 크기)와 `fetch.max.wait.ms`(최대 대기 시간)로 소량 Fetch 왕복을 줄이는 방법, `max.partition.fetch.bytes`(파티션당 최대 Fetch)로 메모리 사용 제어, Consumer 병렬화 전략(스레드 풀 vs Consumer 인스턴스 확장) |
| [03. 파티션 핫스팟 — 원인과 Custom Partitioner](./performance-monitoring/03-partition-hotspot.md) | 키 기반 파티셔닝에서 특정 키에 메시지가 집중될 때 단일 파티션 브로커 CPU/디스크 병목이 발생하는 원인, `kafka-log-dirs`로 파티션별 크기 불균형 진단, Custom Partitioner로 핫스팟 키를 여러 파티션에 분산하는 구현 |
| [04. Kafka 모니터링 — JMX 메트릭과 Grafana 대시보드](./performance-monitoring/04-kafka-monitoring.md) | 핵심 JMX 메트릭(`UnderReplicatedPartitions`, `BytesInPerSec`, `RequestHandlerAvgIdlePercent`, `OfflinePartitionsCount`), `kafka-exporter` + Prometheus + Grafana로 대시보드 구성하는 방법, Consumer Lag 알림 설정 |
| [05. 운영 중 발생하는 문제 패턴 — 진단과 대응](./performance-monitoring/05-operational-patterns.md) | Under-Replicated Partitions 원인 분석 및 복구 절차, Consumer Group 무한 리밸런싱 진단(`kafka-consumer-groups --describe`의 State 확인), 오프셋 리셋 절차(`--reset-offsets --to-earliest/--to-datetime`)와 재처리 시 주의사항 |

</details>

<br/>

### 🔹 Chapter 6: Kafka Streams와 이벤트 기반 설계

> **핵심 질문:** Kafka Streams의 상태 저장소는 어떻게 장애에도 복구되고, KTable과 KStream은 개념적으로 어떻게 다르며, Outbox Pattern은 DB 트랜잭션과 Kafka 발행의 원자성 문제를 어떻게 해결하는가?

<details>
<summary><b>Kafka Streams 아키텍처부터 Outbox Pattern까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Kafka Streams 아키텍처 — Topology와 상태 저장소](./kafka-streams-event/01-kafka-streams-architecture.md) | `Topology`(처리 그래프)와 `StreamTask`(파티션별 처리 단위)의 관계, RocksDB 기반 로컬 상태 저장소가 Changelog 토픽으로 내구성을 보장하는 원리, Task 재배치 시 RocksDB 상태를 Changelog에서 복원하는 과정 |
| [02. KTable과 KStream — 스냅샷 vs 이벤트 스트림](./kafka-streams-event/02-ktable-kstream.md) | KStream(이벤트의 무한 흐름)과 KTable(키별 최신 상태의 스냅샷) 개념 차이, KStream-KTable 조인이 동작하는 원리(이벤트 발생 시 현재 KTable 상태와 결합), GlobalKTable이 모든 파티션을 로컬에서 유지하는 방식과 용도 |
| [03. 윈도우 연산 — 시간 기반 집계와 지연 이벤트](./kafka-streams-event/03-window-operations.md) | Tumbling(겹치지 않는 고정 창) / Hopping(겹치는 고정 창) / Sliding(이벤트 간격 기반) / Session(비활동 간격 기반) Window 차이, Event Time vs Processing Time 기준 선택 이유, `grace period`로 늦게 도착하는 이벤트(Late Arrival)를 수용하는 방법 |
| [04. Exactly-Once in Kafka Streams — processing.guarantee](./kafka-streams-event/04-exactly-once-streams.md) | `processing.guarantee=exactly_once_v2`(EOS-V2)가 내부적으로 트랜잭션 Producer와 `read_committed` Consumer를 자동 구성하는 방식, EOS-V1 대비 EOS-V2가 브로커 트랜잭션 코디네이터 부하를 줄인 개선 내용, EOS 활성화 시 성능 비용 |
| [05. 이벤트 기반 아키텍처 패턴 — Outbox Pattern](./kafka-streams-event/05-event-driven-patterns.md) | DB 트랜잭션 커밋 후 Kafka 발행 사이에 장애가 발생할 때의 원자성 문제, Outbox Pattern(DB 트랜잭션 내 Outbox 테이블 기록 → Debezium CDC로 Kafka 발행)으로 원자성을 보장하는 구조, Saga Pattern과 Choreography vs Orchestration 비교 |

</details>

<br/>

### 🔹 Chapter 7: Spring과 Kafka 통합

> **핵심 질문:** `@KafkaListener`는 내부에서 어떻게 동작하고, Spring Kafka의 에러 처리는 어떻게 구성하며, `@Transactional`과 Kafka 트랜잭션은 어떻게 경계를 나누는가?

<details>
<summary><b>Spring Kafka 기초부터 Spring Batch + Kafka까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Spring Kafka 기초 — KafkaTemplate과 @KafkaListener 내부](./spring-kafka/01-spring-kafka-basics.md) | `KafkaTemplate.send()`가 `ProducerFactory` → `KafkaProducer`로 연결되는 과정, `@KafkaListener`가 `ConcurrentKafkaListenerContainerFactory`를 통해 `KafkaMessageListenerContainer`를 생성하는 방식, `ContainerProperties`로 Offset 커밋 모드(`BATCH`, `RECORD`, `MANUAL`) 설정 |
| [02. 에러 처리와 재시도 — DefaultErrorHandler와 DLT](./spring-kafka/02-error-handling-retry.md) | `DefaultErrorHandler`(Spring Kafka 2.8+)가 `BackOff` 정책으로 재시도하고 소진 시 Dead Letter Topic(DLT)으로 라우팅하는 구조, `SeekToCurrentErrorHandler`의 동작과 차이, DLT 메시지 재처리 전략(수동 재처리 vs 자동 재시도) |
| [03. 트랜잭션 처리 — @Transactional과 KafkaTransactionManager](./spring-kafka/03-transaction-processing.md) | DB 트랜잭션(`@Transactional`)과 Kafka 트랜잭션(`KafkaTransactionManager`)의 경계가 다른 이유, ChainedKafkaTransactionManager로 DB + Kafka 트랜잭션을 묶는 Best Effort 방식의 한계, 트랜잭션 Kafka Producer 설정과 `transactional.id` 관리 |
| [04. Spring Cloud Stream — 함수형 모델과 Kafka Binder](./spring-kafka/04-spring-cloud-stream.md) | Spring Cloud Stream의 함수형 프로그래밍 모델(`Function<Flux<Message>, Flux<Message>>`), Kafka Binder가 내부적으로 `KafkaMessageListenerContainer`와 `KafkaTemplate`을 구성하는 방식, 바인더 추상화로 RabbitMQ → Kafka 교체 시 코드 변경 범위 |
| [05. Spring Batch + Kafka — Chunk 처리와 Exactly-Once](./spring-kafka/05-spring-batch-kafka.md) | `KafkaItemReader`가 Consumer Offset을 JobExecutionContext에 저장하는 방식(Kafka Offset 커밋과 Batch Checkpoint 동기화), `KafkaItemWriter`의 트랜잭션 처리, Chunk 단위 Exactly-Once 달성을 위한 `KafkaTransactionManager` + `JpaTransactionManager` 조합 전략 |

</details>

---

## 🧪 실험 환경

### Docker Compose 시작

```bash
# ⚠️ 이 docker-compose.yml은 ZooKeeper 모드입니다 (Kafka 3.x 기준)
# KRaft 아키텍처 학습(Ch6-06)은 문서 내 kafka-storage 실험을 참고하세요

# 3-브로커 클러스터 + Schema Registry + Kafka UI 시작
docker compose up -d

# 브로커 상태 확인 (컨테이너 내부 포트 9092 사용)
docker exec kafka-1 kafka-broker-api-versions --bootstrap-server localhost:9092

# Kafka UI 접속
open http://localhost:8080
```

### 핵심 CLI 명령어 세트

```bash
# 토픽 생성 (복제 팩터 3, 파티션 3)
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic orders \
  --replication-factor 3 \
  --partitions 3

# ISR 상태 확인 (Leader, Replicas, ISR 구분)
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic orders

# 메시지 발행 (Key 지정)
kafka-console-producer --bootstrap-server localhost:19092 \
  --topic orders \
  --property "key.separator=:" \
  --property "parse.key=true"

# 메시지 소비 (처음부터)
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders \
  --from-beginning \
  --property print.key=true \
  --group order-consumer-group

# Consumer Group Lag 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group order-consumer-group

# 오프셋 리셋 (재처리)
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --group order-consumer-group \
  --topic orders \
  --reset-offsets --to-earliest --execute

# 로그 세그먼트 파일 상세 확인
kafka-log-dirs --bootstrap-server localhost:19092 \
  --topic-list orders \
  --describe

# 브로커별 파티션 리더 분포 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic orders | grep Leader
```

### Spring Kafka 핵심 설정 레퍼런스

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:19092,localhost:29092,localhost:39092
    producer:
      acks: all                        # ISR 전체 확인
      batch-size: 16384                # 16 KB (처리량 우선이면 65536)
      linger-ms: 5                     # 5ms 대기 (처리량 우선이면 20~100)
      buffer-memory: 33554432          # 32 MB
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      properties:
        enable.idempotence: true       # 멱등성 활성화 → retries=MAX, acks=all 자동 설정
        max.in.flight.requests.per.connection: 5
    consumer:
      group-id: my-consumer-group
      auto-offset-reset: earliest
      enable-auto-commit: false        # 수동 커밋
      max-poll-records: 500
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      properties:
        max.poll.interval.ms: 300000   # 5분 처리 허용
        session.timeout.ms: 10000
        heartbeat.interval.ms: 3000
        isolation.level: read_committed # EOS 소비
    listener:
      ack-mode: MANUAL_IMMEDIATE       # 수동 커밋 모드
      concurrency: 3                   # Consumer 스레드 수 (≤ 파티션 수)
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — Kafka를 블랙박스로 두는 접근과 그 결과 |
| ✨ **올바른 접근** | After — 원리를 알고 난 후의 설계/운영 |
| 🔬 **내부 동작 원리** | 브로커/클라이언트 내부 분석 + ASCII 구조도 |
| 💻 **실전 실험** | `kafka-console-producer/consumer`, `kafka-topics`, `kafka-consumer-groups`, `kafka-log-dirs` |
| 📊 **성능/비용 비교** | acks 설정별 처리량, 압축 방식별 비교, 리밸런싱 전략별 다운타임 등 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "acks=all과 min.insync.replicas의 차이를 모른다" — 내구성 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch2-02  ISR 조건과 이탈/복귀 → ISR이 무엇인지 이해
       Ch2-03  acks 완전 분해 → acks=all이 보장하는 것과 보장하지 않는 것
Day 2  Ch2-04  min.insync.replicas → 가용성 vs 내구성 트레이드오프
       Ch2-05  Leader Election → Unclean Election 데이터 유실 시나리오
Day 3  Ch3-01  전달 보장 3단계 → At-Least-Once가 기본인 이유
       Ch3-02  Producer 멱등성 → 중복 제거 원리
```

</details>

<details>
<summary><b>🟡 "리밸런싱이 왜 발생하는지, 중복 처리를 어떻게 막는지 모른다" — Consumer 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-05  Consumer 내부 동작 → Fetch 루프, max.poll 설정
Day 2  Ch4-01  Consumer Group 상태 머신 → Group Coordinator 역할
       Ch4-02  리밸런싱 발생 조건 → heartbeat vs poll 간격
Day 3  Ch4-03  리밸런싱 전략 → Eager Stop-The-World vs Cooperative
       Ch4-04  리밸런싱 중 중복 → 멱등 처리 패턴
Day 4  Ch3-01  전달 보장 3단계 → At-Least-Once 이해
       Ch3-04  Consumer Offset 관리 → 수동 커밋 패턴
Day 5  Ch3-05  EOS 전 과정 → Exactly-Once 달성 방법
Day 6  Ch4-05  Consumer Lag → 원인 진단 트리
Day 7  Ch7-01  Spring Kafka 기초 → @KafkaListener 내부 동작
       Ch7-02  에러 처리와 DLT → 실패 메시지 처리 전략
```

</details>

<details>
<summary><b>🔴 "Kafka 내부를 브로커 파일 수준까지 완전히 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — Kafka 핵심 아키텍처
        → docker exec kafka-1로 .log/.index/.timeindex 파일 직접 확인
        → kafka-dump-log로 세그먼트 파일 내용 분석

2주차  Chapter 2 전체 — 복제와 내구성 보장
        → kafka-3 컨테이너 강제 종료 → ISR 이탈 관찰
        → acks 설정별 처리량 측정 실험

3주차  Chapter 3 전체 — 전달 보장과 멱등성
        → 네트워크 지연 주입으로 중복 메시지 재현
        → 트랜잭션 Producer로 멀티 파티션 원자적 쓰기 실험

4주차  Chapter 4 전체 — Consumer Group과 리밸런싱
        → max.poll.interval.ms 초과로 리밸런싱 의도적 유발
        → Cooperative vs Eager 리밸런싱 다운타임 측정 비교

5주차  Chapter 5 전체 — 성능 튜닝과 모니터링
        → Grafana 대시보드 구성 → Consumer Lag 알림 설정
        → 핫스팟 파티션 재현 → Custom Partitioner 구현

6주차  Chapter 6 전체 — Kafka Streams와 이벤트 기반 설계
        → Topology 직접 구성 → RocksDB 상태 저장소 장애 복구 실험
        → Outbox Pattern + Debezium CDC 연동

7주차  Chapter 7 전체 — Spring과 Kafka 통합
        → DLT 라우팅 흐름 디버깅 → KafkaTransactionManager 설정
        → Spring Batch + Kafka Exactly-Once 검증
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-batch-deep-dive](https://github.com/dev-book-lab/spring-batch-deep-dive) | Chunk 처리, Step 설계, ItemReader/Writer | Ch7-05(Kafka ItemReader/Writer + Exactly-Once) |
| [spring-cloud-deep-dive](https://github.com/dev-book-lab/spring-cloud-deep-dive) | 이벤트 기반 마이크로서비스, Spring Cloud Stream | Ch6-05(Outbox Pattern), Ch7-04(Spring Cloud Stream Binder) |
| [redis-deep-dive](https://github.com/dev-book-lab/redis-deep-dive) | Redis Stream, Pub/Sub vs Stream 비교 | Ch1-01(분산 로그 vs 메시지 큐 — Redis Stream과 비교) |
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | @Transactional 내부, 트랜잭션 경계 | Ch7-03(DB 트랜잭션과 Kafka 트랜잭션 경계) |
| [mysql-deep-dive](https://github.com/dev-book-lab/mysql-deep-dive) | InnoDB 트랜잭션, CDC 기반 Outbox | Ch6-05(Outbox Pattern — DB 트랜잭션과 Kafka 발행 원자성) |

> 💡 이 레포는 **Kafka 내부 동작**에 집중합니다. Spring을 모르더라도 Chapter 1~6을 순수 Kafka 관점으로 학습할 수 있습니다. Chapter 7은 Spring Kafka 실무 경험이 있을 때 더 깊이 연결됩니다.

---

## 🙏 Reference

- [Kafka 공식 문서](https://kafka.apache.org/documentation/)
- [Kafka: The Definitive Guide, 2nd Edition — Neha Narkhede, Gwen Shapira, Todd Palino](https://www.confluent.io/resources/kafka-the-definitive-guide-v2/)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/) — Kafka 관련 챕터
- [Confluent Blog](https://www.confluent.io/blog/)
- [Kafka Improvement Proposals (KIPs)](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals)
- [Spring Kafka Reference Documentation](https://docs.spring.io/spring-kafka/docs/current/reference/html/)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"메시지를 발행하는 것과, Kafka가 어떻게 순서와 내구성을 보장하는지 아는 것은 다르다"*

</div>
