# 🚀 Complete EKS Learning Plan - From Zero to Hero

## 📋 Overview
- **Duration:** 60 Days (2 Months)
- **Level:** Beginner → Advanced → Production Expert
- **Focus:** AWS EKS + Kubernetes + Production Real-time Use Cases

---

## 🗓️ Phase 1: Foundation (Days 1-15)

### Week 1: Kubernetes Basics (Days 1-7)

#### Day 1: What is Kubernetes?
**Core Concepts:**
- Container Orchestration explained
- Why Kubernetes? (Problems it solves)
- Kubernetes vs Docker Swarm vs Docker Compose
- Kubernetes Architecture Overview

**Diagram: High-Level Kubernetes Architecture**
```
┌─────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │   API    │  │ Scheduler│  │Controller│  │  etcd  │ │
│  │  Server  │  │          │  │ Manager  │  │        │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     WORKER NODES                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │    │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │    │
│  │ │  Pods   │ │  │ │  Pods   │ │  │ │  Pods   │ │    │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │    │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │    │
│  │ │  Pods   │ │  │ │  Pods   │ │  │ │  Pods   │ │    │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Key Components:**
- **API Server:** Entry point for all commands
- **Scheduler:** Assigns pods to nodes
- **Controller Manager:** Maintains desired state
- **etcd:** Key-value store for cluster data
- **Kubelet:** Agent on each node
- **Kube-proxy:** Network proxy on each node

**Real-time Use Case:**
- Airbnb uses Kubernetes to manage 1000+ microservices
- Spotify runs 2M+ pods daily on Kubernetes

**Interview Questions (Beginner):**
1. What is Kubernetes and why is it used?
2. What are the main components of Kubernetes architecture?
3. What is the difference between Master and Worker nodes?
4. What is etcd and its role?

**Practice:** Watch Kubernetes 101 video, read official docs introduction

---

#### Day 2: Pods, ReplicaSets, and Deployments
**Core Concepts:**
- Pod: Smallest deployable unit
- ReplicaSet: Maintains desired pod count
- Deployment: Manages ReplicaSets and provides rolling updates

**Diagram: Deployment → ReplicaSet → Pods**
```
┌──────────────────┐
│    Deployment    │
│  (my-app-deploy)│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   ReplicaSet     │
│ (my-app-rs-abc)  │
└────────┬─────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│ Pod 1 │ │ Pod 2 │
│ ┌───┐ │ │ ┌───┐ │
│ │ 🐳│ │ │ │ 🐳│ │
│ └───┘ │ │ └───┘ │
└───────┘ └───────┘
```

**YAML Examples:**
```yaml
# Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80

---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
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
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**Commands:**
```bash
kubectl get pods
kubectl get deployments
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash
```

**Real-time Use Case:**
- Deployment strategies: Rolling update, Blue-Green, Canary
- Zero-downtime deployments

**Interview Questions:**
1. What is a Pod in Kubernetes?
2. What is the difference between Pod and Container?
3. How does a Deployment differ from a ReplicaSet?
4. What are liveness and readiness probes?

**Practice:** Create your first pod and deployment

---

#### Day 3: Services and Networking
**Core Concepts:**
- Service: Expose applications internally/externally
- Types: ClusterIP, NodePort, LoadBalancer, ExternalName

**Diagram: Service Types**
```
┌─────────────────────────────────────────────────────────┐
│                    SERVICE TYPES                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ClusterIP (Default)     NodePort                       │
│  ┌─────────┐            ┌─────────┐                     │
│  │ Service │            │ Service │                     │
│  │  ClusterIP│          │  NodePort│                    │
│  └────┬────┘            └────┬────┘                     │
│       │                      │                          │
│  Internal Only          External Access                 │
│  Pod → Pod              NodeIP:NodePort                 │
│                                                         │
│  LoadBalancer            ExternalName                   │
│  ┌─────────┐            ┌─────────┐                     │
│  │ Service │            │ Service │                     │
│  │   LB    │            │  CNAME  │                     │
│  └────┬────┘            └────┬────┘                     │
│       │                      │                          │
│  AWS ELB/ALB            DNS Redirect                    │
│  External Access         External DNS                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**YAML Examples:**
```yaml
# ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080

