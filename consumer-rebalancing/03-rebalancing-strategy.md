# 리밸런싱 전략 비교 — Eager vs Cooperative

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Eager 리밸런싱이 Stop-The-World를 만드는 이유는?
- Cooperative(Incremental) 리밸런싱이 이동하는 파티션만 재할당하는 원리는?
- `CooperativeStickyAssignor`를 활성화하면 실제로 무엇이 달라지는가?
- Eager에서 Cooperative로 마이그레이션하는 과정은 어떻게 진행되는가?
- `StickyAssignor` vs `CooperativeStickyAssignor`의 차이는?
- Cooperative 리밸런싱에서도 파티션 처리가 완전히 중단되지 않는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka 2.4부터 Cooperative(Incremental) Rebalancing이 도입됐고, Kafka 3.1부터는 `CooperativeStickyAssignor`가 기본 할당 전략으로 권장된다. 기존 `RangeAssignor`(기본값)는 Eager 방식이라 Consumer 추가/제거마다 전체 처리가 멈춘다.

10개의 Consumer가 파티션 30개를 처리 중일 때 Consumer 1개를 추가하면:
- **Eager**: 30개 파티션이 모두 처리 중단 → 재할당 → 10개 파티션이 움직임 → 20초간 중단
- **Cooperative**: 10개 파티션만 재할당 → 나머지 20개는 계속 처리 → 중단 최소화

운영 환경에서 Rolling Restart나 Auto Scaling 시 이 차이는 결정적이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 기본 설정(RangeAssignor)으로 Auto Scaling 적용

  Auto Scaling 정책:
    Consumer Lag > 10,000 → Consumer 추가
    Consumer Lag < 1,000 → Consumer 제거

  문제:
    Lag 급증 → Consumer 1개 추가 → Eager 리밸런싱
    모든 Consumer 처리 중단 ~10~30초
    처리 멈춤 → Lag 더 증가 → Consumer 또 추가 → 또 리밸런싱
    = 리밸런싱 루프 (리밸런싱이 Lag을 더 악화시키는 피드백 루프)

  해결:
    CooperativeStickyAssignor 사용 → 재할당 파티션만 중단
    Lag 임계값을 보수적으로 설정 (리밸런싱 쿨다운 시간 고려)

실수 2: StickyAssignor와 CooperativeStickyAssignor 혼용

  설정:
    Consumer A: partition.assignment.strategy=StickyAssignor
    Consumer B: partition.assignment.strategy=CooperativeStickyAssignor

  결과:
    두 Consumer가 서로 다른 프로토콜을 선언
    GC: 교집합 프로토콜을 찾을 수 없음 → 협상 실패
    Eager 방식으로 폴백 (최소 공통 분모)
    
  마이그레이션 방법:
    모든 Consumer 인스턴스를 동시에 CooperativeStickyAssignor로 변경
    또는 Rolling 업그레이드 중 임시 두 전략 모두 선언

실수 3: Cooperative 리밸런싱 = 중단 없음이라는 오해

  현실:
    Cooperative에서도 재할당 대상 파티션은 처리가 멈춤
    "이동하는 파티션"만 중단, "유지되는 파티션"은 계속 처리
    완전히 중단 없지는 않지만 중단 범위가 대폭 축소됨
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
CooperativeStickyAssignor 설정 (Kafka 2.4+):

  # application.yml (Spring Kafka)
  spring:
    kafka:
      consumer:
        properties:
          partition.assignment.strategy: >
            org.apache.kafka.clients.consumer.CooperativeStickyAssignor

  # kafka-console-consumer
  --consumer-property \
    partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor

Rolling 마이그레이션 (Eager → Cooperative):
  # 1단계: 모든 Consumer에 두 전략 선언 (순서 중요: Cooperative 먼저)
  partition.assignment.strategy=\
    org.apache.kafka.clients.consumer.CooperativeStickyAssignor,\
    org.apache.kafka.clients.consumer.EagerAssignor

  # 2단계: 순차적으로 Consumer 재시작 (Cooperative 우선 선택됨)

  # 3단계: 모든 인스턴스가 Cooperative 지원 확인 후 EagerAssignor 제거
  partition.assignment.strategy=\
    org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

---

## 🔬 내부 동작 원리

### 1. Eager 리밸런싱: Stop-The-World

