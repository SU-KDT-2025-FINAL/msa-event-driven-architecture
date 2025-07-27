# 대용량 데이터 스트리밍 처리 가이드

## 1. Kafka Streams 실시간 분석

### 스트림 토폴로지 설계
```java
@Configuration
@EnableKafkaStreams
public class StreamsConfig {
    
    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public KafkaStreamsConfiguration kStreamsConfig() {
        Map<String, Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "ecommerce-analytics");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 10000);
        props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 0);
        props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);
        
        return new KafkaStreamsConfiguration(props);
    }
    
    @Bean
    public Serde<OrderEvent> orderEventSerde() {
        return Serdes.serdeFrom(new JsonSerializer<>(), new JsonDeserializer<>(OrderEvent.class));
    }
    
    @Bean
    public Serde<ProductViewEvent> productViewEventSerde() {
        return Serdes.serdeFrom(new JsonSerializer<>(), new JsonDeserializer<>(ProductViewEvent.class));
    }
}
```

### 실시간 주문 분석 스트림
```java
@Component
public class OrderAnalyticsStream {
    
    private final Serde<OrderEvent> orderEventSerde;
    private final Serde<OrderMetrics> orderMetricsSerde;
    
    @Autowired
    public void buildPipeline(StreamsBuilder streamsBuilder) {
        // 주문 이벤트 스트림
        KStream<String, OrderEvent> orderStream = streamsBuilder
            .stream("order-events", Consumed.with(Serdes.String(), orderEventSerde))
            .filter((key, value) -> value.getEventType().equals("ORDER_CREATED"));
        
        // 실시간 매출 집계 (5분 윈도우)
        buildRevenueStream(orderStream);
        
        // 상품별 주문 횟수 집계
        buildProductOrderCountStream(orderStream);
        
        // 고객별 주문 패턴 분석
        buildCustomerOrderPatternStream(orderStream);
        
        // 시간대별 주문 트렌드 분석
        buildHourlyOrderTrendStream(orderStream);
    }
    
    private void buildRevenueStream(KStream<String, OrderEvent> orderStream) {
        KTable<Windowed<String>, RevenueMetrics> revenueTable = orderStream
            .selectKey((key, value) -> "total-revenue")
            .groupByKey(Grouped.with(Serdes.String(), orderEventSerde))
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5)).advanceBy(Duration.ofMinutes(1)))
            .aggregate(
                RevenueMetrics::new,
                (key, value, aggregate) -> {
                    aggregate.addOrder(value.getTotalAmount(), value.getOrderDate());
                    return aggregate;
                },
                Materialized.<String, RevenueMetrics, WindowStore<Bytes, byte[]>>as("revenue-store")
                    .withValueSerde(Serdes.serdeFrom(new JsonSerializer<>(), 
                                                   new JsonDeserializer<>(RevenueMetrics.class)))
            );
        
        // 결과를 Kafka 토픽으로 출력
        revenueTable.toStream()
            .map((windowedKey, value) -> {
                String timeWindow = windowedKey.window().startTime().toString() + 
                                  "_" + windowedKey.window().endTime().toString();
                return KeyValue.pair(timeWindow, value);
            })
            .to("revenue-metrics", Produced.with(Serdes.String(), 
                Serdes.serdeFrom(new JsonSerializer<>(), new JsonDeserializer<>(RevenueMetrics.class))));
    }
    
    private void buildProductOrderCountStream(KStream<String, OrderEvent> orderStream) {
        KStream<String, ProductOrderCount> productOrderStream = orderStream
            .flatMap((key, order) -> {
                List<KeyValue<String, ProductOrderCount>> result = new ArrayList<>();
                for (OrderItem item : order.getItems()) {
                    result.add(KeyValue.pair(
                        item.getProductId(),
                        new ProductOrderCount(item.getProductId(), item.getQuantity(), 
                                           order.getTotalAmount(), order.getOrderDate())
                    ));
                }
                return result;
            });
        
        KTable<String, ProductMetrics> productMetricsTable = productOrderStream
            .groupByKey(Grouped.with(Serdes.String(), 
                Serdes.serdeFrom(new JsonSerializer<>(), new JsonDeserializer<>(ProductOrderCount.class))))
            .aggregate(
                ProductMetrics::new,
                (productId, orderCount, aggregate) -> {
                    aggregate.addOrder(orderCount.getQuantity(), orderCount.getRevenue());
                    aggregate.updateLastOrderTime(orderCount.getOrderTime());
                    return aggregate;
                },
                Materialized.<String, ProductMetrics, KeyValueStore<Bytes, byte[]>>as("product-metrics-store")
                    .withValueSerde(Serdes.serdeFrom(new JsonSerializer<>(), 
                                                   new JsonDeserializer<>(ProductMetrics.class)))
            );
        
        productMetricsTable.toStream()
            .to("product-metrics", Produced.with(Serdes.String(), 
                Serdes.serdeFrom(new JsonSerializer<>(), new JsonDeserializer<>(ProductMetrics.class))));
    }
    
    private void buildCustomerOrderPatternStream(KStream<String, OrderEvent> orderStream) {
        KTable<String, CustomerMetrics> customerMetricsTable = orderStream
            .selectKey((key, value) -> value.getCustomerId())
            .groupByKey(Grouped.with(Serdes.String(), orderEventSerde))
            .aggregate(
                CustomerMetrics::new,
                (customerId, order, aggregate) -> {
                    aggregate.addOrder(order.getTotalAmount(), order.getOrderDate());
                    aggregate.updateOrderFrequency();
                    aggregate.calculateAverageOrderValue();
                    return aggregate;
                },
                Materialized.<String, CustomerMetrics, KeyValueStore<Bytes, byte[]>>as("customer-metrics-store")
                    .withValueSerde(Serdes.serdeFrom(new JsonSerializer<>(), 
                                                   new JsonDeserializer<>(CustomerMetrics.class)))
            );
        
        // 고가치 고객 식별
        customerMetricsTable.toStream()
            .filter((customerId, metrics) -> metrics.getTotalRevenue().compareTo(new BigDecimal("1000")) > 0)
            .to("high-value-customers", Produced.with(Serdes.String(), 
                Serdes.serdeFrom(new JsonSerializer<>(), new JsonDeserializer<>(CustomerMetrics.class))));
    }
    
    private void buildHourlyOrderTrendStream(KStream<String, OrderEvent> orderStream) {
        KTable<Windowed<String>, HourlyTrend> hourlyTrendTable = orderStream
            .selectKey((key, value) -> String.valueOf(value.getOrderDate().atZone(ZoneId.systemDefault()).getHour()))
            .groupByKey(Grouped.with(Serdes.String(), orderEventSerde))
            .windowedBy(TimeWindows.of(Duration.ofHours(1)))
            .aggregate(
                HourlyTrend::new,
                (hour, order, aggregate) -> {
                    aggregate.incrementOrderCount();
                    aggregate.addRevenue(order.getTotalAmount());
                    aggregate.addItemCount(order.getItems().size());
                    return aggregate;
                },
                Materialized.<String, HourlyTrend, WindowStore<Bytes, byte[]>>as("hourly-trend-store")
                    .withValueSerde(Serdes.serdeFrom(new JsonSerializer<>(), 
                                                   new JsonDeserializer<>(HourlyTrend.class)))
            );
        
        hourlyTrendTable.toStream()
            .map((windowedKey, value) -> {
                String key = "hour-" + windowedKey.key() + "-" + 
                           windowedKey.window().startTime().toString();
                return KeyValue.pair(key, value);
            })
            .to("hourly-trends", Produced.with(Serdes.String(), 
                Serdes.serdeFrom(new JsonSerializer<>(), new JsonDeserializer<>(HourlyTrend.class))));
    }
}
```

