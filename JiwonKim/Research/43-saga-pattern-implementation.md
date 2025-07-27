# 사가 패턴 구현 가이드

## 1. Orchestration vs Choreography 선택

### Orchestration 패턴 (중앙 집중식)
```java
// Saga Orchestrator 인터페이스
public interface SagaOrchestrator<T> {
    String getSagaType();
    void process(T sagaData);
    void compensate(T sagaData, String failedStep);
}

// 주문 처리 Saga Orchestrator
@Component
public class OrderProcessingSagaOrchestrator implements SagaOrchestrator<OrderSagaData> {
    
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final ShippingService shippingService;
    private final SagaManager sagaManager;
    
    @Override
    public String getSagaType() {
        return "OrderProcessingSaga";
    }
    
    @Override
    public void process(OrderSagaData sagaData) {
        SagaTransaction saga = sagaManager.createSaga(sagaData.getOrderId(), getSagaType());
        
        try {
            // 1단계: 재고 예약
            executeStep(saga, "RESERVE_INVENTORY", () -> {
                InventoryReservationResult result = inventoryService.reserveItems(
                    sagaData.getOrderId(), 
                    sagaData.getItems()
                );
                sagaData.setInventoryReservationId(result.getReservationId());
                return result.isSuccess();
            });
            
            // 2단계: 결제 처리
            executeStep(saga, "PROCESS_PAYMENT", () -> {
                PaymentResult result = paymentService.processPayment(
                    sagaData.getOrderId(),
                    sagaData.getTotalAmount(),
                    sagaData.getPaymentMethod()
                );
                sagaData.setPaymentId(result.getPaymentId());
                return result.isSuccess();
            });
            
            // 3단계: 배송 준비
            executeStep(saga, "PREPARE_SHIPPING", () -> {
                ShippingResult result = shippingService.prepareShipping(
                    sagaData.getOrderId(),
                    sagaData.getShippingAddress()
                );
                sagaData.setShippingId(result.getShippingId());
                return result.isSuccess();
            });
            
            // 모든 단계 성공
            sagaManager.completeSaga(saga.getSagaId());
            
        } catch (SagaExecutionException e) {
            compensate(sagaData, e.getFailedStep());
            sagaManager.failSaga(saga.getSagaId(), e.getMessage());
        }
    }
    
    @Override
    public void compensate(OrderSagaData sagaData, String failedStep) {
        log.info("Starting compensation for order: {}, failed step: {}", 
                sagaData.getOrderId(), failedStep);
        
        // 역순으로 보상 실행
        switch (failedStep) {
            case "PREPARE_SHIPPING":
                compensatePayment(sagaData);
                // fallthrough
            case "PROCESS_PAYMENT":
                compensateInventoryReservation(sagaData);
                break;
            case "RESERVE_INVENTORY":
                // 첫 번째 단계 실패 시 보상할 것이 없음
                break;
        }
    }
    
    private void executeStep(SagaTransaction saga, String stepName, Supplier<Boolean> stepExecution) {
        sagaManager.startStep(saga.getSagaId(), stepName);
        
        try {
            boolean success = stepExecution.get();
            if (success) {
                sagaManager.completeStep(saga.getSagaId(), stepName);
            } else {
                throw new SagaExecutionException("Step failed: " + stepName, stepName);
            }
        } catch (Exception e) {
            sagaManager.failStep(saga.getSagaId(), stepName, e.getMessage());
            throw new SagaExecutionException("Step execution failed: " + stepName, stepName, e);
        }
    }
    
    private void compensateInventoryReservation(OrderSagaData sagaData) {
        if (sagaData.getInventoryReservationId() != null) {
            try {
                inventoryService.releaseReservation(sagaData.getInventoryReservationId());
                log.info("Inventory reservation compensated for order: {}", sagaData.getOrderId());
            } catch (Exception e) {
                log.error("Failed to compensate inventory reservation for order: {}", 
                         sagaData.getOrderId(), e);
            }
        }
    }
    
    private void compensatePayment(OrderSagaData sagaData) {
        if (sagaData.getPaymentId() != null) {
            try {
                paymentService.refund(sagaData.getPaymentId(), sagaData.getTotalAmount());
                log.info("Payment compensated for order: {}", sagaData.getOrderId());
            } catch (Exception e) {
                log.error("Failed to compensate payment for order: {}", 
                         sagaData.getOrderId(), e);
            }
        }
    }
}
```

