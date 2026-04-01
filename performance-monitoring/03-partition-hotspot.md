# 파티션 핫스팟 — 원인과 Custom Partitioner

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 파티션 핫스팟이란 무엇이고 어떤 상황에서 발생하는가?
- `kafka-log-dirs`로 파티션별 크기 불균형을 진단하는 방법은?
- 기본 키 해시 파티셔닝의 한계는?
- Custom Partitioner로 핫스팟 키를 여러 파티션에 분산하는 구현 방법은?
- 핫스팟 해결 시 키 기반 순서 보장을 어떻게 유지하는가?
- Sticky Partitioner가 키 없는 메시지의 불균형을 완화하는 방식은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

파티셔닝 전략을 잘못 선택하면 특정 파티션에 메시지가 몰려서 해당 파티션 Leader 브로커의 CPU/디스크/네트워크가 병목이 된다. 나머지 파티션은 한가하지만 전체 처리량이 핫스팟 파티션의 한계에 묶인다.

예: 사용자 ID를 파티션 키로 사용하는데, 대형 쇼핑몰이 전체 트래픽의 80%를 차지하는 B2B 시스템. 해당 쇼핑몰의 모든 이벤트가 동일 파티션으로 쏠려서 그 파티션 Consumer만 과부하.

파티션 핫스팟을 방치하면:
- 특정 Consumer 과부하 → Lag 집중 → 처리 지연
- 해당 브로커 디스크 불균형 증가
- Consumer 수를 아무리 늘려도 개선 없음 (핫스팟 파티션 = 1개 Consumer만 처리)

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 핫스팟을 모른 채 Consumer만 늘리기

  상황:
    파티션 6개, Consumer 6개
    파티션 0 Lag: 50,000 (집중!)
    파티션 1~5 Lag: 0~100 (여유)

  잘못된 대응:
    Consumer를 12개로 늘림 → 파티션 0에는 여전히 Consumer 1개
    → 파티션 0 Lag 전혀 개선되지 않음

  올바른 진단:
    파티션 0에만 Lag 집중 → 특정 키의 메시지가 파티션 0에 쏠림
    → 파티션 키 분석 → 핫스팟 키 확인 → Custom Partitioner 적용

실수 2: 핫스팟 키를 무작위 파티션으로 분산 (순서 보장 포기)

  현상: user-1234(VIP)의 이벤트가 파티션 0에 집중
  잘못된 해결:
    user-1234의 키를 null로 변경 → Round-Robin으로 분산
    결과: user-1234의 이벤트 순서 보장 깨짐
          주문 생성보다 주문 취소 이벤트가 먼저 처리 → 데이터 불일치

  올바른 해결:
    user-1234를 접미사로 분산: "user-1234-0", "user-1234-1", "user-1234-2"
    → 3개 파티션에 분산되지만 각 파티션 내에서 순서 보장
    → Consumer가 세 파티션의 이벤트를 합쳐서 처리 (시간순 정렬 필요)

실수 3: 파티션 크기 불균형을 Lag으로만 진단

  Lag이 없어도 파티션 크기 불균형이 있을 수 있음:
    Lag = 0 → Consumer가 따라가고 있음
    하지만 파티션 0의 디스크 크기가 나머지보다 10배 크면
    → 브로커 디스크 불균형 → 특정 브로커 디스크 부족 위험

  진단:
    kafka-log-dirs --describe로 파티션별 크기 확인
    Lag과 파티션 크기 모두 모니터링 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
핫스팟 진단 절차:
  1. Lag 확인
     kafka-consumer-groups --describe → 파티션별 Lag 불균형 확인

  2. 파티션 크기 확인
     kafka-log-dirs --topic-list orders --describe

  3. 핫스팟 키 확인
     # 파티션별 메시지 키 분포 샘플링
     kafka-console-consumer --from-beginning \
       --property print.key=true --max-messages 1000 | \
       cut -f1 | sort | uniq -c | sort -rn | head -10

  4. Custom Partitioner 적용 또는 키 설계 변경

핫스팟 해결 전략:
  전략 A: 핫스팟 키를 N개로 샤딩
    key = "user-1234" → "user-1234-0", "user-1234-1", "user-1234-2"
    파티션 0,1,2에 균등 분산
    Consumer: 세 파티션 이벤트를 userId 기준으로 합쳐서 처리

  전략 B: Custom Partitioner로 특정 키 분산
    대부분의 키는 기본 해시
    핫스팟 키만 라운드로빈 또는 서브 파티션 로직 적용