---
# LoadBalancer (AWS ELB)
apiVersion: v1
kind: Service
metadata:
  name: my-lb-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

**Commands:**
```bash
kubectl get services
kubectl describe service <service-name>
kubectl get endpoints <service-name>
```

**Real-time Use Case:**
- Microservices communication via ClusterIP
- External access via LoadBalancer (AWS ELB)

**Interview Questions:**
1. What are the different types of Kubernetes Services?
2. When would you use ClusterIP vs LoadBalancer?
3. How does kube-proxy work?
4. What is a Headless Service?

**Practice:** Create services and test connectivity

---

#### Day 4: Namespaces and ConfigMaps/Secrets
**Core Concepts:**
- Namespaces: Virtual cluster partitioning
- ConfigMaps: Non-confidential configuration
- Secrets: Confidential data (base64 encoded)

**Diagram: Namespace Isolation**
```
┌─────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                    │
│                                                         │
│  ┌─────────────────┐    ┌─────────────────┐            │
│  │    namespace:    │    │    namespace:    │            │
│  │    development   │    │    production    │            │
│  │                  │    │                  │            │
│  │  ┌────┐ ┌────┐  │    │  ┌────┐ ┌────┐  │            │
│  │  │Pod │ │Pod │  │    │  │Pod │ │Pod │  │            │
│  │  └────┘ └────┘  │    │  └────┘ └────┘  │            │
│  │  ┌────┐        │    │  ┌────┐ ┌────┐  │            │
│  │  │Pod │        │    │  │Pod │ │Pod │  │            │
│  │  └────┘        │    │  └────┘ └────┘  │            │
│  └─────────────────┘    └─────────────────┘            │
│                                                         │
│  ┌─────────────────┐    ┌─────────────────┐            │
│  │    namespace:    │    │    namespace:    │            │
│  │     monitoring   │    │       kube-      │            │
│  │                  │    │      system      │            │
│  │  ┌────────────┐ │    │  ┌────────────┐  │            │
│  │  │ Prometheus │ │    │  │   CoreDNS  │  │            │
│  │  └────────────┘ │    │  └────────────┘  │            │
│  └─────────────────┘    └─────────────────┘            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**YAML Examples:**
```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: development

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "mysql-service"
  DATABASE_PORT: "3306"
  LOG_LEVEL: "debug"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_USERNAME: YWRtaW4=  # base64 encoded
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64 encoded
```

**Commands:**
```bash
# Namespaces
kubectl get namespaces
kubectl create namespace development
kubectl config set-context --current --namespace=development

# ConfigMaps
kubectl create configmap app-config --from-literal=DB_HOST=mysql
kubectl get configmap app-config -o yaml

