# ISR(In-Sync Replicas) — 조건과 이탈/복귀

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ISR 멤버가 되려면 어떤 조건을 충족해야 하는가?
- `replica.lag.time.max.ms`가 ISR 이탈을 결정하는 정확한 기준은?
- Follower가 ISR에서 이탈하면 `acks=all` Producer는 어떻게 동작하는가?
- ISR에서 이탈한 Follower가 다시 복귀하려면 무슨 과정을 거치는가?
- ISR 목록은 어디에 저장되고 어떻게 변경되는가?
- ISR 이탈을 예방하는 운영 전략은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`acks=all`의 실제 의미는 "모든 복제본이 받았다"가 아니라 "**현재 ISR의 모든 멤버**가 받았다"이다. ISR이 1개로 줄어있으면 `acks=all`이어도 브로커 1대만 기록하면 성공 응답이 나온다.

이것을 모르면:
- "`acks=all` 설정했으니 절대 유실 없다"고 안심 → ISR=1 상태에서 브로커 장애 시 유실
- ISR 이탈 알람을 무시하다가 운영 장애 발생
- `min.insync.replicas` 설정 없이 `acks=all`만으로 충분하다고 오해

ISR 동작을 이해하면 `acks=all`과 `min.insync.replicas`를 함께 설정해야 하는 이유가 명확해진다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: acks=all = "항상 3개 브로커에 저장"이라는 오해

  설정: replication.factor=3, acks=all
  상황: Broker 2, 3이 동시에 느려져서 ISR에서 이탈 → ISR=[1]
  결과: acks=all인데 Broker 1에만 기록하면 성공 응답!
  
  이유: acks=all은 "현재 ISR 전체에 기록" 을 의미
        ISR이 1개면 1개에만 기록해도 완료
  
  해결: min.insync.replicas=2 설정
        → ISR이 2 미만이면 쓰기 자체를 거부 (NotEnoughReplicasException)

실수 2: ISR 이탈 원인을 "Follower 장애"로만 생각

  이탈 원인은 다양함:
    - Follower 브로커 장애 (완전한 다운)
    - Follower 브로커 GC STW (replica.lag.time.max.ms 동안 응답 없음)
    - 네트워크 파티션 (Follower → Leader 연결 일시 차단)
    - 디스크 I/O 병목 (Follower가 Fetch한 데이터를 디스크에 못 씀)
    - Follower 브로커 과부하 (CPU/메모리 포화)

  결과:
    "ISR 이탈 = 브로커 다운"으로만 생각 → GC나 네트워크 문제를 놓침
    실제로는 "30초간 Fetch 없음 또는 LEO 추격 실패"가 기준

실수 3: ISR 복귀를 즉시 완료로 가정

  상황: Broker 3이 장애 후 복구 → ISR에 바로 들어온다고 가정
  실제:
    1. Broker 3 재시작 → Leader에게 Fetch 시작
    2. Leader의 LEO까지 따라잡아야 ISR 복귀 가능
    3. 1 GB 뒤처진 경우 → 1 GB 복제 완료 후 ISR 복귀
    4. 복귀 중에는 ISR에서 제외된 상태 → acks=all은 2개만으로 처리
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
ISR 이탈 즉시 감지:
  JMX 알람 설정:
    kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions != 0
    → ISR 이탈 발생 시 즉시 PagerDuty/Slack 알람

  kafka-topics 주기적 확인:
    kafka-topics --bootstrap-server localhost:9092 \
      --describe --under-replicated-partitions
    # 출력이 있으면 ISR 이탈 파티션 목록

ISR 이탈 예방 설정:
  # 브로커 설정 (server.properties)
  replica.lag.time.max.ms=30000   # 30초 (기본값 유지 or 증가)
  num.replica.fetchers=4           # 고처리량 클러스터에서 복제 속도 향상
  
  # JVM 설정으로 GC STW 최소화
  KAFKA_HEAP_OPTS="-Xmx6g -Xms6g -XX:+UseG1GC"
  # G1GC로 GC STW를 100ms 이하로 유지
  # replica.lag.time.max.ms(30초) 대비 GC STW는 무시 가능 수준

ISR 상태 해석:
  Replicas: 1,2,3  Isr: 1,2,3  → 완전 정상
  Replicas: 1,2,3  Isr: 1,2    → Broker 3이 ISR 이탈 (조사 필요)
  Replicas: 1,2,3  Isr: 1      → 위험! min.insync.replicas=2면 쓰기 중단
```

---

## 🔬 내부 동작 원리

### 1. ISR 멤버 조건과 이탈 기준

```
ISR 조건: "Leader의 LEO와 충분히 가까운 Follower"

