# Kafka 모니터링 — JMX 메트릭과 Grafana 대시보드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Kafka 운영에 반드시 모니터링해야 하는 핵심 JMX 메트릭은 무엇인가?
- `UnderReplicatedPartitions`, `OfflinePartitionsCount`가 0이 아닐 때 무슨 의미인가?
- `RequestHandlerAvgIdlePercent`가 낮아질 때 어떤 조치가 필요한가?
- kafka-exporter + Prometheus + Grafana 스택을 구성하는 방법은?
- Consumer Lag 알람을 어떻게 설정하는가?
- 브로커, Producer, Consumer 각 레이어에서 확인해야 할 지표는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka는 "모니터링하지 않으면 죽어가는 줄도 모른다"는 특성이 있다. 브로커 디스크가 95%를 넘기거나 Under-Replicated Partition이 증가하는데 아무도 모르다가 장애가 터지는 사고가 실제로 자주 발생한다.

메트릭을 알아야:
- 장애 전에 예방 조치 (디스크 증설, 브로커 추가)
- 장애 발생 시 원인 즉시 파악 (Producer 문제? 브로커 문제? Consumer 문제?)
- 튜닝 효과를 수치로 검증

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Consumer Lag만 모니터링하고 브로커 메트릭 무시

  증상: Consumer Lag 급증
  잘못된 진단: "Consumer가 느리다" → Consumer 스케일 업
  실제 원인:
    브로커의 RequestHandlerAvgIdlePercent가 20% 이하
    브로커 CPU가 요청 처리 병목
    → Consumer를 늘려도 브로커가 못 따라감
    → 브로커 수 증가 또는 파티션 재분산이 해결책

실수 2: UnderReplicatedPartitions 알람 없음

  UnderReplicatedPartitions = ISR이 replication.factor 미달인 파티션 수
  이 수치가 0보다 크면 복제가 진행 중이거나 브로커 장애 상태

  알람 없이 방치 →
    조용히 증가 → 특정 파티션의 ISR이 1개만 남음
    → acks=all이지만 브로커 1대에만 기록되는 상황
    → 해당 브로커 장애 시 데이터 유실

실수 3: 브로커 디스크 사용률을 모니터링 안 함

  디스크 100% 도달 → 브로커 로그 파일 쓰기 실패 → 브로커 다운
  재시작해도 디스크 부족으로 정상화 불가
  운영 중단 시간 길어짐

  운영 기준:
    디스크 80% → 경고 알람
    디스크 90% → 심각 알람 + retention 줄이기 또는 디스크 확장
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
핵심 알람 목록 (우선순위 순):

  [P1 Critical - 즉시 대응]
  1. OfflinePartitionsCount > 0
     → 서비스 중단 가능, 즉시 원인 파악
  2. 브로커 디스크 사용률 > 90%
     → 브로커 다운 임박
  3. UnderReplicatedPartitions > 0 (5분 이상 지속)
     → 데이터 내구성 위험

  [P2 Warning - 30분 내 대응]
  4. Consumer Lag 증가 추세 (5분간 50% 이상)
  5. RequestHandlerAvgIdlePercent < 30%
     → 브로커 처리 능력 한계 근접
  6. BytesInPerSec > 브로커 네트워크 대역폭의 80%

  [P3 Info - 모니터링]
  7. 리밸런싱 빈도 증가
  8. Producer 에러율 > 0
  9. 브로커 GC 시간 > 1초/분
```

---

## 🔬 내부 동작 원리

### 1. 핵심 브로커 JMX 메트릭

```
브로커 상태 지표:

  kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
    값: 0이 정상
    > 0: ISR이 replication.factor 미달인 파티션 존재
    원인: 브로커 장애 / 네트워크 지연 / 디스크 I/O 병목
    조치: 해당 브로커 상태 확인, kafka-topics --under-replicated

  kafka.controller:type=KafkaController,name=OfflinePartitionsCount
    값: 0이 정상
    > 0: ISR이 없어 Leader 선출 불가한 파티션 (서비스 중단!)
    조치: 긴급 대응, 브로커 복구 또는 unclean election 검토

  kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
    값: 브로커로 들어오는 초당 바이트
    사용: 브로커 부하 모니터링, 네트워크 대역폭 계획

  kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec
    값: 브로커에서 나가는 초당 바이트 (Consumer + Follower)
    주의: Consumer 수 × 처리량 + Follower 복제량

  kafka.network:type=RequestMetrics,name=RequestsPerSec,request=Produce
    kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchConsumer
    kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchFollower
    값: 요청 유형별 초당 처리 수

  kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent
    값: 요청 처리 스레드 유휴 비율 (0~1)
    0.7 = 70% 유휴 (정상)
    0.2 = 20% 유휴 (위험, 브로커 요청 처리 병목)
    조치: num.network.threads, num.io.threads 증가 또는 브로커 추가