# Secrets
kubectl create secret generic app-secrets --from-literal=DB_PASSWORD=password123
kubectl get secret app-secrets -o yaml
```

**Interview Questions:**
1. What are Namespaces and why use them?
2. What is the difference between ConfigMap and Secret?
3. How are Secrets stored in Kubernetes?
4. How to mount ConfigMap/Secret in a pod?

**Practice:** Create namespace, configmap, and secret

---

#### Day 5: Persistent Storage
**Core Concepts:**
- PersistentVolume (PV): Cluster-level storage
- PersistentVolumeClaim (PVC): Request for storage
- StorageClass: Dynamic provisioning

**Diagram: PV-PVC-StorageClass**
```
┌─────────────────────────────────────────────────────────┐
│                   Persistent Storage                    │
│                                                         │
│  ┌─────────────┐                                        │
│  │  Pod         │                                        │
│  │  ┌───────┐  │                                        │
│  │  │Volume │  │                                        │
│  │  └───┬───┘  │                                        │
│  └──────┼──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │    PVC       │  PersistentVolumeClaim                │
│  │  (request)   │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │     PV       │  PersistentVolume                     │
│  │  (actual)    │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │ StorageClass │  Dynamic Provisioner                  │
│  │ (AWS EBS)   │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │   AWS EBS   │  Actual Storage                        │
│  │   Volume     │                                        │
│  └─────────────┘                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**YAML Examples:**
```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 10Gi

---
# Pod using PVC
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

**Real-time Use Case:**
- Stateful applications (databases, file servers)
- Persistent logging and data storage

**Interview Questions:**
1. What is the difference between PV and PVC?
2. What is StorageClass and when to use it?
3. What access modes are available for PV?
4. How to perform volume snapshots?

**Practice:** Create PVC and mount in a pod

---

#### Day 6: ConfigMaps and Secrets in Practice
**Hands-on Labs:**
- Create ConfigMap from file
- Create Secret from literals
- Environment variable injection
- Volume mount for ConfigMap/Secret

**Real-time Use Case:**
- Database credentials management
- Application configuration management
- TLS certificate management

**Practice:** Deploy an app with ConfigMap and Secret

---

#### Day 7: Kubernetes Networking Deep Dive
**Core Concepts:**
- Pod-to-Pod communication
- Service discovery
- DNS in Kubernetes
- Network Policies

**Diagram: Kubernetes Networking**
```
┌─────────────────────────────────────────────────────────┐
│                   Network Flow                          │
│                                                         │
│  Pod A ──────► Service ──────► Pod B                    │
│    │              │              │                      │
│    │              │              │                      │
│    ▼              ▼              ▼                      │
│  ┌─────┐    ┌─────────┐    ┌─────┐                     │
│  │eth0 │    │ kube-   │    │eth0 │                     │
│  │10.0 │    │ proxy   │    │10.0 │                     │
│  │.1.1 │    │ iptables│    │.1.2 │                     │
│  └─────┘    └─────────┘    └─────┘                     │
│                                                         │
│  All pods can communicate with each other               │
│  Services provide stable DNS names                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Commands:**
```bash
# Test DNS resolution
kubectl run dns-test --image=busybox --rm -it -- nslookup my-service

# Test connectivity
kubectl exec -it <pod> -- curl <service-name>

# Network Policies
kubectl get networkpolicies
```

**Interview Questions:**
1. How does Kubernetes networking work?
2. What is CNI (Container Network Interface)?
3. What are Network Policies?
4. How does service discovery work in Kubernetes?

**Practice:** Test pod-to-pod and pod-to-service communication

---

### Week 2: AWS EKS Fundamentals (Days 8-14)

#### Day 8: Introduction to Amazon EKS
**Core Concepts:**
- What is Amazon EKS?
- EKS vs Self-managed Kubernetes
- EKS Pricing model
- EKS Control Plane and Worker Nodes

**Diagram: AWS EKS Architecture**
```
┌─────────────────────────────────────────────────────────┐
│                      AWS Cloud                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Amazon EKS Control Plane            │   │
│  │  (AWS Managed - Multi-AZ, Encrypted, HA)        │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐           │   │
│  │  │ API  │ │Sched │ │ Ctrl │ │ etcd │           │   │
│  │  │Server│ │uler  │ │ Mgr  │ │      │           │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘           │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              VPC (Your AWS Account)              │   │
│  │                                                   │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  Public     │  │  Private    │              │   │
│  │  │  Subnet     │  │  Subnet     │              │   │
│  │  │  ┌───────┐ │  │  ┌───────┐ │              │   │
│  │  │  │  NAT  │ │  │  │  EKS  │ │              │   │
│  │  │  │Gateway│ │  │  │ Worker│ │              │   │
│  │  │  └───────┘ │  │  │ Nodes │ │              │   │
│  │  └─────────────┘  │  └───────┘ │              │   │
│  │                    └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────┐  ┌─────────────────┐              │
│  │   Amazon ECR    │  │  Amazon VPC     │              │
│  │  (Container     │  │  (Networking)   │              │
│  │   Registry)     │  │                 │              │
│  └─────────────────┘  └─────────────────┘              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**EKS Benefits:**
- **Managed Control Plane:** AWS manages master nodes
- **High Availability:** Multi-AZ deployment
- **Security:** AWS IAM integration, encryption at rest
- **Scalability:** Auto Scaling with Cluster Autoscaler
- **Compliance:** SOC, PCI, HIPAA compliant

**EKS Pricing:**
- Control Plane: $0.10/hour ($73/month)
- Worker Nodes: EC2 instance costs
- Data Transfer: Standard AWS rates

**Real-time Use Case:**
- Netflix uses EKS for streaming infrastructure
- Samsung runs IoT workloads on EKS

**Interview Questions:**
1. What is Amazon EKS and how does it differ from self-managed Kubernetes?
2. What are the benefits of using EKS?
3. What is the cost structure of EKS?
4. How does EKS handle high availability?

**Practice:** AWS Free Tier setup, EKS console walkthrough

---

#### Day 9: EKS Cluster Setup with eksctl
**Core Concepts:**
- eksctl: CLI tool for EKS
- Cluster creation options
- Node groups configuration
- IAM roles and policies

**Commands:**
```bash
# Install eksctl
# macOS
brew install weaveworks/tap/eksctl

