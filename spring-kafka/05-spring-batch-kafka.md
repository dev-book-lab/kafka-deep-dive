# Spring Batch + Kafka — Chunk 처리와 Exactly-Once

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `KafkaItemReader`가 Consumer Offset을 JobExecutionContext에 저장하는 방식은?
- Kafka Offset 커밋과 Batch Checkpoint가 동기화되지 않을 때 어떤 문제가 생기는가?
- `KafkaItemWriter`의 트랜잭션 처리 방식은?
- Chunk 단위 Exactly-Once 달성을 위한 `KafkaTransactionManager` + `JpaTransactionManager` 조합은?
- 배치 재실행(Restart) 시 Kafka offset은 어디서부터 읽는가?
- Spring Batch의 Chunk 처리와 Kafka Consumer의 at-least-once 보장은 어떻게 결합되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Batch의 Chunk 처리는 N개의 Item을 읽고, 처리하고, 쓰는 단위다. Kafka에서 데이터를 읽어서 DB에 저장하는 배치는 다음 문제를 해결해야 한다.

"Chunk 내 100개 Item을 읽어서 DB에 40개 저장 완료, 41번째 저장 중 DB 오류 → Batch 재시작 → Kafka에서 다시 읽으면 어디서부터 읽어야 하는가?"

답은 `KafkaItemReader`가 offset을 `JobExecutionContext`에 저장하고, Batch가 재시작 시 이 값을 기준으로 Kafka를 seek한다는 것이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: enable.auto.commit=true로 KafkaItemReader 설정

  설정 오류:
    enable.auto.commit=true (기본값 그대로)
    Chunk 처리 중 자동으로 Kafka offset 커밋

  문제:
    Chunk 1: offset 0~99 읽기 → 자동 커밋 (offset=100)
    Chunk 1 처리 완료 → Batch Checkpoint
    Chunk 2: offset 100~199 읽기 → 자동 커밋 (offset=200)
    Chunk 2 DB 저장 실패 → Batch 재시작
    Kafka Consumer Group: offset=200 (이미 커밋)
    Batch 재시작: offset 100부터 읽어야 하지만 Kafka는 200 반환
    → offset 100~199 메시지 누락!

  올바른 설정:
    enable.auto.commit=false
    KafkaItemReader가 Batch Checkpoint와 동기화하여 커밋

실수 2: Chunk 재처리를 고려하지 않은 Writer

  상황:
    Chunk offset 100~199 처리 중 150번째에서 DB 오류
    Batch 재시작 → Checkpoint에서 offset 100부터 재처리
    offset 100~149는 이미 DB에 저장됨!

  Writer가 단순 INSERT이면:
    중복 PRIMARY KEY 오류 또는 중복 데이터

  올바른 접근:
    UPSERT (ON CONFLICT DO UPDATE / INSERT IGNORE)
    중복 처리되어도 결과가 동일한 멱등 Write

실수 3: KafkaItemReader saveState 없이 장기 배치 운영

  문제:
    saveState=false이면 Step Execution이 업데이트되어도
    KafkaItemReader는 offset을 ExecutionContext에 저장 안 함
    Batch 재시작 시 처음부터 읽기 → 전체 재처리
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```java
// KafkaItemReader 올바른 설정
@Bean
@StepScope
public KafkaItemReader<String, Order> orderReader() {
    Properties props = new Properties();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(ConsumerConfig.GROUP_ID_CONFIG, "batch-order-processor");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
              StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
              JsonDeserializer.class);
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // 필수!

    return new KafkaItemReaderBuilder<String, Order>()
        .name("orderReader")
        .consumerProperties(props)
        .topic("orders")
        .partitions(0, 1, 2)
        .saveState(true)          // Checkpoint에 offset 저장
        .pollTimeout(Duration.ofSeconds(5))
        .build();
}

// 멱등 ItemWriter (UPSERT)
@Bean
public ItemWriter<ProcessedOrder> idempotentWriter(JdbcTemplate jdbc) {
    return items -> items.forEach(item ->
        jdbc.update(
            "INSERT INTO processed_orders (id, amount, status) " +
            "VALUES (?, ?, ?) " +
            "ON CONFLICT (id) DO UPDATE " +
            "SET amount = EXCLUDED.amount, status = EXCLUDED.status",
            item.getId(), item.getAmount(), item.getStatus())
    );
}
```

---

## 🔬 내부 동작 원리

### 1. KafkaItemReader offset 관리

