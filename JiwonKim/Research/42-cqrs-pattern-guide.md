# CQRS 패턴 구현 가이드

## 1. Command/Query 모델 분리

### Command 모델 설계
```java
// Command 인터페이스
public interface Command {
    String getCommandId();
    String getAggregateId();
    Instant getTimestamp();
}

// 주문 관련 Command들
public class CreateOrderCommand implements Command {
    
    private final String commandId;
    private final String orderId;
    private final String customerId;
    private final List<OrderItem> items;
    private final Instant timestamp;
    
    public CreateOrderCommand(String orderId, String customerId, List<OrderItem> items) {
        this.commandId = UUID.randomUUID().toString();
        this.orderId = orderId;
        this.customerId = customerId;
        this.items = items;
        this.timestamp = Instant.now();
    }
    
    @Override
    public String getAggregateId() {
        return orderId;
    }
    
    // getter 메서드들
}

public class UpdateOrderCommand implements Command {
    
    private final String commandId;
    private final String orderId;
    private final List<OrderItem> updatedItems;
    private final String reason;
    private final Instant timestamp;
    
    // 생성자, getter 메서드들
}

public class CancelOrderCommand implements Command {
    
    private final String commandId;
    private final String orderId;
    private final String reason;
    private final Instant timestamp;
    
    // 생성자, getter 메서드들
}
```

### Query 모델 설계
```java
// Query 인터페이스
public interface Query<T> {
    String getQueryId();
    Class<T> getResultType();
}

// 주문 관련 Query들
public class GetOrderQuery implements Query<OrderView> {
    
    private final String queryId;
    private final String orderId;
    
    public GetOrderQuery(String orderId) {
        this.queryId = UUID.randomUUID().toString();
        this.orderId = orderId;
    }
    
    @Override
    public Class<OrderView> getResultType() {
        return OrderView.class;
    }
    
    // getter 메서드들
}

public class GetCustomerOrdersQuery implements Query<List<OrderSummaryView>> {
    
    private final String queryId;
    private final String customerId;
    private final Instant from;
    private final Instant to;
    private final Pageable pageable;
    
    @Override
    public Class<List<OrderSummaryView>> getResultType() {
        return (Class<List<OrderSummaryView>>) (Class<?>) List.class;
    }
    
    // 생성자, getter 메서드들
}

public class SearchOrdersQuery implements Query<List<OrderView>> {
    
    private final String queryId;
    private final OrderSearchCriteria criteria;
    private final Pageable pageable;
    
    // 생성자, getter 메서드들
}
```

## 2. Command Handler 구현

### Command Handler 인터페이스
```java
public interface CommandHandler<T extends Command> {
    void handle(T command);
    Class<T> getCommandType();
}

// 주문 Command Handler
@Component
public class OrderCommandHandler implements 
    CommandHandler<CreateOrderCommand>,
    CommandHandler<UpdateOrderCommand>,
    CommandHandler<CancelOrderCommand> {
    
    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    
    public OrderCommandHandler(OrderRepository orderRepository,
                              InventoryService inventoryService,
                              PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
    }
    
    public void handle(CreateOrderCommand command) {
        // 재고 확인
        List<OrderItem> items = command.getItems();
        if (!inventoryService.isAvailable(items)) {
            throw new InsufficientInventoryException("Insufficient inventory for order");
        }
        
        // 주문 생성
        Order order = new Order(
            command.getOrderId(),
            command.getCustomerId(),
            items
        );
        
        // 재고 예약
        inventoryService.reserveItems(items);
        
        // 주문 저장 (이벤트 소싱)
        orderRepository.save(order);
    }
    
    public void handle(UpdateOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId());
        
        // 비즈니스 규칙 검증
        if (order.getStatus() != OrderStatus.PENDING) {
            throw new InvalidOrderStateException("Cannot update order in current state");
        }
        
        // 재고 조정
        inventoryService.adjustReservation(
            order.getItems(), 
            command.getUpdatedItems()
        );
        
        // 주문 업데이트
        order.updateItems(command.getUpdatedItems(), command.getReason());
        orderRepository.save(order);
    }
    
    public void handle(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId());
        
        // 주문 취소
        order.cancel(command.getReason());
        
        // 재고 해제
        inventoryService.releaseReservation(order.getItems());
        
        // 결제 취소 (필요한 경우)
        if (order.getPaymentStatus() == PaymentStatus.COMPLETED) {
            paymentService.refund(order.getId(), order.getTotalAmount());
        }
        
        orderRepository.save(order);
    }
    
    @Override
    public Class<CreateOrderCommand> getCommandType() {
        return CreateOrderCommand.class;
    }
}
```

