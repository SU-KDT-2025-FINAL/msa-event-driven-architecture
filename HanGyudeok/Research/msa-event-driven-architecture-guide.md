
# 📘 Microservices & Event-Driven Architecture 설계와 적용 가이드

(Technical Research: MSA & Event-Driven Architecture for Scalable Systems)

---

## ✨ 핵심 주제

* **큰 시스템을 여러 개의 작고 독립적인 서비스로 나눠서 개발하는 구조 (MSA)** 와
* **이벤트가 발생할 때 자동으로 반응하는 구조 (EDA)** 를
* 실제 시스템에서 어떻게 적용하는지 정리합니다.

---

## 📌 왜 MSA와 이벤트 기반 구조가 필요할까?

### 문제 상황

기존에는 하나의 프로그램이 모든 기능을 처리했어요.
(📦 예: 로그인, 결제, 상품관리, 알림 등 → 전부 하나의 코드로 묶음)

하지만 이렇게 하면...

* **한 기능에 문제가 생기면 전체가 멈추고**
* **새로운 기능을 넣으려면 전체를 수정해야 하고**
* **많은 개발자가 동시에 작업하기 어려워요**

### 해결 방법

1. **각 기능을 작게 나눠서(MSA)** → 독립적으로 만들고
2. **각 기능이 서로 "이벤트"로 연결되도록(EDA)** → 더 유연하게 소통해요.

---

## 💡 주요 개념 풀이 (쉬운 설명)

| 용어                                   | 설명                                    | 중학생 비유                             |
| ------------------------------------ | ------------------------------------- | ---------------------------------- |
| **MSA (Microservices Architecture)** | 전체 시스템을 작고 독립적인 기능 단위(서비스)로 나누는 설계 방식 | 레고처럼 하나하나 분리된 블록을 조립해서 큰 건물을 만드는 것 |
| **모놀리식 (Monolithic)**                | 하나의 덩어리로 된 프로그램 구조                    | 거대한 하나의 벽돌집 (한 벽돌 깨지면 집 전체가 위험해요)  |
| **서비스 간 통신 (API, 메시지)**              | 각 마이크로서비스끼리 정보를 주고받는 방식               | 친구끼리 쪽지 주고받는 것                     |
| **Event (이벤트)**                      | 어떤 일이 발생한 신호                          | “문이 열렸어요!”, “사용자가 결제했어요!” 같은 알림    |
| **Event-Driven Architecture (EDA)**  | 이벤트가 발생하면, 다른 서비스가 그것에 반응하도록 설계하는 구조  | “급식 나왔어요!” 방송을 듣고 급식실에 가는 학생들      |
| **이벤트 브로커 (Kafka, RabbitMQ)**        | 이벤트들을 모아두고 필요한 서비스에 전달하는 도구           | 방송국처럼 방송을 내보내고, 듣는 사람은 필요할 때 듣는 구조 |
| **비동기 통신**                           | 바로 응답을 기다리지 않고, 나중에 반응                | 친구에게 메모 남기고 답은 나중에 받는 것            |
| **동기 통신**                            | 요청하고 바로 응답을 기다림                       | 실시간 전화 통화처럼 바로 대답이 와야 다음으로 진행 가능   |

---

## 🔁 시스템 흐름 요약도

```text
[사용자 요청]
     ↓
[API Gateway가 요청을 받아]
     ↓
[각 마이크로서비스로 전달]
     ↓
[서비스 간 필요한 정보는 메시지 또는 이벤트로 주고받음]
     ↓
[데이터 저장 / 결과 응답]
```

---

## 🧠 실제 구조 예시

📦 쇼핑몰을 예로 들어 보자면!

| 기능    | 독립된 마이크로서비스          | 이벤트 흐름                                    |
| ----- | -------------------- | ----------------------------------------- |
| 상품 보기 | catalog-service      | 사용자 조회 요청 발생 시 응답                         |
| 결제 처리 | payment-service      | 결제 완료 → `order_placed` 이벤트 발생             |
| 배송 처리 | delivery-service     | `order_placed` 이벤트 감지 → 배송 시작             |
| 알림    | notification-service | 배송 시작 → `delivery_started` 이벤트 감지 → 알림 발송 |

---

## ⚙️ 실무에서의 기술 키워드

| 범주              | 키워드                               | 설명                              |
| --------------- | --------------------------------- | ------------------------------- |
| **서비스 통신**      | REST, gRPC, Kafka, RabbitMQ       | 서비스 간 정보 교환 방식 (API or 메시징)     |
| **이벤트 처리**      | Kafka, Redis Stream, NATS         | 메시지를 비동기로 전달하고 처리               |
| **API Gateway** | NGINX, Kong, Spring Cloud Gateway | 외부 요청을 내부 서비스로 연결해주는 입구         |
| **서비스 탐색**      | Service Discovery, Consul, Eureka | 서비스 위치를 자동으로 찾기                 |
| **장애 대응**       | Circuit Breaker, Retry            | 한 서비스가 실패해도 전체 장애로 번지지 않도록 보호   |
| **데이터 정합성**     | Saga, Event Sourcing              | 여러 서비스가 공동 작업 시 데이터 문제를 해결하는 방법 |

---

## 📘 코드 예시 (Kafka 사용, Java/Spring)

```java
// 주문이 생성되었을 때 Kafka로 이벤트 발행
public void createOrder(OrderRequest request) {
    Order order = orderRepository.save(request.toEntity());
    kafkaTemplate.send("order_created", new OrderCreatedEvent(order));
}
```

```java
// 배송 서비스가 이벤트를 받아서 배송 시작
@KafkaListener(topics = "order_created", groupId = "delivery")
public void handleOrderCreated(OrderCreatedEvent event) {
    deliveryService.startDelivery(event.getOrderId());
}
```

---

## 📚 헷갈릴 수 있는 용어 설명

| 용어                  | 쉽게 풀이하면                   |
| ------------------- | ------------------------- |
| **서비스(Service)**    | 기능 하나하나를 담당하는 작은 프로그램이에요  |
| **브로커(Broker)**     | 메시지를 받아서 다른 사람에게 전달해주는 역할 |
| **이벤트(Event)**      | "이 일이 일어났어요!"라고 말해주는 신호   |
| **비동기**             | 요청하고 기다리지 않고 다른 일 먼저 하는 것 |
| **분산(Distributed)** | 여러 컴퓨터에 나눠서 일 시키는 방식      |

---

## 🧪 이 구조에서 개발자가 배워야 할 기술

| 분야      | 추천 학습 키워드                               |
| ------- | --------------------------------------- |
| MSA 기본  | REST API, API Gateway, Service Registry |
| 이벤트 처리  | Kafka, 메시징 구조, Pub/Sub 개념               |
| 데이터 정합성 | Saga, Eventual Consistency 개념           |
| 에러 대응   | Circuit Breaker, Retry, Timeout 설정      |
| 트래픽 분산  | 로드밸런서, 컨테이너 오케스트레이션                     |

---

## 🔍 리서치 키워드 정리

* **Microservices Architecture**
* **Event-Driven Architecture**
* **Kafka / RabbitMQ**
* **Service Communication (API / Messaging)**
* **Saga Pattern**
* **Circuit Breaker / Fault Tolerance**
* **API Gateway**
* **Service Discovery**
* **Eventual Consistency**
* **Distributed Logging & Tracing**
