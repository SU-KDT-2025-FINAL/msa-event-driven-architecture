# 이벤트 소싱 구현 가이드

## 1. EventStore 데이터베이스 설계

### 이벤트 스토어 스키마
```sql
-- 이벤트 스토어 테이블
CREATE TABLE event_store (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id VARCHAR(255) NOT NULL,
    aggregate_type VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    event_data JSONB NOT NULL,
    event_metadata JSONB,
    version BIGINT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT uk_aggregate_version UNIQUE (aggregate_id, version)
);

-- 인덱스 생성
CREATE INDEX idx_event_store_aggregate_id ON event_store (aggregate_id);
CREATE INDEX idx_event_store_aggregate_type ON event_store (aggregate_type);
CREATE INDEX idx_event_store_created_at ON event_store (created_at);
CREATE INDEX idx_event_store_event_type ON event_store (event_type);

-- 스냅샷 테이블
CREATE TABLE aggregate_snapshots (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id VARCHAR(255) NOT NULL,
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_data JSONB NOT NULL,
    version BIGINT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT uk_snapshot_aggregate UNIQUE (aggregate_id)
);
```

### 이벤트 엔터티 설계
```java
@Entity
@Table(name = "event_store")
public class EventEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "aggregate_id", nullable = false)
    private String aggregateId;
    
    @Column(name = "aggregate_type", nullable = false)
    private String aggregateType;
    
    @Column(name = "event_type", nullable = false)
    private String eventType;
    
    @Type(JsonType.class)
    @Column(name = "event_data", nullable = false, columnDefinition = "jsonb")
    private String eventData;
    
    @Type(JsonType.class)
    @Column(name = "event_metadata", columnDefinition = "jsonb")
    private String eventMetadata;
    
    @Column(name = "version", nullable = false)
    private Long version;
    
    @Column(name = "created_at")
    @CreationTimestamp
    private Instant createdAt;
    
    // 생성자, getter, setter
}

@Entity
@Table(name = "aggregate_snapshots")
public class SnapshotEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "aggregate_id", nullable = false, unique = true)
    private String aggregateId;
    
    @Column(name = "aggregate_type", nullable = false)
    private String aggregateType;
    
    @Type(JsonType.class)
    @Column(name = "aggregate_data", nullable = false, columnDefinition = "jsonb")
    private String aggregateData;
    
    @Column(name = "version", nullable = false)
    private Long version;
    
    @Column(name = "created_at")
    @CreationTimestamp
    private Instant createdAt;
    
    // 생성자, getter, setter
}
```

## 2. 이벤트 및 애그리게이트 정의

### 도메인 이벤트 인터페이스
```java
public interface DomainEvent {
    String getAggregateId();
    String getEventType();
    Instant getOccurredAt();
    Long getVersion();
    Map<String, Object> getMetadata();
}

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "eventType")
@JsonSubTypes({
    @JsonSubTypes.Type(value = OrderCreatedEvent.class, name = "OrderCreated"),
    @JsonSubTypes.Type(value = OrderUpdatedEvent.class, name = "OrderUpdated"),
    @JsonSubTypes.Type(value = OrderCancelledEvent.class, name = "OrderCancelled"),
    @JsonSubTypes.Type(value = OrderCompletedEvent.class, name = "OrderCompleted")
})
public abstract class OrderEvent implements DomainEvent {
    
    protected String aggregateId;
    protected Instant occurredAt;
    protected Long version;
    protected Map<String, Object> metadata;
    
    public OrderEvent() {
        this.occurredAt = Instant.now();
        this.metadata = new HashMap<>();
    }
    
    @Override
    public String getEventType() {
        return this.getClass().getSimpleName();
    }
    
    // getter, setter
}
```

### 구체적인 이벤트 클래스
```java
public class OrderCreatedEvent extends OrderEvent {
    
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private OrderStatus status;
    
    public OrderCreatedEvent() {}
    
    public OrderCreatedEvent(String aggregateId, String customerId, 
                           List<OrderItem> items, BigDecimal totalAmount) {
        this.aggregateId = aggregateId;
        this.customerId = customerId;
        this.items = items;
        this.totalAmount = totalAmount;
        this.status = OrderStatus.PENDING;
        this.version = 1L;
    }
    
    // getter, setter
}

public class OrderUpdatedEvent extends OrderEvent {
    
    private List<OrderItem> updatedItems;
    private BigDecimal newTotalAmount;
    private String reason;
    
    // 생성자, getter, setter
}

public class OrderCancelledEvent extends OrderEvent {
    
    private String reason;
    private Instant cancelledAt;
    
    // 생성자, getter, setter
}
```

