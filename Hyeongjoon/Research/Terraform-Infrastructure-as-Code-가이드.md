# Terraform Infrastructure as Code (IaC) 가이드

## 1. Terraform이란?

### 1.1 정의
**Terraform**은 HashiCorp에서 개발한 오픈소스 Infrastructure as Code (IaC) 도구입니다. 클라우드 인프라를 코드로 정의하고 관리할 수 있게 해주는 도구입니다.

### 1.2 핵심 개념
```hcl
# Terraform의 기본 구조
resource "aws_instance" "example" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
  
  tags = {
    Name = "example-instance"
  }
}
```

### 1.3 주요 특징
- **멱등성 (Idempotency)**: 여러 번 실행해도 동일한 결과
- **선언적 (Declarative)**: 원하는 상태를 선언하면 자동으로 구성
- **멀티 클라우드**: AWS, Azure, GCP 등 다양한 클라우드 지원
- **상태 관리**: 현재 인프라 상태를 파일로 관리

## 2. 기존 수동 설정 vs Terraform 자동화

### 2.1 기존 수동 설정 방식
```
1. AWS 콘솔 접속
2. RDS 생성 → 수동으로 설정값 입력
3. VPC 생성 → 서브넷, 라우팅 테이블 수동 설정
4. 보안 그룹 생성 → 규칙 수동 입력
5. EKS 클러스터 생성 → 노드 그룹 수동 설정
6. 환경별로 반복 작업...
```

**문제점:**
- **인간 실수**: 설정값 입력 오류
- **일관성 부족**: 환경별 설정 차이
- **문서화 부족**: 설정 이력 추적 어려움
- **재현 불가능**: 동일한 환경 재생성 어려움

### 2.2 Terraform 자동화 방식
```hcl
# infrastructure/terraform/main.tf
provider "aws" {
  region = "ap-northeast-2"
}

# VPC 생성
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "meeting-place-vpc"
    Environment = var.environment
  }
}

# RDS MySQL 생성
resource "aws_db_instance" "mysql" {
  identifier = "meeting-place-${var.environment}"
  engine = "mysql"
  engine_version = "8.0"
  instance_class = var.db_instance_class
  
  allocated_storage = 20
  storage_type = "gp3"
  
  db_name = "meeting_place"
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  multi_az = var.environment == "production"
  
  tags = {
    Environment = var.environment
    Project = "meeting-place"
  }
}
```

**장점:**
- **자동화**: 코드 실행으로 인프라 자동 생성
- **일관성**: 환경별 동일한 설정 보장
- **버전 관리**: Git으로 변경 이력 추적
- **재현 가능**: 동일한 환경 쉽게 재생성

## 3. Terraform 워크플로우

### 3.1 기본 워크플로우
```bash
# 1. Terraform 초기화
terraform init

# 2. 실행 계획 확인
terraform plan

# 3. 인프라 생성/변경
terraform apply

# 4. 인프라 삭제 (필요시)
terraform destroy
```

### 3.2 Meeting Place 프로젝트 워크플로우
```bash
# 프로젝트 구조
meeting-place/
├── infrastructure/
│   └── terraform/
│       ├── main.tf              # 주요 리소스 정의
│       ├── variables.tf         # 변수 정의
│       ├── outputs.tf           # 출력값 정의
│       ├── providers.tf         # 프로바이더 설정
│       ├── vpc.tf              # VPC 관련 리소스
│       ├── rds.tf              # RDS MySQL 설정
│       ├── eks.tf              # EKS 클러스터 설정
│       ├── redis.tf            # ElastiCache Redis 설정
│       └── terraform.tfvars    # 변수값 설정
```

## 4. GitHub Actions와 Terraform 연동

### 4.1 CI/CD 파이프라인에서의 위치
```
GitHub Repository
├── frontend/                    # React 앱
├── backend/                     # Spring Boot 서비스들
├── infrastructure/
│   └── terraform/              # ← Terraform 코드 위치
├── k8s/                        # Kubernetes 매니페스트
├── .github/
│   └── workflows/
│       ├── ci-pipeline.yml     # 애플리케이션 빌드/테스트
│       ├── infrastructure.yml  # ← Terraform 실행 워크플로우
│       └── deploy.yml          # 애플리케이션 배포
```

