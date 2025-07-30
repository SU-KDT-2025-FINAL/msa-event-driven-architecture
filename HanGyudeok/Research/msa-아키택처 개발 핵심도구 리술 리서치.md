# MSA 아키텍처 핵심 기술 상세 정리

| 구분          | 기술 / 도구                         | 사용 이유                                               | 장점                               | 단점                                |
|-------------|---------------------------------|-----------------------------------------------------|----------------------------------|-----------------------------------|
| **서비스 통신**  | **Spring Cloud Gateway**        | 여러 마이크로서비스를 단일 진입점으로 묶어 요청 관리, 인증·로깅·라우팅 등 공통기능 중앙화 | 유연한 라우팅, 필터 기반 공통처리, Spring과 친화적 | 학습 곡선, 복잡한 필터 로직 관리 어려움 가능        |
| **설정 관리**   | **Spring Cloud Config**         | 환경별 설정을 중앙 저장소(Git 등)에서 관리해 서비스간 일관된 설정 제공          | 설정 버전 관리, 환경별 분리 용이, 동적 설정 변경 가능 | 별도 Config 서버 운영 필요, 네트워크 장애 영향 가능 |
| **인증/인가**   | **Spring Security + JWT**       | 서비스 간 무상태 인증 및 권한 부여, 분산환경에 적합한 토큰 기반 인증            | 확장성 높음, 서버 부담 적음, 다양한 클라이언트 지원   | 토큰 탈취 보안위험, 토큰 갱신 로직 필요           |
|             | **OAuth 2.0**                   | 소셜 로그인, SSO 등 표준 인증 프로토콜 지원                         | 표준화된 보안 프로토콜, 외부 인증 연동 용이        | 설정 복잡, 구현 난이도 높음                  |
| **API 문서화** | **Swagger (Springdoc OpenAPI)** | API 명세 자동화로 개발자간 소통과 테스트 효율 증대                      | UI 제공, 자동 동기화                    | 복잡한 커스텀 API 표현 어려움, 버전 관리 필요      |
| **데이터베이스**  | **MySQL**                       | 안정적 관계형 DBMS, 서비스별 데이터 독립 관리                        | 신뢰성, 커뮤니티, 확장성                   | 분산 트랜잭션 관리 어려움, 스키마 유연성 낮음        |
| **배포/운영**   | **Docker**                      | 개발/운영 환경 일치, 컨테이너 단위 배포 자동화                         | 환경 의존성 최소화, 빠른 배포 및 확장           | 오케스트레이션 추가시 학습 비용 발생              |
| **CI/CD**   | **Jenkins**                     | 빌드·테스트·배포 자동화로 개발 효율 극대화                            | 다양한 플러그인, 커스터마이징 가능              | UI 복잡, 초기 구축 및 관리 비용              |
| **프론트엔드**   | **React Vite**                  | 컴포넌트 단위 UI 개발로 유지보수·재사용성 증대                         | 컴포넌트 재사용성, 풍부한 생태계               | 상태관리, 프로젝트 규모에 따른 복잡성             |

---

# Spring Cloud Gateway 상세 기술서

## 개요

**Spring Cloud Gateway**는 Spring 생태계에서 제공하는 API Gateway 솔루션으로, 마이크로서비스 아키텍처(MSA)에서 각 서비스에 대한 진입점을 단일화하고, 인증, 로깅, 필터링, 라우팅 등의 공통 기능을 처리하는 경량 프록시 서버입니다.

---

## Spring Cloud Gateway 주요 기능 정리

| 기능                                | 설명                                                                 |
|-------------------------------------|----------------------------------------------------------------------|
| **라우팅(Routing)**                 | 클라이언트 요청을 내부 마이크로서비스로 전달                        |
| **필터(Filter)**                    | 요청/응답 전후로 인증, 로깅, 헤더 변경, 토큰 유효성 검사 등 처리     |
| **부하 분산(Load Balancing)**       | `Spring Cloud LoadBalancer` 또는 `Eureka`와 연동하여 트래픽 분산    |
| **Path/Host/Method 기반 라우팅**    | 다양한 조건(Predicate)에 따라 라우팅 제어 가능                       |
| **서킷 브레이커(Circuit Breaker)**  | `Resilience4j` 또는 `Hystrix`와 연동해 장애 발생 시 서비스 격리     |
| **CORS, Rate Limiting, Retry**      | API 게이트웨이에 필수적인 보안 및 안정성 기능들 제공                |


## 의존성 추가

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client' 
}
````

---

## 기본 설정 예시 (`application.yml`)


server:
  port: 8000

spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://localhost:8081
          predicates:
            - Path=/users/**
          filters:
            - AddRequestHeader=X-Request-Gateway, SpringCloudGateway
        - id: order-service
          uri: http://localhost:8082
          predicates:
            - Path=/orders/**


---

## 주요 설정 설명

* **routes.id**: 라우트 식별자
* **uri**: 요청을 전달할 서비스 주소 (http, lb 등 사용 가능)
* **predicates**: 요청 조건 (경로, 헤더, 메서드 등)
* **filters**: 요청/응답 전후 처리를 위한 필터 (인증, 헤더 추가, CORS 등)

---

## Predicate 종류 예시

| Predicate                  | 설명            |
|----------------------------|---------------|
| `Path=/api/**`             | URL 경로 기반 라우팅 |
| `Method=GET`               | HTTP 메서드 기반   |
| `Header=X-Request-Id, \d+` | 헤더 조건         |
| `Host=**.example.com`      | 호스트 이름 기반     |

---

## Filter 예시

```yaml
filters:
  - AddRequestHeader=X-Request-Gateway, Gateway
  - RewritePath=/api/(?<segment>.*), /$\{segment}
  - RequestRateLimiter:
      redis-rate-limiter.replenishRate: 10
      redis-rate-limiter.burstCapacity: 20
```

---

## 고급 기능

### 1. 서킷 브레이커 (Resilience4j 연동)

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: myCircuitBreaker
      fallbackUri: forward:/fallback
```

### 2. CORS 설정

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "http://localhost:3000"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
```

---

## 장점

* Spring 생태계와의 강력한 통합
* 선언적 라우팅과 필터 설정
* 가볍고 빠른 퍼포먼스
* 마이크로서비스간 공통 처리 분리 가능

---

## 단점 및 고려사항

* 복잡한 인증/인가 로직 처리에는 추가 구성 필요 (ex. Keycloak, OAuth2 서버)
* 필터 체인의 관리가 복잡해질 수 있음
* 높은 요청 처리량 환경에서는 성능 튜닝 필요

---

## 참고 링크

* 공식 문서: [https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/)
* GitHub: [https://github.com/spring-cloud/spring-cloud-gateway](https://github.com/spring-cloud/spring-cloud-gateway)
