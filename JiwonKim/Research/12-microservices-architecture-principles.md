# 1-2. 마이크로서비스 아키텍처 원칙

## 개요

마이크로서비스 아키텍처는 복잡한 애플리케이션을 작고 독립적인 서비스들의 집합으로 구성하는 아키텍처 스타일입니다. 각 서비스는 특정 비즈니스 기능을 담당하며, 잘 정의된 API를 통해 통신합니다.

## 서비스 분해 전략

### 도메인 주도 설계(DDD) 정렬

**바운디드 컨텍스트와의 매핑**: 서비스는 바운디드 컨텍스트에 매핑되어야 합니다. 각 바운디드 컨텍스트는 특정 도메인 영역을 나타내며, 해당 영역 내에서 일관된 용어와 개념을 사용합니다.

**애그리게이트 경계**: 애그리게이트 경계가 서비스 경계를 정의합니다. 애그리게이트는 일관성 경계를 나타내며, 트랜잭션의 범위를 결정합니다.

**유비쿼터스 언어**: 서비스 경계 내에서 도메인 전문가와 개발자가 공유하는 공통 언어를 사용합니다.

**컨텍스트 매핑**: 서비스 간의 관계와 통합 패턴을 정의합니다.

```
고객 관리 서비스 (Customer Management)
├── 고객 등록
├── 고객 정보 관리
└── 고객 인증

주문 관리 서비스 (Order Management)
├── 주문 생성
├── 주문 상태 관리
└── 주문 이력

결제 서비스 (Payment Service)
├── 결제 처리
├── 환불 관리
└── 결제 수단 관리
```

### 데이터 소유권과 캡슐화

**독점적 데이터 소유권**: 각 서비스는 자체 데이터를 독점적으로 소유합니다. 다른 서비스는 직접적인 데이터베이스 접근을 하지 않습니다.

**서비스 경계를 넘나드는 데이터 접근 금지**: 서비스 간 직접적인 데이터베이스 접근은 강한 결합을 만들고 서비스 자율성을 해칩니다.

**API 계약을 통한 데이터 공유**: 다른 서비스의 데이터가 필요한 경우 잘 정의된 API를 통해서만 접근합니다.

**이벤트 스트림을 통한 데이터 동기화**: 서비스 간 데이터 일관성은 이벤트를 통해 최종적 일관성으로 달성합니다.

## 통신 패턴 심화

### 동기 통신

**직접 HTTP/gRPC 호출**: 클라이언트가 서버를 직접 호출하고 즉시 응답을 기다립니다.

**차단 작업**: 응답을 받을 때까지 호출자는 다른 작업을 수행할 수 없습니다.

**적합한 용도**:
- 읽기 작업 (조회)
- 중요한 비즈니스 검증
- 실시간 응답이 필요한 경우

**단점**:
- 시간적 결합 (Temporal Coupling)
- 연쇄 실패 (Cascading Failures)
- 가용성 감소

```java
// 동기 통신 예시
@RestController
public class OrderController {
    
    @Autowired
    private PaymentServiceClient paymentService;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        // 결제 서비스를 동기적으로 호출
        PaymentResult result = paymentService.processPayment(request.getPaymentInfo());
        
        if (result.isSuccess()) {
            Order order = orderService.createOrder(request);
            return ResponseEntity.ok(order);
        } else {
            return ResponseEntity.badRequest().build();
        }
    }
}
```

### 비동기 통신

**메시지 기반 통신**: 브로커를 통한 메시지 교환으로 서비스 간 통신을 수행합니다.

**비차단 작업**: 메시지를 보낸 후 즉시 다른 작업을 수행할 수 있습니다.

**적합한 용도**:
- 비즈니스 프로세스 플로우
- 알림 및 이벤트 전파
- 데이터 동기화
- 장시간 실행되는 작업

**장점**:
- 시간적 분리 (Temporal Decoupling)
- 향상된 회복력
- 독립적인 확장

```java
// 비동기 통신 예시
@Service
public class OrderService {
    
    @Autowired
    private EventPublisher eventPublisher;
    
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        order.setStatus(OrderStatus.PENDING);
        orderRepository.save(order);
        
        // 비동기적으로 이벤트 발행
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), 
            order.getCustomerId(), 
            order.getTotalAmount()
        );
        eventPublisher.publish(event);
        
        return order;
    }
}
```

### 서비스 경계 설계

**인터페이스 설계 원칙**:

**조립도가 굵은 인터페이스**: 네트워크 통신을 최소화하기 위해 세밀한 호출보다는 굵은 단위의 호출을 사용합니다.

```java
// 좋지 않은 예: 세밀한 호출
customer.setFirstName("John");
customer.setLastName("Doe");
customer.setEmail("john.doe@example.com");
customer.setPhoneNumber("123-456-7890");

// 좋은 예: 굵은 단위의 호출
CustomerUpdateRequest request = CustomerUpdateRequest.builder()
    .firstName("John")
    .lastName("Doe")
    .email("john.doe@example.com")
    .phoneNumber("123-456-7890")
    .build();
customerService.updateCustomer(customerId, request);
```

**안정적인 계약과 버전 관리**: API 계약은 하위 호환성을 유지하며 점진적으로 진화해야 합니다.

**멱등적 작업**: 동일한 요청을 여러 번 수행해도 같은 결과를 보장합니다.

```java
@PutMapping("/customers/{customerId}")
public ResponseEntity<Customer> updateCustomer(
    @PathVariable String customerId,
    @RequestBody CustomerUpdateRequest request,
    @RequestHeader("Idempotency-Key") String idempotencyKey) {
    
    // 멱등성 키를 확인하여 중복 처리 방지
    if (operationService.isAlreadyProcessed(idempotencyKey)) {
        return ResponseEntity.ok(customerService.getCustomer(customerId));
    }
    
    Customer updatedCustomer = customerService.updateCustomer(customerId, request);
    operationService.markAsProcessed(idempotencyKey);
    
    return ResponseEntity.ok(updatedCustomer);
}
```