구체적 기준 (replica.lag.time.max.ms=30000):
  [조건 1] 마지막 Fetch 요청이 30초 이내
           Follower가 30초 이상 Fetch 요청을 안 보내면 이탈
           (브로커 다운, 네트워크 차단 등)

  [조건 2] Fetch 요청을 보내고 있지만 LEO 추격이 안 될 때
           → 이 경우도 replica.lag.time.max.ms 기준
           Leader LEO - Follower LEO > 일정 이상이 30초 지속되면 이탈

  ISR 이탈 판단 주체: Leader 브로커
    Leader는 각 Follower의 마지막 Fetch 시각과 LEO를 추적
    주기적으로 기준 초과 여부 검사 → 이탈 처리

  ISR 변경 기록:
    ZooKeeper 모드: /kafka/brokers/topics/{topic}/partitions/{n}/state 업데이트
    KRaft 모드:    __cluster_metadata 토픽에 IsrChangeRecord 추가
```

### 2. ISR 이탈 흐름

```
정상 상태:
  ISR = [1, 2, 3]
  Follower 2가 마지막 Fetch: t=0s
  Follower 2의 LEO = Leader LEO = 1000

  t=5s: Follower 2 GC STW 발생 (Fetch 중단)
  t=10s: Follower 2 GC 종료, Fetch 재개 → LEO 추격 시작
         Leader LEO = 1500 (5초간 500개 메시지 추가)
  t=10s: Follower 2 LEO = 1000 (500개 뒤처짐)
  t=20s: Follower 2 복구 완료 → LEO = 1500 (Leader 따라잡음)
  → ISR 이탈 없음 (30초 이내에 회복)

  t=5s: Follower 3 디스크 오류로 완전 중단
  t=35s: 마지막 Fetch로부터 30초 경과
  → Leader가 Follower 3을 ISR에서 제외
  → ISR = [1, 2]
  → ZooKeeper/KRaft에 ISR 변경 기록
  → Controller가 변경 사항을 다른 브로커에 전파
```

### 3. ISR 이탈이 acks=all에 미치는 영향

```
시나리오: replication.factor=3, acks=all, ISR=[1,2,3] → ISR=[1]

  ISR=[1,2,3] 상태:
    Producer → Leader(1) 쓰기
    Leader: Follower 2, 3의 Fetch 응답 기다림
    Follower 2, 3 복제 완료
    → Producer에게 성공 응답 (3개 모두 저장 확인)

  ISR=[1] 상태 (Follower 2, 3 이탈):
    Producer → Leader(1) 쓰기
    Leader: ISR은 자신(1)만 → 즉시 성공 응답 가능
    → Producer에게 성공 응답 (1개만 저장)
    → Broker 1 장애 시 데이터 유실!

  min.insync.replicas=2 설정 시:
    ISR=[1] 상태에서 쓰기 요청
    → Leader: ISR 크기(1) < min.insync.replicas(2)
    → NotEnoughReplicasException 발생
    → Producer 쓰기 실패 (데이터 유실보다 명시적 실패가 낫다)
```

### 4. ISR 복귀 과정

```
Follower 복귀 조건: Leader의 LEO까지 따라잡기

  Follower 재시작 후:
    1. Leader에게 FetchRequest 전송
       (자신의 현재 LEO offset부터 요청)
    2. Leader는 FetchResponse로 이후 데이터 전송
    3. Follower가 순차적으로 복제
    4. Follower LEO == Leader LEO 도달
    5. Leader가 Follower를 ISR에 추가
    6. ISR 변경 ZooKeeper/KRaft에 기록

  복귀 시간 계산:
    뒤처진 데이터 = Leader LEO - Follower LEO
    복제 속도 = num.replica.fetchers × replica.fetch.max.bytes × 주기
    예: 뒤처진 데이터 1 GB, 복제 속도 100 MB/s → 10초 후 ISR 복귀

  복귀 중 동작:
    ISR에서 제외된 상태로 복제 중
    acks=all은 기존 ISR([1,2])만으로 처리
    Follower가 ISR 복귀하면 acks=all에 포함됨
```

### 5. ISR 변경이 미치는 영향 체인

```
ISR 이탈 발생 시 영향:
  1. ISR 크기 감소
     → acks=all 성공 조건 완화 (이탈한 Follower 기다릴 필요 없음)
     → Producer 응답 지연 감소 (빠른 응답이 가능하지만 내구성 약화)

  2. min.insync.replicas 미달 시
     → NotEnoughReplicasException으로 쓰기 거부
     → 가용성 < 내구성 우선 설정

  3. Leader 장애 시 선출 후보 축소
     → ISR=[1]이면 Broker 1만 후보
     → unclean.leader.election.enable=false이면 ISR 멤버만 후보
     → ISR 외 브로커는 데이터가 오래됐으므로 선출 불가 (데이터 유실 방지)