### 복잡한 이벤트 처리 (CEP)
```java
@Component
public class ComplexEventProcessor {
    
    @Autowired
    public void buildFraudDetectionStream(StreamsBuilder streamsBuilder) {
        KStream<String, OrderEvent> orderStream = streamsBuilder
            .stream("order-events", Consumed.with(Serdes.String(), orderEventSerde));
        
        // 의심스러운 패턴 감지
        detectSuspiciousPatterns(orderStream);
        
        // 비정상적인 대량 주문 감지
        detectAbnormalBulkOrders(orderStream);
        
        // 연속적인 실패 패턴 감지
        detectConsecutiveFailures(orderStream);
    }
    
    private void detectSuspiciousPatterns(KStream<String, OrderEvent> orderStream) {
        // 같은 고객이 5분 내에 5개 이상 주문 시 의심스러운 활동으로 간주
        KTable<Windowed<String>, Long> orderCountTable = orderStream
            .selectKey((key, value) -> value.getCustomerId())
            .groupByKey(Grouped.with(Serdes.String(), orderEventSerde))
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
            .count(Materialized.as("customer-order-count-store"));
        
        orderCountTable.toStream()
            .filter((windowedCustomerId, count) -> count >= 5)
            .map((windowedCustomerId, count) -> {
                SuspiciousActivityAlert alert = new SuspiciousActivityAlert(
                    windowedCustomerId.key(),
                    "HIGH_FREQUENCY_ORDERS",
                    count,
                    windowedCustomerId.window().startTime(),
                    windowedCustomerId.window().endTime()
                );
                return KeyValue.pair(windowedCustomerId.key(), alert);
            })
            .to("suspicious-activity-alerts");
    }
    
    private void detectAbnormalBulkOrders(KStream<String, OrderEvent> orderStream) {
        // 비정상적으로 큰 주문 감지 (평균의 3배 이상)
        KTable<String, CustomerOrderStats> customerStatsTable = orderStream
            .selectKey((key, value) -> value.getCustomerId())
            .groupByKey(Grouped.with(Serdes.String(), orderEventSerde))
            .aggregate(
                CustomerOrderStats::new,
                (customerId, order, stats) -> {
                    stats.addOrder(order.getTotalAmount());
                    return stats;
                },
                Materialized.as("customer-stats-store")
            );
        
        // 새로운 주문과 기존 통계 비교
        orderStream
            .selectKey((key, value) -> value.getCustomerId())
            .join(customerStatsTable, (order, stats) -> {
                BigDecimal averageOrderValue = stats.getAverageOrderValue();
                BigDecimal threshold = averageOrderValue.multiply(new BigDecimal("3"));
                
                if (order.getTotalAmount().compareTo(threshold) > 0) {
                    return new AbnormalOrderAlert(
                        order.getOrderId(),
                        order.getCustomerId(),
                        order.getTotalAmount(),
                        averageOrderValue,
                        "ORDER_VALUE_ANOMALY"
                    );
                }
                return null;
            })
            .filter((customerId, alert) -> alert != null)
            .to("abnormal-order-alerts");
    }
}
```

