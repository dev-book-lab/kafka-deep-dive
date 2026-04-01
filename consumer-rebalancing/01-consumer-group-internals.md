# Consumer Group 내부 동작 — Group Coordinator와 상태 머신

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Group Coordinator 브로커는 어떻게 결정되고, 무엇을 관리하는가?
- Consumer Group의 4가지 상태(`Empty` → `PreparingRebalance` → `CompletingRebalance` → `Stable`)는 어떻게 전이되는가?
- Group Leader Consumer와 Group Coordinator의 역할이 어떻게 분리되는가?
- JoinGroup / SyncGroup 요청의 흐름은?
- Consumer가 처음 그룹에 참여할 때 파티션을 어떻게 받는가?
- `__consumer_offsets` 토픽이 Group Coordinator와 어떻게 연결되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Consumer가 추가됐는데 왜 메시지 처리가 잠깐 멈췄지?", "리밸런싱이 왜 이렇게 자주 일어나지?" 같은 질문의 답은 Group Coordinator와 상태 머신을 이해하는 데 있다.

Group Coordinator가 어느 브로커인지 모르면 해당 브로커 장애 시 Consumer Group 전체가 영향받는 것을 예측하지 못한다. Consumer Group 상태 머신을 모르면 리밸런싱 중 `PreparingRebalance` 상태에서 왜 메시지 처리가 멈추는지 설명할 수 없다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Group Coordinator를 모든 브로커라고 오해

  현실:
    Consumer Group 하나 = 특정 브로커 1개가 Group Coordinator
    hash(groupId) % 50 → __consumer_offsets 파티션 N
    → 파티션 N의 Leader 브로커 = Group Coordinator

  결과:
    Group Coordinator 브로커에 장애 → 해당 GC 담당 모든 그룹 일시 중단
    장애 복구 전까지 JoinGroup, offset 커밋 불가
    → GC 브로커 집중 부하 방지: 그룹 ID를 다양하게 설계

실수 2: Consumer 10개 = 항상 10개가 처리 중이라는 가정

  현실:
    CompletingRebalance 상태에서 모든 Consumer가 SyncGroup 응답 대기
    이 시간 동안 Fetch 없음 = 메시지 처리 멈춤
    그룹 크기가 클수록 모든 Consumer가 준비되는 시간 증가

실수 3: Group Leader가 Kafka 브로커라는 오해

  현실:
    Group Leader = JoinGroup 요청을 가장 먼저 보낸 Consumer 인스턴스
    (브로커 아님)
    Group Leader가 파티션 할당 계획 수립
    → 브로커(GC)는 할당 결과를 다른 멤버에게 전달하는 역할만
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Group Coordinator 확인:
  # 특정 그룹의 GC 브로커 확인
  kafka-consumer-groups --bootstrap-server localhost:19092 \
    --describe --group order-group
  # Coordinator: 브로커 ID 표시

  # 브로커별 담당 그룹 수 확인 (부하 분산)
  # 그룹 ID를 다양하게 설계해서 __consumer_offsets 파티션 분산

Consumer Group 상태 모니터링:
  kafka-consumer-groups --bootstrap-server localhost:19092 \
    --describe --group order-group
  # State: Stable / PreparingRebalance / CompletingRebalance / Empty 확인

  JMX 모니터링:
    kafka.coordinator.group:type=GroupMetadataManager
    지표: NumOffsets, NumGroups
    → 그룹 수와 offset 수 추이 모니터링

그룹 ID 설계:
  서비스별: "payment-service-consumer"
  인스턴스별 고유 ID 불필요 (그룹 내 여러 인스턴스가 같은 groupId 사용)
  목적별 분리: 실시간 처리 그룹 / 배치 처리 그룹 / 재처리 그룹 분리
```

---

## 🔬 내부 동작 원리

### 1. Group Coordinator 결정 방식

```
Group Coordinator 할당:

  1. Consumer가 bootstrap.servers에 연결 → 메타데이터 조회
  2. FindCoordinator 요청 전송 (groupId 포함)
  3. 브로커: hash(groupId) % 50 = N
             __consumer_offsets 파티션 N의 Leader 브로커 = GC
  4. Consumer → GC 브로커에 연결

  GC 브로커가 관리하는 것:
    - 해당 그룹의 멤버십 (어떤 Consumer 인스턴스가 있는지)
    - 리밸런싱 조율 (JoinGroup, SyncGroup 처리)
    - offset 커밋 저장 (__consumer_offsets 해당 파티션에 기록)
    - 세션 타임아웃 모니터링

  GC 이중화:
    __consumer_offsets 파티션은 복제 팩터 3 (기본)
    GC 브로커 장애 → 해당 파티션의 새 Leader 브로커가 GC 역할 승계
    약간의 지연 후 그룹 운영 재개
