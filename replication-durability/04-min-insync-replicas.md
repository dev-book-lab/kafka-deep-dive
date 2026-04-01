# min.insync.replicas — 가용성과 내구성의 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `min.insync.replicas`가 정확히 어떤 조건에서 쓰기를 거부하는가?
- `replication.factor=3, min.insync.replicas=2, acks=all`을 황금 조합이라 부르는 근거는?
- 브로커 1대가 장애날 때 이 조합에서 어떤 일이 발생하는가?
- `min.insync.replicas`를 브로커 수준으로 설정하는 것과 토픽 수준으로 설정하는 차이는?
- `NotEnoughReplicasException`이 발생하면 Producer는 어떻게 반응하는가?
- 가용성과 내구성이 서로 반비례하는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`acks=all`은 "현재 ISR의 모든 멤버에 기록"을 보장하지만, ISR이 1개로 줄면 보장의 의미가 약해진다. `min.insync.replicas`는 "ISR이 최소 N개 이상이어야 쓰기를 허용"하는 안전장치다.

이 두 설정의 관계를 모르면:
- `acks=all`로 설정했으니 안전하다고 오해
- `min.insync.replicas=1`(브로커 기본값)이면 ISR=1에서도 쓰기 성공
- 장애 상황에서 서비스가 "가용하지만 데이터 유실 중"인 최악의 상태가 됨

올바른 이해: **내구성을 위해 가용성을 일부 포기하는 것이 `min.insync.replicas`의 역할**

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: min.insync.replicas 없이 acks=all만 설정

  설정:
    replication.factor=3
    acks=all
    min.insync.replicas=1 (브로커 기본값)

  상황: Broker 2, 3 장애 → ISR=[1]
  결과: acks=all이지만 ISR=1이므로 Broker 1에만 기록하면 성공
        Broker 1 추가 장애 시 → 데이터 유실

  올바른 설정: min.insync.replicas=2 필수

실수 2: min.insync.replicas = replication.factor로 설정

  설정:
    replication.factor=3
    min.insync.replicas=3
    acks=all

  결과: 브로커 1대만 장애나도
        ISR=2 < min.insync.replicas=3
        → 즉시 NotEnoughReplicasException
        → 서비스 쓰기 중단
  
  문제: 운영 환경에서 브로커 1대 장애는 흔한 일
        rolling upgrade, 장비 교체 등
        이 설정이면 단순 재시작도 서비스 중단 유발

  올바른 설정: min.insync.replicas = replication.factor - 1

실수 3: 토픽 수준 vs 브로커 수준 설정 혼동

  브로커 기본값: min.insync.replicas=1
  토픽별 설정: 토픽마다 다르게 적용 가능

  실수: 브로커 설정은 높게 했는데 토픽 생성 시 override를 낮게 설정
        또는 브로커 설정은 낮고 중요 토픽에 높게 설정 안 함
  
  확인 방법:
    kafka-configs --bootstrap-server localhost:9092 \
      --entity-type topics --entity-name orders \
      --describe
    # min.insync.replicas 항목 확인
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
황금 조합과 그 근거:
  replication.factor=3    → 복제본 3개
  min.insync.replicas=2   → ISR 2개 이상이어야 쓰기 허용
  acks=all                → ISR 전체 기록 완료 후 응답

  브로커 1대 장애 시:
    ISR=2 >= min.insync.replicas=2 → 쓰기 계속 가능
    2개 브로커에 데이터 보존 → 내구성 유지

  브로커 2대 동시 장애 시:
    ISR=1 < min.insync.replicas=2 → 쓰기 거부
    서비스 중단이지만 데이터 유실 없음
    → 복구 후 무결성 있는 데이터 보장

토픽별 중요도에 따른 설정:
  # 결제/주문 토픽 (최고 내구성)
  kafka-configs --bootstrap-server localhost:9092 \
    --entity-type topics --entity-name payments \
    --alter --add-config min.insync.replicas=2

  # 로그 수집 토픽 (내구성보다 가용성)
  kafka-configs --bootstrap-server localhost:9092 \
    --entity-type topics --entity-name app-logs \
    --alter --add-config min.insync.replicas=1
```

---

## 🔬 내부 동작 원리

### 1. min.insync.replicas 작동 조건

```
적용 조건: acks=all (또는 acks=-1) 일 때만 적용
           acks=0 또는 acks=1이면 min.insync.replicas 무시