# Windows
choco install eksctl

# Create cluster
eksctl create cluster \
  --name my-eks-cluster \
  --region us-west-2 \
  --nodes 2 \
  --node-type t3.medium \
  --managed

# Verify cluster
eksctl get cluster
kubectl get nodes
kubectl cluster-info

# Delete cluster
eksctl delete cluster --name my-eks-cluster --region us-west-2
```

**Diagram: eksctl Cluster Creation Flow**
```
┌─────────────────────────────────────────────────────────┐
│              eksctl Cluster Creation Flow                │
│                                                         │
│  ┌─────────────┐                                        │
│  │  eksctl      │                                        │
│  │  create      │                                        │
│  │  cluster     │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │  Create VPC │                                        │
│  │  (if needed) │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │  Create IAM │                                        │
│  │  Roles      │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │  Create EKS │                                        │
│  │  Control    │                                        │
│  │  Plane      │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │  Create     │                                        │
│  │  Node Group │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │  Configure  │                                        │
│  │  kubectl    │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │  Cluster    │                                        │
│  │  Ready!     │                                        │
│  └─────────────┘                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Real-time Use Case:**
- Infrastructure as Code (IaC) for Kubernetes
- Automated cluster provisioning

**Practice:** Create your first EKS cluster

---

#### Day 10: EKS Networking and VPC
**Core Concepts:**
- VPC configuration for EKS
- Subnet types (Public/Private)
- NAT Gateway vs Internet Gateway
- Security Groups for EKS

