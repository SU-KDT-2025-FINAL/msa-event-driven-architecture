# Apache Kafka 심화 학습 가이드

## 1. Kafka 클러스터 설정 및 운영

### 클러스터 구성 요소
- **Broker**: 메시지를 저장하고 서비스하는 서버
- **ZooKeeper**: 클러스터 메타데이터 관리 (Kafka 2.8+에서는 KRaft 모드 지원)
- **Controller**: 파티션 리더 선출 및 클러스터 상태 관리

### 핵심 설정 매개변수
```properties
# server.properties 주요 설정
broker.id=1
listeners=PLAINTEXT://localhost:9092
log.dirs=/var/kafka-logs
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
```

### 클러스터 모니터링
- **JMX 메트릭**: 브로커 성능 및 상태 모니터링
- **Kafka Manager/CMAK**: 웹 기반 클러스터 관리 도구
- **주요 메트릭**: UnderReplicatedPartitions, OfflinePartitions, LeaderElectionRate

## 2. 파티셔닝 전략 및 컨슈머 그룹

### 파티셔닝 전략
```java
// 커스텀 파티셔너 구현
public class CustomPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, 
                        Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        
        if (key == null) {
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }
        
        // 사용자 ID 기반 파티셔닝
        if (key instanceof String) {
            String userId = (String) key;
            return Math.abs(userId.hashCode()) % numPartitions;
        }
        
        return Math.abs(key.hashCode()) % numPartitions;
    }
}
```

### 컨슈머 그룹 관리
```java
@Component
public class KafkaConsumerConfig {
    
    @Value("${kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processing-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
        
        return new DefaultKafkaConsumerFactory<>(props);
    }
}
```

## 3. 오프셋 관리 및 재처리

### 수동 오프셋 커밋
```java
@KafkaListener(topics = "order-events", groupId = "order-processing-group")
public void handleOrderEvent(ConsumerRecord<String, OrderEvent> record,
                           Acknowledgment acknowledgment) {
    try {
        // 주문 이벤트 처리
        orderService.processOrder(record.value());
        
        // 성공적으로 처리된 경우에만 오프셋 커밋
        acknowledgment.acknowledge();
        
    } catch (Exception e) {
        log.error("Failed to process order event: {}", record.value(), e);
        // 오프셋을 커밋하지 않아서 재처리 대상이 됨
    }
}
```

### 오프셋 리셋 및 재처리
```bash
# 특정 오프셋으로 리셋
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-processing-group \
  --topic order-events \
  --reset-offsets --to-offset 1000 \
  --execute

# 특정 시간으로 리셋
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-processing-group \
  --topic order-events \
  --reset-offsets --to-datetime 2024-01-01T00:00:00.000 \
  --execute
```

## 4. Kafka Connect 데이터 파이프라인

### Source Connector 설정
```json
{
  "name": "mysql-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "localhost",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "database.server.name": "ecommerce",
    "database.include.list": "ecommerce",
    "database.history.kafka.bootstrap.servers": "localhost:9092",
    "database.history.kafka.topic": "dbhistory.ecommerce",
    "include.schema.changes": "true",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
  }
}
```

### Sink Connector 설정
```json
{
  "name": "elasticsearch-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "topics": "ecommerce.orders",
    "connection.url": "http://localhost:9200",
    "type.name": "_doc",
    "key.ignore": "true",
    "schema.ignore": "true",
    "transforms": "TimestampRouter",
    "transforms.TimestampRouter.type": "org.apache.kafka.connect.transforms.TimestampRouter",
    "transforms.TimestampRouter.topic.format": "orders-${timestamp}",
    "transforms.TimestampRouter.timestamp.format": "yyyy-MM-dd"
  }
}
```

## 5. 성능 최적화

### 프로듀서 최적화
```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // 성능 최적화 설정
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 10);
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864);
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.ACKS_CONFIG, "1");
        
        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

### 컨슈머 최적화
```properties
# 배치 처리 최적화
max.poll.records=500
fetch.min.bytes=50000
fetch.max.wait.ms=500

# 네트워크 최적화
receive.buffer.bytes=262144
send.buffer.bytes=131072
```

## 6. 장애 대응 및 복구

### 브로커 장애 대응
```bash
# 브로커 상태 확인
kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# 언더 레플리케이티드 파티션 확인
kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions

# 리더 재선출 트리거
kafka-leader-election.sh --bootstrap-server localhost:9092 --election-type preferred --all-topic-partitions
```

### 데이터 백업 및 복구
```bash
# 토픽 메타데이터 백업
kafka-topics.sh --bootstrap-server localhost:9092 --list > topics_backup.txt

# 컨슈머 그룹 오프셋 백업
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list | \
  xargs -I {} kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group {} --describe > consumer_groups_backup.txt
```

## 7. 모니터링 대시보드

### Prometheus + Grafana 설정
```yaml
# docker-compose.yml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
  
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

### 주요 메트릭 알럿
```yaml
# prometheus-alerts.yml
groups:
- name: kafka-alerts
  rules:
  - alert: KafkaConsumerLag
    expr: kafka_consumer_lag_sum > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kafka consumer lag is high"
      description: "Consumer group {{ $labels.group }} lag is {{ $value }}"
```

## 실습 과제

1. **3-브로커 Kafka 클러스터 구성**: Docker Compose를 사용하여 3개 브로커로 구성된 클러스터 설정
2. **파티셔닝 전략 구현**: 사용자 ID 기반 커스텀 파티셔너 구현 및 테스트
3. **Connect 파이프라인**: MySQL → Kafka → Elasticsearch 데이터 파이프라인 구축
4. **성능 벤치마크**: kafka-producer-perf-test.sh를 사용한 처리량 측정 및 최적화
5. **장애 시뮬레이션**: 브로커 다운 상황에서 자동 복구 테스트

## 참고 자료

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Confluent Platform Documentation](https://docs.confluent.io/)
- [Kafka: The Definitive Guide](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/)
- [Kafka Connect Deep Dive](https://www.confluent.io/blog/kafka-connect-deep-dive-jdbc-source-connector/)