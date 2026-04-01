# ZooKeeper vs KRaft — 메타데이터 관리의 진화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ZooKeeper가 Kafka에서 관리하던 메타데이터는 정확히 무엇인가?
- ZooKeeper 의존성이 Kafka 운영에 어떤 복잡성을 만드는가?
- KRaft에서 Controller 쿼럼이 Raft 합의로 메타데이터를 관리하는 방식은?
- KRaft의 메타데이터 토픽(`__cluster_metadata`)은 ZooKeeper와 무엇이 다른가?
- ZooKeeper → KRaft 마이그레이션 시 고려해야 할 사항은?
- Kafka 3.x와 4.x에서 KRaft의 상태는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka 클러스터를 운영하다 보면 ZooKeeper와 Kafka 브로커를 함께 관리해야 한다. ZooKeeper가 다운되면 Kafka 브로커가 메타데이터에 접근하지 못해 새 파티션 Leader 선출도 불가능하다. 이것이 단일 장애점(SPOF)이 된다.

새로 구축하는 Kafka 클러스터라면 KRaft를 기본으로 선택해야 하는 시점이 되었다. Kafka 3.3부터 KRaft가 Production Ready로 선언되었고, Kafka 4.0부터는 ZooKeeper 모드를 완전히 제거했다.

기존 클러스터를 운영 중이라면 마이그레이션 계획을 이해해야 하고, 새 개발 환경을 구성한다면 KRaft로 시작하는 것이 맞다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: ZooKeeper 없이 Kafka가 동작한다고 생각 (ZooKeeper 모드)

  Kafka 브로커만 띄우고 ZooKeeper 없이 시작하려 함
  → 브로커가 시작 안 됨 (ZooKeeper 연결 필수)

  ZooKeeper가 관리하는 것들:
    - 브로커 등록/탈퇴 (ephemeral znode)
    - 파티션 Leader 정보
    - Controller 선출 (Kafka 컨트롤러 브로커)
    - 토픽 설정, ACL 정보
    - ISR 목록
  → 이 중 하나라도 ZooKeeper 없으면 불가

실수 2: ZooKeeper 클러스터와 Kafka 클러스터를 같은 서버에 배치

  소규모 구축:
    서버 3대: ZooKeeper 3개 + Kafka 3개 혼합 배치
  
  문제:
    Kafka 고부하 시 ZooKeeper의 CPU/메모리/디스크 I/O 경쟁
    ZooKeeper가 응답 지연 → Kafka 세션 만료 → 브로커 재연결 반복
    ZooKeeper 권장: 전용 서버에서 독립 운영

실수 3: KRaft가 ZooKeeper를 단순 대체한다는 오해

  "KRaft는 ZooKeeper 대신 Kafka 내부의 ZooKeeper다"
  실제:
    ZooKeeper와 달리 Raft 합의 알고리즘 기반
    Controller 쿼럼이 메타데이터를 내부 토픽(__cluster_metadata)에 저장
    별도 프로세스(ZooKeeper)가 아닌 Kafka 브로커 내부에 통합
    일부 Kafka 브로커가 Controller 역할을 겸임 (Dedicated Controller도 가능)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
새로 구축하는 환경:
  Kafka 3.3+ → KRaft 모드로 시작
  Kafka 4.0+  → ZooKeeper 모드 제거됨, KRaft만 가능

  KRaft 클러스터 구성:
    소규모 (개발/스테이징):
      3개 노드 = Controller + Broker 역할 겸임
      
    중/대규모 (운영):
      Dedicated Controller 3개 + Broker N개 분리
      Controller 장애가 Broker 성능에 영향 없도록 분리

기존 ZooKeeper 모드 클러스터 마이그레이션 계획:
  1. Kafka 3.x(ZooKeeper 모드)로 업그레이드 (현재 버전에서 단계적으로)
  2. 마이그레이션 전 kafka-storage 도구로 클러스터 ID 생성
  3. kafka-migration-tool로 메타데이터를 KRaft로 마이그레이션
  4. ZooKeeper 연결 제거, KRaft 전용 모드로 전환
  
  주의: 다운타임 없이 마이그레이션 가능하지만
        롤백은 불가능 (KRaft → ZooKeeper 복귀 불가)
        반드시 스테이징에서 검증 후 운영 적용