### Choreography 패턴 (분산 이벤트 기반)
```java
// 이벤트 기반 Saga 참여자들
@Component
public class InventoryServiceSagaParticipant {
    
    private final InventoryService inventoryService;
    private final ApplicationEventPublisher eventPublisher;
    
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            InventoryReservationResult result = inventoryService.reserveItems(
                event.getOrderId(), 
                event.getItems()
            );
            
            if (result.isSuccess()) {
                eventPublisher.publishEvent(new InventoryReservedEvent(
                    event.getOrderId(),
                    result.getReservationId(),
                    event.getItems()
                ));
            } else {
                eventPublisher.publishEvent(new InventoryReservationFailedEvent(
                    event.getOrderId(),
                    result.getFailureReason()
                ));
            }
            
        } catch (Exception e) {
            eventPublisher.publishEvent(new InventoryReservationFailedEvent(
                event.getOrderId(),
                e.getMessage()
            ));
        }
    }
    
    @EventListener
    @Async
    public void handleOrderCancelled(OrderCancelledEvent event) {
        // 재고 예약 해제
        try {
            inventoryService.releaseReservationByOrderId(event.getOrderId());
            eventPublisher.publishEvent(new InventoryReleasedEvent(event.getOrderId()));
        } catch (Exception e) {
            log.error("Failed to release inventory for cancelled order: {}", 
                     event.getOrderId(), e);
        }
    }
    
    @EventListener
    @Async
    public void handlePaymentFailed(PaymentFailedEvent event) {
        // 결제 실패 시 재고 예약 해제
        handleOrderCancelled(new OrderCancelledEvent(event.getOrderId(), "Payment failed"));
    }
}

@Component
public class PaymentServiceSagaParticipant {
    
    private final PaymentService paymentService;
    private final ApplicationEventPublisher eventPublisher;
    
    @EventListener
    @Async
    public void handleInventoryReserved(InventoryReservedEvent event) {
        try {
            PaymentResult result = paymentService.processPayment(
                event.getOrderId(),
                event.getTotalAmount(),
                event.getPaymentMethod()
            );
            
            if (result.isSuccess()) {
                eventPublisher.publishEvent(new PaymentCompletedEvent(
                    event.getOrderId(),
                    result.getPaymentId(),
                    event.getTotalAmount()
                ));
            } else {
                eventPublisher.publishEvent(new PaymentFailedEvent(
                    event.getOrderId(),
                    result.getFailureReason()
                ));
            }
            
        } catch (Exception e) {
            eventPublisher.publishEvent(new PaymentFailedEvent(
                event.getOrderId(),
                e.getMessage()
            ));
        }
    }
    
    @EventListener
    @Async
    public void handleInventoryReservationFailed(InventoryReservationFailedEvent event) {
        // 재고 예약 실패 시 주문 취소
        eventPublisher.publishEvent(new OrderCancelledEvent(
            event.getOrderId(),
            "Inventory reservation failed: " + event.getReason()
        ));
    }
    
    @EventListener
    @Async
    public void handleOrderCancelled(OrderCancelledEvent event) {
        // 주문 취소 시 결제 환불
        try {
            paymentService.refundByOrderId(event.getOrderId());
            eventPublisher.publishEvent(new PaymentRefundedEvent(event.getOrderId()));
        } catch (Exception e) {
            log.error("Failed to refund payment for cancelled order: {}", 
                     event.getOrderId(), e);
        }
    }
}

@Component
public class ShippingServiceSagaParticipant {
    
    private final ShippingService shippingService;
    private final ApplicationEventPublisher eventPublisher;
    
    @EventListener
    @Async
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        try {
            ShippingResult result = shippingService.prepareShipping(
                event.getOrderId(),
                event.getShippingAddress()
            );
            
            if (result.isSuccess()) {
                eventPublisher.publishEvent(new ShippingPreparedEvent(
                    event.getOrderId(),
                    result.getShippingId(),
                    result.getEstimatedDeliveryDate()
                ));
                
                // 주문 완료
                eventPublisher.publishEvent(new OrderCompletedEvent(event.getOrderId()));
            } else {
                eventPublisher.publishEvent(new ShippingPreparationFailedEvent(
                    event.getOrderId(),
                    result.getFailureReason()
                ));
            }
            
        } catch (Exception e) {
            eventPublisher.publishEvent(new ShippingPreparationFailedEvent(
                event.getOrderId(),
                e.getMessage()
            ));
        }
    }
}
```

