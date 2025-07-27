# 스키마 관리 및 진화 가이드

## 1. Confluent Schema Registry 구축

### Docker Compose 설정
```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"
  
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
  
  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    depends_on:
      - kafka
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
      SCHEMA_REGISTRY_KAFKASTORE_TOPIC: _schemas
    ports:
      - "8081:8081"
```

### Schema Registry 클라이언트 설정
```java
@Configuration
public class SchemaRegistryConfig {
    
    @Value("${schema.registry.url}")
    private String schemaRegistryUrl;
    
    @Bean
    public CachedSchemaRegistryClient schemaRegistryClient() {
        return new CachedSchemaRegistryClient(schemaRegistryUrl, 1000);
    }
    
    @Bean
    public AvroSerializer<Object> avroSerializer() {
        Map<String, Object> props = new HashMap<>();
        props.put("schema.registry.url", schemaRegistryUrl);
        props.put("auto.register.schemas", false);
        props.put("use.latest.version", true);
        return new AvroSerializer<>(props);
    }
}
```

## 2. Apache Avro 스키마 설계

### 기본 스키마 구조
```json
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.example.events",
  "doc": "주문 이벤트 스키마",
  "fields": [
    {
      "name": "orderId",
      "type": "string",
      "doc": "주문 고유 식별자"
    },
    {
      "name": "customerId", 
      "type": "string",
      "doc": "고객 고유 식별자"
    },
    {
      "name": "eventType",
      "type": {
        "type": "enum",
        "name": "OrderEventType",
        "symbols": ["CREATED", "UPDATED", "CANCELLED", "COMPLETED"]
      },
      "doc": "이벤트 타입"
    },
    {
      "name": "timestamp",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      },
      "doc": "이벤트 발생 시간"
    },
    {
      "name": "orderItems",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "OrderItem",
          "fields": [
            {"name": "productId", "type": "string"},
            {"name": "quantity", "type": "int"},
            {"name": "price", "type": {"type": "bytes", "logicalType": "decimal", "precision": 10, "scale": 2}}
          ]
        }
      },
      "doc": "주문 상품 목록"
    },
    {
      "name": "metadata",
      "type": {
        "type": "map",
        "values": "string"
      },
      "default": {},
      "doc": "추가 메타데이터"
    }
  ]
}
```

### 복합 타입 스키마
```json
{
  "type": "record",
  "name": "PaymentEvent",
  "namespace": "com.example.events",
  "fields": [
    {
      "name": "paymentId",
      "type": "string"
    },
    {
      "name": "amount",
      "type": {
        "type": "record",
        "name": "Money",
        "fields": [
          {"name": "value", "type": {"type": "bytes", "logicalType": "decimal", "precision": 19, "scale": 4}},
          {"name": "currency", "type": "string"}
        ]
      }
    },
    {
      "name": "paymentMethod",
      "type": {
        "type": "union",
        "name": "PaymentMethodUnion",
        "types": [
          {
            "type": "record",
            "name": "CreditCard",
            "fields": [
              {"name": "cardNumber", "type": "string"},
              {"name": "expiryDate", "type": "string"}
            ]
          },
          {
            "type": "record", 
            "name": "BankTransfer",
            "fields": [
              {"name": "accountNumber", "type": "string"},
              {"name": "bankCode", "type": "string"}
            ]
          }
        ]
      }
    }
  ]
}
```

## 3. 스키마 진화 전략

### 하위 호환성 (Backward Compatibility)
```json
// 기존 스키마 v1
{
  "type": "record",
  "name": "UserEvent",
  "namespace": "com.example.events",
  "fields": [
    {"name": "userId", "type": "string"},
    {"name": "email", "type": "string"},
    {"name": "name", "type": "string"}
  ]
}

// 하위 호환 스키마 v2 (필드 제거)
{
  "type": "record",
  "name": "UserEvent",
  "namespace": "com.example.events", 
  "fields": [
    {"name": "userId", "type": "string"},
    {"name": "email", "type": "string"}
    // name 필드 제거 - 하위 호환
  ]
}
```

### 상위 호환성 (Forward Compatibility)
```json
// 기존 스키마 v1
{
  "type": "record",
  "name": "ProductEvent",
  "fields": [
    {"name": "productId", "type": "string"},
    {"name": "name", "type": "string"},
    {"name": "price", "type": "double"}
  ]
}

// 상위 호환 스키마 v2 (기본값과 함께 필드 추가)
{
  "type": "record",
  "name": "ProductEvent", 
  "fields": [
    {"name": "productId", "type": "string"},
    {"name": "name", "type": "string"},
    {"name": "price", "type": "double"},
    {"name": "category", "type": "string", "default": "GENERAL"},
    {"name": "tags", "type": {"type": "array", "items": "string"}, "default": []}
  ]
}
```