```
KafkaItemReader Checkpoint 동작:

  처음 실행 시:
    partitionOffsets 초기값 또는 Consumer Group 마지막 커밋에서 시작

  Chunk 처리 중:
    read() → KafkaConsumer.poll() → 레코드 1개 반환
    내부 partitionOffset 맵 업데이트

  Chunk 완료 (Step Execution 업데이트):
    현재 partitionOffset → ExecutionContext 저장
    {
      "orders.partition.0.currentOffset": 150,
      "orders.partition.1.currentOffset": 143,
      "orders.partition.2.currentOffset": 157
    }
    JobRepository DB에 기록 (BATCH_STEP_EXECUTION_CONTEXT)

  Batch 재시작 시:
    JobRepository에서 ExecutionContext 로드
    partitionOffsets 복원
    KafkaConsumer.seek(partition, savedOffset)
    → 저장된 offset부터 재처리

  Chunk 경계 정확성:
    saveState=true + Chunk 완료마다 저장
    → Chunk 시작 offset까지는 재처리 없음 (정확한 Checkpoint)
```

### 2. Kafka Offset vs Batch Checkpoint 동기화

```
동기화 타임라인 (enable.auto.commit=false):

  t1: Chunk 1 시작 (offset 0)
  t2: Kafka poll() → offset 0~99 읽기
  t3: DB 저장 완료 (100개)
  t4: Step Execution 업데이트
      ExecutionContext: {partition.0.offset: 100}
      JobRepository: 저장
  t5: Kafka offset 커밋 (KafkaItemReader 내부)

  t6: Chunk 2 시작 (offset 100)
  t7: Kafka poll() → offset 100~199 읽기
  t8: DB 저장 중 (offset 130) 오류 발생!

  t9: Batch 재시작
      ExecutionContext 로드: {partition.0.offset: 100}
      KafkaConsumer.seek(partition, 100)
      → offset 100부터 재처리 (130~199도 포함)

  멱등 Writer 없으면:
      offset 100~129 이미 DB에 있음
      → 중복 저장 오류 또는 중복 데이터
  멱등 Writer (UPSERT) 있으면:
      offset 100~129 UPSERT → 동일 결과 (안전)
      offset 130~199 신규 저장
```

### 3. KafkaItemWriter 트랜잭션 설정

```java
// KafkaItemWriter: Chunk 완료 시 일괄 발행
@Bean
public KafkaItemWriter<String, ProcessedOrder> kafkaWriter(
        KafkaTemplate<String, ProcessedOrder> template) {
    KafkaItemWriter<String, ProcessedOrder> writer =
        new KafkaItemWriter<>();
    writer.setKafkaTemplate(template);
    writer.setItemKeyMapper(item -> item.getOrderId());
    writer.setTopic("processed-orders");
    return writer;
}

// Chunk Step에서 CompositeItemWriter: DB + Kafka 동시
@Bean
public CompositeItemWriter<ProcessedOrder> compositeWriter(
        JpaItemWriter<ProcessedOrder> dbWriter,
        KafkaItemWriter<String, ProcessedOrder> kafkaWriter) {
    CompositeItemWriter<ProcessedOrder> writer =
        new CompositeItemWriter<>();
    writer.setDelegates(Arrays.asList(dbWriter, kafkaWriter));
    return writer;
}
```

### 4. ChainedTransactionManager Chunk 설정

```java
@Bean
public ChainedKafkaTransactionManager<String, ProcessedOrder>
        chainedTm(KafkaTransactionManager<String, ProcessedOrder> kafkaTm,
                  JpaTransactionManager jpaTm) {
    return new ChainedKafkaTransactionManager<>(kafkaTm, jpaTm);
}

@Bean
public Step processOrdersStep(
        JobRepository jobRepository,
        KafkaItemReader<String, Order> reader,
        ItemProcessor<Order, ProcessedOrder> processor,
        CompositeItemWriter<ProcessedOrder> writer,
        ChainedKafkaTransactionManager<String, ProcessedOrder> chainedTm) {
    return new StepBuilder("processOrdersStep", jobRepository)
        .<Order, ProcessedOrder>chunk(100, chainedTm)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .faultTolerant()
        .skip(BusinessException.class)
        .skipLimit(10)
        .build();
}
```

### 5. Skip과 DLT 연동

