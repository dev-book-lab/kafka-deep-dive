# Producer 처리량 최적화 — 배치와 압축

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `batch.size`와 `linger.ms`를 함께 조정할 때 처리량이 높아지는 원리는?
- `buffer.memory`가 가득 찼을 때 Producer 애플리케이션은 어떻게 반응하는가?
- gzip / snappy / lz4 / zstd 각각의 CPU 비용과 압축률 트레이드오프는?
- 압축을 Producer에서 설정하면 브로커와 Consumer에서 각각 어떻게 처리되는가?
- `max.in.flight.requests.per.connection`이 처리량에 미치는 영향은?
- 처리량 최적화 설정을 적용할 때 지연(latency)이 증가하는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka의 기본 Producer 설정은 낮은 지연을 위해 최적화되어 있다. `linger.ms=0`(기본값)은 메시지를 즉시 전송하므로 배치 효과가 없다. 처리량이 중요한 환경(로그 수집, 이벤트 파이프라인)에서는 이 기본값을 유지하면 브로커에 수많은 소량 요청이 쏟아져 처리량 병목이 생긴다.

Producer 튜닝을 모르면:
- 처리량 낮음 → 브로커 서버 무작정 증설 → 비용 낭비
- 압축 없이 운영 → 네트워크 대역폭 과점유
- `buffer.memory` 부족 → 애플리케이션 스레드 블로킹 → 장애

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 기본값(linger.ms=0)으로 고처리량 운영

  환경: 초당 100,000 메시지 발행
  설정: linger.ms=0, batch.size=16384 (기본값)
  
  결과:
    각 배치가 1~3개 메시지만 담아서 전송
    브로커에 초당 ~50,000~100,000번 요청
    브로커 CPU: 요청 처리 오버헤드로 과부하
    처리량: 기대의 30% 수준

  올바른 설정:
    linger.ms=20, batch.size=65536
    → 배치당 평균 50~100개 메시지 → 요청 수 1/50로 감소

실수 2: 압축 방식을 "가장 잘 압축되는" gzip으로 무조건 선택

  문제:
    gzip: CPU 사용량이 snappy/lz4 대비 3~5배 높음
    고처리량 환경에서 Producer CPU가 gzip 압축에 병목
    → 처리량 오히려 감소

  올바른 선택:
    처리량 + CPU 균형: lz4 (빠름, 압축률 적당)
    저장 비용 중요:   zstd (높은 압축률, 적당한 CPU)
    레거시 호환:      snappy

실수 3: buffer.memory를 너무 작게 설정

  설정: buffer.memory=10485760 (10 MB, 기본 32 MB보다 작음)
  상황: 브로커 장애로 전송 지연 → 배치 버퍼에 쌓임
  결과:
    10 MB 가득 참 → send() 호출 시 max.block.ms(기본 60초)까지 블로킹
    애플리케이션 스레드 60초 멈춤 → 서비스 전체 영향
  
  설정 기준:
    buffer.memory >= batch.size × (파티션 수) × (예상 병렬 Producer 스레드 수)
    일반적으로 64 MB ~ 256 MB 권장
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
처리량 우선 설정:
  spring:
    kafka:
      producer:
        batch-size: 65536           # 64 KB (기본 16 KB → 4배)
        linger-ms: 20               # 20ms 대기 (기본 0)
        buffer-memory: 67108864     # 64 MB (기본 32 MB → 2배)
        compression-type: lz4       # 빠른 압축
        acks: all                   # 내구성 유지
        properties:
          enable.idempotence: true
          max.in.flight.requests.per.connection: 5

지연 우선 설정 (실시간 알림 등):
  producer:
    batch-size: 16384    # 기본값 유지
    linger-ms: 0         # 즉시 전송
    compression-type: none

처리량 측정 기준 수립:
  kafka-producer-perf-test \
    --topic perf-test \
    --num-records 1000000 \
    --record-size 1024 \
    --throughput -1 \
    --producer-props \
      bootstrap.servers=localhost:19092 \
      batch.size=65536 \
      linger.ms=20 \
      compression.type=lz4 \
      acks=all
  # 튜닝 전후 msg/sec 비교
```

---

## 🔬 내부 동작 원리

### 1. batch.size와 linger.ms 상호작용

```
RecordAccumulator 배치 전송 조건 복습:
  조건 1: batch.size에 도달 → 즉시 전송
  조건 2: linger.ms 만료  → 크기 무관 전송
  → 둘 중 먼저 충족되는 것