### 애그리게이트 루트 설계
```java
public abstract class AggregateRoot {
    
    protected String id;
    protected Long version;
    protected List<DomainEvent> uncommittedEvents;
    
    public AggregateRoot() {
        this.uncommittedEvents = new ArrayList<>();
        this.version = 0L;
    }
    
    protected void applyEvent(DomainEvent event) {
        event.setVersion(this.version + 1);
        this.version = event.getVersion();
        this.uncommittedEvents.add(event);
        this.apply(event);
    }
    
    protected abstract void apply(DomainEvent event);
    
    public List<DomainEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }
    
    public void markEventsAsCommitted() {
        this.uncommittedEvents.clear();
    }
    
    public void loadFromHistory(List<DomainEvent> history) {
        for (DomainEvent event : history) {
            this.apply(event);
            this.version = event.getVersion();
        }
    }
}

public class Order extends AggregateRoot {
    
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private OrderStatus status;
    private Instant createdAt;
    private Instant updatedAt;
    
    // 기본 생성자 (재구성용)
    public Order() {
        super();
        this.items = new ArrayList<>();
    }
    
    // 비즈니스 생성자
    public Order(String orderId, String customerId, List<OrderItem> items) {
        this();
        this.id = orderId;
        this.customerId = customerId;
        this.totalAmount = calculateTotalAmount(items);
        
        applyEvent(new OrderCreatedEvent(orderId, customerId, items, totalAmount));
    }
    
    // 비즈니스 메서드
    public void updateItems(List<OrderItem> newItems, String reason) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot update order in status: " + status);
        }
        
        BigDecimal newTotal = calculateTotalAmount(newItems);
        applyEvent(new OrderUpdatedEvent(id, newItems, newTotal, reason));
    }
    
    public void cancel(String reason) {
        if (status == OrderStatus.COMPLETED || status == OrderStatus.CANCELLED) {
            throw new IllegalStateException("Cannot cancel order in status: " + status);
        }
        
        applyEvent(new OrderCancelledEvent(id, reason));
    }
    
    public void complete() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot complete order in status: " + status);
        }
        
        applyEvent(new OrderCompletedEvent(id));
    }
    
    @Override
    protected void apply(DomainEvent event) {
        switch (event) {
            case OrderCreatedEvent e -> {
                this.customerId = e.getCustomerId();
                this.items = new ArrayList<>(e.getItems());
                this.totalAmount = e.getTotalAmount();
                this.status = OrderStatus.PENDING;
                this.createdAt = e.getOccurredAt();
                this.updatedAt = e.getOccurredAt();
            }
            case OrderUpdatedEvent e -> {
                this.items = new ArrayList<>(e.getUpdatedItems());
                this.totalAmount = e.getNewTotalAmount();
                this.updatedAt = e.getOccurredAt();
            }
            case OrderCancelledEvent e -> {
                this.status = OrderStatus.CANCELLED;
                this.updatedAt = e.getOccurredAt();
            }
            case OrderCompletedEvent e -> {
                this.status = OrderStatus.COMPLETED;
                this.updatedAt = e.getOccurredAt();
            }
            default -> throw new IllegalArgumentException("Unknown event type: " + event.getClass());
        }
    }
    
    private BigDecimal calculateTotalAmount(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    // getter 메서드들
}
```

## 3. 이벤트 스토어 구현

### EventStore 인터페이스
```java
public interface EventStore {
    
    void saveEvents(String aggregateId, String aggregateType, 
                   List<DomainEvent> events, Long expectedVersion);
    
    List<DomainEvent> getEvents(String aggregateId);
    
    List<DomainEvent> getEvents(String aggregateId, Long fromVersion);
    
    List<DomainEvent> getAllEvents(String aggregateType, Instant from, Instant to);
    
    void saveSnapshot(String aggregateId, String aggregateType, 
                     Object aggregate, Long version);
    
    <T> Optional<T> getSnapshot(String aggregateId, Class<T> aggregateType);
}
```