## 2. Saga 상태 관리

### Saga Transaction Entity
```java
@Entity
@Table(name = "saga_transactions")
public class SagaTransaction {
    
    @Id
    private String sagaId;
    
    @Column(name = "saga_type")
    private String sagaType;
    
    @Column(name = "aggregate_id")
    private String aggregateId;
    
    @Enumerated(EnumType.STRING)
    private SagaStatus status;
    
    @Type(JsonType.class)
    @Column(name = "saga_data", columnDefinition = "jsonb")
    private String sagaData;
    
    @Column(name = "current_step")
    private String currentStep;
    
    @Column(name = "completed_steps", columnDefinition = "text[]")
    private String[] completedSteps;
    
    @Column(name = "failed_step")
    private String failedStep;
    
    @Column(name = "failure_reason")
    private String failureReason;
    
    @Column(name = "created_at")
    @CreationTimestamp
    private Instant createdAt;
    
    @Column(name = "updated_at")
    @UpdateTimestamp
    private Instant updatedAt;
    
    @Column(name = "completed_at")
    private Instant completedAt;
    
    // 생성자, getter, setter
}

public enum SagaStatus {
    STARTED,
    IN_PROGRESS,
    COMPLETED,
    FAILED,
    COMPENSATING,
    COMPENSATED
}
```

### Saga Step Entity
```java
@Entity
@Table(name = "saga_steps")
public class SagaStep {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "saga_id")
    private String sagaId;
    
    @Column(name = "step_name")
    private String stepName;
    
    @Column(name = "step_order")
    private Integer stepOrder;
    
    @Enumerated(EnumType.STRING)
    private StepStatus status;
    
    @Type(JsonType.class)
    @Column(name = "step_data", columnDefinition = "jsonb")
    private String stepData;
    
    @Column(name = "failure_reason")
    private String failureReason;
    
    @Column(name = "retry_count")
    private Integer retryCount;
    
    @Column(name = "started_at")
    private Instant startedAt;
    
    @Column(name = "completed_at")
    private Instant completedAt;
    
    // 생성자, getter, setter
}

public enum StepStatus {
    PENDING,
    IN_PROGRESS,
    COMPLETED,
    FAILED,
    COMPENSATED
}
```

