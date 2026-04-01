# 리밸런싱 발생 조건 — heartbeat와 poll 간격

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `heartbeat.interval.ms`와 `session.timeout.ms`는 어떤 관계이고, 각각 무엇을 측정하는가?
- `max.poll.interval.ms` 초과가 리밸런싱을 유발하는 구체적인 과정은?
- Consumer 추가/제거 외에 리밸런싱을 유발하는 숨겨진 원인들은?
- 리밸런싱을 최소화하는 설정 전략은?
- 리밸런싱 빈도를 모니터링하는 방법은?
- JVM GC가 리밸런싱을 유발하는 시나리오는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

리밸런싱이 발생하면 Consumer Group의 모든 처리가 멈춘다(Eager 방식). 리밸런싱 빈도가 높으면 실제 메시지 처리 시간보다 리밸런싱 중단 시간이 더 길어질 수 있다.

"Consumer를 추가했을 뿐인데 왜 이미 처리 중이던 메시지가 중단되고 다시 처리되는가?" 이 질문의 답은 리밸런싱 발생 조건과 Eager 방식의 Stop-The-World에 있다.

리밸런싱 원인을 정확히 파악하면:
- 불필요한 리밸런싱(GC STW, 느린 처리 로직 등)을 사전에 예방
- 필요한 리밸런싱(Consumer 추가)의 영향을 최소화
- 리밸런싱 로그를 보고 원인을 즉시 진단

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: heartbeat.interval.ms = session.timeout.ms라고 오해

  혼동:
    "session.timeout.ms=30초니까 30초마다 heartbeat 보내면 되겠지"

  현실:
    heartbeat는 session.timeout.ms보다 훨씬 자주 보내야 함
    권장: heartbeat.interval.ms <= session.timeout.ms / 3
    session.timeout.ms=30초 → heartbeat.interval.ms <= 10초

  이유:
    heartbeat 전송 중 네트워크 지연, 패킷 손실 가능
    여러 번 heartbeat를 보내는 동안 1번 실패해도 안전하려면
    타임아웃 전에 여러 번 전송 기회 필요

실수 2: 느린 처리 로직 = DB 문제로만 진단

  증상: 리밸런싱이 주기적으로 발생, 처리 중단
  잘못된 진단: "DB가 느리다" → DBA에게 문의
  실제 원인:
    max.poll.records=500개를 받았는데
    각 레코드당 외부 API 호출 200ms → 500 × 200ms = 100초
    max.poll.interval.ms=300초(5분) → 100초는 OK?
    → 그런데 외부 API 지연이 갑자기 500ms로 늘어나면
      500 × 500ms = 250초 → 5분 이내이지만
      API 지연이 더 늘면 300초 초과 → 리밸런싱!
  
  해결:
    max.poll.records를 줄여서 poll() 간격 보장
    또는 max.poll.interval.ms를 처리 로직 최대 시간보다 크게 설정

실수 3: GC STW = 리밸런싱과 무관하다는 오해

  시나리오:
    Consumer JVM에서 Full GC 발생 → 15초 STW
    HeartbeatThread도 GC 동안 정지 → heartbeat 전송 못 함
    session.timeout.ms=10초 < GC 15초
    → GC 중 세션 타임아웃 → 리밸런싱 발생!

  해결:
    G1GC 사용 → STW 최소화 (목표: 200ms 이하)
    session.timeout.ms를 GC STW 예상 최대값보다 크게 설정
    KAFKA_HEAP_OPTS="-Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
안전한 타임아웃 설정:
  # 일반 서비스
  session.timeout.ms=30000          (30초)
  heartbeat.interval.ms=10000       (10초, session의 1/3)
  max.poll.interval.ms=300000       (5분)
  max.poll.records=500              (처리 시간 × 500 < 5분)

  # GC STW가 길거나 처리 로직이 느린 경우
  session.timeout.ms=60000          (1분)
  heartbeat.interval.ms=20000       (20초)
  max.poll.interval.ms=600000       (10분)
  max.poll.records=100              (적게 받아서 빠르게 처리)

처리 시간 계산:
  max.poll.records × 레코드당 최대 처리 시간 < max.poll.interval.ms
  예: 레코드당 최대 500ms, max.poll.records=200
      200 × 500ms = 100,000ms = 100초 << 300,000ms(5분) → 안전

