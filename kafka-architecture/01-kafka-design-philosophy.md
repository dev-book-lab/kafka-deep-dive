# Kafka 설계 철학 — 왜 메시지 큐가 아닌 분산 로그인가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Kafka가 RabbitMQ 같은 전통적 메시지 큐와 근본적으로 다른 점은 무엇인가?
- 메시지 큐는 메시지를 소비하면 삭제하는데, Kafka는 왜 삭제하지 않는가?
- Push 방식(MQ)과 Pull 방식(Kafka)의 차이가 실제로 어떤 운영 문제를 만드는가?
- 여러 Consumer Group이 동일 토픽을 독립적으로 소비할 수 있는 원리는?
- "분산 커밋 로그(Distributed Commit Log)"가 정확히 무엇을 의미하는가?
- Kafka가 빠른 이유는 "메모리를 쓰기 때문"이 아니라 무엇 때문인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka를 "빠른 메시지 큐"로만 이해하면 설계 결정이 틀어진다.

MQ(RabbitMQ, ActiveMQ)는 **소비하면 사라지는 편지함**이다. Consumer가 메시지를 가져가면 삭제된다. 반면 Kafka는 **모든 변경 사항을 시간 순서대로 쌓아두는 로그**다. Consumer는 로그에서 자신의 위치(offset)만 기억하고, 메시지는 보존 기간(retention)이 지날 때까지 디스크에 남는다.

이 차이를 모르면:
- "Kafka가 메시지를 왜 안 지우지? 디스크가 꽉 찰 것 같다" → 잘못된 설계
- 새로운 분석 시스템을 붙일 때 메시지를 "처음부터 다시" 읽어야 하는데 불가능하다고 생각함
- 장애 후 재처리를 위해 별도 메시지 저장소를 구축하는 낭비

Kafka의 설계 철학을 이해하면 "이 메시지는 30일간 보관하고, 세 개의 다른 시스템이 각자 독립적으로 소비한다"는 설계가 자연스럽게 나온다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Kafka를 RabbitMQ 대체재로만 취급

  설계: 주문 서비스 → Kafka → 결제 서비스
        메시지 하나당 소비자 하나, 소비 후 더 이상 필요 없다고 가정

  문제: 3개월 후 "주문 이력 분석" 시스템 추가 필요
        MQ 방식으로 생각했기 때문에:
        "이미 소비된 메시지를 어떻게 다시 읽지?" → 불가능하다고 판단
        → 별도 DB에 메시지 복사본 저장하는 우회 구조 추가
        → 사실 Kafka에 retention 기간 안에서 offset을 처음으로 리셋하면 가능했음

실수 2: Kafka의 처리량이 높은 이유를 "메모리 사용" 때문이라고 오해

  결과: "Kafka 서버에 RAM을 최대한 늘리면 된다"
        → JVM heap을 늘렸더니 오히려 GC 압박으로 느려짐
        실제 이유: 순차 디스크 I/O + OS 페이지 캐시 활용
                  Kafka는 오히려 JVM heap을 작게(6~8 GB) 유지하고
                  나머지 메모리를 OS 페이지 캐시에 양보하는 것이 최적

실수 3: Consumer가 느릴 때 MQ 스타일의 해결책 시도

  RabbitMQ에서 하던 방식:
    "Consumer가 못 따라오면 브로커가 전송 속도를 줄인다" (backpressure)
  Kafka에서는:
    Kafka는 Consumer 처리 속도에 상관없이 Producer가 계속 발행
    Consumer는 자신의 pace로 Fetch → offset으로 위치 추적
    → "Kafka가 backpressure를 지원 안 하네?" 가 아니라
       설계 철학이 다른 것
    해결책: Consumer 인스턴스 추가(파티션 수 범위 내)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
