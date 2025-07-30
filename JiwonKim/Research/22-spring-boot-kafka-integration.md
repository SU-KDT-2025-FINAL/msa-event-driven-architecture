# Spring Boot Kafka 통합 가이드

## 1. Spring Kafka 설정 및 최적화

### 의존성 설정
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### application.yml 설정
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: 1
      retries: 3
      batch-size: 32768
      linger-ms: 10
      buffer-memory: 67108864
      properties:
        compression.type: snappy
        enable.idempotence: true
    
    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      max-poll-records: 100
      properties:
        spring.json.trusted.packages: "com.example.events"
        session.timeout.ms: 30000
        heartbeat.interval.ms: 10000
```

### KafkaTemplate 구성
```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // 트랜잭션 지원
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-tx");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(props);
    }
    
    @Bean
    public KafkaTransactionManager kafkaTransactionManager() {
        return new KafkaTransactionManager(producerFactory());
    }
}
```

## 2. @KafkaListener 어노테이션 활용

### 기본 리스너 구현
```java
@Component
@Slf4j
public class OrderEventListener {
    
    @Autowired
    private OrderService orderService;
    
    @KafkaListener(
        topics = "order-events",
        groupId = "order-processing-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderCreated(
        @Payload OrderCreatedEvent event,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
        @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        ConsumerRecord<String, OrderCreatedEvent> record
    ) {
        log.info("Received order event: {} from topic: {}, partition: {}, offset: {}", 
                event, topic, partition, offset);
        
        try {
            orderService.processOrderCreated(event);
        } catch (Exception e) {
            log.error("Failed to process order created event: {}", event, e);
            throw e; // 재처리를 위해 예외를 다시 던짐
        }
    }
}
```

### 배치 처리 리스너
```java
@KafkaListener(
    topics = "inventory-events",
    groupId = "inventory-batch-group",
    containerFactory = "batchListenerContainerFactory"
)
public void handleInventoryEventsBatch(
    List<ConsumerRecord<String, InventoryEvent>> records,
    Acknowledgment acknowledgment
) {
    log.info("Processing batch of {} inventory events", records.size());
    
    List<InventoryEvent> events = records.stream()
        .map(ConsumerRecord::value)
        .collect(Collectors.toList());
    
    try {
        inventoryService.processBatch(events);
        acknowledgment.acknowledge();
    } catch (Exception e) {
        log.error("Failed to process inventory events batch", e);
        // 배치 처리 실패 시 개별 재처리 로직
        handleFailedBatch(records, acknowledgment);
    }
}
```

### 조건부 리스너
```java
@KafkaListener(
    topics = "payment-events",
    groupId = "payment-notification-group",
    condition = "headers['eventType'] == 'PAYMENT_COMPLETED'"
)
public void handlePaymentCompleted(PaymentEvent event) {
    notificationService.sendPaymentCompletedNotification(event);
}

@KafkaListener(
    topics = "payment-events",
    groupId = "payment-refund-group",
    condition = "headers['eventType'] == 'PAYMENT_FAILED'"
)
public void handlePaymentFailed(PaymentEvent event) {
    refundService.processRefund(event);
}
```

## 3. 고급 컨슈머 설정

### 커스텀 리스너 컨테이너 팩토리
```java
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> 
           kafkaListenerContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3); // 병렬 컨슈머 스레드 수
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        factory.getContainerProperties().setPollTimeout(3000);
        
        // 에러 핸들링
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new FixedBackOff(1000L, 3)
        ));
        
        // 재시도 토픽 설정
        factory.setRetryTopicConfigurer(retryTopicConfigurer());
        
        return factory;
    }
    
    @Bean
    public RetryTopicConfigurer retryTopicConfigurer() {
        return RetryTopicConfigurer.builder()
            .retryTopicSuffix("-retry")
            .dltSuffix("-dlt")
            .maxAttempts(3)
            .fixedBackoff(Duration.ofSeconds(5))
            .build();
    }
}
```

### 데드 레터 큐 처리
```java
@KafkaListener(topics = "order-events-dlt", groupId = "order-dlt-group")
public void handleOrderEventsDlt(
    ConsumerRecord<String, OrderEvent> record,
    @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage
) {
    log.error("Processing DLT message for order event: {}, exception: {}", 
             record.value(), exceptionMessage);
    
    // DLT 메시지 처리 로직
    dltService.handleFailedOrderEvent(record.value(), exceptionMessage);
}
```

## 4. KafkaTemplate 이벤트 발행

### 기본 이벤트 발행
```java
@Service
@Slf4j
public class OrderEventPublisher {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public OrderEventPublisher(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void publishOrderCreated(OrderCreatedEvent event) {
        String topic = "order-events";
        String key = event.getOrderId();
        
        ListenableFuture<SendResult<String, Object>> future = 
            kafkaTemplate.send(topic, key, event);
        
        future.addCallback(
            result -> log.info("Successfully sent order created event: {}", event),
            failure -> log.error("Failed to send order created event: {}", event, failure)
        );
    }
}
```

### 트랜잭션 이벤트 발행
```java
@Service
@Transactional
public class TransactionalOrderService {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @KafkaTransactional
    public void createOrder(CreateOrderRequest request) {
        // 데이터베이스 트랜잭션과 Kafka 트랜잭션 통합
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Kafka 트랜잭션 내에서 이벤트 발행
        OrderCreatedEvent event = new OrderCreatedEvent(order);
        kafkaTemplate.send("order-events", order.getId(), event);
        
        // 재고 예약 이벤트 발행
        InventoryReservationEvent reservationEvent = 
            new InventoryReservationEvent(order.getItems());
        kafkaTemplate.send("inventory-events", order.getId(), reservationEvent);
    }
}
```

### 헤더와 메타데이터 추가
```java
public void publishEventWithHeaders(OrderEvent event) {
    ProducerRecord<String, Object> record = new ProducerRecord<>(
        "order-events",
        event.getOrderId(),
        event
    );
    
    // 커스텀 헤더 추가
    record.headers().add("eventType", event.getEventType().getBytes());
    record.headers().add("version", "1.0".getBytes());
    record.headers().add("source", "order-service".getBytes());
    record.headers().add("correlationId", UUID.randomUUID().toString().getBytes());
    
    kafkaTemplate.send(record);
}
```

## 5. 스키마 레지스트리 통합

### Avro 스키마 설정
```yaml
spring:
  kafka:
    producer:
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      properties:
        schema.registry.url: http://localhost:8081
        auto.register.schemas: false
        use.latest.version: true
    
