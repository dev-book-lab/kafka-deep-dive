# Consumer Lag — 원인 분석과 처리 속도 향상

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Consumer Lag이란 정확히 무엇이고 어떻게 측정하는가?
- Lag이 쌓이는 원인을 체계적으로 진단하는 트리는?
- `kafka-consumer-groups --describe`로 파티션별 Lag을 어떻게 읽는가?
- Consumer 병렬 처리와 배치 처리가 Lag을 줄이는 원리는?
- Lag이 쌓일 때 Consumer를 단순히 늘리면 안 되는 경우는 언제인가?
- Consumer Lag 모니터링과 알람을 어떻게 설정해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Consumer Lag은 Kafka 운영의 핵심 건강 지표다. Lag = 0은 Consumer가 Producer를 따라가고 있다는 뜻이고, Lag이 증가하면 실시간 처리가 지연되고 있다는 경보다.

Lag을 보지 않으면:
- 결제 알림이 5분 지연됐는데 서버가 정상으로 보여 원인을 모름
- Consumer를 아무리 늘려도 파티션 수보다 많으면 의미 없다는 것을 모름
- DB 병목이 Lag 원인인데 Kafka 서버만 들여다 봄

Lag의 원인을 체계적으로 진단하면 적절한 해결책(파티션 추가, Consumer 추가, 처리 로직 최적화, 배치 전환)을 선택할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Lag이 쌓이면 Consumer 수만 늘리기

  상황: 파티션 3개, Consumer 3개, Lag 증가
  대응: Consumer를 6개로 늘림
  결과:
    Consumer 3개는 파티션 할당 받음 (C1→P0, C2→P1, C3→P2)
    Consumer 나머지 3개는 IDLE (파티션 없음)
    Lag: 변화 없음 (실제 처리 Consumer 수는 그대로 3개)

  올바른 대응:
    파티션 수도 함께 늘리기 (6개 파티션 → 6개 Consumer 활용)
    또는 처리 로직 최적화로 처리 속도 향상

실수 2: Lag = 0이면 모든 메시지가 처리됐다는 오해

  현실:
    Lag = LOG-END-OFFSET - CURRENT-OFFSET
    CURRENT-OFFSET = 마지막으로 커밋된 offset

    Consumer가 메시지를 처리했지만 커밋 안 했으면 Lag이 줄지 않음
    자동 커밋 비활성화 + 수동 커밋 지연 → Lag은 높지만 처리는 됨
    → Lag만으로 "처리 완료 여부" 판단 불가

실수 3: 단일 파티션에 Lag 집중 (핫스팟) 미인지

  kafka-consumer-groups --describe:
  GROUP   TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
  order   orders  0          1000            1000            0
  order   orders  1          500             10000           9500  ← 문제!
  order   orders  2          900             900             0

  파티션 1에 Lag 9500 집중
  원인: 파티션 1의 담당 Consumer가 느림 (장애? GC? 처리 로직?)
  단순히 "전체 Lag" 모니터링하면 파티션별 불균형 놓침
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Lag 진단 첫 단계: 파티션별 상세 확인

  kafka-consumer-groups --bootstrap-server localhost:19092 \
    --describe --group order-group

  출력:
  GROUP       TOPIC   PART  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
  order-group orders  0     1000            1100            100  consumer-1
  order-group orders  1     500             10500           10000  consumer-2  ← 문제!
  order-group orders  2     900             920             20   consumer-3

  진단:
    파티션 1의 consumer-2에 집중 → consumer-2 상태 확인
    전체 Lag vs 파티션별 Lag → 불균형 여부 확인

알람 설정:
  # Prometheus alertmanager 예시
  - alert: KafkaConsumerLagHigh
    expr: kafka_consumer_group_lag > 10000
    for: 5m
    annotations:
      summary: "Consumer Lag 급증"
      description: "Group {{ $labels.group }}, Topic {{ $labels.topic }}"
```

---

## 🔬 내부 동작 원리

### 1. Consumer Lag 정의와 측정

```
Lag 계산:
  Lag = LOG-END-OFFSET - CURRENT-OFFSET

  LOG-END-OFFSET: 해당 파티션에 현재 쓰여진 마지막 offset + 1
                  = Producer가 가장 최근에 발행한 메시지 위치
  CURRENT-OFFSET: 해당 Consumer Group의 마지막 커밋된 offset
                  = 다음에 읽을 위치

  예시:
    파티션 0: LOG-END-OFFSET=1100, CURRENT-OFFSET=1000
    Lag = 100 → 100개 메시지가 처리 대기 중

  Lag 측정 방법:
    kafka-consumer-groups --describe (CLI)
    JMX: kafka.consumer:type=consumer-fetch-manager-metrics
         records-lag, records-lag-avg, records-lag-max
    kafka-exporter + Prometheus + Grafana (운영 환경)
