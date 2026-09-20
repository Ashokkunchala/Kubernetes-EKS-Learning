# 🚀 Complete EKS Projects - Real-time Implementations

## 📋 Project 1: Microservices E-Commerce Application

### Architecture
```
┌─────────────────────────────────────────────────────────┐
│                  E-Commerce Architecture                 │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Ingress (ALB)                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Rules:                                  │   │   │
│  │  │  / → Frontend Service                    │   │   │
│  │  │  /api/products → Product Service         │   │   │
│  │  │  /api/orders → Order Service             │   │   │
│  │  │  /api/users → User Service               │   │   │
│  │  │  /api/payments → Payment Service         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  Frontend   │ │  Product    │ │  Order      │      │
│  │  Service    │ │  Service    │ │  Service    │      │
│  │  (React)    │ │  (Node.js)  │ │  (Go)       │      │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘      │
│         │               │               │              │
│         └───────────────┼───────────────┘              │
│                         │                              │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  User       │ │  Payment    │ │  Redis      │      │
│  │  Service    │ │  Service    │ │  Cache      │      │
│  │  (Python)   │ │  (Java)     │ │             │      │
│  └──────┬──────┘ └──────┬──────┘ └─────────────┘      │
│         │               │                              │
│         └───────────────┼──────────────────────┐      │
│                         │                      │      │
│                         ▼                      ▼      │
│                  ┌─────────────┐       ┌─────────────┐│
│                  │  PostgreSQL │       │  SQS        ││
│                  │  (RDS)      │       │  (Queues)   ││
│                  └─────────────┘       └─────────────┘│
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Directory Structure
```
ecommerce-app/
├── frontend/
│   ├── Dockerfile
│   ├── k8s/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── src/
├── product-service/
│   ├── Dockerfile
│   ├── k8s/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── src/
├── order-service/
│   ├── Dockerfile
│   ├── k8s/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── secret.yaml
│   └── src/
├── user-service/
│   ├── Dockerfile
│   ├── k8s/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── src/
├── payment-service/
│   ├── Dockerfile
│   ├── k8s/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── src/
└── infrastructure/
    ├── eks/
    │   ├── cluster.yaml
    │   └── nodegroups.yaml
    ├── monitoring/
    │   ├── prometheus.yaml
    │   └── grafana.yaml
    └── security/
        ├── network-policies.yaml
        └── rbac.yaml
```

### Implementation Steps

#### Step 1: EKS Cluster Setup
```bash
# Create EKS cluster
eksctl create cluster \
  --name ecommerce-cluster \
  --region us-west-2 \
  --nodes 3 \
  --node-type m5.large \
  --managed \
  --with-oidc

# Install essential addons
eksctl create addon --name aws-ebs-csi-driver --cluster ecommerce-cluster --force
eksctl create addon --name aws-load-balancer-controller --cluster ecommerce-cluster --force
eksctl create addon --name amazon-cloudwatch-observability --cluster ecommerce-cluster --force
```

#### Step 2: Frontend Deployment
```yaml
# frontend/k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: <ECR_URL>/ecommerce-frontend:latest
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: REACT_APP_API_URL
          value: "http://api-gateway.ecommerce.svc.cluster.local"
        livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
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
                  - frontend
              topologyKey: kubernetes.io/hostname
```

#### Step 3: Product Service Deployment
```yaml
# product-service/k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
      - name: product-service
        image: <ECR_URL>/ecommerce-product-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: DATABASE_URL
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: REDIS_URL
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

#### Step 4: Database Setup
```yaml
# infrastructure/rds/postgres.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
  namespace: ecommerce
type: Opaque
data:
  DATABASE_URL: cG9zdGdyZXNxbDovL3VzZXI6cGFzc3dvcmRAZHItZW5kcG9pbnQuZWNzLXVzLXdlc3QtMi5hbWF6b25hd3MuY29tOjU0MzIvZGJfbmFtZQ==
  DB_PASSWORD: cGFzc3dvcmQxMjM=
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: ecommerce
data:
  DB_HOST: "your-rds-endpoint.rds.amazonaws.com"
  DB_PORT: "5432"
  DB_NAME: "ecommerce"
  DB_USER: "admin"
```

#### Step 5: Ingress Configuration
```yaml
# infrastructure/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:ACCOUNT:certificate/CERT_ID
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api/products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 8080
      - path: /api/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
```

