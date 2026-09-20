# 📚 EKS Foundations Review (Days 1-15)
## Consolidated Learning Guide

> **Based on completed materials from EKS_Learning directory**
> **Days 1-15: Kubernetes Basics + AWS EKS Fundamentals**

---

# 🎯 Phase 1 Summary: Foundation (Days 1-15)

## 📅 Week 1: Kubernetes Basics (Days 1-7)

### **Day 1: What is Kubernetes?**
**Key Concepts:**
- Kubernetes as container orchestration platform
- Control plane vs worker node architecture
- Problems solved: auto-scaling, self-healing, service discovery, rolling updates
- Comparison with Docker Swarm and Docker Compose

**Components:**
- **Control Plane:** API Server, Scheduler, Controller Manager, etcd
- **Worker Nodes:** kubelet, kube-proxy, Container Runtime
- etcd as distributed key-value store (cluster brain)

**Real-World Example:** Netflix uses Kubernetes for millions of pods with auto-scaling during peak hours.

### **Day 2: Pods, ReplicaSets, and Deployments**
**Key Concepts:**
- Pod: Smallest deployable unit (1+ containers sharing network/storage)
- ReplicaSet: Ensures desired number of pod replicas
- Deployment: Manages ReplicaSets with rolling updates/rollbacks

**Critical Relationships:**
```
Deployment → ReplicaSet → Pods
```

**Essential kubectl Commands:**
- `kubectl run` (learning only - not for production)
- `kubectl create deployment`
- `kubectl get pods,deployments,rs`
- `kubectl describe pod <name>`
- `kubectl logs <pod-name>`
- `kubectl exec -it <pod> -- /bin/bash`
- `kubectl scale deploy/<name> --replicas=N`
- `kubectl rollout status deploy/<name>`
- `kubectl rollout undo deploy/<name>`

**YAML Structure:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### **Day 3: Services and Networking**
**Key Concepts:**
- Service: Stable endpoint for accessing pods
- Service Types: ClusterIP, NodePort, LoadBalancer, ExternalName
- kube-proxy: Maintains network rules (iptables/IPVS)
- DNS-based service discovery: `<service>.<namespace>.svc.cluster.local`

**Service Types:**
```
ClusterIP: Internal-only communication (default)
NodePort: Exposes service on static port on each node
LoadBalancer: Provisions AWS ELB/ALB (EKS specific)
ExternalName: Maps service to external DNS name
```

**YAML Examples:**
```yaml
# ClusterIP (internal)
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080

# LoadBalancer (AWS ELB)
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

### **Day 4: Namespaces, ConfigMaps, and Secrets**
**Key Concepts:**
- Namespaces: Virtual cluster partitioning (dev, staging, prod)
- ConfigMap: Stores non-confidential configuration data
- Secret: Stores sensitive data (base64 encoded, ideally encrypted)

**Isolation Benefits:**
- Resource quotas per namespace
- RBAC scoping
- Reduced naming conflicts
- Environment separation

**YAML Examples:**
```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: development

# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgres://db:5432/myapp"
  LOG_LEVEL: "info"
  FEATURE_TOGGLE: "true"

# Secret (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
data:
  username: YWRtaW4=        # "admin" in base64
  password: cGFzc3dvcmQ=    # "password" in base64
```

**Usage in Pods:**
```yaml
env:
- name: DATABASE_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DATABASE_URL
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secrets
      key: password
```

### **Day 5: Persistent Storage**
**Key Concepts:**
- PersistentVolume (PV): Cluster-level storage resource
- PersistentVolumeClaim (PVC): Request for storage by pod
- StorageClass: Dynamic provisioning template
- volumeBindingMode: WaitForFirstConsumer (recommended)

**Access Modes:**
- ReadWriteOnce (RWO): Single node read/write
- ReadOnlyMany (ROX): Multiple nodes read-only
- ReadWriteMany (RWX): Multiple nodes read/write

**AWS EBS Specific:**
- volumeBindingMode: WaitForFirstConsumer (pod scheduled before volume created)
- Storage parameters: type (gp2/gp3/io1), encrypted, fsType, iopsPerGB

**YAML Examples:**
```yaml
# StorageClass for AWS EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"

# PVC requesting storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 20Gi

