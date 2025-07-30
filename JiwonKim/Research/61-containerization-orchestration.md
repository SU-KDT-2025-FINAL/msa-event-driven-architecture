# 컨테이너화 및 오케스트레이션 가이드

## 1. Docker 서비스 컨테이너화

### 마이크로서비스 Dockerfile 최적화
```dockerfile
# Multi-stage build for Spring Boot application
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle settings.gradle ./
COPY src ./src

# Gradle 캐시 최적화를 위한 의존성 다운로드
RUN ./gradlew dependencies --no-daemon

# 애플리케이션 빌드
RUN ./gradlew clean bootJar --no-daemon

# Production 이미지
FROM eclipse-temurin:17-jre-alpine

# 보안을 위한 non-root 사용자 생성
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# 애플리케이션 JAR 복사
COPY --from=builder /app/build/libs/*.jar app.jar
COPY --chown=appuser:appgroup scripts/entrypoint.sh /app/entrypoint.sh

# 실행 권한 부여
RUN chmod +x /app/entrypoint.sh

# 헬스체크 추가
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# 포트 노출
EXPOSE 8080

# 사용자 변경
USER appuser

# 애플리케이션 실행
ENTRYPOINT ["/app/entrypoint.sh"]
```

### Docker Compose 전체 시스템 구성
```yaml
version: '3.8'

services:
  # Kafka 클러스터
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log
    networks:
      - ecommerce-network

  kafka-1:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: kafka-1
    volumes:
      - kafka1-data:/var/lib/kafka/data
    networks:
      - ecommerce-network

  kafka-2:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9102
      KAFKA_JMX_HOSTNAME: kafka-2
    volumes:
      - kafka2-data:/var/lib/kafka/data
    networks:
      - ecommerce-network

  kafka-3:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:9094
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9103
      KAFKA_JMX_HOSTNAME: kafka-3
    volumes:
      - kafka3-data:/var/lib/kafka/data
    networks:
      - ecommerce-network

  # Schema Registry
  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka-1:9092,kafka-2:9093,kafka-3:9094
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    networks:
      - ecommerce-network

  # PostgreSQL 데이터베이스
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: ecommerce_user
      POSTGRES_PASSWORD: ecommerce_password
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    networks:
      - ecommerce-network

  # Redis 캐시
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --replica-read-only no
    volumes:
      - redis-data:/data
    networks:
      - ecommerce-network

  # 마이크로서비스들
  user-service:
    build:
      context: ./user-service
      dockerfile: Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/ecommerce
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka-1:9092,kafka-2:9093,kafka-3:9094
      SPRING_REDIS_HOST: redis
    depends_on:
      - postgres
      - kafka-1
      - kafka-2
      - kafka-3
      - redis
    networks:
      - ecommerce-network
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/ecommerce
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka-1:9092,kafka-2:9093,kafka-3:9094
      SPRING_REDIS_HOST: redis
    depends_on:
      - postgres
      - kafka-1
      - kafka-2
      - kafka-3
      - redis
    networks:
      - ecommerce-network
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  payment-service:
    build:
      context: ./payment-service
      dockerfile: Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/ecommerce
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka-1:9092,kafka-2:9093,kafka-3:9094
    depends_on:
      - postgres
      - kafka-1
      - kafka-2
      - kafka-3
    networks:
      - ecommerce-network
    deploy:
      replicas: 2

  inventory-service:
    build:
      context: ./inventory-service
      dockerfile: Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/ecommerce
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka-1:9092,kafka-2:9093,kafka-3:9094
      SPRING_REDIS_HOST: redis
    depends_on:
      - postgres
      - kafka-1
      - kafka-2
      - kafka-3
      - redis
    networks:
      - ecommerce-network
    deploy:
      replicas: 2

  # API Gateway
  api-gateway:
    build:
      context: ./api-gateway
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      USER_SERVICE_URL: http://user-service:8080
      ORDER_SERVICE_URL: http://order-service:8080
      PAYMENT_SERVICE_URL: http://payment-service:8080
      INVENTORY_SERVICE_URL: http://inventory-service:8080
    depends_on:
      - user-service
      - order-service
      - payment-service
      - inventory-service
    networks:
      - ecommerce-network

  # 모니터링 스택
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - ecommerce-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - ecommerce-network

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka1-data:
  kafka2-data:
  kafka3-data:
  postgres-data:
  redis-data:
  prometheus-data:
  grafana-data:

networks:
  ecommerce-network:
    driver: bridge
```