### EventStore 구현
```java
@Repository
@Transactional
public class JpaEventStore implements EventStore {
    
    private final EventEntityRepository eventRepository;
    private final SnapshotEntityRepository snapshotRepository;
    private final ObjectMapper objectMapper;
    
    public JpaEventStore(EventEntityRepository eventRepository,
                        SnapshotEntityRepository snapshotRepository,
                        ObjectMapper objectMapper) {
        this.eventRepository = eventRepository;
        this.snapshotRepository = snapshotRepository;
        this.objectMapper = objectMapper;
    }
    
    @Override
    public void saveEvents(String aggregateId, String aggregateType, 
                          List<DomainEvent> events, Long expectedVersion) {
        
        // 낙관적 동시성 제어
        Long currentVersion = eventRepository.findMaxVersionByAggregateId(aggregateId)
            .orElse(0L);
        
        if (!currentVersion.equals(expectedVersion)) {
            throw new ConcurrencyException(
                String.format("Expected version %d but was %d", expectedVersion, currentVersion)
            );
        }
        
        List<EventEntity> entities = events.stream()
            .map(event -> convertToEntity(aggregateId, aggregateType, event))
            .toList();
        
        eventRepository.saveAll(entities);
    }
    
    @Override
    public List<DomainEvent> getEvents(String aggregateId) {
        List<EventEntity> entities = eventRepository
            .findByAggregateIdOrderByVersionAsc(aggregateId);
        
        return entities.stream()
            .map(this::convertToDomainEvent)
            .toList();
    }
    
    @Override
    public List<DomainEvent> getEvents(String aggregateId, Long fromVersion) {
        List<EventEntity> entities = eventRepository
            .findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(aggregateId, fromVersion);
        
        return entities.stream()
            .map(this::convertToDomainEvent)
            .toList();
    }
    
    @Override
    public List<DomainEvent> getAllEvents(String aggregateType, Instant from, Instant to) {
        List<EventEntity> entities = eventRepository
            .findByAggregateTypeAndCreatedAtBetweenOrderByCreatedAtAsc(
                aggregateType, from, to);
        
        return entities.stream()
            .map(this::convertToDomainEvent)
            .toList();
    }
    
    @Override
    public void saveSnapshot(String aggregateId, String aggregateType, 
                           Object aggregate, Long version) {
        try {
            String aggregateData = objectMapper.writeValueAsString(aggregate);
            
            SnapshotEntity entity = snapshotRepository
                .findByAggregateId(aggregateId)
                .orElse(new SnapshotEntity());
            
            entity.setAggregateId(aggregateId);
            entity.setAggregateType(aggregateType);
            entity.setAggregateData(aggregateData);
            entity.setVersion(version);
            
            snapshotRepository.save(entity);
            
        } catch (JsonProcessingException e) {
            throw new EventStoreException("Failed to serialize aggregate", e);
        }
    }
    
    @Override
    public <T> Optional<T> getSnapshot(String aggregateId, Class<T> aggregateType) {
        return snapshotRepository.findByAggregateId(aggregateId)
            .map(entity -> {
                try {
                    return objectMapper.readValue(entity.getAggregateData(), aggregateType);
                } catch (JsonProcessingException e) {
                    throw new EventStoreException("Failed to deserialize snapshot", e);
                }
            });
    }
    
    private EventEntity convertToEntity(String aggregateId, String aggregateType, 
                                       DomainEvent event) {
        try {
            EventEntity entity = new EventEntity();
            entity.setAggregateId(aggregateId);
            entity.setAggregateType(aggregateType);
            entity.setEventType(event.getEventType());
            entity.setEventData(objectMapper.writeValueAsString(event));
            entity.setEventMetadata(objectMapper.writeValueAsString(event.getMetadata()));
            entity.setVersion(event.getVersion());
            
            return entity;
            
        } catch (JsonProcessingException e) {
            throw new EventStoreException("Failed to serialize event", e);
        }
    }
    
    private DomainEvent convertToDomainEvent(EventEntity entity) {
        try {
            Class<? extends DomainEvent> eventClass = getEventClass(entity.getEventType());
            DomainEvent event = objectMapper.readValue(entity.getEventData(), eventClass);
            
            // 메타데이터 복원
            if (entity.getEventMetadata() != null) {
                Map<String, Object> metadata = objectMapper.readValue(
                    entity.getEventMetadata(), 
                    new TypeReference<Map<String, Object>>() {}
                );
                event.setMetadata(metadata);
            }
            
            return event;
            
        } catch (JsonProcessingException e) {
            throw new EventStoreException("Failed to deserialize event", e);
        }
    }
    
    private Class<? extends DomainEvent> getEventClass(String eventType) {
        // 이벤트 타입에 따른 클래스 매핑
        return switch (eventType) {
            case "OrderCreatedEvent" -> OrderCreatedEvent.class;
            case "OrderUpdatedEvent" -> OrderUpdatedEvent.class;
            case "OrderCancelledEvent" -> OrderCancelledEvent.class;
            case "OrderCompletedEvent" -> OrderCompletedEvent.class;
            default -> throw new IllegalArgumentException("Unknown event type: " + eventType);
        };
    }
}
```