```

---

## 🔬 내부 동작 원리

### 1. ZooKeeper 모드: 메타데이터 관리 구조

```
ZooKeeper + Kafka 클러스터:

  ZooKeeper 앙상블 (3대)          Kafka 브로커 (3대)
  ┌──────────┐                   ┌─────────────────┐
  │ ZK-1     │                   │ Broker 1        │
  │ Leader   │◄─────────────────►│ (Controller)    │
  └──────────┘    메타데이터 R/W    └─────────────────┘
  ┌──────────┐                   ┌─────────────────┐
  │ ZK-2     │◄─────────────────►│ Broker 2        │
  └──────────┘                   └─────────────────┘
  ┌──────────┐                   ┌─────────────────┐
  │ ZK-3     │◄─────────────────►│ Broker 3        │
  └──────────┘                   └─────────────────┘

ZooKeeper znodes 예시:
  /kafka/brokers/ids/1       ← Broker 1 등록 (ephemeral)
  /kafka/brokers/ids/2       ← Broker 2 등록 (ephemeral)
  /kafka/brokers/topics/orders/partitions/0/state
    → {"leader":1,"isr":[1,2,3]}
  /kafka/controller
    → {"brokerid":1}         ← 현재 Controller 브로커 ID

Kafka Controller 선출:
  브로커 시작 시 /kafka/controller에 자신의 ID를 쓰려 시도
  ZooKeeper에서 선착순 1개만 성공
  성공한 브로커 = Controller
  Controller 브로커 다운 → ephemeral znode 삭제 → 나머지 브로커 경쟁 → 재선출

ZooKeeper 모드의 문제점:
  - 모든 메타데이터 변경이 ZooKeeper를 통함 → 병목 가능성
  - 파티션 수 증가 시 ZooKeeper 부하 선형 증가
  - Controller 브로커 재시작 시 ZooKeeper에서 전체 메타데이터 재로드 → 지연
  - 운영 복잡성: ZooKeeper 버전 관리, 별도 모니터링, 별도 설정
```

### 2. KRaft 모드: Raft 합의 기반 메타데이터

```
KRaft 클러스터 구성 (Controller + Broker 겸임):

  ┌──────────────────────────────────────────────────────────┐
  │                    Kafka 브로커 3대                        │
  │                                                          │
  │  Node 1: Controller(Active/Raft Leader) + Broker         │
  │  Node 2: Controller(Follower) + Broker                   │
  │  Node 3: Controller(Follower) + Broker                   │
  │                                                          │
  │  Controller 쿼럼은 Raft 프로토콜로 합의                        │
  └──────────────────────────────────────────────────────────┘

__cluster_metadata 토픽:
  - KRaft가 메타데이터를 저장하는 내부 토픽
  - 파티션 1개, 복제 팩터 = Controller 수
  - 일반 Kafka 로그 형식으로 저장 (세그먼트 파일)
  - 모든 메타데이터 변경 = 이 토픽에 레코드 append

  레코드 타입 예시:
    TopicRecord:        토픽 생성
    PartitionRecord:    파티션 생성
    BrokerRecord:       브로커 등록
    LeaderChangeRecord: 리더 변경
    IsrChangeRecord:    ISR 변경

Raft 합의 과정:
  1. Active Controller가 메타데이터 변경 제안
  2. Follower Controller들에게 전파
  3. 과반수(quorum) 확인 → 메타데이터 커밋
  4. Broker들이 Active Controller에서 메타데이터 직접 Fetch
```

### 3. ZooKeeper vs KRaft: 메타데이터 전파 비교

```
ZooKeeper 모드 토픽 생성 흐름:
  Admin Client → Controller 브로커 → ZooKeeper (메타데이터 쓰기)
  ZooKeeper → Controller 브로커 (watch 알림)
  Controller → 각 Broker (LeaderAndIsr 요청)
  각 Broker → Controller 응답
  → 여러 번의 왕복, ZooKeeper 경유로 지연 발생

KRaft 모드 토픽 생성 흐름:
  Admin Client → Active Controller → Raft 합의 (Follower 확인)
  → __cluster_metadata에 레코드 append
  각 Broker → Active Controller에서 메타데이터 Fetch (pull)
  → ZooKeeper 경유 없음, 지연 감소

Controller 재시작 시간 비교:
  파티션 10,000개:
    ZooKeeper 모드:   ~30초 (ZooKeeper에서 전체 메타데이터 재로드)
    KRaft 모드:       ~2초  (스냅샷 + 증분 로드)

  파티션 100,000개:
    ZooKeeper 모드:   ~5분
    KRaft 모드:       ~10초

  파티션 1,000,000개:
    ZooKeeper 모드:   불안정 (권장 한계 초과)
    KRaft 모드:       ~1분 (스냅샷 로드 포함)