```

### 2. Consumer Group 상태 머신

```
상태 전이 흐름:

  [Empty]
    그룹에 Consumer 없음 / 모든 멤버 이탈
    오프셋만 남아있는 상태 가능
    │
    │ Consumer JoinGroup 요청
    ▼
  [PreparingRebalance]
    리밸런싱 준비 중
    기존 멤버에게 "그룹 참여 재요청" 알림
    모든 멤버가 JoinGroup 요청 보낼 때까지 대기
    ★ 이 상태에서 Consumer는 Fetch 중단 (메시지 처리 멈춤)
    │
    │ 모든 멤버 JoinGroup 완료
    ▼
  [CompletingRebalance]
    Group Leader가 파티션 할당 계획 수립 중
    Group Leader → GC에게 SyncGroup 전송 (할당 결과 포함)
    다른 멤버들도 SyncGroup 요청 (할당 결과 수신 대기)
    ★ 이 상태에서도 Fetch 중단 지속
    │
    │ GC가 모든 멤버에게 SyncGroup 응답 (파티션 할당 정보)
    ▼
  [Stable]
    각 Consumer가 할당받은 파티션에서 Fetch 시작
    heartbeat 주기적 전송
    정상 운영 상태
    │
    │ Consumer 추가/제거 / 세션 타임아웃 / max.poll.interval.ms 초과
    ▼
  [PreparingRebalance] (리밸런싱 재시작)
```

### 3. JoinGroup / SyncGroup 상세 흐름

```
Consumer 3개 (C1, C2, C3), 파티션 3개 그룹 초기화:

  Phase 1: JoinGroup
    C1 → GC: JoinGroup(groupId, memberId=null, protocols=[range,sticky])
    GC → C1: memberId="C1-uuid", leaderId="C1-uuid" (첫 번째 참여자 = Leader)
    
    C2 → GC: JoinGroup(groupId)
    GC → C2: memberId="C2-uuid", leaderId="C1-uuid"
    
    C3 → GC: JoinGroup(groupId)
    GC → C3: memberId="C3-uuid", leaderId="C1-uuid"
    
    GC는 모든 멤버 목록을 JoinGroup 응답에 포함
    → C1(Leader)만 전체 멤버 목록 수신
    → C2, C3는 자신의 멤버 ID만 수신

  Phase 2: SyncGroup
    C1(Leader): 멤버 목록 기반으로 파티션 할당 계산
      Partition 0 → C1
      Partition 1 → C2
      Partition 2 → C3
    
    C1 → GC: SyncGroup(할당 결과 포함)
    C2 → GC: SyncGroup(빈 할당 결과)
    C3 → GC: SyncGroup(빈 할당 결과)
    
    GC → C1: 자신의 할당: [Partition 0]
    GC → C2: 자신의 할당: [Partition 1]
    GC → C3: 자신의 할당: [Partition 2]

  Phase 3: 상태 → Stable
    각 Consumer가 할당 파티션의 마지막 커밋 offset 조회
    해당 offset부터 Fetch 시작
```

### 4. Heartbeat와 세션 관리

```
HeartbeatThread (poll()과 독립적인 별도 스레드):

  heartbeat.interval.ms=3000 마다 GC에게 heartbeat 전송
  GC는 heartbeat 수신 시 세션 갱신

  session.timeout.ms=10000 (10초) 동안 heartbeat 없으면:
    GC: "해당 Consumer 장애" 판단
    GC: 해당 Consumer를 그룹에서 제거
    GC: PreparingRebalance 상태로 전이

  poll() 호출 간격 체크 (HeartbeatThread와 다른 메커니즘):
    max.poll.interval.ms=300000 (5분)
    5분 동안 poll() 없으면:
    → Consumer가 GC에게 LeaveGroup 요청 전송
    → GC: PreparingRebalance 상태로 전이

  두 타이머의 독립성:
    HeartbeatThread는 poll() 없어도 계속 heartbeat 전송
    그러나 max.poll.interval.ms 초과 시 LeaveGroup → 리밸런싱
    → "heartbeat는 정상인데 리밸런싱이 왜?"의 원인