    consumer:
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
      properties:
        schema.registry.url: http://localhost:8081
        specific.avro.reader: true
```

### Avro 이벤트 클래스
```java
// OrderEvent.avsc 스키마 파일에서 생성된 클래스 사용
@Service
public class AvroOrderEventPublisher {
    
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderEvent(String orderId, String customerId, 
                                 List<OrderItem> items) {
        OrderEvent event = OrderEvent.newBuilder()
            .setOrderId(orderId)
            .setCustomerId(customerId)
            .setItems(items)
            .setTimestamp(Instant.now())
            .build();
        
        kafkaTemplate.send("order-events-avro", orderId, event);
    }
}
```

## 6. 테스트 전략

### 통합 테스트
```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}",
    "spring.kafka.consumer.auto-offset-reset=earliest"
})
@EmbeddedKafka(partitions = 1, topics = {"order-events", "inventory-events"})
class OrderEventIntegrationTest {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private OrderEventListener orderEventListener;
    
    @Test
    void shouldProcessOrderCreatedEvent() throws InterruptedException {
        // given
        OrderCreatedEvent event = new OrderCreatedEvent("order-123", "customer-456");
        
        // when
        kafkaTemplate.send("order-events", event.getOrderId(), event);
        
        // then
        verify(orderService, timeout(5000)).processOrderCreated(event);
    }
}
```

### 컨슈머 단위 테스트
```java
@ExtendWith(MockitoExtension.class)
class OrderEventListenerTest {
    
    @Mock
    private OrderService orderService;
    
    @InjectMocks
    private OrderEventListener listener;
    
    @Test
    void shouldHandleOrderCreatedEvent() {
        // given
        OrderCreatedEvent event = new OrderCreatedEvent("order-123", "customer-456");
        ConsumerRecord<String, OrderCreatedEvent> record = 
            new ConsumerRecord<>("order-events", 0, 0L, "order-123", event);
        
        // when
        listener.handleOrderCreated(event, "order-events", 0, 0L, record);
        
        // then
        verify(orderService).processOrderCreated(event);
    }
}
```

## 7. 성능 모니터링

### 액추에이터 메트릭
```yaml
management:
  endpoints:
    web:
      exposure:
        include: metrics, health, info
  metrics:
    export:
      prometheus:
        enabled: true
```

### 커스텀 메트릭
```java
@Component
public class KafkaMetrics {
    
    private final Counter messagesSent;
    private final Counter messagesReceived;
    private final Timer messageProcessingTime;
    
    public KafkaMetrics(MeterRegistry meterRegistry) {
        this.messagesSent = Counter.builder("kafka.messages.sent")
            .description("Number of messages sent to Kafka")
            .tag("service", "order-service")
            .register(meterRegistry);
            
        this.messagesReceived = Counter.builder("kafka.messages.received")
            .description("Number of messages received from Kafka")
            .register(meterRegistry);
            
        this.messageProcessingTime = Timer.builder("kafka.message.processing.time")
            .description("Time taken to process Kafka messages")
            .register(meterRegistry);
    }
    
    public void incrementMessagesSent() {
        messagesSent.increment();
    }
    
    public void recordProcessingTime(Duration duration) {
        messageProcessingTime.record(duration);
    }
}
```

## 실습 과제

1. **이벤트 주도 주문 시스템**: 주문 생성부터 배송까지의 전체 플로우를 이벤트로 구현
2. **배치 처리 최적화**: 대량의 재고 업데이트 이벤트를 배치로 처리하는 컨슈머 구현
3. **트랜잭션 통합**: 데이터베이스 트랜잭션과 Kafka 트랜잭션을 통합한 서비스 구현
4. **스키마 진화**: Avro 스키마의 버전 업그레이드 시나리오 테스트
5. **성능 테스트**: JMeter를 사용한 Kafka 기반 시스템의 부하 테스트

## 참고 자료

- [Spring for Apache Kafka Reference](https://docs.spring.io/spring-kafka/docs/current/reference/html/)
- [Spring Boot Kafka Auto Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/messaging.html#messaging.kafka)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)