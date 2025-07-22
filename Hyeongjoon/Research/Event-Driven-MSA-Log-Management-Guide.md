# 이벤트 기반 MSA 시스템의 로그 통합 관리 가이드

## 목차
1. [로그 설계 원칙](#로그-설계-원칙)
2. [서비스별 로그 유형](#서비스별-로그-유형)
3. [로그 구조 및 포맷](#로그-구조-및-포맷)
4. [로깅 레벨 전략](#로깅-레벨-전략)
5. [로그 수집 및 집계 아키텍처](#로그-수집-및-집계-아키텍처)
6. [서비스별 로그 구현 예시](#서비스별-로그-구현-예시)
7. [모니터링 및 알림](#모니터링-및-알림)
8. [성능 최적화 방안](#성능-최적화-방안)

## 로그 설계 원칙

### 1. 구조화된 로그 (Structured Logging)
```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "service": "order-service",
  "traceId": "trace-12345",
  "spanId": "span-67890",
  "userId": "user-001",
  "eventType": "ORDER_CREATED",
  "message": "Order created successfully",
  "metadata": {
    "orderId": "order-12345",
    "amount": 150.00,
    "currency": "USD"
  }
}
```

### 2. 상관관계 추적 (Correlation Tracking)
- **Trace ID**: 전체 요청 흐름 추적
- **Span ID**: 개별 서비스 내 작업 단위
- **Correlation ID**: 비즈니스 프로세스 연관성

### 3. 컨텍스트 정보 포함
- 서비스 정보, 사용자 정보, 세션 정보
- 비즈니스 도메인 관련 메타데이터

## 서비스별 로그 유형

### 1. 애플리케이션 로그 (Application Logs)
```java
// Spring Boot 예시
@Component
public class OrderEventLogger {
    private static final Logger logger = LoggerFactory.getLogger(OrderEventLogger.class);
    
    public void logOrderEvent(String eventType, String orderId, Object payload) {
        MDC.put("eventType", eventType);
        MDC.put("orderId", orderId);
        logger.info("Order event processed: {}", payload);
        MDC.clear();
    }
}
```

### 2. Kafka 이벤트 로그 (Event Logs)
```java
// Producer 로그
public class KafkaEventLogger {
    
    // 이벤트 발행 전
    public void logEventPublishing(String topic, String key, Object event) {
        log.info("Publishing event to topic: {} with key: {} and payload: {}", 
                topic, key, event);
    }
    
    // 이벤트 발행 성공
    public void logEventPublished(String topic, String key, RecordMetadata metadata) {
        log.info("Event published successfully - Topic: {}, Partition: {}, Offset: {}",
                topic, metadata.partition(), metadata.offset());
    }
    
    // 이벤트 발행 실패
    public void logEventPublishFailure(String topic, String key, Exception ex) {
        log.error("Failed to publish event to topic: {} with key: {}", 
                topic, key, ex);
    }
}
```

### 3. Consumer 처리 로그
```java
@KafkaListener(topics = "order-events")
public void handleOrderEvent(OrderEvent event, 
                           @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                           @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
                           @Header(KafkaHeaders.OFFSET) long offset) {
    
    String traceId = UUID.randomUUID().toString();
    MDC.put("traceId", traceId);
    MDC.put("topic", topic);
    MDC.put("partition", String.valueOf(partition));
    MDC.put("offset", String.valueOf(offset));
    
    try {
        log.info("Processing order event: {}", event);
        // 비즈니스 로직 처리
        orderService.processOrder(event);
        log.info("Order event processed successfully");
    } catch (Exception ex) {
        log.error("Failed to process order event", ex);
        // 에러 처리 로직
    } finally {
        MDC.clear();
    }
}
```

### 4. 성능 및 메트릭 로그
```java
@Component
public class PerformanceLogger {
    
    @EventListener
    public void logKafkaMetrics(KafkaMetricsEvent event) {
        log.info("Kafka metrics - Consumer lag: {}, Throughput: {} msg/s, Error rate: {}%",
                event.getConsumerLag(), event.getThroughput(), event.getErrorRate());
    }
    
    @Scheduled(fixedRate = 30000) // 30초마다
    public void logServiceHealth() {
        HealthIndicator health = healthService.getHealth();
        log.info("Service health check - Status: {}, Response time: {}ms, Active connections: {}",
                health.getStatus(), health.getResponseTime(), health.getActiveConnections());
    }
}
```

## 로그 구조 및 포맷

### 표준 로그 포맷
```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO|WARN|ERROR|DEBUG",
  "service": "service-name",
  "instance": "instance-id",
  "version": "1.0.0",
  "environment": "prod|staging|dev",
  "traceId": "unique-trace-id",
  "spanId": "unique-span-id",
  "userId": "user-identifier",
  "sessionId": "session-identifier",
  "requestId": "request-identifier",
  "eventType": "BUSINESS_EVENT_TYPE",
  "category": "APPLICATION|KAFKA|PERFORMANCE|SECURITY",
  "message": "Human readable message",
  "metadata": {
    "key1": "value1",
    "key2": "value2"
  },
  "kafkaMetadata": {
    "topic": "topic-name",
    "partition": 0,
    "offset": 12345,
    "key": "message-key",
    "headers": {}
  },
  "error": {
    "type": "exception-type",
    "message": "error-message",
    "stackTrace": "stack-trace"
  }
}
```

### Logback 설정 예시 (Spring Boot)
```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp/>
                <logLevel/>
                <loggerName/>
                <mdc/>
                <message/>
                <stackTrace/>
            </providers>
        </encoder>
    </appender>
    
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp/>
                <logLevel/>
                <loggerName/>
                <mdc/>
                <message/>
                <stackTrace/>
            </providers>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

## 로깅 레벨 전략

### 레벨별 사용 가이드

#### ERROR 레벨
- Kafka 연결 실패, 메시지 처리 실패
- 비즈니스 로직 예외
- 시스템 장애 상황

```java
log.error("Failed to process order event for orderId: {} due to payment service unavailable", 
         orderId, exception);
```

#### WARN 레벨
- Consumer lag 증가
- 재시도 발생
- 성능 임계값 초과

```java
log.warn("High consumer lag detected: {} messages behind, topic: {}", lag, topic);
```

#### INFO 레벨
- 정상적인 이벤트 처리
- 중요한 비즈니스 이벤트
- 서비스 상태 변화

```java
log.info("Order {} created successfully for user {}", orderId, userId);
```

#### DEBUG 레벨
- 상세한 처리 과정
- 개발/디버깅 정보

```java
log.debug("Processing order event with payload: {}", eventPayload);
```

## 로그 수집 및 집계 아키텍처

### 1. ELK Stack 기반 아키텍처
```yaml
# docker-compose.yml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config:/usr/share/logstash/config
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash
```

### 2. Logstash 파이프라인 설정
```ruby
# logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][service] {
    mutate {
      add_field => { "service_name" => "%{[fields][service]}" }
    }
  }

  # JSON 로그 파싱
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
    }
  }

  # 타임스탬프 파싱
  if [timestamp] {
    date {
      match => [ "timestamp", "ISO8601" ]
    }
  }

  # Kafka 관련 필드 추출
  if [kafkaMetadata] {
    mutate {
      add_field => {
        "kafka_topic" => "%{[kafkaMetadata][topic]}"
        "kafka_partition" => "%{[kafkaMetadata][partition]}"
        "kafka_offset" => "%{[kafkaMetadata][offset]}"
      }
    }
  }

  # 에러 분류
  if [level] == "ERROR" {
    mutate {
      add_tag => [ "error" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "msa-logs-%{+YYYY.MM.dd}"
  }
}
```

### 3. Filebeat 설정
```yaml
# filebeat.yml
filebeat.inputs:
- type: container
  paths:
    - '/var/lib/docker/containers/*/*.log'
  processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"
  - decode_json_fields:
      fields: ["message"]
      target: ""

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
```

## 서비스별 로그 구현 예시

### 1. Order Service 로그 구현
```java
@Service
@Slf4j
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private EventLogger eventLogger;
    
    public Order createOrder(CreateOrderRequest request) {
        String traceId = UUID.randomUUID().toString();
        MDC.put("traceId", traceId);
        MDC.put("service", "order-service");
        MDC.put("operation", "createOrder");
        
        try {
            log.info("Creating order for user: {}", request.getUserId());
            
            // 주문 생성 로직
            Order order = buildOrder(request);
            orderRepository.save(order);
            
            // 이벤트 발행
            OrderCreatedEvent event = new OrderCreatedEvent(order);
            eventLogger.logEventPublishing("order-events", order.getId(), event);
            
            kafkaTemplate.send("order-events", order.getId(), event)
                .addCallback(
                    result -> eventLogger.logEventPublished("order-events", order.getId(), result.getRecordMetadata()),
                    failure -> eventLogger.logEventPublishFailure("order-events", order.getId(), failure)
                );
            
            log.info("Order created successfully: {}", order.getId());
            return order;
            
        } catch (Exception ex) {
            log.error("Failed to create order for user: {}", request.getUserId(), ex);
            throw ex;
        } finally {
            MDC.clear();
        }
    }
}
```

### 2. Payment Service 로그 구현
```java
@Component
@Slf4j
public class PaymentEventHandler {
    
    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void handleOrderCreated(OrderCreatedEvent event, 
                                 @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                                 @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
                                 @Header(KafkaHeaders.OFFSET) long offset) {
        
        String traceId = event.getTraceId() != null ? event.getTraceId() : UUID.randomUUID().toString();
        MDC.put("traceId", traceId);
        MDC.put("service", "payment-service");
        MDC.put("eventType", "ORDER_CREATED");
        MDC.put("orderId", event.getOrderId());
        MDC.put("kafkaTopic", topic);
        MDC.put("kafkaPartition", String.valueOf(partition));
        MDC.put("kafkaOffset", String.valueOf(offset));
        
        try {
            log.info("Received order created event for order: {}", event.getOrderId());
            
            // 결제 처리 로직
            PaymentResult result = paymentService.processPayment(event);
            
            if (result.isSuccess()) {
                log.info("Payment processed successfully for order: {}", event.getOrderId());
                publishPaymentCompletedEvent(event.getOrderId(), result);
            } else {
                log.warn("Payment failed for order: {}, reason: {}", event.getOrderId(), result.getFailureReason());
                publishPaymentFailedEvent(event.getOrderId(), result);
            }
            
        } catch (Exception ex) {
            log.error("Error processing order created event for order: {}", event.getOrderId(), ex);
            // Dead Letter Queue로 전송하거나 재시도 로직
        } finally {
            MDC.clear();
        }
    }
}
```

### 3. 공통 EventLogger 구현
```java
@Component
@Slf4j
public class EventLogger {
    
    public void logEventPublishing(String topic, String key, Object event) {
        log.info("Publishing event - Topic: {}, Key: {}, EventType: {}", 
                topic, key, event.getClass().getSimpleName());
    }
    
    public void logEventPublished(String topic, String key, RecordMetadata metadata) {
        log.info("Event published - Topic: {}, Partition: {}, Offset: {}, Key: {}", 
                topic, metadata.partition(), metadata.offset(), key);
    }
    
    public void logEventPublishFailure(String topic, String key, Throwable exception) {
        log.error("Event publish failed - Topic: {}, Key: {}", topic, key, exception);
    }
    
    public void logEventConsumed(String topic, int partition, long offset, String eventType) {
        log.info("Event consumed - Topic: {}, Partition: {}, Offset: {}, EventType: {}", 
                topic, partition, offset, eventType);
    }
    
    public void logEventProcessingTime(String eventType, long processingTimeMs) {
        log.info("Event processing completed - EventType: {}, ProcessingTime: {}ms", 
                eventType, processingTimeMs);
    }
}
```

## 모니터링 및 알림

### 1. Kibana 대시보드 설정
```json
{
  "dashboard": "MSA Event-Driven System Monitoring",
  "visualizations": [
    {
      "name": "Error Rate by Service",
      "type": "line_chart",
      "query": "level:ERROR",
      "aggregation": "count",
      "group_by": "service"
    },
    {
      "name": "Kafka Consumer Lag",
      "type": "gauge",
      "query": "category:PERFORMANCE AND kafkaMetadata.lag:*",
      "aggregation": "avg",
      "field": "kafkaMetadata.lag"
    },
    {
      "name": "Event Processing Time",
      "type": "histogram",
      "query": "eventType:* AND processingTime:*",
      "aggregation": "percentiles",
      "field": "processingTime"
    }
  ]
}
```

### 2. 알림 규칙 설정
```yaml
# ElastAlert 설정
rules:
  - name: "High Error Rate Alert"
    type: "frequency"
    index: "msa-logs-*"
    num_events: 10
    timeframe:
      minutes: 5
    filter:
      - term:
          level: "ERROR"
    alert:
      - "slack"
    slack:
      slack_webhook_url: "https://hooks.slack.com/..."
      slack_channel_override: "#alerts"

  - name: "Kafka Consumer Lag Alert"
    type: "any"
    index: "msa-logs-*"
    filter:
      - range:
          kafkaMetadata.lag:
            gt: 1000
    alert:
      - "email"
    email:
      - "ops-team@company.com"
```

### 3. 메트릭 수집 (Micrometer + Prometheus)
```java
@Component
public class KafkaMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final Counter eventPublishedCounter;
    private final Counter eventConsumedCounter;
    private final Timer eventProcessingTimer;
    
    public KafkaMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.eventPublishedCounter = Counter.builder("kafka.events.published")
                .description("Number of events published to Kafka")
                .register(meterRegistry);
        this.eventConsumedCounter = Counter.builder("kafka.events.consumed")
                .description("Number of events consumed from Kafka")
                .register(meterRegistry);
        this.eventProcessingTimer = Timer.builder("kafka.event.processing.time")
                .description("Time taken to process events")
                .register(meterRegistry);
    }
    
    public void recordEventPublished(String topic, String eventType) {
        eventPublishedCounter.increment(
                Tags.of(
                        "topic", topic,
                        "event_type", eventType
                )
        );
    }
    
    public void recordEventConsumed(String topic, String eventType) {
        eventConsumedCounter.increment(
                Tags.of(
                        "topic", topic,
                        "event_type", eventType
                )
        );
    }
    
    public Timer.Sample startProcessingTimer() {
        return Timer.start(meterRegistry);
    }
}
```

## 성능 최적화 방안

### 1. 비동기 로깅
```java
@Configuration
public class LoggingConfiguration {
    
    @Bean
    public AsyncAppender asyncAppender() {
        AsyncAppender asyncAppender = new AsyncAppender();
        asyncAppender.setQueueSize(1024);
        asyncAppender.setDiscardingThreshold(20);
        asyncAppender.setIncludeCallerData(false);
        return asyncAppender;
    }
}
```

### 2. 로그 샘플링
```java
@Component
public class SamplingLogger {
    
    private final Logger logger = LoggerFactory.getLogger(SamplingLogger.class);
    private final AtomicLong counter = new AtomicLong(0);
    private final int sampleRate = 100; // 1/100 샘플링
    
    public void logWithSampling(String message, Object... args) {
        if (counter.incrementAndGet() % sampleRate == 0) {
            logger.info(message, args);
        }
    }
}
```

### 3. 로그 압축 및 보관 정책
```xml
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
            <cleanHistoryOnStart>true</cleanHistoryOnStart>
        </rollingPolicy>
    </appender>
</configuration>
```

### 4. 로그 레벨 동적 조정
```java
@RestController
@RequestMapping("/admin/logging")
public class LoggingController {
    
    @PostMapping("/level")
    public ResponseEntity<Void> changeLogLevel(@RequestParam String logger, 
                                              @RequestParam String level) {
        LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
        ch.qos.logback.classic.Logger logbackLogger = context.getLogger(logger);
        logbackLogger.setLevel(Level.valueOf(level));
        return ResponseEntity.ok().build();
    }
}
```

## 모범 사례 요약

### 1. 로그 설계 시 고려사항
- **일관성**: 모든 서비스에서 동일한 로그 포맷 사용
- **상관관계**: Trace ID를 통한 요청 흐름 추적
- **컨텍스트**: 충분한 메타데이터 포함
- **구조화**: JSON 형태의 구조화된 로그 사용

### 2. 성능 고려사항
- **비동기 로깅**: 애플리케이션 성능 영향 최소화
- **샘플링**: 고빈도 로그의 샘플링 적용
- **압축**: 로그 파일 압축 및 보관 정책 수립
- **레벨 조정**: 운영 환경에서 적절한 로그 레벨 설정

### 3. 운영 관점 고려사항
- **모니터링**: 실시간 로그 모니터링 및 알림 설정
- **대시보드**: 비즈니스 메트릭과 기술 메트릭 시각화
- **알림**: 임계값 기반 자동 알림 시스템 구축
- **백업**: 로그 데이터 백업 및 복구 계획 수립

이러한 로그 통합 관리 방안을 통해 이벤트 기반 MSA 시스템의 가시성을 확보하고, 효과적인 모니터링 및 디버깅이 가능한 환경을 구축할 수 있습니다. 