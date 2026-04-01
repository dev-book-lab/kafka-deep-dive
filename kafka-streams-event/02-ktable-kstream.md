# KTable과 KStream — 스냅샷 vs 이벤트 스트림

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- KStream과 KTable의 근본적인 개념 차이는 무엇인가?
- KStream-KTable 조인이 동작하는 원리는?
- 동일한 토픽을 KStream으로 읽을 때와 KTable로 읽을 때 동작이 어떻게 다른가?
- GlobalKTable이 일반 KTable과 다른 점은 무엇이고, 언제 사용하는가?
- `groupByKey()`와 `groupBy()`의 차이는?
- KTable에서 같은 키로 새 값이 오면 어떻게 처리되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

KStream과 KTable의 차이를 모르면 조인 결과가 예상과 다르거나 메모리/디스크 사용량이 폭발할 수 있다.

예: 사용자 프로필 업데이트 이벤트를 KStream으로 처리하면 이전 프로필과 새 프로필 "모두"를 처리 대상으로 간주한다. KTable로 처리하면 최신 프로필만 유지한다. 이 차이가 조인 결과와 집계 결과를 완전히 바꾼다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 사용자 정보 토픽을 KStream으로 조인

  상황: 주문 스트림을 사용자 정보와 조인해서 이름 enrichment
  
  잘못된 코드:
    KStream<String, Order> orders = builder.stream("orders");
    KStream<String, User> users = builder.stream("users");
    orders.join(users, ...)  // KStream-KStream 조인

  문제:
    users 토픽에 user-1의 프로필 업데이트가 10번 발생했다면
    KStream은 10개의 업데이트 이벤트를 모두 처리
    join 결과: order-1이 user-1의 10개 프로필 버전과 10번 조인
    → 10개의 결과 메시지 발행 (의도치 않은 중복)

  올바른 방법:
    KTable<String, User> users = builder.table("users");
    orders.join(users, ...)  // KStream-KTable 조인
    → user-1의 최신 프로필 1개와만 조인 → 결과 1개

실수 2: GlobalKTable이 메모리를 과점유

  상황: 상품 카탈로그(10만 개 항목)를 GlobalKTable로 로드
  GlobalKTable 특성: 모든 파티션의 데이터를 모든 인스턴스에 로드
  
  문제:
    인스턴스 5개 × 10만 항목 × 1 KB = 500 MB 메모리 × 5 = 2.5 GB
    상품이 100만 개면 25 GB → OOM

  올바른 판단:
    작은 룩업 테이블 (수만 개 이하): GlobalKTable 적합
    대용량 상태 (수백만 개 이상): 일반 KTable + co-partitioning

실수 3: KTable 업데이트를 개별 이벤트로 오해

  상황: user-1의 프로필을 5번 업데이트 후 집계
  
  KTable 동작:
    이벤트 1: key=user-1, value={name:Alice}    → KTable[user-1]={name:Alice}
    이벤트 2: key=user-1, value={name:Alice2}   → KTable[user-1]={name:Alice2}
    이벤트 3: key=user-1, value={name:Alice Kim} → KTable[user-1]={name:Alice Kim}
    
    KTable은 5개 이벤트를 모두 처리하지만
    결과로 저장되는 것은 최신값 1개 (name:Alice Kim)
    count()나 aggregate() 시 5번 업데이트되지만 최종 상태는 1개

  KStream 동작:
    5개 이벤트 모두 독립적으로 처리 (이력 유지)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
KStream vs KTable 선택 기준:

  KStream을 선택하는 경우:
    ✅ 이벤트 이력이 중요 (클릭, 구매, 로그)
    ✅ 모든 이벤트를 개별적으로 처리해야 함
    ✅ 이벤트 발생 순서와 시점이 중요
    예: 클릭스트림 분석, 트랜잭션 로그

  KTable을 선택하는 경우:
    ✅ 현재 상태만 중요 (사용자 프로필, 상품 가격, 설정)
    ✅ 같은 키의 이전 값을 새 값으로 대체
    ✅ 최신 상태 기반 조인이 필요
    예: 사용자 정보 enrichment, 실시간 설정 조회

  GlobalKTable 선택 기준:
    ✅ 데이터 크기가 작음 (수만 건 이하)
    ✅ 모든 파티션 데이터에 파티션 무관하게 접근 필요
    ✅ 국가 코드, 카테고리 같은 참조 데이터
    ❌ 수백만 건 이상 → 일반 KTable 사용
