# 구조화된 로깅 및 모니터링 가이드

## 1. 구조화된 로깅 설계

### JSON 통합 로그 포맷
```java
// Logback 설정 - logback-spring.xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="!local">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
                <providers>
                    <timestamp/>
                    <logLevel/>
                    <loggerName/>
                    <mdc/>
                    <message/>
                    <arguments/>
                    <stackTrace/>
                </providers>
            </encoder>
        </appender>
    </springProfile>
    
    <springProfile name="local">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId},%X{spanId}] %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </springProfile>
    
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp/>
                <logLevel/>
                <loggerName/>
                <mdc/>
                <message/>
                <arguments/>
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

### 구조화된 로깅 유틸리티
```java
@Component
@Slf4j
public class StructuredLogger {
    
    private final ObjectMapper objectMapper;
    
    public StructuredLogger(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }
    
    public void logEvent(String eventType, Object eventData) {
        try {
            Map<String, Object> logEntry = Map.of(
                "eventType", eventType,
                "eventData", eventData,
                "timestamp", Instant.now(),
                "service", "order-service",
                "version", "1.0"
            );
            
            log.info("Event: {}", objectMapper.writeValueAsString(logEntry));
        } catch (Exception e) {
            log.error("Failed to log structured event", e);
        }
    }
    
    public void logKafkaMessage(String topic, String key, Object message, String operation) {
        Map<String, Object> kafkaLog = Map.of(
            "component", "kafka",
            "operation", operation,
            "topic", topic,
            "messageKey", key,
            "messageType", message.getClass().getSimpleName(),
            "timestamp", Instant.now()
        );
        
        MDC.put("kafka.topic", topic);
        MDC.put("kafka.operation", operation);
        
        try {
            log.info("Kafka {}: {}", operation, objectMapper.writeValueAsString(kafkaLog));
        } catch (Exception e) {
            log.error("Failed to log Kafka message", e);
        } finally {
            MDC.clear();
        }
    }
}
```

## 2. MDC 상관관계 추적

### 트레이스 ID 관리
```java
@Component
public class TraceContextManager {
    
    private static final String TRACE_ID_KEY = "traceId";
    private static final String SPAN_ID_KEY = "spanId";
    private static final String USER_ID_KEY = "userId";
    
    public void setTraceContext(String traceId, String spanId, String userId) {
        MDC.put(TRACE_ID_KEY, traceId);
        MDC.put(SPAN_ID_KEY, spanId);
        if (userId != null) {
            MDC.put(USER_ID_KEY, userId);
        }
    }
    
    public String getTraceId() {
        return MDC.get(TRACE_ID_KEY);
    }
    
    public void clearContext() {
        MDC.clear();
    }
    
    public void propagateToNewThread(Runnable task) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        CompletableFuture.runAsync(() -> {
            try {
                if (contextMap != null) {
                    MDC.setContextMap(contextMap);
                }
                task.run();
            } finally {
                MDC.clear();
            }
        });
    }
}
```

### HTTP 요청 추적 필터
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class TraceFilter implements Filter {
    
    private final TraceContextManager traceContextManager;
    
    public TraceFilter(TraceContextManager traceContextManager) {
        this.traceContextManager = traceContextManager;
    }
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        String traceId = extractOrGenerateTraceId(httpRequest);
        String spanId = generateSpanId();
        String userId = extractUserId(httpRequest);
        
        try {
            traceContextManager.setTraceContext(traceId, spanId, userId);
            
            // 응답 헤더에 트레이스 ID 추가
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            httpResponse.setHeader("X-Trace-Id", traceId);
            
            chain.doFilter(request, response);
            
        } finally {
            traceContextManager.clearContext();
        }
    }
    
    private String extractOrGenerateTraceId(HttpServletRequest request) {
        String traceId = request.getHeader("X-Trace-Id");
        return traceId != null ? traceId : UUID.randomUUID().toString();
    }
    
    private String generateSpanId() {
        return UUID.randomUUID().toString().substring(0, 8);
    }
    
    private String extractUserId(HttpServletRequest request) {
        // JWT 토큰에서 사용자 ID 추출
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            // JWT 파싱 로직
            return parseUserIdFromJwt(authHeader.substring(7));
        }
        return null;
    }
}
```