```

### 2. 핵심 Producer / Consumer JMX 메트릭

```
Producer 지표:
  kafka.producer:type=producer-metrics,client-id=*
    record-error-rate:      전송 실패율 (0이 정상)
    record-retry-rate:      재시도율 (높으면 브로커/네트워크 문제)
    request-latency-avg:    평균 브로커 요청 지연 (ms)
    batch-size-avg:         평균 배치 크기 (낮으면 linger.ms 증가 고려)
    compression-rate-avg:   압축률 (낮을수록 압축 효과 좋음)
    buffer-available-bytes: 버퍼 여유 공간 (0에 가까우면 max.block.ms 위험)

Consumer 지표:
  kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*
    records-lag:        현재 Lag (파티션별)
    records-lag-max:    최대 Lag (가장 뒤처진 파티션)
    fetch-rate:         초당 Fetch 요청 수
    fetch-size-avg:     평균 Fetch 크기 (낮으면 fetch.min.bytes 증가 고려)
    records-consumed-rate: 초당 처리 레코드 수

  kafka.consumer:type=consumer-coordinator-metrics,client-id=*
    rebalance-rate:                 초당 리밸런싱 횟수 (0이 정상)
    last-rebalance-seconds-ago:     마지막 리밸런싱 경과 시간
    commit-rate:                    초당 offset 커밋 수
```

### 3. kafka-exporter + Prometheus + Grafana 구성

```yaml
# docker-compose.yml에 추가
  kafka-exporter:
    image: danielqsj/kafka-exporter:latest
    command:
      - "--kafka.server=kafka-1:9092"
      - "--kafka.server=kafka-2:9092"
      - "--kafka.server=kafka-3:9092"
    ports:
      - "9308:9308"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

# prometheus.yml
scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-exporter:9308']
    scrape_interval: 15s
```

### 4. 핵심 Prometheus 쿼리

```promql
# Consumer Lag (그룹, 토픽, 파티션별)
kafka_consumergroup_lag{consumergroup="order-group"}

# Under-Replicated 파티션 수
kafka_topic_partition_under_replicated_partition

# 브로커 디스크 사용률 (kafka-exporter가 아닌 node-exporter 필요)
(node_filesystem_size_bytes - node_filesystem_avail_bytes)
  / node_filesystem_size_bytes * 100

# 초당 메시지 수신량 (토픽별)
rate(kafka_topic_partition_current_offset[1m])

# Consumer Lag 증가 추세 알람 (5분간 1000 이상 증가)
increase(kafka_consumergroup_lag[5m]) > 1000
```

### 5. Grafana 대시보드 핵심 패널 구성

```
권장 대시보드 구조:

  Row 1: 클러스터 상태 (Overview)
    패널 1: OfflinePartitionsCount (숫자 패널, 0이면 초록)
    패널 2: UnderReplicatedPartitions (숫자 패널, 0이면 초록)
    패널 3: 활성 브로커 수

  Row 2: 처리량
    패널 4: BytesInPerSec (브로커별 시계열)
    패널 5: BytesOutPerSec (브로커별 시계열)
    패널 6: MessagesInPerSec (토픽별)

  Row 3: Consumer Lag
    패널 7: Consumer Group Lag (그룹별 히트맵)
    패널 8: 파티션별 Lag (테이블)
    패널 9: Lag 변화율 (시계열, 증가 추세 감지)

  Row 4: 브로커 성능
    패널 10: RequestHandlerAvgIdlePercent (낮으면 빨간색)
    패널 11: 요청 지연 P99 (Producer, Consumer)
    패널 12: 브로커 JVM GC 시간