```

### 4. KRaft Controller 장애 복구

```
Active Controller(Raft Leader) 장애 시:

  정상 상태:
    Node1: Active Controller (Raft Leader)
    Node2: Follower Controller
    Node3: Follower Controller

  Node1 다운:
    Node2, Node3가 Election Timeout 감지
    Node2가 Raft 선거 시작 (Vote 요청 전송)
    Node3가 Vote 응답 → 과반수(2/3) 달성
    Node2가 새 Active Controller

  전환 시간:
    ZooKeeper 모드: 수십 초 (ZooKeeper session timeout + 재선출)
    KRaft 모드:     수 초 이내

  설정:
    controller.quorum.voters=1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
```

### 5. KRaft 스냅샷과 메타데이터 압축

```
문제: __cluster_metadata 토픽이 계속 커질 수 있음

해결: 주기적 스냅샷 생성
  현재 메타데이터 상태 전체를 스냅샷 파일로 저장
  스냅샷 이전 레코드는 삭제 가능
  새 Controller 시작 시: 스냅샷 로드 → 이후 레코드만 재생

설정:
  metadata.log.max.record.bytes.between.snapshots=20971520
  (기본 20 MB마다 스냅샷 생성)
```

---

## 💻 실전 실험

### 실험 1: KRaft 모드 단일 브로커 시작

```bash
# KRaft 설정 파일 생성
cat > /tmp/kraft-server.properties << 'PROPS'
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@localhost:9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
inter.broker.listener.name=PLAINTEXT
advertised.listeners=PLAINTEXT://localhost:9092
controller.listener.names=CONTROLLER
log.dirs=/tmp/kraft-logs
PROPS

# 클러스터 ID 생성 및 스토리지 포맷
CLUSTER_ID=$(kafka-storage random-uuid)
kafka-storage format -t $CLUSTER_ID -c /tmp/kraft-server.properties

# 브로커 시작 (백그라운드)
kafka-server-start /tmp/kraft-server.properties &

# Controller 쿼럼 상태 확인
kafka-metadata-quorum --bootstrap-server localhost:9092 describe --status
# 출력:
# ClusterId:         xxxx-yyyy
# LeaderId:          1
# LeaderEpoch:       1
# HighWatermark:     5
# MaxFollowerLag:    0
# CurrentVoters:     [1]
# CurrentObservers:  []
```

### 실험 2: __cluster_metadata 내용 확인

```bash
# kafka-metadata-shell로 메타데이터 탐색
kafka-metadata-shell \
  --snapshot /tmp/kraft-logs/__cluster_metadata-0/00000000000000000000.checkpoint

# 대화형 셸에서:
# >> ls /topics
# orders  test-topic ...

# >> cat /topics/orders/0
# PartitionRegistration(id=0, replicas=[1], isr=[1], leader=1, ...)
```

### 실험 3: ZooKeeper 모드 메타데이터 확인 (비교)

```bash
# ZooKeeper CLI로 Kafka 메타데이터 직접 확인
docker exec zookeeper bash -c "
  /opt/bitnami/zookeeper/bin/zkCli.sh -server localhost:2181 get /kafka/controller
"
# 출력: {"version":1,"brokerid":1,"timestamp":"1700000000000"}

docker exec zookeeper bash -c "
  /opt/bitnami/zookeeper/bin/zkCli.sh -server localhost:2181 \
  get /kafka/brokers/topics/orders/partitions/0/state
"
# 출력: {"controller_epoch":1,"leader":1,"isr":[1,2,3],"leader_epoch":0}
```

---

## 📊 성능/비용 비교

### 운영 관리 비용 비교

```
ZooKeeper 모드:
  서버 수:         ZooKeeper 3대 + Kafka N대
  모니터링 항목:   ZooKeeper 세션 수, 지연 시간, GC / Kafka JMX
  보안 설정:       ZooKeeper SASL + Kafka SASL (이중)
  버전 관리:       ZooKeeper 버전 + Kafka 버전 호환성 확인

KRaft 모드:
  서버 수:         Kafka N대 (Controller 겸임 또는 전용)
  모니터링 항목:   Kafka JMX 통합
  보안 설정:       Kafka SASL 단일
  버전 관리:       Kafka 버전만 관리

Kafka 버전별 KRaft 지원:
  Kafka 2.8.0: Early Access (실험적)
  Kafka 3.3.0: Production Ready 선언
  Kafka 4.0.0: ZooKeeper 모드 완전 제거
