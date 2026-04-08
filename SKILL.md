---
name: devops-automation
description: "DevOps 自动化助手。CI/CD流水线、Docker、K8s、监控告警、IaC。触发词：devops、ci/cd、docker、kubernetes、terraform、ansible、自动化部署。"
metadata: {"openclaw": {"emoji": "🚀"}}
---

# DevOps Automation

## 功能说明

DevOps 全流程自动化，从代码提交到生产部署。

## 技术栈覆盖

| 领域 | 工具 |
|------|------|
| CI/CD | GitHub Actions, GitLab CI, Jenkins, ArgoCD |
| Container | Docker, Podman |
| Orchestration | Kubernetes, Helm |
| IaC | Terraform, Pulumi, Ansible |
| Monitoring | Prometheus, Grafana, ELK |
| Cloud | AWS, GCP, Azure |

## 1. GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # === 测试阶段 ===
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run lint
        run: npm run lint
      
      - name: Run type check
        run: npm run typecheck
      
      - name: Run tests
        run: npm run test:coverage
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: true

  # === 构建阶段 ===
  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # === 部署阶段 ===
  deploy-staging:
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Staging
        run: |
          kubectl config use-context staging
          kubectl set image deployment/app app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          kubectl rollout status deployment/app --timeout=300s
      
      - name: Smoke test
        run: |
          sleep 30
          curl -f https://staging.example.com/health || exit 1

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    concurrency:
      group: production
      cancel-in-progress: false
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Production
        run: |
          kubectl config use-context production
          kubectl set image deployment/app app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          kubectl rollout status deployment/app --timeout=600s
```

## 2. Docker 最佳实践

```dockerfile
# 构建多阶段镜像
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# 生产镜像
FROM node:20-alpine AS runner
WORKDIR /app

# 安全：非root运行
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser

COPY --from=builder --chown=appuser:nodejs /app/dist ./dist
COPY --from=builder --chown=appuser:nodejs /app/node_modules ./node_modules

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node /app/dist/health.js

CMD ["node", "dist/index.js"]
```

```dockerfile
# .dockerignore
node_modules
.git
*.log
.env*
coverage
dist
```

## 3. Kubernetes 部署

```yaml
# k8s/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
        version: v1
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: app
                topologyKey: kubernetes.io/hostname
      containers:
        - name: app
          image: ghcr.io/org/app:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3
          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
```

```yaml
# k8s/service.yml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: app
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP

---
# k8s/ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
        - www.example.com
      secretName: app-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
```

## 4. Terraform IaC

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "tf-state-bucket"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    dynamodb_table = "tf-locks"
    encrypt = true
  }
}

provider "aws" {
  region = "us-east-1"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "main-vpc"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# 子网
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
    Type = "public"
  }
}

# RDS
resource "aws_db_instance" "main" {
  identifier     = "app-db"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.medium"
  
  db_name  = "app"
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  
  allocated_storage     = 100
  max_allocated_storage = 500
  storage_encrypted    = true
  storage_type         = "gp3"
  
  multi_az               = true
  availability_zone      = data.aws_availability_zones.available.names[0]
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"
  
  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# EKS
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  
  cluster_name    = "app-cluster"
  cluster_version = "1.28"
  
  vpc_id                   = aws_vpc.main.id
  subnet_ids               = aws_subnet.private[*].id
  
  eks_managed_node_groups = {
    app = {
      min_size       = 2
      max_size       = 10
      desired_size   = 3
      instance_type  = "t3.medium"
      capacity_type   = "SPOT"
      
      labels = {
        workload = "app"
      }
    }
  }
}
```

## 5. Ansible 部署

```yaml
# ansible/playbook.yml
- hosts: webservers
  become: yes
  vars:
    app_version: "1.2.3"
    deploy_path: "/opt/app"
  
  tasks:
    - name: Ensure deploy directory exists
      file:
        path: "{{ deploy_path }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
    
    - name: Deploy application
      synchronize:
        src: ./dist/
        dest: "{{ deploy_path }}/current"
        delete: yes
        rsync_opts:
          - "--exclude=.env"
    
    - name: Restart application
      systemd:
        name: app
        state: restarted
        daemon_reload: yes
    
    - name: Health check
      uri:
        url: "http://localhost:3000/health"
        status_code: 200
      register: health_check
      until: health_check.status == 200
      retries: 10
      delay: 5
```

## 6. 监控告警

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "alerts/*.yml"

scrape_configs:
  - job_name: 'app'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: app
```

```yaml
# prometheus/alerts.yml
groups:
  - name: app
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(http_errors_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | printf \"%.2f\" }}%"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P95 latency is {{ $value | printf \"%.2f\" }}s"
```

## 常见场景

| 场景 | 方案 |
|------|------|
| 零停机部署 | RollingUpdate + readinessProbe + 预检查 |
| 回滚 | `kubectl rollout undo deployment/app` |
| 蓝绿部署 | 双环境 + SLB 切换 |
| 金丝雀发布 | Argo Rollouts + 流量权重 |
| 数据库迁移 | 肥沃迁移 + 回滚脚本 |
| 密钥管理 | Vault / Sealed Secrets |
