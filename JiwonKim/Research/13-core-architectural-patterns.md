# 1-3. 핵심 아키텍처 패턴

## 개요

이벤트 기반 마이크로서비스 아키텍처에서 핵심적인 세 가지 패턴인 이벤트 소싱(Event Sourcing), CQRS(Command Query Responsibility Segregation), 사가 패턴(Saga Pattern)에 대해 심도 있게 다룹니다.

## 1. 이벤트 소싱 (Event Sourcing)

### 개념적 기초

이벤트 소싱은 도메인 객체를 현재 상태로 저장하는 대신 이벤트의 시퀀스로 저장하는 패턴입니다. 현재 상태는 처음부터 이벤트를 재생하여 파생됩니다.

### 전통적 상태 저장 vs 이벤트 소싱

**전통적 방식**:
```sql
-- 계좌 테이블
CREATE TABLE accounts (
    id UUID PRIMARY KEY,
    balance DECIMAL(15,2),
    last_updated TIMESTAMP
);

-- 현재 상태만 저장
INSERT INTO accounts VALUES ('account-123', 1000.00, NOW());
UPDATE accounts SET balance = 1500.00 WHERE id = 'account-123';
```

**이벤트 소싱 방식**:
```sql
-- 이벤트 스토어
CREATE TABLE event_store (
    stream_id VARCHAR(255),
    version BIGINT,
    event_type VARCHAR(255),
    event_data JSONB,
    metadata JSONB,
    timestamp TIMESTAMP,
    PRIMARY KEY (stream_id, version)
);

-- 모든 변화를 이벤트로 저장
INSERT INTO event_store VALUES 
('account-123', 1, 'AccountOpened', '{"initialBalance": 1000.00}', '{}', NOW()),
('account-123', 2, 'DepositMade', '{"amount": 500.00}', '{}', NOW());
```

### 이벤트 스토어 설계

**스트림 기반 저장**:
```java
public class EventStore {
    
    public void appendEvents(String streamId, long expectedVersion, List<DomainEvent> events) {
        // 낙관적 동시성 제어
        long currentVersion = getCurrentVersion(streamId);
        if (currentVersion != expectedVersion) {
            throw new ConcurrencyException("Expected version " + expectedVersion + 
                " but current version is " + currentVersion);
        }
        
        // 이벤트 저장
        for (DomainEvent event : events) {
            EventData eventData = EventData.builder()
                .streamId(streamId)
                .version(++currentVersion)
                .eventType(event.getClass().getSimpleName())
                .eventData(serialize(event))
                .metadata(createMetadata(event))
                .timestamp(Instant.now())
                .build();
                
            eventRepository.save(eventData);
        }
    }
    
    public List<DomainEvent> getEvents(String streamId, long fromVersion) {
        return eventRepository.findByStreamIdAndVersionGreaterThan(streamId, fromVersion)
            .stream()
            .map(this::deserialize)
            .collect(Collectors.toList());
    }
}
```

### 스냅샷 전략

이벤트가 많아질 경우 성능 최적화를 위해 스냅샷을 사용합니다.

```java
public class SnapshotStrategy {
    
    private static final int SNAPSHOT_FREQUENCY = 100;
    
    public <T extends AggregateRoot> T loadAggregate(String streamId, Class<T> aggregateType) {
        // 최신 스냅샷 조회
        Optional<Snapshot> snapshot = snapshotRepository.findLatestSnapshot(streamId);
        
        long fromVersion = 0;
        T aggregate = null;
        
        if (snapshot.isPresent()) {
            aggregate = deserializeSnapshot(snapshot.get(), aggregateType);
            fromVersion = snapshot.get().getVersion();
        } else {
            aggregate = createNewAggregate(aggregateType);
        }
        
        // 스냅샷 이후 이벤트들을 재생
        List<DomainEvent> events = eventStore.getEvents(streamId, fromVersion);
        aggregate.replay(events);
        
        return aggregate;
    }
    
    public void saveAggregate(AggregateRoot aggregate) {
        List<DomainEvent> uncommittedEvents = aggregate.getUncommittedEvents();
        eventStore.appendEvents(
            aggregate.getId(), 
            aggregate.getVersion(), 
            uncommittedEvents
        );
        
        // 스냅샷 저장 조건 확인
        if (shouldCreateSnapshot(aggregate)) {
            Snapshot snapshot = Snapshot.builder()
                .streamId(aggregate.getId())
                .version(aggregate.getVersion())
                .data(serialize(aggregate))
                .timestamp(Instant.now())
                .build();
            snapshotRepository.save(snapshot);
        }
        
        aggregate.markEventsAsCommitted();
    }
    
    private boolean shouldCreateSnapshot(AggregateRoot aggregate) {
        return aggregate.getVersion() % SNAPSHOT_FREQUENCY == 0;
    }
}
```