```

---

## 🔬 내부 동작 원리

### 1. KStream: 이벤트의 무한 흐름

```
KStream 토픽 해석:

  토픽: "clicks"
  이벤트:
    offset 0: key=user-1, value={page:/home, ts:10:00}
    offset 1: key=user-1, value={page:/product, ts:10:01}
    offset 2: key=user-2, value={page:/home, ts:10:02}
    offset 3: key=user-1, value={page:/cart, ts:10:03}

  KStream으로 읽으면:
    4개의 독립적인 이벤트 (user-1이 3번 클릭한 이력)
    각 이벤트는 고유한 사건

  집계 (count by userId):
    user-1: 3번 클릭
    user-2: 1번 클릭

  핵심 속성:
    - 동일 키의 이전 이벤트를 대체하지 않음
    - 모든 이벤트가 보존
    - 무한 스트림 (끝이 없음)
```

### 2. KTable: 키별 최신 상태 스냅샷

```
KTable 토픽 해석:

  토픽: "user-profiles"
  이벤트:
    offset 0: key=user-1, value={name:Alice, age:25}
    offset 1: key=user-2, value={name:Bob, age:30}
    offset 2: key=user-1, value={name:Alice, age:26}   ← update
    offset 3: key=user-1, value={name:Alice Kim, age:26} ← update

  KTable로 읽으면:
    현재 상태:
      user-1: {name:Alice Kim, age:26}   ← 최신값
      user-2: {name:Bob, age:30}

  처리 과정:
    offset 0 → KTable[user-1] = {name:Alice, age:25}
               다운스트림에 emit: key=user-1, NEW={age:25}
    offset 2 → KTable[user-1] = {name:Alice, age:26}
               다운스트림에 emit: key=user-1, OLD={age:25}, NEW={age:26}
    offset 3 → KTable[user-1] = {name:Alice Kim, age:26}

  삭제: key에 value=null 전송 → KTable에서 해당 키 삭제 (Tombstone)
```

### 3. KStream-KTable 조인 원리

```
조인 동작 (이벤트 기반 + 현재 상태):

  KStream: "orders" 토픽
    t=10:00 → order-1, userId=user-1, amount=100
    t=10:05 → order-2, userId=user-2, amount=200
    t=10:10 → order-3, userId=user-1, amount=300

  KTable: "user-profiles" 토픽 (현재 상태)
    t=09:00 → user-1: {name:Alice, tier:Silver}
    t=10:08 → user-1: {name:Alice Kim, tier:Gold}  ← 업데이트

  orders.join(userProfiles, ...)로 조인:
    t=10:00 → order-1 (userId=user-1) JOIN KTable[user-1]
               KTable 현재값: {name:Alice, tier:Silver}
               결과: {order:order-1, user:Alice, tier:Silver}

    t=10:10 → order-3 (userId=user-1) JOIN KTable[user-1]
               KTable 현재값: {name:Alice Kim, tier:Gold} (10:08 업데이트)
               결과: {order:order-3, user:Alice Kim, tier:Gold}

  핵심: 조인 시점의 KTable 현재 상태와 결합
        KTable이 업데이트되면 그 이후 도착하는 KStream 이벤트는 새 상태와 조인
        동일 스트림 이벤트가 시점에 따라 다른 KTable 값과 조인

  leftJoin: KTable에 키 없어도 결과 생성 (user 없으면 null)
  join:     KTable에 키 있어야 결과 생성 (user 없으면 결과 없음)
```

### 4. GlobalKTable: 모든 파티션 로컬 유지

```
일반 KTable vs GlobalKTable:

  일반 KTable:
    파티션별로 분산 (co-partitioning 필요)
    Task 0: user-1~3 상태
    Task 1: user-4~6 상태
    Task 2: user-7~9 상태
    
    조인 제약: orders 파티션 0의 user-1 이벤트는
               user-profiles 파티션 0의 user-1 상태와만 조인 가능
               → co-partitioning 필수 (동일 파티션 키 전략)

  GlobalKTable:
    모든 파티션 데이터를 모든 인스턴스에 로드
    Instance 1: user-1~9 상태 (전체)
    Instance 2: user-1~9 상태 (전체 복사본)
    
    조인 장점: 파티션 무관하게 어떤 userId로도 조인 가능
    → 국가 코드 → 국가명 변환, 상품 ID → 카테고리 변환 등

  GlobalKTable 단점:
    모든 파티션 데이터를 로컬에 저장 → 인스턴스당 전체 데이터 크기
    Changelog 없이 직접 토픽을 읽어서 RocksDB 구성
    → 인스턴스 재시작 시 전체 토픽 재생 (느린 시작)