## 2. 이벤트 기반 ETL 파이프라인

### Kafka Connect를 활용한 데이터 파이프라인
```yaml
# Source Connector - 데이터베이스에서 Kafka로
source-connector:
  name: "ecommerce-db-source"
  config:
    connector.class: "io.debezium.connector.postgresql.PostgresConnector"
    tasks.max: "3"
    database.hostname: "postgres-db"
    database.port: "5432"
    database.user: "debezium"
    database.password: "password"
    database.dbname: "ecommerce"
    database.server.name: "ecommerce-db"
    table.include.list: "public.orders,public.order_items,public.customers,public.products"
    database.history.kafka.bootstrap.servers: "kafka:9092"
    database.history.kafka.topic: "schema-changes.ecommerce"
    transforms: "unwrap"
    transforms.unwrap.type: "io.debezium.transforms.ExtractNewRecordState"
    transforms.unwrap.drop.tombstones: "false"
    transforms.unwrap.delete.handling.mode: "rewrite"

# Sink Connector - Kafka에서 Data Warehouse로
sink-connector:
  name: "ecommerce-warehouse-sink"
  config:
    connector.class: "io.confluent.connect.jdbc.JdbcSinkConnector"
    tasks.max: "3"
    connection.url: "jdbc:postgresql://warehouse-db:5432/analytics"
    connection.user: "warehouse_user"
    connection.password: "password"
    topics: "ecommerce-db.public.orders,ecommerce-db.public.customers"
    auto.create: "true"
    auto.evolve: "true"
    insert.mode: "upsert"
    pk.mode: "record_key"
    table.name.format: "analytics.${topic}"
    transforms: "TimestampRouter"
    transforms.TimestampRouter.type: "org.apache.kafka.connect.transforms.TimestampRouter"
    transforms.TimestampRouter.topic.format: "${topic}_${timestamp}"
    transforms.TimestampRouter.timestamp.format: "yyyy_MM_dd"
```

