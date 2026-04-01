# Leader Election — 파티션 리더 선출과 Unclean Election

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 파티션 Leader 브로커가 장애났을 때 새 Leader는 어떤 과정으로 선출되는가?
- ISR에서 새 Leader를 선출하는 기준은 무엇인가?
- `unclean.leader.election.enable=true`가 데이터 유실을 일으키는 구체적 시나리오는?
- Preferred Leader란 무엇이고, `kafka-preferred-replica-election`은 무엇을 하는가?
- Controller 브로커가 Leader 선출에서 하는 역할은?
- Leader 선출 중 해당 파티션의 쓰기/읽기는 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Leader 선출은 브로커 장애 복구의 핵심이다. 장애 후 몇 초 안에 새 Leader가 선출되어야 서비스가 재개된다. 하지만 잘못된 설정(`unclean.leader.election.enable=true`)이면 빠른 복구 대신 데이터 유실을 선택하게 된다.

또한 브로커 재시작이 반복되면 Leader가 특정 브로커로 쏠리는 현상이 발생한다. 이를 방치하면 특정 브로커에 부하가 집중된다. Preferred Leader Election은 이를 재균형하는 도구다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: unclean.leader.election.enable=true 설정

  의도: "빠른 복구를 위해 ISR 밖의 브로커도 Leader 허용"
  결과:
    ISR=[1,2], Broker 1이 Leader, offset 0~100 기록
    Broker 2는 offset 0~80까지만 복제 완료
    Broker 1 장애 → ISR=[2] → Broker 2가 새 Leader
    
    만약 unclean=false: Broker 2만 후보 → 정상 (offset 0~80만 있지만)
    
    ISR=[2]도 없어지면:
    unclean=true: ISR 밖 Broker 3 (offset 0~70)도 Leader 후보
    Broker 3이 새 Leader가 되면?
    → Consumer는 offset 71~100 못 읽음 (데이터 유실)
    → 더 나쁜 경우: Broker 1 복구 시 offset 71~100을 가지고 Follower로 참여
      → Leader Epoch로 offset 71~100 제거 (Truncation)
      → 돌이킬 수 없는 데이터 유실

실수 2: Leader 쏠림 현상 방치

  상황: 브로커 롤링 재시작 후
        Broker 1 재시작 → Leader였던 파티션들이 Broker 2, 3으로 이동
        Broker 2 재시작 → 파티션들이 Broker 1, 3으로 이동
        Broker 3 재시작 → 파티션들이 Broker 1, 2로 이동
        
        결과: Broker 3이 대부분 파티션의 Leader → Broker 3 과부하
              Broker 1이 가장 적은 Leader → 유휴 상태
  
  해결: kafka-preferred-replica-election로 Leader를 Preferred Leader로 재배분

실수 3: Leader 선출 중 파티션 상태 오해

  가정: "Leader 선출이 시작되면 파티션이 즉시 새 Leader로 서빙"
  실제:
    Leader 선출 완료 전까지 해당 파티션은 Offline 상태
    Producer: LEADER_NOT_AVAILABLE 에러
    Consumer: UNKNOWN_LEADER_EPOCH 에러
    → 선출 완료(수 초 이내)까지 해당 파티션 일시적 불가용
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
안전한 설정:
  unclean.leader.election.enable=false  (기본값, 유지 권장)
  → ISR에서만 Leader 선출
  → ISR이 없으면 Offline (서비스 중단)이지만 데이터 유실 없음

  "데이터 유실 있는 빠른 복구 < 데이터 유실 없는 늦은 복구"
  → 결제, 주문 등 비즈니스 중요 데이터는 반드시 false

Preferred Leader 관리:
  # 모든 파티션의 Preferred Leader로 재선출
  kafka-preferred-replica-election \
    --bootstrap-server localhost:9092

  # 자동 Preferred Leader 선출 활성화
  # server.properties에 설정:
  auto.leader.rebalance.enable=true        (기본값 true)
  leader.imbalance.check.interval.seconds=300  (5분마다 체크)
  leader.imbalance.per.broker.percentage=10    (10% 이상 불균형 시 재조정)

장애 복구 우선순위:
  1. ISR에서 가장 최신 Follower를 Leader로 선출
  2. 선출 완료 후 새 Leader에서 서비스 재개
  3. 장애 브로커 복구 → Follower로 ISR 복귀