### Command Bus 구현
```java
@Service
public class CommandBus {
    
    private final Map<Class<? extends Command>, CommandHandler<? extends Command>> handlers;
    
    public CommandBus(List<CommandHandler<?>> handlerList) {
        this.handlers = handlerList.stream()
            .collect(Collectors.toMap(
                CommandHandler::getCommandType,
                Function.identity()
            ));
    }
    
    @SuppressWarnings("unchecked")
    public <T extends Command> void send(T command) {
        CommandHandler<T> handler = (CommandHandler<T>) handlers.get(command.getClass());
        
        if (handler == null) {
            throw new CommandHandlerNotFoundException(
                "No handler found for command: " + command.getClass().getSimpleName()
            );
        }
        
        try {
            handler.handle(command);
        } catch (Exception e) {
            throw new CommandExecutionException(
                "Command execution failed: " + command.getCommandId(), e
            );
        }
    }
    
    @Async
    public <T extends Command> CompletableFuture<Void> sendAsync(T command) {
        return CompletableFuture.runAsync(() -> send(command));
    }
}
```

## 3. Query Handler 및 읽기 모델

### Read Model Views
```java
// 상세 조회용 View
@Entity
@Table(name = "order_view")
public class OrderView {
    
    @Id
    private String orderId;
    
    @Column(name = "customer_id")
    private String customerId;
    
    @Column(name = "customer_name")
    private String customerName;
    
    @Column(name = "customer_email")
    private String customerEmail;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "total_amount")
    private BigDecimal totalAmount;
    
    @Column(name = "item_count")
    private Integer itemCount;
    
    @Column(name = "created_at")
    private Instant createdAt;
    
    @Column(name = "updated_at")
    private Instant updatedAt;
    
    @OneToMany(mappedBy = "orderId", fetch = FetchType.LAZY)
    private List<OrderItemView> items;
    
    // 생성자, getter, setter
}

@Entity
@Table(name = "order_item_view")
public class OrderItemView {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_id")
    private String orderId;
    
    @Column(name = "product_id")
    private String productId;
    
    @Column(name = "product_name")
    private String productName;
    
    @Column(name = "product_category")
    private String productCategory;
    
    private Integer quantity;
    
    @Column(name = "unit_price")
    private BigDecimal unitPrice;
    
    @Column(name = "total_price")
    private BigDecimal totalPrice;
    
    // 생성자, getter, setter
}

// 리스트 조회용 Summary View
@Entity
@Table(name = "order_summary_view")
public class OrderSummaryView {
    
    @Id
    private String orderId;
    
    @Column(name = "customer_id")
    private String customerId;
    
    @Column(name = "customer_name")
    private String customerName;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "total_amount")
    private BigDecimal totalAmount;
    
    @Column(name = "item_count")
    private Integer itemCount;
    
    @Column(name = "created_at")
    private Instant createdAt;
    
    // 생성자, getter, setter
}
```

### Query Handler 구현
```java
public interface QueryHandler<Q extends Query<T>, T> {
    T handle(Q query);
    Class<Q> getQueryType();
}

@Component
public class OrderQueryHandler implements 
    QueryHandler<GetOrderQuery, OrderView>,
    QueryHandler<GetCustomerOrdersQuery, List<OrderSummaryView>>,
    QueryHandler<SearchOrdersQuery, List<OrderView>> {
    
    private final OrderViewRepository orderViewRepository;
    private final OrderSummaryViewRepository summaryViewRepository;
    
    public OrderQueryHandler(OrderViewRepository orderViewRepository,
                           OrderSummaryViewRepository summaryViewRepository) {
        this.orderViewRepository = orderViewRepository;
        this.summaryViewRepository = summaryViewRepository;
    }
    
    @Override
    public OrderView handle(GetOrderQuery query) {
        return orderViewRepository.findById(query.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(query.getOrderId()));
    }
    
    @Override
    public List<OrderSummaryView> handle(GetCustomerOrdersQuery query) {
        if (query.getFrom() != null && query.getTo() != null) {
            return summaryViewRepository.findByCustomerIdAndCreatedAtBetween(
                query.getCustomerId(),
                query.getFrom(),
                query.getTo(),
                query.getPageable()
            );
        } else {
            return summaryViewRepository.findByCustomerId(
                query.getCustomerId(),
                query.getPageable()
            );
        }
    }
    
    @Override
    public List<OrderView> handle(SearchOrdersQuery query) {
        OrderSearchCriteria criteria = query.getCriteria();
        
        // 동적 쿼리 빌드
        return orderViewRepository.findByCriteria(criteria, query.getPageable());
    }
    
    @Override
    public Class<GetOrderQuery> getQueryType() {
        return GetOrderQuery.class;
    }
}
```