### 실시간 데이터 변환 및 집계
```java
@Component
public class RealTimeETLProcessor {
    
    @Autowired
    public void buildETLPipeline(StreamsBuilder streamsBuilder) {
        // 원본 데이터 스트림
        KStream<String, DatabaseEvent> dbEventStream = streamsBuilder
            .stream("ecommerce-db.public.orders", 
                   Consumed.with(Serdes.String(), databaseEventSerde));
        
        // 데이터 정제 및 변환
        KStream<String, OrderDW> cleanedOrderStream = dbEventStream
            .filter(this::isValidOrder)
            .map(this::transformToDataWarehouseFormat)
            .filter((key, value) -> value != null);
        
        // 차원 테이블과 조인
        enrichWithDimensions(cleanedOrderStream);
        
        // 집계 테이블 생성
        buildDailyAggregates(cleanedOrderStream);
        buildMonthlyAggregates(cleanedOrderStream);
        
        // 실시간 KPI 계산
        calculateRealTimeKPIs(cleanedOrderStream);
    }
    
    private boolean isValidOrder(String key, DatabaseEvent event) {
        if (event.getPayload() == null) return false;
        
        Map<String, Object> orderData = event.getPayload().getAfter();
        if (orderData == null) return false;
        
        // 데이터 품질 체크
        Object totalAmount = orderData.get("total_amount");
        Object customerId = orderData.get("customer_id");
        Object status = orderData.get("status");
        
        return totalAmount != null && 
               customerId != null && 
               status != null &&
               !status.equals("CANCELLED");
    }
    
    private KeyValue<String, OrderDW> transformToDataWarehouseFormat(String key, DatabaseEvent event) {
        try {
            Map<String, Object> orderData = event.getPayload().getAfter();
            
            OrderDW orderDW = OrderDW.builder()
                .orderId((String) orderData.get("id"))
                .customerId((String) orderData.get("customer_id"))
                .totalAmount(new BigDecimal(orderData.get("total_amount").toString()))
                .status(OrderStatus.valueOf((String) orderData.get("status")))
                .orderDate(parseTimestamp(orderData.get("created_at")))
                .lastModified(parseTimestamp(orderData.get("updated_at")))
                .build();
            
            return KeyValue.pair(orderDW.getOrderId(), orderDW);
            
        } catch (Exception e) {
            log.error("Failed to transform order data: {}", event, e);
            return KeyValue.pair(key, null);
        }
    }
    
    private void enrichWithDimensions(KStream<String, OrderDW> orderStream) {
        // 고객 차원 테이블과 조인
        KTable<String, CustomerDimension> customerTable = streamsBuilder
            .table("customer-dimension", 
                  Consumed.with(Serdes.String(), customerDimensionSerde));
        
        // 상품 차원 테이블과 조인
        KTable<String, ProductDimension> productTable = streamsBuilder
            .table("product-dimension", 
                  Consumed.with(Serdes.String(), productDimensionSerde));
        
        KStream<String, EnrichedOrderDW> enrichedOrderStream = orderStream
            .selectKey((key, order) -> order.getCustomerId())
            .join(customerTable, (order, customer) -> {
                return EnrichedOrderDW.builder()
                    .order(order)
                    .customerSegment(customer.getSegment())
                    .customerRegion(customer.getRegion())
                    .customerLifetimeValue(customer.getLifetimeValue())
                    .build();
            });
        
        // Data Warehouse로 전송
        enrichedOrderStream
            .selectKey((key, enrichedOrder) -> enrichedOrder.getOrder().getOrderId())
            .to("enriched-orders-dw", 
               Produced.with(Serdes.String(), enrichedOrderDWSerde));
    }
    
    private void buildDailyAggregates(KStream<String, OrderDW> orderStream) {
        KTable<Windowed<String>, DailyOrderAggregate> dailyAggregateTable = orderStream
            .selectKey((key, order) -> formatDate(order.getOrderDate()))
            .groupByKey(Grouped.with(Serdes.String(), orderDWSerde))
            .windowedBy(TimeWindows.of(Duration.ofDays(1)))
            .aggregate(
                DailyOrderAggregate::new,
                (date, order, aggregate) -> {
                    aggregate.addOrder(order);
                    return aggregate;
                },
                Materialized.<String, DailyOrderAggregate, WindowStore<Bytes, byte[]>>as("daily-aggregates-store")
                    .withValueSerde(dailyOrderAggregateSerde)
            );
        
        dailyAggregateTable.toStream()
            .map((windowedKey, aggregate) -> {
                String key = windowedKey.key() + "_" + 
                           formatDate(windowedKey.window().startTime());
                return KeyValue.pair(key, aggregate);
            })
            .to("daily-order-aggregates", 
               Produced.with(Serdes.String(), dailyOrderAggregateSerde));
    }
    
    private void calculateRealTimeKPIs(KStream<String, OrderDW> orderStream) {
        // 실시간 매출 KPI
        KTable<String, BigDecimal> revenueKPI = orderStream
            .selectKey((key, order) -> "total-revenue")
            .groupByKey(Grouped.with(Serdes.String(), orderDWSerde))
            .aggregate(
                () -> BigDecimal.ZERO,
                (key, order, revenue) -> revenue.add(order.getTotalAmount()),
                Materialized.<String, BigDecimal, KeyValueStore<Bytes, byte[]>>as("revenue-kpi-store")
                    .withValueSerde(Serdes.serdeFrom(new JsonSerializer<>(), 
                                                   new JsonDeserializer<>(BigDecimal.class)))
            );
        
        // 실시간 주문 수 KPI
        KTable<String, Long> orderCountKPI = orderStream
            .selectKey((key, order) -> "total-orders")
            .groupByKey(Grouped.with(Serdes.String(), orderDWSerde))
            .count(Materialized.as("order-count-kpi-store"));
        
        // 평균 주문 가격 KPI
        KTable<String, Double> avgOrderValueKPI = orderStream
            .selectKey((key, order) -> "avg-order-value")
            .groupByKey(Grouped.with(Serdes.String(), orderDWSerde))
            .aggregate(
                AvgOrderValueCalculator::new,
                (key, order, calculator) -> {
                    calculator.addOrder(order.getTotalAmount());
                    return calculator;
                },
                Materialized.<String, AvgOrderValueCalculator, KeyValueStore<Bytes, byte[]>>as("avg-order-value-store")
                    .withValueSerde(avgOrderValueCalculatorSerde)
            )
            .mapValues(AvgOrderValueCalculator::getAverage);
        
        // KPI를 실시간 대시보드로 전송
        revenueKPI.toStream().to("kpi-revenue");
        orderCountKPI.toStream().to("kpi-order-count");
        avgOrderValueKPI.toStream().to("kpi-avg-order-value");
    }
}
```

