# Spring Cloud Stream — 함수형 모델과 Kafka Binder

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Spring Cloud Stream의 함수형 프로그래밍 모델은 `@KafkaListener`와 어떻게 다른가?
- `Function<Message<IN>, Message<OUT>>`이 입력 토픽과 출력 토픽에 어떻게 연결되는가?
- Kafka Binder가 내부적으로 `KafkaMessageListenerContainer`와 `KafkaTemplate`을 어떻게 구성하는가?
- 바인더 추상화로 RabbitMQ에서 Kafka로 교체 시 코드 변경 범위는?
- `spring.cloud.stream.bindings` 설정이 토픽 이름과 Consumer Group을 어떻게 매핑하는가?
- Reactive 스트리밍(`Flux`)을 활용하는 경우는 언제인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Kafka를 직접 사용하면 토픽 이름, Consumer 설정, Producer 설정이 코드에 직접 결합된다. Spring Cloud Stream은 메시징 인프라(Kafka, RabbitMQ 등)를 추상화해서 함수형 비즈니스 로직을 인프라와 분리한다.

MSA 환경에서 메시징 인프라를 변경하거나 로컬 테스트에서 다른 바인더를 사용할 때 코드 변경 없이 설정만 바꿀 수 있다는 것이 Spring Cloud Stream의 핵심 가치다.

단, 추상화 레이어가 추가되므로 내부 동작을 모르면 디버깅이 어렵고, 설정이 직관적이지 않아 혼란을 야기할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 함수 이름과 토픽 바인딩 관계 오해

  코드:
    @Bean
    public Function<String, String> process() {
        return input -> input.toUpperCase();
    }

  기대: "process"라는 토픽에서 읽어서 처리 후 어딘가에 발행
  실제:
    기본 바인딩 이름 = 함수이름-in-0 / 함수이름-out-0
    입력: process-in-0 → 실제 토픽: process-in-0 (설정 없으면)
    출력: process-out-0 → 실제 토픽: process-out-0 (설정 없으면)
    
  올바른 토픽 이름 매핑:
    spring.cloud.stream.bindings.process-in-0.destination=orders
    spring.cloud.stream.bindings.process-out-0.destination=processed-orders

실수 2: 함수형 빈이 여러 개일 때 자동 연결 혼란

  코드:
    @Bean Function<String, String> process() { ... }
    @Bean Function<String, String> validate() { ... }

  오해: Spring이 자동으로 validate → process로 체인 구성
  실제: 각 Function이 독립적인 리스너로 생성됨
        체인이 필요하면:
          spring.cloud.function.definition=validate|process
        또는 단일 Function으로 통합

실수 3: Consumer Group 없이 운영

  설정 누락:
    spring.cloud.stream.bindings.process-in-0.destination=orders
    # group 설정 없음!

  문제:
    group 없으면 익명 그룹 생성 (랜덤 UUID)
    애플리케이션 재시작마다 새 그룹 → offset 리셋
    이전에 처리한 메시지를 처음부터 재처리

  올바른 설정:
    spring.cloud.stream.bindings.process-in-0.group=order-processor
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Spring Cloud Stream 권장 설정:

  spring:
    cloud:
      stream:
        bindings:
          process-in-0:
            destination: orders          # 입력 토픽
            group: order-processor       # Consumer Group (필수!)
          process-out-0:
            destination: processed-orders # 출력 토픽
        kafka:
          binder:
            brokers: localhost:9092
          bindings:
            process-in-0:
              consumer:
                startOffset: latest
                ackEachRecord: false      # BATCH 모드
                maxPollRecords: 500
            process-out-0:
              producer:
                compressionType: lz4
      function:
        definition: process              # 사용할 함수 지정

  # 함수형 빈
  @Bean
  public Function<Message<Order>, Message<ProcessedOrder>> process() {
      return message -> {
          Order order = message.getPayload();
          ProcessedOrder result = processOrder(order);
          return MessageBuilder.withPayload(result)
              .copyHeaders(message.getHeaders())
              .build();
      };
  }
