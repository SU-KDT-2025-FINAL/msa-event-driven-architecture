# 🍽️ Do Eat – 공동 배달 매칭 플랫폼 기술서

## 1. 개요

**Do Eat**은 배달비와 최소 주문 금액에 부담을 느끼는 사람들을 위해,  
**같은 지역, 같은 시간에 식사하려는 사람들끼리 배달 주문을 공동으로 진행**할 수 있도록 돕는 서비스입니다.

---

## 2. 문제 인식

| 문제 | 설명 |
|------|------|
| 💸 배달비 부담 | 1인 주문 시 평균 3,000~5,000원의 배달비 발생 |
| ⚖️ 최소 주문 금액 문제 | 최소 주문 15,000원 등 기준을 혼자서 충족하기 어려움 |
| 🥲 혼밥 스트레스 | 매번 혼자 밥을 먹는 외로움, 식사 만족도 저하 |

---

## 3. 핵심 타겟

- **🎓 대학생**
    - 기숙사/자취방 등 인접 지역에서 공동주문 수요 높음
    - 강의 시간 등 생활 패턴이 비슷함 → 매칭 용이
- **🏢 아파트 단지 거주자**
    - 같은 단지 내 이웃과의 공동주문으로 배달비 절약
- **👤 1인 가구 직장인**
    - 식사 시간에 맞춰 빠르게 매칭
- **👥 소셜 활동 선호 사용자**
    - 새로운 사람과의 연결을 긍정적으로 여김

---

## 4. 서비스 핵심 기능

### 4.1 🧑‍🤝‍🧑 공동 주문 파티 매칭
- 위치 기반 사용자 매칭
- 동일 상점/메뉴 선호도 기준 파티 생성 및 참여

### 4.2 📍 위치 기반 매칭
- GPS 또는 단지/학교 단위 선택
- 가까운 사용자 우선 정렬

### 4.3 💰 주문 금액 및 배달비 자동 분배
- 총 금액 계산 → 인원수로 자동 분할
- 배달비 포함 여부 표시

### 4.4 💬 채팅 및 소통 기능
- 실시간 채팅 기능으로 세부 조율 가능
- 닉네임 기반 비공개 채팅 (익명성 유지)

### 4.5 🔔 실시간 알림
- 참여 인원, 주문 시간 도달 등 이벤트 알림

### 4.6 📆 에브리타임 시간표 연동 _(추가기능)_
- 사용자의 **강의 시간표 데이터를 기반으로**
- 공강 시간대 기준 자동 추천 매칭 기능
- 사용자가 설정한 **선호 시간대**와 연동 가능

---

## 5. 아키텍처 (MSA 기반)

| 서비스           | 역할                    |
|---------------|-----------------------|
| 👤 사용자 서비스    | 회원가입, 로그인, 프로필, 선호 지역 |
| 🛍️ 가게/메뉴 서비스 | 상점 목록 관리, 메뉴 정보 제공    |
| 👥 매칭 서비스     | 파티 생성, 참여, 매칭 로직      |
| 💳 정산 서비스     | 주문금액 분배, 결제 모듈        |
| 📍 위치 서비스     | 거리 계산, 위치 기반 필터링      |
| 📨 알림 서비스     | 주문 상태 알림, 푸시/웹소켓      |
| 🧠 추천 서비스     | 시간표 기반 추천 (에브리타임 연동)  |
| 📊 통계/분석 서비스  | 사용자 행동 분석, 피크 시간대 추적  |

> CI/CD 및 배포: `GitOps`, `Docker`, `Kubernetes`, `ArgoCD` 사용  
> 데이터 연동: `REST API`, `OAuth2`, `WebSocket`, `MySQL`, `Redis`

---

## 5.1 CI/CD 파이프라인 아키텍처

### 🚀 추천 CI/CD 도구 조합

| 단계 | 도구 | 역할 | 선택 이유 |
|------|------|------|-----------|
| **소스 관리** | `GitHub` | 코드 저장소 | 팀 협업, PR 리뷰, 브랜치 보호 |
| **빌드/테스트** | `GitHub Actions` | CI 파이프라인 | GitHub 통합, 무료 크레딧, YAML 기반 |
| **컨테이너화** | `Docker` | 이미지 빌드 | 표준화, 멀티스테이지 빌드 |
| **레지스트리** | `GitHub Container Registry` | 이미지 저장 | GitHub 통합, 보안 스캔 |
| **배포** | `ArgoCD` | GitOps 배포 | 선언적 배포, 자동 동기화 |
| **오케스트레이션** | `Kubernetes` | 컨테이너 오케스트레이션 | 확장성, 서비스 메시 지원 |
| **모니터링** | `Prometheus + Grafana` | 메트릭 수집/시각화 | 오픈소스, 커스터마이징 |
| **로깅** | `ELK Stack` | 로그 집계 | 실시간 분석, 검색 기능 |
| **보안** | `Trivy` | 취약점 스캔 | 컨테이너, 코드 보안 검사 |

