# CI/CD 파이프라인 구축 가이드

## 1. GitLab CI/CD 파이프라인

### .gitlab-ci.yml 구성
```yaml
stages:
  - test
  - build
  - security
  - deploy-staging
  - integration-test
  - deploy-production

variables:
  DOCKER_REGISTRY: "registry.gitlab.com"
  DOCKER_IMAGE_TAG: $CI_COMMIT_SHA
  KUBECONFIG_PATH: "/tmp/kubeconfig"
  HELM_CHART_PATH: "./helm-chart"

# 공통 스크립트
.common_scripts: &common_scripts
  - echo "Setting up environment..."
  - export GRADLE_USER_HOME="$CI_PROJECT_DIR/.gradle"
  - export DOCKER_IMAGE_NAME="$DOCKER_REGISTRY/$CI_PROJECT_PATH"

# 테스트 단계
unit-test:
  stage: test
  image: openjdk:17-jdk-alpine
  <<: *common_scripts
  script:
    - ./gradlew clean test jacocoTestReport
  artifacts:
    reports:
      junit: build/test-results/test/TEST-*.xml
      coverage: build/reports/jacoco/test/jacocoTestReport.xml
    paths:
      - build/reports/
    expire_in: 1 week
  coverage: '/Total.*?([0-9]{1,3})%/'
  cache:
    key: gradle-cache
    paths:
      - .gradle/wrapper
      - .gradle/caches

integration-test:
  stage: test
  image: openjdk:17-jdk-alpine
  services:
    - postgres:13-alpine
    - redis:6-alpine
    - confluentinc/cp-kafka:latest
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: test_user
    POSTGRES_PASSWORD: test_password
    SPRING_PROFILES_ACTIVE: integration-test
  <<: *common_scripts
  script:
    - ./gradlew integrationTest
  artifacts:
    reports:
      junit: build/test-results/integrationTest/TEST-*.xml
    paths:
      - build/reports/
    expire_in: 1 week

# 코드 품질 검사
code-quality:
  stage: test
  image: openjdk:17-jdk-alpine
  <<: *common_scripts
  script:
    - ./gradlew sonarqube -Dsonar.projectKey=$CI_PROJECT_NAME -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.login=$SONAR_TOKEN
  only:
    - main
    - develop
    - merge_requests

# 빌드 단계
build-application:
  stage: build
  image: openjdk:17-jdk-alpine
  <<: *common_scripts
  script:
    - ./gradlew clean bootJar
  artifacts:
    paths:
      - build/libs/*.jar
    expire_in: 1 day
  only:
    - main
    - develop
    - tags

build-docker-image:
  stage: build
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  dependencies:
    - build-application
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG .
    - docker build -t $DOCKER_IMAGE_NAME:latest .
    - docker push $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG
    - docker push $DOCKER_IMAGE_NAME:latest
  only:
    - main
    - develop
    - tags

# 보안 스캔
security-scan:
  stage: security
  image: owasp/zap2docker-stable:latest
  script:
    - mkdir -p /zap/wrk
    - zap-baseline.py -t http://staging-api.ecommerce.com -J baseline-report.json
  artifacts:
    reports:
      junit: baseline-report.json
    paths:
      - baseline-report.json
    expire_in: 1 week
  allow_failure: true

container-security-scan:
  stage: security
  image: aquasec/trivy:latest
  dependencies:
    - build-docker-image
  script:
    - trivy image --format template --template "@contrib/junit.tpl" -o trivy-report.xml $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG
    - trivy image $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG
  artifacts:
    reports:
      junit: trivy-report.xml
    paths:
      - trivy-report.xml
    expire_in: 1 week

# 스테이징 배포
deploy-staging:
  stage: deploy-staging
  image: alpine/helm:latest
  dependencies:
    - build-docker-image
  environment:
    name: staging
    url: https://staging-api.ecommerce.com
  script:
    - apk add --no-cache curl
    - curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    - chmod +x kubectl && mv kubectl /usr/local/bin/
    - echo $KUBE_CONFIG_STAGING | base64 -d > $KUBECONFIG_PATH
    - export KUBECONFIG=$KUBECONFIG_PATH
    - helm upgrade --install $CI_PROJECT_NAME-staging $HELM_CHART_PATH
        --namespace ecommerce-staging
        --create-namespace
        --set image.tag=$DOCKER_IMAGE_TAG
        --set environment=staging
        --set replicas=2
        --set resources.requests.memory=512Mi
        --set resources.requests.cpu=250m
        --wait --timeout=600s
  only:
    - develop
    - main

# 통합 테스트 (스테이징 환경)
staging-integration-test:
  stage: integration-test
  image: postman/newman:alpine
  dependencies:
    - deploy-staging
  environment:
    name: staging
    url: https://staging-api.ecommerce.com
  script:
    - newman run postman/integration-tests.json 
        --environment postman/staging-environment.json
        --reporters junit,cli
        --reporter-junit-export integration-test-results.xml
  artifacts:
    reports:
      junit: integration-test-results.xml
    paths:
      - integration-test-results.xml
    expire_in: 1 week
  only:
    - develop
    - main

# 성능 테스트
performance-test:
  stage: integration-test
  image: loadimpact/k6:latest
  dependencies:
    - deploy-staging
  environment:
    name: staging
    url: https://staging-api.ecommerce.com
  script:
    - k6 run --out json=performance-results.json performance-tests/load-test.js
  artifacts:
    paths:
      - performance-results.json
    expire_in: 1 week
  allow_failure: true
  only:
    - develop
    - main

# 프로덕션 배포 (수동 승인)
deploy-production:
  stage: deploy-production
  image: alpine/helm:latest
  dependencies:
    - build-docker-image
  environment:
    name: production
    url: https://api.ecommerce.com
  when: manual
  script:
    - apk add --no-cache curl
    - curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    - chmod +x kubectl && mv kubectl /usr/local/bin/
    - echo $KUBE_CONFIG_PRODUCTION | base64 -d > $KUBECONFIG_PATH
    - export KUBECONFIG=$KUBECONFIG_PATH
    # Blue-Green 배포 전략
    - ./scripts/blue-green-deploy.sh $CI_PROJECT_NAME $DOCKER_IMAGE_TAG production
  only:
    - main
    - tags

# 카나리 배포 (Argo Rollouts)
deploy-canary:
  stage: deploy-production
  image: argoproj/argo-rollouts:stable
  dependencies:
    - build-docker-image
  environment:
    name: production-canary
    url: https://api.ecommerce.com
  script:
    - echo $KUBE_CONFIG_PRODUCTION | base64 -d > $KUBECONFIG_PATH
    - export KUBECONFIG=$KUBECONFIG_PATH
    - kubectl argo rollouts set image $CI_PROJECT_NAME=$DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG -n ecommerce-production
    - kubectl argo rollouts promote $CI_PROJECT_NAME -n ecommerce-production
  when: manual
  only:
    - main

# 롤백
rollback-production:
  stage: deploy-production
  image: alpine/helm:latest
  environment:
    name: production
    url: https://api.ecommerce.com
  script:
    - echo $KUBE_CONFIG_PRODUCTION | base64 -d > $KUBECONFIG_PATH
    - export KUBECONFIG=$KUBECONFIG_PATH
    - helm rollback $CI_PROJECT_NAME -n ecommerce-production
  when: manual
  only:
    - main
```