### 4.2 GitHub Actions Terraform 워크플로우
```yaml
# .github/workflows/infrastructure.yml
name: Infrastructure Deployment

on:
  push:
    branches: [main]
    paths: ['infrastructure/terraform/**']
  pull_request:
    branches: [main]
    paths: ['infrastructure/terraform/**']

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0
        
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-2
        
    - name: Terraform Init
      working-directory: ./infrastructure/terraform
      run: terraform init
      
    - name: Terraform Format Check
      working-directory: ./infrastructure/terraform
      run: terraform fmt -check
      
    - name: Terraform Plan
      working-directory: ./infrastructure/terraform
      run: terraform plan -out=tfplan
      
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      working-directory: ./infrastructure/terraform
      run: terraform apply tfplan
```

### 4.3 환경별 배포 전략
```yaml
# 환경별 Terraform 워크플로우
jobs:
  terraform-dev:
    runs-on: ubuntu-latest
    environment: development
    steps:
    - name: Terraform Apply (Dev)
      run: |
        cd infrastructure/terraform
        terraform workspace select dev
        terraform apply -var-file=dev.tfvars -auto-approve
      
  terraform-staging:
    runs-on: ubuntu-latest
    environment: staging
    needs: terraform-dev
    steps:
    - name: Terraform Apply (Staging)
      run: |
        cd infrastructure/terraform
        terraform workspace select staging
        terraform apply -var-file=staging.tfvars -auto-approve
        
  terraform-production:
    runs-on: ubuntu-latest
    environment: production
    needs: terraform-staging
    steps:
    - name: Terraform Apply (Production)
      run: |
        cd infrastructure/terraform
        terraform workspace select production
        terraform apply -var-file=production.tfvars -auto-approve
```

## 5. Meeting Place 프로젝트 Terraform 구성

### 5.1 변수 정의
```hcl
# infrastructure/terraform/variables.tf
variable "environment" {
  description = "Environment (dev, staging, production)"
  type = string
}

variable "project_name" {
  description = "Project name"
  type = string
  default = "meeting-place"
}

variable "db_instance_class" {
  description = "RDS instance class"
  type = string
  default = "db.t3.micro"
}

variable "db_username" {
  description = "RDS master username"
  type = string
  sensitive = true
}

variable "db_password" {
  description = "RDS master password"
  type = string
  sensitive = true
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type = string
  default = "10.0.0.0/16"
}
```

### 5.2 VPC 구성
```hcl
# infrastructure/terraform/vpc.tf
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  enable_dns_hostnames = true
  enable_dns_support = true
  
  tags = {
    Name = "${var.project_name}-vpc"
    Environment = var.environment
  }
}

# 퍼블릭 서브넷 (ALB용)
resource "aws_subnet" "public" {
  count = 2
  
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.project_name}-public-${count.index + 1}"
    Environment = var.environment
  }
}

# 프라이빗 서브넷 (EKS 노드용)
resource "aws_subnet" "private" {
  count = 2
  
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "${var.project_name}-private-${count.index + 1}"
    Environment = var.environment
  }
}

# 데이터베이스 서브넷 (RDS용)
resource "aws_subnet" "database" {
  count = 2
  
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.${count.index + 20}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "${var.project_name}-database-${count.index + 1}"
    Environment = var.environment
  }
}
```

### 5.3 RDS MySQL 구성
```hcl
# infrastructure/terraform/rds.tf
resource "aws_db_subnet_group" "main" {
  name = "${var.project_name}-db-subnet-group"
  subnet_ids = aws_subnet.database[*].id
  
  tags = {
    Name = "${var.project_name}-db-subnet-group"
    Environment = var.environment
  }
}

resource "aws_security_group" "rds" {
  name = "${var.project_name}-rds-sg"
  description = "Security group for RDS MySQL"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port = 3306
    to_port = 3306
    protocol = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }
  
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-rds-sg"
    Environment = var.environment
  }
}

resource "aws_db_instance" "mysql" {
  identifier = "${var.project_name}-${var.environment}"
  
  engine = "mysql"
  engine_version = "8.0"
  instance_class = var.db_instance_class
  
  allocated_storage = 20
  max_allocated_storage = 100
  storage_type = "gp3"
  storage_encrypted = true
  
  db_name = "meeting_place"
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  backup_window = "03:00-04:00"
  maintenance_window = "sun:04:00-sun:05:00"
  
  multi_az = var.environment == "production"
  publicly_accessible = false
  skip_final_snapshot = var.environment != "production"
  
  tags = {
    Name = "${var.project_name}-mysql"
    Environment = var.environment
  }
}
```