### Query Bus 구현
```java
@Service
public class QueryBus {
    
    private final Map<Class<? extends Query<?>>, QueryHandler<?, ?>> handlers;
    
    public QueryBus(List<QueryHandler<?, ?>> handlerList) {
        this.handlers = handlerList.stream()
            .collect(Collectors.toMap(
                QueryHandler::getQueryType,
                Function.identity()
            ));
    }
    
    @SuppressWarnings("unchecked")
    public <Q extends Query<T>, T> T send(Q query) {
        QueryHandler<Q, T> handler = (QueryHandler<Q, T>) handlers.get(query.getClass());
        
        if (handler == null) {
            throw new QueryHandlerNotFoundException(
                "No handler found for query: " + query.getClass().getSimpleName()
            );
        }
        
        try {
            return handler.handle(query);
        } catch (Exception e) {
            throw new QueryExecutionException(
                "Query execution failed: " + query.getQueryId(), e
            );
        }
    }
    
    @Async
    public <Q extends Query<T>, T> CompletableFuture<T> sendAsync(Q query) {
        return CompletableFuture.supplyAsync(() -> send(query));
    }
}
```

## 4. 읽기 전용 프로젝션 구성

### Event Projection 인터페이스
```java
public interface EventProjection {
    String getName();
    String getAggregateType();
    void handle(DomainEvent event);
    void reset();
}

// 주문 View 프로젝션
@Component
@EventListener
public class OrderViewProjection implements EventProjection {
    
    private final OrderViewRepository orderViewRepository;
    private final OrderItemViewRepository itemViewRepository;
    private final CustomerService customerService;
    private final ProductService productService;
    
    public OrderViewProjection(OrderViewRepository orderViewRepository,
                             OrderItemViewRepository itemViewRepository,
                             CustomerService customerService,
                             ProductService productService) {
        this.orderViewRepository = orderViewRepository;
        this.itemViewRepository = itemViewRepository;
        this.customerService = customerService;
        this.productService = productService;
    }
    
    @Override
    public String getName() {
        return "OrderViewProjection";
    }
    
    @Override
    public String getAggregateType() {
        return "Order";
    }
    
    @Override
    public void handle(DomainEvent event) {
        switch (event) {
            case OrderCreatedEvent e -> handleOrderCreated(e);
            case OrderUpdatedEvent e -> handleOrderUpdated(e);
            case OrderCancelledEvent e -> handleOrderCancelled(e);
            case OrderCompletedEvent e -> handleOrderCompleted(e);
            default -> log.debug("Unhandled event type: {}", event.getClass().getSimpleName());
        }
    }
    
    private void handleOrderCreated(OrderCreatedEvent event) {
        // 고객 정보 조회
        Customer customer = customerService.getCustomer(event.getCustomerId());
        
        // OrderView 생성
        OrderView orderView = new OrderView();
        orderView.setOrderId(event.getAggregateId());
        orderView.setCustomerId(event.getCustomerId());
        orderView.setCustomerName(customer.getName());
        orderView.setCustomerEmail(customer.getEmail());
        orderView.setStatus(OrderStatus.PENDING);
        orderView.setTotalAmount(event.getTotalAmount());
        orderView.setItemCount(event.getItems().size());
        orderView.setCreatedAt(event.getOccurredAt());
        orderView.setUpdatedAt(event.getOccurredAt());
        
        orderViewRepository.save(orderView);
        
        // OrderItemView 생성
        List<OrderItemView> itemViews = event.getItems().stream()
            .map(item -> createOrderItemView(event.getAggregateId(), item))
            .toList();
        
        itemViewRepository.saveAll(itemViews);
    }
    
    private void handleOrderUpdated(OrderUpdatedEvent event) {
        OrderView orderView = orderViewRepository.findById(event.getAggregateId())
            .orElseThrow(() -> new IllegalStateException("OrderView not found"));
        
        // 기존 아이템 삭제
        itemViewRepository.deleteByOrderId(event.getAggregateId());
        
        // 새로운 아이템 생성
        List<OrderItemView> itemViews = event.getUpdatedItems().stream()
            .map(item -> createOrderItemView(event.getAggregateId(), item))
            .toList();
        
        itemViewRepository.saveAll(itemViews);
        
        // OrderView 업데이트
        orderView.setTotalAmount(event.getNewTotalAmount());
        orderView.setItemCount(event.getUpdatedItems().size());
        orderView.setUpdatedAt(event.getOccurredAt());
        
        orderViewRepository.save(orderView);
    }
    
    private void handleOrderCancelled(OrderCancelledEvent event) {
        OrderView orderView = orderViewRepository.findById(event.getAggregateId())
            .orElseThrow(() -> new IllegalStateException("OrderView not found"));
        
        orderView.setStatus(OrderStatus.CANCELLED);
        orderView.setUpdatedAt(event.getOccurredAt());
        
        orderViewRepository.save(orderView);
    }
    
    private void handleOrderCompleted(OrderCompletedEvent event) {
        OrderView orderView = orderViewRepository.findById(event.getAggregateId())
            .orElseThrow(() -> new IllegalStateException("OrderView not found"));
        
        orderView.setStatus(OrderStatus.COMPLETED);
        orderView.setUpdatedAt(event.getOccurredAt());
        
        orderViewRepository.save(orderView);
    }
    
    private OrderItemView createOrderItemView(String orderId, OrderItem item) {
        Product product = productService.getProduct(item.getProductId());
        
        OrderItemView itemView = new OrderItemView();
        itemView.setOrderId(orderId);
        itemView.setProductId(item.getProductId());
        itemView.setProductName(product.getName());
        itemView.setProductCategory(product.getCategory());
        itemView.setQuantity(item.getQuantity());
        itemView.setUnitPrice(item.getPrice());
        itemView.setTotalPrice(item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())));
        
        return itemView;
    }
    
    @Override
    public void reset() {
        itemViewRepository.deleteAll();
        orderViewRepository.deleteAll();
    }
}
```