### 🔄 CI/CD 파이프라인 플로우

```yaml
# .github/workflows/ci-cd.yml 예시
name: Do Eat CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Tests
        run: |
          ./gradlew test
          ./gradlew integrationTest
  
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
  
  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker images
        run: |
          docker build -t ghcr.io/doeat/user-service:${{ github.sha }} ./user-service
          docker push ghcr.io/doeat/user-service:${{ github.sha }}
```

### 🎯 GitOps 배포 전략

#### 환경별 배포 전략
- **Development**: `develop` 브랜치 → 자동 배포
- **Staging**: `main` 브랜치 → 수동 승인 후 배포  
- **Production**: `release/*` 태그 → 수동 승인 + 롤백 준비

#### ArgoCD Application 예시
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: doeat-user-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/doeat/k8s-manifests
    targetRevision: HEAD
    path: user-service
  destination:
    server: https://kubernetes.default.svc
    namespace: doeat-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 📊 모니터링 & 관찰성

#### 메트릭 수집
- **Application Metrics**: Spring Boot Actuator + Micrometer
- **Infrastructure Metrics**: Node Exporter + cAdvisor
- **Business Metrics**: 커스텀 메트릭 (매칭 성공률, 주문 완료율)

#### 로그 전략
- **Structured Logging**: JSON 포맷으로 통일
- **Centralized Logging**: Fluentd → Elasticsearch
- **Log Correlation**: Trace ID 기반 요청 추적

#### 알림 설정
- **Slack Integration**: 배포 성공/실패, 에러 알림
- **PagerDuty**: P0/P1 이슈 시 즉시 알림
- **Grafana Alerts**: 임계값 기반 자동 알림

### 🔒 보안 강화

#### 컨테이너 보안
- **Image Scanning**: Trivy로 취약점 사전 검사
- **Runtime Security**: Falco로 이상 행동 탐지
- **Secrets Management**: Kubernetes Secrets + Sealed Secrets

#### 네트워크 보안
- **Service Mesh**: Istio로 서비스 간 통신 제어
- **Network Policies**: Pod 간 통신 제한
- **TLS**: 모든 서비스 간 mTLS 적용

### 🚀 성능 최적화

#### 빌드 최적화
- **Multi-stage Docker**: 빌드 레이어 최소화
- **Build Cache**: GitHub Actions 캐시 활용
- **Parallel Jobs**: 독립적인 서비스 병렬 빌드

#### 배포 최적화
- **Blue-Green Deployment**: 무중단 배포
- **Rolling Updates**: 점진적 업데이트
- **Health Checks**: Readiness/Liveness Probe

---

## 6. 기대 효과

| 항목        | 내용                        |
|-----------|---------------------------|
| 💸 배달비 절감 | 1/N로 분할 주문으로 효율적인 소비      |
| ⏱ 시간 효율   | 시간표 기반 자동 추천 → 빠른 매칭      |
| 🤝 관계 형성  | 소규모 지역 커뮤니티 활성화 유도        |
| 📊 데이터 활용 | 인기 메뉴, 지역 기반 식사 트렌드 분석 가능 |

---

## 7. 향후 확장 방향

- 🛒 **배달앱 API 연동 (배민/요기요)** → 실주문까지 연동
- 🧭 **타겟별 맞춤 추천 알고리즘** → 개인화 강화
- 🎁 **리워드 시스템 도입** → 파티 참여율 증가 유도
- 🏠 **지역 커뮤니티 채널화** → 아파트, 대학교, 캠퍼스 전용 채널 운영

---

## 8. 사용 기술 스택

- **Frontend**: `React`, `TailwindCSS`, `Vite`, `WebSocket`
- **Backend**: `Spring Boot`, `JPA`, `MySQL`, `Redis`, `OAuth2`
- **DevOps**: `Docker`, `Kubernetes`, `GitHub Actions`, `ArgoCD`
- **기타**: `OpenAPI`, `에브리타임 시간표 크롤러 (or 연동)`, `Kafka (이벤트 처리)`

---

## 9. 간단한 슬로건

> **"같이 시켜, 같이 먹자 – Do Eat!"**

___