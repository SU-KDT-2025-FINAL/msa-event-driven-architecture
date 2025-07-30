# 분산 추적 시스템 구축 가이드

## 1. Spring Cloud Sleuth + Zipkin 구성

### 의존성 설정
```xml
<dependencies>
    <!-- Spring Cloud Sleuth -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
    
    <!-- Zipkin -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-sleuth-zipkin</artifactId>
    </dependency>
    
    <!-- Brave (Zipkin의 Java 클라이언트) -->
    <dependency>
        <groupId>io.zipkin.brave</groupId>
        <artifactId>brave-instrumentation-kafka-clients</artifactId>
    </dependency>
</dependencies>
```

### application.yml 설정
```yaml
spring:
  application:
    name: order-service
  
  sleuth:
    sampler:
      probability: 1.0  # 개발환경에서는 100% 샘플링
    zipkin:
      base-url: http://localhost:9411
      sender:
        type: web
    
    # Kafka 추적 활성화
    kafka:
      enabled: true
    
    # HTTP 추적 설정
    web:
      client:
        enabled: true
      servlet:
        enabled: true
    
    # 데이터베이스 쿼리 추적
    jdbc:
      enabled: true
      includes:
        - connection
        - query
        - fetch
    
    # 비동기 처리 추적
    async:
      enabled: true

# Zipkin 서버 설정 (별도 서비스)
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,httptrace
  tracing:
    sampling:
      probability: 1.0
```

### Zipkin 서버 Docker 설정
```yaml
# docker-compose.yml
version: '3.8'
services:
  zipkin:
    image: openzipkin/zipkin:latest
    container_name: zipkin
    ports:
      - "9411:9411"
    environment:
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=elasticsearch:9200
    depends_on:
      - elasticsearch
    
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
```

## 2. 커스텀 스팬 생성 및 관리

### 수동 스팬 생성
```java
@Service
@Slf4j
public class OrderProcessingService {
    
    private final Tracer tracer;
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    
    public OrderProcessingService(Tracer tracer, 
                                 OrderRepository orderRepository,
                                 PaymentService paymentService) {
        this.tracer = tracer;
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
    
    public void processOrder(CreateOrderRequest request) {
        Span orderSpan = tracer.nextSpan()
            .name("order-processing")
            .tag("order.customer-id", request.getCustomerId())
            .tag("order.item-count", String.valueOf(request.getItems().size()))
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(orderSpan)) {
            // 주문 생성 스팬
            Span createOrderSpan = tracer.nextSpan()
                .name("create-order")
                .start();
            
            try (Tracer.SpanInScope ws2 = tracer.withSpanInScope(createOrderSpan)) {
                Order order = createOrder(request);
                createOrderSpan.tag("order.id", order.getId());
                createOrderSpan.tag("order.total-amount", order.getTotalAmount().toString());
            } finally {
                createOrderSpan.end();
            }
            
            // 결제 처리 스팬
            Span paymentSpan = tracer.nextSpan()
                .name("process-payment")
                .start();
            
            try (Tracer.SpanInScope ws3 = tracer.withSpanInScope(paymentSpan)) {
                PaymentResult result = paymentService.processPayment(order);
                paymentSpan.tag("payment.method", result.getPaymentMethod());
                paymentSpan.tag("payment.status", result.getStatus().toString());
            } catch (PaymentException e) {
                paymentSpan.tag("error", e.getMessage());
                throw e;
            } finally {
                paymentSpan.end();
            }
            
        } catch (Exception e) {
            orderSpan.tag("error", e.getMessage());
            throw e;
        } finally {
            orderSpan.end();
        }
    }
}
```

### 어노테이션 기반 추적
```java
@NewSpan("validate-order")
public void validateOrder(@SpanTag("orderId") String orderId, 
                         @SpanTag("customerId") String customerId) {
    // 주문 유효성 검사 로직
    if (!isValidOrder(orderId)) {
        tracer.currentSpan().tag("validation.result", "failed");
        throw new InvalidOrderException("Invalid order: " + orderId);
    }
    tracer.currentSpan().tag("validation.result", "success");
}

@ContinueSpan
public PaymentResult processPayment(@SpanTag("paymentMethod") String paymentMethod,
                                   @SpanTag("amount") BigDecimal amount) {
    // 결제 처리 로직
    return paymentGateway.charge(paymentMethod, amount);
}
```

## 3. Kafka 메시지 추적