#### Step 6: Monitoring Setup
```yaml
# infrastructure/monitoring/prometheus.yaml
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

---

## 📋 Project 2: CI/CD Pipeline with GitHub Actions

### Architecture
```
┌─────────────────────────────────────────────────────────┐
│                CI/CD Pipeline Architecture               │
│                                                         │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  Git    │────►│  GitHub     │────►│  Build &    │     │
│  │  Push   │    │  Actions    │    │  Test       │     │
│  └─────────┘    └─────────────┘    └──────┬──────┘     │
│                                           │             │
│                                           ▼             │
│                                    ┌─────────────┐      │
│                                    │  Push to    │      │
│                                    │  ECR        │      │
│                                    └──────┬──────┘      │
│                                           │             │
│                                           ▼             │
│  ┌─────────────────────────────────────────────────┐   │
│  │              ArgoCD (GitOps)                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  1. Detect image update                  │   │   │
│  │  │  2. Pull new image from ECR             │   │   │
│  │  │  3. Deploy to EKS cluster               │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Amazon EKS                          │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  Development│  │  Production │              │   │
│  │  │  Namespace  │  │  Namespace  │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### GitHub Actions Workflow
```yaml
# .github/workflows/ci-cd.yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-west-2
  ECR_REPOSITORY: ecommerce-app
  EKS_CLUSTER: ecommerce-cluster

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run tests
      run: |
        npm test
        npm run lint
    
    - name: Run security scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build, tag, and push image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
    
    - name: Update Kubernetes manifests
      run: |
        sed -i "s|image:.*|image: $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG|g" k8s/deployment.yaml
        
    - name: Commit and push changes
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"
        git add k8s/
        git commit -m "Update image tag to $IMAGE_TAG"
        git push
```

### ArgoCD Application
```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/ecommerce-app.git
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: frontend
```

---

## 📋 Project 3: Production-Grade Microservices

### Architecture
```
┌─────────────────────────────────────────────────────────┐
│              Production Microservices Architecture        │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Service Mesh (Istio)                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Sidecar Proxy (Envoy)                  │   │   │
│  │  │  - Traffic management                    │   │   │
│  │  │  - Security (mTLS)                       │   │   │
│  │  │  - Observability                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  Frontend   │ │  API        │ │  Auth       │      │
│  │  (React)    │ │  Gateway    │ │  Service    │      │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘      │
│         │               │               │              │
│         └───────────────┼───────────────┘              │
│                         │                              │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  User       │ │  Product    │ │  Order       │      │
│  │  Service    │ │  Service    │ │  Service     │      │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘      │
│         │               │               │              │
│         └───────────────┼───────────────┘              │
│                         │                              │
│                         ▼                              │
│  ┌─────────────────────────────────────────────────┐  │
│  │              Data Layer                          │  │
│  │  ┌─────────────┐  ┌─────────────┐              │  │
│  │  │  PostgreSQL │  │  Redis      │              │  │
│  │  │  (RDS)      │  │  (ElastiCache) │           │  │
│  │  └─────────────┘  └─────────────┘              │  │
│  └─────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐  │
│  │              Monitoring Stack                     │  │
│  │  ┌─────────────┐  ┌─────────────┐              │  │
│  │  │  Prometheus │  │  Grafana    │              │  │
│  │  │  (Metrics)  │  │  (Dashboard)│              │  │
│  │  └─────────────┘  └─────────────┘              │  │
│  │  ┌─────────────┐  ┌─────────────┐              │  │
│  │  │  Jaeger     │  │  Fluent Bit │              │  │
│  │  │  (Tracing)  │  │  (Logging)  │              │  │
│  │  └─────────────┘  └─────────────┘              │  │
│  └─────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Istio Configuration
```yaml
# istio/gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ecommerce-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "ecommerce.example.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ecommerce-routes
  namespace: ecommerce
spec:
  hosts:
  - "ecommerce.example.com"
  gateways:
  - istio-system/ecommerce-gateway
  http:
  - match:
    - uri:
        prefix: /api/products
    route:
    - destination:
        host: product-service
        port:
          number: 8080
  - match:
    - uri:
        prefix: /api/orders
    route:
    - destination:
        host: order-service
        port:
          number: 8080