참고 대시보드:
  Grafana Dashboard ID 721: Kafka Overview
  Grafana Dashboard ID 7589: Kafka Exporter Overview
  → Grafana에서 "Import dashboard" → ID 입력
```

---

## 💻 실전 실험

### 실험 1: JMX 메트릭 직접 확인

```bash
# 브로커 JMX 포트 활성화 (docker-compose.yml)
# KAFKA_JMX_PORT=9999
# KAFKA_JMX_HOSTNAME=localhost

# jmxterm으로 직접 확인
java -jar jmxterm.jar
open localhost:9999

# UnderReplicatedPartitions 확인
bean kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
run get Value

# RequestHandlerAvgIdlePercent 확인
bean kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent
run get Value

# BytesInPerSec 확인
bean kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
run get OneMinuteRate
```

### 실험 2: Prometheus + Grafana 구성

```bash
# docker-compose up으로 전체 스택 시작
docker-compose up -d

# Prometheus 타겟 확인
curl http://localhost:9090/api/v1/targets | python3 -m json.tool

# kafka-exporter 메트릭 확인
curl http://localhost:9308/metrics | grep kafka_consumergroup_lag

# Grafana에서 대시보드 임포트
# http://localhost:3000 (admin/admin)
# Dashboard → Import → ID: 7589
```

### 실험 3: Consumer Lag 알람 설정

```yaml
# alertmanager.yml (Prometheus 알람 규칙)
groups:
  - name: kafka_alerts
    rules:
      - alert: KafkaConsumerLagHigh
        expr: kafka_consumergroup_lag > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Consumer Lag 급증 - {{ $labels.consumergroup }}"

      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_topic_partition_under_replicated_partition > 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Under-Replicated Partition 발생"

      - alert: KafkaOfflinePartitions
        expr: kafka_brokers < 3
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "브로커 다운 감지"
```

---

## 📊 성능/비용 비교

### 모니터링 스택별 특성

```
기본 CLI 모니터링 (kafka-consumer-groups, kafka-topics):
  비용:   없음
  가시성: 낮음 (매번 수동 실행)
  알람:   없음
  적합:  개발 환경, 즉각적인 진단

JMX + 사내 모니터링 시스템 연동:
  비용:   기존 인프라 활용
  가시성: 중간
  알람:   기존 알람 시스템과 통합

kafka-exporter + Prometheus + Grafana:
  비용:   서버 1~2대 추가 (Prometheus, Grafana)
  가시성: 높음 (시각적 대시보드)
  알람:   Alertmanager로 Slack/PagerDuty 통합
  적합:  운영 환경 표준

Confluent Cloud 또는 MSK (클라우드 관리형):
  비용:   높음 (관리형 서비스 비용)
  가시성: 높음 (내장 모니터링)
  알람:   클라우드 알람 통합
  적합:  운영 관리 부담을 줄이고 싶은 경우
```

---

## ⚖️ 트레이드오프

```
모니터링 깊이:
  적게 (Consumer Lag만):
    ✅ 구성 단순
    ❌ 브로커 문제 늦게 발견
    ❌ 원인 진단 어려움

  많게 (브로커 + Producer + Consumer 전층):
    ✅ 조기 문제 감지
    ✅ 정확한 원인 진단
    ❌ 모니터링 인프라 구성/운영 비용
    ❌ 알람 피로도 (너무 많은 알람)

알람 임계값:
  너무 민감 (낮은 임계값):
    잦은 알람 → 알람 피로도 → 무시하게 됨
  너무 둔감 (높은 임계값):
    장애 후 알람 → 사후 대응만 가능

권장:
  P1(Critical): 즉시 대응 필요한 장애 지표만
  P2(Warning):  예방 조치 가능한 지표
  P3(Info):     추세 분석용
```

---

## 📌 핵심 정리

```
Kafka 모니터링 핵심:

1. 브로커 최우선 지표 (P1 알람):
   OfflinePartitionsCount > 0 → 서비스 중단
   UnderReplicatedPartitions > 0 (5분 이상) → 내구성 위험
   브로커 디스크 > 90% → 브로커 다운 임박