### 이벤트 스키마 진화

**이벤트 업캐스팅**:
```java
public class EventUpcaster {
    
    public DomainEvent upcast(EventData eventData) {
        String eventType = eventData.getEventType();
        int version = extractVersion(eventData);
        
        switch (eventType) {
            case "CustomerRegisteredV1":
                if (version == 1) {
                    return upcastToV2(eventData);
                }
                break;
            case "CustomerRegisteredV2":
                return deserializeV2(eventData);
        }
        
        throw new UnsupportedEventVersionException(eventType, version);
    }
    
    private CustomerRegisteredV2 upcastToV2(EventData eventData) {
        CustomerRegisteredV1 v1Event = deserializeV1(eventData);
        
        return CustomerRegisteredV2.builder()
            .customerId(v1Event.getCustomerId())
            .firstName(v1Event.getName()) // 이름을 성/이름으로 분리
            .lastName("") // 기본값
            .email(v1Event.getEmail())
            .registrationDate(v1Event.getRegistrationDate())
            .preferences(new CustomerPreferences()) // 새 필드
            .build();
    }
}
```

### 프로젝션과 읽기 모델

```java
@EventHandler
public class CustomerProjectionHandler {
    
    @Autowired
    private CustomerReadModelRepository readModelRepository;
    
    @EventHandler
    public void handle(CustomerRegistered event) {
        CustomerReadModel readModel = CustomerReadModel.builder()
            .id(event.getCustomerId())
            .fullName(event.getFirstName() + " " + event.getLastName())
            .email(event.getEmail())
            .registrationDate(event.getRegistrationDate())
            .status("ACTIVE")
            .build();
            
        readModelRepository.save(readModel);
    }
    
    @EventHandler
    public void handle(CustomerDeactivated event) {
        CustomerReadModel readModel = readModelRepository.findById(event.getCustomerId());
        readModel.setStatus("INACTIVE");
        readModel.setDeactivationDate(event.getDeactivationDate());
        readModelRepository.save(readModel);
    }
}
```

### 장점과 단점

**장점**:
- 완전한 감사 추적 및 시간적 쿼리
- 이벤트 기반 아키텍처와의 자연스러운 통합
- 복잡한 비즈니스 로직 재생 지원
- 디버깅을 위한 이벤트 재생 기능

**단점**:
- 저장 공간 요구량 증가
- 현재 상태 쿼리의 복잡성
- 이벤트 스키마 진화의 복잡성
- 스냅샷 관리 오버헤드

## 2. CQRS (Command Query Responsibility Segregation)

### 아키텍처 분리

CQRS는 읽기와 쓰기 모델을 분리하여 명령과 쿼리 경로를 독립적으로 최적화할 수 있게 합니다.

### 구현 패턴

**단순 CQRS**: 공유 데이터베이스를 사용하되 명령과 쿼리 핸들러를 분리
```java
// Command Handler
@Component
public class CreateOrderCommandHandler {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    public void handle(CreateOrderCommand command) {
        Order order = Order.builder()
            .customerId(command.getCustomerId())
            .items(command.getItems())
            .totalAmount(calculateTotal(command.getItems()))
            .status(OrderStatus.PENDING)
            .build();
            
        orderRepository.save(order);
        
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount()
        );
        eventPublisher.publish(event);
    }
}

// Query Handler
@Component
public class OrderQueryHandler {
    
    @Autowired
    private OrderReadModelRepository readModelRepository;
    
    public List<OrderSummary> getOrdersByCustomer(String customerId) {
        return readModelRepository.findByCustomerId(customerId);
    }
    
    public OrderDetails getOrderDetails(String orderId) {
        return readModelRepository.findOrderDetailsById(orderId);
    }
}
```

