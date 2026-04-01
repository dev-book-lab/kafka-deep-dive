# 윈도우 연산 — 시간 기반 집계와 지연 이벤트

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Tumbling / Hopping / Sliding / Session Window의 차이는?
- Event Time과 Processing Time 중 어떤 기준을 언제 사용해야 하는가?
- `grace period`가 지연 이벤트(Late Arrival)를 수용하는 원리는?
- 윈도우 집계 결과가 여러 번 발행되는 이유는 무엇인가?
- `suppress()`가 최종 결과만 발행하도록 하는 원리는?
- Session Window에서 비활동 간격(inactivity gap)은 어떻게 결정되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"1분 동안 발생한 주문 수"를 집계할 때 단순히 처리 시간(Processing Time)으로 1분을 자르면 안 된다. 네트워크 지연, 모바일 오프라인 등으로 이벤트가 늦게 도착할 때 실제 발생 시각(Event Time) 기준으로 정확한 윈도우에 포함해야 한다.

윈도우 연산을 모르면:
- 1분 창에 59초에 발생한 이벤트가 60초 늦게 도착해서 다음 창에 잘못 집계
- Hopping Window를 써야 하는데 Tumbling Window를 써서 겹치는 집계 불가
- Session Window 대신 Tumbling Window를 써서 사용자 세션이 임의로 끊김

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Processing Time 기준으로 정확한 집계를 기대

  코드:
    TimeWindows.of(Duration.ofMinutes(1))  // Processing Time 기준

  문제 시나리오:
    실제 이벤트 발생: 11:59:50 (1분 윈도우 마지막 10초)
    네트워크 지연: 20초
    Kafka 도착: 12:00:10 (다음 1분 윈도우에 포함!)
    
    결과: 11:59:50에 발생한 이벤트가 12:00~12:01 윈도우로 집계
    → 시간당 정확한 집계가 필요한 결제/매출 보고서에서 부정확

  올바른 방법:
    Event Time 기준 + grace period 설정
    TimeWindows.of(Duration.ofMinutes(1))
               .grace(Duration.ofSeconds(30))
    // 윈도우 종료 후 30초까지 늦게 도착한 이벤트도 수용

실수 2: Hopping Window를 모르고 수동으로 중복 집계

  요구사항: 현재 시각 기준 최근 5분간 주문 수 (1분마다 갱신)
  
  잘못된 접근:
    5개의 1분 Tumbling Window 결과를 매 1분마다 합산
    → 복잡한 수동 집계 로직, 정확성 보장 어려움

  올바른 방법:
    Hopping Window: 크기 5분, 간격 1분
    TimeWindows.of(Duration.ofMinutes(5))
               .advanceBy(Duration.ofMinutes(1))
    → 1분마다 새 윈도우 시작, 각 이벤트는 5개 윈도우에 중복 포함

실수 3: suppress() 없이 중간 집계 결과를 최종값으로 오해

  WindowedKTable 집계:
    이벤트가 도착할 때마다 윈도우 집계가 업데이트됨
    각 업데이트마다 결과 토픽에 발행

  오해: "윈도우가 끝나면 최종 결과 1개만 나온다"
  실제: 윈도우 기간 동안 이벤트가 10번 오면 10번 발행
        최종 결과 = 마지막 발행값이지만 중간 결과도 다 나옴

  suppress() 사용:
    .suppress(Suppressed.untilWindowCloses(...))
    → 윈도우가 완전히 닫힌 후 최종 결과 1개만 발행
    주의: 윈도우 닫힘 대기 → 결과 지연 증가
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
윈도우 선택 기준:

  Tumbling Window:
    겹치지 않는 고정 크기 창
    예: 1시간별 집계, 일별 집계
    코드: TimeWindows.of(Duration.ofHours(1))

  Hopping Window:
    겹치는 고정 크기 창
    예: 최근 5분간 집계를 1분마다 업데이트
    코드: TimeWindows.of(Duration.ofMinutes(5))
                    .advanceBy(Duration.ofMinutes(1))

  Sliding Window:
    두 이벤트 간 최대 간격이 일정한 창
    예: 같은 사용자의 5분 내 모든 이벤트 쌍 분석
    코드: SlidingWindows.withTimeDifferenceAndGrace(
              Duration.ofMinutes(5), Duration.ZERO)

  Session Window:
    비활동 기간으로 구분되는 동적 창
    예: 사용자 세션 (5분 이상 활동 없으면 새 세션)
    코드: SessionWindows.ofInactivityGapWithNoGrace(
              Duration.ofMinutes(5))