### 5.4 EKS 클러스터 구성
```hcl
# infrastructure/terraform/eks.tf
resource "aws_eks_cluster" "main" {
  name = "${var.project_name}-${var.environment}"
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
    Name = "${var.project_name}-eks"
    Environment = var.environment
  }
}

resource "aws_eks_node_group" "main" {
  cluster_name = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-node-group"
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
    Name = "${var.project_name}-node-group"
    Environment = var.environment
  }
}
```

### 5.5 ElastiCache Redis 구성
```hcl
# infrastructure/terraform/redis.tf
resource "aws_elasticache_subnet_group" "main" {
  name = "${var.project_name}-redis-subnet-group"
  subnet_ids = aws_subnet.database[*].id
  
  tags = {
    Name = "${var.project_name}-redis-subnet-group"
    Environment = var.environment
  }
}

resource "aws_security_group" "redis" {
  name = "${var.project_name}-redis-sg"
  description = "Security group for ElastiCache Redis"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port = 6379
    to_port = 6379
    protocol = "tcp"
    security_groups = [aws_security_group.eks_nodes.id]
  }
  
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-redis-sg"
    Environment = var.environment
  }
}

resource "aws_elasticache_cluster" "redis" {
  cluster_id = "${var.project_name}-${var.environment}"
  engine = "redis"
  node_type = var.environment == "production" ? "cache.t3.micro" : "cache.t3.micro"
  num_cache_nodes = var.environment == "production" ? 2 : 1
  parameter_group_name = "default.redis7"
  port = 6379
  subnet_group_name = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]
  
  tags = {
    Name = "${var.project_name}-redis"
    Environment = var.environment
  }
}
```

## 6. 환경별 설정 파일

### 6.1 개발 환경
```hcl
# infrastructure/terraform/dev.tfvars
environment = "dev"
db_instance_class = "db.t3.micro"
vpc_cidr = "10.0.0.0/16"
```

### 6.2 스테이징 환경
```hcl
# infrastructure/terraform/staging.tfvars
environment = "staging"
db_instance_class = "db.t3.small"
vpc_cidr = "10.1.0.0/16"
```

### 6.3 프로덕션 환경
```hcl
# infrastructure/terraform/production.tfvars
environment = "production"
db_instance_class = "db.r5.large"
vpc_cidr = "10.2.0.0/16"
```

## 7. Terraform 상태 관리

### 7.1 로컬 상태 vs 원격 상태
```hcl
# 원격 상태 저장 (S3 + DynamoDB)
terraform {
  backend "s3" {
    bucket = "meeting-place-terraform-state"
    key = "infrastructure/terraform.tfstate"
    region = "ap-northeast-2"
    dynamodb_table = "meeting-place-terraform-locks"
    encrypt = true
  }
}
```

### 7.2 상태 파일 구조
```
terraform.tfstate (S3에 저장)
├── resources: 생성된 리소스 목록
├── outputs: 출력값들
└── version: Terraform 버전
```

## 8. 보안 및 모범 사례

### 8.1 민감한 정보 관리
```hcl
# AWS Secrets Manager 사용
data "aws_secretsmanager_secret" "db_credentials" {
  name = "meeting-place/db-credentials"
}

data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = data.aws_secretsmanager_secret.db_credentials.id
}

locals {
  db_credentials = jsondecode(data.aws_secretsmanager_secret_version.db_credentials.secret_string)
}

resource "aws_db_instance" "mysql" {
  username = local.db_credentials.username
  password = local.db_credentials.password
  # ... 기타 설정
}
```