처리량 최적화 원리:

  시나리오: 초당 10,000 메시지, 메시지당 1 KB

  linger.ms=0 (기본):
    메시지 1개 → 즉시 전송 (배치 = 1개)
    초당 10,000번 브로커 요청
    브로커 요청 처리 오버헤드 지배적

  linger.ms=20ms, batch.size=64KB:
    t=0ms:    메시지 1개 → 배치 시작
    t=1~19ms: 메시지 19개 추가 → 배치에 쌓임
    t=20ms:   linger.ms 만료 → 20개 배치 전송
    초당 500번 브로커 요청 (10,000 / 20)
    요청 수 1/20로 감소 → 처리량 대폭 향상

  batch.size=64KB 달성 시:
    64KB ÷ 1KB = 64개 메시지가 쌓이면 20ms 전 즉시 전송
    초당 156번 브로커 요청 (10,000 / 64)

핵심: linger.ms는 "최대 대기 시간"
      batch.size는 "조기 전송 트리거"
      둘을 함께 늘릴수록 배치가 커지고 처리량이 높아짐
      단, linger.ms만큼의 지연이 모든 메시지에 추가됨
```

### 2. 압축 방식별 특성

```
브로커 기준 1 KB 메시지 100만 건 처리:

  no compression:
    Producer CPU:    낮음
    압축률:          0%  (원본 크기)
    브로커 저장:     1 GB
    네트워크 절감:   없음
    Consumer CPU:   낮음

  snappy:
    Producer CPU:    중간
    압축률:          ~30% (저장: 700 MB)
    네트워크 절감:   ~30%
    Consumer CPU:   낮음 (빠른 압축 해제)
    특징: 구글 개발, 속도와 압축률 균형

  lz4:
    Producer CPU:    낮음 (snappy보다 빠름)
    압축률:          ~25% (저장: 750 MB)
    네트워크 절감:   ~25%
    Consumer CPU:   매우 낮음
    특징: 처리량 최우선 환경 권장

  gzip:
    Producer CPU:    높음 (lz4 대비 3~5배)
    압축률:          ~70% (저장: 300 MB)
    네트워크 절감:   ~70%
    Consumer CPU:   중간
    특징: 저장 비용이 핵심인 경우

  zstd (Kafka 2.1+):
    Producer CPU:    중간
    압축률:          ~60% (저장: 400 MB)
    네트워크 절감:   ~60%
    Consumer CPU:   낮음
    특징: gzip 수준의 압축률 + lz4 수준의 속도

브로커 재압축:
  Producer compression.type=lz4 설정
  브로커 topic config에 compression.type=producer (기본값)
  → 브로커가 Producer 압축 그대로 저장 (재압축 없음)

  브로커 topic config에 compression.type=gzip 설정
  → 브로커가 Producer 메시지를 gzip으로 재압축
  → CPU 오버헤드 발생
  → 운영 환경에서는 보통 Producer에서 압축 권장
```

### 3. buffer.memory와 배압 메커니즘

```
buffer.memory = Producer 전체 메모리 풀

  정상 상태:
    send() → RecordAccumulator에 즉시 배치
    Sender 스레드가 배치를 브로커로 전송
    전송 완료 → 메모리 반환

  브로커 장애 또는 네트워크 지연:
    Sender 전송 지연 → 배치가 버퍼에 쌓임
    buffer.memory 90% 채워짐

  buffer.memory 100% 가득 참:
    send() 호출 → 공간 없음
    → 애플리케이션 스레드: max.block.ms(기본 60초) 대기
    → 60초 내 공간 생기면 진행
    → 60초 후에도 공간 없음 → TimeoutException

  buffer.memory 설정 기준:
    batch.size × 동시 활성 파티션 수 × 여유 배수
    예: batch.size=64KB, 파티션 30개, 여유 2배
        = 64KB × 30 × 2 = 3.84 MB (최소)
    실무: 64 MB ~ 256 MB (충분한 여유)

  max.block.ms 최적화:
    기본 60초는 너무 길 수 있음
    빠른 장애 감지 원하면: max.block.ms=5000 (5초)
    TimeoutException 발생 시 서킷 브레이커 활성화