```

### Network Policies
```yaml
# security/network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ecommerce
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### RBAC Configuration
```yaml
# security/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: product-service-sa
  namespace: ecommerce
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/product-service-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: product-service-role
  namespace: ecommerce
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: product-service-binding
  namespace: ecommerce
subjects:
- kind: ServiceAccount
  name: product-service-sa
  namespace: ecommerce
roleRef:
  kind: Role
  name: product-service-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 📋 Project 4: Disaster Recovery Setup

### Architecture
```
┌─────────────────────────────────────────────────────────┐
│              Disaster Recovery Architecture               │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Primary Region (us-west-2)          │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  EKS Cluster (Production)               │   │   │
│  │  │  - 3 Worker Nodes                       │   │   │
│  │  │  - Multi-AZ Deployment                  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  RDS (Multi-AZ)                         │   │   │
│  │  │  - Primary Instance                     │   │   │
│  │  │  - Standby Instance                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Cross-Region Replication             │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  ECR Replication                        │   │   │
│  │  │  - Image replication to us-east-1       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  S3 Cross-Region Replication            │   │   │
│  │  │  - Backup replication to us-east-1      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              DR Region (us-east-1)               │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  EKS Cluster (Standby)                  │   │   │
│  │  │  - 2 Worker Nodes (scaled up during DR) │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  RDS (Read Replica)                     │   │   │
│  │  │  - Promoted to primary during DR        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Backup Strategy
```yaml
# backup/velero-schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  template:
    includedNamespaces:
    - ecommerce
    - monitoring
    storageLocation: default
    ttl: 720h  # 30 days retention
```

### Failover Script
```bash
#!/bin/bash
# failover.sh - DR failover script

# Configuration
PRIMARY_REGION="us-west-2"
DR_REGION="us-east-1"
CLUSTER_NAME="ecommerce-cluster"

echo "Starting DR failover..."

# 1. Update Route53 to point to DR region
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "ecommerce.example.com",
        "Type": "A",
        "AliasTarget": {
          "DNSName": "'$DR_ALB_DNS'",
          "HostedZoneId": "'$DR_ALB_ZONE_ID'",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# 2. Promote RDS Read Replica to Primary
aws rds promote-read-replica \
  --db-instance-identifier $DR_RDS_INSTANCE

# 3. Scale up EKS nodes in DR region
eksctl scale nodegroup \
  --cluster $CLUSTER_NAME \
  --name dr-workers \
  --nodes 3 \
  --region $DR_REGION

# 4. Deploy applications to DR cluster
kubectl --context=dr-cluster apply -f k8s/overlays/dr/

echo "DR failover completed!"
```

---

## 📋 Project 5: Cost Optimization with Spot Instances

### Architecture
```
┌─────────────────────────────────────────────────────────┐
│              Cost-Optimized EKS Cluster                  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Node Groups                          │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  On-Demand Node Group                    │   │   │
│  │  │  (Critical Workloads)                    │   │   │
│  │  │  - m5.xlarge (3 nodes)                  │   │   │
│  │  │  - Production databases                 │   │   │
│  │  │  - Core services                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Spot Node Group                         │   │   │
│  │  │  (Non-critical Workloads)               │   │   │
│  │  │  - m5.large (2-10 nodes)                │   │   │
│  │  │  - Batch processing                     │   │   │
│  │  │  - Development environments             │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Fargate (Serverless)                    │   │   │
│  │  │  (Variable Workloads)                    │   │   │
│  │  │  - API endpoints                        │   │   │
│  │  │  - Background jobs                      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Spot Instance Node Group
```yaml
# infrastructure/nodegroups/spot-nodes.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ecommerce-cluster
  region: us-west-2

nodeGroups:
  - name: spot-workers
    instanceType: m5.large
    desiredCapacity: 2
    minSize: 1
    maxSize: 10
    spot: true
    labels:
      workload-type: spot
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/ecommerce-cluster: owned
```

### Karpenter Configuration
```yaml
# karpenter/provisioner.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
  - key: "karpenter.sh/capacity-type"
    operator: In
    values: ["spot", "on-demand"]
  - key: "node.kubernetes.io/instance-type"
    operator: In
    values: ["m5.large", "m5.xlarge", "m5.2xlarge"]
  limits:
    resources:
      cpu: 100
      memory: 200Gi
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: ecommerce-cluster
  securityGroupSelector:
    karpenter.sh/discovery: ecommerce-cluster
  instanceProfile: KarpenterNodeInstanceProfile-ecommerce-cluster
  tags:
    karpenter.sh/discovery: ecommerce-cluster