2. 브로커 성능 지표:
   RequestHandlerAvgIdlePercent < 30% → 브로커 과부하
   BytesInPerSec → 네트워크 대역폭 계획

3. Consumer 지표:
   records-lag-max → 가장 뒤처진 파티션의 Lag
   rebalance-rate → 리밸런싱 빈도 (0 정상)

4. 모니터링 스택:
   kafka-exporter + Prometheus + Grafana (운영 표준)
   Dashboard ID 7589 (Kafka Exporter Overview)

5. 알람 계층:
   Critical: OfflinePartitions, UnderReplicated, 디스크 90%
   Warning: Consumer Lag 급증, IdlePercent < 30%
   Info:    추세 모니터링
```

---

## 🤔 생각해볼 문제

**Q1. `UnderReplicatedPartitions`가 1이면 반드시 브로커 장애인가요?**

<details>
<summary>해설 보기</summary>

꼭 장애는 아닙니다. Under-Replicated 상태는 ISR이 `replication.factor`보다 적은 파티션이 존재한다는 의미로, 다음 상황에서도 발생합니다.

**정상적인 일시 상태**: 브로커 롤링 재시작 중, 브로커가 재시작 후 ISR에 복귀하는 동안 일시적으로 Under-Replicated 상태가 됩니다. 보통 몇 초~수십 초 내에 0으로 돌아옵니다.

**지속되는 경우가 문제**: 5분 이상 0보다 크면 실제 브로커 장애, 네트워크 단절, 디스크 I/O 병목 등 지속적인 문제가 있을 가능성이 높습니다.

알람 설정 시 `for: 2~5분`으로 설정해서 일시적인 상태와 지속적인 문제를 구분하는 것이 중요합니다.

</details>

---

**Q2. `RequestHandlerAvgIdlePercent`가 낮을 때 어떤 설정을 먼저 확인해야 하나요?**

<details>
<summary>해설 보기</summary>

처리 스레드 수 설정을 먼저 확인합니다.

`num.network.threads`(기본 3): 네트워크 I/O를 담당하는 스레드 수. 초당 요청 수가 많을 때 증가 필요.

`num.io.threads`(기본 8): 실제 요청을 처리하는 스레드 수. `RequestHandlerAvgIdlePercent`가 낮을 때 이 값을 늘립니다. CPU 코어 수에 비례해서 설정(예: CPU 코어 수 × 2).

단, 스레드를 늘리기 전에 **요청 유형**을 분석해야 합니다. `RequestsPerSec`에서 Produce vs Fetch 비율을 보고 어떤 요청이 많은지 파악합니다. 처리량이 이미 한계라면 스레드 증가보다 **브로커 수 증가** 또는 **파티션 재분배**가 근본 해결책입니다.

</details>

---

**Q3. Consumer Lag이 0인 상태에서도 Grafana 대시보드를 봐야 하는 이유가 있나요?**

<details>
<summary>해설 보기</summary>

Lag이 0이어도 미래 문제의 전조를 찾을 수 있습니다.

**추세 분석**: BytesInPerSec가 지속적으로 증가하고 있다면 몇 주 후 브로커 디스크나 네트워크 한계에 도달할 것을 예측하고 용량 계획을 세울 수 있습니다.

**비정상 패턴 감지**: 평소 Lag이 0인데 특정 시간(예: 매일 오전 9시)에 잠깐 100까지 올라갔다 돌아오는 패턴을 발견하면, 배치 작업이 일시적으로 메시지를 폭발적으로 발행하는 것을 사전에 파악할 수 있습니다.

**브로커 불균형 감지**: BytesInPerSec가 특정 브로커에 몰려있다면 파티션 리더 불균형이 있음을 알 수 있습니다. Lag이 0이어도 브로커 과부하가 진행 중일 수 있습니다.

대시보드는 문제 발생 후 진단에만 쓰는 것이 아니라 사전 예방을 위한 지속적인 관찰 도구입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 파티션 핫스팟](./03-partition-hotspot.md)** | **[홈으로 🏠](../README.md)** | **[다음: 운영 문제 패턴 ➡️](./05-operational-patterns.md)**

</div>