```

### 5. __consumer_offsets와 GC의 관계

```
offset 커밋 흐름:

  Consumer → GC 브로커: OffsetCommitRequest
  GC 브로커: __consumer_offsets 파티션 N에 기록
             (GC = 해당 파티션 Leader이므로 로컬 기록)

  offset 조회 흐름:
  Consumer 시작 → GC 브로커: OffsetFetchRequest
  GC 브로커: __consumer_offsets에서 해당 그룹의 최신 offset 반환
  Consumer: 반환된 offset부터 Fetch 시작

  GC 장애 시:
    __consumer_offsets 파티션 Leader 교체 (ISR에서 새 Leader 선출)
    새 Leader = 새 GC
    Consumer: 새 GC로 재연결 (FindCoordinator 재요청)
    → 약간의 지연 후 정상화
```

---

## 💻 실전 실험

### 실험 1: Consumer Group 상태 모니터링

```bash
# 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic group-test --partitions 3 --replication-factor 1

# Consumer 1개 시작 (터미널 1)
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic group-test --group monitor-group

# 상태 확인 (터미널 2)
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group monitor-group
# State: Stable 확인

# Consumer 2개 추가 (터미널 3, 4)
# 추가 시점에 상태 변화 관찰:
watch -n 1 'kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group monitor-group'
# State: Stable → PreparingRebalance → CompletingRebalance → Stable
```

### 실험 2: Group Coordinator 브로커 확인

```bash
# 특정 그룹의 GC 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group monitor-group 2>&1 | head -5
# Coordinator (id: 1, host: kafka-1, port: 9092)  ← GC 브로커

# groupId별 GC 분산 확인
for group in group-a group-b group-c; do
  kafka-consumer-groups --bootstrap-server localhost:19092 \
    --describe --group $group 2>&1 | grep "Coordinator"
done
# 각 그룹의 GC가 다른 브로커인지 확인 (부하 분산)
```

### 실험 3: JoinGroup / SyncGroup 로그 확인

```bash
# 브로커 로그에서 리밸런싱 관련 로그 확인
docker exec kafka-1 bash -c \
  "tail -f /var/log/kafka/server.log | grep -E 'Preparing|Completing|Stable|JoinGroup|SyncGroup'"

# Consumer 추가 시 로그 패턴:
# [GroupCoordinator] Group monitor-group with generation ... is now PreparingRebalance
# [GroupCoordinator] Stabilized group monitor-group with 3 members
```

---

## 📊 성능/비용 비교

### 그룹 크기별 리밸런싱 소요 시간

```
Consumer 수에 따른 JoinGroup + SyncGroup 소요 시간:

  Consumer 3개:
    JoinGroup 완료: ~1~3초 (모든 멤버 응답 대기)
    SyncGroup 완료: ~0.5~1초
    총 중단 시간: ~1.5~4초

  Consumer 10개:
    JoinGroup 완료: ~3~8초
    SyncGroup 완료: ~1~2초
    총 중단 시간: ~4~10초

  Consumer 50개:
    JoinGroup 완료: ~10~30초
    SyncGroup 완료: ~2~5초
    총 중단 시간: ~12~35초

→ 그룹 크기가 클수록 리밸런싱 중단 시간 증가
→ 대규모 그룹에서 Cooperative Rebalancing 필수 (Ch4-03 참고)

rebalance.timeout.ms (기본 300초):
  JoinGroup 응답 대기 최대 시간
  초과 시 응답 안 한 Consumer를 제외하고 진행
```

---

## ⚖️ 트레이드오프

```
Consumer Group 크기:
  크게 (많은 Consumer):
    ✅ 파티션당 처리 부하 분산
    ❌ 리밸런싱 중단 시간 증가
    ❌ GC 브로커 부하 증가 (많은 heartbeat 관리)

  작게 (적은 Consumer):
    ✅ 리밸런싱 빠름
    ❌ 각 Consumer가 더 많은 파티션 처리 (처리량 한계)