리밸런싱 빈도 모니터링:
  # Consumer 로그에서 리밸런싱 감지
  grep "Rebalance" consumer.log | wc -l

  # JMX 지표
  kafka.consumer:type=consumer-coordinator-metrics,client-id=*
  지표: rebalance-rate (초당 리밸런싱 횟수)
       last-rebalance-seconds-ago (마지막 리밸런싱 경과 시간)
```

---

## 🔬 내부 동작 원리

### 1. session.timeout.ms vs heartbeat.interval.ms vs max.poll.interval.ms

```
세 설정의 역할 분리:

  heartbeat.interval.ms=10000 (10초):
    HeartbeatThread가 10초마다 GC에게 heartbeat 전송
    "나 아직 살아있음"
    → poll() 호출과 무관하게 별도 스레드에서 동작

  session.timeout.ms=30000 (30초):
    GC가 heartbeat 없이 30초가 지나면 해당 Consumer 장애 판단
    → Consumer를 그룹에서 제거 → 리밸런싱
    → 검사 대상: HeartbeatThread의 heartbeat 주기

  max.poll.interval.ms=300000 (5분):
    Consumer 라이브러리가 내부에서 poll() 간격을 측정
    5분 동안 poll() 없으면 Consumer가 직접 GC에게 LeaveGroup 전송
    → Consumer 스스로 그룹 이탈 → 리밸런싱
    → 검사 대상: 애플리케이션의 poll() 호출 주기

  타이머 동작 그림:
    ─────────────────────────────────────────────────────────►
    poll()       HeartbeatThread  poll()
     │                │     │     │
     t=0             t=10  t=20  t=25
     ▲               ▲     ▲     ▲
     처리 시작        HB    HB    처리 완료 + 다음 poll()
    
    t=0~25초 처리 중 → HeartbeatThread는 t=10, t=20에 heartbeat 전송
    → session.timeout.ms(30초) 내에 heartbeat 있음 → 세션 유지
    → max.poll.interval.ms(5분) 내에 poll() 있음 → LeaveGroup 없음
    → 리밸런싱 없음 ✅
```

### 2. 리밸런싱 유발 원인 분류

```
[명시적 원인]
  1. Consumer 추가
     새 인스턴스가 JoinGroup 요청 → GC: PreparingRebalance 전이

  2. Consumer 정상 종료
     consumer.close() → LeaveGroup 요청 → GC: 리밸런싱

  3. 구독 토픽 변경
     consumer.subscribe(newTopics) → GC: 리밸런싱

[암묵적 원인 (주의)]
  4. max.poll.interval.ms 초과
     처리 로직이 너무 오래 걸림
     → Consumer가 LeaveGroup 전송 → 리밸런싱

  5. session.timeout.ms 내 heartbeat 부재
     JVM GC STW / 네트워크 단절
     → GC가 Consumer 장애로 판단 → 리밸런싱

  6. 토픽 파티션 수 변경
     kafka-topics --alter --partitions 증가 시
     → GC: 파티션 재할당 필요 → 리밸런싱

  7. 구독 토픽의 새 파티션 생성
     Pattern 구독(subscribe(Pattern)) + 새 토픽 생성
     → GC: 메타데이터 변경 감지 → 리밸런싱

  8. rebalance.timeout.ms 내 일부 Consumer 응답 없음
     JoinGroup 대기 시간 내 응답 없는 Consumer 제외 후 진행
     → 제외된 Consumer는 다음 cycle에 재참여 → 또 리밸런싱
```

### 3. GC STW로 인한 리밸런싱 시나리오

```
Consumer JVM 설정: -Xmx8g -XX:+UseParallelGC (위험!)

  t=0:    Consumer C1, C2, C3 Stable 상태
  t=60:   C1 JVM에서 Full GC 발생 (ParallelGC = STW)
          - HeartbeatThread 정지 (GC STW 동안 모든 스레드 정지)
          - heartbeat 전송 중단
  t=70:   session.timeout.ms=10초 초과
          GC(Group Coordinator): C1 세션 타임아웃 → 제거
          GC: PreparingRebalance 전이
  t=75:   C1 Full GC 종료 (15초 STW)
  t=76:   C1 재연결 시도
          GC: 이미 PreparingRebalance 중 → C1 JoinGroup 받음
  t=80:   리밸런싱 완료 → Stable (C1 포함)
  결과:   15초 GC로 인해 전체 그룹 ~10초 처리 중단

  G1GC 사용 시:
    -XX:+UseG1GC -XX:MaxGCPauseMillis=200
    GC STW 목표: 200ms 이하
    session.timeout.ms=10000(10초) >> GC STW 200ms
    → GC 중에도 세션 유지 → 리밸런싱 없음 ✅