Event Time 기준 + grace period:
  .withTimestampExtractor((record, prevTimestamp) ->
      ((Event) record.value()).getOccurredAt().toEpochMilli())
  .grace(Duration.ofSeconds(30))
  → 이벤트 내 타임스탬프 사용, 30초 지연 수용
```

---

## 🔬 내부 동작 원리

### 1. 4가지 윈도우 타입 비교

```
타임라인: ──────────────────────────────────────►

이벤트:    e1   e2         e3    e4    e5

Tumbling Window (크기=3시간):
  창1: [0~3h)  → e1, e2 포함
  창2: [3h~6h) → e3, e4, e5 포함
  특징: 각 이벤트는 정확히 하나의 창에만 속함

Hopping Window (크기=3시간, 간격=1시간):
  창1: [0~3h)   → e1, e2
  창2: [1h~4h)  → e1, e2, e3    ← 겹침!
  창3: [2h~5h)  → e2, e3, e4    ← 겹침!
  창4: [3h~6h)  → e3, e4, e5
  특징: 각 이벤트가 여러 창에 포함 (크기/간격 배수)

Sliding Window (최대 이벤트 간격=2시간):
  (e1, e2): e2 - e1 <= 2h → 같은 창
  (e2, e3): e3 - e2 > 2h → 다른 창
  특징: 이벤트 쌍 간의 시간 차이 기준

Session Window (비활동 간격=2시간):
  e1~e2: 간격 < 2h → 같은 세션
  e2~e3: 간격 > 2h → 새 세션 시작
  창1: [e1 시작 ~ e2 끝]  동적 크기
  창2: [e3 시작 ~ e5 끝]  동적 크기
  특징: 이벤트가 없는 비활동 기간으로 세션 구분
```

### 2. Event Time vs Processing Time

```
Processing Time (기본값):
  Kafka 브로커가 메시지를 받은 시각
  또는 Kafka Streams가 처리하는 시각
  
  장점: 단순, 예측 가능, 지연 이벤트 없음
  단점: 실제 발생 시각과 다를 수 있음
         모바일 오프라인 이벤트, 재처리 시 부정확

Event Time:
  이벤트 자체에 포함된 발생 시각 (비즈니스 의미의 시간)
  
  사용 방법:
    TimeWindows.of(Duration.ofMinutes(5))
               .withTimestampExtractor(...)
    
    // TimestampExtractor 구현
    public class EventTimeExtractor implements TimestampExtractor {
        @Override
        public long extract(ConsumerRecord<?, ?> record, long partitionTime) {
            Event event = (Event) record.value();
            return event.getOccurredAt().toEpochMilli();
        }
    }
  
  장점: 비즈니스적으로 정확한 집계
  단점: 늦게 도착하는 이벤트(Late Arrival) 처리 필요
         grace period 설정 필수

Ingestion Time:
  Kafka 브로커가 메시지를 받은 시각 (Processing Time과 유사)
  message.timestamp.type=LogAppendTime 설정 시 사용