```

### 4. max.in.flight.requests와 처리량

```
max.in.flight.requests.per.connection=5 (기본값):

  브로커로부터 응답 대기 중인 요청: 최대 5개
  5개 모두 전송 중 → 6번째 전송 대기

  파이프라이닝 효과:
    요청1 전송 → 응답 기다리는 동안 요청2,3,4,5 전송
    → 네트워크 왕복 시간(RTT) 동안 idle 없음
    → 처리량 max.in.flight 배수로 향상

  max.in.flight=1 (순서 보장만, 멱등성 없을 때):
    요청1 응답 받아야 요청2 전송
    → RTT만큼 idle → 처리량 1/5로 감소

  enable.idempotence=true 조합:
    max.in.flight=5 유지 가능
    순서 보장 + 중복 제거 + 높은 처리량 동시 달성
```

---

## 💻 실전 실험

### 실험 1: linger.ms와 batch.size 튜닝 전후 처리량 비교

```bash
# 기본값 (낮은 처리량)
kafka-producer-perf-test \
  --topic perf-topic \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:19092 \
    linger.ms=0 \
    batch.size=16384 \
    acks=1
# 예시: 95,000 records/sec

# 최적화 (높은 처리량)
kafka-producer-perf-test \
  --topic perf-topic \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:19092 \
    linger.ms=20 \
    batch.size=65536 \
    compression.type=lz4 \
    acks=all \
    enable.idempotence=true
# 예시: 320,000 records/sec → ~3.4배 향상
```

### 실험 2: 압축 방식별 처리량 비교

```bash
for CODEC in none snappy lz4 gzip zstd; do
  echo "=== $CODEC ==="
  kafka-producer-perf-test \
    --topic perf-topic \
    --num-records 500000 \
    --record-size 1024 \
    --throughput -1 \
    --producer-props \
      bootstrap.servers=localhost:19092 \
      linger.ms=20 \
      batch.size=65536 \
      compression.type=$CODEC \
      acks=1
  echo ""
done
# 압축 방식별 records/sec, MB/sec, 압축률 비교
```

### 실험 3: buffer.memory 소진 상황 재현

```bash
# 브로커 연결 차단 (iptables)
docker exec kafka-1 iptables -I INPUT -p tcp --dport 9092 -j DROP

# 짧은 max.block.ms로 빠른 실패 확인
kafka-producer-perf-test \
  --topic perf-topic \
  --num-records 10000 \
  --record-size 10240 \
  --throughput 10000 \
  --producer-props \
    bootstrap.servers=localhost:19092 \
    buffer.memory=1048576 \
    max.block.ms=5000
# buffer.memory 가득 참 → 5초 후 TimeoutException

# 브로커 연결 복구
docker exec kafka-1 iptables -D INPUT -p tcp --dport 9092 -j DROP
```

---

## 📊 성능/비용 비교

### 설정 조합별 처리량 (1 KB 메시지, 파티션 3개, acks=all)

```
기본값 (linger.ms=0, batch.size=16KB, 압축 없음):
  처리량: ~100,000 msg/sec
  네트워크: ~100 MB/sec
  Producer CPU: 낮음

linger.ms=20, batch.size=64KB, 압축 없음:
  처리량: ~280,000 msg/sec (+180%)
  네트워크: ~280 MB/sec

linger.ms=20, batch.size=64KB, lz4:
  처리량: ~350,000 msg/sec (+250%)
  네트워크: ~210 MB/sec (-25% 절감)
  Producer CPU: 약간 증가

linger.ms=20, batch.size=64KB, zstd:
  처리량: ~300,000 msg/sec (+200%)
  네트워크: ~140 MB/sec (-50% 절감)
  Producer CPU: 중간

linger.ms=20, batch.size=64KB, gzip:
  처리량: ~180,000 msg/sec (+80%)
  네트워크: ~90 MB/sec (-65% 절감)
  Producer CPU: 높음 (CPU 병목)

결론:
  순수 처리량 → lz4
  네트워크/저장 비용 절감 → zstd
  CPU 여유 있고 저장 비용이 가장 중요 → gzip
```

---

## ⚖️ 트레이드오프

```
linger.ms 증가:
  ✅ 배치가 커짐 → 처리량 증가
  ❌ 최소 linger.ms만큼 지연 추가 (실시간성 저하)

batch.size 증가:
  ✅ 배치 효율 향상 → 처리량 증가, 압축 효율 향상
  ❌ 실패 시 큰 배치 재전송 → 재시도 비용 증가

압축 활성화:
  ✅ 네트워크 대역폭 절감, 디스크 사용 절감
  ❌ Producer CPU 증가
  ❌ Consumer에서 압축 해제 CPU 증가

buffer.memory 증가:
  ✅ 브로커 지연 시 버퍼 여유 확보 → 장애 내성 향상
  ❌ JVM 힙 외 메모리 사용 → OOM 위험 (너무 크게 설정 시)