### Saga Manager 구현
```java
@Service
@Transactional
public class SagaManager {
    
    private final SagaTransactionRepository sagaRepository;
    private final SagaStepRepository stepRepository;
    private final ObjectMapper objectMapper;
    
    public SagaTransaction createSaga(String aggregateId, String sagaType) {
        SagaTransaction saga = new SagaTransaction();
        saga.setSagaId(UUID.randomUUID().toString());
        saga.setSagaType(sagaType);
        saga.setAggregateId(aggregateId);
        saga.setStatus(SagaStatus.STARTED);
        saga.setCompletedSteps(new String[0]);
        
        return sagaRepository.save(saga);
    }
    
    public void startStep(String sagaId, String stepName) {
        SagaTransaction saga = getSaga(sagaId);
        saga.setCurrentStep(stepName);
        saga.setStatus(SagaStatus.IN_PROGRESS);
        sagaRepository.save(saga);
        
        SagaStep step = new SagaStep();
        step.setSagaId(sagaId);
        step.setStepName(stepName);
        step.setStatus(StepStatus.IN_PROGRESS);
        step.setRetryCount(0);
        step.setStartedAt(Instant.now());
        
        stepRepository.save(step);
    }
    
    public void completeStep(String sagaId, String stepName) {
        SagaTransaction saga = getSaga(sagaId);
        
        // 완료된 스텝 목록에 추가
        List<String> completedSteps = new ArrayList<>(Arrays.asList(saga.getCompletedSteps()));
        completedSteps.add(stepName);
        saga.setCompletedSteps(completedSteps.toArray(new String[0]));
        
        sagaRepository.save(saga);
        
        // 스텝 상태 업데이트
        SagaStep step = stepRepository.findBySagaIdAndStepName(sagaId, stepName)
            .orElseThrow(() -> new IllegalStateException("Step not found"));
        
        step.setStatus(StepStatus.COMPLETED);
        step.setCompletedAt(Instant.now());
        stepRepository.save(step);
    }
    
    public void failStep(String sagaId, String stepName, String reason) {
        SagaTransaction saga = getSaga(sagaId);
        saga.setFailedStep(stepName);
        saga.setFailureReason(reason);
        sagaRepository.save(saga);
        
        SagaStep step = stepRepository.findBySagaIdAndStepName(sagaId, stepName)
            .orElseThrow(() -> new IllegalStateException("Step not found"));
        
        step.setStatus(StepStatus.FAILED);
        step.setFailureReason(reason);
        step.setCompletedAt(Instant.now());
        stepRepository.save(step);
    }
    
    public void completeSaga(String sagaId) {
        SagaTransaction saga = getSaga(sagaId);
        saga.setStatus(SagaStatus.COMPLETED);
        saga.setCompletedAt(Instant.now());
        sagaRepository.save(saga);
    }
    
    public void failSaga(String sagaId, String reason) {
        SagaTransaction saga = getSaga(sagaId);
        saga.setStatus(SagaStatus.FAILED);
        saga.setFailureReason(reason);
        sagaRepository.save(saga);
    }
    
    public void startCompensation(String sagaId) {
        SagaTransaction saga = getSaga(sagaId);
        saga.setStatus(SagaStatus.COMPENSATING);
        sagaRepository.save(saga);
    }
    
    public void completeCompensation(String sagaId) {
        SagaTransaction saga = getSaga(sagaId);
        saga.setStatus(SagaStatus.COMPENSATED);
        saga.setCompletedAt(Instant.now());
        sagaRepository.save(saga);
    }
    
    private SagaTransaction getSaga(String sagaId) {
        return sagaRepository.findById(sagaId)
            .orElseThrow(() -> new SagaNotFoundException("Saga not found: " + sagaId));
    }
    
    public List<SagaTransaction> findStuckSagas(Duration timeout) {
        Instant cutoff = Instant.now().minus(timeout);
        return sagaRepository.findByStatusInAndUpdatedAtBefore(
            List.of(SagaStatus.STARTED, SagaStatus.IN_PROGRESS),
            cutoff
        );
    }
}
```

## 3. 보상 트랜잭션 설계

### 보상 액션 인터페이스
```java
public interface CompensationAction {
    String getActionName();
    boolean compensate(String sagaId, Object compensationData);
    boolean isIdempotent();
}

// 재고 예약 보상
@Component
public class InventoryReservationCompensation implements CompensationAction {
    
    private final InventoryService inventoryService;
    
    @Override
    public String getActionName() {
        return "COMPENSATE_INVENTORY_RESERVATION";
    }
    
    @Override
    public boolean compensate(String sagaId, Object compensationData) {
        try {
            InventoryCompensationData data = (InventoryCompensationData) compensationData;
            
            // 재고 예약 해제
            inventoryService.releaseReservation(data.getReservationId());
            
            log.info("Inventory reservation compensated for saga: {}, reservation: {}", 
                    sagaId, data.getReservationId());
            
            return true;
            
        } catch (Exception e) {
            log.error("Failed to compensate inventory reservation for saga: {}", sagaId, e);
            return false;
        }
    }
    
    @Override
    public boolean isIdempotent() {
        return true; // 여러 번 실행해도 안전
    }
}

// 결제 보상
@Component
public class PaymentCompensation implements CompensationAction {
    
    private final PaymentService paymentService;
    
    @Override
    public String getActionName() {
        return "COMPENSATE_PAYMENT";
    }
    
    @Override
    public boolean compensate(String sagaId, Object compensationData) {
        try {
            PaymentCompensationData data = (PaymentCompensationData) compensationData;
            
            // 결제 환불
            RefundResult result = paymentService.refund(
                data.getPaymentId(), 
                data.getAmount()
            );
            
            if (result.isSuccess()) {
                log.info("Payment compensated for saga: {}, payment: {}", 
                        sagaId, data.getPaymentId());
                return true;
            } else {
                log.error("Payment compensation failed for saga: {}, reason: {}", 
                         sagaId, result.getFailureReason());
                return false;
            }
            
        } catch (Exception e) {
            log.error("Failed to compensate payment for saga: {}", sagaId, e);
            return false;
        }
    }
    
    @Override
    public boolean isIdempotent() {
        return false; // 중복 환불 방지 필요
    }
}
```

