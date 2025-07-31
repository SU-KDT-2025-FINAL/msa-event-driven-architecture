# Kubernetes 클러스터 (EKS) 가이드

## 1. 쿠버네티스 클러스터란?

### 1.1 정의
**쿠버네티스 클러스터**는 여러 서버(노드)를 하나로 묶어서 컨테이너 애플리케이션을 관리하는 플랫폼입니다.

### 1.2 EKS (Elastic Kubernetes Service)
**EKS**는 AWS에서 제공하는 **관리형 쿠버네티스 서비스**입니다. AWS가 쿠버네티스 마스터 노드를 관리해주고, 사용자는 워커 노드만 관리하면 됩니다.

### 1.3 클러스터 구조
```
EKS 클러스터
├── 마스터 노드 (AWS 관리)
│   ├── API Server
│   ├── etcd (데이터 저장소)
│   ├── Scheduler (스케줄링)
│   └── Controller Manager
│
└── 워커 노드 (사용자 관리)
    ├── Node 1 (EC2 인스턴스)
    │   ├── Pod 1 (user-service)
    │   ├── Pod 2 (location-service)
    │   └── Pod 3 (place-service)
    │
    ├── Node 2 (EC2 인스턴스)
    │   ├── Pod 4 (game-service)
    │   ├── Pod 5 (meeting-service)
    │   └── Pod 6 (notification-service)
    │
    └── Node 3 (EC2 인스턴스)
        ├── Pod 7 (user-service)
        ├── Pod 8 (location-service)
        └── Pod 9 (place-service)
```

## 2. Meeting Place 프로젝트에서의 역할

### 2.1 MSA 서비스 배포
```yaml
# k8s/deployments/user-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: meeting-place
spec:
  replicas: 3  # 3개의 Pod 실행
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: meeting-place/user-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: host
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

### 2.2 서비스 간 통신
```yaml
# k8s/services/user-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: meeting-place
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # 클러스터 내부 통신
```

### 2.3 로드 밸런싱
```yaml
# k8s/ingress/meeting-place-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: meeting-place-ingress
  namespace: meeting-place
  annotations:
    kubernetes.io/ingress.class: "alb"
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/target-type: "ip"
spec:
  rules:
  - host: api.meeting-place.com
    http:
      paths:
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /api/locations
        pathType: Prefix
        backend:
          service:
            name: location-service
            port:
              number: 80
      - path: /api/places
        pathType: Prefix
        backend:
          service:
            name: place-service
            port:
              number: 80
```

## 3. EKS 클러스터 구성

### 3.1 Terraform으로 EKS 생성
```hcl
# infrastructure/terraform/eks.tf
resource "aws_eks_cluster" "main" {
  name = "meeting-place-${var.environment}"
  role_arn = aws_iam_role.eks_cluster.arn
  version = "1.27"
  
  vpc_config {
    subnet_ids = concat(aws_subnet.private[*].id, aws_subnet.public[*].id)
    endpoint_private_access = true
    endpoint_public_access = true
  }
  
  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_iam_role_policy_attachment.eks_vpc_resource_controller,
  ]
  
  tags = {
    Name = "meeting-place-eks"
    Environment = var.environment
  }
}

# EKS 노드 그룹
resource "aws_eks_node_group" "main" {
  cluster_name = aws_eks_cluster.main.name
  node_group_name = "meeting-place-node-group"
  node_role_arn = aws_iam_role.eks_nodes.arn
  subnet_ids = aws_subnet.private[*].id
  
  scaling_config {
    desired_size = var.environment == "production" ? 3 : 2
    max_size = var.environment == "production" ? 5 : 3
    min_size = var.environment == "production" ? 2 : 1
  }
  
  instance_types = ["t3.medium"]
  
  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ec2_container_registry_read_only,
  ]
  
  tags = {
    Name = "meeting-place-node-group"
    Environment = var.environment
  }
}
```

### 3.2 환경별 클러스터 구성
```hcl
# 개발 환경
locals {
  dev_config = {
    cluster_name = "meeting-place-dev"
    node_count = 2
    instance_type = "t3.medium"
    node_disk_size = 20
  }
  
  staging_config = {
    cluster_name = "meeting-place-staging"
    node_count = 3
    instance_type = "t3.large"
    node_disk_size = 50
  }
  
  production_config = {
    cluster_name = "meeting-place-prod"
    node_count = 5
    instance_type = "t3.large"
    node_disk_size = 100
  }
}
```

## 4. MSA 서비스 배포 전략

### 4.1 서비스별 배포 구성
```yaml
# k8s/namespaces/meeting-place.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: meeting-place
  labels:
    name: meeting-place
    environment: production
```

```yaml
# k8s/configmaps/application-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
  namespace: meeting-place
data:
  SPRING_PROFILES_ACTIVE: "production"
  KAFKA_BOOTSTRAP_SERVERS: "kafka-service:9092"
  REDIS_HOST: "redis-service"
  ELASTICSEARCH_HOST: "elasticsearch-service"
```

### 4.2 시크릿 관리
```yaml
# k8s/secrets/database-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: meeting-place
type: Opaque
data:
  username: bWVldGluZ19wbGFjZQ==  # base64 encoded
  password: cGFzc3dvcmQxMjM=      # base64 encoded
  host: cmRzLW1lZXRpbmctcGxhY2U=  # base64 encoded