올바른 이해 1: Kafka는 분산 커밋 로그

  로그처럼 설계:
    모든 메시지는 파티션의 끝에 추가(append)만 됨
    Consumer는 읽을 위치(offset)를 직접 관리
    메시지는 retention 기간 동안 보존 (소비 여부와 무관)

  실무 적용:
    새로운 소비자가 생기면 → offset 0부터 시작해서 전체 이력 재처리 가능
    장애 후 재처리 → offset을 특정 시점으로 리셋
    분석 시스템, 실시간 처리, 감사 로그를 동일 토픽에서 각자 독립 소비

올바른 이해 2: 처리량의 비결은 순차 I/O

  최적 Kafka 서버 구성:
    JVM heap: 6~8 GB (작게)
    OS 페이지 캐시: 나머지 RAM 전체 활용
    디스크: 여러 개의 JBOD(Just a Bunch Of Disks) 또는 SSD

  Kafka는 쓸 때 OS 페이지 캐시에 쓰고 디스크로 flush는 OS에 맡김
  읽을 때도 페이지 캐시에서 바로 Consumer로 zero-copy(sendfile())
  → 디스크 읽기/쓰기가 거의 메모리 속도로 동작

올바른 이해 3: Pull 방식의 장점 활용

  Consumer가 자신의 처리 능력에 맞게 Fetch:
    max.poll.records=500   → 한 번에 최대 500개
    fetch.min.bytes=1024   → 최소 1 KB가 쌓이면 Fetch
    fetch.max.wait.ms=500  → 최대 500ms 대기 후 Fetch
  → Consumer가 느려도 브로커에 영향 없음
  → Consumer를 늘려서 처리량 수평 확장 가능
```

---

## 🔬 내부 동작 원리

### 1. 전통적 MQ vs Kafka: 구조 비교

```
[ 전통적 메시지 큐 (RabbitMQ) ]

  Producer → Exchange → Queue → Consumer A
                              ↗
                       Queue → Consumer B

  특징:
  - 메시지는 Queue에서 Consumer로 Push
  - Consumer가 ACK 보내면 Queue에서 삭제
  - 여러 Consumer가 같은 Queue를 소비 → 메시지 분산 (경쟁 소비)
  - Consumer B가 원하면 별도 Queue에 같은 메시지 복사 (Exchange 바인딩)
  - Consumer가 느리면 Queue 메시지 증가 → 브로커 메모리 압박
  - 메시지 재처리: Queue에서 이미 삭제됨 → 별도 저장소 필요


[ Kafka ]

  Producer → Topic (Partition 0) → offset: 0,1,2,3,4,5,...
                 (Partition 1) → offset: 0,1,2,3,...
                 (Partition 2) → offset: 0,1,2,3,...

  Consumer Group A:              Consumer Group B:
    Consumer A1 → Partition 0      Consumer B1 → Partition 0,1
    Consumer A2 → Partition 1      Consumer B2 → Partition 2
    Consumer A3 → Partition 2
    각자의 offset 독립 관리           각자의 offset 독립 관리

  특징:
  - 메시지는 파티션 로그에 append (삭제 안 함)
  - Consumer가 Pull 방식으로 원하는 속도로 Fetch
  - 여러 Consumer Group → 동일 메시지를 각자 독립 소비 가능
  - Consumer가 느려도 브로커에 영향 없음 (Lag만 쌓임)
  - 메시지 재처리: offset 리셋으로 언제든 재처리 가능 (retention 기간 내)
```

### 2. 분산 커밋 로그(Distributed Commit Log)란

```
커밋 로그 = 변경 사항을 시간 순서대로, append-only로 기록하는 구조

  파티션 0의 로그:
  ┌────────────────────────────────────────────────────────────┐
  │ offset:0  │ offset:1  │ offset:2  │ offset:3  │ offset:4   │
  │ key=A     │ key=B     │ key=A     │ key=C     │ key=B      │
  │ value=100 │ value=200 │ value=150 │ value=300 │ value=250  │
  │ ts=10:00  │ ts=10:01  │ ts=10:02  │ ts=10:03  │ ts=10:04   │ 
  └────────────────────────────────────────────────────────────┘
               ↑ 쓰기 방향: 항상 끝에만 추가

  Consumer Group A의 현재 위치: offset=3 (offset 3까지 처리 완료)
  Consumer Group B의 현재 위치: offset=1 (아직 offset 1까지만 처리)
  → A와 B는 완전히 독립적. 서로 영향 없음