### 보상 실행 엔진
```java
@Service
public class CompensationEngine {
    
    private final Map<String, CompensationAction> compensationActions;
    private final SagaManager sagaManager;
    
    public CompensationEngine(List<CompensationAction> actions, SagaManager sagaManager) {
        this.compensationActions = actions.stream()
            .collect(Collectors.toMap(
                CompensationAction::getActionName,
                Function.identity()
            ));
        this.sagaManager = sagaManager;
    }
    
    public void executeCompensation(String sagaId, List<CompensationInstruction> instructions) {
        sagaManager.startCompensation(sagaId);
        
        boolean allSuccess = true;
        
        // 역순으로 보상 실행
        Collections.reverse(instructions);
        
        for (CompensationInstruction instruction : instructions) {
            boolean success = executeCompensationAction(sagaId, instruction);
            
            if (!success) {
                allSuccess = false;
                // 실패한 보상이 있어도 계속 진행 (최선 노력)
                log.warn("Compensation action failed for saga: {}, action: {}", 
                        sagaId, instruction.getActionName());
            }
        }
        
        if (allSuccess) {
            sagaManager.completeCompensation(sagaId);
        } else {
            // 일부 보상 실패 - 수동 개입 필요
            sagaManager.failSaga(sagaId, "Some compensation actions failed - manual intervention required");
            
            // 알림 발송
            sendCompensationFailureAlert(sagaId, instructions);
        }
    }
    
    private boolean executeCompensationAction(String sagaId, CompensationInstruction instruction) {
        CompensationAction action = compensationActions.get(instruction.getActionName());
        
        if (action == null) {
            log.error("Compensation action not found: {}", instruction.getActionName());
            return false;
        }
        
        try {
            if (action.isIdempotent() || !instruction.isAlreadyExecuted()) {
                return action.compensate(sagaId, instruction.getCompensationData());
            } else {
                log.info("Skipping non-idempotent compensation action already executed: {}", 
                        instruction.getActionName());
                return true;
            }
        } catch (Exception e) {
            log.error("Compensation action execution failed: {}", instruction.getActionName(), e);
            return false;
        }
    }
    
    private void sendCompensationFailureAlert(String sagaId, List<CompensationInstruction> instructions) {
        // 운영팀에게 알림 발송
        // 예: Slack, Email, 모니터링 시스템 등
    }
}

public class CompensationInstruction {
    
    private String actionName;
    private Object compensationData;
    private boolean alreadyExecuted;
    
    // 생성자, getter, setter
}
```

## 4. 분산 트랜잭션 상태 관리

### Saga State Machine
```java
public class SagaStateMachine {
    
    private final Map<SagaStatus, Set<SagaStatus>> allowedTransitions;
    
    public SagaStateMachine() {
        this.allowedTransitions = Map.of(
            SagaStatus.STARTED, Set.of(SagaStatus.IN_PROGRESS, SagaStatus.FAILED),
            SagaStatus.IN_PROGRESS, Set.of(SagaStatus.COMPLETED, SagaStatus.FAILED, SagaStatus.COMPENSATING),
            SagaStatus.FAILED, Set.of(SagaStatus.COMPENSATING),
            SagaStatus.COMPENSATING, Set.of(SagaStatus.COMPENSATED, SagaStatus.FAILED),
            SagaStatus.COMPLETED, Set.of(), // 최종 상태
            SagaStatus.COMPENSATED, Set.of() // 최종 상태
        );
    }
    
    public boolean canTransition(SagaStatus from, SagaStatus to) {
        return allowedTransitions.getOrDefault(from, Set.of()).contains(to);
    }
    
    public void validateTransition(SagaStatus from, SagaStatus to) {
        if (!canTransition(from, to)) {
            throw new IllegalStateTransitionException(
                String.format("Invalid state transition from %s to %s", from, to)
            );
        }
    }
}
```