**Diagram: EKS VPC Architecture**
```
┌─────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16)                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Public Subnets (Auto-assign Public IP) │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  AZ-1a      │  │  AZ-1b      │              │   │
│  │  │ 10.0.1.0/24 │  │ 10.0.2.0/24 │              │   │
│  │  │  ┌───────┐  │  │  ┌───────┐  │              │   │
│  │  │  │  EKS  │  │  │  │  EKS  │  │              │   │
│  │  │  │Worker │  │  │  │Worker │  │              │   │
│  │  │  │Node   │  │  │  │Node   │  │              │   │
│  │  │  └───────┘  │  │  └───────┘  │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Private Subnets (No Public IP)         │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  AZ-1a      │  │  AZ-1b      │              │   │
│  │  │ 10.0.3.0/24 │  │ 10.0.4.0/24 │              │   │
│  │  │  ┌───────┐  │  │  ┌───────┐  │              │   │
│  │  │  │  EKS  │  │  │  │  EKS  │  │              │   │
│  │  │  │Worker │  │  │  │Worker │  │              │   │
│  │  │  │Node   │  │  │  │Node   │  │              │   │
│  │  │  └───────┘  │  │  └───────┘  │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  │                    ┌───────┐                    │   │
│  │                    │  NAT  │                    │   │
│  │                    │Gateway│                    │   │
│  │                    └───────┘                    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Key Points:**
- Use private subnets for worker nodes (security best practice)
- NAT Gateway for outbound internet access
- Internet Gateway for public subnets
- Multiple AZs for high availability

**Interview Questions:**
1. Why use private subnets for EKS worker nodes?
2. What is the difference between NAT Gateway and Internet Gateway?
3. How to configure VPC for EKS?
4. What are the networking requirements for EKS?

**Practice:** Review VPC configuration in AWS Console

---

#### Day 11: EKS IAM Roles and Policies
**Core Concepts:**
- IAM Roles for Service Accounts (IRSA)
- Node IAM Role
- EKS Cluster IAM Role
- OIDC Provider for IRSA

**Diagram: IAM Roles Architecture**
```
┌─────────────────────────────────────────────────────────┐
│                   IAM Roles for EKS                     │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │          EKS Cluster IAM Role                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Permissions:                            │   │   │
│  │  │  - eks:CreateCluster                    │   │   │
│  │  │  - eks:DescribeCluster                  │   │   │
│  │  │  - ec2:Describe*                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │          Node IAM Role                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Permissions:                            │   │   │
│  │  │  - ecr:GetAuthorizationToken            │   │   │
│  │  │  - ecr:BatchCheckLayerAvailability      │   │   │
│  │  │  - ecr:GetDownloadUrlForLayer           │   │   │
│  │  │  - ecr:BatchGetImage                   │   │   │
│  │  │  - logs:CreateLogGroup                  │   │   │
│  │  │  - logs:CreateLogStream                 │   │   │
│  │  │  - logs:PutLogEvents                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │       IAM Role for Service Accounts (IRSA)      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  OIDC Provider                          │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  Pod → Service Account → IAM    │   │   │   │
│  │  │  │  Role → AWS Service             │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Commands:**
```bash
# Create OIDC Provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-eks-cluster \
  --approve

# Create IRSA for S3 access
eksctl create iamserviceaccount \
  --cluster=my-eks-cluster \
  --name=s3-access-sa \
  --namespace=default \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

**Interview Questions:**
1. What is IRSA (IAM Roles for Service Accounts)?
2. How does IRSA work with OIDC?
3. What are the different IAM roles in EKS?
4. How to grant AWS permissions to pods?

**Practice:** Create IRSA for S3 access

---

#### Day 12: EKS Storage with EBS
**Core Concepts:**
- Amazon EBS CSI Driver
- StorageClass configuration
- Volume types for EKS
- Dynamic provisioning

**Diagram: EBS CSI Driver**
```
┌─────────────────────────────────────────────────────────┐
│              EBS CSI Driver Architecture                 │
│                                                         │
│  ┌─────────────┐                                        │
│  │   Pod        │                                        │
│  │  ┌───────┐  │                                        │
│  │  │Volume │  │                                        │
│  │  └───┬───┘  │                                        │
│  └──────┼──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │  PVC        │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │  StorageClass│                                       │
│  │  (ebs-sc)   │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │ EBS CSI     │                                        │
│  │ Driver      │                                        │
│  └──────┬──────┘                                        │
│         │                                               │
│         ▼                                               │
│  ┌─────────────┐                                        │
│  │ Amazon EBS  │                                        │
│  │ Volume      │                                        │
│  └─────────────┘                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Commands:**
```bash
# Install EBS CSI Driver
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster my-eks-cluster \
  --force

# Create StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
EOF

# Create PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 20Gi
EOF
```

**Real-time Use Case:**
- Database storage (RDS on EKS)
- Persistent logging
- Stateful applications

**Practice:** Install EBS CSI Driver and create a volume

---

#### Day 13: EKS Load Balancing with ALB
**Core Concepts:**
- AWS Load Balancer Controller
- ALB Ingress
- NLB Service
- Target Group Binding

**Diagram: ALB Ingress Controller**
```
┌─────────────────────────────────────────────────────────┐
│           AWS Load Balancer Controller                   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                    Internet                       │   │
│  └─────────────────────┬───────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Application Load Balancer            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Listener Rules:                         │   │   │
│  │  │  - /app/* → Target Group 1 (service-a)  │   │   │
│  │  │  - /api/* → Target Group 2 (service-b)  │   │   │
│  │  │  - /*     → Target Group 3 (frontend)   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  Target     │ │  Target     │ │  Target     │      │
│  │  Group 1    │ │  Group 2    │ │  Group 3    │      │
│  │  (service-a)│ │  (service-b)│ │  (frontend) │      │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘      │
│         │               │               │              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │    Pods     │ │    Pods     │ │    Pods     │      │
│  └─────────────┘ └─────────────┘ └─────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Commands:**
```bash
# Install AWS Load Balancer Controller
eksctl create addon \
  --name aws-load-balancer-controller \
  --cluster my-eks-cluster \
  --force