**이벤트 소싱과 함께하는 CQRS**: 명령은 이벤트를 생성하고 쿼리는 프로젝션을 사용
```java
@Component
public class OrderProjectionUpdater {
    
    @EventHandler
    public void handle(OrderCreatedEvent event) {
        OrderSummary summary = OrderSummary.builder()
            .orderId(event.getOrderId())
            .customerId(event.getCustomerId())
            .totalAmount(event.getTotalAmount())
            .status("PENDING")
            .createdDate(event.getTimestamp())
            .build();
            
        orderSummaryRepository.save(summary);
    }
    
    @EventHandler
    public void handle(OrderShippedEvent event) {
        OrderSummary summary = orderSummaryRepository.findById(event.getOrderId());
        summary.setStatus("SHIPPED");
        summary.setShippedDate(event.getTimestamp());
        orderSummaryRepository.save(summary);
    }
}
```

**분리된 저장소를 가진 CQRS**: 읽기와 쓰기 전용 데이터베이스 사용
```java
@Configuration
public class DatabaseConfiguration {
    
    @Bean
    @Primary
    @ConfigurationProperties("app.datasource.write")
    public DataSource writeDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("app.datasource.read")
    public DataSource readDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public JdbcTemplate writeJdbcTemplate(@Qualifier("writeDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
    
    @Bean
    public JdbcTemplate readJdbcTemplate(@Qualifier("readDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### 동기화 전략

**동기적 동기화**: 트랜잭션 경계 내에서 즉시 일관성 보장
```java
@Transactional
public void handle(CreateOrderCommand command) {
    // 1. 명령 처리
    Order order = processOrder(command);
    orderRepository.save(order);
    
    // 2. 읽기 모델 즉시 업데이트
    OrderSummary summary = createOrderSummary(order);
    orderSummaryRepository.save(summary);
    
    // 3. 이벤트 발행
    eventPublisher.publish(new OrderCreatedEvent(order));
}
```

**비동기적 동기화**: 이벤트 기반 프로젝션 업데이트
```java
@EventHandler
@Async
public void handle(OrderCreatedEvent event) {
    // 비동기적으로 읽기 모델 업데이트
    OrderSummary summary = createOrderSummary(event);
    orderSummaryRepository.save(summary);
}
```

### 이점과 고려사항

**이점**:
- 읽기와 쓰기 워크로드의 독립적 확장
- 특정 사용 사례에 최적화된 데이터 모델
- 명확한 관심사 분리
- 여러 쿼리 패턴 지원

**고려사항**:
- 아키텍처 복잡성 증가
- 데이터 동기화 도전과제
- 잠재적 일관성 문제
- 추가적인 인프라스트럭처 요구사항

## 3. 사가 패턴 (Saga Pattern)

### 분산 트랜잭션 관리

사가는 분산 트랜잭션 없이 여러 서비스에 걸친 장시간 실행되는 비즈니스 프로세스를 관리하며, 보상을 통해 최종적 일관성을 보장합니다.

### 오케스트레이션 vs 코레오그래피

**오케스트레이션 패턴**: 중앙 조정자가 사가 실행을 관리
```java
@Component
public class OrderProcessingSaga {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private ShippingService shippingService;
    
    @SagaOrchestrationStart
    public void startOrderProcessing(OrderCreatedEvent event) {
        SagaInstance saga = SagaInstance.builder()
            .sagaId(UUID.randomUUID().toString())
            .orderId(event.getOrderId())
            .customerId(event.getCustomerId())
            .totalAmount(event.getTotalAmount())
            .currentStep("PAYMENT_PROCESSING")
            .build();
            
        sagaRepository.save(saga);
        processPayment(saga);
    }
    
    @SagaStep(compensatingAction = "cancelPayment")
    public void processPayment(SagaInstance saga) {
        try {
            PaymentRequest request = PaymentRequest.builder()
                .orderId(saga.getOrderId())
                .customerId(saga.getCustomerId())
                .amount(saga.getTotalAmount())
                .build();
                
            PaymentResult result = paymentService.processPayment(request);
            
            if (result.isSuccess()) {
                saga.setCurrentStep("INVENTORY_RESERVATION");
                saga.setPaymentId(result.getPaymentId());
                sagaRepository.save(saga);
                reserveInventory(saga);
            } else {
                handlePaymentFailure(saga, result);
            }
        } catch (Exception e) {
            handleSagaFailure(saga, "PAYMENT_PROCESSING", e);
        }
    }
    