## 3. 배치/스트림 처리 하이브리드

### 람다 아키텍처 구현
```java
@Component
public class HybridDataProcessor {
    
    private final BatchProcessingService batchService;
    private final StreamProcessingService streamService;
    private final DataReconciliationService reconciliationService;
    
    @Scheduled(cron = "0 0 2 * * ?") // 매일 새벽 2시 배치 처리
    public void runDailyBatchProcessing() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        
        try {
            // 배치 레이어: 정확한 히스토리컬 데이터 처리
            BatchProcessingResult batchResult = batchService.processHistoricalData(yesterday);
            
            // 스트림 레이어와 배치 레이어 결과 비교
            StreamProcessingResult streamResult = streamService.getStreamResult(yesterday);
            
            // 데이터 정합성 검증 및 보정
            reconciliationService.reconcile(batchResult, streamResult);
            
            log.info("Daily batch processing completed for date: {}", yesterday);
            
        } catch (Exception e) {
            log.error("Daily batch processing failed for date: {}", yesterday, e);
            // 알림 발송
            sendBatchProcessingFailureAlert(yesterday, e);
        }
    }
    
    @Component
    public static class BatchProcessingService {
        
        private final JdbcTemplate jdbcTemplate;
        private final RedisTemplate<String, Object> redisTemplate;
        
        public BatchProcessingResult processHistoricalData(LocalDate date) {
            String startDate = date.atStartOfDay().toString();
            String endDate = date.plusDays(1).atStartOfDay().toString();
            
            // 대용량 데이터 배치 처리
            List<DailyMetrics> metrics = calculateDailyMetrics(startDate, endDate);
            List<CustomerSegmentation> segments = recalculateCustomerSegments(date);
            List<ProductRanking> rankings = calculateProductRankings(startDate, endDate);
            
            // 배치 처리 결과를 캐시에 저장
            storeBatchResults(date, metrics, segments, rankings);
            
            return new BatchProcessingResult(date, metrics, segments, rankings);
        }
        
        private List<DailyMetrics> calculateDailyMetrics(String startDate, String endDate) {
            String sql = """
                SELECT 
                    DATE(created_at) as order_date,
                    COUNT(*) as order_count,
                    SUM(total_amount) as total_revenue,
                    AVG(total_amount) as avg_order_value,
                    COUNT(DISTINCT customer_id) as unique_customers
                FROM orders 
                WHERE created_at >= ? AND created_at < ?
                GROUP BY DATE(created_at)
                """;
            
            return jdbcTemplate.query(sql, 
                (rs, rowNum) -> DailyMetrics.builder()
                    .date(rs.getDate("order_date").toLocalDate())
                    .orderCount(rs.getLong("order_count"))
                    .totalRevenue(rs.getBigDecimal("total_revenue"))
                    .avgOrderValue(rs.getBigDecimal("avg_order_value"))
                    .uniqueCustomers(rs.getLong("unique_customers"))
                    .build(),
                startDate, endDate);
        }
        
        private List<CustomerSegmentation> recalculateCustomerSegments(LocalDate date) {
            // RFM 분석을 통한 고객 세분화
            String sql = """
                WITH customer_rfm AS (
                    SELECT 
                        customer_id,
                        EXTRACT(DAYS FROM (CURRENT_DATE - MAX(created_at))) as recency,
                        COUNT(*) as frequency,
                        AVG(total_amount) as monetary
                    FROM orders 
                    WHERE created_at <= ?
                    GROUP BY customer_id
                ),
                rfm_scores AS (
                    SELECT 
                        customer_id,
                        recency,
                        frequency,
                        monetary,
                        NTILE(5) OVER (ORDER BY recency DESC) as r_score,
                        NTILE(5) OVER (ORDER BY frequency) as f_score,
                        NTILE(5) OVER (ORDER BY monetary) as m_score
                    FROM customer_rfm
                )
                SELECT 
                    customer_id,
                    r_score,
                    f_score,
                    m_score,
                    CASE 
                        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
                        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Loyal Customers'
                        WHEN r_score >= 3 AND f_score <= 2 AND m_score >= 3 THEN 'Potential Loyalists'
                        WHEN r_score <= 2 AND f_score <= 2 AND m_score <= 2 THEN 'At Risk'
                        ELSE 'Others'
                    END as segment
                FROM rfm_scores
                """;
            
            return jdbcTemplate.query(sql,
                (rs, rowNum) -> CustomerSegmentation.builder()
                    .customerId(rs.getString("customer_id"))
                    .recencyScore(rs.getInt("r_score"))
                    .frequencyScore(rs.getInt("f_score"))
                    .monetaryScore(rs.getInt("m_score"))
                    .segment(rs.getString("segment"))
                    .calculatedDate(date)
                    .build(),
                date.plusDays(1));
        }
    }
    
    @Component
    public static class DataReconciliationService {
        
        private final MeterRegistry meterRegistry;
        private final Counter reconciliationCounter;
        private final Timer reconciliationTimer;
        
        public DataReconciliationService(MeterRegistry meterRegistry) {
            this.meterRegistry = meterRegistry;
            this.reconciliationCounter = Counter.builder("data.reconciliation")
                .description("Data reconciliation operations")
                .register(meterRegistry);
            this.reconciliationTimer = Timer.builder("data.reconciliation.duration")
                .description("Data reconciliation duration")
                .register(meterRegistry);
        }
        
        public void reconcile(BatchProcessingResult batchResult, 
                            StreamProcessingResult streamResult) {
            Timer.Sample sample = Timer.start(meterRegistry);
            
            try {
                // 매출 데이터 정합성 검증
                reconcileRevenue(batchResult, streamResult);
                
                // 주문 수 정합성 검증
                reconcileOrderCount(batchResult, streamResult);
                
                // 고객 메트릭 정합성 검증
                reconcileCustomerMetrics(batchResult, streamResult);
                
                reconciliationCounter.increment(Tags.of("status", "success"));
                
            } catch (Exception e) {
                reconciliationCounter.increment(Tags.of("status", "failed"));
                throw e;
            } finally {
                sample.stop(reconciliationTimer);
            }
        }
        
        private void reconcileRevenue(BatchProcessingResult batchResult, 
                                    StreamProcessingResult streamResult) {
            BigDecimal batchRevenue = batchResult.getTotalRevenue();
            BigDecimal streamRevenue = streamResult.getTotalRevenue();
            
            BigDecimal difference = batchRevenue.subtract(streamRevenue).abs();
            BigDecimal threshold = batchRevenue.multiply(new BigDecimal("0.01")); // 1% 허용 오차
            
            if (difference.compareTo(threshold) > 0) {
                log.warn("Revenue reconciliation failed. Batch: {}, Stream: {}, Difference: {}", 
                        batchRevenue, streamRevenue, difference);
                
                // 스트림 결과를 배치 결과로 보정
                correctStreamResult("revenue", batchRevenue, streamRevenue);
                
                // 알림 발송
                sendReconciliationAlert("revenue", batchRevenue, streamRevenue, difference);
            } else {
                log.info("Revenue reconciliation successful. Difference within threshold: {}", difference);
            }
        }
        
        private void correctStreamResult(String metric, Object batchValue, Object streamValue) {
            // 스트림 처리 결과를 배치 처리 결과로 보정
            StreamCorrectionEvent correctionEvent = new StreamCorrectionEvent(
                metric,
                streamValue,
                batchValue,
                Instant.now()
            );
            
            // 보정 이벤트 발행
            eventPublisher.publishEvent(correctionEvent);
            
            // 메트릭 업데이트
            meterRegistry.counter("data.correction", "metric", metric).increment();
        }
    }
}
```