### Summary Projection
```java
@Component
@EventListener
public class OrderSummaryProjection implements EventProjection {
    
    private final OrderSummaryViewRepository summaryRepository;
    private final CustomerService customerService;
    
    @Override
    public void handle(DomainEvent event) {
        switch (event) {
            case OrderCreatedEvent e -> handleOrderCreated(e);
            case OrderUpdatedEvent e -> handleOrderUpdated(e);
            case OrderCancelledEvent e -> handleOrderCancelled(e);
            case OrderCompletedEvent e -> handleOrderCompleted(e);
            default -> log.debug("Unhandled event type: {}", event.getClass().getSimpleName());
        }
    }
    
    private void handleOrderCreated(OrderCreatedEvent event) {
        Customer customer = customerService.getCustomer(event.getCustomerId());
        
        OrderSummaryView summary = new OrderSummaryView();
        summary.setOrderId(event.getAggregateId());
        summary.setCustomerId(event.getCustomerId());
        summary.setCustomerName(customer.getName());
        summary.setStatus(OrderStatus.PENDING);
        summary.setTotalAmount(event.getTotalAmount());
        summary.setItemCount(event.getItems().size());
        summary.setCreatedAt(event.getOccurredAt());
        
        summaryRepository.save(summary);
    }
    
    private void handleOrderUpdated(OrderUpdatedEvent event) {
        OrderSummaryView summary = summaryRepository.findById(event.getAggregateId())
            .orElseThrow(() -> new IllegalStateException("OrderSummaryView not found"));
        
        summary.setTotalAmount(event.getNewTotalAmount());
        summary.setItemCount(event.getUpdatedItems().size());
        
        summaryRepository.save(summary);
    }
    
    private void handleOrderCancelled(OrderCancelledEvent event) {
        OrderSummaryView summary = summaryRepository.findById(event.getAggregateId())
            .orElseThrow(() -> new IllegalStateException("OrderSummaryView not found"));
        
        summary.setStatus(OrderStatus.CANCELLED);
        summaryRepository.save(summary);
    }
    
    private void handleOrderCompleted(OrderCompletedEvent event) {
        OrderSummaryView summary = summaryRepository.findById(event.getAggregateId())
            .orElseThrow(() -> new IllegalStateException("OrderSummaryView not found"));
        
        summary.setStatus(OrderStatus.COMPLETED);
        summaryRepository.save(summary);
    }
    
    @Override
    public void reset() {
        summaryRepository.deleteAll();
    }
    
    @Override
    public String getName() {
        return "OrderSummaryProjection";
    }
    
    @Override
    public String getAggregateType() {
        return "Order";
    }
}
```