## 2. Kubernetes MSA 배포

### Namespace 및 기본 리소스
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    name: ecommerce

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ecommerce
data:
  application.yml: |
    spring:
      profiles:
        active: kubernetes
      kafka:
        bootstrap-servers: kafka-cluster:9092
        producer:
          acks: 1
          retries: 3
        consumer:
          group-id: ${spring.application.name}
          auto-offset-reset: earliest
      datasource:
        url: jdbc:postgresql://postgres-service:5432/ecommerce
        username: ${DB_USERNAME}
        password: ${DB_PASSWORD}
      redis:
        host: redis-service
        port: 6379
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
      metrics:
        export:
          prometheus:
            enabled: true

---
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: ecommerce
type: Opaque
data:
  db-username: ZWNvbW1lcmNlX3VzZXI=  # ecommerce_user
  db-password: ZWNvbW1lcmNlX3Bhc3N3b3Jk  # ecommerce_password
  jwt-secret: bXlfc3VwZXJfc2VjcmV0X2p3dF9rZXk=
```

### Kafka 클러스터 배포
```yaml
# kafka-cluster.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: ecommerce
spec:
  serviceName: kafka-headless
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.4.0
        ports:
        - containerPort: 9092
        - containerPort: 9101
        env:
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['kafka.apache.org/broker-id']
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: "zookeeper-service:2181"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(POD_NAME).kafka-headless.ecommerce.svc.cluster.local:9092"
        - name: KAFKA_LISTENERS
          value: "PLAINTEXT://0.0.0.0:9092"
        - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
          value: "3"
        - name: KAFKA_TRANSACTION_STATE_LOG_MIN_ISR
          value: "2"
        - name: KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR
          value: "3"
        - name: KAFKA_JMX_PORT
          value: "9101"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        volumeMounts:
        - name: kafka-storage
          mountPath: /var/lib/kafka/data
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "kafka-broker-api-versions --bootstrap-server localhost:9092"
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "kafka-broker-api-versions --bootstrap-server localhost:9092"
          initialDelaySeconds: 60
          periodSeconds: 30
  volumeClaimTemplates:
  - metadata:
      name: kafka-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi

---
apiVersion: v1
kind: Service
metadata:
  name: kafka-cluster
  namespace: ecommerce
spec:
  selector:
    app: kafka
  ports:
  - port: 9092
    targetPort: 9092
  type: ClusterIP

---
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: ecommerce
spec:
  selector:
    app: kafka
  ports:
  - port: 9092
    targetPort: 9092
  clusterIP: None
```

### 마이크로서비스 배포
```yaml
# order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
  labels:
    app: order-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: v1
  template:
    metadata:
      labels:
        app: order-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
      - name: order-service
        image: ecommerce/order-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: kubernetes
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
  labels:
    app: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 8080
    targetPort: 8080
    name: http
  type: ClusterIP

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
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
      selectPolicy: Max
```

## 3. Helm Chart 배포 자동화

### Chart 구조
```
ecommerce-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── configmap.yaml
│   ├── secrets.yaml
│   ├── services/
│   │   ├── user-service.yaml
│   │   ├── order-service.yaml
│   │   ├── payment-service.yaml
│   │   └── inventory-service.yaml
│   ├── infrastructure/
│   │   ├── kafka.yaml
│   │   ├── postgres.yaml
│   │   └── redis.yaml
│   └── ingress.yaml
└── charts/
    ├── kafka/
    ├── postgresql/
    └── redis/
```

### Chart.yaml
```yaml
apiVersion: v2
name: ecommerce
description: Event-driven microservices e-commerce platform
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
- name: postgresql
  version: 12.1.2
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
- name: redis
  version: 17.4.3
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled
- name: kafka
  version: 20.0.1
  repository: https://charts.bitnami.com/bitnami
  condition: kafka.enabled

maintainers:
- name: DevOps Team
  email: devops@ecommerce.com
```

### values.yaml
```yaml
# Global settings
global:
  namespace: ecommerce
  imageRegistry: "your-registry.com"
  imagePullSecrets:
    - name: registry-secret