```

### 4. max.poll.interval.ms 초과 상세

```
Consumer 코드:
  while (true) {
    ConsumerRecords records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord r : records) {
      callSlowAPI(r);  // 레코드당 최대 1초
    }
    // poll() 재호출
  }

  max.poll.records=500, callSlowAPI=1초/레코드
  최악: 500초 처리 > max.poll.interval.ms=300초(5분)

  초과 감지 메커니즘:
    poll() 내부에서 이전 poll() 호출 시각과 현재 시각 비교
    초과 시: consumer 스스로 GC에게 LeaveGroup 요청
    poll() 호출 위치에서 CommitFailedException 또는 WakeupException 발생

  로그 패턴:
    WARN  Offset commit cannot be completed since the consumer is not
          part of an active group for auto partition assignment.
    ERROR KafkaConsumer is not currently assigned any partitions
```

---

## 💻 실전 실험

### 실험 1: session.timeout.ms 초과 리밸런싱 재현

```bash
# Consumer 시작 (session.timeout.ms=5초, 짧게 설정)
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic group-test \
  --group timeout-test \
  --consumer-property session.timeout.ms=5000 \
  --consumer-property heartbeat.interval.ms=1500

# Consumer 프로세스 일시 중단 (SIGSTOP)
kill -STOP $(pgrep -f "kafka.tools.ConsoleConsumer")

# 5초 후 리밸런싱 로그 확인
sleep 7
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group timeout-test
# State: PreparingRebalance 또는 Empty 확인

# Consumer 재개
kill -CONT $(pgrep -f "kafka.tools.ConsoleConsumer")
```

### 실험 2: max.poll.interval.ms 초과 시뮬레이션

```java
// max.poll.interval.ms=10초로 짧게 설정
Properties props = new Properties();
props.put("max.poll.interval.ms", "10000");  // 10초
props.put("max.poll.records", "1");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("group-test"));

while (true) {
    ConsumerRecords<K,V> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<K,V> record : records) {
        Thread.sleep(15000);  // 15초 대기 → max.poll.interval.ms(10초) 초과!
    }
    // 다음 poll()은 10초 후 → CommitFailedException 또는 리밸런싱
}
```

### 실험 3: 리밸런싱 빈도 JMX 모니터링

```bash
# jmxterm으로 rebalance-rate 확인
java -jar jmxterm.jar
open kafka-1:9999
bean kafka.consumer:type=consumer-coordinator-metrics,client-id=*
get rebalance-rate
get last-rebalance-seconds-ago

# 또는 kafka-consumer-groups로 주기적 확인
watch -n 5 'kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group timeout-test | grep -E "State|Lag"'
```

---

## 📊 성능/비용 비교

### 타임아웃 설정별 리밸런싱 빈도 영향

```
시나리오: 1분마다 GC STW 발생 (1~3초)

  session.timeout.ms=5000 (5초):
    GC 3초 STW → 세션 초과 → 리밸런싱
    리밸런싱 빈도: ~1회/분
    처리 중단: ~2~5초 × 1회/분 = 분당 2~5초 중단

  session.timeout.ms=30000 (30초):
    GC 3초 STW << 30초 → 세션 유지
    리밸런싱 없음
    처리 중단: 0초

결론:
  session.timeout.ms를 GC STW 예상 최대값의 3배 이상으로 설정
  G1GC 사용 시 STW < 200ms → session.timeout.ms=10초(기본값)도 충분
  ParallelGC(구버전) 사용 시 STW가 수 초 → session.timeout.ms 늘리거나 GC 교체
```

---

## ⚖️ 트레이드오프

```
session.timeout.ms 짧게:
  ✅ 장애 Consumer 빠르게 감지 → 빠른 파티션 재할당
  ❌ 일시적 GC/네트워크 지연으로 불필요한 리밸런싱 빈발

session.timeout.ms 길게:
  ✅ 일시적 지연에도 세션 유지 → 리밸런싱 안정
  ❌ 실제 장애 감지 늦음 → 해당 파티션 처리 지연

max.poll.interval.ms 길게:
  ✅ 느린 처리 로직에도 안전
  ❌ 진짜 처리 멈춤(무한루프 등)도 늦게 감지

max.poll.records 적게:
  ✅ poll() 간격 단축 → max.poll.interval.ms 초과 위험 감소
  ❌ 처리량 감소 (Fetch 요청 빈도 증가)