### Kafka 프로듀서 추적
```java
@Component
public class TracedKafkaProducer {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Tracing tracing;
    
    public TracedKafkaProducer(KafkaTemplate<String, Object> kafkaTemplate,
                              Tracing tracing) {
        this.kafkaTemplate = kafkaTemplate;
        this.tracing = tracing;
        
        // Kafka 프로듀서에 추적 설정
        this.kafkaTemplate.setProducerFactory(
            kafkaTracing().producerFactory(kafkaTemplate.getProducerFactory())
        );
    }
    
    private KafkaTracing kafkaTracing() {
        return KafkaTracing.newBuilder(tracing)
            .remoteServiceName("kafka")
            .build();
    }
    
    public void sendOrderEvent(OrderEvent event) {
        Span span = tracing.tracer().nextSpan()
            .name("kafka-send")
            .tag("kafka.topic", "order-events")
            .tag("event.type", event.getEventType())
            .tag("order.id", event.getOrderId())
            .start();
        
        try (Tracer.SpanInScope ws = tracing.tracer().withSpanInScope(span)) {
            // 트레이스 컨텍스트를 헤더에 추가
            ProducerRecord<String, Object> record = new ProducerRecord<>(
                "order-events", 
                event.getOrderId(), 
                event
            );
            
            // Sleuth가 자동으로 추적 헤더를 추가
            kafkaTemplate.send(record);
            
        } finally {
            span.end();
        }
    }
}
```

### Kafka 컨슈머 추적
```java
@KafkaListener(topics = "order-events", groupId = "order-processing-group")
public void handleOrderEvent(ConsumerRecord<String, OrderEvent> record,
                           @Header Map<String, Object> headers) {
    
    // Sleuth가 자동으로 스팬을 생성하고 추적 컨텍스트를 복원
    Span currentSpan = tracer.currentSpan();
    if (currentSpan != null) {
        currentSpan.tag("kafka.topic", record.topic())
                  .tag("kafka.partition", String.valueOf(record.partition()))
                  .tag("kafka.offset", String.valueOf(record.offset()))
                  .tag("event.type", record.value().getEventType());
    }
    
    try {
        // 이벤트 처리 로직
        processOrderEvent(record.value());
        
        if (currentSpan != null) {
            currentSpan.tag("processing.status", "success");
        }
        
    } catch (Exception e) {
        if (currentSpan != null) {
            currentSpan.tag("error", e.getMessage())
                      .tag("processing.status", "failed");
        }
        throw e;
    }
}
```

## 4. OpenTelemetry 도입

### OpenTelemetry 의존성
```xml
<dependencies>
    <!-- OpenTelemetry -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-api</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-zipkin</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry.instrumentation</groupId>
        <artifactId>opentelemetry-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

### OpenTelemetry 설정
```java
@Configuration
public class OpenTelemetryConfig {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        return OpenTelemetrySDK.builder()
            .setTracerProvider(
                SdkTracerProvider.builder()
                    .addSpanProcessor(BatchSpanProcessor.builder(
                        ZipkinSpanExporter.builder()
                            .setEndpoint("http://localhost:9411/api/v2/spans")
                            .build())
                        .build())
                    .setResource(Resource.getDefault()
                        .merge(Resource.builder()
                            .put(ResourceAttributes.SERVICE_NAME, "order-service")
                            .put(ResourceAttributes.SERVICE_VERSION, "1.0.0")
                            .build()))
                    .build())
            .buildAndRegisterGlobal();
    }
    
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("order-service", "1.0.0");
    }
}
```

### OpenTelemetry 수동 계측
```java
@Service
public class OpenTelemetryOrderService {
    
    private final Tracer tracer;
    
    public OpenTelemetryOrderService(Tracer tracer) {
        this.tracer = tracer;
    }
    
    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("create-order")
            .setSpanKind(SpanKind.INTERNAL)
            .setAttribute("order.customer-id", request.getCustomerId())
            .setAttribute("order.item-count", request.getItems().size())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 주문 생성 로직
            Order order = new Order(request);
            
            span.setAttribute("order.id", order.getId())
                .setAttribute("order.total-amount", order.getTotalAmount().doubleValue())
                .setStatus(StatusCode.OK);
            
            // 데이터베이스 저장
            saveOrderWithTracing(order);
            
            return order;
            
        } catch (Exception e) {
            span.recordException(e)
                .setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
    
    private void saveOrderWithTracing(Order order) {
        Span dbSpan = tracer.spanBuilder("db-save-order")
            .setSpanKind(SpanKind.CLIENT)
            .setAttribute("db.system", "postgresql")
            .setAttribute("db.name", "ecommerce")
            .setAttribute("db.operation", "INSERT")
            .setAttribute("db.table", "orders")
            .startSpan();
        
        try (Scope scope = dbSpan.makeCurrent()) {
            orderRepository.save(order);
            dbSpan.setStatus(StatusCode.OK);
        } catch (Exception e) {
            dbSpan.recordException(e)
                 .setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            dbSpan.end();
        }
    }
}
```

## 5. 서비스 간 요청 추적

### RestTemplate 추적
```java
@Configuration
public class TracingConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        
        // Sleuth가 자동으로 추적 헤더를 추가하는 인터셉터 설정
        List<ClientHttpRequestInterceptor> interceptors = restTemplate.getInterceptors();
        interceptors.add(new TracingClientHttpRequestInterceptor());
        restTemplate.setInterceptors(interceptors);
        
        return restTemplate;
    }
}

@Service
public class PaymentServiceClient {
    
    private final RestTemplate restTemplate;
    private final Tracer tracer;
    