```

### Pod Disruption Budget
```yaml
# pdb/frontend-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
  namespace: ecommerce
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: frontend
```

---

## 📋 Project 6: Complete Monitoring Stack

### Architecture
```
┌─────────────────────────────────────────────────────────┐
│              Complete Monitoring Stack                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Metrics Collection                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Prometheus                               │   │   │
│  │  │  - Cluster metrics                       │   │   │
│  │  │  - Application metrics                   │   │   │
│  │  │  - Custom metrics                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  CloudWatch Container Insights            │   │   │
│  │  │  - Node metrics                          │   │   │
│  │  │  - Pod metrics                           │   │   │
│  │  │  - Network metrics                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Visualization                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Grafana                                 │   │   │
│  │  │  - Cluster dashboard                    │   │   │
│  │  │  - Application dashboard                │   │   │
│  │  │  - Cost dashboard                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Logging                              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Fluent Bit                               │   │   │
│  │  │  - Container logs                        │   │   │
│  │  │  - Application logs                      │   │   │
│  │  │  - Audit logs                            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  CloudWatch Logs                         │   │   │
│  │  │  - Log storage                           │   │   │
│  │  │  - Log queries                           │   │   │
│  │  │  - Alarms                                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Alerting                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Prometheus Alertmanager                 │   │   │
│  │  │  - Slack notifications                  │   │   │
│  │  │  - PagerDuty integration                │   │   │
│  │  │  - Email alerts                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Prometheus Configuration
```yaml
# monitoring/prometheus/prometheus.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - "alerts/*.yml"
    
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - alertmanager:9093
    
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: (.+):(?:\d+)
            replacement: ${1}:$1
      
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
```

### Grafana Dashboard
```json
{
  "dashboard": {
    "title": "EKS Cluster Monitoring",
    "panels": [
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total{namespace=\"ecommerce\"}[5m])",
            "legendFormat": "{{pod}}"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{namespace=\"ecommerce\"}",
            "legendFormat": "{{pod}}"
          }
        ]
      },
      {
        "title": "Network Receive",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_network_receive_bytes_total{namespace=\"ecommerce\"}[5m])",
            "legendFormat": "{{pod}}"
          }
        ]
      }
    ]
  }
}
```

### Alert Rules
```yaml
# monitoring/prometheus/alerts/high-cpu.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: monitoring
data:
  high-cpu.yaml: |
    groups:
    - name: cpu-alerts
      rules:
      - alert: HighCPUUsage
        expr: rate(container_cpu_usage_seconds_total{namespace="ecommerce"}[5m]) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "Pod {{ $labels.pod }} has high CPU usage"
      
      - alert: HighMemoryUsage
        expr: container_memory_usage_bytes{namespace="ecommerce"} / container_spec_memory_limit_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Pod {{ $labels.pod }} has high memory usage"
```

---

## 🎯 Project Implementation Checklist

### Project 1: Microservices E-Commerce
- [ ] Set up EKS cluster
- [ ] Create namespaces (dev, staging, prod)
- [ ] Deploy frontend service
- [ ] Deploy product service
- [ ] Deploy order service
- [ ] Deploy user service
- [ ] Deploy payment service
- [ ] Configure ALB Ingress
- [ ] Set up PostgreSQL (RDS)
- [ ] Set up Redis (ElastiCache)
- [ ] Implement health checks
- [ ] Configure auto-scaling
- [ ] Set up monitoring

### Project 2: CI/CD Pipeline
- [ ] Create GitHub repository
- [ ] Set up GitHub Actions
- [ ] Configure ECR repository
- [ ] Implement build and test
- [ ] Set up ArgoCD
- [ ] Configure GitOps workflow
- [ ] Implement rollback procedures
- [ ] Set up environment promotion

### Project 3: Production Microservices
- [ ] Implement Istio service mesh
- [ ] Configure mTLS
- [ ] Set up traffic management
- [ ] Implement network policies
- [ ] Configure RBAC
- [ ] Set up IRSA for AWS services
- [ ] Implement secret management
- [ ] Configure audit logging

### Project 4: Disaster Recovery
- [ ] Set up cross-region replication
- [ ] Configure Velero backups
- [ ] Implement failover procedures
- [ ] Test DR scenarios
- [ ] Document runbooks
- [ ] Set up monitoring for DR

### Project 5: Cost Optimization
- [ ] Implement spot instances
- [ ] Configure Karpenter
- [ ] Set up cluster autoscaler
- [ ] Implement resource quotas
- [ ] Monitor costs with Cost Explorer
- [ ] Optimize instance types

### Project 6: Complete Monitoring
- [ ] Deploy Prometheus
- [ ] Deploy Grafana
- [ ] Configure Fluent Bit
- [ ] Set up CloudWatch Container Insights
- [ ] Implement alerting
- [ ] Create dashboards
- [ ] Set up log aggregation

---

**Remember:** Start with Project 1 and gradually build up to more complex projects. Each project builds on the previous one and adds new concepts. Good luck! 🚀