max.in.flight 증가:
  ✅ 파이프라이닝 처리량 증가
  ❌ 멱등성 없이 retries와 함께 사용 시 순서 역전 위험
```

---

## 📌 핵심 정리

```
Producer 처리량 최적화 핵심:

1. linger.ms + batch.size 조합이 핵심
   linger.ms: 배치를 채울 대기 시간
   batch.size: 조기 전송 트리거 크기
   처리량 환경: linger.ms=10~20, batch.size=32KB~64KB

2. 압축 선택:
   처리량 우선: lz4 (빠른 압축, CPU 낮음)
   비용 절감:   zstd (높은 압축률, 중간 CPU)
   극한 절약:   gzip (최고 압축률, CPU 높음)

3. buffer.memory: 64 MB 이상 권장
   브로커 장애 시 버퍼 역할
   max.block.ms를 짧게 → 빠른 장애 감지

4. max.in.flight=5 + enable.idempotence=true
   파이프라이닝 처리량 + 순서 보장 + 중복 제거 동시 달성

5. 튜닝 전 kafka-producer-perf-test로 베이스라인 측정
   변경마다 비교 측정으로 실제 효과 확인
```

---

## 🤔 생각해볼 문제

**Q1. `linger.ms=100`으로 설정하면 모든 메시지에 최소 100ms 지연이 추가되나요?**

<details>
<summary>해설 보기</summary>

최대 100ms 지연이 추가됩니다. `linger.ms`는 배치가 `batch.size`에 도달하기 전 최대 대기 시간입니다.

메시지가 빠르게 들어와서 100ms 전에 `batch.size`에 도달하면 즉시 전송되므로 지연이 없습니다. 메시지가 드문드문 들어오는 경우만 100ms 대기합니다.

예: 초당 10,000개 메시지, `batch.size=64KB`(약 64개/1KB 메시지)이면 6.4ms마다 배치가 가득 참 → `linger.ms=100`이어도 실제 지연은 최대 6.4ms입니다. 처리량이 낮아질 때 비로소 `linger.ms`가 실제 대기 시간으로 작용합니다.

</details>

---

**Q2. Producer에서 lz4로 압축했는데 브로커에서 gzip으로 재압축하도록 설정했습니다. 이중 압축이 되나요?**

<details>
<summary>해설 보기</summary>

네, 이중 처리가 발생합니다. 브로커 토픽 설정의 `compression.type`을 `gzip`으로 지정하면, 브로커는 Producer가 보낸 lz4 압축 배치를 받아서 압축 해제 후 gzip으로 재압축하여 저장합니다. CPU 낭비가 발생합니다.

올바른 설정:
- `compression.type=producer`(브로커 기본값): Producer가 압축한 방식 그대로 저장. 재압축 없음.
- `compression.type=uncompressed`(브로커): 항상 압축 없이 저장. Producer 압축 무시.
- `compression.type=lz4`(브로커): 항상 lz4로 저장. Producer가 다른 방식 사용 시 재압축.

실무에서는 브로커 설정을 `compression.type=producer`로 두고, Producer에서 원하는 압축 방식을 지정하는 것이 가장 효율적입니다.

</details>

---

**Q3. 배치 크기를 키웠더니 처리량이 오히려 줄었습니다. 무슨 이유일까요?**

<details>
<summary>해설 보기</summary>

몇 가지 원인이 가능합니다.

**브로커 최대 메시지 크기 초과**: `message.max.bytes`(브로커 기본값 1 MB) 또는 `max.request.size`(Producer 기본값 1 MB)를 배치 크기가 초과하면 오류가 발생합니다. `batch.size=2MB`인데 브로커 제한이 1MB이면 전송 실패 → 재시도 오버헤드로 처리량 감소.

**GC 압박**: `buffer.memory`와 `batch.size`를 크게 설정하면 JVM 힙에서 더 많은 메모리를 할당/해제합니다. GC 빈도가 높아지면 STW로 처리량이 떨어집니다.

**소량 메시지 환경**: 메시지가 드문드문 들어오는 환경에서 `batch.size`만 크게 설정하면 `linger.ms` 만료까지 기다리게 됩니다. `linger.ms=0`인 상태에서 `batch.size`만 늘리면 배치 효과가 없습니다. `linger.ms`도 함께 늘려야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Consumer 처리량 최적화 ➡️](./02-consumer-throughput.md)**

</div>