    @NewSpan("payment-service-call")
    public PaymentResult processPayment(@SpanTag("orderId") String orderId,
                                       @SpanTag("amount") BigDecimal amount) {
        
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            currentSpan.tag("http.url", "http://payment-service/payments")
                      .tag("http.method", "POST");
        }
        
        try {
            PaymentRequest request = new PaymentRequest(orderId, amount);
            ResponseEntity<PaymentResult> response = restTemplate.postForEntity(
                "http://payment-service/payments", 
                request, 
                PaymentResult.class
            );
            
            if (currentSpan != null) {
                currentSpan.tag("http.status_code", String.valueOf(response.getStatusCodeValue()));
            }
            
            return response.getBody();
            
        } catch (Exception e) {
            if (currentSpan != null) {
                currentSpan.tag("error", e.getMessage());
            }
            throw e;
        }
    }
}
```

### WebClient 추적
```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .filter(ExchangeFilterFunction.ofRequestProcessor(request -> {
                // Sleuth가 자동으로 추적 헤더를 추가
                return Mono.just(request);
            }))
            .build();
    }
}

@Service
public class InventoryServiceClient {
    
    private final WebClient webClient;
    
    public Mono<InventoryStatus> checkInventory(String productId, int quantity) {
        return webClient.get()
            .uri("http://inventory-service/inventory/{productId}", productId)
            .retrieve()
            .bodyToMono(InventoryStatus.class)
            .doOnNext(status -> {
                Span currentSpan = tracer.currentSpan();
                if (currentSpan != null) {
                    currentSpan.tag("inventory.available", String.valueOf(status.getAvailable()))
                             .tag("inventory.reserved", String.valueOf(status.getReserved()));
                }
            });
    }
}
```

## 6. 성능 병목 지점 식별

### 커스텀 메트릭과 추적 통합
```java
@Component
public class PerformanceTracker {
    
    private final MeterRegistry meterRegistry;
    private final Tracer tracer;
    
    public PerformanceTracker(MeterRegistry meterRegistry, Tracer tracer) {
        this.meterRegistry = meterRegistry;
        this.tracer = tracer;
    }
    
    public <T> T trackPerformance(String operationName, Supplier<T> operation) {
        Timer.Sample sample = Timer.start(meterRegistry);
        Span span = tracer.nextSpan()
            .name(operationName)
            .tag("performance.tracking", "true")
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            T result = operation.get();
            
            sample.stop(Timer.builder("operation.duration")
                .tag("operation", operationName)
                .tag("status", "success")
                .register(meterRegistry));
                
            span.tag("status", "success");
            return result;
            
        } catch (Exception e) {
            sample.stop(Timer.builder("operation.duration")
                .tag("operation", operationName)
                .tag("status", "error")
                .register(meterRegistry));
                
            span.tag("status", "error")
                .tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### 느린 트레이스 감지
```java
@Component
public class SlowTraceDetector {
    
    private final Tracer tracer;
    private final ApplicationEventPublisher eventPublisher;
    
    @EventListener
    public void handleSpanClosed(SpanClosedEvent event) {
        long duration = event.getDurationMillis();
        String operationName = event.getSpan().getName();
        
        // 임계값 초과 시 알림
        if (duration > getThreshold(operationName)) {
            Span currentSpan = tracer.currentSpan();
            if (currentSpan != null) {
                currentSpan.tag("performance.slow", "true")
                          .tag("performance.threshold_exceeded", String.valueOf(duration));
            }
            
            eventPublisher.publishEvent(new SlowOperationEvent(
                operationName, 
                duration, 
                event.getSpan().getTraceId()
            ));
        }
    }
    
    private long getThreshold(String operationName) {
        // 작업별 임계값 설정
        return switch (operationName) {
            case "database-query" -> 1000;
            case "external-api-call" -> 3000;
            case "kafka-processing" -> 500;
            default -> 2000;
        };
    }
}
```

## 7. 모니터링 및 알럿

### Jaeger 통합
```yaml
# Jaeger 설정
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

### 알럿 규칙 설정
```yaml
# prometheus-alerts.yml
groups:
- name: tracing-alerts
  rules:
  - alert: HighTraceLatency
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High trace latency detected"
      description: "95th percentile latency is {{ $value }}s"
  
  - alert: TraceErrorRate
    expr: rate(trace_errors_total[5m]) > 0.1
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High trace error rate"
      description: "Error rate is {{ $value }}"
```

## 실습 과제

1. **전체 트레이스 구성**: HTTP 요청부터 Kafka 메시지, 데이터베이스 쿼리까지 전체 플로우 추적
2. **성능 분석**: 느린 트레이스 식별 및 병목 지점 분석 시스템 구축
3. **오류 추적**: 분산 환경에서 오류 발생 시 전체 요청 플로우 추적
4. **커스텀 메트릭**: 비즈니스 로직에 특화된 추적 정보 수집
5. **대시보드 구성**: Zipkin/Jaeger 기반 성능 모니터링 대시보드 구축

## 참고 자료

- [Spring Cloud Sleuth Documentation](https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/html/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Zipkin Documentation](https://zipkin.io/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)