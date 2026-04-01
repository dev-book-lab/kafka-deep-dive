# 토픽과 파티션 — 병렬성의 기본 단위

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 파티션이 "병렬성의 기본 단위"라는 말의 정확한 의미는?
- Consumer Group 내에서 파티션과 Consumer가 어떻게 1:1로 매핑되는가?
- 파티션 수를 늘리면 처리량이 증가하지만, 어떤 비용이 함께 증가하는가?
- 메시지 키를 지정하면 동일 키의 메시지가 순서가 보장되는 원리는?
- 파티션 수 선택 시 어떤 기준으로 결정해야 하는가?
- 파티션을 늘리는 것은 가능한데, 왜 줄이는 것은 불가능한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

파티션 수 선택은 Kafka 설계에서 가장 중요한 결정 중 하나다. 처음에 파티션을 3개로 만들었다가 나중에 처리량이 부족해서 30개로 늘리면, 키 기반으로 특정 파티션에 저장되던 메시지의 순서 보장이 깨진다. 반대로 파티션을 무조건 많이 만들면 브로커 파일 핸들 수와 Leader Election 비용이 증가한다.

파티션과 Consumer의 관계를 모르면:
- Consumer 인스턴스를 10개로 늘렸는데 파티션이 3개라 7개가 놀고 있는 상황
- 처리량을 높이려고 Consumer를 늘렸지만 파티션 수보다 Consumer 수가 많아서 효과 없음
- 특정 파티션에 메시지가 집중되는 "핫스팟"이 발생했는데 원인을 모름

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 파티션 수보다 Consumer를 많이 생성

  설정: 파티션 3개, Consumer 인스턴스 10개 (같은 group)
  결과:
    Consumer 1 → Partition 0 처리
    Consumer 2 → Partition 1 처리
    Consumer 3 → Partition 2 처리
    Consumer 4~10 → 아무 파티션도 할당받지 못함 (IDLE 상태)
  
  비용: Consumer 인스턴스 7개의 서버 비용 낭비
  해결: 파티션 수 = Consumer 인스턴스 수 맞추거나
        파티션 수만큼만 Consumer 인스턴스 유지

실수 2: 파티션 수를 처음부터 너무 적게 설정

  처음 설계: 파티션 3개, Consumer 3개
  6개월 후 트래픽 3배 증가 → 파티션을 9개로 늘림

  문제: 파티션 수 변경 시 키 해싱 결과가 달라짐
    키 "order-123":
      파티션 3개일 때: hash("order-123") % 3 = 0 (Partition 0)
      파티션 9개일 때: hash("order-123") % 9 = 6 (Partition 6)
    → 동일 키의 메시지가 다른 파티션으로 분산
    → 순서 보장이 요구되는 시나리오(같은 주문 ID의 이벤트)에서 순서 깨짐

실수 3: 파티션을 줄이려고 시도

  kafka-topics --alter --partitions 2  # 현재 파티션 3개 → 2개로 줄이려 함
  결과: ERROR: Partition count can only be increased (감소 불가)
  이유: 기존 파티션의 데이터를 어떤 파티션으로 병합할지 결정 불가능
       데이터 유실 없이 파티션 축소하는 방법 없음
  해결: 새 토픽(파티션 수 조정)을 만들고 트래픽을 점진적으로 이동
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
올바른 파티션 수 결정 공식:

  목표 처리량 기준:
    목표: 초당 1,000,000 메시지
    단일 파티션 처리량: ~100,000 msg/sec (메시지 크기, 처리 로직에 따라 다름)
    필요 파티션 수: 1,000,000 / 100,000 = 10개

  Consumer 기준:
    최대 Consumer 인스턴스 수 = 파티션 수
    예상 최대 병렬 Consumer 수를 고려해서 파티션 수 설정

  실무 권장 시작점:
    소규모: 파티션 6개 (Consumer 6개까지 병렬 가능)
    중규모: 파티션 12~24개
    대규모: 파티션 100개 이상 (Kafka는 수천 개 파티션도 가능)

  처음부터 여유 있게 설정:
    나중에 늘리면 키 해시 분포가 달라짐
    처음부터 예상 최대 처리량의 2~3배 파티션 수 설정 권장