"분산"의 의미:
  - 파티션이 여러 브로커에 분산
  - 복제(replication)로 내구성 보장
  - 클라이언트도 분산(Producer 여러 개, Consumer Group 여러 개)
```

### 3. Zero-Copy로 Consumer에게 전달

```
일반적인 파일 → 네트워크 전송 경로:
  디스크 → 커널 버퍼 → 유저 버퍼(JVM) → 소켓 버퍼 → NIC
  (데이터 복사: 4회)

Kafka의 Zero-Copy (sendfile() 시스템 콜):
  디스크 → 커널 버퍼 → NIC (커널 내에서 직접)
  (데이터 복사: 2회, 유저 공간 복사 없음)

  Consumer가 Fetch 요청 → Kafka 브로커가 sendfile() 호출
  → OS 페이지 캐시에서 직접 소켓으로 → Consumer에게 전달
  → JVM 내에서 데이터를 읽고 다시 쓰는 과정 없음

이것이 Kafka가 디스크 기반임에도 높은 처리량을 내는 핵심 이유
```

### 4. Push vs Pull: 왜 Pull을 선택했는가

```
Push 방식 (RabbitMQ):
  브로커가 Consumer에게 메시지 전송
  ┌ 장점: Consumer가 메시지를 즉시 받음 (지연 최소화)
  └ 단점: Consumer가 처리 못하면? → 브로커가 속도 조절 필요
          Consumer마다 처리 능력이 다를 때 복잡한 backpressure 로직 필요
          Consumer가 다운되면? → 브로커가 재전송 책임

Pull 방식 (Kafka):
  Consumer가 브로커에서 메시지를 가져감
  ┌ 장점: Consumer가 자신의 처리 능력에 맞게 fetch
  │       Consumer가 느려도 브로커 부하 없음
  │       배치로 가져오면서 throughput 최적화 가능
  │       Consumer가 다운 → 재시작 후 last offset부터 재개
  └ 단점: 메시지가 없을 때도 polling → Long Polling으로 완화
          (fetch.max.wait.ms로 대기 시간 설정)
```

---

## 💻 실전 실험

### 실험 1: 동일 토픽을 두 Consumer Group이 독립 소비

```bash
# 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic orders --partitions 3 --replication-factor 1

# Consumer Group A 시작 (터미널 1)
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders \
  --group group-A \
  --from-beginning

# Consumer Group B 시작 (터미널 2)
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders \
  --group group-B \
  --from-beginning

# 메시지 발행 (터미널 3)
echo "order-1" | kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic orders
echo "order-2" | kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic orders

# 결과: group-A와 group-B 모두 동일하게 "order-1", "order-2" 수신
# → 메시지는 삭제되지 않고 두 그룹이 각자 독립적으로 소비
```

### 실험 2: offset 리셋으로 메시지 재처리

```bash
# 현재 Consumer Group 오프셋 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group group-A

# 출력 예시:
# GROUP   TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# group-A orders  0          5               5               0
# group-A orders  1          3               3               0
# group-A orders  2          4               4               0

# group-A의 offset을 처음으로 리셋 (재처리)
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --group group-A \
  --topic orders \
  --reset-offsets --to-earliest --execute

# 다시 소비 시작하면 처음 메시지부터 재처리
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders \
  --group group-A
# → group-B에는 영향 없음 (독립적 offset 관리)
```

### 실험 3: Kafka의 retention 설정 확인

```bash
# 토픽의 retention 설정 확인
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics \
  --entity-name orders \
  --describe

# retention 변경 (1시간으로 단축)
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics \
  --entity-name orders \
  --alter \
  --add-config retention.ms=3600000

# 브로커 기본 retention 확인 (default: 7일)
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type brokers \
  --entity-name 1 \
  --describe | grep retention