### Timeout 및 재시도 관리
```java
@Component
public class SagaTimeoutManager {
    
    private final SagaManager sagaManager;
    private final CompensationEngine compensationEngine;
    
    @Scheduled(fixedDelay = 60000) // 1분마다 실행
    public void checkTimeouts() {
        // 30분 이상 진행 중인 Saga 찾기
        Duration timeout = Duration.ofMinutes(30);
        List<SagaTransaction> stuckSagas = sagaManager.findStuckSagas(timeout);
        
        for (SagaTransaction saga : stuckSagas) {
            handleTimeout(saga);
        }
    }
    
    private void handleTimeout(SagaTransaction saga) {
        log.warn("Saga timeout detected: {}, type: {}, duration: {}", 
                saga.getSagaId(), 
                saga.getSagaType(),
                Duration.between(saga.getCreatedAt(), Instant.now()));
        
        switch (saga.getStatus()) {
            case STARTED, IN_PROGRESS -> {
                // 진행 중인 Saga 보상 실행
                List<CompensationInstruction> instructions = 
                    buildCompensationInstructions(saga);
                compensationEngine.executeCompensation(saga.getSagaId(), instructions);
            }
            case COMPENSATING -> {
                // 보상 중인 Saga 강제 완료
                sagaManager.completeCompensation(saga.getSagaId());
                log.warn("Forced completion of compensating saga: {}", saga.getSagaId());
            }
            default -> {
                // 이미 완료된 상태
                log.debug("Saga already in final state: {}", saga.getSagaId());
            }
        }
    }
    
    private List<CompensationInstruction> buildCompensationInstructions(SagaTransaction saga) {
        // 완료된 스텝들에 대한 보상 명령 생성
        List<CompensationInstruction> instructions = new ArrayList<>();
        
        for (String completedStep : saga.getCompletedSteps()) {
            CompensationInstruction instruction = createCompensationInstruction(
                saga, completedStep
            );
            if (instruction != null) {
                instructions.add(instruction);
            }
        }
        
        return instructions;
    }
    
    private CompensationInstruction createCompensationInstruction(SagaTransaction saga, 
                                                                String stepName) {
        // 스텝별 보상 명령 생성 로직
        return switch (stepName) {
            case "RESERVE_INVENTORY" -> new CompensationInstruction(
                "COMPENSATE_INVENTORY_RESERVATION",
                extractInventoryData(saga.getSagaData()),
                false
            );
            case "PROCESS_PAYMENT" -> new CompensationInstruction(
                "COMPENSATE_PAYMENT",
                extractPaymentData(saga.getSagaData()),
                false
            );
            default -> null;
        };
    }
}
```

## 5. 실패 시나리오 처리

### 재시도 메커니즘
```java
@Component
public class SagaRetryManager {
    
    private final SagaManager sagaManager;
    private final Map<String, SagaOrchestrator<?>> orchestrators;
    
    @RetryableTopic(
        attempts = "3",
        backoff = @Backoff(delay = 5000, multiplier = 2),
        dltStrategy = DltStrategy.FAIL_ON_ERROR
    )
    @KafkaListener(topics = "saga-retry-topic")
    public void retrySagaStep(SagaRetryMessage retryMessage) {
        String sagaId = retryMessage.getSagaId();
        String stepName = retryMessage.getStepName();
        
        try {
            SagaTransaction saga = sagaManager.getSaga(sagaId);
            SagaOrchestrator<?> orchestrator = orchestrators.get(saga.getSagaType());
            
            if (orchestrator != null) {
                // 특정 스텝부터 재시작
                orchestrator.retryFromStep(saga, stepName);
            }
            
        } catch (Exception e) {
            log.error("Saga retry failed: {}, step: {}", sagaId, stepName, e);
            
            // 최대 재시도 횟수 초과 시 보상 실행
            if (retryMessage.getRetryCount() >= 3) {
                triggerCompensation(sagaId);
            }
        }
    }
    
    private void triggerCompensation(String sagaId) {
        // 보상 트랜잭션 실행
        SagaTransaction saga = sagaManager.getSaga(sagaId);
        List<CompensationInstruction> instructions = buildCompensationInstructions(saga);
        compensationEngine.executeCompensation(sagaId, instructions);
    }
}
```

