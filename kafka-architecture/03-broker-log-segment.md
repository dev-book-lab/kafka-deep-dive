# 브로커 내부 구조 — 로그 세그먼트와 순차 I/O

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `.log`, `.index`, `.timeindex` 파일 각각이 저장하는 내용은?
- 순차 I/O(Sequential Write)가 랜덤 I/O보다 빠른 이유는?
- Kafka가 OS 페이지 캐시를 활용하는 방식이 Redis의 메모리 저장과 어떻게 다른가?
- 세그먼트가 롤링(rolling)되는 조건은 무엇이고, 롤링 후 오래된 세그먼트는 어떻게 삭제되는가?
- `kafka-log-dirs`와 `kafka-dump-log`로 실제 로그 파일을 어떻게 분석하는가?
- Zero-Copy(`sendfile()`)가 Consumer에게 메시지를 전달하는 과정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka가 "디스크 기반인데 왜 빠른가"를 설명하지 못하면, 성능 문제가 생겼을 때 엉뚱한 곳을 최적화하게 된다.

"Kafka가 느리다" → JVM heap을 늘린다 → 오히려 GC 압박이 심해진다

실제로 Kafka의 성능은 **순차 디스크 I/O**와 **OS 페이지 캐시** 활용에서 나온다. JVM heap을 크게 잡으면 OS 페이지 캐시로 쓰일 메모리가 줄어들어 오히려 성능이 저하된다.

브로커 내부 파일 구조를 이해하면:
- 디스크 경고 알람이 울릴 때 어느 파티션이 얼마나 차지하는지 진단 가능
- 세그먼트 롤링 설정이 retention 삭제에 어떻게 영향을 주는지 이해
- 특정 오프셋의 메시지를 빠르게 찾는 `.index` 파일의 역할 이해

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Kafka 성능 최적화를 JVM heap으로 시도

  설정: KAFKA_HEAP_OPTS="-Xmx16g -Xms16g"  (16 GB 서버에서 JVM 16 GB)
  결과:
    OS 페이지 캐시 가용 메모리: 0 (JVM이 전부 점유)
    Kafka가 쓴 데이터를 Consumer가 읽을 때 → 디스크 I/O 발생
    GC 발생 시 수백 ms~수 초 STW → 처리 지연

  올바른 설정:
    KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"  (JVM은 6 GB만)
    나머지 10 GB → OS 페이지 캐시 → 디스크 접근 없이 메모리 속도로 서빙

실수 2: 로그 세그먼트 크기를 너무 작게 설정

  설정: log.segment.bytes=1048576  (1 MB)
  결과:
    활발한 토픽에서 세그먼트 파일이 분 단위로 생성/롤링
    파일 수가 급증 → OS 파일 핸들 한계 도달 → Broker 오류
    retention 삭제도 세그먼트 단위라 1 MB짜리 파일 수천 개를 관리

  기본값: log.segment.bytes=1073741824 (1 GB) 유지 권장

실수 3: 세그먼트 롤링과 retention 삭제 타이밍 오해

  "retention.ms=86400000 (1일) 설정했는데 왜 디스크가 안 줄어들지?"
  
  실제 동작:
    Kafka는 세그먼트 단위로 삭제
    현재 쓰고 있는 Active 세그먼트는 retention 기간이 지나도 삭제 안 됨
    롤링(rolling) 후 inactive 세그먼트가 되어야 삭제 대상
    
    설정: log.segment.bytes=1GB, 하루에 500 MB만 쓴다면
    → Active 세그먼트가 1 GB 채우는 데 2일 소요
    → 2일 지나야 롤링 → 1일 retention 기간 체크 → 삭제
    → 실제로 최대 3일치 데이터가 디스크에 남을 수 있음
    
  해결: log.roll.ms=86400000 (1일) 추가 설정
        → 크기 미달이어도 1일마다 강제 롤링 → retention 적용 가능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
올바른 JVM 설정:
  총 RAM 32 GB 서버 기준:
  KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"
  → 남은 26 GB: OS 페이지 캐시 + 기타 프로세스
  
  Producer가 쓴 데이터는 즉시 페이지 캐시에 존재
  Consumer가 바로 읽으면 → 디스크 접근 없이 캐시에서 서빙
  → "RAM은 Kafka에게 직접 주는 것보다 OS에 맡기는 것이 더 효율적"

올바른 세그먼트 설정:
  log.segment.bytes=536870912    (512 MB, 기본 1 GB보다 작게)
  log.roll.ms=86400000           (1일마다 강제 롤링)
  log.retention.ms=604800000     (7일 보존)
  log.retention.check.interval.ms=300000  (5분마다 삭제 체크)