# Application configuration
app:
  name: ecommerce
  version: "1.0.0"
  
# Database configuration
postgresql:
  enabled: true
  auth:
    postgresPassword: "postgres-password"
    username: "ecommerce_user"
    password: "ecommerce_password"
    database: "ecommerce"
  primary:
    persistence:
      enabled: true
      size: 20Gi
      storageClass: "fast-ssd"

# Cache configuration
redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      enabled: true
      size: 8Gi
      storageClass: "fast-ssd"

# Message queue configuration
kafka:
  enabled: true
  replicaCount: 3
  zookeeper:
    replicaCount: 3
  persistence:
    enabled: true
    size: 20Gi
    storageClass: "fast-ssd"

# Microservices configuration
services:
  userService:
    enabled: true
    replicaCount: 2
    image:
      repository: ecommerce/user-service
      tag: "1.0.0"
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "500m"
    autoscaling:
      enabled: true
      minReplicas: 2
      maxReplicas: 8
      targetCPUUtilizationPercentage: 70

  orderService:
    enabled: true
    replicaCount: 3
    image:
      repository: ecommerce/order-service
      tag: "1.0.0"
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "500m"
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70

  paymentService:
    enabled: true
    replicaCount: 2
    image:
      repository: ecommerce/payment-service
      tag: "1.0.0"
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "500m"
    autoscaling:
      enabled: true
      minReplicas: 2
      maxReplicas: 6
      targetCPUUtilizationPercentage: 70

  inventoryService:
    enabled: true
    replicaCount: 2
    image:
      repository: ecommerce/inventory-service
      tag: "1.0.0"
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "500m"
    autoscaling:
      enabled: true
      minReplicas: 2
      maxReplicas: 6
      targetCPUUtilizationPercentage: 70

# Ingress configuration
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: api.ecommerce.com
      paths:
        - path: /users
          pathType: Prefix
          service: user-service
        - path: /orders
          pathType: Prefix
          service: order-service
        - path: /payments
          pathType: Prefix
          service: payment-service
        - path: /inventory
          pathType: Prefix
          service: inventory-service
  tls:
    - secretName: ecommerce-tls
      hosts:
        - api.ecommerce.com

# Monitoring
monitoring:
  prometheus:
    enabled: true
  grafana:
    enabled: true
  jaeger:
    enabled: true

# Security
security:
  networkPolicies:
    enabled: true
  podSecurityPolicy:
    enabled: true
```

### 서비스 템플릿
```yaml
# templates/services/order-service.yaml
{{- if .Values.services.orderService.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "ecommerce.fullname" . }}-order-service
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "ecommerce.labels" . | nindent 4 }}
    app.kubernetes.io/component: order-service
spec:
  replicas: {{ .Values.services.orderService.replicaCount }}
  selector:
    matchLabels:
      {{- include "ecommerce.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: order-service
  template:
    metadata:
      labels:
        {{- include "ecommerce.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      {{- with .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: order-service
        image: "{{ .Values.global.imageRegistry }}/{{ .Values.services.orderService.image.repository }}:{{ .Values.services.orderService.image.tag }}"
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:postgresql://{{ include "ecommerce.fullname" . }}-postgresql:5432/{{ .Values.postgresql.auth.database }}"
        - name: SPRING_DATASOURCE_USERNAME
          value: "{{ .Values.postgresql.auth.username }}"
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "ecommerce.fullname" . }}-postgresql
              key: password
        - name: SPRING_KAFKA_BOOTSTRAP_SERVERS
          value: "{{ include "ecommerce.fullname" . }}-kafka:9092"
        - name: SPRING_REDIS_HOST
          value: "{{ include "ecommerce.fullname" . }}-redis-master"
        resources:
          {{- toYaml .Values.services.orderService.resources | nindent 10 }}
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: {{ include "ecommerce.fullname" . }}-order-service
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "ecommerce.labels" . | nindent 4 }}
    app.kubernetes.io/component: order-service