```

---

## 🔬 내부 동작 원리

### 1. 함수형 모델과 바인딩 구조

```
Spring Cloud Stream 함수형 빈 타입:

  Function<I, O>: 입력 → 처리 → 출력
    입력 바인딩: {name}-in-0
    출력 바인딩: {name}-out-0

  Consumer<I>: 입력 → 처리 (출력 없음)
    입력 바인딩: {name}-in-0
    출력 바인딩: 없음

  Supplier<O>: 처리 → 출력 (입력 없음, 폴링 기반)
    입력 바인딩: 없음
    출력 바인딩: {name}-out-0

  멀티 입력/출력 (함수 체인):
    spring.cloud.function.definition=validate|process|notify
    validate-in-0 → validate-out-0 → process-in-0 → process-out-0
                                  → notify-in-0 → notify-out-0

바인딩 이름 규칙:
  {functionName}-in-{index}   ← 입력 채널
  {functionName}-out-{index}  ← 출력 채널
  index: 여러 입력/출력이 있으면 0, 1, 2, ...
```

### 2. Kafka Binder 내부 구조

```
Kafka Binder가 Function 빈을 처리하는 방식:

  1. FunctionBindingRegistrar
     Function 빈 탐색 → bindInOut() 호출

  2. 입력 바인딩 (Consumer 측):
     KafkaBinderConfiguration
     → DefaultKafkaConsumerFactory 생성
     → KafkaMessageListenerContainer 생성
        (spring.cloud.stream.kafka.bindings.{name}-in-0 설정 적용)
     → MessageHandlerAdapter: Consumer.accept() 또는 Function.apply() 호출

  3. 출력 바인딩 (Producer 측):
     KafkaBinderConfiguration
     → DefaultKafkaProducerFactory 생성
     → KafkaTemplate 생성
     → Function 반환값 → KafkaTemplate.send() 자동 호출

  결과: 개발자는 Function 빈만 구현
        Kafka 연결, Consumer/Producer 관리는 Binder가 담당
```

### 3. 바인더 추상화의 실제 교체 범위

```
코드 (바인더 독립):
  @Bean
  public Function<Message<Order>, Message<ProcessedOrder>> process() {
      return msg -> {
          Order order = msg.getPayload();
          return MessageBuilder.withPayload(processOrder(order)).build();
      };
  }
  // Kafka, RabbitMQ, Kinesis 어떤 바인더에도 동일한 코드

Kafka Binder 사용 (pom.xml):
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-kafka</artifactId>
  </dependency>
  설정: spring.cloud.stream.bindings.process-in-0.destination=orders

RabbitMQ Binder로 교체 (pom.xml만 변경):
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
  </dependency>
  설정: spring.cloud.stream.bindings.process-in-0.destination=orders
       (Rabbit Exchange 이름)

변경 범위:
  ✅ 비즈니스 로직 코드: 변경 없음
  ✅ 함수형 빈 시그니처: 변경 없음
  변경 필요: pom.xml 의존성, application.yml 바인더별 설정

완전한 이식성의 한계:
  - Kafka 파티셔닝, 헤더 포맷은 바인더 종속
  - 토픽 설정 (retention, 파티션 수) 등 인프라 설정은 별도
  - 바인더별 고급 기능 (Kafka 트랜잭션 등)은 Kafka 바인더 전용 설정
```

### 4. Reactive 스트리밍 (Flux)

```
Reactive Function으로 고처리량 처리:

  @Bean
  public Function<Flux<Message<Order>>, Flux<Message<ProcessedOrder>>>
          processReactive() {
      return flux -> flux
          .filter(msg -> msg.getPayload().getAmount() > 0)
          .map(msg -> {
              ProcessedOrder result = process(msg.getPayload());
              return MessageBuilder.withPayload(result)
                  .copyHeaders(msg.getHeaders())
                  .build();
          })
          .onErrorResume(e -> {
              log.error("처리 오류", e);
              return Mono.empty();  // 오류 무시하고 계속
          });
  }

Reactive 사용 시점:
  ✅ 비동기 I/O (WebClient, Reactive DB)와 결합
  ✅ 배압(backpressure) 제어가 필요한 경우
  ✅ 복잡한 스트림 변환 (flatMap, merge, zip)
  
  일반 Function으로 충분한 경우:
  ✅ 동기 처리 로직
  ✅ 단순 변환/필터링
  ✅ Spring MVC(블로킹) 애플리케이션

  Reactive 주의사항:
    Reactor Scheduler와 Kafka 폴링 스레드 관리
    onError 핸들링 명시적 구현 필요
    오프셋 커밋 시점 제어 복잡