### Circuit Breaker 통합
```java
@Component
public class ResilientSagaOrchestrator {
    
    private final CircuitBreaker inventoryCircuitBreaker;
    private final CircuitBreaker paymentCircuitBreaker;
    private final CircuitBreaker shippingCircuitBreaker;
    
    public ResilientSagaOrchestrator() {
        this.inventoryCircuitBreaker = CircuitBreaker.ofDefaults("inventory-service");
        this.paymentCircuitBreaker = CircuitBreaker.ofDefaults("payment-service");
        this.shippingCircuitBreaker = CircuitBreaker.ofDefaults("shipping-service");
    }
    
    public void processOrderWithCircuitBreaker(OrderSagaData sagaData) {
        SagaTransaction saga = sagaManager.createSaga(sagaData.getOrderId(), "OrderProcessingSaga");
        
        try {
            // 재고 예약 (Circuit Breaker 적용)
            Supplier<Boolean> inventoryReservation = CircuitBreaker
                .decorateSupplier(inventoryCircuitBreaker, () -> {
                    return inventoryService.reserveItems(
                        sagaData.getOrderId(), 
                        sagaData.getItems()
                    ).isSuccess();
                });
            
            boolean inventorySuccess = inventoryReservation.get();
            if (!inventorySuccess) {
                throw new SagaExecutionException("Inventory reservation failed", "RESERVE_INVENTORY");
            }
            
            // 결제 처리 (Circuit Breaker 적용)
            Supplier<Boolean> paymentProcessing = CircuitBreaker
                .decorateSupplier(paymentCircuitBreaker, () -> {
                    return paymentService.processPayment(
                        sagaData.getOrderId(),
                        sagaData.getTotalAmount(),
                        sagaData.getPaymentMethod()
                    ).isSuccess();
                });
            
            boolean paymentSuccess = paymentProcessing.get();
            if (!paymentSuccess) {
                throw new SagaExecutionException("Payment processing failed", "PROCESS_PAYMENT");
            }
            
            sagaManager.completeSaga(saga.getSagaId());
            
        } catch (CircuitBreakerOpenException e) {
            log.warn("Circuit breaker open for saga: {}", saga.getSagaId());
            // Circuit breaker가 열린 경우 지연 후 재시도
            scheduleRetry(saga.getSagaId(), Duration.ofMinutes(5));
            
        } catch (SagaExecutionException e) {
            // 보상 실행
            compensate(sagaData, e.getFailedStep());
            sagaManager.failSaga(saga.getSagaId(), e.getMessage());
        }
    }
    
    private void scheduleRetry(String sagaId, Duration delay) {
        // 지연 후 재시도 스케줄링
        CompletableFuture.delayedExecutor(delay.toMillis(), TimeUnit.MILLISECONDS)
            .execute(() -> {
                SagaRetryMessage retryMessage = new SagaRetryMessage(sagaId, "RETRY_ALL", 0);
                kafkaTemplate.send("saga-retry-topic", retryMessage);
            });
    }
}
```

## 6. 모니터링 및 추적