```

### 2. Lag 원인 진단 트리

```
Consumer Lag 증가
    │
    ├─── [파티션 수 < Consumer 수]
    │    증상: 일부 Consumer IDLE, 활성 Consumer 과부하
    │    해결: 파티션 수 증가 (Consumer 수에 맞게)
    │
    ├─── [처리 로직 병목]
    │    증상: Consumer CPU/메모리 정상, 처리 속도 낮음
    │    원인:
    │    │
    │    ├─── DB 왕복 느림
    │    │    해결: 배치 INSERT/UPSERT, 커넥션 풀 최적화
    │    │
    │    ├─── 외부 API 호출 느림
    │    │    해결: 비동기 처리, 타임아웃 설정, 서킷 브레이커
    │    │
    │    └─── 단일 레코드 처리 로직 무거움
    │         해결: 로직 최적화, 배치 처리로 전환
    │
    ├─── [반복 리밸런싱]
    │    증상: Consumer 로그에 Rebalance 빈발
    │    해결: session.timeout.ms 조정, max.poll.records 감소
    │         CooperativeStickyAssignor 전환
    │
    ├─── [Consumer 장애/느린 인스턴스]
    │    증상: 특정 파티션만 Lag 집중
    │    해결: 해당 Consumer 재시작, 장애 원인 제거
    │
    └─── [Producer 처리량 급증]
         증상: 모든 파티션 Lag 동시 증가
         해결: Consumer 수 + 파티션 수 함께 증가
               배치 처리 효율화
```

### 3. Consumer 처리 속도 향상 전략

```
전략 1: 파티션 수 + Consumer 수 증가 (수평 확장)

  현재: 파티션 3개, Consumer 3개 → 처리량 한계
  변경: 파티션 6개, Consumer 6개 → 처리량 2배

  주의:
    파티션 수 변경 시 키 해시 분포 변경 → 순서 보장 영향
    Consumer 추가 시 리밸런싱 발생 (Cooperative로 영향 최소화)

전략 2: 배치 처리 (단일 Consumer 처리량 향상)

  Before: 레코드마다 개별 DB INSERT
    for record in records:
        db.insert(record)  → 레코드당 DB 왕복 1회
    100개 처리 = DB 왕복 100회

  After: 배치 DB INSERT
    orders = [r.toOrder() for r in records]
    db.batchInsert(orders)  → 배치 전체 DB 왕복 1회
    100개 처리 = DB 왕복 1회
    → 처리량 10~100배 향상

전략 3: 비동기 처리 + 스레드 풀

  Consumer 스레드 (poll 전담)
       │
       ├─→ 스레드 풀 (처리 전담)
       │      Worker1 → processRecord(r1)
       │      Worker2 → processRecord(r2)
       │      Worker3 → processRecord(r3)
       │
       └─ poll() 계속 실행 (heartbeat 유지)

  주의:
    offset 커밋은 처리 완료 후 (비동기 완료 추적 필요)
    consumer.pause()/resume()으로 poll 속도 조절
    처리 완료 전 poll이 너무 빨라서 OOM 주의

전략 4: fetch.min.bytes 증가 (Fetch 효율)

  현재: fetch.min.bytes=1 → 1바이트라도 있으면 Fetch
  최적: fetch.min.bytes=65536 (64 KB) → 64 KB 쌓이면 Fetch
  효과: Fetch 요청 횟수 감소 → 브로커 부하 감소 → 처리량 증가
  단점: 메시지 적을 때 fetch.max.wait.ms까지 대기 → 지연 증가
```

### 4. Lag 모니터링 지표

```
핵심 JMX 지표:

  kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*
    records-lag:      현재 파티션의 Lag
    records-lag-avg:  평균 Lag
    records-lag-max:  최대 Lag (가장 뒤처진 파티션)

  kafka.consumer:type=consumer-coordinator-metrics,client-id=*
    rebalance-rate:           초당 리밸런싱 횟수 (0이어야 정상)
    last-rebalance-seconds-ago: 마지막 리밸런싱 경과 시간

  kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*
    fetch-rate:     초당 Fetch 요청 수
    fetch-size-avg: 평균 Fetch 크기 (작으면 fetch.min.bytes 증가 고려)