디스크 용량 계산:
  필요 디스크 = (초당 유입 MB) × (retention 초) × (복제 팩터)
  예: 100 MB/s × 604800초(7일) × 3 = 181 TB
  → 실제 파티션별, 브로커별로 분산되므로 브로커당 60 TB
  → 압축(compression) 적용 시 50~70% 절감 가능
```

---

## 🔬 내부 동작 원리

### 1. 파티션 디렉토리와 파일 구조

```
/var/kafka-logs/orders-0/        ← orders 토픽, Partition 0
├── 00000000000000000000.log      ← 메시지 데이터 (offset 0부터 시작)
├── 00000000000000000000.index    ← offset → 파일 위치 인덱스
├── 00000000000000000000.timeindex← timestamp → offset 인덱스
├── 00000000000001000000.log      ← 세그먼트 롤링 후 (offset 1,000,000부터)
├── 00000000000001000000.index
├── 00000000000001000000.timeindex
└── leader-epoch-checkpoint        ← Leader Epoch 정보

파일명의 숫자 = 해당 세그먼트의 첫 번째 offset (20자리 0 패딩)
Active 세그먼트 = 현재 쓰고 있는 .log 파일 (가장 최신)
Inactive 세그먼트 = 롤링 완료된 .log 파일들 (삭제 후보)
```

### 2. .log 파일: 메시지 배치 저장 포맷

```
.log 파일 내부 구조 (Record Batch 단위로 기록):

  ┌─────────────────────────────────────────────────────────────────┐
  │ Record Batch (offset 0~9)                                       │
  │ ┌──────────────────────────────────────────────────────────────┐│
  │ │ baseOffset: 0          (배치 첫 메시지 offset)                  ││
  │ │ batchLength: 1024      (배치 크기 bytes)                       ││
  │ │ magic: 2               (레코드 포맷 버전)                        ││
  │ │ crc: 0xABCD1234        (체크섬)                                ││
  │ │ attributes: compression=LZ4, timestampType=CREATE            ││
  │ │ lastOffsetDelta: 9     (마지막 메시지 offset - baseOffset)      ││
  │ │ firstTimestamp: 1700000000000                                ││
  │ │ maxTimestamp:   1700000001000                                ││
  │ │ producerId: 12345      (멱등성/트랜잭션용)                        ││
  │ │ producerEpoch: 1                                             ││
  │ │ baseSequence: 0                                              ││
  │ │ Records:                                                     ││
  │ │   [Record 0] key="order-1" value={"amount":100}              ││
  │ │   [Record 1] key="order-2" value={"amount":200}              ││
  │ │   ...                                                        ││
  │ │   [Record 9] key="order-9" value={"amount":900}              ││
  │ └──────────────────────────────────────────────────────────────┘│
  ├─────────────────────────────────────────────────────────────────┤
  │ Record Batch (offset 10~19)                                     │
  │ ...                                                             │
  └─────────────────────────────────────────────────────────────────┘
```

### 3. .index 파일: Sparse Index로 빠른 offset 탐색

```
.index 파일 = offset → 파일 내 바이트 위치 매핑

  구조: (상대 offset, 파일 내 position) 쌍의 배열
  고정 크기 항목: 8 bytes (4 bytes offset + 4 bytes position)
  
  ┌────────────────────────────────┐
  │ relativeOffset: 0    pos: 0    │  ← offset 0은 파일 위치 0
  │ relativeOffset: 100  pos: 1024 │  ← offset 100은 파일 위치 1024 bytes
  │ relativeOffset: 200  pos: 2100 │
  │ relativeOffset: 300  pos: 3200 │
  └────────────────────────────────┘

  Sparse Index (희소 인덱스):
    모든 offset을 저장하지 않고 index.interval.bytes(기본 4096 bytes)마다
    하나의 항목을 저장 → 인덱스 파일 크기를 작게 유지

  특정 offset 탐색 과정:
    찾는 offset = 250
    1. 인덱스에서 250보다 작은 최대값 = 200 (pos: 2100)
    2. .log 파일의 2100 byte 위치부터 순차 스캔
    3. offset 250을 찾으면 해당 레코드 반환
    → O(log N) 이진 탐색 + 소량의 순차 스캔