올바른 키 설계:
  순서 보장이 필요한 단위 = 파티션 키
  예: 주문 처리 (주문 ID별 순서 보장 필요)
      → key = orderId
      → 같은 orderId의 메시지는 항상 같은 파티션
      → 파티션 내에서 순서 보장됨

  순서 보장 불필요한 경우:
      → key = null (Round-Robin 또는 Sticky)
      → 균등한 파티션 분산으로 처리량 최대화
```

---

## 🔬 내부 동작 원리

### 1. 토픽과 파티션의 물리적 구조

```
토픽 "orders" (파티션 3개, 복제 팩터 2):

  Broker 1                  Broker 2                  Broker 3
  ┌───────────────┐         ┌───────────────┐         ┌───────────────┐
  │ Partition 0   │ ←Leader │ Partition 0   │←Follower│               │
  │ offset: 0~99  │         │ offset: 0~99  │         │               │
  ├───────────────┤         ├───────────────┤         ├───────────────┤
  │               │         │ Partition 1   │ ←Leader │ Partition 1   │
  │               │         │ offset: 0~74  │         │ offset: 0~74  │
  ├───────────────┤         ├───────────────┤         ├───────────────┤
  │ Partition 2   │←Follower│               │         │ Partition 2   │
  │ offset: 0~120 │         │               │         │ offset: 0~120 │
  └───────────────┘         └───────────────┘         └───────────────┘
  
  - 각 파티션 = 독립된 순서가 있는 로그
  - 파티션 내에서 offset은 단조 증가 (0, 1, 2, ...)
  - 파티션 간에는 순서 보장 없음
  - 각 파티션은 브로커 파일시스템의 디렉토리로 존재
    /var/kafka-logs/orders-0/  ← Partition 0
    /var/kafka-logs/orders-1/  ← Partition 1
    /var/kafka-logs/orders-2/  ← Partition 2
```

### 2. Consumer Group과 파티션 할당 (1:1 규칙)

```
Consumer Group "order-processors" (파티션 3개):

  시나리오 A: Consumer 1개
    Consumer 1 → Partition 0 + Partition 1 + Partition 2 (전부 처리)

  시나리오 B: Consumer 2개
    Consumer 1 → Partition 0 + Partition 1
    Consumer 2 → Partition 2

  시나리오 C: Consumer 3개 (최적)
    Consumer 1 → Partition 0
    Consumer 2 → Partition 1
    Consumer 3 → Partition 2
    → 파티션 수 = Consumer 수 = 최대 병렬성

  시나리오 D: Consumer 4개 (낭비)
    Consumer 1 → Partition 0
    Consumer 2 → Partition 1
    Consumer 3 → Partition 2
    Consumer 4 → (IDLE, 어떤 파티션도 할당받지 못함)

규칙: 하나의 파티션은 동일 Consumer Group 내에서 
      반드시 하나의 Consumer에만 할당됨 (동시에 두 Consumer가 같은 파티션 읽기 불가)
이유: 파티션 내 순서 보장과 offset 관리의 단순성
```

### 3. 파티션 선택 알고리즘

```
Producer가 메시지를 보낼 때 파티션 선택:

  1. 키가 있을 때 (Key 해시):
     partition = hash(key) % numPartitions
     
     murmur2 해시 함수 사용:
       hash("order-123") = 1234567890
       1234567890 % 3 = 0 → Partition 0
       
     동일 키 → 항상 동일 파티션 → 파티션 내 순서 보장
     단, numPartitions 변경 시 동일 키가 다른 파티션으로 이동

  2. 키가 없고 partitioner.class=DefaultPartitioner일 때:
     Kafka 2.4 이전: Round-Robin
       메시지 1 → Partition 0
       메시지 2 → Partition 1
       메시지 3 → Partition 2
       메시지 4 → Partition 0 ...
     
     Kafka 2.4 이후: Sticky Partitioner (기본값)
       배치가 완성될 때까지 동일 파티션으로 전송
       배치 완성 → 다음 파티션으로 전환
       
       이유: Round-Robin은 각 배치 크기가 작아져 처리량 저하
             Sticky는 배치를 꽉 채워서 전송 → 더 효율적

  3. 커스텀 파티셔너:
     implements Partitioner 인터페이스
     partition() 메서드에서 원하는 로직 구현
     핫스팟 키를 여러 파티션에 분산하는 데 활용
