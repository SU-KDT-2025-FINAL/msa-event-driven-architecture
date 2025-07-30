# MSA 이벤트 기반 아키텍처 학습 로드맵

## 1단계: 기초

### 핵심 개념 및 이론
- **이벤트 기반 아키텍처 기초**
  - 이벤트, 생산자, 소비자, 브로커
  - 장점: 느슨한 결합, 확장성, 회복력
  - 도메인 이벤트 vs 시스템 이벤트

- **MSA 기본 원칙**
  - 마이크로서비스 통신 패턴
  - 동기 vs 비동기 통신
  - 서비스 경계 및 설계

- **핵심 패턴**
  - 이벤트 소싱 개념
  - CQRS (명령 쿼리 책임 분리)
  - 사가 패턴 소개

## 2단계: 기술 스택

### Apache Kafka 심화
- Kafka 클러스터 설정 및 운영
- 파티셔닝 전략 및 컨슈머 그룹
- 오프셋 관리 및 재처리
- Kafka Connect 데이터 파이프라인

### Spring Boot 통합
- Spring Kafka 설정 및 최적화
- @KafkaListener 어노테이션
- KafkaTemplate 이벤트 발행
- Spring Cloud Stream 함수형 프로그래밍

### 스키마 관리
- Confluent Schema Registry 구축
- Apache Avro 스키마 설계
- 스키마 진화 전략 (하위/상위 호환성)
- Protocol Buffers 대안

## 3단계: 모니터링 및 운영

### 구조화된 로깅
- JSON 통합 로그 포맷 설계
- MDC 상관관계 추적 (traceId, spanId)
- Logback 롤링 정책
- 이벤트 처리 성능 로깅

### ELK Stack 구현
- Elasticsearch 클러스터 구성
- Logstash 파이프라인 설계
- Filebeat 로그 수집
- Kibana 대시보드 생성

### 분산 추적
- Spring Cloud Sleuth + Zipkin 구축
- OpenTelemetry 도입
- 서비스 간 요청 추적
- 성능 병목 지점 식별

## 4단계: 고급 패턴

### 이벤트 소싱 구현
- EventStore 데이터베이스 설계
- 이벤트 재생 메커니즘
- 스냅샷 생성 전략
- 이벤트 버전 관리

### CQRS 패턴 적용
- Command/Query 모델 분리
- 읽기 전용 프로젝션 구성
- 데이터 동기화 전략
- 성능 최적화

### 사가 패턴 구현
- Orchestration vs Choreography 선택
- 보상 트랜잭션 설계
- 분산 트랜잭션 상태 관리
- 실패 시나리오 처리

## 5단계: 실전 프로젝트

### 전자상거래 시스템 구현
- User, Order, Payment, Inventory 서비스 설계
- 주문 생성부터 결제 완료까지 이벤트 플로우
- 재고 예약 및 해제 로직
- 알림 서비스 통합

### 대용량 데이터 처리
- Kafka Streams 실시간 분석
- 이벤트 기반 ETL 파이프라인
- 배치/스트림 처리 하이브리드
- 데이터 품질 모니터링

## 6단계: DevOps 및 운영

### 컨테이너화 및 오케스트레이션
- Docker 서비스 컨테이너화
- Kubernetes MSA 배포
- Helm Chart 배포 자동화
- Service Mesh (Istio) 도입

### CI/CD 파이프라인
- GitLab/GitHub Actions 파이프라인
- 마이크로서비스별 독립 배포
- 컨테이너 이미지 빌드/배포 자동화
- Blue-Green/Canary 배포 전략

### 성능 및 부하 테스트
- JMeter/Gatling 성능 테스트
- Kafka 파티션 튜닝
- JVM 최적화
- 리소스 사용량 모니터링

## 7단계: 보안 및 거버넌스

### 이벤트 보안
- Kafka SSL/SASL 설정
- 이벤트 암호화 및 서명
- 마이크로서비스 간 인증/인가
- API Gateway 접근 제어

### 데이터 거버넌스
- GDPR 준수를 위한 개인정보 처리
- 이벤트 데이터 보관 정책
- 감사 로그 관리
- 데이터 품질 관리