```

---

## ⚖️ 트레이드오프

```
ZooKeeper 모드:
  ✅ 오랜 검증 기간과 방대한 운영 레퍼런스
  ❌ 별도 ZooKeeper 앙상블 운영 비용
  ❌ 대규모 파티션에서 확장성 한계
  ❌ Controller 재시작 지연
  ❌ Kafka 4.0부터 지원 중단

KRaft 모드:
  ✅ 단일 시스템 운영 (ZooKeeper 불필요)
  ✅ 대규모 파티션 확장성 대폭 향상
  ✅ 빠른 Controller 장애 복구
  ✅ 통합 보안 설정
  ✅ Kafka의 공식 미래 방향
  ❌ 상대적으로 짧은 운영 레퍼런스 (3.3+부터 Production Ready)
  ❌ KRaft → ZooKeeper 롤백 불가
```

---

## 📌 핵심 정리

```
ZooKeeper 모드:
  외부 ZooKeeper 앙상블이 Kafka 메타데이터 관리
  Controller 브로커가 ZooKeeper를 통해 메타데이터 조율
  ZooKeeper = 필수 외부 의존성

KRaft 모드:
  ZooKeeper 제거, Raft 합의로 Kafka 내부에서 메타데이터 관리
  __cluster_metadata 내부 토픽에 모든 변경 기록
  Controller 쿼럼(홀수) = Raft Leader 선출로 고가용성

실무 선택:
  신규 클러스터:        KRaft 선택 (Kafka 3.3+)
  기존 ZooKeeper 클러스터: Kafka 3.x에서 마이그레이션 도구로 전환
  Kafka 4.0+:         KRaft only (ZooKeeper 지원 없음)
  학습 환경:          KRaft 권장 (docker-compose 간소화)
```

---

## 🤔 생각해볼 문제

**Q1. KRaft에서 Controller 쿼럼을 3개 대신 5개로 구성하면 어떤 장단점이 있나요?**

<details>
<summary>해설 보기</summary>

**장점**: 내결함성이 높아집니다. 쿼럼 크기 N에서 (N-1)/2개 장애까지 허용합니다. 3개 쿼럼은 1개 장애 허용, 5개 쿼럼은 2개 장애 허용입니다.

**단점**: Raft 합의에 필요한 과반수가 커집니다. 3개 쿼럼에서는 2개 확인으로 커밋되지만, 5개 쿼럼에서는 3개 확인이 필요합니다. 더 많은 네트워크 왕복과 확인 대기가 필요해서 메타데이터 쓰기 지연이 증가합니다.

대부분의 실무에서는 3개 쿼럼이면 충분하고, 가용성이 매우 중요한 대규모 클러스터에서만 5개를 고려합니다.

</details>

---

**Q2. ZooKeeper에서 브로커 등록에 ephemeral znode를 사용하는 이유는 무엇인가요?**

<details>
<summary>해설 보기</summary>

ZooKeeper의 ephemeral znode는 **클라이언트 세션이 유지되는 동안만 존재**하는 임시 노드입니다. 브로커가 ZooKeeper 세션을 잃으면(정상 종료 또는 장애) ephemeral znode가 자동으로 삭제됩니다.

이를 활용해 Kafka는 브로커 상태를 자동으로 감지합니다. 브로커 장애 → ZooKeeper 세션 종료 → `/kafka/brokers/ids/{brokerId}` znode 삭제 → watch를 설정한 Controller가 알림 수신 → Controller가 해당 브로커의 파티션 리더를 재선출합니다.

KRaft에서는 이 역할을 Raft 프로토콜의 heartbeat로 대체합니다.

</details>

---

**Q3. KRaft에서 Active Controller 장애 시 Kafka 클러스터의 읽기/쓰기 동작은 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

기존 파티션의 **읽기/쓰기는 계속 동작**합니다. Broker 노드들은 Controller와 독립적으로 동작하며, 이미 할당된 파티션에서 메시지를 받고 전달하는 것은 Controller 없이도 가능합니다.

다만 **메타데이터 변경이 필요한 작업은 불가**합니다. 새 토픽 생성, 파티션 추가, 파티션 Leader 재선출은 Active Controller가 없으면 처리되지 않습니다. Controller 장애 중에 Broker 장애도 발생하면 해당 파티션의 Leader 재선출이 지연됩니다.

KRaft의 Raft Election Timeout(기본 수초)이 지나면 새 Active Controller가 선출되고 정상 동작이 재개됩니다. ZooKeeper 모드의 수십 초 대비 훨씬 빠른 복구가 가능합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Consumer 내부 동작](./05-consumer-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — 파티션 복제 ➡️](../replication-durability/01-partition-replication.md)**

</div>