## 4. 데이터 품질 모니터링

### 실시간 데이터 품질 체크
```java
@Component
public class DataQualityMonitor {
    
    private final MeterRegistry meterRegistry;
    private final Counter qualityIssueCounter;
    private final Gauge dataCompletenessGauge;
    
    @Autowired
    public void buildQualityMonitoringStream(StreamsBuilder streamsBuilder) {
        KStream<String, OrderEvent> orderStream = streamsBuilder
            .stream("order-events", Consumed.with(Serdes.String(), orderEventSerde));
        
        // 데이터 완전성 체크
        KStream<String, DataQualityResult> qualityCheckStream = orderStream
            .map(this::checkDataQuality);
        
        // 품질 이슈 필터링 및 알림
        qualityCheckStream
            .filter((key, result) -> !result.isValid())
            .foreach(this::handleQualityIssue);
        
        // 품질 메트릭 집계
        aggregateQualityMetrics(qualityCheckStream);
    }
    
    private KeyValue<String, DataQualityResult> checkDataQuality(String key, OrderEvent order) {
        DataQualityResult result = new DataQualityResult(order.getOrderId());
        
        // 필수 필드 체크
        if (order.getCustomerId() == null || order.getCustomerId().trim().isEmpty()) {
            result.addIssue("MISSING_CUSTOMER_ID", "Customer ID is missing or empty");
        }
        
        if (order.getTotalAmount() == null || order.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
            result.addIssue("INVALID_TOTAL_AMOUNT", "Total amount is missing or invalid");
        }
        
        if (order.getItems() == null || order.getItems().isEmpty()) {
            result.addIssue("MISSING_ORDER_ITEMS", "Order items are missing");
        }
        
        // 데이터 일관성 체크
        if (order.getItems() != null && !order.getItems().isEmpty()) {
            BigDecimal calculatedTotal = order.getItems().stream()
                .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            
            BigDecimal difference = order.getTotalAmount().subtract(calculatedTotal).abs();
            BigDecimal tolerance = order.getTotalAmount().multiply(new BigDecimal("0.01"));
            
            if (difference.compareTo(tolerance) > 0) {
                result.addIssue("TOTAL_AMOUNT_MISMATCH", 
                    String.format("Calculated total %s doesn't match provided total %s", 
                                calculatedTotal, order.getTotalAmount()));
            }
        }
        
        // 비즈니스 규칙 체크
        if (order.getOrderDate() != null && order.getOrderDate().isAfter(Instant.now().plus(Duration.ofHours(1)))) {
            result.addIssue("FUTURE_ORDER_DATE", "Order date is in the future");
        }
        
        // 데이터 형식 체크
        if (!isValidEmail(order.getCustomerEmail())) {
            result.addIssue("INVALID_EMAIL_FORMAT", "Customer email format is invalid");
        }
        
        return KeyValue.pair(order.getOrderId(), result);
    }
    
    private void handleQualityIssue(String orderId, DataQualityResult result) {
        // 품질 이슈 카운터 증가
        for (DataQualityIssue issue : result.getIssues()) {
            qualityIssueCounter.increment(
                Tags.of("issue_type", issue.getType(), "severity", issue.getSeverity().name())
            );
        }
        
        // 심각한 이슈인 경우 즉시 알림
        if (result.hasCriticalIssues()) {
            DataQualityAlert alert = new DataQualityAlert(
                orderId,
                result.getIssues(),
                Instant.now()
            );
            
            // 알림 발송
            kafkaTemplate.send("data-quality-alerts", orderId, alert);
            
            // 장애 복구를 위한 데이터 격리
            quarantineData(orderId, result);
        }
        
        log.warn("Data quality issues detected for order {}: {}", orderId, result.getIssues());
    }
    
    private void aggregateQualityMetrics(KStream<String, DataQualityResult> qualityStream) {
        // 시간대별 데이터 품질 집계
        KTable<Windowed<String>, DataQualityMetrics> qualityMetricsTable = qualityStream
            .selectKey((key, result) -> "quality-metrics")
            .groupByKey(Grouped.with(Serdes.String(), dataQualityResultSerde))
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
            .aggregate(
                DataQualityMetrics::new,
                (key, result, metrics) -> {
                    metrics.incrementTotalRecords();
                    if (result.isValid()) {
                        metrics.incrementValidRecords();
                    } else {
                        metrics.incrementInvalidRecords();
                        for (DataQualityIssue issue : result.getIssues()) {
                            metrics.addIssueType(issue.getType());
                        }
                    }
                    return metrics;
                },
                Materialized.as("quality-metrics-store")
            );
        
        // 품질 메트릭을 모니터링 시스템으로 전송
        qualityMetricsTable.toStream()
            .map((windowedKey, metrics) -> {
                String timeWindow = formatTimeWindow(windowedKey.window());
                QualityMetricsReport report = new QualityMetricsReport(
                    timeWindow,
                    metrics.calculateQualityScore(),
                    metrics.getIssueTypeCounts(),
                    metrics.getTotalRecords(),
                    metrics.getValidRecords(),
                    metrics.getInvalidRecords()
                );
                return KeyValue.pair(timeWindow, report);
            })
            .to("quality-metrics-reports");
    }
    
    private void quarantineData(String orderId, DataQualityResult result) {
        QuarantinedRecord quarantinedRecord = new QuarantinedRecord(
            orderId,
            result.getOriginalData(),
            result.getIssues(),
            Instant.now()
        );
        
        // 격리된 데이터를 별도 토픽으로 이동
        kafkaTemplate.send("quarantined-data", orderId, quarantinedRecord);
        
        // 격리 메트릭 업데이트
        meterRegistry.counter("data.quarantined", "reason", "quality_issues").increment();
    }
    
    @EventListener
    public void handleQuarantinedDataReview(QuarantinedDataReviewEvent event) {
        // 격리된 데이터 검토 및 복구 프로세스
        if (event.getAction() == QuarantineAction.APPROVE) {
            // 데이터 정정 후 재처리
            reprocessQuarantinedData(event.getRecordId(), event.getCorrectedData());
        } else if (event.getAction() == QuarantineAction.REJECT) {
            // 영구 삭제
            permanentlyDeleteQuarantinedData(event.getRecordId());
        }
    }
    
    private boolean isValidEmail(String email) {
        if (email == null || email.trim().isEmpty()) {
            return false;
        }
        return email.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$");
    }
}
```