Grafana 대시보드 필수 패널:
  1. Consumer Group Lag by Partition (파티션별 Lag 시계열)
  2. Consumer Group State (Stable/Rebalancing 상태)
  3. Records Consumed Rate vs Records Produced Rate
  4. Processing Time (레코드당 평균 처리 시간)
```

---

## 💻 실전 실험

### 실험 1: Lag 측정 및 파티션별 분석

```bash
# Lag 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group order-group

# 출력 예시:
# GROUP       TOPIC   PART  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-group orders  0     1000            1100            100
# order-group orders  1     500             10500           10000  ← 집중!
# order-group orders  2     900             920             20

# IDLE Consumer 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group order-group \
  | grep "no current offset"
# IDLE Consumer는 파티션 할당 없음
```

### 실험 2: 배치 처리 전환 효과 측정

```bash
# 단일 처리 Consumer 처리량 측정
kafka-consumer-perf-test \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --messages 100000 \
  --group perf-test

# max.poll.records 조정으로 배치 효과
kafka-consumer-perf-test \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --messages 100000 \
  --group perf-test-batch \
  --consumer-props max.poll.records=1000 \
                   fetch.min.bytes=65536

# 처리량(msg/sec) 비교
```

### 실험 3: Lag 알람 트리거 시뮬레이션

```bash
# Consumer를 멈추고 Lag 증가 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group order-group
# Lag=0 (정상)

# Consumer 중단
pkill -f "kafka.tools.ConsoleConsumer"

# 메시지 계속 발행
for i in $(seq 1 10000); do
  echo "msg-$i"
done | kafka-consumer-groups --bootstrap-server localhost:19092 \
  --topic orders

# Lag 증가 확인
sleep 5
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group order-group
# Lag: 10000으로 증가

# Consumer 재시작 → Lag 감소 추이 확인
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders --group order-group
```

---

## 📊 성능/비용 비교

### 처리 전략별 처리량 비교

```
조건: Consumer 3개, 파티션 3개, DB INSERT, 1 KB 메시지

  단일 처리 (레코드마다 개별 INSERT):
    처리량: ~500 msg/sec (DB 왕복 병목)
    Lag 소진 속도: 낮음

  배치 처리 (100개씩 batchInsert):
    처리량: ~10,000 msg/sec
    Lag 소진 속도: 20배 향상

  비동기 처리 (스레드 풀 10개):
    처리량: ~5,000 msg/sec
    Lag 소진 속도: 10배 향상
    주의: offset 커밋 복잡도 증가

  파티션 + Consumer 수평 확장 (3→12개):
    처리량: ~2,000 msg/sec × 4 = ~8,000 msg/sec
    비용: 서버 4배

결론:
  배치 처리 최적화가 비용 대비 가장 효과적
  수평 확장은 배치 최적화 후 여전히 부족할 때 추가 고려
```

---

## ⚖️ 트레이드오프

```
파티션 + Consumer 수 증가:
  ✅ 처리 병렬도 향상
  ✅ 각 Consumer 부하 감소
  ❌ 리밸런싱 영향 증가 (그룹 크기 커짐)
  ❌ 파티션 수 증가 시 키 해시 분포 변경
  ❌ 파티션 수는 줄일 수 없음

배치 처리:
  ✅ 단일 Consumer 처리량 극대화
  ✅ DB 왕복 감소
  ❌ max.poll.records 증가 → max.poll.interval.ms 초과 위험
  ❌ 실패 시 배치 전체 재처리 (개별 레코드 실패 처리 복잡)

비동기 처리 (스레드 풀):
  ✅ I/O 병목을 CPU로 우회
  ❌ offset 커밋 타이밍 복잡
  ❌ OOM 위험 (처리 속도 < 수신 속도 시 큐 무한 증가)
  ❌ consumer.pause()/resume() 제어 필요

fetch.min.bytes 증가:
  ✅ Fetch 효율 향상
  ❌ 메시지 적을 때 지연 증가
  → 처리량이 중요한 배치 시스템에 적합
  → 실시간 알림 같은 낮은 지연 요구에는 부적합
```

---

## 📌 핵심 정리

```
Consumer Lag 핵심:

1. Lag = LOG-END-OFFSET - CURRENT-OFFSET
   파티션별로 확인 (전체 합계만으로는 불균형 놓침)