# Pod using PVC
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secrets
          key: MYSQL_ROOT_PASSWORD
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: mysql-pvc
```

### **Day 6: ConfigMaps and Secrets in Practice**
**Practical Applications:**
- Environment variable injection
- Volume mount configuration
- Command-line arguments
- Image pull secrets
- Application configuration files

**Best Practices:**
- Never store plaintext secrets in ConfigMaps
- Use Sealed Secrets or AWS Secrets Manager for production
- Rotate secrets regularly
- Implement least privilege access
- Audit secret usage

### **Day 7: Kubernetes Networking Deep Dive**
**Key Concepts:**
- Pod-to-pod communication (flat network model)
- Service discovery via DNS
- kube-proxy modes: iptables (legacy) vs IPVS (performance)
- CNI (Container Network Interface) plugins
- Network policies for traffic control

**Network Flow:**
```
Pod A (10.0.1.5) → kube-proxy → Service (10.0.0.10) → kube-proxy → Pod B (10.0.2.15)
```

**DNS Resolution:**
- Service: `my-service.namespace.svc.cluster.local`
- Headless Service: Returns pod IPs directly
- ExternalName: CNAME record to external DNS

---

## 📅 Week 2: AWS EKS Fundamentals (Days 8-15)

### **Day 8: Introduction to Amazon EKS**
**Key Concepts:**
- EKS = AWS-managed Kubernetes control plane + EC2 worker nodes
- Control plane: Highly available, single-tenant, AWS managed
- Worker nodes: EC2 instances in customer's VPC
- Pricing: $0.10/hour for control plane + EC2 instance costs

**Benefits:**
- Managed control plane (patching, upgrades, HA)
- AWS integration (IAM, VPC, Security Groups, etc.)
- Compliance (SOC, PCI, HIPAA, etc.)
- Support for AWS services (ECR, ELB, CloudWatch, etc.)

**Architecture:**
```
AWS Cloud
├── EKS Control Plane (Managed)
│   ├── API Server (HA across AZs)
│   ├── etcd (3-5 node cluster)
│   ├── Scheduler
│   └── Controllers
└── VPC (Customer)
    ├── Public Subnets (NAT Gateways)
    ├── Private Subsets (Worker Nodes)
    │   ├── EC2 Instances
    │   ├── kubelet
    │   ├── kube-proxy
    │   └── Container Runtime
    └── AWS Services Integration
        ├── IAM (IRSA, node IAM roles)
        ├── ECR (image storage)
        ├── ELB/ALB (LoadBalancer services)
        └── CloudWatch (monitoring/logs)
```

### **Day 9: EKS Cluster Setup with eksctl**
**Key Concepts:**
- eksctl: CLI tool for EKS cluster lifecycle management
- Declarative cluster configuration via YAML
- Automated VPC, IAM, and security group creation
- Managed node groups vs self-managed nodes

**Essential eksctl Commands:**
```bash
# Create cluster
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --managed \
  --with-oidc \
  --ssh-access \
  --ssh-public-key ~/.ssh/id_rsa.pub

# Get cluster info
eksctl get cluster --name my-cluster --region us-west-2

# Add nodegroup
eksctl create nodegroup \
  --cluster my-cluster \
  --region us-west-2 \
  --name extra-workers \
  --node-type t3.large \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4