### Saga 메트릭 수집
```java
@Component
public class SagaMetrics {
    
    private final Counter sagaStartedCounter;
    private final Counter sagaCompletedCounter;
    private final Counter sagaFailedCounter;
    private final Timer sagaDurationTimer;
    private final Gauge activeSagasGauge;
    
    public SagaMetrics(MeterRegistry meterRegistry, SagaManager sagaManager) {
        this.sagaStartedCounter = Counter.builder("saga.started")
            .description("Number of sagas started")
            .register(meterRegistry);
            
        this.sagaCompletedCounter = Counter.builder("saga.completed")
            .description("Number of sagas completed")
            .register(meterRegistry);
            
        this.sagaFailedCounter = Counter.builder("saga.failed")
            .description("Number of sagas failed")
            .register(meterRegistry);
            
        this.sagaDurationTimer = Timer.builder("saga.duration")
            .description("Saga execution duration")
            .register(meterRegistry);
            
        this.activeSagasGauge = Gauge.builder("saga.active")
            .description("Number of active sagas")
            .register(meterRegistry, this, this::getActiveSagaCount);
    }
    
    private double getActiveSagaCount(SagaMetrics metrics) {
        return sagaManager.countActiveSagas();
    }
    
    @EventListener
    public void handleSagaStarted(SagaStartedEvent event) {
        sagaStartedCounter.increment(
            Tags.of("saga_type", event.getSagaType())
        );
    }
    
    @EventListener
    public void handleSagaCompleted(SagaCompletedEvent event) {
        sagaCompletedCounter.increment(
            Tags.of("saga_type", event.getSagaType())
        );
        
        Duration duration = Duration.between(event.getStartTime(), event.getEndTime());
        sagaDurationTimer.record(duration);
    }
    
    @EventListener
    public void handleSagaFailed(SagaFailedEvent event) {
        sagaFailedCounter.increment(
            Tags.of(
                "saga_type", event.getSagaType(),
                "failure_reason", event.getFailureReason()
            )
        );
    }
}
```

### Saga 대시보드
```java
@RestController
@RequestMapping("/api/sagas")
public class SagaMonitoringController {
    
    private final SagaManager sagaManager;
    private final SagaMetrics sagaMetrics;
    
    @GetMapping("/status")
    public SagaStatusSummary getSagaStatus() {
        return SagaStatusSummary.builder()
            .activeSagas(sagaManager.countActiveSagas())
            .completedSagas(sagaManager.countCompletedSagas())
            .failedSagas(sagaManager.countFailedSagas())
            .compensatedSagas(sagaManager.countCompensatedSagas())
            .build();
    }
    
    @GetMapping("/failed")
    public List<SagaTransaction> getFailedSagas(@RequestParam(defaultValue = "24") int hours) {
        Instant since = Instant.now().minus(Duration.ofHours(hours));
        return sagaManager.findFailedSagasSince(since);
    }
    
    @GetMapping("/{sagaId}")
    public SagaDetails getSagaDetails(@PathVariable String sagaId) {
        SagaTransaction saga = sagaManager.getSaga(sagaId);
        List<SagaStep> steps = sagaManager.getSagaSteps(sagaId);
        
        return SagaDetails.builder()
            .saga(saga)
            .steps(steps)
            .timeline(buildTimeline(saga, steps))
            .build();
    }
    
    @PostMapping("/{sagaId}/retry")
    public ResponseEntity<Void> retrySaga(@PathVariable String sagaId) {
        try {
            sagaManager.retrySaga(sagaId);
            return ResponseEntity.ok().build();
        } catch (Exception e) {
            return ResponseEntity.badRequest().build();
        }
    }
}
```

## 실습 과제

1. **완전한 Saga 시스템**: Orchestration과 Choreography 패턴을 모두 구현하여 비교
2. **복잡한 비즈니스 플로우**: 다중 서비스를 거치는 복잡한 주문 처리 플로우 구현
3. **장애 시뮬레이션**: 다양한 장애 상황에서의 보상 트랜잭션 동작 테스트
4. **성능 최적화**: 대용량 Saga 처리를 위한 성능 최적화 및 병목 지점 분석
5. **모니터링 시스템**: Saga 상태 및 성능을 실시간으로 모니터링할 수 있는 대시보드 구현

## 참고 자료

- [Saga Pattern](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)
- [Microservices Patterns: Saga](https://microservices.io/patterns/data/saga.html)
- [Distributed Sagas: A Protocol for Coordinating Microservices](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)
- [Implementing Saga Pattern with Spring Boot](https://spring.io/guides/gs/messaging-jms/)