```

---

## 📊 성능/비용 비교

### MQ vs Kafka 처리량 특성 비교

```
처리량 (단일 노드 기준, 1 KB 메시지):

  RabbitMQ (push, AMQP):
    쓰기:   ~50,000 msg/sec
    읽기:   ~50,000 msg/sec
    특이사항: 메시지 수가 증가할수록 메모리 → 디스크 paging 발생

  Kafka (pull, 순차 I/O):
    쓰기:   ~800,000 msg/sec (SSD 기준)
    읽기:   ~1,000,000+ msg/sec (페이지 캐시 히트 시)
    특이사항: 메시지가 쌓여도 성능 저하 없음 (디스크 순차 접근)

페이지 캐시 효과:
  Producer가 방금 쓴 메시지는 페이지 캐시에 있음
  Consumer가 바로 읽으면 → 디스크 접근 없이 캐시에서 직접 서빙
  → Kafka가 가장 빠른 케이스: 실시간 스트리밍 (생산 직후 소비)
```

### Consumer Group 수와 처리량 관계

```
동일 토픽을 소비하는 Consumer Group 수:
  Group 1개:   처리량 100%
  Group 3개:   처리량 100% (각 그룹 독립, 브로커는 3배 읽기 발생)
  Group 10개:  처리량 100% (단, 브로커 Fetch 부하 10배)

  → 페이지 캐시 덕분에 동일 데이터를 여러 Group이 읽어도
    디스크 I/O는 최초 1회만 발생
  → Consumer Group 수 증가 시 브로커 네트워크 대역폭이 먼저 병목
```

---

## ⚖️ 트레이드오프

```
장점:
  ✅ 동일 데이터를 여러 시스템이 독립적으로 소비 가능
  ✅ 장애 후 offset 리셋으로 재처리 (retention 기간 내)
  ✅ Consumer 처리 속도와 브로커 성능이 독립
  ✅ 순차 I/O + 페이지 캐시로 극한의 처리량
  ✅ 수평 확장 용이 (파티션과 Consumer 추가)

단점:
  ❌ 개별 메시지 라우팅(교환기 기반) 불가
     → RabbitMQ의 Exchange/Routing Key 같은 복잡한 라우팅은 Kafka에 없음
     → 토픽을 세분화하거나 Consumer에서 필터링해야 함

  ❌ 메시지 우선순위 큐 미지원
     → 우선순위 높은 메시지를 먼저 처리하는 기능 없음
     → 별도 토픽으로 분리하고 Consumer가 우선 순위 토픽을 먼저 poll하는 방식으로 구현

  ❌ 디스크 스토리지 비용
     → 모든 메시지를 retention 기간 동안 보관
     → Log Compaction으로 키별 최신 값만 유지하는 방식으로 완화 가능

  ❌ 운영 복잡성
     → ZooKeeper(또는 KRaft), 브로커, Schema Registry 등 컴포넌트 다수
     → RabbitMQ 대비 초기 설정과 운영 난이도가 높음
```

---

## 📌 핵심 정리

```
Kafka의 핵심 설계 결정:

1. Distributed Commit Log
   모든 메시지는 파티션에 append-only로 기록
   Consumer는 offset으로 위치를 추적하고, 메시지는 retention 동안 보존
   → 여러 시스템이 동일 데이터를 독립적으로 소비 가능

2. Pull 방식 소비
   Consumer가 자신의 페이스로 Fetch
   → 브로커에 backpressure 없이 Consumer 속도 독립성 보장

3. 순차 I/O + Zero-Copy
   디스크에 순차적으로 쓰고(append)
   OS 페이지 캐시 → sendfile()로 Consumer에게 zero-copy 전달
   → RAM이 아닌 디스크 기반임에도 높은 처리량

4. Consumer Group 독립성
   같은 토픽을 다른 Group이 소비해도 서로 영향 없음
   → "이 데이터를 여러 팀이 각자 다른 방식으로 활용"하는 이벤트 스트리밍 플랫폼

MQ로 써야 할 때:
  - 복잡한 라우팅 규칙 (Exchange/Binding)
  - 메시지 우선순위
  - 소규모, 단순 비동기 처리