# Delete cluster
eksctl delete cluster --name my-cluster --region us-west-2
```

**Cluster Creation Flow:**
1. Create VPC (if needed)
2. Create IAM roles (control plane, node, service)
3. Create EKS control plane
4. Create node group (EC2 Auto Scaling Group)
5. Configure aws-auth ConfigMap
6. Install essential add-ons (VPC CNI, kube-proxy, etc.)

### **Day 10: EKS Networking and VPC**
**Key Concepts:**
- VPC requirements for EKS
- Subnet strategy (public vs private)
- NAT Gateway vs Internet Gateway
- Security groups for EKS
- CNI custom networking options

**VPC Architecture Best Practices:**
```
VPC (10.0.0.0/16)
├── Public Subnets (2+ AZs)
│   ├── NAT Gateways (for private subnet internet access)
│   └── Bastion Hosts (optional, for SSH)
├── Private Subnets (2+ AZs) ← Recommended for Workers
│   ├── EKS Worker Nodes
│   └── No direct internet access (use NAT GW)
├── Route Tables
│   ├── Public: 0.0.0.0/0 → Internet Gateway
│   └── Private: 0.0.0.0/0 → NAT Gateway
├── Security Groups
│   ├── Control Plane: Allow 443 from worker nodes
│   ├── Worker Nodes: 
│   │   - Allow 443 from control plane SG
│   │   - Allow 1025-65535 from self (pod communication)
│   │   - Allow 22 from bastion SG (optional)
│   └── Application SGs: 
│       - Allow 80/443 from internet (if public)
│       - Allow database ports from app SG
└── NACLs (optional additional layer)
```

**EKS Networking Requirements:**
- Minimum 2 subnets (different AZs)
- Sufficient IP address space (consider pod density)
- Proper DNS resolution (enable DNS hostnames in VPC)
- NAT Gateway for private subnets (internet access for updates/patches)

### **Day 11: EKS IAM Roles and Policies**
**Key Concepts:**
- IAM Roles for Service Accounts (IRSA)
- Node IAM Role
- EKS Cluster IAM Role
- OIDC Provider for IRSA federation
- Principle of least privilege

**IAM Roles Architecture:**
```
AWS IAM
├── EKS Cluster IAM Role
│   ├── Permissions: eks:DescribeCluster, ec2:Describe*, etc.
│   └── Trust: EKS service
├── Node IAM Role
│   ├── Permissions: 
│   │   - ecr:GetAuthorizationToken
│   │   - ecr:BatchCheckLayerAvailability
│   │   - ec2:Describe*
│   │   - logs:CreateLogGroup/Stream
│   │   - ec2:CreateNetworkInterface
│   │   - AssumeRole for IRSA
│   └── Trust: EC2 service
├── IRSA Roles (per service/account)
│   ├── Permissions: 
│   │   - Specific AWS service access (S3, DynamoDB, etc.)
│   │   - least privilege
│   └── Trust: 
│       - Federated via OIDC
│       - Condition: StringLike: 
│         "sts:ExternalSubject": "system:serviceaccount:<namespace>:<sa-name>"
└── OIDC Provider
    ├── Created per EKS cluster
    └── URL: https://oidc.eks.<region>.amazonaws.com/id/<ID>
```

**IRSA Implementation:**
```bash
# 1. Create OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# 2. Create IAM role for service account
eksctl create iamserviceaccount \
  --name s3-access \
  --namespace production \
  --cluster my-cluster \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve \
  --override-existing-serviceaccounts
```

**Resulting K8s Manifest:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/EKS-prod-s3-access-role
```

### **Day 12: EKS Storage with EBS**
**Key Concepts:**
- Amazon EBS CSI Driver
- Dynamic volume provisioning
- StorageClasses for different volume types
- Volume snapshots and cloning
- Encryption at rest with KMS

**EBS CSI Driver Installation:**
```bash
# Install via eksctl
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster my-cluster \
  --force

# Or via manifest
kubectl apply -f https://github.com/kubernetes-sigs/aws-ebs-csi-driver/releases/download/v1.10.0/aws-ebs-csi-driver.yaml
```

**StorageClasses for Different Workloads:**
```yaml
# General Purpose SSD (gp3) - Default
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc-default
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
  iopsPerGB: "10"  # Optional for gp3

# Provisioned IOPS (io1/io2) - Databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc-io2
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: io2
  iopsPerGB: "50"
  encrypted: "true"
  fsType: ext4

# Throughput Optimized HDD (st1) - Big Data
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc-st1
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: st1
  encrypted: "true"
```

**Volume Encryption:**
- EBS volumes encrypted by default if StorageClass specifies encrypted: "true"
- Uses AWS KMS (customer-managed or AWS-managed keys)
- Can specify specific KMS key ARN in StorageClass parameters

### **Day 13: EKS Load Balancing with ALB**
**Key Concepts:**
- AWS Load Balancer Controller
- ALB Ingress Controller (Layer 7)
- NLB Service (Layer 4)
- Target types: instance vs ip
- Certificate management with ACM

**Components:**
```
Internet
├── Route 53 (DNS) → myapp.example.com
├── ACM Certificate (myapp.example.com)
├── Application Load Balancer (ALB)
│   ├── Listener: 443 HTTPS (SSL termination)
│   ├── Listener: 80 HTTP → redirect to 443
│   ├── Target Groups:
│   │   - frontend-tg (instance or ip)
│   │   - api-tg (instance or ip)
│   │   - webhook-tg (instance or ip)
│   ├── Rules:
│   │   - / → frontend-tg
│   │   - /api/* → api-tg
│   │   - /webhooks/* → webhook-tg
│   └── Health Checks: /health on each target
└── EKS Cluster
    ├── Worker Nodes (in private subnets)
    ├── Pods (with application containers)
    ├── Services (ClusterIP or NodePort)
    └── Ingress Resource
```