### Kafka 메시지 추적
```java
@Component
public class KafkaTraceAspect {
    
    private final TraceContextManager traceContextManager;
    
    @Around("@annotation(org.springframework.kafka.annotation.KafkaListener)")
    public Object traceKafkaListener(ProceedingJoinPoint joinPoint) throws Throwable {
        ConsumerRecord<?, ?> record = findConsumerRecord(joinPoint.getArgs());
        
        if (record != null) {
            String traceId = extractTraceIdFromHeaders(record.headers());
            String spanId = UUID.randomUUID().toString().substring(0, 8);
            
            traceContextManager.setTraceContext(traceId, spanId, null);
        }
        
        try {
            return joinPoint.proceed();
        } finally {
            traceContextManager.clearContext();
        }
    }
    
    private String extractTraceIdFromHeaders(Headers headers) {
        Header traceHeader = headers.lastHeader("traceId");
        return traceHeader != null ? new String(traceHeader.value()) : 
               UUID.randomUUID().toString();
    }
}
```

## 3. ELK Stack 구현

### Docker Compose ELK 설정
```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    
  kibana:
    image: docker.elastic.co/kibana/kibana:8.8.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    
  logstash:
    image: docker.elastic.co/logstash/logstash:8.8.0
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

volumes:
  elasticsearch_data:
```

### Logstash 파이프라인 설정
```ruby
# logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }
  
  tcp {
    port => 5000
    codec => json_lines
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
  
  # 로그 레벨 정규화
  if [level] {
    mutate {
      uppercase => [ "level" ]
    }
  }
  
  # Kafka 관련 필드 추출
  if [kafka] {
    mutate {
      add_field => { 
        "kafka_topic" => "%{[kafka][topic]}"
        "kafka_operation" => "%{[kafka][operation]}"
      }
    }
  }
  
  # 에러 태그 추가
  if [level] == "ERROR" {
    mutate {
      add_tag => [ "error" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "microservices-logs-%{+YYYY.MM.dd}"
    template_name => "microservices"
    template => "/usr/share/logstash/templates/microservices-template.json"
    template_overwrite => true
  }
  
  stdout {
    codec => rubydebug
  }
}
```

### Elasticsearch 인덱스 템플릿
```json
{
  "index_patterns": ["microservices-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "level": {"type": "keyword"},
        "service_name": {"type": "keyword"},
        "traceId": {"type": "keyword"},
        "spanId": {"type": "keyword"},
        "userId": {"type": "keyword"},
        "message": {"type": "text"},
        "logger_name": {"type": "keyword"},
        "kafka_topic": {"type": "keyword"},
        "kafka_operation": {"type": "keyword"},
        "stack_trace": {"type": "text"},
        "eventType": {"type": "keyword"},
        "eventData": {
          "type": "object",
          "enabled": true
        }
      }
    }
  }
}
```

## 4. Filebeat 로그 수집

### Filebeat 설정
```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/microservices/*.log
  fields:
    service: microservice
  fields_under_root: true
  multiline.pattern: '^\{'
  multiline.negate: true
  multiline.match: after

- type: docker
  containers.ids:
    - "*"
  processors:
    - add_docker_metadata: ~

output.logstash:
  hosts: ["logstash:5044"]

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

### Docker Compose Filebeat 통합
```yaml
# docker-compose에 추가
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.8.0
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log:/var/log:ro
    depends_on:
      - logstash
    command: filebeat -e -strict.perms=false
```

## 5. Kibana 대시보드 생성

### 인덱스 패턴 생성
```bash
# Kibana API를 통한 인덱스 패턴 생성
curl -X POST "kibana:5601/api/saved_objects/index-pattern" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d '{
    "attributes": {
      "title": "microservices-logs-*",
      "timeFieldName": "@timestamp"
    }
  }'
```

### 대시보드 구성 요소
```json
{
  "version": "8.8.0",
  "objects": [
    {
      "id": "microservices-overview",
      "type": "dashboard",
      "attributes": {
        "title": "Microservices Overview Dashboard",
        "description": "마이크로서비스 전체 현황 대시보드",
        "panelsJSON": "[...]",
        "timeRestore": true,
        "timeTo": "now",
        "timeFrom": "now-24h"
      }
    }
  ]
}
```

### Saved Searches 및 Visualizations
```bash
# 에러 로그 검색 저장
curl -X POST "kibana:5601/api/saved_objects/search" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d '{
    "attributes": {
      "title": "Error Logs",
      "description": "모든 에러 레벨 로그",
      "columns": ["@timestamp", "service_name", "level", "message"],
      "sort": [["@timestamp", "desc"]],
      "kibanaSavedObjectMeta": {
        "searchSourceJSON": "{\"query\":{\"match\":{\"level\":\"ERROR\"}},\"filter\":[]}"
      }
    }
  }'