```

### 3. grace period와 Late Arrival 처리

```
grace period: 윈도우 종료 후 늦게 도착한 이벤트를 수용하는 추가 시간

  윈도우 [10:00 ~ 11:00), grace period = 30분

  timeline:
    10:59 → 이벤트 A 발생
    11:00 → 윈도우 종료 (grace period 시작)
    11:15 → 이벤트 A Kafka 도착 (15분 지연) → 수용! [10:00~11:00 창에 포함]
    11:30 → grace period 종료
    11:35 → 이벤트 B 발생 시각=10:55, 도착 → 거부! (35분 지연 > grace 30분)

  grace period 설정:
    TimeWindows.of(Duration.ofHours(1))
               .grace(Duration.ofMinutes(30))

  트레이드오프:
    grace 길게: 더 많은 지연 이벤트 수용 → 결과 지연
    grace 짧게: 적시 결과 → 일부 지연 이벤트 누락

  grace = 0이면:
    윈도우 종료 즉시 늦은 이벤트 거부
    실시간 집계 지표에 적합 (약간의 부정확 허용)
```

### 4. suppress(): 최종 결과만 발행

```
suppress() 없는 경우:

  윈도우 [10:00~11:00)에 이벤트 5개 도착:
    10:05 → 이벤트1 도착 → 집계=1 → 발행: key=[10:00~11:00], value=1
    10:20 → 이벤트2 도착 → 집계=2 → 발행: key=[10:00~11:00], value=2
    10:35 → 이벤트3 도착 → 집계=3 → 발행: key=[10:00~11:00], value=3
    10:50 → 이벤트4 도착 → 집계=4 → 발행: key=[10:00~11:00], value=4
    10:58 → 이벤트5 도착 → 집계=5 → 발행: key=[10:00~11:00], value=5

  총 5개의 중간 결과 발행 (최종값은 5이지만 1,2,3,4도 다 나옴)

suppress() 사용:
  .suppress(Suppressed.untilWindowCloses(
      BufferConfig.unbounded()))  // 메모리에 중간 결과 버퍼링

  → 윈도우 종료(11:00) + grace period 경과 후
  → 최종 결과 1개만 발행: key=[10:00~11:00], value=5

  비용:
    suppress() 동안 윈도우 결과를 메모리에 버퍼링
    오랜 윈도우 + 많은 키 → OOM 위험
    BufferConfig.maxBytes(10MB).emitEarlyWhenFull() 로 제한 가능
```

### 5. Session Window 동적 병합

```
Session Window 처리:

  이벤트: user-1의 클릭 스트림
  비활동 간격 = 5분

  t=10:00 → 클릭1
    세션 [10:00~10:00] 생성

  t=10:03 → 클릭2
    클릭1의 세션 [10:00~10:00]과 클릭2 간격 = 3분 < 5분
    → 세션 병합: [10:00~10:03] (클릭1 + 클릭2)

  t=10:12 → 클릭3
    이전 세션 [10:00~10:03] 종료로부터 9분 > 5분
    → 새 세션 [10:12~10:12] 생성

  t=10:15 → 클릭4
    클릭3의 세션 [10:12~10:12]과 간격 = 3분 < 5분
    → 세션 병합: [10:12~10:15] (클릭3 + 클릭4)

  최종 세션:
    세션1: [10:00~10:03] → 이벤트 2개
    세션2: [10:12~10:15] → 이벤트 2개

  Session Window = 동적 크기 (고정 크기 없음)
  비활동 간격이 유일한 파라미터
```

---

## 💻 실전 실험

### 실험 1: Tumbling vs Hopping Window 비교

```java
StreamsBuilder builder = new StreamsBuilder();
KStream<String, Long> clicks = builder.stream("click-events");

// Tumbling Window: 1시간별 집계
clicks.groupByKey()
      .windowedBy(TimeWindows.of(Duration.ofHours(1)))
      .count()
      .toStream()
      .foreach((key, count) ->
          System.out.printf("Tumbling [%s]: %d%n",
              key.window().startTime(), count));

// Hopping Window: 1시간 크기, 15분마다 업데이트
clicks.groupByKey()
      .windowedBy(TimeWindows.of(Duration.ofHours(1))
                              .advanceBy(Duration.ofMinutes(15)))
      .count()
      .toStream()
      .foreach((key, count) ->
          System.out.printf("Hopping [%s]: %d%n",
              key.window().startTime(), count));