**AWS Load Balancer Controller Installation:**
```bash
eksctl create addon \
  --name aws-load-balancer-controller \
  --cluster my-cluster \
  --force
```

**Ingress YAML:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:123456789012:certificate/abcdef12-3456-7890-abcd-ef1234567890
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/group.name: app-group
    alb.ingress.kubernetes.io/load-balancer-name: my-app-alb
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /webhooks
        pathType: Exact
        backend:
          service:
            name: webhook-service
            port:
              number: 9000
```

### **Day 14: EKS Monitoring and Logging**
**Key Concepts:**
- CloudWatch Container Insights
- Prometheus and Grafana
- AWS X-Ray for tracing
- Fluent Bit/Fluentd for log aggregation
- Metrics: CPU, memory, network, storage, application-level

**Monitoring Stack:**
```
Applications
├── Custom Metrics (StatsD, Prometheus client libs)
├── Access Logs (NGINX, application logs)
├── Debug Logs (application, framework)
└── Audit Logs (Kubernetes API server)

Collection Agents
├── CloudWatch Agent (node-level metrics)
├── Fluent Bit/Fluentd (log aggregation)
├── Prometheus Node Exporter (host metrics)
├── Prometheus Adapter (custom metrics API)
└── AWS X-Ray Daemon (tracing)

Storage & Visualization
├── Amazon CloudWatch Logs (log storage)
├── Amazon CloudWatch Metrics (metric storage)
├── Amazon Managed Service for Prometheus (AMP)
├── Amazon Managed Grafana (AMG)
├── Elasticsearch Service (log analytics)
└── Amazon OpenSearch Service (log analytics)

Alerting & Notification
├── CloudWatch Alarms (metric-based)
├── SNS Topics (notification routing)
├── Lambda Functions (custom processing)
├── PagerDuty/Slack/Email (notification channels)
└── Auto Scaling Policies (reactive scaling)
```

**CloudWatch Container Insights Installation:**
```bash
eksctl create addon \
  --name amazon-cloudwatch-observability \
  --cluster my-cluster \
  --force
```

**Key Metrics to Monitor:**
```
Node-Level:
- CPU Utilization (%)
- Memory Utilization (%)
- Disk Utilization (%)
- Network In/Out (packets/sec, bytes/sec)
- File Descriptor Usage
- Process Count

Pod/Container-Level:
- CPU Usage (cores, millicores)
- Memory Usage (bytes, %)
- Network RX/TX (bytes/sec)
- Filesystem Usage (bytes, %)
- Container Restarts (count)
- Ready Containers (count/desired)

Application-Level:
- Request Rate (requests/sec)
- Error Rate (%)
- Latency (p50, p95, p99 ms)
- Throughput (bytes/sec)
- Business Metrics (custom)

Cluster-Level:
- Node Count (ready/total)
- Pod Count (running/pending/failed)
- Namespace Resource Usage
- API Server Latency
- etcd Performance
```

### **Day 15: EKS Security Best Practices**
**Key Concepts:**
- Pod Security Standards (replaced PSPs)
- Network Policies (Calico, AWS VPC CNI)
- Secrets Encryption (KMS)
- Image Scanning (ECR, Trivy)
- Runtime Security (Falco, Sysdig)
- RBAC and Authorization
- Audit Logging
- Secure Cluster Access

**Security Layers:**
```
Layer 1: Cluster Security
├── API Server Access Control
│   ├── Private endpoint (recommended)
│   ├── Authorized IP ranges
│   └── Authentication (IAM, OIDC, ldap)
├── Control Plane Protection
│   ├── Security Groups
│   ├── Encryption at rest (KMS for etcd/secrets)
│   └── Audit Logs (CloudWatch Logs)
├── Node Registration
│   ├── Node IAM role validation
│   └── Bootstrapping security

Layer 2: Node Security
├── Worker Node Configuration
│   ├── EC2 Launch Templates (minimized AMI)
│   ├── Security Groups (least privilege)
│   ├── IAM Role (EKS worker node policy)
│   ├── SSM Agent (for patching/management)
│   └── User Data (secure bootstrapping)
├── OS Hardening
│   ├── CIS Benchmarks
│   ├── Kernel hardening
│   ├── SELinux/AppArmor
│   └── Automatic security updates