### 8.2 태그 전략
```hcl
locals {
  common_tags = {
    Project = var.project_name
    Environment = var.environment
    ManagedBy = "terraform"
    Owner = "meeting-place-team"
  }
}

resource "aws_db_instance" "mysql" {
  # ... 설정
  tags = local.common_tags
}
```

## 9. 모니터링 및 로깅

### 9.1 CloudWatch 로그
```hcl
resource "aws_cloudwatch_log_group" "rds" {
  name = "/aws/rds/instance/${var.project_name}-${var.environment}"
  retention_in_days = 7
  
  tags = local.common_tags
}
```

### 9.2 알람 설정
```hcl
resource "aws_cloudwatch_metric_alarm" "rds_cpu" {
  alarm_name = "${var.project_name}-rds-cpu-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods = "2"
  metric_name = "CPUUtilization"
  namespace = "AWS/RDS"
  period = "300"
  statistic = "Average"
  threshold = "80"
  
  dimensions = {
    DBInstanceIdentifier = aws_db_instance.mysql.id
  }
  
  tags = local.common_tags
}
```

## 10. 비용 최적화

### 10.1 환경별 리소스 크기 조정
```hcl
locals {
  instance_sizes = {
    dev = {
      db_instance = "db.t3.micro"
      eks_node_type = "t3.medium"
      redis_node_type = "cache.t3.micro"
    }
    staging = {
      db_instance = "db.t3.small"
      eks_node_type = "t3.medium"
      redis_node_type = "cache.t3.micro"
    }
    production = {
      db_instance = "db.r5.large"
      eks_node_type = "t3.large"
      redis_node_type = "cache.t3.small"
    }
  }
}

resource "aws_db_instance" "mysql" {
  instance_class = local.instance_sizes[var.environment].db_instance
  # ... 기타 설정
}
```

### 10.2 자동 스케일링
```hcl
resource "aws_eks_node_group" "main" {
  scaling_config {
    desired_size = var.environment == "production" ? 3 : 2
    max_size = var.environment == "production" ? 5 : 3
    min_size = var.environment == "production" ? 2 : 1
  }
}
```

## 11. 장애 복구 및 백업

### 11.1 RDS 백업 설정
```hcl
resource "aws_db_instance" "mysql" {
  backup_retention_period = 7
  backup_window = "03:00-04:00"
  maintenance_window = "sun:04:00-sun:05:00"
  
  # 프로덕션에서만 Multi-AZ 활성화
  multi_az = var.environment == "production"
  
  # 스냅샷 설정
  skip_final_snapshot = var.environment != "production"
  final_snapshot_identifier = "${var.project_name}-${var.environment}-final-snapshot"
}
```

### 11.2 재해 복구 계획
```hcl
# 다른 리전에 백업 인프라 구성
provider "aws" {
  alias = "backup"
  region = "ap-southeast-1"
}

resource "aws_db_instance" "backup" {
  provider = aws.backup
  
  # 백업 전용 설정
  identifier = "${var.project_name}-backup-${var.environment}"
  # ... 기타 설정
}
```

## 12. 결론

### 12.1 Terraform 도입 효과
- **자동화**: 수동 작업 제거로 인적 오류 감소
- **일관성**: 환경별 동일한 인프라 구성 보장
- **재현성**: 동일한 환경 쉽게 재생성
- **협업**: 코드 리뷰를 통한 인프라 변경 관리
- **비용 관리**: 리소스 사용량 추적 및 최적화

### 12.2 Meeting Place 프로젝트 적용 시나리오
1. **초기 설정**: Terraform으로 기본 인프라 구성
2. **개발 환경**: 개발자별 독립적인 환경 제공
3. **스테이징 환경**: 테스트용 환경 자동 구성
4. **프로덕션 환경**: 안정적인 운영 환경 구성
5. **확장**: 사용자 증가에 따른 인프라 자동 스케일링

### 12.3 학습 로드맵
1. **기본 개념**: Terraform 문법 및 핵심 개념 학습
2. **AWS 리소스**: 주요 AWS 서비스 Terraform 구성법
3. **모듈화**: 재사용 가능한 Terraform 모듈 작성
4. **CI/CD 연동**: GitHub Actions와 Terraform 통합
5. **고급 기능**: 상태 관리, 백엔드 구성, 보안 설정 