```

---

## 🔬 내부 동작 원리

### 1. Controller 브로커의 역할

```
Kafka 클러스터에는 항상 1개의 Controller 브로커가 존재
Controller 역할:
  - 파티션 Leader 선출 결정
  - ISR 변경 감지 및 처리
  - 브로커 장애 감지
  - 새 토픽/파티션 생성 조율

Controller 선출 (ZooKeeper 모드):
  /kafka/controller에 자신의 ID를 쓴 브로커 = Controller
  Controller 브로커 장애 → ephemeral znode 삭제 → 다른 브로커 경쟁

Controller 선출 (KRaft 모드):
  Raft 합의로 Active Controller 선출
  보다 빠른 선출 (수 초 이내)
```

### 2. 파티션 Leader 선출 흐름

```
Broker 1 (Partition 0 Leader) 장애 발생:

  1. ZooKeeper/KRaft가 Broker 1 세션 종료 감지
     (session.timeout.ms 또는 Raft heartbeat 기준)

  2. Controller 브로커가 Broker 1 장애 인지

  3. Controller가 Broker 1이 Leader인 파티션 목록 파악
     예: Partition 0, Partition 5, Partition 8 ...

  4. 각 파티션의 ISR 확인
     Partition 0 ISR: [2, 3] (Broker 1 제외)

  5. ISR 중 새 Leader 선택
     기본 전략: ISR 목록의 첫 번째 멤버
     Partition 0 → Broker 2가 새 Leader

  6. LeaderAndIsrRequest를 Broker 2에게 전송
     "Partition 0의 Leader가 됩니다"

  7. Broker 2가 Leader로 전환 완료

  8. MetadataUpdate를 다른 모든 브로커에게 전파
     "Partition 0의 Leader = Broker 2"

  9. Producer/Consumer 클라이언트가 새 Leader 정보 업데이트
     (메타데이터 갱신)

  전체 소요 시간:
    ZooKeeper 모드: ~10~30초 (session.timeout.ms 포함)
    KRaft 모드:     ~3~10초 (Raft heartbeat 기반)
```

### 3. Unclean Leader Election 상세 분석

```
설정: unclean.leader.election.enable=true (위험!)

시나리오:
  ISR=[1,2,3], Leader=1
  Broker 1 장애 → ISR=[2,3]
  Broker 2, 3도 장애 → ISR=[] (비어있음)

  unclean=false 동작:
    ISR 비어있음 → Leader 선출 불가
    파티션 Offline
    → 브로커 복구 후 ISR 복귀 시 서비스 재개
    데이터 유실 없음 (복구 후 전체 데이터 intact)

  unclean=true 동작:
    ISR 비어있음 → ISR 밖 Broker 4 (Partition 0 Follower, 많이 뒤처짐)가 후보
    Broker 4: offset 0~50까지만 복제 (Broker 1: 0~100)
    Broker 4가 새 Leader로 선출
    Broker 4 기준으로 새 메시지 offset 51부터 쌓임
    
    나중에 Broker 1 복구 → Follower로 참여
    Leader Epoch 불일치 → Broker 1의 offset 51~100 Truncation
    → offset 51~100 영구 유실!

  결론: unclean=true는 "가용성을 위해 데이터를 버린다"는 결정
        비즈니스 중요 데이터에는 절대 금지
```

### 4. Preferred Leader와 Leader 재균형

```
Preferred Leader = 토픽 생성 시 Replicas 목록의 첫 번째 브로커

  kafka-topics --describe:
  Partition 0: Replicas: 1,2,3  ← Preferred Leader = Broker 1
  Partition 1: Replicas: 2,3,1  ← Preferred Leader = Broker 2
  Partition 2: Replicas: 3,1,2  ← Preferred Leader = Broker 3

  이상적: Broker 1 → 파티션 3개씩 Leader
           파티션이 Preferred Leader에 고르게 분산

  브로커 재시작 후 Leader 쏠림:
  Broker 1 재시작 → Broker 1의 파티션이 Broker 2, 3으로 이동
  Broker 1 복구됐지만 Preferred Leader 복귀 안 함
  → Broker 1: Leader 0개, Broker 2: Leader 12개, Broker 3: Leader 12개

  Preferred Leader Election:
    Controller가 각 파티션의 Preferred Leader를 현재 Leader로 변경
    조건: Preferred Leader가 ISR에 있어야 선출 가능
    결과: 부하가 다시 균등 분산

  자동 설정:
    auto.leader.rebalance.enable=true (기본값)
    leader.imbalance.check.interval.seconds=300
    leader.imbalance.per.broker.percentage=10
    → 5분마다 Leader 불균형 체크, 10% 초과 시 자동 재조정