```
초기 상태: Consumer 3개, 파티션 6개

  C1 → [P0, P1]
  C2 → [P2, P3]
  C3 → [P4, P5]

  이벤트: C4 추가

  Eager 리밸런싱 흐름:

  Phase 1 (Stop-The-World 시작):
    GC: PreparingRebalance 상태 전이
    C1, C2, C3: 현재 파티션 모두 반납 (revoke)
    → 모든 파티션 처리 중단!
    onPartitionsRevoked([P0,P1]) → onPartitionsRevoked([P2,P3]) → ...

  Phase 2: JoinGroup
    C1, C2, C3, C4 모두 JoinGroup 전송
    GC: 새 할당 계획 수립 대기

  Phase 3: SyncGroup
    Group Leader(C1): 새 할당 계획 수립
    C1 → [P0, P1]  (변화 없음)
    C2 → [P2, P3]  (변화 없음)
    C3 → [P4, P5]  (변화 없음)
    C4 → []        (아직 없음, 이후 재배분)
    
    두 번째 리밸런싱:
    C3이 P5를 C4에게 양보 → 또 리밸런싱 발생 (할당 전략에 따라)
    또는 처음 할당 시 C4에게 P5 배분:
    C1 → [P0, P1], C2 → [P2, P3], C3 → [P4], C4 → [P5]

  Phase 4: Stable
    onPartitionsAssigned([...]) 호출 → Fetch 재개

  중단 시간: Phase 1~4 전체 = ~5~30초 (그룹 크기에 따라)
  중단된 파티션: 6개 전부 (실제 이동은 1개 파티션뿐이었는데)
```

### 2. Cooperative(Incremental) 리밸런싱

```
초기 상태: Consumer 3개, 파티션 6개

  C1 → [P0, P1]
  C2 → [P2, P3]
  C3 → [P4, P5]

  이벤트: C4 추가

  Cooperative 리밸런싱 흐름:

  Round 1: "누가 파티션을 포기해야 하나?" 결정
    GC: PreparingRebalance
    C1, C2, C3, C4: JoinGroup (현재 보유 파티션 정보 포함)
    Group Leader: 새 목표 할당 계산
      목표: C1[P0,P1], C2[P2,P3], C3[P4], C4[P5]
    SyncGroup 응답:
      C1: 아무것도 반납하지 않아도 됨 → 계속 처리
      C2: 아무것도 반납하지 않아도 됨 → 계속 처리
      C3: P5 반납 예정 (revoke 대상으로 표시)
      C4: 아무 파티션 없음 (아직 할당 못 받음)

    ★ C1, C2는 중단 없이 계속 처리!
    ★ C3만 P5를 revoke (P4는 계속 처리)

  Round 2: "반납된 파티션을 새 Consumer에게 배정"
    GC: 다시 PreparingRebalance (2차 리밸런싱)
    C3: P5 revoke 완료 → JoinGroup
    C4: JoinGroup (P5 받을 준비)
    SyncGroup: C4 → [P5] 할당

  결과:
    C1[P0,P1], C2[P2,P3]: 전체 과정에서 처리 중단 없음
    C3[P4]: P5 revoke 구간만 일시 중단 → 곧 재개
    C4[P5]: Round 2 완료 후 처리 시작
    
  중단된 파티션: P5만 (1개)
  Eager 대비: 6개 → 1개 중단 (83% 감소)
```

### 3. StickyAssignor vs CooperativeStickyAssignor

```
공통: "기존 할당을 최대한 유지" (Sticky)
  리밸런싱 후에도 가능하면 기존 Consumer에게 같은 파티션 유지

차이:

  StickyAssignor (Eager 방식):
    Stop-The-World
    모든 파티션 반납 후 재할당
    재할당 결과는 Sticky (기존과 동일하게 유지)
    단, 반납~재할당 사이에 전체 중단 발생

  CooperativeStickyAssignor (Cooperative 방식):
    이동 필요한 파티션만 반납
    재할당 결과도 Sticky
    이동 불필요한 파티션은 중단 없이 계속 처리

  RangeAssignor (기존 기본값, Eager):
    토픽별로 Range로 할당 (P0~P1 → C1, P2~P3 → C2 등)
    Sticky 없음: 리밸런싱마다 할당이 달라질 수 있음
    Stop-The-World

선택 기준 (2024년 기준):
  신규 프로젝트: CooperativeStickyAssignor (Kafka 2.4+)
  레거시: 마이그레이션 계획 수립 → CooperativeStickyAssignor 전환
  특수 요건 (특정 파티션을 특정 Consumer에 고정):
    CustomPartitionAssignor 구현
```

### 4. 2라운드 리밸런싱의 이해