```

### 5. groupByKey vs groupBy

```
groupByKey():
  기존 키를 유지하며 그룹화
  repartitioning 없음 (네트워크 전송 없음)
  
  사용: 메시지 키 = 그룹화 키인 경우

groupBy():
  새로운 키로 그룹화 (키 변환)
  repartitioning 발생: 새 키 기준으로 파티션 재배분
  내부: Repartition 토픽 자동 생성 → 새 키로 재발행 → 다시 읽기
  
  사용: 메시지 키 ≠ 그룹화 키인 경우
  비용: 추가 토픽 + 추가 Kafka 왕복

예시:
  // orders 토픽의 키는 orderId
  // userId 기준으로 집계하고 싶음
  
  orders.groupBy((orderId, order) -> order.getUserId())  // groupBy = repartition
        .count()  // userId별 주문 수

  // vs 이미 키가 userId라면
  orders.groupByKey()  // repartition 없음
        .count()
```

---

## 💻 실전 실험

### 실험 1: KStream vs KTable 동작 차이 확인

```java
// 동일 토픽을 KStream과 KTable로 읽기
StreamsBuilder builder = new StreamsBuilder();

// KStream: 모든 이벤트 출력
KStream<String, String> stream = builder.stream("updates");
stream.foreach((key, value) ->
    System.out.println("KStream: key=" + key + " value=" + value));

// KTable: 최신 상태만 유지
KTable<String, String> table = builder.table("updates");
table.toStream()
     .foreach((key, value) ->
         System.out.println("KTable update: key=" + key + " value=" + value));

// 동일 키로 3번 업데이트 발행:
// key=user-1 value=v1
// key=user-1 value=v2
// key=user-1 value=v3

// KStream 출력: v1, v2, v3 (3개)
// KTable 출력: v1, v2, v3 (업데이트마다 emit하지만 최종 상태는 v3)
```

### 실험 2: KStream-KTable 조인 실습

```bash
# 토픽 생성
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic orders --partitions 3 --replication-factor 1
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic user-profiles --partitions 3 --replication-factor 1

# 사용자 프로필 발행 (KTable)
echo "user-1:{name:Alice,tier:Gold}" | kafka-console-producer \
  --bootstrap-server localhost:9092 --topic user-profiles \
  --property "parse.key=true" --property "key.separator=:"

# 주문 발행 (KStream) - 조인 결과 확인
echo "order-1:{userId:user-1,amount:100}" | kafka-console-producer \
  --bootstrap-server localhost:9092 --topic orders \
  --property "parse.key=true" --property "key.separator=:"
```

### 실험 3: GlobalKTable 로드 시간 측정

```java
// GlobalKTable 로드 시간 확인
long start = System.currentTimeMillis();
KafkaStreams streams = new KafkaStreams(topology, config);
streams.start();

// GlobalKTable 완전 로드 대기
while (streams.state() != KafkaStreams.State.RUNNING) {
    Thread.sleep(100);
}
long elapsed = System.currentTimeMillis() - start;
System.out.println("GlobalKTable 로드 시간: " + elapsed + "ms");

// 토픽 크기에 따라 수 초 ~ 수 분 소요
// → 대용량 GlobalKTable의 시작 시간 영향 확인
```

---

## 📊 성능/비용 비교

### KStream vs KTable vs GlobalKTable 비교

```
조건: 1백만 개 키, 각 키당 100byte 상태

  KTable (파티션 3개):
    인스턴스당 상태 크기: ~33 MB (1/3 파티션)
    조인 제약: co-partitioning 필요
    repartition: groupBy 사용 시 추가 토픽

  GlobalKTable:
    인스턴스당 상태 크기: ~100 MB (전체)
    조인 제약: 없음 (어떤 키도 조인 가능)
    시작 시간: 전체 토픽 재생 → 느림
    
  1,000만 개 키:
    KTable 인스턴스당: ~333 MB
    GlobalKTable 인스턴스당: ~1 GB → 주의 필요
    
  1억 개 키:
    KTable: 3.3 GB/인스턴스 (파티션 3개 기준)
    GlobalKTable: 10 GB/인스턴스 → 사실상 불가
```

---

## ⚖️ 트레이드오프

```
KStream:
  ✅ 이벤트 이력 완전 보존
  ✅ 모든 이벤트 개별 처리
  ❌ 같은 키의 과거 상태 직접 조회 불가 (집계 필요)
  ❌ 대용량 상태 유지 시 join 연산 복잡