```

---

## 💻 실전 실험

### 실험 1: Leader 선출 과정 관찰

```bash
# 현재 Leader 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic orders
# Partition 0 Leader: 1

# Broker 1 중단
docker stop kafka-1

# 선출 진행 중 상태 모니터링 (빠르게 여러 번)
for i in 1 2 3 4 5; do
  sleep 2
  kafka-topics --bootstrap-server localhost:19092 \
    --describe --topic orders 2>&1 | grep "Partition: 0"
done
# 선출 전: LEADER_NOT_AVAILABLE 또는 연결 오류
# 선출 후: Leader가 2 또는 3으로 변경됨
```

### 실험 2: Preferred Leader Election

```bash
# 롤링 재시작 시뮬레이션
docker restart kafka-1
sleep 5
docker restart kafka-2
sleep 5
docker restart kafka-3
sleep 10

# Leader 분포 불균형 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic orders
# 특정 브로커에 Leader 집중 현상 관찰

# Preferred Leader로 재선출
kafka-preferred-replica-election \
  --bootstrap-server localhost:19092
sleep 5

# 재선출 후 균등 분산 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic orders
```

### 실험 3: unclean election 동작 확인 (데이터 유실 주의!)

```bash
# 테스트용 토픽 (실험 환경에서만!)
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic unclean-test --partitions 1 --replication-factor 3

# unclean election 활성화 (테스트용)
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics --entity-name unclean-test \
  --alter --add-config unclean.leader.election.enable=true

# 메시지 발행
for i in $(seq 1 50); do echo "msg-$i"; done | \
  kafka-console-producer --bootstrap-server localhost:19092 \
  --topic unclean-test

# 모든 브로커 중단 (ISR 비움)
docker stop kafka-1 kafka-2 kafka-3
sleep 5

# Broker 3만 재시작 (뒤처진 Follower)
docker start kafka-3
sleep 5

# ISR 밖 Broker 3이 Leader가 됨 (unclean=true이므로)
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic unclean-test
# Leader: 3 (ISR 밖이었던 브로커)

# 데이터 손실 확인
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic unclean-test --from-beginning --max-messages 50
# 50개보다 적을 수 있음 (유실 발생)
```

---

## 📊 성능/비용 비교

### Leader 선출 소요 시간 비교

```
파티션 1000개 기준, 브로커 1대 장애 시:

  ZooKeeper 모드:
    세션 타임아웃: zookeeper.session.timeout.ms (기본 6000ms)
    Controller 감지: ~6초
    리더 선출 처리: ~1~5초 (파티션 수에 비례)
    총 선출 시간: ~7~11초

  KRaft 모드:
    Raft 감지: ~1~3초
    리더 선출 처리: ~1~3초
    총 선출 시간: ~2~6초

파티션 100,000개 기준:
  ZooKeeper 모드: ~수분 (파티션 목록 재로드 포함)
  KRaft 모드:     ~30~60초
```

### unclean=true vs false 서비스 영향

```
ISR 완전 소멸 시나리오:

  unclean=false:
    파티션 Offline → 서비스 중단
    복구: 브로커 1대 복구 시 즉시 ISR 복귀 → 서비스 재개
    데이터: 완전 보존

  unclean=true:
    ISR 밖 브로커가 Leader → 서비스 빠른 재개
    복구 후: 더 최신 데이터를 가진 브로커 복귀 시 Truncation
    데이터: 유실 (복구 후에도 복원 불가)

결론: 대부분의 운영 환경에서 unclean=false가 올바른 선택
      "데이터 유실이 있는 서비스 가용 < 데이터 보존하는 서비스 중단"
```

---

## ⚖️ 트레이드오프

```
unclean.leader.election.enable=true:
  ✅ ISR 소멸 시에도 서비스 가용성 유지
  ❌ 데이터 유실 (복구 불가)
  ❌ 의도하지 않은 Truncation으로 시스템 불일관성
  → 센서 데이터, 완전 유실 허용 로그에만 고려

unclean.leader.election.enable=false (기본값):
  ✅ 데이터 유실 없음
  ❌ ISR 소멸 시 파티션 Offline → 서비스 중단
  → 비즈니스 중요 데이터 표준