## 4. 애그리게이트 리포지토리 구현

### 이벤트 소싱 리포지토리
```java
public interface OrderRepository {
    Order findById(String orderId);
    void save(Order order);
    boolean exists(String orderId);
}

@Repository
public class EventSourcedOrderRepository implements OrderRepository {
    
    private final EventStore eventStore;
    private final ApplicationEventPublisher eventPublisher;
    
    // 스냅샷 생성 임계값
    private static final int SNAPSHOT_THRESHOLD = 20;
    
    public EventSourcedOrderRepository(EventStore eventStore,
                                     ApplicationEventPublisher eventPublisher) {
        this.eventStore = eventStore;
        this.eventPublisher = eventPublisher;
    }
    
    @Override
    public Order findById(String orderId) {
        // 스냅샷부터 로드 시도
        Optional<Order> snapshot = eventStore.getSnapshot(orderId, Order.class);
        
        Order order;
        Long fromVersion = 0L;
        
        if (snapshot.isPresent()) {
            order = snapshot.get();
            fromVersion = order.getVersion();
        } else {
            order = new Order();
        }
        
        // 스냅샷 이후 이벤트들 로드
        List<DomainEvent> events = eventStore.getEvents(orderId, fromVersion);
        
        if (events.isEmpty() && snapshot.isEmpty()) {
            throw new AggregateNotFoundException("Order not found: " + orderId);
        }
        
        // 이벤트 재생을 통한 상태 복원
        order.loadFromHistory(events);
        
        return order;
    }
    
    @Override
    public void save(Order order) {
        List<DomainEvent> uncommittedEvents = order.getUncommittedEvents();
        
        if (uncommittedEvents.isEmpty()) {
            return;
        }
        
        // 이벤트 저장
        eventStore.saveEvents(
            order.getId(),
            Order.class.getSimpleName(),
            uncommittedEvents,
            order.getVersion() - uncommittedEvents.size()
        );
        
        // 이벤트 발행
        uncommittedEvents.forEach(eventPublisher::publishEvent);
        
        // 스냅샷 생성 검토
        if (order.getVersion() % SNAPSHOT_THRESHOLD == 0) {
            eventStore.saveSnapshot(
                order.getId(),
                Order.class.getSimpleName(),
                order,
                order.getVersion()
            );
        }
        
        order.markEventsAsCommitted();
    }
    
    @Override
    public boolean exists(String orderId) {
        try {
            findById(orderId);
            return true;
        } catch (AggregateNotFoundException e) {
            return false;
        }
    }
}
```

## 5. 이벤트 재생 메커니즘