```

### 실험 2: Event Time 기준 집계

```java
// 이벤트 내 타임스탬프 추출
TimestampExtractor extractor = (record, partitionTime) -> {
    ClickEvent event = (ClickEvent) record.value();
    return event.getOccurredAt().toEpochMilli();
};

KStream<String, ClickEvent> clicks = builder.stream(
    "click-events",
    Consumed.with(Serdes.String(), clickEventSerde)
            .withTimestampExtractor(extractor)
);

clicks.groupByKey()
      .windowedBy(
          TimeWindows.of(Duration.ofMinutes(5))
                     .grace(Duration.ofSeconds(30))  // 30초 지연 수용
      )
      .count()
      .suppress(Suppressed.untilWindowCloses(
          BufferConfig.maxBytes(5 * 1024 * 1024)  // 5 MB 버퍼
                      .emitEarlyWhenFull()))
      .toStream()
      .to("click-count-results");
```

### 실험 3: Session Window 세션 분석

```java
// 사용자 세션 집계 (5분 비활동 시 새 세션)
clicks.groupByKey()
      .windowedBy(
          SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(5))
      )
      .count()
      .toStream()
      .foreach((windowedKey, count) -> {
          long sessionDuration = windowedKey.window().end()
                                - windowedKey.window().start();
          System.out.printf("User %s: session %dms, clicks %d%n",
              windowedKey.key(),
              sessionDuration,
              count);
      });
```

---

## 📊 성능/비용 비교

### 윈도우 타입별 상태 저장소 크기

```
조건: 1분 활성 사용자 10만 명, 1시간 집계 창

  Tumbling Window (1시간):
    동시 활성 윈도우: 1개 (1시간)
    상태 크기: ~10만 키 × 1개 윈도우 = 10만 항목

  Hopping Window (1시간 크기, 15분 간격):
    동시 활성 윈도우: 4개 (1시간 / 15분)
    상태 크기: ~10만 키 × 4개 윈도우 = 40만 항목

  Session Window:
    동시 활성 세션: 사용자당 1개 (비활동 전까지)
    상태 크기: ~10만 개 활성 세션 + grace period 동안 추가
    주의: 장시간 세션이 있으면 상태 크기 무제한 증가 가능

  suppress() 메모리 사용:
    윈도우 종료 전까지 중간 결과 버퍼링
    크기 = 활성 윈도우 수 × 평균 결과 크기
    1시간 Tumbling + 10만 키 → ~수 MB
```

---

## ⚖️ 트레이드오프

```
Event Time:
  ✅ 비즈니스적으로 정확한 집계
  ❌ Late Arrival 처리 필요 (grace period)
  ❌ 결과 발행이 grace period만큼 지연

Processing Time:
  ✅ 구현 단순, 지연 이벤트 없음
  ❌ 실제 발생 시각과 다를 수 있음

grace period 길게:
  ✅ 더 많은 지연 이벤트 수용
  ❌ 결과 발행 지연 증가, 상태 저장소 크기 증가

suppress():
  ✅ 최종 결과만 발행 (다운스트림 단순화)
  ❌ 윈도우 종료까지 결과 지연
  ❌ 메모리 버퍼링 필요 (OOM 위험)

Session Window:
  ✅ 사용자 행동 패턴에 자연스러운 집계
  ❌ 상태 크기 예측 어려움 (동적 창 크기)
  ❌ 세션 병합으로 인한 상태 재계산
```

---

## 📌 핵심 정리

```
윈도우 연산 핵심:

1. 4가지 윈도우:
   Tumbling: 겹치지 않음, 각 이벤트는 1개 창
   Hopping:  겹침, 각 이벤트는 여러 창에 포함
   Sliding:  이벤트 간 시간 차이 기반
   Session:  비활동 간격으로 동적 창 결정