### 완전 호환성 (Full Compatibility)
```json
// 안전한 스키마 진화 - 옵셔널 필드만 추가
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "status", "type": "string"},
    {
      "name": "deliveryAddress", 
      "type": ["null", {
        "type": "record",
        "name": "Address",
        "fields": [
          {"name": "street", "type": "string"},
          {"name": "city", "type": "string"},
          {"name": "zipCode", "type": "string"}
        ]
      }],
      "default": null
    }
  ]
}
```

## 4. 스키마 버전 관리

### Gradle 빌드 스크립트
```gradle
plugins {
    id 'com.github.davidmc24.gradle.plugin.avro' version '1.8.0'
    id 'io.confluent.schema-registry' version '0.4.0'
}

dependencies {
    implementation 'org.apache.avro:avro:1.11.1'
    implementation 'io.confluent:kafka-avro-serializer:7.4.0'
}

avro {
    outputCharacterEncoding = 'UTF-8'
    stringType = 'String'
    fieldVisibility = 'PRIVATE'
}

schemaRegistry {
    url = 'http://localhost:8081'
    
    register {
        subject('order-events-value', 'src/main/avro/OrderEvent.avsc')
        subject('payment-events-value', 'src/main/avro/PaymentEvent.avsc')
    }
    
    compatibility {
        subject('order-events-value', 'BACKWARD')
        subject('payment-events-value', 'FULL')
    }
}
```

### 스키마 레지스트리 REST API 활용
```bash
# 스키마 등록
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '@OrderEvent.avsc' \
  http://localhost:8081/subjects/order-events-value/versions

# 스키마 조회
curl -X GET http://localhost:8081/subjects/order-events-value/versions/latest

# 호환성 검사
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '@NewOrderEvent.avsc' \
  http://localhost:8081/compatibility/subjects/order-events-value/versions/latest
```

## 5. Java 코드 생성 및 사용

### Maven 플러그인 설정
```xml
<plugin>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro-maven-plugin</artifactId>
    <version>1.11.1</version>
    <executions>
        <execution>
            <phase>generate-sources</phase>
            <goals>
                <goal>schema</goal>
            </goals>
            <configuration>
                <sourceDirectory>${project.basedir}/src/main/avro/</sourceDirectory>
                <outputDirectory>${project.basedir}/src/main/java/</outputDirectory>
                <stringType>String</stringType>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 생성된 Avro 클래스 사용
```java
@Service
public class OrderEventService {
    
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(String orderId, String customerId, 
                                   List<OrderItem> items) {
        // Builder 패턴을 사용한 이벤트 생성
        OrderEvent event = OrderEvent.newBuilder()
            .setOrderId(orderId)
            .setCustomerId(customerId)
            .setEventType(OrderEventType.CREATED)
            .setTimestamp(System.currentTimeMillis())
            .setOrderItems(items)
            .setMetadata(Map.of(
                "source", "order-service",
                "version", "1.0"
            ))
            .build();
        
        kafkaTemplate.send("order-events", orderId, event);
    }
    
    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(OrderEvent event) {
        switch (event.getEventType()) {
            case CREATED:
                handleOrderCreated(event);
                break;
            case UPDATED:
                handleOrderUpdated(event);
                break;
            case CANCELLED:
                handleOrderCancelled(event);
                break;
            case COMPLETED:
                handleOrderCompleted(event);
                break;
        }
    }
}
```

## 6. Protocol Buffers 대안

### .proto 파일 정의
```protobuf
syntax = "proto3";

package com.example.events;

import "google/protobuf/timestamp.proto";

message OrderEvent {
  string order_id = 1;
  string customer_id = 2;
  OrderEventType event_type = 3;
  google.protobuf.Timestamp timestamp = 4;
  repeated OrderItem order_items = 5;
  map<string, string> metadata = 6;
}

enum OrderEventType {
  UNKNOWN = 0;
  CREATED = 1;
  UPDATED = 2;
  CANCELLED = 3;
  COMPLETED = 4;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double price = 3;
}
```

### Protobuf Spring Boot 설정
```java
@Configuration
public class ProtobufConfig {
    
    @Bean
    public ProducerFactory<String, Message> protobufProducerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ProtobufSerializer.class);
        
        return new DefaultKafkaProducerFactory<>(props);
    }
    
    @Bean
    public KafkaTemplate<String, Message> protobufKafkaTemplate() {
        return new KafkaTemplate<>(protobufProducerFactory());
    }
}
```

## 7. 스키마 테스트 전략

### 스키마 호환성 테스트
```java
@Test
public class SchemaCompatibilityTest {
    
    private SchemaRegistryClient schemaRegistry;
    
    @BeforeEach
    void setUp() {
        schemaRegistry = new MockSchemaRegistryClient();
    }
    