2. Lag 원인 진단 트리:
   파티션 수 < Consumer 수 → IDLE Consumer (파티션 늘리기)
   처리 로직 병목 → 배치 처리 / 비동기 처리
   반복 리밸런싱 → 설정 최적화 / Cooperative 전략
   특정 파티션 집중 → 해당 Consumer 장애 확인

3. 핵심 해결 전략:
   1순위: 배치 처리 (DB 왕복 최소화)
   2순위: 파티션 + Consumer 수 증가 (수평 확장)
   3순위: 비동기 처리 (복잡도 높음)

4. Consumer 늘리기 = 파티션 수 확인 먼저
   Consumer > 파티션이면 늘려도 효과 없음

5. 모니터링:
   JMX records-lag-max 알람 설정
   파티션별 Lag 그래프로 핫스팟 감지
   kafka-consumer-groups --describe 주기적 확인
```

---

## 🤔 생각해볼 문제

**Q1. Consumer Lag이 0인데 실시간 처리가 지연되는 것처럼 느껴집니다. 어떤 원인이 가능한가요?**

<details>
<summary>해설 보기</summary>

Lag이 0이라는 것은 Consumer가 Producer를 따라잡았다는 의미입니다. 하지만 다음 원인으로 체감 지연이 발생할 수 있습니다.

**처리 시간 자체의 지연**: Consumer가 메시지를 받아서 처리하는 데 걸리는 시간 자체가 길 수 있습니다. Lag이 0이어도 메시지가 들어온 직후 처리가 완료되지 않으면 체감 지연이 생깁니다.

**end-to-end 지연 측정 필요**: 메시지가 발행된 시각(Producer timestamp)과 Consumer가 처리 완료한 시각의 차이를 측정합니다. `ConsumerRecord.timestamp()`로 발행 시각을 알 수 있습니다.

**Producer → 브로커 지연**: 네트워크 지연으로 브로커가 메시지를 받는 데 지연이 생길 수 있습니다.

**자동 커밋 지연**: `enable.auto.commit=true`이면 처리가 완료됐어도 커밋 주기(`auto.commit.interval.ms`)까지 Lag이 줄지 않습니다. 이것은 측정 artifact이지 실제 지연은 아닙니다.

</details>

---

**Q2. Consumer 처리 속도가 Producer 속도보다 빠른데 Lag이 계속 쌓입니다. 어떻게 가능한가요?**

<details>
<summary>해설 보기</summary>

이 경우는 드물지만 다음 상황에서 발생합니다.

**IDLE Consumer 존재**: Consumer가 더 많아도 파티션 수만큼만 처리합니다. 파티션 3개에 Consumer 10개면 7개가 IDLE이고, 3개만 처리합니다. 실제 처리 Consumer 수가 적어서 Lag이 쌓일 수 있습니다.

**특정 파티션 병목**: 전체 Consumer 처리 속도는 빠르지만 특정 파티션을 담당하는 Consumer가 느리면 그 파티션의 Lag만 쌓입니다. 평균 속도는 빠르지만 최악 파티션이 느린 경우입니다.

**리밸런싱 중 처리 중단**: 리밸런싱이 자주 발생하면 처리 중단 구간에 Producer 메시지가 쌓여서 Lag이 축적됩니다. 처리 속도는 빠르지만 중단 시간이 더 길 수 있습니다.

</details>

---

**Q3. Lag 알람 임계값을 얼마로 설정해야 하나요?**

<details>
<summary>해설 보기</summary>

절대값(예: "Lag > 10000이면 알람")보다 상대값과 증가 추세를 기반으로 설정하는 것이 더 실용적입니다.

**증가 추세 기반**: "5분 동안 Lag이 50% 이상 증가하면 알람" — 절대값이 크더라도 감소 추세면 정상 회복 중입니다. 증가 추세가 중요합니다.

**처리 지연 기반**: "Lag / (현재 처리 속도) > N분 분량이면 알람" — 현재 속도로 처리하는 데 걸리는 예상 시간이 SLA(예: 5분)를 초과하면 알람입니다.

**비즈니스 SLA 기반**: 결제 알림은 "1분 내 처리"가 SLA라면 "Producer 처리량 × 60초"를 초과하면 알람입니다.

실무에서는 (1) Lag이 1분 이상 지속적으로 증가하면 Warning, (2) Lag이 5분 이상 감소 없이 쌓이면 Critical로 2단계 알람을 설정하는 방식이 일반적입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 리밸런싱 중 중복 처리](./04-rebalancing-duplicate.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — Producer 처리량 최적화 ➡️](../performance-monitoring/01-producer-throughput.md)**

</div>