```

---

## 🔬 내부 동작 원리

### 1. 파티션 핫스팟 발생 메커니즘

```
시나리오: 주문 이벤트 토픽, 키=상점 ID

  일반 상점 트래픽: 상점당 초당 10개 주문
  VIP 대형 쇼핑몰 (shop-9999): 초당 5,000개 주문

  기본 해시 파티셔닝:
    hash("shop-9999") % 6 = 2  → 파티션 2에 집중!

  결과:
    파티션 0,1,3,4,5: ~10 msg/sec (여유)
    파티션 2:          ~5,000 msg/sec (과부하!)

  파티션 2 담당 브로커:
    디스크 I/O: 파티션 2만 과부하
    Consumer 2: 5,000 msg/sec 처리 필요
    나머지 Consumer: 10 msg/sec 처리

  처리량 병목:
    전체 시스템 처리량 = 파티션 2 처리 한계에 묶임
    브로커 전체는 한가한데 특정 파티션만 포화
```

### 2. kafka-log-dirs로 파티션 불균형 진단

```
kafka-log-dirs --bootstrap-server localhost:19092 \
  --topic-list orders --describe

출력 예시 (JSON 형식):
{
  "brokers": [
    {
      "broker": 1,
      "logDirs": [{
        "partitions": [
          {"partition": "orders-0", "size": 52428800},   ← 50 MB
          {"partition": "orders-1", "size": 52428800},   ← 50 MB
          {"partition": "orders-2", "size": 524288000},  ← 500 MB (10배!)
          {"partition": "orders-3", "size": 51380224},   ← 49 MB
        ]
      }]
    }
  ]
}

파티션 2가 500 MB로 나머지 대비 10배 크면 핫스팟 의심

추가 진단:
  파티션별 메시지 수 확인:
  kafka-run-class kafka.tools.GetOffsetShell \
    --bootstrap-server localhost:19092 \
    --topic orders \
    --time -1  ← 최신 offset (= 총 메시지 수 근사값)

  출력:
  orders:0:10500
  orders:1:10200
  orders:2:95000  ← 다른 파티션의 9배!
  orders:3:10100
```

### 3. Custom Partitioner 구현 패턴

```java
// 핫스팟 키를 서브 파티션으로 분산하는 Custom Partitioner
public class HotspotPartitioner implements Partitioner {

    // 핫스팟으로 알려진 키 목록 (설정 또는 동적 감지)
    private final Set<String> hotspotKeys = new HashSet<>(
        Arrays.asList("shop-9999", "shop-8888", "shop-7777")
    );
    private final int hotspotShards = 3; // 핫스팟 키를 3개로 분산
    private final AtomicInteger counter = new AtomicInteger(0);

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int numPartitions = cluster.partitionCountForTopic(topic);
        String keyStr = key.toString();

        if (hotspotKeys.contains(keyStr)) {
            // 핫스팟 키: 라운드로빈으로 0~(hotspotShards-1) 파티션에 분산
            return counter.getAndIncrement() % hotspotShards;
        }

        // 일반 키: 기본 murmur2 해시
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }

    @Override
    public void configure(Map<String, ?> configs) {
        // hotspotKeys를 설정에서 읽어올 수 있음
    }
}
```

### 4. 키 샤딩으로 순서 보장 유지

```
전략: 핫스팟 키에 샤드 번호 접미사 추가

  원래 키: "shop-9999"
  샤딩 후: "shop-9999-0", "shop-9999-1", "shop-9999-2"

  파티셔닝:
    hash("shop-9999-0") % 6 = 2  → 파티션 2
    hash("shop-9999-1") % 6 = 4  → 파티션 4
    hash("shop-9999-2") % 6 = 0  → 파티션 0

  Producer 코드:
    int shard = eventSequenceId % 3;  // 또는 random, round-robin
    String shardedKey = originalKey + "-" + shard;
    producer.send(new ProducerRecord<>(topic, shardedKey, value));

  Consumer에서 재결합:
    세 파티션의 이벤트를 원래 키("shop-9999")로 그룹화
    시간순 정렬 후 처리 (이벤트 타임스탬프 활용)

  순서 보장 유지:
    같은 샤드 내에서는 순서 보장 (같은 파티션)
    다른 샤드 간에는 타임스탬프 기반 정렬
    → 엄격한 순서가 필요하면 단일 샤드 사용
       유연한 순서면 샤딩 적용