## 5. 성능 최적화 및 스케일링

### 파티션 최적화 전략
```java
@Configuration
public class KafkaStreamOptimization {
    
    @Bean
    public KafkaStreamsConfiguration optimizedStreamsConfig() {
        Map<String, Object> props = new HashMap<>();
        
        // 기본 설정
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "optimized-ecommerce-analytics");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-cluster:9092");
        
        // 성능 최적화 설정
        props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 8);
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 30000);
        props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 100 * 1024 * 1024); // 100MB
        props.put(StreamsConfig.BUFFERED_RECORDS_PER_PARTITION_CONFIG, 2000);
        
        // 메모리 최적화
        props.put(StreamsConfig.ROCKSDB_CONFIG_SETTER_CLASS_CONFIG, CustomRocksDBConfig.class);
        
        // 네트워크 최적화
        props.put(StreamsConfig.PRODUCER_PREFIX + ProducerConfig.BATCH_SIZE_CONFIG, 65536);
        props.put(StreamsConfig.PRODUCER_PREFIX + ProducerConfig.LINGER_MS_CONFIG, 100);
        props.put(StreamsConfig.PRODUCER_PREFIX + ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        
        // 컨슈머 최적화
        props.put(StreamsConfig.CONSUMER_PREFIX + ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 50000);
        props.put(StreamsConfig.CONSUMER_PREFIX + ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);
        
        return new KafkaStreamsConfiguration(props);
    }
    
    public static class CustomRocksDBConfig implements RocksDBConfigSetter {
        
        @Override
        public void setConfig(String storeName, Options options, Map<String, Object> configs) {
            // 메모리 사용량 최적화
            options.setMaxWriteBufferNumber(3);
            options.setWriteBufferSize(64 * 1024 * 1024); // 64MB
            options.setMaxBackgroundJobs(4);
            
            // 압축 설정
            options.setCompressionType(CompressionType.SNAPPY_COMPRESSION);
            options.setCompactionStyle(CompactionStyle.LEVEL);
            
            // 블록 캐시 설정
            BlockBasedTableConfig tableConfig = new BlockBasedTableConfig();
            tableConfig.setBlockCacheSize(128 * 1024 * 1024); // 128MB
            tableConfig.setBlockSize(16 * 1024); // 16KB
            tableConfig.setCacheIndexAndFilterBlocks(true);
            options.setTableFormatConfig(tableConfig);
        }
        
        @Override
        public void close(String storeName, Options options) {
            // 리소스 정리
        }
    }
}
```