```

### 4.3 서비스 디스커버리
```yaml
# k8s/services/all-services.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: meeting-place
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: location-service
  namespace: meeting-place
spec:
  selector:
    app: location-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: place-service
  namespace: meeting-place
spec:
  selector:
    app: place-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

## 5. 자동 스케일링

### 5.1 HPA (Horizontal Pod Autoscaler)
```yaml
# k8s/hpa/user-service-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: meeting-place
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
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
```

### 5.2 VPA (Vertical Pod Autoscaler)
```yaml
# k8s/vpa/user-service-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: user-service-vpa
  namespace: meeting-place
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1
        memory: 500Mi
      controlledValues: RequestsAndLimits
```

## 6. 모니터링 및 로깅

### 6.1 Prometheus 메트릭 수집
```yaml
# k8s/monitoring/prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### 6.2 Grafana 대시보드
```yaml
# k8s/monitoring/grafana-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard
  namespace: monitoring
data:
  meeting-place-dashboard.json: |
    {
      "dashboard": {
        "title": "Meeting Place MSA Dashboard",
        "panels": [
          {
            "title": "Pod CPU Usage",
            "type": "graph",
            "targets": [
              {
                "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"meeting-place\"}[5m])) by (pod)"
              }
            ]
          },
          {
            "title": "Pod Memory Usage",
            "type": "graph",
            "targets": [
              {
                "expr": "sum(container_memory_usage_bytes{namespace=\"meeting-place\"}) by (pod)"
              }
            ]
          }
        ]
      }
    }
```

## 7. 보안

### 7.1 네트워크 정책
```yaml
# k8s/network-policies/default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: meeting-place
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```yaml
# k8s/network-policies/allow-internal.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal
  namespace: meeting-place
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: meeting-place
    ports:
    - protocol: TCP
      port: 8080
```

### 7.2 RBAC (Role-Based Access Control)
```yaml
# k8s/rbac/service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: meeting-place-sa
  namespace: meeting-place
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: meeting-place-role
  namespace: meeting-place
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: meeting-place-role-binding
  namespace: meeting-place
subjects:
- kind: ServiceAccount
  name: meeting-place-sa
  namespace: meeting-place
roleRef:
  kind: Role
  name: meeting-place-role
  apiGroup: rbac.authorization.k8s.io
```

## 8. CI/CD 파이프라인

### 8.1 GitHub Actions + ArgoCD
```yaml
# .github/workflows/deploy.yml
name: Deploy to EKS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-2
        
    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name meeting-place-production
      
    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f k8s/namespaces/
        kubectl apply -f k8s/configmaps/
        kubectl apply -f k8s/secrets/
        kubectl apply -f k8s/deployments/
        kubectl apply -f k8s/services/
        kubectl apply -f k8s/ingress/
```

### 8.2 ArgoCD 애플리케이션
```yaml
# argocd/applications/meeting-place.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: meeting-place
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/meeting-place
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: meeting-place
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

## 9. 비용 최적화

### 9.1 Spot 인스턴스 활용
```hcl
# infrastructure/terraform/eks-spot.tf
resource "aws_eks_node_group" "spot" {
  cluster_name = aws_eks_cluster.main.name
  node_group_name = "meeting-place-spot"
  node_role_arn = aws_iam_role.eks_nodes.arn
  subnet_ids = aws_subnet.private[*].id
  
  instance_types = ["t3.medium", "t3.large"]  # 여러 인스턴스 타입
  
  capacity_type = "SPOT"  # Spot 인스턴스 사용
  
  scaling_config {
    desired_size = 2
    max_size = 5
    min_size = 1
  }
}
```

### 9.2 클러스터 오토스케일러
```yaml
# k8s/cluster-autoscaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      containers:
      - name: cluster-autoscaler
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.27.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/meeting-place
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```

## 10. 장애 복구

### 10.1 다중 AZ 배포
```yaml
# k8s/deployments/multi-az.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: meeting-place
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - user-service
              topologyKey: kubernetes.io/hostname
      containers:
      - name: user-service
        image: meeting-place/user-service:latest
```

### 10.2 백업 및 복구
```yaml
# k8s/backup/velero-backup.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: meeting-place-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 매일 새벽 2시
  template:
    includedNamespaces:
    - meeting-place
    includedResources:
    - deployments
    - services
    - configmaps
    - secrets
    storageLocation: default
    volumeSnapshotLocations:
    - default
```

## 11. 결론

### 11.1 EKS 클러스터의 장점
- **자동화**: 서비스 배포, 스케일링, 장애 복구 자동화
- **확장성**: 트래픽에 따른 자동 스케일링
- **안정성**: 다중 AZ, 자동 복구
- **관리 편의성**: AWS가 마스터 노드 관리
- **비용 효율성**: Spot 인스턴스, 자동 스케일링

### 11.2 Meeting Place 프로젝트 적용 효과
1. **MSA 서비스 관리**: 각 마이크로서비스를 독립적으로 배포/관리
2. **트래픽 대응**: 사용자 증가에 따른 자동 스케일링
3. **장애 대응**: Pod 장애 시 자동 복구
4. **개발 효율성**: CI/CD 파이프라인으로 자동 배포

### 11.3 운영 시 고려사항
- **모니터링**: Prometheus + Grafana로 리소스 모니터링
- **로깅**: 중앙화된 로그 수집 및 분석
- **보안**: 네트워크 정책, RBAC 설정
- **비용 관리**: Spot 인스턴스, 자동 스케일링 최적화 