```java
// Skip된 Item을 DLT로 전송
@Bean
public SkipListener<Order, ProcessedOrder> skipToDlt(
        KafkaTemplate<String, Order> kafkaTemplate) {
    return new SkipListener<>() {
        @Override
        public void onSkipInProcess(Order item, Throwable t) {
            log.error("Process Skip: orderId={}", item.getId(), t);
            kafkaTemplate.send("orders.DLT",
                item.getId().toString(), item);
        }

        @Override
        public void onSkipInWrite(ProcessedOrder item, Throwable t) {
            log.error("Write Skip: orderId={}", item.getOrderId(), t);
            // 원본 Order를 DLT로 전송해야 하므로
            // ProcessedOrder에서 원본 정보를 보존해두어야 함
        }
    };
}
```

---

## 💻 실전 실험

### 실험 1: KafkaItemReader Checkpoint 확인

```bash
# 1. Kafka에 200개 메시지 발행
for i in $(seq 1 200); do
  echo '{"id":"order-'$i'","amount":'$((RANDOM % 1000))'}'
done | kafka-console-producer --bootstrap-server localhost:9092 \
  --topic orders

# 2. Batch 실행 (Chunk=100)
# → 100개 처리 후 강제 종료

# 3. DB에서 Checkpoint 확인
psql -c "SELECT short_context FROM BATCH_STEP_EXECUTION_CONTEXT
         ORDER BY step_execution_id DESC LIMIT 1;"
# {"orders.partition.0.currentOffset":100}

# 4. Batch 재시작
# → ExecutionContext에서 offset 100 복원
# → 나머지 100개 처리

# Kafka Consumer Group offset 확인
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group batch-order-processor
```

### 실험 2: 멱등 Writer 중복 방지 확인

```sql
-- 처리 전 DB 상태
SELECT COUNT(*) FROM processed_orders;  -- 40개 (Chunk 1의 일부)

-- Batch 재시작 후 (offset 0부터 재처리)
-- UPSERT로 40개 중복 처리 → 동일 결과
SELECT COUNT(*) FROM processed_orders;  -- 100개 (중복 없음)

-- UPSERT 없이 INSERT만 쓰면
-- UNIQUE CONSTRAINT 오류 또는 중복 레코드
```

### 실험 3: Batch 성능 측정

```bash
# Chunk size별 처리 시간 비교
for chunk in 10 100 1000; do
  echo "=== Chunk size: $chunk ==="
  time java -jar batch-app.jar \
    --spring.batch.job.name=processOrdersJob \
    --chunkSize=$chunk \
    --spring.batch.job.parameters=run.id=$(date +%s)
done
```

---

## 📊 성능/비용 비교

### Chunk Size별 처리량

```
KafkaItemReader + JpaItemWriter, 1 KB 메시지:

  chunk=10:
    처리량:    ~5,000 msg/sec
    DB 커밋:  빈번 (JobRepository 업데이트 포함)
    재처리 범위: 최대 10개

  chunk=100:
    처리량:    ~20,000 msg/sec
    DB 커밋:  중간
    재처리 범위: 최대 100개

  chunk=1000:
    처리량:    ~50,000 msg/sec
    DB batchInsert 효과 극대화
    재처리 범위: 최대 1,000개 (실패 시 영향 큼)

  권장: 처리 오류 허용도에 따라 100~500
       중요 데이터: 작은 Chunk (재처리 범위 최소화)
       로그성 데이터: 큰 Chunk (처리량 최대화)
```

---

## ⚖️ 트레이드오프

```
enable.auto.commit=false:
  ✅ Batch Checkpoint와 Kafka offset 동기화
  ✅ 재시작 후 정확한 offset에서 재처리
  ❌ Kafka offset 커밋이 지연됨
     (Batch 완료 전 다른 Consumer가 해당 offset을 중복 처리 가능)

멱등 Writer (UPSERT):
  ✅ 재처리 안전 (At-Least-Once를 사실상 Exactly-Once로)
  ✅ 구현 단순
  ❌ INSERT보다 약간 느림
  ❌ 이메일 발송 등 부작용은 처리 불가

ChainedTransactionManager:
  ✅ Kafka offset + DB 동시 커밋 시도
  ❌ Best Effort (Kafka 커밋 후 DB 커밋 실패 시 불일치)
  → 멱등 Writer와 반드시 함께 사용

큰 Chunk Size:
  ✅ 처리량 향상, DB 왕복 감소
  ❌ 실패 시 재처리 범위 증가
  ❌ 메모리 사용량 증가
```

---

## 📌 핵심 정리