    @Test
    void testBackwardCompatibility() throws Exception {
        // 기존 스키마 등록
        String oldSchema = loadSchema("OrderEventV1.avsc");
        int schemaId = schemaRegistry.register("order-events-value", new AvroSchema(oldSchema));
        
        // 새로운 스키마 호환성 검사
        String newSchema = loadSchema("OrderEventV2.avsc");
        boolean isCompatible = schemaRegistry.testCompatibility(
            "order-events-value", 
            new AvroSchema(newSchema)
        );
        
        assertTrue(isCompatible, "새로운 스키마가 기존 스키마와 호환되지 않습니다");
    }
    
    @Test
    void testSchemaEvolution() throws Exception {
        // 스키마 진화 시나리오 테스트
        Schema oldSchema = loadAvroSchema("OrderEventV1.avsc");
        Schema newSchema = loadAvroSchema("OrderEventV2.avsc");
        
        // 기존 데이터를 새로운 스키마로 읽기 테스트
        GenericRecord oldRecord = createOldRecord(oldSchema);
        byte[] serialized = serialize(oldRecord, oldSchema);
        GenericRecord newRecord = deserialize(serialized, oldSchema, newSchema);
        
        assertNotNull(newRecord);
        assertEquals(oldRecord.get("orderId"), newRecord.get("orderId"));
    }
}
```

### 성능 테스트
```java
@Test
public class SerializationPerformanceTest {
    
    @Test
    void compareSerializationPerformance() {
        OrderEvent event = createSampleOrderEvent();
        int iterations = 100000;
        
        // Avro 직렬화 성능 측정
        long avroTime = measureTime(() -> {
            for (int i = 0; i < iterations; i++) {
                byte[] serialized = avroSerializer.serialize("topic", event);
                avroDeserializer.deserialize("topic", serialized);
            }
        });
        
        // JSON 직렬화 성능 측정
        long jsonTime = measureTime(() -> {
            for (int i = 0; i < iterations; i++) {
                String json = objectMapper.writeValueAsString(event);
                objectMapper.readValue(json, OrderEvent.class);
            }
        });
        
        System.out.printf("Avro: %dms, JSON: %dms%n", avroTime, jsonTime);
        assertTrue(avroTime < jsonTime, "Avro should be faster than JSON");
    }
}
```

## 8. 모니터링 및 거버넌스

### 스키마 메트릭 수집
```java
@Component
public class SchemaMetrics {
    
    private final Counter schemaRegistrations;
    private final Counter schemaValidationErrors;
    private final Gauge activeSchemas;
    
    public SchemaMetrics(MeterRegistry meterRegistry, 
                        SchemaRegistryClient schemaRegistry) {
        this.schemaRegistrations = Counter.builder("schema.registrations")
            .description("Number of schema registrations")
            .register(meterRegistry);
            
        this.schemaValidationErrors = Counter.builder("schema.validation.errors")
            .description("Number of schema validation errors")
            .register(meterRegistry);
            
        this.activeSchemas = Gauge.builder("schema.active.count")
            .description("Number of active schemas")
            .register(meterRegistry, this, this::getActiveSchemaCount);
    }
    
    private double getActiveSchemaCount(SchemaMetrics metrics) {
        try {
            return schemaRegistry.getAllSubjects().size();
        } catch (Exception e) {
            return 0;
        }
    }
}
```

### 스키마 거버넌스 정책
```yaml
# schema-governance.yml
governance:
  naming-conventions:
    subjects: "^[a-z][a-z0-9-]*-value$"
    namespaces: "^com\\.example\\.[a-z][a-z0-9]*$"
  
  compatibility-policies:
    default: BACKWARD
    critical-topics:
      - order-events: FULL
      - payment-events: FULL
  
  retention-policies:
    schema-versions: 10
    subject-retention-days: 365
```

## 실습 과제

1. **스키마 레지스트리 구축**: Docker Compose를 사용한 완전한 스키마 레지스트리 환경 구성
2. **스키마 진화 시나리오**: 필드 추가, 제거, 타입 변경 등 다양한 진화 시나리오 테스트
3. **성능 벤치마크**: Avro vs JSON vs Protobuf 직렬화 성능 비교
4. **CI/CD 통합**: 스키마 호환성 검사를 포함한 배포 파이프라인 구축
5. **모니터링 대시보드**: 스키마 사용량 및 호환성 메트릭 대시보드 구성

## 참고 자료

- [Confluent Schema Registry Documentation](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Apache Avro Specification](https://avro.apache.org/docs/current/spec.html)
- [Protocol Buffers Language Guide](https://developers.google.com/protocol-buffers/docs/proto3)
- [Schema Evolution and Compatibility](https://docs.confluent.io/platform/current/schema-registry/avro.html#schema-evolution-and-compatibility)