## 5. 데이터 동기화 전략

### Eventual Consistency 처리
```java
@Component
public class ProjectionSynchronizer {
    
    private final EventStore eventStore;
    private final List<EventProjection> projections;
    private final ProjectionStateRepository stateRepository;
    
    @Scheduled(fixedDelay = 30000) // 30초마다 실행
    public void synchronizeProjections() {
        projections.parallelStream()
            .forEach(this::synchronizeProjection);
    }
    
    private void synchronizeProjection(EventProjection projection) {
        try {
            ProjectionState state = stateRepository
                .findByProjectionName(projection.getName())
                .orElse(new ProjectionState(projection.getName(), Instant.EPOCH));
            
            Instant lastProcessedTime = state.getLastProcessedTime();
            List<DomainEvent> newEvents = eventStore.getAllEvents(
                projection.getAggregateType(),
                lastProcessedTime,
                Instant.now()
            );
            
            if (!newEvents.isEmpty()) {
                newEvents.forEach(projection::handle);
                
                state.setLastProcessedTime(Instant.now());
                state.setLastEventCount(newEvents.size());
                stateRepository.save(state);
                
                log.info("Synchronized {} events for projection: {}", 
                        newEvents.size(), projection.getName());
            }
            
        } catch (Exception e) {
            log.error("Failed to synchronize projection: {}", 
                     projection.getName(), e);
        }
    }
}

@Entity
@Table(name = "projection_state")
public class ProjectionState {
    
    @Id
    private String projectionName;
    
    @Column(name = "last_processed_time")
    private Instant lastProcessedTime;
    
    @Column(name = "last_event_count")
    private Integer lastEventCount;
    
    @Column(name = "updated_at")
    @LastModifiedDate
    private Instant updatedAt;
    
    // 생성자, getter, setter
}
```

### Saga Pattern과 CQRS 통합
```java
@Component
public class OrderProcessingSaga {
    
    private final CommandBus commandBus;
    private final QueryBus queryBus;
    
    @EventListener
    @Async
    public void handle(OrderCreatedEvent event) {
        try {
            // 재고 확인 Query
            CheckInventoryQuery inventoryQuery = new CheckInventoryQuery(event.getItems());
            InventoryStatus inventoryStatus = queryBus.send(inventoryQuery);
            
            if (inventoryStatus.isAvailable()) {
                // 재고 예약 Command
                ReserveInventoryCommand reserveCommand = new ReserveInventoryCommand(
                    event.getAggregateId(), 
                    event.getItems()
                );
                commandBus.send(reserveCommand);
                
                // 결제 처리 Command
                ProcessPaymentCommand paymentCommand = new ProcessPaymentCommand(
                    event.getAggregateId(),
                    event.getTotalAmount(),
                    event.getCustomerId()
                );
                commandBus.send(paymentCommand);
                
            } else {
                // 주문 취소 Command
                CancelOrderCommand cancelCommand = new CancelOrderCommand(
                    event.getAggregateId(),
                    "Insufficient inventory"
                );
                commandBus.send(cancelCommand);
            }
            
        } catch (Exception e) {
            log.error("Order processing saga failed for order: {}", 
                     event.getAggregateId(), e);
            
            // 보상 트랜잭션 실행
            executeCompensation(event.getAggregateId());
        }
    }
    
    @EventListener
    @Async
    public void handle(PaymentCompletedEvent event) {
        // 주문 완료 Command
        CompleteOrderCommand completeCommand = new CompleteOrderCommand(event.getOrderId());
        commandBus.send(completeCommand);
        
        // 배송 시작 Command
        StartShippingCommand shippingCommand = new StartShippingCommand(
            event.getOrderId(),
            event.getCustomerId()
        );
        commandBus.send(shippingCommand);
    }
    
    @EventListener
    @Async
    public void handle(PaymentFailedEvent event) {
        // 재고 해제 Command
        ReleaseInventoryCommand releaseCommand = new ReleaseInventoryCommand(
            event.getOrderId()
        );
        commandBus.send(releaseCommand);
        
        // 주문 취소 Command
        CancelOrderCommand cancelCommand = new CancelOrderCommand(
            event.getOrderId(),
            "Payment failed: " + event.getReason()
        );
        commandBus.send(cancelCommand);
    }
    
    private void executeCompensation(String orderId) {
        // 보상 트랜잭션 로직
        ReleaseInventoryCommand releaseCommand = new ReleaseInventoryCommand(orderId);
        commandBus.send(releaseCommand);
        
        CancelOrderCommand cancelCommand = new CancelOrderCommand(
            orderId, 
            "System error - order cancelled"
        );
        commandBus.send(cancelCommand);
    }
}
```