```

### 4. .timeindex 파일: 시간 기반 탐색

```
.timeindex = timestamp → offset 매핑

  구조: (timestamp, relativeOffset) 쌍
  
  ┌────────────────────────────────────────────┐
  │ timestamp: 1700000000000  offset: 0        │
  │ timestamp: 1700000010000  offset: 100      │
  │ timestamp: 1700000020000  offset: 200      │
  └────────────────────────────────────────────┘

  활용:
    --offset=earliest/latest는 .index 사용
    --offset=timestamp (특정 시각부터 소비)는 .timeindex 사용
    retention.ms 기반 삭제 시 timeindex로 오래된 세그먼트 식별
```

### 5. 순차 I/O와 페이지 캐시

```
순차 I/O가 빠른 이유 (HDD 기준):

  랜덤 I/O:
    헤드가 여러 위치로 이동 → 각 이동마다 seek time (~10 ms)
    100개 랜덤 읽기 = 100 × 10 ms = 1000 ms = 1초

  순차 I/O:
    헤드가 한 방향으로만 이동 → seek time 최소
    100 MB 순차 읽기 = 연속된 위치라 ~50 ms
    → 수십 배 차이

  SSD에서도:
    랜덤 I/O: ~100 μs/operation
    순차 I/O: 대역폭 한계까지 활용 (예: 500 MB/s)
    순차가 여전히 2~5배 빠름

Kafka의 페이지 캐시 활용:

  Producer → Kafka 브로커 → write() 시스템 콜
     → OS 페이지 캐시에 쓰기 (메모리)
     → OS가 비동기로 디스크에 flush
                               ↓
  Consumer → Kafka 브로커 → sendfile() 시스템 콜
     → 페이지 캐시에서 바로 소켓으로 전달 (Zero-Copy)
     → 디스크 접근 불필요 (캐시 히트 시)

  Producer가 최근에 쓴 데이터를 Consumer가 바로 읽는 경우
  → 100% 페이지 캐시 히트 → 디스크 접근 0
  → 메모리 속도로 전달
```

### 6. 세그먼트 롤링과 retention 삭제

```
세그먼트 롤링 트리거:
  1. log.segment.bytes 초과 (기본 1 GB)
  2. log.roll.ms 초과 (기본 없음, 설정 필요)
  3. log.roll.jitter.ms (롤링 시점 분산)
  4. index 파일이 log.index.size.max.bytes 초과

롤링 후 상태:
  이전 세그먼트 → Inactive (읽기 전용, 삭제 후보)
  새 세그먼트 → Active (쓰기 가능)

Retention 삭제 로직 (log.retention.check.interval.ms 주기로 실행):
  각 Inactive 세그먼트에 대해:
    IF (현재시각 - 세그먼트 최대 timestamp) > retention.ms → 삭제 대상
    OR IF 총 파티션 크기 > retention.bytes → 가장 오래된 세그먼트부터 삭제

  Active 세그먼트는 retention 기간이 지나도 삭제 안 됨
  → log.roll.ms를 설정해야 Active가 롤링되어 삭제 가능
```

---

## 💻 실전 실험

### 실험 1: 로그 세그먼트 파일 직접 확인

```bash
# Docker 내부에서 로그 디렉토리 확인
docker exec kafka-1 ls -la /var/kafka-logs/orders-0/

# 출력 예시:
# -rw-r--r-- 1 root root 10485760 Dec 1 10:00 00000000000000000000.index
# -rw-r--r-- 1 root root  1048576 Dec 1 10:00 00000000000000000000.log
# -rw-r--r-- 1 root root 10485760 Dec 1 10:00 00000000000000000000.timeindex
# -rw-r--r-- 1 root root       8 Dec 1 10:00 leader-epoch-checkpoint

# kafka-log-dirs로 파티션별 크기 확인
kafka-log-dirs --bootstrap-server localhost:19092 \
  --topic-list orders \
  --describe

# 출력 예시 (JSON 형태):
# {"version":1,"brokers":[
#   {"broker":1,"logDirs":[
#     {"logDir":"/var/kafka-logs","error":null,"partitions":[
#       {"partition":"orders-0","size":1048576,"offsetLag":0,...}
#     ]}
#   ]}
# ]}
```

### 실험 2: kafka-dump-log로 세그먼트 내용 분석

```bash
# Docker 내부에서 실행
docker exec kafka-1 bash

# .log 파일 내용 분석 (Record Batch 단위)
kafka-dump-log --files /var/kafka-logs/orders-0/00000000000000000000.log \
  --print-data-log

# 출력 예시:
# Dumping /var/kafka-logs/orders-0/00000000000000000000.log
# Starting offset: 0
# baseOffset: 0 lastOffset: 0 count: 1 baseSequence: 0 ...
# | offset: 0 isValid: true CreateTime: 1700000000000 ...
#   keysize: 7 valuesize: 15 sequence: 0 headerKeys: []
#   key: order-1 payload: {"amount":100}