### Blue-Green 배포 스크립트
```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

set -e

SERVICE_NAME=$1
IMAGE_TAG=$2
NAMESPACE=$3

echo "Starting Blue-Green deployment for $SERVICE_NAME:$IMAGE_TAG in $NAMESPACE"

# 현재 활성 색상 확인
CURRENT_COLOR=$(kubectl get service $SERVICE_NAME -n $NAMESPACE -o jsonpath='{.spec.selector.color}' 2>/dev/null || echo "blue")

if [ "$CURRENT_COLOR" = "blue" ]; then
    NEW_COLOR="green"
else
    NEW_COLOR="blue"
fi

echo "Current color: $CURRENT_COLOR, New color: $NEW_COLOR"

# 새로운 색상의 배포 업데이트
helm upgrade --install $SERVICE_NAME-$NEW_COLOR ./helm-chart \
    --namespace $NAMESPACE \
    --set image.tag=$IMAGE_TAG \
    --set color=$NEW_COLOR \
    --set service.enabled=false \
    --wait --timeout=600s

# 헬스체크
echo "Performing health check on $NEW_COLOR deployment..."
kubectl wait --for=condition=ready pod -l app=$SERVICE_NAME,color=$NEW_COLOR -n $NAMESPACE --timeout=300s

# 새로운 배포의 헬스체크
NEW_POD=$(kubectl get pods -l app=$SERVICE_NAME,color=$NEW_COLOR -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD -n $NAMESPACE -- wget --spider --timeout=10 http://localhost:8080/actuator/health

echo "Health check passed. Switching traffic to $NEW_COLOR..."

# 트래픽 전환
kubectl patch service $SERVICE_NAME -n $NAMESPACE -p '{"spec":{"selector":{"color":"'$NEW_COLOR'"}}}'

# 이전 색상의 배포 제거 (5분 후)
echo "Waiting 5 minutes before removing $CURRENT_COLOR deployment..."
sleep 300

helm uninstall $SERVICE_NAME-$CURRENT_COLOR -n $NAMESPACE || true

echo "Blue-Green deployment completed successfully!"
```