## 6. 성능 최적화

### Read Model 캐싱
```java
@Service
public class CachedOrderQueryService {
    
    private final QueryBus queryBus;
    private final RedisTemplate<String, Object> redisTemplate;
    
    @Cacheable(value = "orders", key = "#orderId")
    public OrderView getOrder(String orderId) {
        GetOrderQuery query = new GetOrderQuery(orderId);
        return queryBus.send(query);
    }
    
    @Cacheable(value = "customer-orders", key = "#customerId + ':' + #pageable.pageNumber")
    public List<OrderSummaryView> getCustomerOrders(String customerId, Pageable pageable) {
        GetCustomerOrdersQuery query = new GetCustomerOrdersQuery(customerId, pageable);
        return queryBus.send(query);
    }
    
    @CacheEvict(value = {"orders", "customer-orders"}, key = "#orderId")
    public void evictOrderCache(String orderId) {
        // 캐시 무효화
    }
    
    @EventListener
    public void handleOrderUpdated(OrderUpdatedEvent event) {
        evictOrderCache(event.getAggregateId());
        
        // 고객별 주문 목록 캐시도 무효화
        String customerCachePattern = "customer-orders::" + event.getCustomerId() + ":*";
        redisTemplate.delete(redisTemplate.keys(customerCachePattern));
    }
}
```

### 비동기 프로젝션 업데이트
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "projectionExecutor")
    public Executor projectionExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("projection-");
        executor.initialize();
        return executor;
    }
}

@Component
public class AsyncProjectionHandler {
    
    @EventListener
    @Async("projectionExecutor")
    public void handleOrderEvent(OrderEvent event) {
        // 비동기적으로 프로젝션 업데이트
        updateProjections(event);
    }
    
    private void updateProjections(OrderEvent event) {
        try {
            // 여러 프로젝션 병렬 업데이트
            CompletableFuture.allOf(
                CompletableFuture.runAsync(() -> updateOrderView(event)),
                CompletableFuture.runAsync(() -> updateOrderSummary(event)),
                CompletableFuture.runAsync(() -> updateAnalytics(event))
            ).join();
            
        } catch (Exception e) {
            log.error("Failed to update projections for event: {}", event, e);
        }
    }
}
```

## 실습 과제

1. **완전한 CQRS 시스템**: 주문 도메인을 위한 전체 CQRS 구현
2. **성능 비교**: CQRS vs 전통적 CRUD 성능 벤치마크
3. **복잡한 쿼리 최적화**: 다양한 검색 조건을 가진 복잡한 쿼리 최적화
4. **실시간 대시보드**: WebSocket을 활용한 실시간 주문 현황 대시보드
5. **멀티 테넌트 CQRS**: 테넌트별로 분리된 읽기 모델 구현

## 참고 자료

- [CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [CQRS Journey](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10))
- [Axon Framework CQRS](https://docs.axoniq.io/reference-guide/axon-framework/architecture-overview/command-handling)
- [Event Sourcing and CQRS](https://www.eventstore.com/blog/what-is-event-sourcing)