```

---

## 💻 실전 실험

### 실험 1: ISR 이탈 실시간 모니터링

```bash
# 복제 팩터 3 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic isr-test \
  --partitions 1 --replication-factor 3

# ISR 상태 실시간 감시 (5초마다)
watch -n 5 'kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic isr-test'

# 별도 터미널: Broker 2 중단
docker stop kafka-2

# 30초 후 출력 변화 확인:
# 정상: Isr: 1,2,3
# 이탈: Isr: 1,3  ← Broker 2 이탈
```

### 실험 2: replica.lag.time.max.ms 단축으로 빠른 이탈 테스트

```bash
# replica.lag.time.max.ms를 10초로 단축
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type brokers \
  --entity-name 1 \
  --alter \
  --add-config replica.lag.time.max.ms=10000

# Broker 3 중단
docker stop kafka-3

# 10초 후 ISR 이탈 확인
sleep 12
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic isr-test

# 원복
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type brokers --entity-name 1 \
  --alter --delete-config replica.lag.time.max.ms
```

### 실험 3: ISR 축소 상태에서 acks=all 쓰기 확인

```bash
# Broker 2, 3 모두 중단 → ISR=[1]
docker stop kafka-2 kafka-3
sleep 35  # lag 시간 대기

# min.insync.replicas=1 (기본값) → 쓰기 성공
kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic isr-test \
  --producer-property acks=all
# 메시지 입력 → 성공 (ISR=1이지만 min.insync.replicas=1이므로 허용)

# min.insync.replicas=2 설정 → 쓰기 실패 확인
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics --entity-name isr-test \
  --alter --add-config min.insync.replicas=2

kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic isr-test \
  --producer-property acks=all
# 메시지 입력 → ERROR: NotEnoughReplicasException
```

---

## 📊 성능/비용 비교

### ISR 크기에 따른 acks=all 처리량

```
replication.factor=3, 1 KB 메시지:

  ISR=[1,2,3] acks=all:
    쓰기: Leader 기록 + ISR 2개 복제 완료 대기
    추가 지연: ~5~15 ms (네트워크 왕복 + Follower 기록)
    처리량: ~100,000 msg/sec

  ISR=[1,2] acks=all:
    쓰기: Leader 기록 + ISR 1개(Follower 2) 복제 완료 대기
    추가 지연: ~3~8 ms (Follower 1개만 기다림)
    처리량: ~130,000 msg/sec (약간 증가)

  ISR=[1] acks=all:
    쓰기: Leader 기록만으로 완료
    추가 지연: ~0 ms
    처리량: ~200,000 msg/sec (최고이지만 내구성 최저)

  → ISR 이탈이 단기적으로 처리량을 높이는 것처럼 보이지만
    내구성을 심각하게 약화시키는 위험 신호
```

### replica.lag.time.max.ms 설정별 트레이드오프

```
짧게 (예: 5초):
  ✅ ISR 이탈 빠름 → acks=all이 느린 Follower를 오래 기다리지 않음
  ❌ 일시적 GC/네트워크 지연으로도 ISR 이탈 빈발
     → ISR 변동이 잦아져 안정성 저하

길게 (예: 60초):
  ✅ 일시적 지연에도 ISR 유지 → 안정적 복제 상태
  ❌ 느린 Follower가 ISR에 남아 acks=all 지연 증가
  ❌ ISR 이탈 감지가 느려 진짜 장애도 늦게 인지

권장: 30초 (기본값)
  일반적인 GC STW(100ms 이내)와 네트워크 지연은 허용
  실제 장애는 30초 이내에 이탈
```

---

## ⚖️ 트레이드오프

```
ISR 크게 유지 (빠른 복제):
  ✅ 더 많은 브로커가 데이터를 안전하게 보유
  ✅ Leader 장애 시 최신 데이터를 가진 Follower가 많음
  ❌ acks=all 지연 증가 (더 많은 Follower 복제 대기)

ISR 작게 허용 (ISR 이탈 관대):
  ✅ acks=all 지연 감소 (기다릴 Follower 적음)
  ❌ 데이터 내구성 저하
  ❌ Leader 장애 시 최신 Follower 없을 수 있음

min.insync.replicas 설정:
  높게 (=replication.factor): ✅ 최강 내구성 ❌ ISR 이탈 시 쓰기 중단
  낮게 (=1):                  ✅ 항상 쓰기 가능 ❌ ISR=1일 때 데이터 유실 위험
  권장 (=replication.factor-1): 균형 (브로커 1대 장애에도 쓰기 가능)