KTable:
  ✅ 최신 상태 즉시 조회
  ✅ 이벤트 스트림과 자연스러운 조인
  ❌ 이력 정보 유지 안 됨 (변경 이력 필요 시 KStream 활용)
  ❌ co-partitioning 제약 (동일 파티션 전략 필요)

GlobalKTable:
  ✅ co-partitioning 제약 없음
  ✅ 소규모 참조 데이터 enrichment에 이상적
  ❌ 대용량 데이터에 OOM 위험
  ❌ 인스턴스 시작 시 전체 토픽 재로드 → 느린 시작

groupBy vs groupByKey:
  groupByKey: ✅ 빠름, repartition 없음 ❌ 키가 그룹화 기준이어야 함
  groupBy:    ✅ 유연한 그룹화 ❌ repartition 추가 토픽/지연 비용
```

---

## 📌 핵심 정리

```
KStream vs KTable 핵심:

1. KStream = 이벤트의 무한 흐름
   모든 이벤트 개별 처리
   동일 키의 이전 값을 대체하지 않음

2. KTable = 키별 최신 상태 스냅샷
   동일 키의 새 값이 오면 이전 값 대체
   현재 상태 기반 조인에 사용

3. KStream-KTable 조인:
   이벤트 도착 시점의 KTable 현재 상태와 결합
   KTable 업데이트 후 도착하는 이벤트는 새 상태와 조인

4. GlobalKTable:
   모든 파티션 데이터를 모든 인스턴스에 로드
   co-partitioning 불필요
   소규모 참조 데이터에만 사용 (수만 개 이하)

5. groupBy vs groupByKey:
   groupByKey: 현재 키로 그룹화 (repartition 없음)
   groupBy: 새 키로 그룹화 (repartition 발생, 추가 비용)
```

---

## 🤔 생각해볼 문제

**Q1. KTable에서 Tombstone(value=null)을 전송하면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

해당 키가 KTable에서 삭제됩니다. KTable의 상태 저장소(RocksDB)에서 해당 키가 제거되고, 이후 그 키로 조인을 시도하면 null이 반환됩니다(leftJoin의 경우) 또는 결과가 없습니다(inner join의 경우).

Tombstone은 KTable의 `cleanup.policy=compact` 토픽에서도 의미가 있습니다. Compaction 후 해당 키의 Tombstone만 남고, `delete.retention.ms` 이후에 Tombstone 자체도 삭제됩니다.

실무 예: 사용자 탈퇴 시 user-profiles 토픽에 `key=userId, value=null` 발행 → KTable에서 해당 사용자 삭제 → 이후 그 사용자의 주문 이벤트는 조인 결과 없음.

</details>

---

**Q2. KStream-KTable 조인에서 KTable이 아직 로드되지 않았을 때 KStream 이벤트가 도착하면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

KTable이 아직 데이터를 로드 중이면 해당 키가 KTable에 없는 것으로 간주합니다. inner join이면 결과가 없고, left join이면 null과 조인된 결과가 나옵니다.

이것이 Kafka Streams 시작 시 중요한 타이밍 이슈입니다. 애플리케이션을 처음 시작하거나 재시작할 때 KTable이 Changelog에서 복원되는 동안 KStream 이벤트가 계속 들어올 수 있습니다. 이 기간 동안 잘못된 조인 결과가 발생할 수 있습니다.

해결책: `KafkaStreams.localThreadsMetadata()`나 `state()` 확인으로 `RUNNING` 상태가 될 때까지 외부 트래픽 차단 후 Kafka Streams를 활성화합니다. 또는 Standby Replica와 `num.standby.replicas`를 설정해서 복원 시간을 최소화합니다.

</details>

---

**Q3. 같은 Kafka Streams 애플리케이션에서 KStream과 KTable을 동시에 같은 토픽에서 만들 수 있나요?**

<details>
<summary>해설 보기</summary>

기술적으로는 가능하지만 권장하지 않습니다. 동일한 토픽을 동시에 KStream과 KTable로 읽으면 두 개의 독립적인 Consumer 구독이 생성되고, 각각 독립적으로 처리됩니다. 상태 저장소도 별도로 유지됩니다.

실무에서 같은 토픽을 두 방식으로 읽어야 할 때는:
- 이벤트 이력 처리 → KStream
- 현재 상태 조인 → KTable

이 두 요구사항이 동시에 필요하다면 KTable의 `.toStream()`으로 변환해서 사용하거나, 토픽을 분리하는 것이 더 명확한 설계입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Kafka Streams 아키텍처](./01-kafka-streams-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: 윈도우 연산 ➡️](./03-window-operations.md)**

</div>