### 이벤트 재생 서비스
```java
@Service
public class EventReplayService {
    
    private final EventStore eventStore;
    
    public EventReplayService(EventStore eventStore) {
        this.eventStore = eventStore;
    }
    
    public void replayEvents(String aggregateType, Instant from, Instant to,
                           Consumer<DomainEvent> eventHandler) {
        
        List<DomainEvent> events = eventStore.getAllEvents(aggregateType, from, to);
        
        log.info("Replaying {} events for aggregate type: {}", events.size(), aggregateType);
        
        events.forEach(event -> {
            try {
                eventHandler.accept(event);
            } catch (Exception e) {
                log.error("Failed to replay event: {}", event, e);
                throw new EventReplayException("Event replay failed", e);
            }
        });
    }
    
    public <T extends AggregateRoot> T reconstructAggregate(String aggregateId, 
                                                           Class<T> aggregateClass) {
        try {
            T aggregate = aggregateClass.getDeclaredConstructor().newInstance();
            List<DomainEvent> events = eventStore.getEvents(aggregateId);
            aggregate.loadFromHistory(events);
            return aggregate;
            
        } catch (Exception e) {
            throw new AggregateReconstructionException(
                "Failed to reconstruct aggregate: " + aggregateId, e);
        }
    }
    
    public Map<String, Object> getAggregateStatistics(String aggregateType, 
                                                     Instant from, Instant to) {
        List<DomainEvent> events = eventStore.getAllEvents(aggregateType, from, to);
        
        Map<String, Long> eventCounts = events.stream()
            .collect(Collectors.groupingBy(
                DomainEvent::getEventType,
                Collectors.counting()
            ));
        
        Map<String, Long> dailyCounts = events.stream()
            .collect(Collectors.groupingBy(
                event -> event.getOccurredAt().truncatedTo(ChronoUnit.DAYS).toString(),
                Collectors.counting()
            ));
        
        return Map.of(
            "totalEvents", events.size(),
            "eventTypes", eventCounts,
            "dailyCounts", dailyCounts,
            "period", Map.of("from", from.toString(), "to", to.toString())
        );
    }
}
```

### 프로젝션 재구성
```java
@Service
public class ProjectionRebuildService {
    
    private final EventStore eventStore;
    private final List<EventProjection> projections;
    
    public ProjectionRebuildService(EventStore eventStore, 
                                  List<EventProjection> projections) {
        this.eventStore = eventStore;
        this.projections = projections;
    }
    
    public void rebuildProjection(String projectionName) {
        EventProjection projection = findProjection(projectionName);
        
        log.info("Starting projection rebuild for: {}", projectionName);
        
        // 프로젝션 초기화
        projection.reset();
        
        // 모든 이벤트 재생
        Instant from = Instant.EPOCH;
        Instant to = Instant.now();
        
        List<DomainEvent> events = eventStore.getAllEvents(
            projection.getAggregateType(), from, to);
        
        events.forEach(event -> {
            try {
                projection.handle(event);
            } catch (Exception e) {
                log.error("Failed to apply event {} to projection {}", 
                         event, projectionName, e);
                throw new ProjectionRebuildException(
                    "Projection rebuild failed", e);
            }
        });
        
        log.info("Projection rebuild completed for: {}", projectionName);
    }
    
    public void rebuildAllProjections() {
        projections.parallelStream()
            .forEach(projection -> rebuildProjection(projection.getName()));
    }
    
    private EventProjection findProjection(String name) {
        return projections.stream()
            .filter(p -> p.getName().equals(name))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "Projection not found: " + name));
    }
}
```

## 6. 스냅샷 생성 전략

### 스냅샷 정책
```java
@Component
public class SnapshotPolicy {
    
    @Value("${eventsourcing.snapshot.threshold:20}")
    private int snapshotThreshold;
    
    @Value("${eventsourcing.snapshot.interval:PT1H}")
    private Duration snapshotInterval;
    
    public boolean shouldCreateSnapshot(String aggregateId, Long currentVersion) {
        return currentVersion % snapshotThreshold == 0;
    }
    
    public boolean shouldCreatePeriodicSnapshot(Instant lastSnapshotTime) {
        return lastSnapshotTime.plus(snapshotInterval).isBefore(Instant.now());
    }
}

@Service
public class SnapshotService {
    
    private final EventStore eventStore;
    private final SnapshotPolicy snapshotPolicy;
    
    @Async
    public CompletableFuture<Void> createSnapshotIfNeeded(String aggregateId, 
                                                         String aggregateType,
                                                         Long version) {
        if (snapshotPolicy.shouldCreateSnapshot(aggregateId, version)) {
            return createSnapshot(aggregateId, aggregateType);
        }
        
        return CompletableFuture.completedFuture(null);
    }
    
    @Async
    public CompletableFuture<Void> createSnapshot(String aggregateId, 
                                                String aggregateType) {
        try {
            // 애그리게이트 재구성
            Class<?> aggregateClass = Class.forName(aggregateType);
            Object aggregate = reconstructAggregate(aggregateId, aggregateClass);
            
            // 스냅샷 저장
            if (aggregate instanceof AggregateRoot aggregateRoot) {
                eventStore.saveSnapshot(
                    aggregateId,
                    aggregateType,
                    aggregate,
                    aggregateRoot.getVersion()
                );
                
                log.info("Snapshot created for aggregate: {} version: {}", 
                        aggregateId, aggregateRoot.getVersion());
            }
            
        } catch (Exception e) {
            log.error("Failed to create snapshot for aggregate: {}", aggregateId, e);
            throw new SnapshotCreationException("Snapshot creation failed", e);
        }
        
        return CompletableFuture.completedFuture(null);
    }
    
    private Object reconstructAggregate(String aggregateId, Class<?> aggregateClass) 
            throws Exception {
        
        Object aggregate = aggregateClass.getDeclaredConstructor().newInstance();
        List<DomainEvent> events = eventStore.getEvents(aggregateId);
        
        if (aggregate instanceof AggregateRoot aggregateRoot) {
            aggregateRoot.loadFromHistory(events);
        }
        
        return aggregate;
    }
}
```

