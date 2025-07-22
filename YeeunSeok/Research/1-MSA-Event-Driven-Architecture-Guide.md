# 1-MSA 이벤트 드리븐 아키텍처 종합 가이드

이 문서는 Event Storming부터 도메인 이벤트 식별, 마이크로서비스 경계 설정, 이벤트 스키마 설계, Kafka Topic 전략, CQRS/Event Sourcing 패턴 적용까지의 단계별 방법론과 각 단계에서 고려해야 할 기술적 결정 사항, 그리고 실제 구현 예시를 포함한 종합 가이드입니다.

---

## 1. Event Storming

### 개요
- 도메인 전문가, 개발자, 기획자가 한자리에 모여 도메인 이벤트를 중심으로 비즈니스 프로세스를 시각화하는 워크숍 기법

### 방법론
1. **타임라인 그리기**: 비즈니스 프로세스의 흐름을 시간 순서대로 벽에 붙임
2. **도메인 이벤트 식별**: "~되었다" 형태의 이벤트(예: 주문이 생성되었다)를 포스트잇에 작성
3. **커맨드, 액터, 정책, 시스템 등 추가**: 이벤트를 유발하는 커맨드, 액터, 정책, 외부 시스템 등을 식별

### 기술적 결정 사항
- 모든 이해관계자 참여 필수
- 이벤트 중심으로 사고 전환 필요

### 예시
- "주문이 생성되었다", "결제가 완료되었다", "배송이 시작되었다"

---

## 2. 도메인 이벤트 식별

### 방법론
- Event Storming 결과를 바탕으로 시스템에서 발생하는 의미 있는 상태 변화를 이벤트로 정의
- 이벤트는 불변(Immutable) 객체로 설계

### 기술적 결정 사항
- 이벤트 명명 규칙(과거형, 명확한 의미)
- 이벤트 페이로드(어떤 데이터가 필요한지)

### 예시 (Java)
```java
public class OrderCreatedEvent {
    private final String orderId;
    private final String userId;
    private final LocalDateTime createdAt;
    // ...생성자 및 getter
}
```

---

## 3. 마이크로서비스 경계 설정

### 방법론
- 이벤트 흐름을 따라 자연스럽게 서비스 경계를 도출 (Bounded Context)
- 각 서비스는 독립적으로 배포/확장 가능해야 함

### 기술적 결정 사항
- 서비스 간 데이터 중복 허용 여부
- 서비스 간 통신 방식(동기/비동기)

### 예시
- 주문(Order), 결제(Payment), 배송(Delivery) 서비스로 분리

---

## 4. 이벤트 스키마 설계

### 방법론
- 이벤트의 구조(스키마)를 명확히 정의 (Avro, JSON Schema 등 활용)
- 스키마 레지스트리 도입 고려

### 기술적 결정 사항
- 스키마 버전 관리
- Backward/Forward 호환성

### 예시 (JSON Schema)
```json
{
  "type": "object",
  "properties": {
    "orderId": { "type": "string" },
    "userId": { "type": "string" },
    "createdAt": { "type": "string", "format": "date-time" }
  },
  "required": ["orderId", "userId", "createdAt"]
}
```

---

## 5. Kafka Topic 전략

### 방법론
- 도메인 이벤트별로 토픽을 분리 (ex: order-events, payment-events)
- 파티션 전략(키 선정, 병렬성 고려)

### 기술적 결정 사항
- 토픽 네이밍 규칙
- 파티션/리플리케이션 설정
- 메시지 Retention 정책

### 예시
- `order-events` 토픽에 `OrderCreatedEvent`, `OrderCancelledEvent` 등 발행

---

## 6. CQRS / Event Sourcing 패턴 적용

### 방법론
- **CQRS**: Command(쓰기)와 Query(읽기) 모델을 분리
- **Event Sourcing**: 상태를 이벤트의 시퀀스로 저장, 리플레이로 상태 복원

### 기술적 결정 사항
- 이벤트 저장소 선택 (Kafka, EventStoreDB 등)
- 스냅샷 전략(이벤트 리플레이 최적화)
- 읽기 모델 동기화 방식

### 예시 (Spring Boot + Kafka)
```java
// Command Handler
public void handle(CreateOrderCommand cmd) {
    OrderCreatedEvent event = new OrderCreatedEvent(cmd.getOrderId(), cmd.getUserId(), LocalDateTime.now());
    kafkaTemplate.send("order-events", event);
}

// Event Listener (Read Model 업데이트)
@KafkaListener(topics = "order-events")
public void onOrderCreated(OrderCreatedEvent event) {
    orderReadRepository.save(event.toReadModel());
}
```

---

## 참고 자료
- [Event Storming 공식 사이트](https://www.eventstorming.com/)
- [MSA와 Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [Kafka 공식 문서](https://kafka.apache.org/documentation/)
- [CQRS & Event Sourcing 패턴](https://martinfowler.com/bliki/CQRS.html) 

---

## MSA 및 Kafka 이벤트 아키텍처 운영의 공통 원칙

두 문서에서 반복적으로 강조된 공통 원칙은 다음과 같습니다:

### 1. Kafka Topic 설계 전략
- 파티셔닝, 리플리케이션, 네이밍 규칙 등 토픽 설계의 중요성
- 병렬 처리, 내결함성, 일관성 있는 네이밍을 통한 운영 효율성 확보

### 2. 이벤트 스키마 및 스키마 레지스트리 관리
- Avro, JSON Schema 등으로 이벤트 구조를 명확히 정의
- 스키마 버전 관리와 호환성 정책 수립의 필요성

### 3. Kafka 기반 이벤트 처리 구조
- Kafka를 이벤트 브로커로 활용하여 마이크로서비스 간 비동기 통신 구현
- Producer/Consumer 패턴의 신뢰성 및 확장성 확보

### 4. 운영 및 모니터링의 중요성
- 클러스터 상태, 지표, 알람 등 운영 관점에서의 모니터링 필수
- 데이터 내구성, 장애 대응, 신뢰성 확보를 위한 운영 전략 필요

이러한 원칙들은 MSA와 Kafka 기반 이벤트 아키텍처 모두에서 안정적이고 확장 가능한 시스템을 구축하는 데 핵심적인 역할을 합니다. 