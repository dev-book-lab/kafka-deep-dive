# 운영 중 발생하는 문제 패턴 — 진단과 대응

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Under-Replicated Partitions의 원인을 단계적으로 진단하는 방법은?
- Consumer Group 무한 리밸런싱을 `kafka-consumer-groups`로 어떻게 확인하는가?
- `--reset-offsets`의 `--to-earliest`, `--to-latest`, `--to-datetime` 각각의 사용 시점은?
- 재처리 시 중복 방지를 위해 어떤 준비를 해야 하는가?
- 오래된 Consumer Group의 offset 정보가 만료되면 어떤 일이 발생하는가?
- 브로커 장애 후 복구 절차는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka 운영 장애는 예고 없이 찾아온다. Under-Replicated 파티션이 쌓이거나 Consumer Group이 무한 리밸런싱을 반복할 때 원인을 빠르게 파악하고 올바른 명령어로 대응해야 한다.

잘못된 offset 리셋은 데이터 중복 처리나 데이터 유실을 야기한다. 재처리 전 반드시 소비 시스템의 멱등성을 확인해야 한다. 이 문서는 실제 운영에서 마주치는 패턴과 대응 절차를 담았다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Under-Replicated 발생 시 즉각 브로커 재시작

  잘못된 대응:
    UnderReplicatedPartitions > 0 알람 → 해당 브로커 재시작
    결과: 브로커 재시작 → ISR에서 이탈 → Under-Replicated 더 증가
          복제 진행 중이었던 데이터도 처음부터 재동기화 필요
          복구 시간 2배 이상 증가

  올바른 대응:
    1. 원인 파악 먼저 (디스크? 네트워크? CPU?)
    2. 원인 해소 (디스크 공간 확보, 네트워크 복구)
    3. 브로커가 자연스럽게 ISR 복귀 기다리기
    4. 복귀 안 되면 그때 브로커 재시작

실수 2: --reset-offsets 실행 중 Consumer 실행 상태

  명령어:
    kafka-consumer-groups ... --reset-offsets --to-earliest --execute

  문제:
    Consumer 실행 중에 offset 리셋
    → Consumer가 계속 커밋하면서 리셋된 offset이 다시 덮어씌워짐
    → 리셋 효과 없음 또는 부분적으로만 적용

  올바른 절차:
    1. Consumer Group 완전 중단 확인 (State=Empty)
    2. offset 리셋 실행
    3. 리셋 결과 확인
    4. Consumer 재시작

실수 3: --to-earliest로 전체 재처리 시 멱등성 미확인

  명령어: --reset-offsets --to-earliest
  결과: 처음부터 모든 메시지 재처리
  문제:
    처리 로직이 멱등하지 않으면 중복 데이터 생성
    → DB에 중복 주문, 중복 결제 레코드 생성
    → 돌이키기 매우 어려움

  필수 체크리스트:
    [] 처리 로직이 UPSERT 또는 중복 체크 적용됨?
    [] 이메일/SMS 발송이 재처리 시 중복 발송되지 않는가?
    [] 재처리 완료 후 Lag=0 확인 계획이 있는가?
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
장애 대응 원칙:
  1. 측정 먼저 (메트릭, 로그 확인)
  2. 원인 파악 (가설 → 검증)
  3. 최소 침습적 조치 (재시작은 최후 수단)
  4. 결과 확인 (정상화 검증)

자주 쓰는 진단 명령어 세트:
  # 클러스터 전체 상태
  kafka-topics --bootstrap-server localhost:19092 \
    --describe --under-replicated-partitions

  # Consumer Group 전체 목록
  kafka-consumer-groups --bootstrap-server localhost:19092 --list

  # 특정 그룹 상태
  kafka-consumer-groups --bootstrap-server localhost:19092 \
    --describe --group order-group

  # 파티션 크기 확인
  kafka-log-dirs --bootstrap-server localhost:19092 \
    --topic-list orders --describe

  # 브로커 구성 확인
  kafka-configs --bootstrap-server localhost:19092 \
    --entity-type brokers --entity-name 1 --describe