## 2. GitHub Actions 파이프라인

### 워크플로우 구성
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  release:
    types: [ published ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:6
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Cache Gradle packages
      uses: actions/cache@v3
      with:
        path: |
          ~/.gradle/caches
          ~/.gradle/wrapper
        key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
        restore-keys: |
          ${{ runner.os }}-gradle-

    - name: Run tests
      run: ./gradlew clean test jacocoTestReport
      env:
        SPRING_PROFILES_ACTIVE: test
        SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/test_db
        SPRING_DATASOURCE_USERNAME: postgres
        SPRING_DATASOURCE_PASSWORD: postgres
        SPRING_REDIS_HOST: localhost

    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results
        path: build/test-results/test/TEST-*.xml

    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: build/reports/jacoco/test/jacocoTestReport.xml
        name: coverage-report
        fail_ci_if_error: true

  security-scan:
    runs-on: ubuntu-latest
    needs: test
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'

  build:
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop' || github.event_name == 'release'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Build application
      run: ./gradlew clean bootJar

    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}

    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        platforms: linux/amd64,linux/arm64

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging-api.ecommerce.com
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure kubectl
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}

    - name: Deploy to staging
      uses: azure/k8s-deploy@v1
      with:
        manifests: |
          k8s/staging/
        images: |
          ${{ needs.build.outputs.image-tag }}
        namespace: ecommerce-staging

    - name: Run integration tests
      run: |
        npm install -g newman
        newman run postman/integration-tests.json \
          --environment postman/staging-environment.json \
          --reporters cli,junit \
          --reporter-junit-export integration-test-results.xml

    - name: Upload integration test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: integration-test-results
        path: integration-test-results.xml

  deploy-production:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main' || github.event_name == 'release'
    environment:
      name: production
      url: https://api.ecommerce.com
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure kubectl
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG_PRODUCTION }}

    - name: Deploy with Argo Rollouts
      run: |
        kubectl argo rollouts set image order-service \
          order-service=${{ needs.build.outputs.image-tag }} \
          -n ecommerce-production
        
        kubectl argo rollouts promote order-service -n ecommerce-production
        
        kubectl argo rollouts status order-service -n ecommerce-production

  notify:
    runs-on: ubuntu-latest
    needs: [deploy-staging, deploy-production]
    if: always()
    steps:
    - name: Notify Slack
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        username: 'GitHub Actions'
        text: |
          Deployment Status: ${{ job.status }}
          Repository: ${{ github.repository }}
          Branch: ${{ github.ref }}
          Commit: ${{ github.sha }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 재사용 가능한 워크플로우
```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
      namespace:
        required: true
        type: string
    secrets:
      KUBE_CONFIG:
        required: true
      SLACK_WEBHOOK_URL:
        required: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure kubectl
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG }}

    - name: Deploy application
      run: |
        helm upgrade --install order-service ./helm-chart \
          --namespace ${{ inputs.namespace }} \
          --create-namespace \
          --set image.tag=${{ inputs.image-tag }} \
          --set environment=${{ inputs.environment }} \
          --wait --timeout=600s

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/order-service \
          -n ${{ inputs.namespace }} --timeout=600s
        
        kubectl get pods -n ${{ inputs.namespace }} -l app=order-service

    - name: Run health check
      run: |
        kubectl wait --for=condition=ready pod \
          -l app=order-service \
          -n ${{ inputs.namespace }} \
          --timeout=300s
        
        # 서비스 헬스체크
        SERVICE_URL=$(kubectl get service order-service \
          -n ${{ inputs.namespace }} \
          -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        
        curl -f http://$SERVICE_URL:8080/actuator/health || exit 1

    - name: Notify on failure
      if: failure() && secrets.SLACK_WEBHOOK_URL
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        channel: '#alerts'
        text: |
          🚨 Deployment failed!
          Environment: ${{ inputs.environment }}
          Image: ${{ inputs.image-tag }}
          Repository: ${{ github.repository }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 3. 마이크로서비스별 독립 배포

### 모노레포 구조에서의 개별 서비스 배포
```yaml
# .github/workflows/service-deploy.yml
name: Service Specific Deploy

on:
  push:
    paths:
      - 'services/user-service/**'
      - 'services/order-service/**'
      - 'services/payment-service/**'
      - 'services/inventory-service/**'

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      user-service: ${{ steps.changes.outputs.user-service }}
      order-service: ${{ steps.changes.outputs.order-service }}
      payment-service: ${{ steps.changes.outputs.payment-service }}
      inventory-service: ${{ steps.changes.outputs.inventory-service }}
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Detect changes
      uses: dorny/paths-filter@v2
      id: changes
      with:
        filters: |
          user-service:
            - 'services/user-service/**'
          order-service:
            - 'services/order-service/**'
          payment-service:
            - 'services/payment-service/**'
          inventory-service:
            - 'services/inventory-service/**'

  deploy-user-service:
    needs: detect-changes
    if: needs.detect-changes.outputs.user-service == 'true'
    uses: ./.github/workflows/service-ci-cd.yml
    with:
      service-name: user-service
      service-path: services/user-service
    secrets: inherit

  deploy-order-service:
    needs: detect-changes
    if: needs.detect-changes.outputs.order-service == 'true'
    uses: ./.github/workflows/service-ci-cd.yml
    with:
      service-name: order-service
      service-path: services/order-service
    secrets: inherit

  deploy-payment-service:
    needs: detect-changes
    if: needs.detect-changes.outputs.payment-service == 'true'
    uses: ./.github/workflows/service-ci-cd.yml
    with:
      service-name: payment-service
      service-path: services/payment-service
    secrets: inherit

  deploy-inventory-service:
    needs: detect-changes
    if: needs.detect-changes.outputs.inventory-service == 'true'
    uses: ./.github/workflows/service-ci-cd.yml
    with:
      service-name: inventory-service
      service-path: services/inventory-service
    secrets: inherit
```

### 서비스별 CI/CD 템플릿
```yaml
# .github/workflows/service-ci-cd.yml
name: Service CI/CD Template

on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
      service-path:
        required: true
        type: string
    secrets:
      REGISTRY_USERNAME:
        required: true
      REGISTRY_PASSWORD:
        required: true
      KUBE_CONFIG_STAGING:
        required: true
      KUBE_CONFIG_PRODUCTION:
        required: true

env:
  REGISTRY: ghcr.io
  SERVICE_NAME: ${{ inputs.service-name }}
  SERVICE_PATH: ${{ inputs.service-path }}

jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ inputs.service-path }}
    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Cache Gradle packages
      uses: actions/cache@v3
      with:
        path: |
          ~/.gradle/caches
          ~/.gradle/wrapper
        key: ${{ runner.os }}-gradle-${{ inputs.service-name }}-${{ hashFiles(format('{0}/**/*.gradle*', inputs.service-path)) }}

    - name: Run tests
      run: ./gradlew clean test

    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: ${{ inputs.service-name }}-test-results
        path: ${{ inputs.service-path }}/build/test-results/

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    defaults:
      run:
        working-directory: ${{ inputs.service-path }}
    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Build application
      run: ./gradlew clean bootJar

    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ github.repository }}/${{ inputs.service-name }}
        tags: |
          type=sha,prefix={{branch}}-
          type=ref,event=branch
          type=semver,pattern={{version}}

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: ${{ inputs.service-path }}
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment:
      name: staging
    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Configure kubectl
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}

    - name: Deploy to staging
      run: |
        helm upgrade --install ${{ inputs.service-name }} \
          ./helm-charts/${{ inputs.service-name }} \
          --namespace ecommerce-staging \
          --create-namespace \
          --set image.tag=${{ needs.build.outputs.image-tag }} \
          --set environment=staging \
          --wait --timeout=600s

  deploy-production:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Configure kubectl
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG_PRODUCTION }}

    - name: Deploy to production
      run: |
        # Argo Rollouts를 사용한 카나리 배포
        kubectl argo rollouts set image ${{ inputs.service-name }} \
          ${{ inputs.service-name }}=${{ needs.build.outputs.image-tag }} \
          -n ecommerce-production
        
        kubectl argo rollouts promote ${{ inputs.service-name }} \
          -n ecommerce-production --full