```

---

## 💻 실전 실험

### 실험 1: 기본 Function 빈 설정

```java
@SpringBootApplication
public class OrderProcessorApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderProcessorApplication.class, args);
    }

    @Bean
    public Function<Order, ProcessedOrder> process() {
        return order -> {
            log.info("주문 처리: {}", order.getId());
            return new ProcessedOrder(order.getId(),
                                      order.getAmount() * 1.1,  // 10% 수수료
                                      "PROCESSED");
        };
    }
}
```

```yaml
# application.yml
spring:
  cloud:
    stream:
      bindings:
        process-in-0:
          destination: orders
          group: order-processor
        process-out-0:
          destination: processed-orders
      function:
        definition: process
    kafka:
      binder:
        brokers: localhost:9092
```

### 실험 2: Consumer 체인 (validate|process)

```java
@Bean
public Function<Order, Order> validate() {
    return order -> {
        if (order.getAmount() <= 0) throw new IllegalArgumentException("Invalid amount");
        return order;
    };
}

@Bean
public Function<Order, ProcessedOrder> process() {
    return order -> new ProcessedOrder(order.getId(), order.getAmount(), "PROCESSED");
}
```

```yaml
spring:
  cloud:
    function:
      definition: validate|process  # 체인: orders → validate → process → processed-orders
    stream:
      bindings:
        validateprocess-in-0:       # 체인 함수는 이름이 합쳐짐
          destination: orders
          group: validator-processor
        validateprocess-out-0:
          destination: processed-orders
```

### 실험 3: RabbitMQ → Kafka 바인더 교체 테스트

```bash
# Kafka 바인더로 실행
./gradlew bootRun --args='--spring.profiles.active=kafka'

# 메시지 발행 (Kafka)
kafka-console-producer --bootstrap-server localhost:9092 \
  --topic orders
# {"id": "order-1", "amount": 100}

# 처리 결과 확인 (Kafka)
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic processed-orders --from-beginning

# RabbitMQ 바인더로 교체 (pom.xml만 변경, 코드 변경 없음)
# ./gradlew bootRun --args='--spring.profiles.active=rabbit'
```

---

## 📊 성능/비용 비교

### Spring Cloud Stream vs Spring Kafka 직접 사용

```
Spring Kafka (@KafkaListener):
  코드량:    중간 (Consumer Factory, ContainerFactory 설정)
  유연성:    높음 (세밀한 Kafka 설정 직접 제어)
  이식성:    낮음 (Kafka 종속)
  디버깅:    용이 (Kafka 레이어 직접)
  처리량:    최고 (추상화 오버헤드 없음)

Spring Cloud Stream:
  코드량:    적음 (Function 빈만 구현)
  유연성:    중간 (바인더별 설정)
  이식성:    높음 (바인더 교체 가능)
  디버깅:    복잡 (추상화 레이어 추가)
  처리량:    약간 낮음 (~5% 추상화 오버헤드)

선택 기준:
  다수 메시징 시스템 지원 필요: Spring Cloud Stream
  Kafka 전용, 성능/세밀한 제어: Spring Kafka
  MSA + 바인더 교체 가능성: Spring Cloud Stream
```

---

## ⚖️ 트레이드오프

```
Spring Cloud Stream 함수형 모델:
  ✅ 비즈니스 로직과 인프라 분리
  ✅ 바인더 교체 시 코드 변경 없음
  ✅ 코드량 감소 (보일러플레이트 제거)
  ❌ Kafka 고급 기능(트랜잭션, 헤더) 사용 시 Kafka 종속 설정 필요
  ❌ 디버깅 복잡 (추상화 레이어)
  ❌ 학습 곡선 (바인딩 이름 규칙, 설정 방식)

Reactive Flux 모델:
  ✅ 비동기 I/O와 자연스러운 결합
  ✅ 복잡한 스트림 연산
  ❌ 오프셋 커밋 제어 복잡
  ❌ 에러 핸들링 명시적 구현 필요
  ❌ 블로킹 코드 혼용 위험 (reactor scheduler blocking)
```

---

## 📌 핵심 정리

```
Spring Cloud Stream 핵심:

1. 함수형 빈 타입:
   Function<I,O>: 읽기 + 변환 + 쓰기
   Consumer<I>:   읽기 + 처리
   Supplier<O>:   생성 + 쓰기