```

---

## 🔬 내부 동작 원리

### 1. Under-Replicated Partitions 진단 트리

```
UnderReplicatedPartitions > 0 알람 발생:

  Step 1. 어느 파티션이 Under-Replicated?
    kafka-topics --describe --under-replicated-partitions
    출력: Partition 3: Leader: 2, Replicas: 1,2,3, Isr: 2,3
    → Broker 1이 ISR에서 이탈

  Step 2. Broker 1 상태 확인
    브로커가 살아있는가?
      NO: 브로커 다운 → 복구 절차
      YES: 다음 단계

  Step 3. Broker 1의 복제 지연 원인
    디스크 I/O 병목?
      kafka-log-dirs 파티션 크기 확인
      iostat, df -h 확인

    네트워크 지연?
      ping kafka-1, iperf3로 브로커 간 대역폭 측정

    JVM GC 지연?
      Broker 1 GC 로그 확인
      jstat -gcutil <PID> 1s

  Step 4. 자연 복구 기다리기
    원인 해소 후 replica.lag.time.max.ms(30초) 내에 복귀
    kafka-topics --describe 반복 확인

  Step 5. 자연 복구 안 되면
    kafka-preferred-replica-election으로 리더 재조정
    또는 해당 브로커 재시작 (최후 수단)
```

### 2. Consumer Group 무한 리밸런싱 진단

```
증상:
  kafka-consumer-groups --describe → State: PreparingRebalance 반복
  Consumer 로그: "Rebalance..." 반복
  Consumer Lag: 처리 없이 계속 증가

  Step 1. Group 상태 확인
    kafka-consumer-groups --describe --group order-group
    # State: PreparingRebalance / CompletingRebalance가 반복되는지 확인

  Step 2. Consumer 상태 확인 (CONSUMER-ID 확인)
    # 각 Consumer의 호스트와 client-id 확인
    # 특정 Consumer가 계속 참여/이탈 반복하는지 확인

  Step 3. 원인별 대응

    session.timeout.ms 초과 (heartbeat 문제):
      Consumer 로그: "Consumer session timed out"
      대응: session.timeout.ms 증가 또는 G1GC 설정

    max.poll.interval.ms 초과 (처리 지연):
      Consumer 로그: "poll() interval exceeded timeout"
      대응: max.poll.records 감소 또는 max.poll.interval.ms 증가

    Consumer 수 > 파티션 수:
      IDLE Consumer가 있음 → 참여 시도 반복 아닌지 확인

    Cooperative 전략 미적용:
      현재 RangeAssignor(Eager) → CooperativeStickyAssignor 전환 권장
```

### 3. Offset 리셋 절차

```
Offset 리셋 전 준비:
  1. Consumer Group 완전 중단
     kafka-consumer-groups --describe → State=Empty 확인
     (Active Member가 있으면 리셋 후 덮어씌워질 수 있음)

  2. 리셋 대상 토픽/파티션 확인
     kafka-consumer-groups --describe --group order-group

  3. 멱등성 확인
     재처리 로직이 중복에 안전한지 검토

리셋 옵션:
  --to-earliest:
    파티션의 첫 번째 offset으로 이동 (전체 재처리)
    사용 케이스: 데이터 파이프라인 전체 재구성

  --to-latest:
    파티션의 마지막 offset으로 이동 (이후 메시지만 처리)
    사용 케이스: 과거 데이터 무시하고 새 메시지부터 처리

  --to-offset <N>:
    특정 offset으로 이동
    사용 케이스: 특정 시점 이후 재처리

  --to-datetime <YYYY-MM-DDTHH:mm:SS.sss>:
    특정 시각 이후 메시지부터 처리
    사용 케이스: 특정 장애 시점 이후만 재처리
    예: --to-datetime 2024-01-15T09:00:00.000

  --by-duration PT1H:
    현재 시각에서 1시간 전으로 이동
    사용 케이스: 최근 1시간치 재처리