```

### 4. 파티션과 처리량의 관계

```
처리량 확장 방법:

  Producer 관점:
    파티션이 많을수록 → Producer가 병렬로 여러 파티션에 쓰기 가능
    단, 각 파티션에 대한 배치 크기가 줄어들 수 있음

  Consumer 관점:
    파티션 수 = 최대 병렬 Consumer 수
    파티션 10개 → Consumer 10개까지 병렬 처리 가능

  브로커 관점:
    파티션마다 Leader 역할 → 파티션 많을수록 브로커 부하 증가
    파티션마다 파일 핸들 유지 (*.log, *.index, *.timeindex)
    → 브로커당 파티션 수가 너무 많으면 파일 핸들 한계 도달

  파티션 수의 부작용:
    ├── 파티션 수 ×2 → Leader Election 시 처리할 파티션 ×2
    ├── 복제 팩터 3 × 파티션 100개 = 파일 300세트 (브로커 분산 기준)
    └── Kafka 권장: 브로커당 최대 4,000 파티션, 클러스터당 200,000 이하
```

---

## 💻 실전 실험

### 실험 1: 파티션 수와 Consumer 할당 확인

```bash
# 파티션 3개 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic orders \
  --partitions 3 \
  --replication-factor 1

# Consumer 1개 시작 (터미널 1)
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders \
  --group test-group \
  --property print.partition=true

# Consumer 2개 시작 (터미널 2) - 같은 group
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders \
  --group test-group \
  --property print.partition=true

# Consumer 할당 상태 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group test-group

# 출력 예시:
# GROUP       TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
# test-group  orders  0          0               0               0    consumer-1-xxx
# test-group  orders  1          2               2               0    consumer-1-xxx
# test-group  orders  2          1               1               0    consumer-2-xxx
# → Consumer 1: Partition 0,1  Consumer 2: Partition 2
```

### 실험 2: 키 기반 파티션 분배 확인

```bash
# 키와 함께 메시지 발행
kafka-console-producer --bootstrap-server localhost:19092 \
  --topic orders \
  --property "parse.key=true" \
  --property "key.separator=:"

# 입력:
# order-1:{"amount":100}
# order-2:{"amount":200}
# order-1:{"amount":150}   ← order-1은 항상 같은 파티션
# order-3:{"amount":300}

# 어느 파티션으로 갔는지 확인
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true \
  --group verify-group

# 출력 예시:
# Partition: 0  Key: order-1  Value: {"amount":100}
# Partition: 2  Key: order-2  Value: {"amount":200}
# Partition: 0  Key: order-1  Value: {"amount":150}  ← 동일 파티션
# Partition: 1  Key: order-3  Value: {"amount":300}
```

### 실험 3: 파티션 수 증가

```bash
# 파티션 3개 → 6개로 증가
kafka-topics --bootstrap-server localhost:19092 \
  --alter --topic orders \
  --partitions 6

# 변경 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic orders

# 파티션 감소 시도 (실패 확인)
kafka-topics --bootstrap-server localhost:19092 \
  --alter --topic orders \
  --partitions 3
