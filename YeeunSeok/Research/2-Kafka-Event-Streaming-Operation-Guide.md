# Kafka 이벤트 스트리밍 아키텍처 운영 가이드

이 문서는 Kafka 기반 이벤트 스트리밍 아키텍처의 운영을 위한 실무 가이드입니다. 토픽 설계, 프로듀서/컨슈머 최적화, 스키마 레지스트리, 모니터링, 백업/복구, 성능 튜닝까지 단계별로 정리합니다.

---

## 1. Topic 설계 전략

### 1.1 파티셔닝(Partitioning)
- **병렬 처리 및 확장성** 확보를 위해 파티션 수를 충분히 설정
- 파티션 키 선정: 동일한 키는 동일 파티션에 할당되어 순서 보장
- 파티션 수는 컨슈머 수, 처리량, 브로커 수를 고려해 결정

### 1.2 리플리케이션(Replication)
- **내결함성** 확보를 위해 리플리케이션 팩터(일반적으로 3) 설정
- 최소 ISR(In-Sync Replicas) 설정으로 데이터 내구성 강화

### 1.3 네이밍 규칙
- 일관성 있는 네이밍: `<도메인>.<이벤트유형>`, `<서비스명>.<기능>` 등
- 예시: `order.created`, `payment.completed`, `user.signup`

---

## 2. 프로듀서/컨슈머 최적화 설정

### 2.1 프로듀서
- `acks=all`: 데이터 내구성 보장
- `batch.size`, `linger.ms`: 배치 전송으로 처리량 향상
- `compression.type`: snappy, lz4 등 압축 적용
- `retries`, `max.in.flight.requests.per.connection` 조정

### 2.2 컨슈머
- `enable.auto.commit=false`: 수동 커밋으로 데이터 처리 신뢰성 확보
- `max.poll.records`, `fetch.min.bytes` 등 튜닝
- 컨슈머 그룹을 통한 스케일 아웃
- 재처리(Replay) 시 idempotent 처리 로직 구현

---

## 3. 스키마 레지스트리 운영

- Avro, Protobuf, JSON Schema 등 지원
- 스키마 등록/호환성 정책(Backward/Forward/Full)
- 스키마 버전 관리 및 변경 프로세스 수립
- CI/CD와 연동하여 스키마 변경 자동화

---

## 4. 클러스터 모니터링 및 알람 설정

- **모니터링 도구**: Prometheus + Grafana, Confluent Control Center, Datadog 등
- 주요 지표: lag, throughput, under-replicated partitions, offline partitions, disk usage, JVM metrics
- 알람: lag 임계치, 브로커 다운, 디스크 부족, ISR 감소 등
- JMX Exporter, Kafka Exporter 활용

---

## 5. 백업/복구 전략

- 토픽별 데이터 Retention 정책 설정
- MirrorMaker, Confluent Replicator 등으로 원격지 복제
- 브로커 데이터 디렉토리 주기적 백업
- 메타데이터(ZooKeeper/KRaft) 백업
- 복구 시 토픽/파티션/오프셋 단위로 복원

---

## 6. 성능 튜닝 (JVM/OS 레벨)

### 6.1 JVM 튜닝
- Heap 크기 조정(-Xmx, -Xms)
- GC 알고리즘(G1GC, ZGC 등) 선택
- GC 로그 모니터링 및 Full GC 최소화
- JMX로 JVM 상태 모니터링

### 6.2 OS 레벨 최적화
- 디스크: SSD 사용, RAID 구성, noatime 옵션
- 네트워크: MTU, TCP window size, 커널 파라미터 튜닝
- 파일 디스크립터/메모리 제한 상향 (ulimit)
- Swappiness=1, Transparent Huge Pages 비활성화

---

## 참고 자료
- [Kafka 공식 문서](https://kafka.apache.org/documentation/)
- [Confluent 운영 가이드](https://docs.confluent.io/platform/current/operations/index.html)
- [Kafka JVM 튜닝](https://www.confluent.io/blog/kafka-fastest-messaging-system/)
- [Kafka 모니터링 Best Practice](https://www.datadoghq.com/blog/monitor-kafka-performance-metrics/) 


## MSA 및 Kafka 이벤트 아키텍처 운영의 공통 원칙
1, 2번 두 문서에서 반복적으로 강조된 공통 원칙은 다음과 같습니다:

### 1. Kafka Topic 설계 전략
- 파티셔닝, 리플리케이션, 네이밍 규칙 등 토픽 설계의 중요성
- 병렬 처리, 내결함성, 일관성 있는 네이밍을 통한 운영 효율성 확보

### 2. 이벤트 스키마 및 스키마 레지스트리 관리
- Avro, JSON Schema 등으로 이벤트 구조를 명확히 정의
- 스키마 버전 관리와 호환성 정책 수립의 필요성

### 3. Kafka 기반 이벤트 처리 구조
- Kafka를 이벤트 브로커로 활용하여 마이크로서비스 간 비동기 통신 구현
- Producer/Consumer 패턴의 신뢰성 및 확장성 확보

### 4. 운영 및 모니터링의 중요성
- 클러스터 상태, 지표, 알람 등 운영 관점에서의 모니터링 필수
- 데이터 내구성, 장애 대응, 신뢰성 확보를 위한 운영 전략 필요

이러한 원칙들은 MSA와 Kafka 기반 이벤트 아키텍처 모두에서 안정적이고 확장 가능한 시스템을 구축하는 데 핵심적인 역할을 합니다. 