실행 절차:
  # 1. Dry-run (실제 변경 없이 확인)
  kafka-consumer-groups --bootstrap-server localhost:19092 \
    --group order-group \
    --topic orders \
    --reset-offsets --to-earliest

  # 2. 결과 확인 후 실제 실행
  kafka-consumer-groups --bootstrap-server localhost:19092 \
    --group order-group \
    --topic orders \
    --reset-offsets --to-earliest --execute

  # 3. 리셋 결과 확인
  kafka-consumer-groups --bootstrap-server localhost:19092 \
    --describe --group order-group
```

### 4. 브로커 장애 복구 절차

```
브로커 다운 시 즉각 조치:

  Step 1. 영향 범위 파악
    kafka-topics --under-replicated-partitions
    → 영향받은 파티션 수 확인

  Step 2. 파티션 서비스 가능 여부 확인
    kafka-topics --describe --under-min-isr-partitions
    → NotEnoughReplicas 에러 발생 파티션 확인
    → min.insync.replicas 미달 파티션은 쓰기 불가

  Step 3. 긴급 조치 (필요 시)
    단기간 내 복구 불가 + 서비스 중단이 더 심각할 경우:
    해당 토픽의 min.insync.replicas를 일시적으로 1로 낮추기
    (내구성 약화를 감수하고 서비스 가용성 확보)

  Step 4. 브로커 복구
    장비 교체 / 재시작 / 디스크 복구 등

  Step 5. ISR 복귀 확인
    복구된 브로커가 각 파티션의 ISR에 복귀하는 것 모니터링
    kafka-topics --describe로 Isr 목록 변화 확인

  Step 6. 긴급 조치 원복
    min.insync.replicas를 원래 값으로 복구
    kafka-configs --alter --delete-config min.insync.replicas
```

### 5. offset 만료 문제

```
offsets.retention.minutes=10080 (7일, 기본값):
  Consumer Group이 7일 동안 활동 없으면 offset 정보 삭제

  문제 시나리오:
    주간 배치 Consumer Group: 매주 토요일 실행
    2주 이상 미실행 → offset 만료
    재실행 시 auto.offset.reset=latest → 최신부터 읽기 시작
    → 2주치 메시지 처리 안 됨

  해결책:
    1. offsets.retention.minutes 증가
       중요 배치 그룹은 충분히 크게 설정

    2. 주기적 offset 갱신
       Consumer를 주기적으로 실행해서 offset 유지
       또는 더미 poll() + commit으로 offset 갱신

    3. --reset-offsets --to-datetime으로 재처리
       마지막으로 처리한 시각부터 재처리

Consumer Group 정리:
  오래된 Empty 그룹 삭제:
    kafka-consumer-groups --bootstrap-server localhost:19092 \
      --delete --group old-unused-group
```

---

## 💻 실전 실험

### 실험 1: Under-Replicated Partitions 복구 시뮬레이션

```bash
# 브로커 1 중단
docker stop kafka-1
sleep 35  # lag 타임아웃 대기

# Under-Replicated 파티션 확인
kafka-topics --bootstrap-server localhost:19092 \
  --describe --under-replicated-partitions

# Broker 1 복구
docker start kafka-1

# ISR 복귀 모니터링 (5초마다)
watch -n 5 'kafka-topics --bootstrap-server localhost:19092 \
  --describe --under-replicated-partitions'
# 빈 출력이 되면 복구 완료
```

### 실험 2: --to-datetime으로 특정 시점 재처리

```bash
# 현재 시각 기록
REPROCESS_FROM=$(date -u +"%Y-%m-%dT%H:%M:%S.000")

# 메시지 100개 발행 (이 시점 이후)
for i in $(seq 1 100); do
  echo "msg-$i"
done | kafka-console-producer \
  --bootstrap-server localhost:19092 --topic orders

# 정상 소비 후 Group State=Empty 확인
kafka-console-consumer --bootstrap-server localhost:19092 \
  --topic orders --group reprocess-group --max-messages 100