# Create Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-frontend-service
            port:
              number: 80
EOF
```

**Interview Questions:**
1. What is the difference between ALB and NLB?
2. How does the AWS Load Balancer Controller work?
3. What are Ingress annotations?
4. How to configure SSL/TLS on ALB?

**Practice:** Deploy ALB Ingress and test external access

---

#### Day 14: EKS Monitoring and Logging
**Core Concepts:**
- CloudWatch Container Insights
- Prometheus and Grafana
- AWS X-Ray for tracing
- Fluent Bit for logging

**Diagram: Monitoring Architecture**
```
┌─────────────────────────────────────────────────────────┐
│              EKS Monitoring Stack                        │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Amazon CloudWatch                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Container Insights                       │   │   │
│  │  │  - CPU/Memory metrics                    │   │   │
│  │  │  - Network metrics                       │   │   │
│  │  │  - Storage metrics                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Prometheus + Grafana                 │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Prometheus:                             │   │   │
│  │  │  - Metrics collection                   │   │   │
│  │  │  - Alerting rules                        │   │   │
│  │  │                                          │   │   │
│  │  │  Grafana:                                │   │   │
│  │  │  - Dashboards                            │   │   │
│  │  │  - Visualization                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Fluent Bit → CloudWatch Logs         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  - Container logs                       │   │   │
│  │  │  - Application logs                     │   │   │
│  │  │  - Audit logs                           │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Commands:**
```bash
# Install Container Insights
eksctl create addon \
  --name amazon-cloudwatch-observability \
  --cluster my-eks-cluster \
  --force

# View metrics in CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace ContainerInsights \
  --metric-name cpu_utilization \
  --dimensions Name=ClusterName,Value=my-eks-cluster \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

**Real-time Use Case:**
- Production monitoring and alerting
- Performance optimization
- Cost optimization through metrics

**Practice:** Set up CloudWatch Container Insights

---

#### Day 15: EKS Security Best Practices
**Core Concepts:**
- Pod Security Policies (PSP) / Pod Security Standards
- Network Policies
- Secret encryption
- Image scanning with ECR

**Diagram: EKS Security Layers**
```
┌─────────────────────────────────────────────────────────┐
│              EKS Security Layers                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 1: Cluster Security                       │   │
│  │  - IAM roles and policies                       │   │
│  │  - Cluster endpoint access                      │   │
│  │  - Encryption at rest (KMS)                     │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 2: Node Security                         │   │
│  │  - Private subnets                              │   │
│  │  - Security groups                              │   │
│  │  - Node IAM role (least privilege)              │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 3: Pod Security                          │   │
│  │  - Pod Security Standards                       │   │
│  │  - Security contexts                            │   │
│  │  - Read-only root filesystem                    │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 4: Container Security                    │   │
│  │  - Image scanning (ECR)                         │   │
│  │  - Image signing                                │   │
│  │  - Minimal base images                          │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 5: Network Security                      │   │
│  │  - Network Policies                             │   │
│  │  - Service mesh (Istio/App Mesh)                │   │
│  │  - WAF integration                              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Commands:**
```bash
# Enable encryption at rest
aws eks update-cluster-config \
  --name my-eks-cluster \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:us-west-2:ACCOUNT:key/KEY_ID"}}]'

# Restrict cluster endpoint access
aws eks update-cluster-config \
  --name my-eks-cluster \
  --resources-vpc-config 'endpointPublicAccess=false,endpointPrivateAccess=true'

# Scan images in ECR
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=latest
```

**Interview Questions:**
1. What are Pod Security Standards?
2. How to encrypt secrets in EKS?
3. What is IRSA and why is it important?
4. How to implement network policies in EKS?

**Practice:** Implement security best practices

---

## 🗓️ Phase 2: Intermediate (Days 16-30)

### Week 3-4: Advanced EKS Topics

#### Day 16-20: Advanced Networking
- VPC CNI Plugin
- Pod networking with AWS VPC
- Network policies with Calico
- Service mesh (Istio)

#### Day 21-25: Advanced Storage
- Amazon EFS CSI Driver
- FSx for Lustre
- S3 CSI Driver
- Volume snapshots and restore