session.timeout.ms:
  짧게: 빠른 장애 감지, 일시적 GC로도 리밸런싱 빈발
  길게: 안정적이지만 실제 장애 감지 늦음
  권장: 10~30초 (기본 10초 유지 or 적절히 증가)

Group Leader 역할 분리:
  ✅ GC 브로커가 할당 로직 없이 조율만 담당 → 브로커 부하 최소화
  ✅ 할당 알고리즘을 Client에서 구현 → 유연한 커스텀 할당 가능
  ❌ Group Leader Consumer 장애 시 다른 멤버가 Leader 역할 재선출 필요
```

---

## 📌 핵심 정리

```
Consumer Group 내부 핵심:

1. Group Coordinator = hash(groupId) % 50번 __consumer_offsets 파티션 Leader 브로커
   GC가 멤버십, 리밸런싱, offset 커밋 관리

2. Group 상태 머신:
   Empty → PreparingRebalance → CompletingRebalance → Stable
   PreparingRebalance + CompletingRebalance 구간 = Fetch 중단 = 처리 멈춤

3. 역할 분리:
   GC(브로커): 조율 담당 (JoinGroup 수집, SyncGroup 배포)
   Group Leader(Consumer): 파티션 할당 계산

4. Heartbeat vs max.poll.interval:
   heartbeat: Consumer 생존 신호 (HeartbeatThread)
   max.poll.interval.ms: Consumer 처리 속도 감시 (poll() 간격)
   둘 다 초과 시 리밸런싱 발생 (원인이 다름)

5. offset 커밋 = GC 브로커의 __consumer_offsets 파티션에 기록
   GC 장애 → 새 Leader 브로커가 GC 역할 승계
```

---

## 🤔 생각해볼 문제

**Q1. 동일한 `groupId`를 가진 Consumer 인스턴스 100개가 있고, 파티션이 3개라면 어떻게 할당되나요?**

<details>
<summary>해설 보기</summary>

파티션 3개는 Consumer 3개에만 할당됩니다. 나머지 97개는 파티션을 받지 못하고 IDLE 상태가 됩니다. Kafka에서 한 파티션은 동일 Consumer Group 내에서 하나의 Consumer에만 할당되기 때문입니다.

IDLE Consumer도 heartbeat는 유지하고, 그룹에 남아있습니다. 만약 활성 Consumer 중 하나가 장애나면 리밸런싱 후 IDLE Consumer 중 하나가 해당 파티션을 받아 처리를 재개합니다. 대기 Consumer가 많으면 장애 복구 시 파티션이 빠르게 재배정됩니다. 하지만 불필요한 서버 비용과 GC 부하를 야기합니다.

최적: Consumer 수 = 파티션 수

</details>

---

**Q2. Group Leader Consumer가 처리 중 크래시되면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

리밸런싱이 다시 발생합니다. Group Leader가 크래시되면 session.timeout.ms 후 GC가 해당 Consumer를 그룹에서 제거하고 PreparingRebalance 상태로 전이합니다. 남은 멤버들이 JoinGroup 요청을 다시 보내고, 이때 가장 먼저 응답한 Consumer가 새 Group Leader가 됩니다.

Group Leader 역할은 세션 단위로 부여됩니다. 특정 Consumer 인스턴스가 항상 Leader가 되는 것이 아니라, 리밸런싱마다 가장 먼저 JoinGroup을 보낸 인스턴스가 Leader입니다.

</details>

---

**Q3. 같은 애플리케이션을 두 개의 다른 `groupId`로 실행하면 어떤 차이가 있나요?**

<details>
<summary>해설 보기</summary>

두 그룹이 완전히 독립적으로 같은 토픽을 소비합니다. 각 그룹은 자신만의 offset을 관리하므로 groupId-A가 offset 300까지 처리했어도 groupId-B는 여전히 offset 0부터 읽을 수 있습니다.

이것이 Kafka의 강력한 기능 중 하나입니다. 실시간 처리 그룹, 분석 그룹, 감사 로그 그룹이 동일 토픽을 각자 독립적으로 소비할 수 있습니다. 단, 각 그룹이 독립적으로 Fetch하므로 브로커의 네트워크 아웃바운드 트래픽이 그룹 수에 비례합니다. 브로커의 PageCache가 충분하면 디스크 I/O 없이 여러 그룹이 효율적으로 읽을 수 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 리밸런싱 발생 조건 ➡️](./02-rebalancing-trigger.md)**

</div>