# offset을 특정 시점으로 리셋
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --group reprocess-group \
  --topic orders \
  --reset-offsets --to-datetime ${REPROCESS_FROM} --execute

# 재처리 확인
kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group reprocess-group
# CURRENT-OFFSET이 해당 시점의 offset으로 변경됨
```

### 실험 3: 무한 리밸런싱 진단 실습

```bash
# max.poll.interval.ms를 매우 짧게 설정 (리밸런싱 유발)
kafka-console-consumer \
  --bootstrap-server localhost:19092 \
  --topic orders \
  --group rebalance-test \
  --consumer-property max.poll.interval.ms=5000 &

# 5초마다 State 변화 확인
watch -n 2 'kafka-consumer-groups --bootstrap-server localhost:19092 \
  --describe --group rebalance-test | grep -E "State|CONSUMER-ID"'
# State가 PreparingRebalance와 Stable 사이를 반복하는 것 확인
```

---

## 📊 성능/비용 비교

### 운영 장애 유형별 복구 시간

```
Under-Replicated Partitions (브로커 1대 장애):
  자연 복구: 재시작 후 재동기화 시간 = 파티션 데이터 크기 / 복제 속도
  예: 100 GB 데이터, 복제 속도 100 MB/s → ~17분 복구
  서비스 영향: min.insync.replicas 기준 (1대 장애 허용 시 서비스 유지)

Consumer Group 무한 리밸런싱:
  원인 파악 + 설정 수정 + 재배포: ~10~30분
  서비스 영향: 처리 중단 (Lag 급증)

Offset 잘못 리셋 (전체 재처리):
  재처리 시간 = Lag 크기 / Consumer 처리 속도
  예: 1,000만 레코드, 처리 속도 10만/분 → ~100분 재처리
  서비스 영향: 처리 자원 점유, 일부 시스템 중복 처리 위험

브로커 디스크 100% 도달:
  디스크 확장 전까지 브로커 다운 → 즉각 대응 필요
  예방: 80% 알람 설정, retention 조정
```

---

## ⚖️ 트레이드오프

```
긴급 조치 (min.insync.replicas 임시 낮추기):
  ✅ 서비스 가용성 즉각 회복
  ❌ 내구성 약화 (일부 브로커 장애 시 데이터 유실 위험)
  → 원복 필수, 감사 로그 남기기

--to-earliest 전체 재처리:
  ✅ 데이터 완전성 보장 (모든 이력 재처리)
  ❌ 처리 시간 길고, 중복 처리 위험
  → 멱등 처리 확인 후 실행

--to-latest (과거 건너뜀):
  ✅ 즉각 현재 메시지부터 처리 재개
  ❌ 건너뛴 메시지 처리 안 됨 (데이터 유실)
  → 건너뛴 데이터가 비즈니스 관점에서 중요하지 않을 때만

CooperativeStickyAssignor 전환:
  ✅ 무한 리밸런싱 근본 해결
  ❌ 마이그레이션 필요 (잠깐의 리밸런싱 발생)
  → 트래픽 낮은 시간에 적용
```

---

## 📌 핵심 정리

```
운영 장애 대응 핵심:

1. Under-Replicated Partitions:
   측정 → 원인 파악 → 자연 복구 기다리기 → 최후에 재시작
   절대 즉각 재시작 금지

2. Consumer Group 무한 리밸런싱:
   --describe로 State 확인
   session.timeout.ms 초과 vs max.poll.interval.ms 초과 구분
   CooperativeStickyAssignor 전환으로 근본 해결

3. Offset 리셋 황금 규칙:
   Consumer Group 완전 중단 (State=Empty) 후 실행
   Dry-run 먼저 → 결과 확인 → 실제 실행
   멱등성 확인 후 재처리

4. Offset 리셋 옵션:
   --to-earliest: 전체 재처리 (신중하게)
   --to-latest: 과거 건너뜀 (데이터 유실)
   --to-datetime: 특정 시점부터 재처리 (가장 실용적)