```

## 4. 컨테이너 이미지 빌드 및 배포 자동화

### 멀티 스테이지 Dockerfile 최적화
```dockerfile
# 개발 단계
FROM eclipse-temurin:17-jdk-alpine AS development
WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle settings.gradle ./
RUN ./gradlew dependencies --no-daemon

# 빌드 단계
FROM development AS builder
COPY src ./src
RUN ./gradlew clean bootJar --no-daemon --parallel

# 테스트 단계
FROM builder AS test
RUN ./gradlew test jacocoTestReport --no-daemon
COPY --from=builder /app/build/reports /test-reports

# 프로덕션 준비 단계
FROM eclipse-temurin:17-jre-alpine AS production-base
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup && \
    apk add --no-cache curl && \
    mkdir -p /app/logs && \
    chown -R appuser:appgroup /app

# 최종 프로덕션 이미지
FROM production-base AS production
WORKDIR /app
COPY --from=builder --chown=appuser:appgroup /app/build/libs/*.jar app.jar

# JVM 튜닝 및 환경 설정
ENV JAVA_OPTS="-XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75.0 \
    -XX:+UseG1GC \
    -XX:+UseStringDeduplication \
    -XX:+PrintGCDetails \
    -XX:+PrintGCTimeStamps \
    -Xloggc:/app/logs/gc.log"

EXPOSE 8080
USER appuser

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### 이미지 캐싱 전략
```yaml
# Docker 빌드 캐시 최적화
build-optimized:
  runs-on: ubuntu-latest
  steps:
  - name: Set up Docker Buildx
    uses: docker/setup-buildx-action@v2
    with:
      driver-opts: |
        image=moby/buildkit:buildx-stable-1

  - name: Build with cache
    uses: docker/build-push-action@v4
    with:
      context: .
      target: production
      platforms: linux/amd64,linux/arm64
      cache-from: type=gha
      cache-to: type=gha,mode=max
      push: true
      tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  - name: Generate SBOM
    uses: anchore/sbom-action@v0
    with:
      image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
      format: spdx-json
      output-file: sbom.spdx.json

  - name: Scan image with Grype
    uses: anchore/scan-action@v3
    with:
      image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
      fail-build: false
      severity-cutoff: high
```

## 5. Blue-Green 및 Canary 배포 전략

### Argo Rollouts 구성
```yaml
# argo-rollouts/order-service-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
  namespace: ecommerce-production
spec:
  replicas: 5
  strategy:
    canary:
      canaryService: order-service-canary
      stableService: order-service-stable
      analysis:
        templates:
        - templateName: success-rate
        - templateName: latency
        startingStep: 2
        args:
        - name: service-name
          value: order-service
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - setWeight: 20
      - pause: {duration: 2m}
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: order-service
      - setWeight: 40
      - pause: {duration: 5m}
      - setWeight: 60
      - pause: {duration: 5m}
      - setWeight: 80
      - pause: {duration: 10m}
      trafficRouting:
        nginx:
          stableIngress: order-service-stable
          annotationPrefix: nginx.ingress.kubernetes.io
          additionalIngressAnnotations:
            canary-by-header: X-Canary
            canary-by-header-value: "true"
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
        image: ghcr.io/ecommerce/order-service:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
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
# Analysis Templates
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: ecommerce-production
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 2m
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(http_requests_total{job="{{args.service-name}}",status!~"5.."}[2m])) /
          sum(rate(http_requests_total{job="{{args.service-name}}"}[2m]))

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency
  namespace: ecommerce-production
spec:
  args:
  - name: service-name
  metrics:
  - name: latency
    interval: 2m
    successCondition: result[0] <= 0.5
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket{job="{{args.service-name}}"}[2m])) by (le)
          )
```

### GitOps with ArgoCD
```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-services
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/ecommerce/k8s-manifests
    targetRevision: HEAD
    path: environments/production
    helm:
      valueFiles:
      - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce-production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

---
# Notification Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} is now running new version.
  template.app-health-degraded: |
    message: |
      {{if eq .serviceType "slack"}}:exclamation:{{end}} Application {{.app.metadata.name}} has degraded.
  template.app-sync-failed: |
    message: |
      {{if eq .serviceType "slack"}}:exclamation:{{end}} Application {{.app.metadata.name}} sync is failed.
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send:
      - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
  trigger.on-health-degraded: |
    - description: Application has degraded
      send:
      - app-health-degraded
      when: app.status.health.status == 'Degraded'
  trigger.on-sync-failed: |
    - description: Application syncing has failed
      send:
      - app-sync-failed
      when: app.status.operationState.phase in ['Error', 'Failed']
  subscriptions: |
    - recipients:
      - slack:deployments
      triggers:
      - on-deployed
      - on-health-degraded
      - on-sync-failed
```

## 실습 과제

1. **완전한 CI/CD 파이프라인**: GitLab 또는 GitHub Actions를 사용한 전체 CI/CD 파이프라인 구축
2. **멀티 서비스 배포**: 모노레포에서 변경 감지 기반 개별 서비스 배포 시스템 구현
3. **Blue-Green 배포**: 무중단 배포를 위한 Blue-Green 배포 전략 구현
4. **카나리 배포**: Argo Rollouts를 활용한 자동화된 카나리 배포 시스템 구축
5. **GitOps 구현**: ArgoCD를 활용한 선언적 GitOps 배포 파이프라인 구현

## 참고 자료

- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Argo Rollouts Documentation](https://argo-rollouts.readthedocs.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)