```

### 5. 키 없는 메시지에서의 분산 (Sticky Partitioner)

```
키 없는 메시지(key=null) Sticky Partitioner 동작:
  현재 배치가 채워질 때까지 동일 파티션으로 전송
  배치 완성 → 다음 파티션으로 전환

  이점:
    배치가 꽉 차서 전송 → 처리량 향상
    Round-Robin보다 균등한 파티션 분산 (장기적으로)
    각 파티션에 의미있는 배치 크기 도달

  Round-Robin 문제:
    메시지 1 → P0, 메시지 2 → P1, 메시지 3 → P2
    각 파티션 배치 크기: 1개씩
    배치 효과 없음 → 처리량 낮음

  Sticky 해결:
    메시지 1~64 → P0 (배치 채워질 때까지)
    메시지 65~128 → P1
    배치 효과 극대화 → 처리량 향상
```

---

## 💻 실전 실험

### 실험 1: 파티션 크기 불균형 진단

```bash
# 핫스팟 시뮬레이션: 특정 키로 집중 발행
for i in $(seq 1 10000); do
  echo "shop-9999:event-$i"
done | kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --property "parse.key=true" \
  --property "key.separator=:"

# 파티션 크기 불균형 확인
kafka-log-dirs --bootstrap-server localhost:19092 \
  --topic-list orders --describe | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for broker in data['brokers']:
    for logdir in broker['logDirs']:
        for part in logdir['partitions']:
            print(f\"{part['partition']}: {part['size']:,} bytes\")
"
```

### 실험 2: Custom Partitioner 적용 전후 분산 비교

```bash
# 기본 파티셔너 (핫스팟 발생)
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group lag-monitor | awk '{print $3, $6}'
# PARTITION LAG: 특정 파티션에 집중 확인

# Custom Partitioner 적용 후 (Producer 재시작)
# Spring Kafka 설정:
# spring.kafka.producer.properties.partitioner.class=com.example.HotspotPartitioner

# 재확인: Lag 균등 분산 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group lag-monitor | awk '{print $3, $6}'
```

### 실험 3: 파티션 키 분포 샘플링

```bash
# 파티션별 키 분포 분석
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --from-beginning \
  --max-messages 10000 \
  --property print.key=true \
  --property print.partition=true | \
  awk '{print $1}' | sort | uniq -c | sort -rn | head -20
# 가장 많이 나타나는 키 = 핫스팟 후보
```

---

## 📊 성능/비용 비교

### 핫스팟 전후 처리량 비교

```
조건: 파티션 6개, Consumer 6개, VIP 키 80% 트래픽

  핫스팟 발생 시:
    파티션 2: 4,000 msg/sec (80% 트래픽 집중)
    파티션 0,1,3,4,5: 200 msg/sec 각
    Consumer 2: 과부하, Lag 증가
    전체 처리량 한계: ~4,000 + 5×200 = 5,000 msg/sec
    (Consumer 2가 병목)

  Custom Partitioner 적용 후:
    VIP 키를 3개 파티션에 균등 분산 (~1,333 msg/sec/파티션)
    파티션 0,1,2: ~1,333 msg/sec 각
    파티션 3,4,5: ~200 msg/sec 각
    전체 처리량: 3×1,333 + 3×200 = 4,599 msg/sec (각 Consumer 균형)
    → 처리량 병목 해소
```

---

## ⚖️ 트레이드오프

```
기본 키 해시 파티셔닝:
  ✅ 동일 키 → 동일 파티션 → 완전한 순서 보장
  ✅ 구현 단순
  ❌ 키 분포 불균형 시 핫스팟 발생

Custom Partitioner (핫스팟 분산):
  ✅ 핫스팟 해소 → 균등한 처리량
  ❌ 같은 키의 메시지가 여러 파티션으로 분산 → 순서 보장 약화
  ❌ 구현 및 유지보수 복잡성 증가
  ❌ 동적 핫스팟 키 변경 시 Partitioner 재설정 필요

키 샤딩:
  ✅ 순서 보장 유지하면서 분산
  ❌ Consumer에서 재결합 로직 필요 (복잡도 증가)
  ❌ 타임스탬프 기반 정렬 오버헤드

파티션 수 증가:
  ✅ 더 세밀한 분산 가능
  ❌ 파티션 증가 시 키 해시 분포 변경 → 기존 순서 보장 깨짐
  ❌ 파티션 수는 줄일 수 없음
```

---

## 📌 핵심 정리

```
파티션 핫스팟 핵심:

1. 핫스팟 = 특정 키의 트래픽 쏠림 → 단일 파티션 과부하
   Consumer를 늘려도 해당 파티션은 1개 Consumer만 처리

2. 진단 방법:
   kafka-consumer-groups: 파티션별 Lag 불균형 확인
   kafka-log-dirs: 파티션별 크기 불균형 확인
   메시지 키 분포 샘플링: 핫스팟 키 식별

3. 해결 전략:
   Custom Partitioner: 핫스팟 키를 여러 파티션으로 강제 분산
   키 샤딩: 키에 샤드 번호 추가 → 여러 파티션 + 순서 유지

4. 순서 보장과 분산의 트레이드오프:
   동일 파티션 = 완전한 순서 보장
   여러 파티션 = 타임스탬프 기반 정렬 필요

5. Sticky Partitioner (키 없는 경우):
   배치 채워질 때까지 동일 파티션 → 처리량 향상
   장기적으로 균등한 파티션 분산
```

---

## 🤔 생각해볼 문제

**Q1. 파티션이 6개인데 핫스팟 키를 10개 파티션으로 분산하려 했습니다. 어떻게 해야 하나요?**

<details>
<summary>해설 보기</summary>

먼저 파티션 수를 10개 이상으로 늘려야 합니다. 현재 파티션 6개이면 10개로 분산할 수 없습니다. `kafka-topics --alter --partitions 10`으로 늘릴 수 있지만, 기존 키 해시 분포가 바뀌어 이미 저장된 데이터의 파티션 배치와 불일치가 발생합니다.

만약 파티션 수를 늘리기 어렵다면 6개 파티션 내에서 최대한 분산하는 방향으로 Custom Partitioner를 설계합니다. 핫스팟 키를 3개로 샤딩해서 파티션 0, 2, 4에 배치하고 일반 키는 나머지 파티션과 함께 분산하는 방식을 쓸 수 있습니다.

</details>

---

**Q2. 핫스팟 키를 동적으로 감지해서 자동으로 분산하는 방법이 있나요?**

<details>
<summary>해설 보기</summary>

Custom Partitioner에서 슬라이딩 윈도우로 키별 메시지 빈도를 추적하고 임계값을 초과하면 자동으로 분산 로직을 적용할 수 있습니다.

```java
// 슬라이딩 윈도우 키 빈도 추적
ConcurrentHashMap<String, AtomicLong> keyCounter = new ConcurrentHashMap<>();
long windowMs = 60_000; // 1분 윈도우
long hotspotThreshold = 1000; // 1분에 1000개 이상이면 핫스팟

public int partition(...) {
    long count = keyCounter.computeIfAbsent(keyStr, k -> new AtomicLong())
                           .incrementAndGet();
    if (count > hotspotThreshold) {
        // 핫스팟: 라운드로빈으로 분산
        return counter.getAndIncrement() % numPartitions;
    }
    // 일반: 키 해시
    return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
}
```

단, 이 방식은 Producer 인스턴스마다 독립적으로 추적하므로 분산 환경에서 집계가 부정확합니다. 정밀한 동적 감지가 필요하면 Redis 같은 공유 카운터를 사용하거나, Kafka Streams로 실시간 키 빈도를 집계해서 설정에 반영하는 방식을 사용합니다.

</details>

---

**Q3. 핫스팟 키의 이벤트를 여러 파티션에 분산했을 때 Consumer에서 이벤트 순서를 어떻게 복원하나요?**

<details>
<summary>해설 보기</summary>

이벤트에 포함된 타임스탬프나 시퀀스 번호를 기준으로 정렬합니다.

`shop-9999-0`, `shop-9999-1`, `shop-9999-2` 세 파티션에서 이벤트를 읽어서 원래 키(`shop-9999`)로 그룹화한 후, `eventTimestamp` 또는 `sequenceId` 기준으로 오름차순 정렬합니다. 이후 정렬된 순서로 처리합니다.

단, 이 방식은 세 파티션에서 최소한 일정 분량의 이벤트가 모두 수집된 후에 정렬할 수 있습니다. 무한정 대기할 수 없으므로 타임아웃(예: 이벤트 타임스탬프 기준 N초 내의 이벤트를 모아서 처리)을 설정합니다. 이것이 스트림 처리에서의 "윈도우 연산"과 "지연 이벤트 처리" 문제로 이어집니다. Kafka Streams나 Flink의 이벤트 타임 윈도우가 이 문제를 체계적으로 다룹니다.

</details>

---

<div align="center">

**[⬅️ 이전: Consumer 처리량 최적화](./02-consumer-throughput.md)** | **[홈으로 🏠](../README.md)** | **[다음: Kafka 모니터링 ➡️](./04-kafka-monitoring.md)**

</div>