```
Spring Batch + Kafka 핵심:

1. enable.auto.commit=false 필수
   Batch Checkpoint와 Kafka offset 동기화 보장

2. KafkaItemReader + saveState=true
   Chunk 완료마다 partition offset → ExecutionContext 저장
   재시작 시 저장된 offset부터 Kafka seek

3. 멱등 Writer (UPSERT) = 가장 실용적인 Exactly-Once
   재처리되어도 DB 결과 동일
   ChainedTransactionManager보다 단순하고 신뢰성 높음

4. Chunk Size 선택:
   재처리 허용 범위 (작을수록 안전)
   처리량 요구사항 (클수록 빠름)
   권장: 100~500

5. Skip + SkipListener + DLT:
   처리 실패 Item을 격리하고 배치 계속 진행
   DLT에서 나중에 원인 파악 및 재처리
```

---

## 🤔 생각해볼 문제

**Q1. Spring Batch Job이 완료된 후 동일 파라미터로 다시 실행하면 KafkaItemReader는 어디서부터 읽나요?**

<details>
<summary>해설 보기</summary>

Spring Batch는 동일한 `JobParameters`로 완료된 Job을 재실행하지 않습니다. `JobInstanceAlreadyCompleteException`이 발생합니다. 새로운 실행을 하려면 `JobParameters`를 다르게 설정해야 합니다(예: 실행 타임스탬프 추가).

새 `JobParameters`로 새 `JobExecution`이 생성되면 `ExecutionContext`가 비어있습니다. `KafkaItemReader`는 `partitionOffsets` 초기 설정값에서 시작합니다. 초기값이 없으면 Consumer Group의 마지막 커밋 offset에서 시작합니다(`auto.offset.reset` 기준).

정기 배치 패턴에서는 `JobParameters`에 처리 날짜를 포함시키고, 매일 새 실행을 생성합니다. 각 날짜별 Kafka offset 범위를 `--to-datetime`으로 설정하면 날짜별 정확한 데이터 처리가 가능합니다.

</details>

---

**Q2. `KafkaItemReader`가 Batch Checkpoint를 저장하면 `__consumer_offsets`에도 커밋되나요?**

<details>
<summary>해설 보기</summary>

아닙니다. Batch Checkpoint는 `JobRepository`(DB의 `BATCH_STEP_EXECUTION_CONTEXT` 테이블)에 저장되고, `__consumer_offsets`에 대한 Kafka offset 커밋은 별도로 발생합니다.

`KafkaItemReader`는 Batch Step이 완료될 때(또는 설정에 따라) `__consumer_offsets`에 Kafka offset을 커밋합니다. 두 저장이 원자적으로 일어나지는 않습니다.

재시작 시 `KafkaItemReader`는 `__consumer_offsets`보다 `ExecutionContext`의 offset을 우선 사용합니다. 따라서 Kafka offset 커밋이 실패해도 Batch Checkpoint에서 복원하므로 중복 없이 재처리할 수 있습니다.

</details>

---

**Q3. `KafkaItemReader`와 `KafkaItemWriter`를 동시에 사용할 때 입력 Consumer Group과 출력 Producer의 트랜잭션을 묶으려면 어떻게 하나요?**

<details>
<summary>해설 보기</summary>

`KafkaTransactionManager`를 Chunk Step의 트랜잭션 매니저로 설정하면 됩니다. Chunk 처리가 트랜잭션 범위 내에서 실행되어 입력 Consumer offset 커밋과 출력 Producer 발행이 하나의 Kafka 트랜잭션으로 묶입니다.

```java
Step step = new StepBuilder("step", jobRepository)
    .<Order, ProcessedOrder>chunk(100, kafkaTransactionManager)
    .reader(kafkaReader)    // KafkaItemReader
    .processor(processor)
    .writer(kafkaWriter)    // KafkaItemWriter
    .build();
```

내부적으로 `KafkaTransactionManager`가 Chunk 시작 시 `beginTransaction()`을 호출하고, Chunk 완료 시 `commitTransaction()`을 호출합니다. 이 트랜잭션 내에서 `KafkaItemWriter.write()`의 발행과 `KafkaItemReader`의 offset 커밋이 함께 포함됩니다.

단, DB 저장도 있다면 `ChainedTransactionManager(kafkaTm, jpaTm)`를 사용하세요. 완전한 원자성은 아니지만(Best Effort) 간단하고 실용적입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Spring Cloud Stream](./04-spring-cloud-stream.md)** | **[홈으로 🏠](../README.md)**

</div>