5. 브로커 장애 복구:
   ISR 복귀 모니터링 → 긴급 시 min.insync.replicas 임시 조정
   복구 후 반드시 원복
```

---

## 🤔 생각해볼 문제

**Q1. Under-Replicated Partitions가 갑자기 30개 발생했습니다. 브로커는 모두 살아있습니다. 무엇을 먼저 확인해야 하나요?**

<details>
<summary>해설 보기</summary>

브로커가 모두 살아있지만 Under-Replicated가 발생했다면 **복제 지연**이 원인일 가능성이 높습니다.

확인 순서: (1) `kafka-topics --describe --under-replicated-partitions`로 어느 브로커가 ISR에서 이탈했는지 확인합니다. (2) 해당 브로커의 `replica.lag.time.max.ms` 동안 Fetch 요청을 안 보냈을 가능성 — 브로커 로그에서 Follower Fetch 활동 확인합니다. (3) 해당 브로커의 JMX에서 `MaxLag`(replica fetcher manager)가 높은지 확인합니다. (4) 디스크 I/O 포화 여부 — `iostat`으로 해당 브로커의 디스크 사용률 확인합니다.

갑자기 30개 파티션이 동시에 문제가 생겼다면, 특정 브로커 하나가 I/O 포화 또는 네트워크 이슈로 Follower Fetch가 지연된 것일 가능성이 큽니다. `num.replica.fetchers`를 늘려서 복제 스레드를 증가시키거나, 디스크 I/O 병목을 해소하면 자연 복귀됩니다.

</details>

---

**Q2. `--reset-offsets --to-datetime`을 실행했는데 해당 시각의 offset을 찾지 못하는 경우가 있나요?**

<details>
<summary>해설 보기</summary>

네, 두 가지 경우에 발생합니다.

**retention 만료**: 지정한 시각의 메시지가 `retention.ms`가 지나서 이미 삭제됐다면 해당 timestamp를 가진 segment가 없습니다. 이 경우 가장 오래된 남아있는 offset(`--to-earliest`)으로 대체됩니다.

**Future timestamp**: 지정한 시각이 현재보다 미래이면 가장 최신 offset(`--to-latest`)으로 대체됩니다.

Kafka는 `ListOffsets API`를 사용해서 timestamp 기반으로 offset을 찾습니다. 내부적으로 `.timeindex` 파일을 조회해서 해당 timestamp에 가장 가까운 offset을 반환합니다. 정확한 timestamp가 없으면 해당 timestamp 이후 첫 번째 메시지의 offset을 반환합니다.

</details>

---

**Q3. Consumer Group을 삭제(`--delete`)하면 어떤 영향이 있나요?**

<details>
<summary>해설 보기</summary>

Consumer Group 삭제는 `__consumer_offsets` 토픽에서 해당 그룹의 offset 정보를 제거합니다. 이후 영향:

**즉각적 영향**: 해당 그룹 ID로 Consumer를 다시 시작하면 `auto.offset.reset` 설정에 따라 `earliest` 또는 `latest`부터 읽기 시작합니다. 이전에 처리하던 offset 정보가 없어집니다.

**실행 중 Consumer 영향**: 실행 중인 Consumer가 있는 상태에서 그룹을 삭제하면 에러가 발생합니다. `kafka-consumer-groups` CLI는 State=Empty(활성 멤버 없음)인 그룹만 삭제를 허용합니다.

**위험**: 그룹을 잘못 삭제하면 해당 Consumer가 다음 실행 시 처음부터 재처리하거나 최신부터 읽어서 데이터 처리 공백이 생깁니다. 삭제 전 반드시 그룹 이름과 토픽을 확인하고, 필요하다면 현재 offset을 기록해두세요.

</details>

---

<div align="center">

**[⬅️ 이전: Kafka 모니터링](./04-kafka-monitoring.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — Kafka Streams 아키텍처 ➡️](../kafka-streams-event/01-kafka-streams-architecture.md)**

</div>