# .index 파일 내용 분석
kafka-dump-log --files /var/kafka-logs/orders-0/00000000000000000000.index \
  --index-sanity-check
```

### 실험 3: 세그먼트 롤링 강제 실행

```bash
# 작은 세그먼트 크기로 토픽 생성 (실험용)
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic test-segment \
  --partitions 1 \
  --replication-factor 1 \
  --config segment.bytes=1048576    # 1 MB마다 롤링
  --config retention.ms=60000       # 1분 보존

# 메시지 대량 발행
for i in $(seq 1 10000); do
  echo "message-$i"
done | kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic test-segment

# 세그먼트 파일 수 확인 (여러 파일이 생성됨)
docker exec kafka-1 ls -la /var/kafka-logs/test-segment-0/

# 1분 후 오래된 세그먼트 삭제 확인
sleep 70
docker exec kafka-1 ls -la /var/kafka-logs/test-segment-0/
```

---

## 📊 성능/비용 비교

### 순차 I/O vs 랜덤 I/O 처리량 비교

```
디스크 유형별 처리량 (단일 파티션, 1 KB 메시지):

  HDD (7200 RPM):
    순차 쓰기:   ~150 MB/s  → Kafka: ~120,000 msg/s
    랜덤 쓰기:   ~1 MB/s    → 랜덤 저장 방식: ~800 msg/s
    차이: ~150배

  SSD (SATA):
    순차 쓰기:   ~500 MB/s  → Kafka: ~400,000 msg/s
    랜덤 쓰기:   ~100 MB/s  → 차이: ~5배

  SSD (NVMe):
    순차 쓰기:   ~3000 MB/s → Kafka: ~2,400,000 msg/s
    랜덤 쓰기:   ~500 MB/s  → 차이: ~6배

결론: HDD에서도 Kafka는 순차 I/O 덕분에 충분한 성능
      SSD/NVMe 사용 시 극한의 처리량 달성 가능
```

### JVM Heap 크기와 페이지 캐시 효과

```
32 GB RAM 서버, 200 MB/s 유입, Consumer 즉시 소비:

  JVM 24 GB 설정:
    페이지 캐시: ~8 GB
    최근 쓴 데이터(40초치)만 캐시 → 그 이상은 디스크 읽기
    GC: 자주 발생, STW ~수백 ms

  JVM 6 GB 설정:
    페이지 캐시: ~26 GB
    최근 쓴 데이터(130초치)까지 캐시
    Consumer 지연이 2분 이내면 디스크 접근 0
    GC: 드물게 발생, STW ~수십 ms

→ Kafka는 JVM heap을 줄이고 OS 페이지 캐시를 극대화하는 것이 최적
```

---

## ⚖️ 트레이드오프

```
순차 I/O 설계의 트레이드오프:

  ✅ HDD에서도 높은 쓰기 처리량
  ✅ OS 페이지 캐시 자동 활용 (별도 캐시 레이어 불필요)
  ✅ 단순한 파일 기반 저장 → 운영 및 복구 용이
  ❌ 특정 메시지 랜덤 접근이 느림
     (offset 인덱스 + 순차 스캔이 필요)
  ❌ 오래된 데이터 조회 시 디스크 I/O 발생
     (페이지 캐시에서 evict된 경우)

세그먼트 크기 설정 트레이드오프:

  세그먼트 크게 (1 GB 이상):
    ✅ 파일 수 적음 → OS 파일 핸들 절약
    ✅ 인덱스 항목 수 적음 → 인덱스 탐색 효율
    ❌ 롤링 주기 길어짐 → retention 삭제 지연
    ❌ 대용량 파일 → 복구 시 세그먼트 단위 복사 비용 증가

  세그먼트 작게 (64 MB 이하):
    ✅ 빈번한 롤링 → retention 정밀 적용
    ❌ 파일 수 급증 → OS 파일 핸들 한계 위험
    ❌ 인덱스 파일 수 증가 → 메모리 압박
```

---

## 📌 핵심 정리

```
Kafka 브로커 내부 파일 구조:

  파티션 디렉토리 = 세그먼트 파일 세트 (*.log, *.index, *.timeindex)
  
  .log   → 실제 메시지 데이터 (Record Batch 단위로 append)
  .index → offset → 파일 위치 매핑 (Sparse Index, 빠른 탐색)
  .timeindex → timestamp → offset 매핑 (시간 기반 탐색)