동작 흐름:
  Producer → acks=all → Leader에 쓰기 요청

  Leader 처리 순서:
  1. 현재 ISR 크기 확인
  2. ISR 크기 >= min.insync.replicas?
     YES: 정상 처리 (ISR 전체 복제 대기 후 응답)
     NO:  즉시 NotEnoughReplicasException 반환

  ┌──────────────────────────────────────────────────┐
  │                  Leader 로직                      │
  │                                                  │
  │  if (isr.size() < minInsyncReplicas) {           │
  │    throw new NotEnoughReplicasException(         │
  │      "Number of insync replicas for partition " +│
  │      partition + " is [" + isr.size() + "]," +   │
  │      " below required minimum [" +               │
  │      minInsyncReplicas + "]"                     │
  │    );                                            │
  │  }                                               │
  └──────────────────────────────────────────────────┘

  예외 수신 시 Producer 동작:
    retries 설정에 따라 재시도
    기본: Integer.MAX_VALUE (무한 재시도)
    delivery.timeout.ms(기본 120초) 내에 성공 못하면 최종 실패
```

### 2. 브로커 1대 장애 시나리오별 분석

```
설정: replication.factor=3, min.insync.replicas=2, acks=all

  시나리오 A: Broker 3 정상 종료 (rolling restart)
    단계:
    1. 관리자가 kafka-preferred-replica-election 실행 (옵션)
    2. Broker 3 셧다운 신호
    3. Broker 3이 ISR에서 정상 이탈
       ISR: [1,2,3] → [1,2]
    4. ISR=2 >= min.insync.replicas=2 → 쓰기 계속 허용
    5. Broker 3 재시작 → 복제 동기화 → ISR: [1,2,3] 복귀
    결과: 서비스 중단 없음, 데이터 유실 없음 ✅

  시나리오 B: Broker 3 갑작스러운 장애
    단계:
    1. Broker 3 갑자기 다운
    2. replica.lag.time.max.ms(30초) 후 Leader가 ISR에서 제거
       ISR: [1,2,3] → [1,2]
    3. ISR=2 >= min.insync.replicas=2 → 쓰기 계속 허용
    4. Broker 3 복구 → ISR 복귀
    결과: 30초 동안 acks=all이지만 2개 브로커에만 기록
          (데이터 내구성 일시 약화이지만 여전히 2개 보유)

  시나리오 C: Broker 2, 3 동시 장애
    단계:
    1. Broker 2, 3 다운
    2. 30초 후 ISR: [1]
    3. ISR=1 < min.insync.replicas=2 → 쓰기 거부!
       NotEnoughReplicasException
    결과: 쓰기 서비스 중단 (가용성 저하)
          데이터는 Broker 1에 보존 (내구성 유지)
```

### 3. min.insync.replicas 설정 위치와 우선순위

```
설정 위치:
  1. 브로커 레벨 (server.properties):
     min.insync.replicas=1  ← 모든 토픽에 적용되는 기본값
  
  2. 토픽 레벨 (토픽별 override):
     kafka-configs --alter --add-config min.insync.replicas=2
     → 해당 토픽에만 적용, 브로커 설정 override

우선순위: 토픽 설정 > 브로커 설정

현재 설정 확인:
  # 토픽 설정 확인
  kafka-configs --bootstrap-server localhost:9092 \
    --entity-type topics --entity-name orders --describe
  # Dynamic configs for topic 'orders':
  # min.insync.replicas=2  (sensitiveConfigs: false)

  # 브로커 기본값 확인
  kafka-configs --bootstrap-server localhost:9092 \
    --entity-type brokers --entity-name 1 --describe
  # min.insync.replicas=1

실무 권장:
  브로커 기본값: min.insync.replicas=1 (낮게 유지, 오래된 토픽 영향 최소화)
  운영 토픽 생성 시: --config min.insync.replicas=2 명시적 설정
  또는 토픽 생성 자동화 스크립트에서 항상 포함