# ERROR: org.apache.kafka.common.errors.InvalidPartitionsException:
#        Topic currently has 6 partitions, which is higher than the requested 3.
```

---

## 📊 성능/비용 비교

### 파티션 수에 따른 성능 특성

```
파티션 수별 처리량 (단일 브로커, 1 KB 메시지, 복제 팩터 1):

  파티션 1개:
    쓰기 처리량: ~100 MB/s
    순서 보장:  전체 메시지 순서 보장
    Consumer:   최대 1개 병렬

  파티션 3개:
    쓰기 처리량: ~280 MB/s (파티션당 ~93 MB/s)
    순서 보장:  같은 키 내에서만
    Consumer:   최대 3개 병렬

  파티션 12개:
    쓰기 처리량: ~900 MB/s (디스크 I/O 한계에 근접)
    순서 보장:  같은 키 내에서만
    Consumer:   최대 12개 병렬

  파티션 100개 이상:
    쓰기 처리량: 디스크 I/O 한계에 의해 결정 (파티션 수 효과 감소)
    브로커 부하: Leader Election 비용, 파일 핸들 증가
    장애 복구:  재선출해야 하는 파티션 수 증가 → 복구 시간 증가

파티션 수 증가의 부작용:
  파티션 100개, 복제 팩터 3 = 파일 세트 300개/브로커
  브로커 1대 장애 시 재선출 파티션: 최대 100개
  재선출 소요 시간: 파티션 수 × 수 ms → 초 단위 증가 가능
```

### Consumer 인스턴스 수와 파티션 수의 최적 비율

```
  파티션 : Consumer = 1:1    최적 (각 Consumer가 1개 파티션 전담)
  파티션 : Consumer = 2:1    Consumer 1개가 2개 파티션 처리, 처리량 절반
  파티션 : Consumer = 1:2    Consumer 절반이 IDLE (낭비)

실무 추천:
  처리량 요구사항을 계산해서 필요 파티션 수 결정
  Consumer = 파티션 수로 맞추고
  처리량 부족 시 파티션 + Consumer 함께 증가
```

---

## ⚖️ 트레이드오프

```
파티션을 많이 만들면:
  ✅ Consumer 병렬성 증가 → 처리량 증가
  ✅ Producer 병렬 쓰기 → 쓰기 처리량 증가
  ❌ 브로커당 파일 핸들 수 증가
  ❌ Leader Election 시 복구 시간 증가
  ❌ 파티션 간 순서 보장 없음 (키 기반만 순서 보장)
  ❌ 메모리 사용량 증가 (파티션마다 버퍼 유지)

파티션을 적게 만들면:
  ✅ 운영 단순성 (파티션 수가 적을수록 관리 쉬움)
  ✅ Leader Election 빠름
  ✅ 전체 메시지 순서 보장 가능 (파티션 1개 시)
  ❌ Consumer 병렬성 제한
  ❌ 처리량 한계 (나중에 늘리면 키 해시 분포 깨짐)

키 기반 파티셔닝의 트레이드오프:
  ✅ 동일 키의 메시지 순서 보장
  ✅ 동일 키 관련 데이터를 동일 파티션에서 처리 → 로컬 상태 유지 용이
  ❌ 특정 키에 메시지 집중 시 파티션 핫스팟 발생
     예: key=userId이고 VIP 사용자가 전체 트래픽의 80%를 차지하는 경우
```

---

## 📌 핵심 정리

```
토픽과 파티션 핵심 규칙:

1. 파티션 = 병렬성의 최소 단위
   동일 Consumer Group 내에서 파티션 1개 = Consumer 1개 할당
   Consumer 수를 파티션 수보다 늘려도 병렬성은 증가하지 않음

2. 파티션 내 순서 O, 파티션 간 순서 X
   키를 지정하면 동일 키 → 동일 파티션 → 순서 보장
   키 없으면 Round-Robin/Sticky → 파티션 균등 분산, 순서 무보장

3. 파티션 수는 늘릴 수 있지만 줄일 수 없음
   처음 설계 시 예상 최대 처리량의 2~3배로 여유 있게 설정
   파티션 수 변경 시 키 해시 분포 변경 → 순서 보장 깨질 수 있음