성능의 핵심:
  1. 순차 I/O: .log 파일 끝에만 append → 디스크 헤드 이동 최소
  2. OS 페이지 캐시: JVM이 아닌 OS가 캐싱 → 메모리 효율 최대
  3. Zero-Copy(sendfile()): 페이지 캐시 → NIC 직접 전달 → CPU/메모리 복사 없음

운영 핵심:
  JVM heap = 6~8 GB (나머지는 페이지 캐시에 양보)
  log.segment.bytes = 512 MB ~ 1 GB
  log.roll.ms = 86400000 (1일, retention 정확도 위해 필수)
  kafka-log-dirs로 파티션별 크기 주기적 모니터링
```

---

## 🤔 생각해볼 문제

**Q1. Kafka는 디스크에 데이터를 저장하는데, Redis는 메모리에 저장합니다. 그런데 왜 Kafka의 쓰기 처리량이 Redis보다 높을 수 있을까요?**

<details>
<summary>해설 보기</summary>

비교 대상과 조건에 따라 다르지만, 핵심은 **병목 지점**의 차이입니다.

Redis는 메모리에 쓰고, 단일 스레드로 처리합니다. 메모리 쓰기 자체는 빠르지만, 단일 스레드 + 명령어별 처리 구조상 수십만~수백만 ops/sec가 한계입니다.

Kafka는 디스크에 순차적으로 씁니다. 순차 디스크 쓰기는 메모리의 랜덤 쓰기보다 느리지만, Kafka는 OS 페이지 캐시에 쓰고(실제로는 메모리), OS가 비동기로 디스크에 flush합니다. 따라서 Producer 입장에서 실제 쓰기는 메모리 속도입니다. 게다가 Producer가 배치로 여러 메시지를 한 번에 전송하므로 처리량이 높습니다.

결론: Kafka의 쓰기는 실질적으로 OS 페이지 캐시(메모리) 쓰기입니다. 순차 append 방식이라 디스크 seek 없고, 대용량 배치로 네트워크 왕복도 줄입니다.

</details>

---

**Q2. `.index` 파일이 모든 offset을 저장하지 않고 "Sparse Index"를 사용하는 이유는 무엇인가요?**

<details>
<summary>해설 보기</summary>

모든 offset을 인덱싱하면 인덱스 파일 크기가 .log 파일 크기와 비례해서 커집니다. 파티션당 수 GB의 .log 파일이 있을 때 dense index는 수백 MB의 인덱스 파일을 생성합니다.

Sparse Index는 `index.interval.bytes`(기본 4096 bytes)마다 하나의 항목만 저장합니다. 4096 bytes = 약 4개의 메시지(1 KB 메시지 기준)마다 인덱스 항목 1개. 이로 인해 인덱스 파일 크기는 .log 파일의 약 1/4000 수준으로 작습니다.

탐색 시에는 인덱스에서 목표 offset보다 작은 최대값을 이진 탐색으로 찾고, 해당 파일 위치부터 순차 스캔합니다. 최악의 경우 4096 bytes를 스캔해야 하지만, 이는 SSD에서 수 μs 수준으로 무시할 만한 비용입니다. 인덱스 파일이 작아서 전체가 OS 페이지 캐시에 상주하므로 탐색 자체도 메모리 속도로 동작합니다.

</details>

---

**Q3. Active 세그먼트는 retention 기간이 지나도 삭제되지 않습니다. 이것이 실제 운영에서 어떤 문제를 일으킬 수 있나요?**

<details>
<summary>해설 보기</summary>

처리량이 낮은 토픽에서 디스크 사용량이 예상보다 많을 수 있습니다.

예시: `retention.ms=86400000`(1일), `log.segment.bytes=1073741824`(1 GB), 토픽 유입: 하루 100 MB

이 경우 Active 세그먼트가 1 GB를 채우는 데 10일이 걸립니다. 10일이 지나야 롤링 → Inactive → retention 체크 → 삭제. 그 동안 1개의 Active 세그먼트(최대 1 GB) + 이미 삭제된 Inactive 세그먼트들이 공존합니다.

더 심각한 경우: 토픽이 거의 사용되지 않으면 Active 세그먼트가 수십 일 동안 롤링되지 않아 오래된 데이터가 계속 보존됩니다.

해결책: `log.roll.ms=86400000`(1일)을 설정해서 크기 미달이어도 1일마다 강제 롤링. 이렇게 하면 retention이 의도한 대로 동작합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 토픽과 파티션](./02-topic-partition.md)** | **[홈으로 🏠](../README.md)** | **[다음: Producer 내부 동작 ➡️](./04-producer-internals.md)**

</div>