Kafka로 써야 할 때:
  - 높은 처리량 (수십만 msg/sec)
  - 여러 소비자가 동일 이벤트를 각자 활용
  - 장애 후 재처리 / 이력 재분석
  - 이벤트 소싱, CQRS 등 이벤트 기반 아키텍처
```

---

## 🤔 생각해볼 문제

**Q1. Kafka는 Consumer가 메시지를 소비해도 삭제하지 않습니다. 그렇다면 디스크가 무한정 늘어나지 않을까요?**

<details>
<summary>해설 보기</summary>

Kafka는 두 가지 방식으로 디스크 사용량을 제어합니다.

**시간 기반 보존 (`retention.ms`)**: 기본값 7일. 설정 기간이 지난 메시지는 세그먼트 단위로 삭제됩니다. 세그먼트 롤링(`log.segment.bytes` 또는 `log.roll.ms`)이 완료된 오래된 세그먼트가 삭제 대상이 됩니다.

**크기 기반 보존 (`retention.bytes`)**: 파티션당 최대 크기를 지정합니다. 초과 시 오래된 세그먼트부터 삭제합니다.

**Log Compaction**: `cleanup.policy=compact`로 설정하면 같은 키의 오래된 값은 삭제하고 최신 값만 보존합니다. `__consumer_offsets` 토픽이 이 방식을 사용합니다.

실무에서는 `retention.ms=604800000`(7일)으로 두고, 처리량에 따라 `retention.bytes`를 추가로 제한하는 방식을 많이 씁니다.

</details>

---

**Q2. Consumer Group이 10개라면 브로커는 동일한 메시지를 10번 읽어서 10개 그룹에 보내야 할까요? 이것이 성능 병목이 되지 않을까요?**

<details>
<summary>해설 보기</summary>

Kafka가 OS 페이지 캐시를 활용하기 때문에 실제로는 디스크 I/O가 10배가 되지 않습니다.

Producer가 최근에 쓴 메시지는 OS 페이지 캐시에 존재합니다. 10개 Consumer Group이 Fetch 요청을 보내면, 브로커는 디스크에서 읽는 것이 아니라 페이지 캐시에서 바로 각 Consumer에게 전달합니다. 즉 디스크 I/O는 최초 1회(Producer 쓰기)만 발생하고, 이후 Consumer 읽기는 모두 메모리(페이지 캐시) 접근입니다.

다만 병목이 되는 지점은 **브로커의 네트워크 대역폭**입니다. 10개 그룹이 동시에 Fetch하면 아웃바운드 트래픽이 10배가 됩니다. Consumer Group 수가 늘어날 때는 브로커의 네트워크 대역폭을 먼저 모니터링해야 합니다.

</details>

---

**Q3. Kafka는 Pull 방식인데, Consumer가 메시지를 매우 빠르게 poll 하면 브로커에 불필요한 요청이 많아지지 않을까요? 어떻게 해결하나요?**

<details>
<summary>해설 보기</summary>

`fetch.min.bytes`와 `fetch.max.wait.ms`로 Long Polling을 구현합니다.

- `fetch.min.bytes=1024`: 최소 1 KB의 데이터가 쌓일 때까지 브로커가 응답을 보류
- `fetch.max.wait.ms=500`: 데이터가 없어도 최대 500ms 후에는 빈 응답을 반환

이 두 설정의 조합으로 Consumer는 데이터가 없을 때 빈 응답을 500ms마다 받는 방식으로 polling 합니다. 메시지가 활발히 들어올 때는 1 KB가 쌓이는 즉시 응답받으므로 지연이 낮습니다.

이것이 Kafka가 "Pull이지만 실시간처럼 동작"할 수 있는 이유입니다. 실시간성이 중요하면 `fetch.min.bytes=1`, `fetch.max.wait.ms=100`으로 줄이고, 처리량이 중요하면 `fetch.min.bytes=65536`, `fetch.max.wait.ms=500`으로 늘립니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 토픽과 파티션 ➡️](./02-topic-partition.md)**

</div>