```

### 4. NotEnoughReplicasException 핸들링

```
Producer 입장에서 NotEnoughReplicasException:

  기본 동작 (retries=Integer.MAX_VALUE):
    브로커로부터 NotEnoughReplicasException 수신
    Producer가 retry.backoff.ms(기본 100ms) 대기
    재시도 → 또 실패 → 다시 대기 ...
    delivery.timeout.ms(기본 120초) 초과
    → TimeoutException을 콜백/Future에 전달

  Spring Kafka 처리:
    @KafkaListener의 Producer 쪽에서 발생 시:
      KafkaTemplate.send()의 Future에서 ExecutionException 발생
      원인 예외: KafkaException > NotEnoughReplicasException

  운영 대응:
    1. 알람: NotEnoughReplicasException 발생 → Slack/PagerDuty
    2. ISR 상태 즉시 확인: kafka-topics --under-replicated-partitions
    3. 장애 브로커 복구 또는 min.insync.replicas 임시 낮추기 (비상 조치)
       → 임시 조치 후 복구 즉시 원복 필수
```

---

## 💻 실전 실험

### 실험 1: 황금 조합 설정 및 검증

```bash
# 황금 조합으로 토픽 생성
kafka-topics --bootstrap-server localhost:19092 \
  --create --topic critical-orders \
  --partitions 3 \
  --replication-factor 3 \
  --config min.insync.replicas=2

# 설정 확인
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics --entity-name critical-orders \
  --describe
# min.insync.replicas=2 확인

# 정상 쓰기 확인 (ISR=3)
kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic critical-orders \
  --producer-property acks=all
# 메시지 입력 → 성공
```

### 실험 2: ISR 부족 시 쓰기 거부 확인

```bash
# Broker 2, 3 중단
docker stop kafka-2 kafka-3
sleep 35  # ISR 이탈 대기

# ISR 상태 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --topic critical-orders
# Isr: 1  (ISR=1 < min.insync.replicas=2)

# acks=all로 쓰기 시도
kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic critical-orders \
  --producer-property acks=all \
  --producer-property request.timeout.ms=5000 \
  --producer-property delivery.timeout.ms=10000
# 메시지 입력 → ERROR: Not enough replicas

# acks=1로는 여전히 쓰기 가능 (min.insync.replicas는 acks=all에만 적용)
kafka-console-producer \
  --bootstrap-server localhost:19092 \
  --topic critical-orders \
  --producer-property acks=1
# 메시지 입력 → 성공 (하지만 내구성 약화 상태)
```

### 실험 3: 토픽별 min.insync.replicas 동적 변경

```bash
# 긴급 상황에서 min.insync.replicas 임시 낮추기
kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics --entity-name critical-orders \
  --alter --add-config min.insync.replicas=1
# → 비상 조치: ISR=1이어도 쓰기 허용

# 브로커 복구 후 즉시 원복
docker start kafka-2 kafka-3
sleep 30  # ISR 복귀 대기

kafka-configs --bootstrap-server localhost:19092 \
  --entity-type topics --entity-name critical-orders \
  --alter --add-config min.insync.replicas=2
# → 원상 복구
```

---

## 📊 성능/비용 비교

### 설정 조합별 시나리오 분석

```
브로커 3대, 파티션 1개 기준:

  A. replication.factor=3, min.insync.replicas=1, acks=all
     브로커 1대 장애: ISR=2 >= 1 → 쓰기 가능, 2개에 저장
     브로커 2대 장애: ISR=1 >= 1 → 쓰기 가능, 1개에 저장 (위험!)
     가용성: 최고  내구성: 낮음

  B. replication.factor=3, min.insync.replicas=2, acks=all (황금 조합)
     브로커 1대 장애: ISR=2 >= 2 → 쓰기 가능, 2개에 저장
     브로커 2대 장애: ISR=1 < 2  → 쓰기 거부! (서비스 중단)
     가용성: 중간  내구성: 높음

  C. replication.factor=3, min.insync.replicas=3, acks=all
     브로커 1대 장애: ISR=2 < 3  → 즉시 쓰기 거부!
     가용성: 낮음  내구성: 최고 (불실용적)

권장: 설정 B (황금 조합)
  브로커 1대까지는 투명하게 처리
  브로커 2대 동시 장애(드문 케이스)에서만 서비스 중단
```

---

## ⚖️ 트레이드오프

```
min.insync.replicas 높게:
  ✅ 더 강한 내구성 보장
  ❌ ISR 이탈 시 더 빠른 쓰기 거부 → 가용성 저하
  ❌ 브로커 유지보수(rolling restart) 중 서비스 영향

min.insync.replicas 낮게(1):
  ✅ ISR 이탈에도 쓰기 가능 → 높은 가용성
  ❌ ISR=1 상태에서 브로커 추가 장애 → 데이터 유실 위험
  ❌ "가용하지만 내구성 없는" 최악의 상태 가능

