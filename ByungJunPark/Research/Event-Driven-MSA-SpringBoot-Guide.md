# 이벤트 기반 MSA 아키텍처 - Spring Boot 기술 스택 가이드

## 목차
1. [이벤트 기반 MSA 개요](#이벤트-기반-msa-개요)
2. [Spring Boot 이벤트 처리 라이브러리](#spring-boot-이벤트-처리-라이브러리)
3. [메시지 브로커 비교](#메시지-브로커-비교)
4. [Kafka 상세 명세](#kafka-상세-명세)
5. [Redis 상세 명세](#redis-상세-명세)
6. [기타 메시지 브로커](#기타-메시지-브로커)
7. [아키텍처 패턴](#아키텍처-패턴)
8. [구현 예제](#구현-예제)

## 이벤트 기반 MSA 개요

### 핵심 개념
- **비동기 통신**: 서비스 간 느슨한 결합을 통한 확장성 향상
- **이벤트 소싱**: 상태 변경을 이벤트로 저장하여 시스템 상태 추적
- **CQRS**: 명령과 조회의 분리를 통한 성능 최적화
- **Eventual Consistency**: 최종 일관성을 통한 분산 시스템 설계

### 장점
- 높은 확장성과 가용성
- 서비스 간 독립성 보장
- 장애 격리 및 복구 용이성
- 비즈니스 로직의 명확한 분리

## Spring Boot 이벤트 처리 라이브러리

### 1. Spring Events
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.0</version>
</dependency>
```

**특징:**
- Spring Framework 내장 이벤트 시스템
- `@EventListener`, `@Async` 어노테이션 지원
- 동일 JVM 내에서만 작동

**주요 클래스:**
- `ApplicationEvent`
- `ApplicationEventPublisher`
- `@EventListener`

### 2. Spring Integration
```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-core</artifactId>
    <version>6.0.0</version>
</dependency>
```

**특징:**
- Enterprise Integration Patterns 구현
- 다양한 프로토콜 및 데이터 포맷 지원
- 채널, 게이트웨이, 어댑터 패턴 제공

### 3. Spring Cloud Stream
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream</artifactId>
    <version>4.0.0</version>
</dependency>
```

**특징:**
- 메시지 브로커와의 추상화 레이어 제공
- Kafka, RabbitMQ, Redis 등 다양한 바인더 지원
- 함수형 프로그래밍 모델 지원

### 4. Spring Boot Actuator
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>3.2.0</version>
</dependency>
```

**특징:**
- 애플리케이션 모니터링 및 관리
- 헬스 체크, 메트릭, 트레이싱 기능

### 5. Spring Data
```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jpa</artifactId>
    <version>3.2.0</version>
</dependency>
```

**특징:**
- 이벤트 소싱을 위한 데이터 접근 추상화
- Repository 패턴 구현
- 트랜잭션 관리

## 메시지 브로커 비교

| 특징 | Apache Kafka | Redis Pub/Sub | RabbitMQ | Amazon SQS |
|------|-------------|---------------|----------|------------|
| **처리량** | 매우 높음 | 높음 | 중간 | 높음 |
| **지연시간** | 낮음 | 매우 낮음 | 낮음 | 중간 |
| **내구성** | 높음 | 낮음 | 높음 | 높음 |
| **확장성** | 수평 확장 | 제한적 | 수직 확장 | 완전 관리형 |
| **복잡성** | 높음 | 낮음 | 중간 | 낮음 |
| **용도** | 로그 수집, 스트리밍 | 캐싱, 실시간 알림 | 작업 큐, 라우팅 | 큐 기반 메시징 |

## Kafka 상세 명세

### 핵심 구성요소
- **Producer**: 메시지 발행자
- **Consumer**: 메시지 구독자
- **Topic**: 메시지 카테고리
- **Partition**: 토픽의 물리적 분할
- **Broker**: Kafka 서버 노드
- **Zookeeper**: 클러스터 메타데이터 관리

### Spring Boot with Kafka 의존성
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.0.0</version>
</dependency>
```

### 주요 설정
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.JsonSerializer
      acks: all
      retries: 3
    consumer:
      group-id: user-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.JsonDeserializer
      auto-offset-reset: earliest
```

### 주요 어노테이션
- `@KafkaListener`: 메시지 수신
- `@EnableKafka`: Kafka 활성화
- `@KafkaHandler`: 메시지 타입별 처리

### 성능 특징
- **처리량**: 초당 수백만 메시지 처리 가능
- **저장**: 설정 가능한 보존 정책
- **파티셔닝**: 수평 확장 지원
- **복제**: 장애 복구를 위한 다중 복사본

### 적용 사례
- 실시간 로그 수집
- 이벤트 소싱
- 스트림 프로세싱
- CDC (Change Data Capture)

## Redis 상세 명세

### 핵심 특징
- **In-Memory**: 메모리 기반 고속 처리
- **Pub/Sub**: 발행-구독 메시징 패턴
- **데이터 구조**: String, Hash, List, Set, Sorted Set
- **영속성**: RDB, AOF 백업 옵션

### Spring Boot with Redis 의존성
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>3.2.0</version>
</dependency>
```

### 주요 설정
```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: 
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
```

### Redis Pub/Sub 구현
```java
@Component
public class RedisMessagePublisher {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void publish(String channel, Object message) {
        redisTemplate.convertAndSend(channel, message);
    }
}

@Component
public class RedisMessageSubscriber {
    @EventListener
    public void handleMessage(String message) {
        // 메시지 처리 로직
    }
}
```

### 성능 특징
- **지연시간**: 마이크로초 단위
- **처리량**: 초당 수십만 operations
- **메모리 효율성**: 압축 및 최적화된 데이터 구조
- **클러스터링**: Redis Cluster를 통한 수평 확장

### 적용 사례
- 세션 스토어
- 캐시 레이어
- 실시간 알림
- 리더보드
- 분산 락

## 기타 메시지 브로커

### RabbitMQ
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
    <version>3.2.0</version>
</dependency>
```

**특징:**
- AMQP 프로토콜 지원
- 복잡한 라우팅 규칙
- 메시지 확인 및 재전송

### Amazon SQS
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-aws-messaging</artifactId>
    <version>2.2.6.RELEASE</version>
</dependency>
```

**특징:**
- 완전 관리형 서비스
- 표준 및 FIFO 큐 지원
- Auto Scaling

### Apache Pulsar
```xml
<dependency>
    <groupId>org.apache.pulsar</groupId>
    <artifactId>pulsar-client</artifactId>
    <version>3.0.0</version>
</dependency>
```

**특징:**
- 멀티 테넌시 지원
- 계층적 스토리지
- 스키마 레지스트리

## 아키텍처 패턴

### 1. Event Sourcing
```java
@Entity
public class EventStore {
    private String aggregateId;
    private String eventType;
    private String eventData;
    private LocalDateTime timestamp;
    // ...
}
```

### 2. CQRS (Command Query Responsibility Segregation)
```java
// Command Side
@Service
public class UserCommandService {
    public void createUser(CreateUserCommand command) {
        // 명령 처리
    }
}

// Query Side
@Service
public class UserQueryService {
    public UserView getUserById(String id) {
        // 조회 처리
    }
}
```

### 3. Saga Pattern
```java
@Service
public class OrderSagaOrchestrator {
    public void processOrder(OrderEvent event) {
        // 분산 트랜잭션 조정
    }
}
```

### 4. Outbox Pattern
```java
@Entity
public class OutboxEvent {
    private String eventId;
    private String eventType;
    private String payload;
    private boolean processed;
    // ...
}
```

## 구현 예제

### Kafka Producer 예제
```java
@Service
public class UserEventPublisher {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishUserCreated(User user) {
        UserCreatedEvent event = new UserCreatedEvent(user.getId(), user.getName());
        kafkaTemplate.send("user-events", event);
    }
}
```

### Kafka Consumer 예제
```java
@Component
public class UserEventConsumer {
    
    @KafkaListener(topics = "user-events", groupId = "notification-service")
    public void handleUserCreated(UserCreatedEvent event) {
        // 사용자 생성 이벤트 처리
        sendWelcomeEmail(event.getUserId());
    }
}
```

### Redis Pub/Sub 예제
```java
@Service
public class NotificationService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void sendNotification(String userId, String message) {
        NotificationEvent event = new NotificationEvent(userId, message);
        redisTemplate.convertAndSend("notifications", event);
    }
}
```

### Spring Cloud Stream 예제
```java
@Component
public class OrderProcessor {
    
    @Bean
    public Function<OrderEvent, PaymentEvent> processOrder() {
        return orderEvent -> {
            // 주문 처리 로직
            return new PaymentEvent(orderEvent.getOrderId(), orderEvent.getAmount());
        };
    }
}
```

## 모니터링 및 운영

### 메트릭 수집
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

### 분산 트레이싱
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
    <version>3.1.0</version>
</dependency>
```

### 로그 집계
```yaml
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level [%X{traceId},%X{spanId}] %logger{36} - %msg%n"
```

## 베스트 프랙티스

### 1. 이벤트 설계
- 이벤트는 불변(Immutable)으로 설계
- 스키마 버전 관리
- 의미 있는 이벤트 명명

### 2. 에러 처리
- Dead Letter Queue 구현
- 재시도 정책 설정
- Circuit Breaker 패턴 적용

### 3. 성능 최적화
- 배치 처리 활용
- 적절한 파티션 전략
- 커넥션 풀 최적화

### 4. 보안
- SSL/TLS 암호화
- 인증 및 권한 부여
- 메시지 암호화

---

**참고 문서:**
- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Redis Documentation](https://redis.io/documentation)
- [Spring Cloud Stream Reference](https://spring.io/projects/spring-cloud-stream) 