권장 균형점:
  G1GC + session.timeout.ms=30초 + max.poll.interval.ms=5분
  max.poll.records = 처리 시간 기반 계산
```

---

## 📌 핵심 정리

```
리밸런싱 발생 조건 핵심:

1. session.timeout.ms: heartbeat 없음 감지 → GC가 Consumer 제거
   원인: Consumer 장애 / GC STW / 네트워크 단절

2. max.poll.interval.ms: poll() 간격 초과 → Consumer가 스스로 LeaveGroup
   원인: 처리 로직이 너무 오래 걸림

3. 명시적 원인: Consumer 추가/제거 / 파티션 수 변경 / 구독 변경

4. 숨겨진 원인: GC STW / 느린 처리 / 토픽 파티션 증가

5. 최소화 전략:
   G1GC 사용 (STW 최소화)
   session.timeout.ms >= GC STW 예상 최대값 × 3
   max.poll.records × 레코드당 처리 시간 << max.poll.interval.ms

6. 모니터링:
   JMX rebalance-rate, last-rebalance-seconds-ago
   Consumer 로그의 "Rebalance" 키워드
```

---

## 🤔 생각해볼 문제

**Q1. `heartbeat.interval.ms=3초, session.timeout.ms=3초`로 설정하면 어떤 문제가 발생하나요?**

<details>
<summary>해설 보기</summary>

heartbeat 전송 주기와 타임아웃이 같으면 단 1번의 heartbeat 실패(네트워크 지연, 패킷 손실 등)로 세션 타임아웃이 발생합니다. 정상 운영 중에도 일시적인 네트워크 지터(jitter)로 heartbeat가 3초를 살짝 넘기면 즉시 리밸런싱이 발생합니다.

Kafka는 `heartbeat.interval.ms`가 `session.timeout.ms`의 1/3을 초과하면 경고 로그를 남깁니다. 권장 비율은 `heartbeat.interval.ms <= session.timeout.ms / 3`입니다. 예를 들어 `session.timeout.ms=30초`이면 `heartbeat.interval.ms=10초` 이하로 설정합니다.

</details>

---

**Q2. Consumer가 메시지 처리 중 외부 API 호출이 가끔 30초씩 걸립니다. `max.poll.interval.ms`를 얼마로 설정해야 할까요?**

<details>
<summary>해설 보기</summary>

단순히 `max.poll.interval.ms`를 늘리는 것보다 `max.poll.records`를 줄이는 것이 더 안전합니다.

외부 API가 최대 30초 걸리고 `max.poll.records=100`이면 최악 100 × 30초 = 3,000초가 필요합니다. `max.poll.interval.ms=3600000`(1시간)으로 설정해도 이론적으로는 안전하지만, 진짜 Consumer 장애(무한루프 등)를 1시간 동안 감지하지 못하는 문제가 생깁니다.

더 나은 접근: `max.poll.records=1`로 줄이고, 각 레코드를 처리하는 시간이 `max.poll.interval.ms` 내에 완료되도록 합니다. 또는 외부 API 호출에 타임아웃(예: 5초)을 설정하고, 타임아웃 시 DLT로 보내는 방식으로 처리 시간을 예측 가능하게 만드는 것이 근본 해결책입니다.

</details>

---

**Q3. 리밸런싱이 계속 반복되는 "리밸런싱 루프"가 발생하면 어떻게 진단하나요?**

<details>
<summary>해설 보기</summary>

Consumer 로그에서 패턴을 찾는 것이 첫 번째 단계입니다.

**진단 체크리스트**:
1. `max.poll.interval.ms` 초과: 로그에 `poll() interval exceeded timeout` 메시지 확인. `max.poll.records`를 줄이거나 처리 로직 최적화
2. `session.timeout.ms` 초과: 로그에 `Consumer session timed out`. GC 로그 확인(Full GC 시간), `session.timeout.ms` 증가
3. Consumer 수 > 파티션 수: IDLE Consumer가 있으면 그룹 참여/이탈이 반복될 수 있음
4. 토픽 파티션 수가 자주 변경됨: 파티션 추가 이벤트 확인
5. `rebalance.timeout.ms` 초과: 일부 Consumer가 JoinGroup에 늦게 응답해서 리밸런싱이 재시작됨

JMX `rebalance-rate`가 0이 아닌 값을 지속적으로 보인다면 위 순서대로 점검합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Consumer Group 내부 동작](./01-consumer-group-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: 리밸런싱 전략 비교 ➡️](./03-rebalancing-strategy.md)**

</div>