내구성 vs 가용성의 본질적 트레이드오프:
  "데이터를 더 안전하게 저장하려면, 안전하지 않을 때 쓰기를 거부해야 한다"
  CAP 정리에서 CP(일관성+분산내결함성) 선택에 해당
  Kafka는 기본적으로 PA(가용성+분산내결함성) 설정이지만
  min.insync.replicas=2로 PC(일관성 강화) 방향으로 이동
```

---

## 📌 핵심 정리

```
min.insync.replicas 핵심:

1. 작동 조건: acks=all일 때만 적용
   acks=0, acks=1이면 무시됨

2. ISR < min.insync.replicas 시 NotEnoughReplicasException
   쓰기 거부 = 데이터 유실 방지를 위한 의도적 가용성 희생

3. 황금 조합: replication.factor=3, min.insync.replicas=2, acks=all
   브로커 1대 장애까지 쓰기 가능 + 데이터 2개 이상에 보존
   브로커 2대 동시 장애 시 쓰기 중단 (but 데이터 유실 없음)

4. 토픽별 설정으로 중요도에 따라 차등 적용
   결제/주문: min.insync.replicas=2
   로그 수집: min.insync.replicas=1 (가용성 우선)

5. 브로커 기본값(min.insync.replicas=1)에 의존하지 말 것
   중요 토픽 생성 시 명시적으로 설정
```

---

## 🤔 생각해볼 문제

**Q1. `min.insync.replicas=2`인데 브로커가 2대뿐이라면 어떤 문제가 생기나요?**

<details>
<summary>해설 보기</summary>

안전 마진이 사라집니다. `replication.factor=2, min.insync.replicas=2`면 두 브로커 모두 ISR에 있어야 쓰기가 가능합니다. 브로커 1대만 장애나도 ISR=1 < 2가 되어 즉시 쓰기 중단이 됩니다.

Rolling restart(버전 업그레이드, 설정 변경)를 할 때도 브로커 1대씩 재시작하는 동안 쓰기가 중단됩니다. 운영 환경에서는 replication.factor=3, 브로커 최소 3대, min.insync.replicas=2를 권장하는 이유입니다.

</details>

---

**Q2. `NotEnoughReplicasException` 발생 시 이미 전송 중인 배치는 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

전송 중인 배치는 실패로 처리됩니다. Producer는 `retries` 설정에 따라 재시도하고, `delivery.timeout.ms` 동안 계속 재시도합니다. ISR이 회복되면(장애 브로커 복구) 재시도가 성공합니다.

`enable.idempotence=true`이면 재시도 중 동일 배치가 중복으로 기록되지 않습니다. Producer ID + Sequence Number로 브로커가 중복을 감지하고 거부합니다.

Spring Kafka에서는 `KafkaTemplate.send()`의 `ListenableFuture`(또는 `CompletableFuture`)에서 예외가 발생합니다. `@KafkaListener`의 메시지 처리 중 Producer로 보내는 경우에는 해당 리스너 메서드가 예외를 받아 처리해야 합니다.

</details>

---

**Q3. 가용성보다 내구성을 선택하는 것이 비즈니스적으로 옳은 경우는 언제인가요?**

<details>
<summary>해설 보기</summary>

서비스 중단보다 데이터 유실이 더 심각한 결과를 낳는 경우입니다.

결제 시스템: 쓰기가 잠깐 중단되면 "잠시 후 다시 시도" 안내로 대처 가능하지만, 결제 완료 기록이 사라지면 재정적/법적 문제가 발생합니다. 쓰기 거부가 훨씬 낫습니다.

의료 기록: 데이터 유실은 생명에 영향을 줄 수 있습니다. 잠시 쓰기 불가가 훨씬 안전합니다.

반면 가용성이 더 중요한 경우: 실시간 사용자 행동 로그, A/B 테스트 이벤트, 광고 클릭 로그 등은 일부 유실이 있어도 통계적으로 의미있는 분석이 가능하므로 가용성을 선택합니다. 이 경우 `min.insync.replicas=1`이 합리적입니다.

</details>

---

<div align="center">

**[⬅️ 이전: acks 설정 완전 분해](./03-acks-deep-dive.md)** | **[홈으로 🏠](../README.md)** | **[다음: Leader Election ➡️](./05-leader-election.md)**

</div>