#### Day 26-30: Advanced IAM and Security
- OIDC provider setup
- IRSA for multiple AWS services
- Pod-level IAM permissions
- AWS Secrets Manager integration

---

## 🗓️ Phase 3: Production Ready (Days 31-45)

### Week 5-6: Production Practices

#### Day 31-35: GitOps with ArgoCD
```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/user/repo
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### Day 36-40: CI/CD Pipelines
- AWS CodePipeline integration
- GitHub Actions with EKS
- Jenkins on EKS

#### Day 41-45: Monitoring and Observability
- Prometheus Operator
- Grafana dashboards
- AWS X-Ray tracing
- Custom metrics with CloudWatch

---

## 🗓️ Phase 4: Expert (Days 46-60)

### Week 7-8: Advanced Topics

#### Day 46-50: Multi-Cluster Management
- EKS Anywhere
- Multi-cluster networking
- Cluster Federation

#### Day 51-55: Cost Optimization
- Spot Instances for EKS
- Right-sizing nodes
- Cluster Autoscaler vs Karpenter

#### Day 56-60: Disaster Recovery
- Cluster backup strategies
- Cross-region replication
- Failover procedures

---

## 🎯 Complete Projects

### Project 1: Microservices Deployment (Days 1-15)
**Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│                  Microservices App                       │
│                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│  │ Frontend │  │  API    │  │  Auth   │                │
│  │ (React)  │  │ (Node)  │  │ (Go)   │                │
│  └────┬────┘  └────┬────┘  └────┬────┘                │
│       │            │            │                       │
│       └────────────┼────────────┘                       │
│                    │                                    │
│                    ▼                                    │
│            ┌─────────────┐                              │
│            │  Database   │                              │
│            │  (PostgreSQL)│                              │
│            └─────────────┘                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Deliverables:**
- [ ] Deploy frontend, API, and auth services
- [ ] Configure services and ingress
- [ ] Set up persistent storage for database
- [ ] Implement health checks and auto-scaling
- [ ] Configure logging and monitoring

---

### Project 2: Production EKS Cluster (Days 16-30)
**Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│                Production EKS Setup                      │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  VPC (Multi-AZ)                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  Public      │  │  Private     │              │   │
│  │  │  Subnets     │  │  Subnets     │              │   │
│  │  │  ┌───────┐  │  │  ┌───────┐  │              │   │
│  │  │  │  ALB  │  │  │  │  EKS  │  │              │   │
│  │  │  └───────┘  │  │  │Worker │  │              │   │
│  │  └─────────────┘  │  └───────┘  │              │   │
│  │                    └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────┐  ┌─────────────────┐              │
│  │  ECR Repository │  │  CloudWatch     │              │
│  │  (Container     │  │  (Monitoring)   │              │
│  │   Registry)     │  │                 │              │
│  └─────────────────┘  └─────────────────┘              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Deliverables:**
- [ ] Production VPC with private subnets
- [ ] EKS cluster with managed node groups
- [ ] ECR for container images
- [ ] ALB Ingress Controller
- [ ] CloudWatch monitoring

---

### Project 3: CI/CD Pipeline (Days 31-45)
**Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│                CI/CD Pipeline                            │
│                                                         │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐            │
│  │  Git    │────►│  CI     │────►│  CD     │            │
│  │  Push   │    │  Build  │    │  Deploy  │            │
│  └─────────┘    └────┬────┘    └────┬────┘            │
│                      │              │                   │
│                      ▼              ▼                   │
│               ┌─────────────┐ ┌─────────────┐          │
│               │    ECR      │ │    EKS      │          │
│               │  (Images)   │ │  (Deploy)   │          │
│               └─────────────┘ └─────────────┘          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Deliverables:**
- [ ] GitHub Actions workflow
- [ ] Build and push to ECR
- [ ] Deploy to EKS with ArgoCD
- [ ] Automated testing
- [ ] Rollback procedures

---

### Project 4: Complete Production Stack (Days 46-60)
**Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│              Complete Production Stack                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Application Layer                               │   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐              │   │
│  │  │App1 │ │App2 │ │App3 │ │App4 │              │   │
│  │  └─────┘ └─────┘ └─────┘ └─────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Infrastructure Layer                           │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │   EKS       │  │   RDS       │              │   │
│  │  │  Cluster    │  │  (Database) │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Operations Layer                                │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  Monitoring │  │  Logging    │              │   │
│  │  │  (Prometheus│  │  (Fluent Bit│              │   │
│  │  │   + Grafana)│  │   → CW)    │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Deliverables:**
- [ ] Complete microservices deployment
- [ ] Database integration (RDS/ElastiCache)
- [ ] Full monitoring stack
- [ ] CI/CD pipeline
- [ ] Security implementation
- [ ] Disaster recovery plan

---

## 📚 Interview Questions by Level

### Beginner Level (0-2 years)
1. What is Kubernetes and why is it used?
2. What is a Pod and how is it different from a container?
3. What is a Service in Kubernetes?
4. What are Namespaces?
5. What is a Deployment?
6. What is the difference between Docker and Kubernetes?
7. How does Kubernetes networking work?
8. What is a ConfigMap?
9. What is a Secret?
10. What are Labels and Selectors?

### Intermediate Level (2-5 years)
1. How does the Kubernetes scheduler work?
2. What are DaemonSets and when to use them?
3. How does Kubernetes handle rolling updates?
4. What are PersistentVolumes and PersistentVolumeClaims?
5. How does service discovery work in Kubernetes?
6. What are Network Policies?
7. How does horizontal pod autoscaling work?
8. What are StatefulSets?
9. How does Kubernetes handle secrets management?
10. What are init containers?

### Senior Level (5+ years)
1. How would you design a highly available Kubernetes cluster?
2. Explain Kubernetes networking in detail (CNI, DNS, Services)
3. How would you implement multi-tenancy in Kubernetes?
4. What are the security best practices for Kubernetes?
5. How would you handle disaster recovery in Kubernetes?
6. Explain the Kubernetes API and admission controllers
7. How would you optimize Kubernetes for cost?
8. What are the challenges of running stateful applications on Kubernetes?
9. How would you implement GitOps in Kubernetes?
10. Explain the differences between EKS, GKE, and AKS

### AWS EKS Specific Questions
1. What is the difference between EKS and self-managed Kubernetes?
2. How does EKS handle control plane updates?
3. What is IRSA and why is it important?
4. How would you configure networking for EKS?
5. What are the best practices for EKS security?
6. How does EKS integrate with AWS services?
7. What is the difference between Fargate and EC2 for EKS?
8. How would you set up multi-region EKS?
9. What is EKS Anywhere?
10. How would you optimize EKS costs?

---

## 📖 Daily Study Schedule

### Morning (2 hours)
- **30 min:** Read documentation/tutorial
- **30 min:** Watch video/course
- **1 hour:** Hands-on practice

### Evening (1 hour)
- **30 min:** Review concepts
- **30 min:** Practice interview questions

### Weekly Review
- **Weekend:** Build mini-project
- **Weekend:** Review and document learnings

---

## 📚 Recommended Resources

### Official Documentation
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [eksctl Documentation](https://eksctl.io/)

### Video Courses
- [A Cloud Guru - EKS](https://acloudguru.com/)
- [Udemy - Kubernetes for Beginners](https://www.udemy.com/)
- [YouTube - TechWorld with Nana](https://www.youtube.com/c/TechWorldwithNana)

### Hands-on Labs
- [Katacoda Kubernetes Scenarios](https://www.katacoda.com/courses/kubernetes)
- [KillerCoda EKS Labs](https://killercoda.com/playgrounds/scenario/eks)
- [AWS EKS Workshop](https://www.eksworkshop.com/)

### Practice Platforms
- [KodeKloud](https://kodekloud.com/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

---

## 🎯 Success Metrics

By the end of 60 days, you should be able to:
- [ ] Create and manage EKS clusters
- [ ] Deploy and scale applications on EKS
- [ ] Configure networking, storage, and security
- [ ] Set up monitoring and logging
- [ ] Implement CI/CD pipelines
- [ ] Design production-grade architectures
- [ ] Pass EKS certification exams
- [ ] Answer interview questions confidently

---

**Remember:** Consistency is key! Practice every day, build projects, and document your learnings. Good luck! 🚀