```

## 6. 성능 로깅 및 메트릭

### 메서드 실행 시간 측정
```java
@Aspect
@Component
@Slf4j
public class PerformanceLoggingAspect {
    
    private final StructuredLogger structuredLogger;
    private final MeterRegistry meterRegistry;
    
    @Around("@annotation(com.example.annotation.LogPerformance)")
    public Object logPerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        Instant startTime = Instant.now();
        
        try {
            Object result = joinPoint.proceed();
            
            Duration duration = Duration.between(startTime, Instant.now());
            sample.stop(Timer.builder("method.execution.time")
                .tag("class", className)
                .tag("method", methodName)
                .tag("status", "success")
                .register(meterRegistry));
            
            structuredLogger.logEvent("method_performance", Map.of(
                "className", className,
                "methodName", methodName,
                "duration", duration.toMillis(),
                "status", "success"
            ));
            
            return result;
            
        } catch (Exception e) {
            Duration duration = Duration.between(startTime, Instant.now());
            sample.stop(Timer.builder("method.execution.time")
                .tag("class", className)
                .tag("method", methodName)
                .tag("status", "error")
                .register(meterRegistry));
                
            structuredLogger.logEvent("method_performance", Map.of(
                "className", className,
                "methodName", methodName,
                "duration", duration.toMillis(),
                "status", "error",
                "error", e.getMessage()
            ));
            
            throw e;
        }
    }
}
```

### Kafka 처리 성능 로깅
```java
@Component
public class KafkaPerformanceLogger {
    
    private final StructuredLogger structuredLogger;
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void handleKafkaProducerEvent(KafkaProducerEvent event) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        long sendTime = event.getSendTime();
        long ackTime = event.getAckTime();
        long duration = ackTime - sendTime;
        
        sample.stop(Timer.builder("kafka.producer.latency")
            .tag("topic", event.getTopic())
            .register(meterRegistry));
        
        structuredLogger.logEvent("kafka_producer_performance", Map.of(
            "topic", event.getTopic(),
            "partition", event.getPartition(),
            "offset", event.getOffset(),
            "latency", duration,
            "messageSize", event.getMessageSize()
        ));
    }
    
    @EventListener
    public void handleKafkaConsumerEvent(KafkaConsumerEvent event) {
        long processingTime = event.getProcessingEndTime() - event.getProcessingStartTime();
        long lag = event.getCurrentTime() - event.getMessageTimestamp();
        
        Timer.builder("kafka.consumer.processing.time")
            .tag("topic", event.getTopic())
            .tag("group", event.getConsumerGroup())
            .register(meterRegistry)
            .record(processingTime, TimeUnit.MILLISECONDS);
        
        structuredLogger.logEvent("kafka_consumer_performance", Map.of(
            "topic", event.getTopic(),
            "partition", event.getPartition(),
            "offset", event.getOffset(),
            "processingTime", processingTime,
            "consumerLag", lag,
            "consumerGroup", event.getConsumerGroup()
        ));
    }
}
```

## 실습 과제

1. **통합 로깅 시스템 구축**: 마이크로서비스 환경에서 중앙화된 로깅 시스템 구성
2. **트레이스 추적 구현**: HTTP 요청부터 Kafka 메시지 처리까지 전체 트레이스 추적
3. **성능 모니터링**: 메서드 실행 시간, Kafka 처리 성능 등 핵심 메트릭 수집
4. **알럿 시스템**: Elasticsearch Watcher 또는 Kibana 알럿을 사용한 이상 상황 감지
5. **대시보드 커스터마이징**: 비즈니스 요구사항에 맞는 맞춤형 Kibana 대시보드 구성

## 참고 자료

- [Elastic Stack Documentation](https://www.elastic.co/guide/index.html)
- [Logback Documentation](http://logback.qos.ch/documentation.html)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)