### 자동 스케일링 구성
```yaml
# Kubernetes HPA 설정
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kafka-streams-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kafka-streams-processor
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: kafka_consumer_lag_sum
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Max

---
# KEDA ScaledObject for Kafka-based scaling
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-streams-scaledobject
spec:
  scaleTargetRef:
    name: kafka-streams-processor
  minReplicaCount: 3
  maxReplicaCount: 20
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-cluster:9092
      consumerGroup: ecommerce-analytics
      topic: order-events
      lagThreshold: '1000'
      offsetResetPolicy: earliest
```

## 실습 과제

1. **완전한 스트리밍 파이프라인**: Kafka Streams를 활용한 실시간 전자상거래 분석 시스템 구현
2. **하이브리드 데이터 처리**: 배치와 스트림 처리를 결합한 람다 아키텍처 구현
3. **데이터 품질 모니터링**: 실시간 데이터 품질 체크 및 자동 복구 시스템 구현
4. **성능 벤치마크**: 대용량 데이터 처리 성능 최적화 및 스케일링 테스트
5. **ML 통합**: 실시간 스트림 데이터를 활용한 머신러닝 파이프라인 구축

## 참고 자료

- [Kafka Streams Documentation](https://kafka.apache.org/documentation/streams/)
- [Confluent Platform Documentation](https://docs.confluent.io/)
- [Building Streaming Applications with Apache Kafka](https://www.oreilly.com/library/view/kafka-streams-in/9781491974285/)
- [The Lambda Architecture](http://lambda-architecture.net/)