2. 바인딩 이름 규칙:
   {functionName}-in-0 → 입력 토픽
   {functionName}-out-0 → 출력 토픽

3. 필수 설정:
   destination: 실제 토픽 이름
   group: Consumer Group (재시작 후 재처리 방지)

4. Kafka Binder 내부:
   KafkaMessageListenerContainer + KafkaTemplate 자동 구성
   Function 빈 실행을 MessageHandlerAdapter가 연결

5. 바인더 교체:
   pom.xml 의존성 변경 + application.yml 바인더 설정
   비즈니스 코드 변경 없음 (추상화의 핵심)
```

---

## 🤔 생각해볼 문제

**Q1. Spring Cloud Stream에서 Consumer Group을 설정하지 않으면 어떤 일이 발생하나요?**

<details>
<summary>해설 보기</summary>

`group`을 설정하지 않으면 애플리케이션이 시작될 때마다 새로운 랜덤 Consumer Group ID가 생성됩니다. 이로 인해 두 가지 문제가 발생합니다.

첫째, `auto.offset.reset` 기본값(`latest`)에 따라 이전에 처리한 메시지는 처리하지 않습니다. 재시작 후 기존 메시지를 놓칩니다.

둘째, 여러 인스턴스를 실행할 때 각 인스턴스가 독립적인 그룹으로 동작하여 모든 메시지가 모든 인스턴스에서 처리됩니다. 이것이 의도된 동작(브로드캐스트)이라면 문제없지만, 일반적인 경쟁 소비(Competition Consumer) 패턴에는 부적합합니다.

운영 환경에서는 반드시 고유하고 의미 있는 그룹 ID를 설정하세요.

</details>

---

**Q2. `Function<Message<IN>, Message<OUT>>`과 `Function<IN, OUT>`의 차이는 무엇인가요?**

<details>
<summary>해설 보기</summary>

`Function<IN, OUT>`은 페이로드만 처리합니다. Spring Cloud Stream이 자동으로 역직렬화/직렬화하고, Kafka 헤더는 자동으로 전파됩니다.

`Function<Message<IN>, Message<OUT>>`은 메시지 전체(페이로드 + 헤더)를 처리합니다. Kafka 헤더(`kafka_receivedTopic`, `kafka_offset` 등)에 접근하거나, 출력 메시지에 커스텀 헤더를 추가해야 할 때 사용합니다.

실무에서는 간단한 변환이면 `Function<IN, OUT>`, 헤더 조작이나 라우팅이 필요하면 `Function<Message<IN>, Message<OUT>>`을 사용합니다. 출력 토픽을 동적으로 결정하려면 출력 헤더에 `spring.cloud.stream.sendto.destination`을 설정할 수 있습니다.

</details>

---

**Q3. Spring Cloud Stream으로 구현한 파이프라인에서 Spring Kafka의 `DefaultErrorHandler`를 사용할 수 있나요?**

<details>
<summary>해설 보기</summary>

네, 가능합니다. Kafka Binder는 내부적으로 `KafkaMessageListenerContainer`를 사용하기 때문에 Spring Kafka의 에러 처리 설정을 그대로 사용할 수 있습니다.

Kafka Binder 설정에서 `errorHandlerBeanName`으로 `DefaultErrorHandler` 빈을 지정합니다:

```yaml
spring:
  cloud:
    stream:
      kafka:
        bindings:
          process-in-0:
            consumer:
              errorHandlerBeanName: myErrorHandler
```

또는 `ListenerContainerCustomizer` 빈을 등록해서 컨테이너를 직접 커스텀합니다:

```java
@Bean
public ListenerContainerCustomizer<AbstractMessageListenerContainer<?, ?>>
        customizer(DefaultErrorHandler errorHandler) {
    return (container, dest, group) -> 
        container.setCommonErrorHandler(errorHandler);
}
```

`@RetryableTopic` 어노테이션도 Spring Cloud Stream 함수형 빈에서 사용할 수 있습니다. 단, `@KafkaListener`와 함께 쓸 때보다 설정 방법이 약간 다릅니다.

</details>

---

<div align="center">

**[⬅️ 이전: 트랜잭션 처리](./03-transaction-processing.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Batch + Kafka ➡️](./05-spring-batch-kafka.md)**

</div>