2. Event Time vs Processing Time:
   Event Time: 이벤트 발생 시각 (정확, grace period 필요)
   Processing Time: 처리 시각 (단순, 오프라인 이벤트 부정확)

3. grace period = Late Arrival 수용 시간
   윈도우 종료 후 N시간까지 늦게 도착한 이벤트 수용

4. 윈도우 중간 결과:
   이벤트 도착마다 업데이트 발행 (여러 번)
   suppress()로 최종 결과만 발행 가능 (지연 증가)

5. 상태 저장소 크기:
   Tumbling < Hopping < Session (예측 어려움)
   grace period 길수록 더 많은 상태 유지
```

---

## 🤔 생각해볼 문제

**Q1. 이벤트가 생성된 시각이 미래 시각으로 잘못 기록됐습니다. Event Time 기준 윈도우에서 어떤 문제가 발생하나요?**

<details>
<summary>해설 보기</summary>

미래 시각의 이벤트가 도착하면 해당 이벤트는 아직 시작되지 않은 미래 윈도우에 할당됩니다. 이 윈도우는 열린 상태로 오랫동안 유지되고, 해당 윈도우를 닫기 위해 처리 시간이 미래 시각을 넘어설 때까지 상태 저장소에 남아있습니다.

더 심각한 문제는 워터마크(watermark) 또는 스트림 시간(stream time)에 영향입니다. Kafka Streams는 처리하는 이벤트들의 최대 타임스탬프를 스트림 시간으로 사용합니다. 미래 시각의 이벤트 하나가 오면 스트림 시간이 미래로 점프하고, 이전의 정상적인 타임스탬프를 가진 이벤트가 Late Arrival로 분류되어 거부될 수 있습니다.

해결책: `TimestampExtractor`에서 미래 타임스탬프 검증 로직 추가. 현재 처리 시각보다 일정 시간(예: 24시간) 이상 미래이면 Processing Time으로 대체합니다.

</details>

---

**Q2. 하나의 이벤트가 Hopping Window에서 여러 창에 중복 포함되면 집계 결과가 잘못된 것 아닌가요?**

<details>
<summary>해설 보기</summary>

Hopping Window는 의도적으로 이벤트가 여러 창에 포함됩니다. 이것은 "현재 시각 기준 최근 N분간"이라는 이동 평균/이동 집계를 표현하기 위한 설계입니다.

예를 들어 "최근 1시간 주문 수를 15분마다 업데이트"하는 요구사항이 있다면, 각 이벤트가 4개의 1시간 창(15분 간격)에 중복 포함되는 것이 올바른 동작입니다. 각 창의 집계값은 "해당 창의 시작부터 1시간 동안 발생한 총 주문 수"를 나타냅니다.

혼동하지 말아야 할 것은 Hopping Window는 "겹치는 구간을 중복 집계하는 것"이 아니라 "여러 시간 관점에서 동일 기간을 집계하는 것"입니다. 전체 이벤트 수를 알고 싶다면 Tumbling Window를 사용해야 합니다.

</details>

---

**Q3. Session Window에서 두 세션 사이에 Tombstone을 발행하면 어떻게 되나요?**

<details>
<summary>해설 보기</summary>

세션이 삭제됩니다. Tombstone(value=null)을 해당 키로 발행하면 현재 활성 세션이 종료되고 세션 상태가 KTable에서 제거됩니다.

이것은 실제로 Session Window에서 특정 사용자의 세션을 강제로 종료해야 할 때 활용할 수 있습니다. 예를 들어 명시적인 "로그아웃" 이벤트를 Tombstone으로 표현하면 비활동 간격 타임아웃을 기다리지 않고 즉시 세션을 종료할 수 있습니다.

단, Tombstone 이후 같은 키로 새 이벤트가 오면 새로운 세션이 시작됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: KTable과 KStream](./02-ktable-kstream.md)** | **[홈으로 🏠](../README.md)** | **[다음: Exactly-Once in Kafka Streams ➡️](./04-exactly-once-streams.md)**

</div>