## 7. 이벤트 버전 관리

### 이벤트 업캐스팅
```java
public interface EventUpgrader<T extends DomainEvent> {
    Class<T> getEventType();
    int getFromVersion();
    int getToVersion();
    DomainEvent upgrade(T oldEvent);
}

@Component
public class OrderCreatedEventUpgrader implements EventUpgrader<OrderCreatedEventV1> {
    
    @Override
    public Class<OrderCreatedEventV1> getEventType() {
        return OrderCreatedEventV1.class;
    }
    
    @Override
    public int getFromVersion() {
        return 1;
    }
    
    @Override
    public int getToVersion() {
        return 2;
    }
    
    @Override
    public DomainEvent upgrade(OrderCreatedEventV1 oldEvent) {
        // V1에서 V2로 업그레이드
        OrderCreatedEventV2 newEvent = new OrderCreatedEventV2();
        newEvent.setAggregateId(oldEvent.getAggregateId());
        newEvent.setCustomerId(oldEvent.getCustomerId());
        newEvent.setItems(oldEvent.getItems());
        newEvent.setTotalAmount(oldEvent.getTotalAmount());
        
        // V2에 추가된 필드 설정
        newEvent.setCurrency("USD"); // 기본값 설정
        newEvent.setOrderSource("WEB"); // 기본값 설정
        
        return newEvent;
    }
}

@Service
public class EventUpgradeService {
    
    private final Map<Class<? extends DomainEvent>, EventUpgrader<?>> upgraders;
    
    public EventUpgradeService(List<EventUpgrader<?>> upgraderList) {
        this.upgraders = upgraderList.stream()
            .collect(Collectors.toMap(
                EventUpgrader::getEventType,
                Function.identity()
            ));
    }
    
    @SuppressWarnings("unchecked")
    public DomainEvent upgradeEvent(DomainEvent event) {
        EventUpgrader<DomainEvent> upgrader = 
            (EventUpgrader<DomainEvent>) upgraders.get(event.getClass());
        
        if (upgrader != null) {
            return upgrader.upgrade(event);
        }
        
        return event;
    }
    
    public List<DomainEvent> upgradeEvents(List<DomainEvent> events) {
        return events.stream()
            .map(this::upgradeEvent)
            .toList();
    }
}
```

## 실습 과제

1. **완전한 이벤트 소싱 시스템**: 주문 도메인을 위한 전체 이벤트 소싱 구현
2. **스냅샷 최적화**: 다양한 스냅샷 전략 구현 및 성능 테스트
3. **이벤트 재생**: 특정 시점으로 애그리게이트 상태 복원 기능 구현
4. **이벤트 업그레이드**: 이벤트 스키마 진화에 대응하는 업캐스팅 시스템 구현
5. **성능 측정**: 이벤트 소싱 vs 전통적 CRUD 성능 비교 분석

## 참고 자료

- [Event Sourcing Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Building Event-Driven Microservices](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/)
- [Axon Framework Documentation](https://docs.axoniq.io/reference-guide/)
- [EventStore Documentation](https://developers.eventstore.com/)