4. 파티션 수 선택 기준
   = max(목표 처리량 / 단일 파티션 처리량, 최대 Consumer 인스턴스 수)
   실무 시작: 6~12개, 이후 모니터링하며 조정

5. 브로커당 파티션 수 한계
   Kafka 권장: 브로커당 최대 4,000 파티션
   초과 시 브로커 장애 복구 시간 급증
```

---

## 🤔 생각해볼 문제

**Q1. 파티션 수를 1개로 설정하면 전체 토픽의 순서가 완벽하게 보장됩니다. 그러면 모든 토픽을 파티션 1개로 쓰면 안 되나요?**

<details>
<summary>해설 보기</summary>

파티션이 1개면 순서는 완벽히 보장되지만 두 가지 심각한 문제가 있습니다.

**병렬성 제한**: Consumer를 아무리 많이 늘려도 파티션 1개이므로 Consumer 1개만 처리합니다. 처리량을 늘리려면 파티션 수를 늘려야 합니다.

**단일 장애점**: 파티션 1개의 Leader가 있는 브로커에 문제가 생기면 (Follower가 있더라도 Leader 재선출 동안) 해당 파티션이 일시적으로 불가용합니다. 파티션이 여러 개면 나머지 파티션은 계속 동작합니다.

실무에서 전체 메시지 순서가 필요한 경우는 매우 드뭅니다. 대부분 "사용자별 순서"나 "주문별 순서"처럼 키 단위 순서가 필요합니다. 이런 경우 파티션을 여러 개 두고 키 기반 파티셔닝을 사용합니다.

</details>

---

**Q2. Consumer Group A가 파티션 0을 처리하던 중 Consumer 한 명이 추가되면 어떤 일이 발생하나요?**

<details>
<summary>해설 보기</summary>

리밸런싱(Rebalancing)이 발생합니다. Group Coordinator가 새 Consumer 참여를 감지하면 전체 파티션 할당을 재조정합니다.

예를 들어 파티션 3개, Consumer 3개(각 1:1 할당) 상태에서 Consumer 4번째가 참여하면:
1. 기존 Consumer 3개 모두 파티션 처리를 일시 중단
2. Group Coordinator가 새 할당 계획 수립
3. Consumer 1,2,3,4에 재배분 → Consumer 4는 어떤 파티션도 받지 못함(파티션 3개이므로)
4. 재배분 완료 후 처리 재개

이 과정에서 잠깐의 중단(Stop-The-World)이 발생합니다. 이것이 Chapter 4에서 다루는 리밸런싱 문제입니다. Cooperative Rebalancing을 사용하면 이 중단을 최소화할 수 있습니다.

</details>

---

**Q3. Sticky Partitioner가 Round-Robin보다 처리량이 높은 이유는 무엇인가요?**

<details>
<summary>해설 보기</summary>

배치 효율성 차이입니다.

Round-Robin은 메시지를 하나씩 파티션에 돌아가며 배정합니다. 메시지 10개를 파티션 3개에 보내면:
- Partition 0 배치: [msg1, msg4, msg7, msg10] (4개)
- Partition 1 배치: [msg2, msg5, msg8] (3개)
- Partition 2 배치: [msg3, msg6, msg9] (3개)

Sticky Partitioner는 배치가 `batch.size`에 도달하거나 `linger.ms`가 만료될 때까지 같은 파티션으로 메시지를 모읍니다. 10개 모두 Partition 0에 쌓이면 한 번에 전송하고, 그 다음 Partition 1으로 전환합니다:
- Partition 0 배치: [msg1~msg8] (8개, batch.size 도달)
- Partition 1 배치: [msg9, msg10] (2개, linger.ms 만료)

Sticky는 배치가 꽉 찬 상태로 전송되므로 브로커 요청 횟수가 줄고, 압축 효율도 높습니다. 결과적으로 동일한 시간에 더 많은 메시지를 처리할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Kafka 설계 철학](./01-kafka-design-philosophy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 브로커 내부 구조 ➡️](./03-broker-log-segment.md)**

</div>