auto.leader.rebalance.enable=true:
  ✅ 자동으로 Leader 부하 균등 분산
  ❌ Leader 선출 중 일시적 파티션 중단 (짧지만 발생)
  → 트래픽이 낮은 시간대에 실행되도록 스케줄 고려
```

---

## 📌 핵심 정리

```
Leader Election 핵심:

1. Controller 브로커가 Leader 선출 결정
   ZooKeeper/KRaft를 통해 브로커 장애 감지

2. ISR에서만 새 Leader 선출 (unclean=false 기본값)
   ISR이 비어있으면 파티션 Offline → 데이터 유실 없음

3. unclean.leader.election.enable=true = 데이터 유실 허용
   비즈니스 중요 데이터에는 절대 false 유지

4. Preferred Leader = Replicas 목록 첫 번째 브로커
   auto.leader.rebalance.enable=true로 자동 재균형
   kafka-preferred-replica-election으로 수동 재균형

5. 선출 중 파티션 일시 Offline
   KRaft 모드에서 선출 시간 단축 (수 초 이내)
```

---

## 🤔 생각해볼 문제

**Q1. 파티션의 ISR이 비어있어서 Offline 상태가 됐을 때, 운영자로서 어떤 선택을 해야 하나요?**

<details>
<summary>해설 보기</summary>

기본적으로 `unclean.leader.election.enable=false`를 유지하고 원인을 찾아야 합니다.

**1단계**: 장애 브로커를 빠르게 복구합니다. Kafka 브로커가 재시작되면 해당 파티션의 복제본이 ISR에 복귀하고 Leader 선출이 가능해집니다.

**2단계**: 브로커 복구가 불가능한 경우(하드웨어 영구 장애), `unclean.leader.election.enable=true`를 해당 토픽에 임시로 설정하고 ISR 밖 브로커를 Leader로 선출합니다. 이 경우 데이터 유실이 발생하고, 이후 분석/복구 절차가 필요합니다.

**3단계**: unclean election 후에는 반드시 해당 설정을 다시 false로 원복하고, 데이터 유실 범위를 파악합니다.

예방: ISR 이탈 알람 설정으로 사전에 감지하고, `min.insync.replicas=2`로 단일 브로커 의존을 방지합니다.

</details>

---

**Q2. Controller 브로커 자체가 장애나면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

새 Controller 브로커가 선출됩니다. ZooKeeper 모드에서는 Controller의 `/kafka/controller` ephemeral znode가 삭제되고, 나머지 브로커들이 해당 znode를 선착순으로 생성하려 경쟁합니다. 성공한 브로커가 새 Controller가 됩니다.

KRaft 모드에서는 Raft 프로토콜로 새 Active Controller(Raft Leader)가 선출됩니다. ZooKeeper 모드보다 선출이 빠르고 안정적입니다.

새 Controller 선출 중에는 파티션 Leader 변경 등 메타데이터 변경이 불가합니다. 기존 파티션의 읽기/쓰기는 계속 가능합니다(기존 Leader 브로커들이 그대로 서비스). Controller 장애가 파티션 서비스에 직접 영향을 주지는 않습니다.

</details>

---

**Q3. `preferred.leader`와 현재 Leader가 다를 때 강제로 Preferred Leader로 전환하면 어떤 위험이 있나요?**

<details>
<summary>해설 보기</summary>

Preferred Leader 선출 자체에는 데이터 유실 위험이 없습니다. ISR에 있는 브로커 간의 Leader 전환이기 때문입니다.

단, 선출 과정에서 해당 파티션이 수 ms ~ 수백 ms 동안 일시적으로 불가용합니다. Producer는 `LEADER_NOT_AVAILABLE`을 받고 재시도하고, Consumer는 잠깐 Fetch가 중단됩니다.

**위험이 있는 경우**: Preferred Leader가 현재 ISR에 없을 때 강제 선출을 시도하면 실패합니다. Preferred Leader가 ISR에 복귀할 때까지 대기해야 합니다.

**트래픽 영향 최소화**: 트래픽이 낮은 새벽 시간에 Preferred Leader 재선출을 실행하거나, `kafka-reassign-partitions`로 파티션을 재분배할 때 함께 처리하는 것이 일반적입니다.

</details>

---

<div align="center">

**[⬅️ 이전: min.insync.replicas](./04-min-insync-replicas.md)** | **[홈으로 🏠](../README.md)** | **[다음: Log Compaction ➡️](./06-log-compaction.md)**

</div>