```

---

## 📌 핵심 정리

```
ISR 핵심:

1. ISR = "현재 Leader LEO에 충분히 가까운 Follower 집합"
   기준: replica.lag.time.max.ms(기본 30초) 내에 Fetch 있고 LEO 추격 중

2. acks=all = "현재 ISR 전체에 기록 완료"
   ISR 크기가 줄면 acks=all의 내구성도 함께 약화

3. ISR 이탈 원인 다양
   브로커 다운 / GC STW / 네트워크 / 디스크 I/O 병목
   → 단순히 "브로커 죽음"이 아니라 복합 원인

4. ISR 복귀 = LEO 따라잡기
   뒤처진 데이터를 모두 복제 완료 후 ISR에 재합류

5. min.insync.replicas와 함께 설정 필수
   acks=all만으론 부족 → ISR=1일 때도 쓰기 성공하기 때문
   운영 표준: replication.factor=3, min.insync.replicas=2, acks=all
```

---

## 🤔 생각해볼 문제

**Q1. Follower 브로커가 정상 동작 중인데도 ISR에서 이탈할 수 있나요?**

<details>
<summary>해설 보기</summary>

네, 가능합니다. ISR 이탈의 기준은 "Leader LEO를 replica.lag.time.max.ms 내에 따라잡는가"이지 "브로커가 살아있는가"가 아닙니다.

다음 상황에서 브로커가 정상이어도 ISR 이탈이 발생합니다:
- **GC STW**: JVM GC가 30초 이상 지속되면(비정상적이지만 발생 가능) Fetch 중단 → 이탈
- **처리량 급증**: Producer가 갑자기 메시지를 폭발적으로 보내서 Follower 복제 속도가 따라가지 못할 때
- **디스크 I/O 경합**: 동일 브로커의 다른 파티션이 디스크를 과점유해서 Follower의 log 쓰기가 지연될 때
- **네트워크 혼잡**: Follower → Leader 구간 네트워크가 일시적으로 포화될 때

이런 경우 브로커 자체는 살아있지만 ISR에서 이탈합니다. 따라서 ISR 모니터링은 단순 브로커 생존 모니터링과 별개로 반드시 필요합니다.

</details>

---

**Q2. ISR이 [1]만 남은 상태에서 Broker 1이 장애나면 어떤 일이 발생하나요?**

<details>
<summary>해설 보기</summary>

`unclean.leader.election.enable` 설정에 따라 달라집니다.

**false(기본값)**: ISR 외의 Follower는 Leader 선출 대상이 아닙니다. ISR에 남은 브로커가 없으면 파티션이 Offline 상태가 됩니다. 쓰기와 읽기 모두 불가. Broker 1이 복구될 때까지 해당 파티션은 사용 불가입니다.

**true**: ISR에 없는 오래된 Follower도 새 Leader 후보가 됩니다. 서비스는 계속되지만 해당 Follower에 없는 메시지(Broker 1에만 있던 최신 데이터)는 유실됩니다.

이것이 ISR 이탈을 방치하면 안 되는 이유입니다. ISR=[1]인 상태가 방치되면 Broker 1 장애 시 파티션 Offline(가용성 0) 또는 데이터 유실 중 하나를 선택해야 하는 상황이 됩니다.

</details>

---

**Q3. `replica.lag.time.max.ms=30000`인데 왜 Kafka Streams나 Spring Kafka에서는 세션 타임아웃과 혼동하지 않아야 하나요?**

<details>
<summary>해설 보기</summary>

이름이 비슷해 보이지만 완전히 다른 레이어의 설정입니다.

`replica.lag.time.max.ms`는 **브로커-브로커 간** 복제 지연에 관한 설정입니다. Follower 브로커가 Leader 브로커에게 Fetch 요청을 보내는 주기와 관련됩니다.

`session.timeout.ms`는 **Consumer-브로커 간** heartbeat에 관한 설정입니다. Consumer 인스턴스가 Group Coordinator에게 heartbeat를 보내지 않으면 그 Consumer를 그룹에서 제거하는 타임아웃입니다.

`max.poll.interval.ms`는 **Consumer 처리 시간**에 관한 설정으로, poll() 호출 간격이 이 값을 초과하면 Consumer가 그룹에서 제거됩니다.

세 설정이 모두 "시간 초과 시 제거"라는 패턴을 가지지만, 각각 복제 동기화, Consumer heartbeat, Consumer 처리 시간을 관리하는 독립적인 메커니즘입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 파티션 복제](./01-partition-replication.md)** | **[홈으로 🏠](../README.md)** | **[다음: acks 설정 완전 분해 ➡️](./03-acks-deep-dive.md)**

</div>