```
Cooperative 리밸런싱이 "2라운드"인 이유:

  Round 1 목적: "어떤 파티션이 이동해야 하는지" 결정 + 이동 대상 revoke
  Round 2 목적: revoke된 파티션을 새 Consumer에게 배정

  왜 1라운드에서 완료 못 하나?
    Round 1 SyncGroup 시점에 C3이 P5를 아직 revoke 안 함
    C4에게 P5를 할당하려면 C3가 P5를 먼저 반납해야 함
    → 순서 보장: revoke 완료 후 assign (같은 파티션의 중복 처리 방지)
    → 2라운드가 불가피

  2라운드 리밸런싱의 비용:
    Round 1: JoinGroup + SyncGroup (전체 멤버)
    Round 2: JoinGroup + SyncGroup (전체 멤버, 다시)
    → 총 2번의 프로토콜 오버헤드 (Eager의 1번보다 많음)
    → 단, 각 라운드의 처리 중단이 적어서 전체 영향 감소

  파티션 이동이 많을수록 라운드 수 증가 가능
  (각 라운드에서 일부 파티션씩 이동)
```

---

## 💻 실전 실험

### 실험 1: Eager vs Cooperative 처리 중단 시간 비교

```bash
# Eager (기본값 RangeAssignor)
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic rebalance-test --group eager-group

# 별도 터미널에서 Consumer 추가 (처리 중단 시간 측정)
time kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic rebalance-test --group eager-group &

# CooperativeStickyAssignor
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic rebalance-test --group cooperative-group \
  --consumer-property \
    partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor

# Consumer 추가 (처리 중단 시간 측정)
time kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic rebalance-test --group cooperative-group \
  --consumer-property \
    partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor &

# 처리 중단 감지: 메시지 타임스탬프 갭으로 측정
```

### 실험 2: 할당 결과 변화 관찰

```bash
# CooperativeStickyAssignor 적용 Consumer 3개 시작
for i in 1 2 3; do
  kafka-console-consumer \
    --bootstrap-server localhost:19092 \
    --topic rebalance-test --group sticky-group \
    --consumer-property partition.assignment.strategy=\
org.apache.kafka.clients.consumer.CooperativeStickyAssignor \
    --consumer-property client.id=consumer-$i &
done

# 할당 상태 확인
sleep 5
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group sticky-group

# Consumer 1개 추가
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic rebalance-test --group sticky-group \
  --consumer-property partition.assignment.strategy=\
org.apache.kafka.clients.consumer.CooperativeStickyAssignor \
  --consumer-property client.id=consumer-4 &

# 리밸런싱 후 할당 변화 확인 (최소 변경 확인)
sleep 10
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group sticky-group
# consumer-1,2,3의 파티션 할당이 최대한 유지됐는지 확인
```

### 실험 3: 리밸런싱 라운드 수 로그 확인

```bash
# Consumer 로그에서 Cooperative 리밸런싱 패턴 확인
grep -E "Cooperative|Round|Rebalance" consumer.log

# 예상 패턴:
# [Consumer] Request joining group due to: group is already rebalancing
# [Consumer] Revoked partitions assigned at previous generation: [rebalance-test-5]
# [Consumer] Successfully joined group with generation 3
# [Consumer] Notifying assignor about the cooperative assignment
```

---

## 📊 성능/비용 비교

### Eager vs Cooperative 리밸런싱 영향 비교

```
조건: Consumer 10개, 파티션 30개, 1개 Consumer 추가

  Eager (RangeAssignor):
    중단된 파티션: 30개 (전체)
    중단 시간: ~10~30초
    이동한 파티션: 3개 (실제 필요한 이동)
    낭비: 27개 파티션이 불필요하게 중단

  Cooperative (CooperativeStickyAssignor):
    중단된 파티션: 3개 (이동 필요한 것만)
    중단 시간: ~2~5초 (2라운드 포함)
    이동한 파티션: 3개 (정확히 필요한 만큼)
    중단 감소: 90% (30개 → 3개)

파티션 100개, Consumer 20개:
  Eager: 100개 중단, ~30~60초
  Cooperative: 5개 중단, ~2~10초
  중단 감소: 95%
```

---

## ⚖️ 트레이드오프

```
Eager (RangeAssignor, 기본값):
  ✅ 구현 단순, 예측 가능
  ✅ 1라운드에 리밸런싱 완료
  ❌ Stop-The-World: 전체 파티션 중단
  ❌ 대규모 그룹에서 중단 시간 수십 초
  ❌ Auto Scaling 시 Lag 악화 피드백 루프

CooperativeStickyAssignor:
  ✅ 이동 파티션만 중단 → 처리 연속성 높음
  ✅ Auto Scaling 친화적
  ✅ Sticky로 기존 할당 최대 유지
  ❌ 2라운드 리밸런싱 → 프로토콜 오버헤드 (소수 추가)
  ❌ 마이그레이션 필요 (기존 Eager 전략과 혼용 불가)
  ❌ 일부 엣지 케이스에서 할당 불균형 가능 (드묾)

권장 (2024 기준):
  신규: CooperativeStickyAssignor 기본 설정
  운영 중: 점진적 마이그레이션 (두 전략 동시 선언 → 순차 재시작)
```