    @SagaStep(compensatingAction = "releaseInventory")
    public void reserveInventory(SagaInstance saga) {
        try {
            InventoryReservationRequest request = InventoryReservationRequest.builder()
                .orderId(saga.getOrderId())
                .items(getOrderItems(saga.getOrderId()))
                .build();
                
            InventoryReservationResult result = inventoryService.reserveInventory(request);
            
            if (result.isSuccess()) {
                saga.setCurrentStep("SHIPPING_ARRANGEMENT");
                saga.setReservationId(result.getReservationId());
                sagaRepository.save(saga);
                arrangeShipping(saga);
            } else {
                handleInventoryFailure(saga, result);
            }
        } catch (Exception e) {
            handleSagaFailure(saga, "INVENTORY_RESERVATION", e);
        }
    }
    
    @SagaStep
    public void arrangeShipping(SagaInstance saga) {
        try {
            ShippingRequest request = ShippingRequest.builder()
                .orderId(saga.getOrderId())
                .customerId(saga.getCustomerId())
                .build();
                
            ShippingResult result = shippingService.arrangeShipping(request);
            
            if (result.isSuccess()) {
                saga.setCurrentStep("COMPLETED");
                saga.setStatus("SUCCESS");
                sagaRepository.save(saga);
                completeSaga(saga);
            } else {
                handleShippingFailure(saga, result);
            }
        } catch (Exception e) {
            handleSagaFailure(saga, "SHIPPING_ARRANGEMENT", e);
        }
    }
    
    // 보상 액션들
    public void cancelPayment(SagaInstance saga) {
        if (saga.getPaymentId() != null) {
            paymentService.cancelPayment(saga.getPaymentId());
        }
    }
    
    public void releaseInventory(SagaInstance saga) {
        if (saga.getReservationId() != null) {
            inventoryService.releaseReservation(saga.getReservationId());
        }
    }
    
    private void handleSagaFailure(SagaInstance saga, String failedStep, Exception e) {
        saga.setStatus("FAILED");
        saga.setFailedStep(failedStep);
        saga.setErrorMessage(e.getMessage());
        sagaRepository.save(saga);
        
        compensate(saga);
    }
}
```

**코레오그래피 패턴**: 분산된 조정으로 이벤트를 통한 조정
```java
// 주문 서비스
@EventHandler
public class OrderEventHandler {
    
    public void handle(OrderCreatedEvent event) {
        // 결제 처리 요청 이벤트 발행
        PaymentProcessingRequested paymentRequest = PaymentProcessingRequested.builder()
            .orderId(event.getOrderId())
            .customerId(event.getCustomerId())
            .amount(event.getTotalAmount())
            .correlationId(event.getOrderId())
            .build();
            
        eventPublisher.publish(paymentRequest);
    }
}

// 결제 서비스
@EventHandler
public class PaymentEventHandler {
    
    public void handle(PaymentProcessingRequested event) {
        PaymentResult result = processPayment(event);
        
        if (result.isSuccess()) {
            PaymentProcessedEvent successEvent = PaymentProcessedEvent.builder()
                .orderId(event.getOrderId())
                .paymentId(result.getPaymentId())
                .amount(event.getAmount())
                .correlationId(event.getCorrelationId())
                .build();
                
            eventPublisher.publish(successEvent);
        } else {
            PaymentFailedEvent failureEvent = PaymentFailedEvent.builder()
                .orderId(event.getOrderId())
                .reason(result.getFailureReason())
                .correlationId(event.getCorrelationId())
                .build();
                
            eventPublisher.publish(failureEvent);
        }
    }
}

// 재고 서비스
@EventHandler
public class InventoryEventHandler {
    