spec:
  selector:
    {{- include "ecommerce.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: order-service
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
    name: http

{{- if .Values.services.orderService.autoscaling.enabled }}
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "ecommerce.fullname" . }}-order-service-hpa
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "ecommerce.labels" . | nindent 4 }}
    app.kubernetes.io/component: order-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "ecommerce.fullname" . }}-order-service
  minReplicas: {{ .Values.services.orderService.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.services.orderService.autoscaling.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.services.orderService.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
{{- end }}
```

## 4. Service Mesh (Istio) 도입

### Istio 설치 및 구성
```bash
# Istio 설치
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.19.0
export PATH=$PWD/bin:$PATH

# Istio 설치
istioctl install --set values.defaultRevision=default

# 네임스페이스에 사이드카 자동 주입 활성화
kubectl label namespace ecommerce istio-injection=enabled
```

### Service Mesh 구성
```yaml
# istio-gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: ecommerce-gateway
  namespace: ecommerce
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - api.ecommerce.com
    tls:
      httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: ecommerce-tls
    hosts:
    - api.ecommerce.com

---
# Virtual Service
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ecommerce-vs
  namespace: ecommerce
spec:
  hosts:
  - api.ecommerce.com
  gateways:
  - ecommerce-gateway
  http:
  - match:
    - uri:
        prefix: /users
    route:
    - destination:
        host: user-service
        port:
          number: 8080
    fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
  - match:
    - uri:
        prefix: /orders
    route:
    - destination:
        host: order-service
        port:
          number: 8080
        subset: v1
      weight: 90
    - destination:
        host: order-service
        port:
          number: 8080
        subset: v2
      weight: 10
    timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s

---
# Destination Rules
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-dr
  namespace: ecommerce
spec:
  host: order-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 10
    circuitBreaker:
      consecutiveErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

---
# Security Policy
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ecommerce
spec:
  mtls:
    mode: STRICT

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-authz
  namespace: ecommerce
spec:
  selector:
    matchLabels:
      app: order-service
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/ecommerce/sa/user-service"]
    - source:
        principals: ["cluster.local/ns/ecommerce/sa/payment-service"]
  - to:
    - operation:
        methods: ["GET", "POST", "PUT"]
```

### 카나리 배포 설정
```yaml
# canary-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service-rollout
  namespace: ecommerce
spec:
  replicas: 5
  strategy:
    canary:
      canaryService: order-service-canary
      stableService: order-service-stable
      trafficRouting:
        istio:
          virtualService:
            name: order-service-vs
            routes:
            - primary
          destinationRule:
            name: order-service-dr
            canarySubsetName: canary
            stableSubsetName: stable
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - setWeight: 20
      - pause: {duration: 2m}
      - setWeight: 50
      - pause: {duration: 2m}
      - setWeight: 80
      - pause: {duration: 2m}
      analysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: order-service
        - name: canary-hash
          valueFrom:
            podTemplateHashValue: Latest
      scaleDownDelaySeconds: 30
      abortScaleDownDelaySeconds: 30
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: ecommerce/order-service:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"

---
# Analysis Template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: ecommerce
spec:
  args:
  - name: service-name
  - name: canary-hash
  metrics:
  - name: success-rate
    interval: 2m
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(
            istio_requests_total{
              destination_service_name="{{args.service-name}}",
              destination_version="{{args.canary-hash}}",
              response_code!~"5.*"
            }[2m]
          )) / 
          sum(rate(
            istio_requests_total{
              destination_service_name="{{args.service-name}}",
              destination_version="{{args.canary-hash}}"
            }[2m]
          ))
  - name: avg-response-time
    interval: 2m
    successCondition: result[0] <= 0.5
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          histogram_quantile(0.95,
            sum(rate(
              istio_request_duration_milliseconds_bucket{
                destination_service_name="{{args.service-name}}",
                destination_version="{{args.canary-hash}}"
              }[2m]
            )) by (le)
          ) / 1000
```

## 실습 과제

1. **완전한 컨테이너화**: 전체 마이크로서비스 시스템을 Docker로 컨테이너화하고 Docker Compose로 구성
2. **Kubernetes 배포**: Helm Chart를 사용한 완전한 Kubernetes 배포 파이프라인 구축
3. **Service Mesh 도입**: Istio를 활용한 서비스 간 통신 보안 및 트래픽 관리
4. **카나리 배포**: Argo Rollouts를 사용한 안전한 카나리 배포 전략 구현
5. **모니터링 통합**: Prometheus, Grafana, Jaeger를 통한 통합 모니터링 시스템 구축

## 참고 자료

- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Istio Documentation](https://istio.io/latest/docs/)