Layer 3: Pod Security
├── Pod Security Standards (PSS)
│   ├── Privileged (avoid in production)
│   ├── Baseline (minimal restrictions)
│   └── Restricted (recommended for production)
├── Security Context
│   ├── runAsNonRoot: true
│   ├── runAsUser: 1000
│   ├── fsGroup: 2000
│   ├── readOnlyRootFilesystem: true
│   └── allowPrivilegeEscalation: false
├── Capabilities
│   ├── drop: ["ALL"]
│   └── add: ["NET_BIND_SERVICE"] (if needed)
└── Seccomp/AppArmor Profiles
    ├── RuntimeDefault or Custom
    └── Profile localhost/<profile-name>

Layer 4: Container Security
├── Image Scanning
│   ├── ECR basic scanning (free)
│   ├── ECR enhanced scanning (advanced)
│   ├── Trivy (open source)
│   └── Clair, Anchore, etc.
├── Image Signing
│   ├── Cosign (cosign verify)
│   └── Notary
├── Base Images
│   ├── Distroless
│   ├── Alpine (minimal)
│   └── Scratch (when possible)
└── Resource Constraints
    ├── requests/limits (CPU, memory)
    └── OOMKill prevention

Layer 5: Network Security
├── Network Policies
│   ├── Default deny-ingress/egress
│   ├── Allow specific namespace/pod communication
│   └── Egress controls (DNS, external APIs)
├── Service Mesh
│   ├── Istio (mTLS, AuthorizationPolicy)
│   ├── App Mesh (AWS native)
│   └── Linkerd, Consul, etc.
├── TLS everywhere
│   ├── Service-to-service (mTLS)
│   ├── Ingress (client-to-service)
│   └── etcd (peer-to-peer)
├── WAF & DDoS Protection
│   ├── AWS WAF (ALB protection)
│   ├── AWS Shield Standard/Advanced
│   └── Rate limiting, IP allow/block lists
└── API Security
    ├── Rate limiting
    ├── Input validation
    └── JWT/OAuth validation

Layer 6: Data Security
├── Encryption at Rest
│   ├── EBS volumes (KMS)
│   ├── RDS (KMS)
│   ├── S3 (KMS or S3-Managed Keys)
│   └── etcd (KMS)
├── Encryption in Transit
│   ├── TLS 1.2+ everywhere
│   └── Mutual TLS for service-to-service
├── Data Classification
│   ├── PII, PHI, financial data
│   └── Handling procedures per classification
└── Backup & Recovery
    ├── Encrypted backups
    ├── Cross-region replication
    └── Regular restore testing
```

**Essential Security Commands:**
```bash
# Enable encryption at rest for secrets
aws eks update-cluster-config \
  --name my-cluster \
  --region us-west-2 \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:us-west-2:123456789012:key/12345678-1234-1234-1234-123456789012"}}]'

# Restrict cluster endpoint access
aws eks update-cluster-config \
  --name my-cluster \
  --region us-west-2 \
  --resources-vpc-config 'endpointPublicAccess=false,endpointPrivateAccess=true'

# Scan images in ECR
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=latest