    public void handle(PaymentProcessedEvent event) {
        // 재고 예약 처리
        InventoryReservationResult result = reserveInventory(event.getOrderId());
        
        if (result.isSuccess()) {
            InventoryReservedEvent successEvent = InventoryReservedEvent.builder()
                .orderId(event.getOrderId())
                .reservationId(result.getReservationId())
                .correlationId(event.getCorrelationId())
                .build();
                
            eventPublisher.publish(successEvent);
        } else {
            // 재고 예약 실패 시 결제 취소 이벤트 발행
            PaymentCancellationRequested cancellationEvent = PaymentCancellationRequested.builder()
                .orderId(event.getOrderId())
                .paymentId(event.getPaymentId())
                .reason("Inventory reservation failed")
                .correlationId(event.getCorrelationId())
                .build();
                
            eventPublisher.publish(cancellationEvent);
        }
    }
}
```

### 보상 전략

**의미적 보상**: 비즈니스 수준의 롤백
```java
public class PaymentCompensationService {
    
    public void compensatePayment(String paymentId, String reason) {
        Payment payment = paymentRepository.findById(paymentId);
        
        if (payment.getStatus() == PaymentStatus.COMPLETED) {
            // 환불 처리
            Refund refund = Refund.builder()
                .paymentId(paymentId)
                .amount(payment.getAmount())
                .reason(reason)
                .refundDate(Instant.now())
                .build();
                
            refundRepository.save(refund);
            
            payment.setStatus(PaymentStatus.REFUNDED);
            paymentRepository.save(payment);
            
            // 환불 완료 이벤트 발행
            RefundProcessedEvent event = new RefundProcessedEvent(
                refund.getId(),
                paymentId,
                refund.getAmount()
            );
            eventPublisher.publish(event);
        }
    }
}
```

**구조적 보상**: 데이터 수준의 롤백
```java
public class InventoryCompensationService {
    
    public void compensateInventoryReservation(String reservationId) {
        InventoryReservation reservation = reservationRepository.findById(reservationId);
        
        if (reservation.getStatus() == ReservationStatus.RESERVED) {
            // 예약된 재고 해제
            for (ReservationItem item : reservation.getItems()) {
                Product product = productRepository.findById(item.getProductId());
                product.setAvailableQuantity(product.getAvailableQuantity() + item.getQuantity());
                productRepository.save(product);
            }
            
            reservation.setStatus(ReservationStatus.CANCELLED);
            reservationRepository.save(reservation);
            
            // 재고 해제 이벤트 발행
            InventoryReleasedEvent event = new InventoryReleasedEvent(
                reservationId,
                reservation.getOrderId()
            );
            eventPublisher.publish(event);
        }
    }
}
```

### 상태 관리와 복구

```java
@Entity
public class SagaInstance {
    @Id
    private String sagaId;
    private String orderId;
    private String customerId;
    private BigDecimal totalAmount;
    private String currentStep;
    private String status; // RUNNING, COMPLETED, FAILED, COMPENSATING
    private String paymentId;
    private String reservationId;
    private String shippingId;
    private Instant startTime;
    private Instant endTime;
    private String errorMessage;
    private int retryCount;
    private Instant nextRetryTime;
    
    // getters and setters
}

@Component
public class SagaRecoveryService {
    
    @Scheduled(fixedDelay = 60000) // 1분마다 실행
    public void recoverStuckSagas() {
        List<SagaInstance> stuckSagas = sagaRepository.findStuckSagas(
            Instant.now().minus(5, ChronoUnit.MINUTES)
        );
        
        for (SagaInstance saga : stuckSagas) {
            if (saga.getRetryCount() < MAX_RETRY_COUNT) {
                retrySaga(saga);
            } else {
                markSagaAsFailed(saga);
            }
        }
    }
    
    private void retrySaga(SagaInstance saga) {
        saga.setRetryCount(saga.getRetryCount() + 1);
        saga.setNextRetryTime(calculateNextRetryTime(saga.getRetryCount()));
        sagaRepository.save(saga);
        
        // 현재 단계부터 다시 실행
        sagaOrchestrator.continueFromStep(saga, saga.getCurrentStep());
    }
}
```

### 오류 처리 패턴

**순방향 복구**: 실패에도 불구하고 계속 진행
**역방향 복구**: 완료된 단계들을 보상
**혼합 모드**: 비즈니스 규칙에 따른 조합

이러한 핵심 패턴들은 이벤트 기반 마이크로서비스 아키텍처의 기반을 형성하며, 각각의 장단점과 사용 사례를 이해하는 것이 성공적인 구현의 핵심입니다.