**벌크헤드 패턴**: 장애 격리를 위해 리소스를 분리합니다.

### 데이터 일관성 모델

**강한 일관성**: 서비스 경계 내에서 ACID 속성을 유지합니다. 단일 서비스 내의 트랜잭션은 즉시 일관성을 보장합니다.

**최종적 일관성**: 서비스 간 일관성은 시간이 지남에 따라 달성됩니다. 보상 작업을 통해 불일치를 해결합니다.

**약한 일관성**: 중요하지 않은 작업에 대해서는 일시적인 불일치를 허용합니다.

## 서비스 자율성과 독립성

### 개발 자율성

**독립적인 개발 팀**: 각 서비스는 독립적인 팀에 의해 개발되고 관리됩니다.

**기술 스택 선택의 자유**: 서비스의 요구사항에 가장 적합한 기술을 선택할 수 있습니다.

**독립적인 릴리스 사이클**: 서비스는 다른 서비스와 독립적으로 배포할 수 있습니다.

### 운영 자율성

**독립적인 확장**: 각 서비스는 자체 부하 패턴에 따라 독립적으로 확장할 수 있습니다.

**격리된 장애**: 한 서비스의 장애가 다른 서비스에 직접적인 영향을 주지 않습니다.

**독립적인 모니터링**: 각 서비스는 자체적인 메트릭과 로깅을 가집니다.

## 서비스 디스커버리와 라우팅

### 서비스 디스커버리 패턴

**클라이언트 사이드 디스커버리**: 클라이언트가 직접 서비스 레지스트리를 조회하여 서비스 인스턴스를 찾습니다.

**서버 사이드 디스커버리**: 로드 밸런서나 프록시가 서비스 위치를 관리합니다.

**서비스 레지스트리**: 사용 가능한 서비스 인스턴스의 중앙 저장소입니다.

### API 게이트웨이 패턴

**단일 진입점**: 모든 클라이언트 요청의 단일 진입점을 제공합니다.

**라우팅과 조합**: 클라이언트 요청을 적절한 백엔드 서비스로 라우팅하고 여러 서비스의 응답을 조합합니다.

**교차 관심사**: 인증, 인가, 로깅, 모니터링 등의 공통 기능을 처리합니다.

```java
@RestController
public class ApiGatewayController {
    
    @GetMapping("/api/order-summary/{customerId}")
    public ResponseEntity<OrderSummary> getOrderSummary(@PathVariable String customerId) {
        // 여러 서비스 호출을 조합
        Customer customer = customerService.getCustomer(customerId);
        List<Order> orders = orderService.getOrdersByCustomer(customerId);
        PaymentInfo paymentInfo = paymentService.getPaymentInfo(customerId);
        
        OrderSummary summary = OrderSummary.builder()
            .customer(customer)
            .orders(orders)
            .paymentInfo(paymentInfo)
            .build();
            
        return ResponseEntity.ok(summary);
    }
}
```

## 마이크로서비스의 이점과 도전과제

### 이점

**기술적 다양성**: 각 서비스에 가장 적합한 기술 스택을 선택할 수 있습니다.

**팀 자율성**: 작은 팀이 서비스 전체를 소유하고 관리할 수 있습니다.

**독립적 배포**: 서비스별로 독립적인 배포와 릴리스가 가능합니다.

**확장성**: 필요에 따라 개별 서비스를 선택적으로 확장할 수 있습니다.

**장애 격리**: 한 서비스의 장애가 전체 시스템에 미치는 영향을 제한할 수 있습니다.

### 도전과제

**분산 시스템 복잡성**: 네트워크 통신, 부분 실패, 일관성 문제 등을 다뤄야 합니다.

**데이터 일관성**: 분산 트랜잭션과 최종적 일관성 관리가 복잡합니다.

**운영 오버헤드**: 더 많은 서비스와 인프라스트럭처를 관리해야 합니다.

**테스팅 복잡성**: 서비스 간 통합 테스트가 복잡해집니다.

**모니터링과 디버깅**: 분산된 시스템에서의 문제 추적이 어려워집니다.

## 설계 원칙과 모범 사례

### 단일 책임 원칙

각 서비스는 하나의 비즈니스 기능에 집중해야 합니다. 서비스가 너무 많은 책임을 가지게 되면 모놀리식의 문제가 다시 발생합니다.

### 데이터베이스 분리

각 서비스는 자체 데이터베이스를 가져야 하며, 다른 서비스의 데이터베이스에 직접 접근하지 않아야 합니다.

### 장애 허용성 설계

- **회로 차단기**: 실패하는 서비스에 대한 호출을 중단합니다.
- **타임아웃**: 모든 원격 호출에 적절한 타임아웃을 설정합니다.
- **재시도**: 일시적 실패에 대한 재시도 로직을 구현합니다.
- **벌크헤드**: 리소스를 격리하여 장애 전파를 방지합니다.

### 모니터링과 로깅

- **분산 추적**: 요청이 여러 서비스를 거쳐가는 과정을 추적합니다.
- **중앙 집중식 로깅**: 모든 서비스의 로그를 중앙에서 관리합니다.
- **메트릭 수집**: 비즈니스 메트릭과 기술적 메트릭을 모두 수집합니다.

이러한 원칙들을 따르면 마이크로서비스 아키텍처의 이점을 최대화하면서 복잡성을 관리할 수 있습니다.