# Check Pod Security Standards
kubectl pod-security-check
```

**Production Security Checklist:**
```
[ ] Control plane endpoint access restricted to private
[ ] Secrets encrypted with customer-managed KMS key
[ ] Worker nodes in private subnets
[ ] Worker node security groups least privilege
[ ] EBSSCSI driver encryption enabled
[ ] IRSA used for AWS service access (no static credentials)
[ ] Image scanning integrated in CI/CD
[ ] Pod Security Standards enforced (restricted profile)
[ ] Network policies implemented (default deny)
[ ] RBAC least privilege implemented
[ ] Audit logging enabled (CloudWatch Logs)
[ ] Regular security scanning (Trivy, kube-bench)
[ ] Incident response plan documented
[ ] Regular access review (IAM roles, RBAC)
```

---

## 🔑 Key Takeaways from Foundation Phase

### **Architectural Principles:**
1. **Separation of Concerns**: Control plane (managed) vs worker nodes (customer)
2. **Security in Layers**: Network → Compute → Container → Application
3. **Observability First**: Metrics, logs, tracing from day one
4. **Infrastructure as Code**: eksctl, CloudFormation, Terraform for reproducibility
5. **Immutability**: Treat infrastructure as code, patch by replacing

### **Production Readiness Checklist (Days 1-15):**
```
[ ] EKS cluster created with proper VPC (public/private subnets)
[ ] Worker nodes in private subnets with NAT gateway
[ ] Control plane access restricted to private endpoint
[ ] Essential add-ons installed (VPC CNI, kube-proxy, CoreDNS)
[ ] IAM roles configured (node role, IRSA for AWS access)
[ ] Storage configured (EBS CSI driver with encryption)
[ ] Load balancing configured (ALB Ingress Controller)
[ ] Monitoring/Logging enabled (CloudWatch Container Insights)
[ ] Security basics implemented (endpoint restriction, basic RBAC)
[ ] Sample applications deployed and accessible
[ ] Basic troubleshooting procedures documented
[ ] Backup/restore procedures for etcd and persistent volumes
```

### **Common Day 1-15 Interview Questions:**
1. **What are the main components of Kubernetes architecture?**
   - Control plane (API server, scheduler, controller manager, etcd)
   - Worker nodes (kubelet, kube-proxy, container runtime)

2. **How does EKS differ from self-managed Kubernetes?**
   - AWS manages control plane (HA, patching, upgrades)
   - Customer manages worker nodes in their VPC
   - Integrated AWS services (IAM, VPC, ECR, ELB, CloudWatch)

3. **What is the purpose of a Deployment vs a ReplicaSet?**
   - Deployment manages ReplicaSets and provides declarative updates
   - ReplicaSet ensures desired number of pod replicas
   - Always use Deployments for stateless applications in production

4. **How do pods get network connectivity in EKS?**
   - Via VPC CNI plugin assigning VPC IP addresses to pods
   - Pods get IPs from VPC subnet, enabling direct communication with AWS resources

5. **What is IRSA and why is it important?**
   - IAM Roles for Service Accounts
   - Allows pods to access AWS services without static credentials
   - Uses OIDC federation for just-in-time credentials
   - More secure than storing AWS keys in secrets or environment variables

6. **How does a Service differ from an Ingress?**
   - Service: Layer 4 load balancing (TCP/UDP), cluster-internal by default
   - Ingress: Layer 7 HTTP/HTTPS routing, provides external access via ALB/NLB
   - Ingress requires Ingress Controller (AWS Load Balancer Controller in EKS)

7. **What are PersistentVolumes and PersistentVolumeClaims?**
   - PV: Cluster-wide storage resource (like a disk)
   - PVC: Request for storage by a pod (like a partition request)
   - StorageClass enables dynamic provisioning (like a disk type)

8. **How do you secure sensitive data in Kubernetes?**
   - Secrets resources (base64 encoded, ideally encrypted at rest)
   - Prefer external secret stores (AWS Secrets Manager, HashiCorp Vault)
   - Use CSI drivers for secret management
   - Never commit secrets to version control

9. **What is the purpose of resource requests and limits?**
   - Requests: Guaranteed resources for scheduler placement
   - Limits: Maximum resources container can use (prevents noisy neighbors)
   - Critical for QoS classes and cluster stability

10. **How does EKS handle control plane upgrades?**
    - Zero downtime for workloads
    - New control plane instances created in parallel
    - etcd data migrated to new instances
    - API endpoint switched via DNS (brief interruption possible)
    - Worker nodes can lag up to 3 versions behind control plane
```

---

## 📝 Next Steps for Continued Learning

Based on this foundation review, you're ready to proceed to:

### **Phase 2: Intermediate EKS Topics (Days 16-30)**
- Advanced Networking (VPC CNI deep dive, Calico, Istio service mesh)
- Advanced Storage (EFS, FSx for Lustre, S3 CSI drivers)
- Advanced IAM and Security (OIDC, IRSA for multiple services, pod-level IAM, Secrets Manager)
- Practice exercises and production scenarios

### **Recommended Study Approach:**
1. Review one day's material per session
2. Create mind maps connecting concepts
3. Practice explaining concepts aloud
4. Work through the hands-on exercises in your own environment
5. Document challenges and solutions
6. Prepare for interview questions by teaching the material to someone else

### **Production-Focused Exercises to Try:**
1. Build a 3-tier application (frontend, API, database) with proper security
2. Implement blue/green deployment using ArgoCD or Flagger
3. Set up comprehensive monitoring with alerts for key metrics
4. Practice disaster recovery by simulating node/AZ failure
5. Optimize costs using spot instances and resource right-sizing
6. Implement GitOps workflow with automated promotion between environments

---

**Remember**: Expertise comes from **doing**, not just reading. Use this review as a foundation, but the real learning happens when you:
1. Try the concepts in your own environment
2. Encounter and solve real problems
3. Build mental models through practice
4. Teach others what you've learned

Keep building, keep breaking things (in safe environments), and keep learning! 🚀

*Review compiled from Days 1-15 of EKS_Learning materials*
*Last updated: September 2026*