---

## 📌 핵심 정리

```
리밸런싱 전략 핵심:

1. Eager = Stop-The-World
   모든 파티션 반납 → JoinGroup → SyncGroup → 전체 재할당
   실제 이동 파티션보다 훨씬 많은 파티션이 중단

2. Cooperative = 이동 파티션만 revoke
   Round 1: 이동 대상 파티션 확인 + revoke
   Round 2: revoke된 파티션을 새 Consumer에게 배정
   유지 파티션은 전체 과정 중 중단 없음

3. CooperativeStickyAssignor = Cooperative + Sticky
   이동 최소화 + 기존 할당 최대 유지
   Kafka 3.1+에서 권장 기본 전략

4. 마이그레이션:
   두 전략 동시 선언 → 순차 재시작 → 단일 전략으로 정리

5. Auto Scaling 환경에서 Cooperative 필수
   Eager의 리밸런싱 루프(리밸런싱 → Lag 증가 → 또 리밸런싱) 예방
```

---

## 🤔 생각해볼 문제

**Q1. Cooperative 리밸런싱에서 Round 1과 Round 2 사이에 revoke된 파티션을 다른 Consumer가 임시로 처리하면 안 되나요?**

<details>
<summary>해설 보기</summary>

안 됩니다. Revoke된 파티션은 Round 2가 완료될 때까지 어떤 Consumer에게도 할당되지 않습니다. 이 빈틈이 Cooperative 리밸런싱의 유일한 중단 구간입니다.

임시 처리를 허용하면 중복 처리 문제가 발생합니다. C3이 아직 P5의 offset 300을 처리 중인데 C4가 P5를 받아서 offset 300부터 처리를 시작하면 300번 메시지가 두 번 처리됩니다. Kafka는 이를 방지하기 위해 revoke된 파티션이 새 Consumer에 완전히 넘어가기 전까지 처리 공백을 허용합니다.

이 공백을 최소화하려면 `onPartitionsRevoked()`에서 처리 중인 작업을 빠르게 완료하고 offset을 커밋하는 것이 중요합니다.

</details>

---

**Q2. `CooperativeStickyAssignor`로 마이그레이션 중 일부 Consumer는 구버전(RangeAssignor), 일부는 신버전(Cooperative)을 사용하면 어떤 할당 전략이 적용되나요?**

<details>
<summary>해설 보기</summary>

Kafka는 그룹 내 모든 Consumer가 공통으로 지원하는 할당 전략을 선택합니다. Group Leader가 모든 멤버의 지원 전략 목록을 받아 교집합을 구합니다.

만약 일부는 `[RangeAssignor]`, 일부는 `[CooperativeStickyAssignor]`만 선언했다면 교집합이 없어 협상 실패가 발생하고 에러가 납니다.

올바른 마이그레이션 방법은 과도기에 모든 Consumer가 두 전략을 모두 선언하는 것입니다: `partition.assignment.strategy=CooperativeStickyAssignor,RangeAssignor`. 모든 인스턴스가 두 전략을 지원하면 Group Leader가 `CooperativeStickyAssignor`(높은 우선순위)를 선택합니다. 이후 `RangeAssignor`를 제거하면 완전히 전환됩니다.

</details>

---

**Q3. `StickyAssignor`를 이미 사용 중인데 `CooperativeStickyAssignor`로 전환하면 운영 중에 어떤 위험이 있나요?**

<details>
<summary>해설 보기</summary>

가장 큰 위험은 전환 과정에서 일시적으로 Eager 방식으로 폴백될 수 있다는 점입니다.

단계별 안전한 전환:
1. 모든 Consumer 설정에 `CooperativeStickyAssignor,StickyAssignor` 순서로 두 전략을 모두 선언합니다.
2. 순차적으로 Consumer를 재시작합니다. 이 과정에서 `StickyAssignor`가 공통 분모로 적용됩니다.
3. 모든 Consumer가 재시작 완료되면 모든 인스턴스가 두 전략을 지원합니다. 다음 리밸런싱에서 `CooperativeStickyAssignor`가 선택됩니다.
4. 이후 `StickyAssignor`를 설정에서 제거합니다.

이 과정에서 리밸런싱은 발생하지만 서비스 중단은 최소화됩니다. 피크 타임을 피해 마이그레이션하는 것이 안전합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 리밸런싱 발생 조건](./02-rebalancing-trigger.md)** | **[홈으로 🏠](../README.md)** | **[다음: 리밸런싱 중 중복 처리 ➡️](./04-rebalancing-duplicate.md)**

</div>
