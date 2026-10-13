# 📅 Day 9: EKS Cluster Setup with eksctl

## 🎯 Learning Objectives
By the end of today, you will understand:
- How to create an EKS cluster using eksctl
- Node groups configuration
- Cluster validation and testing
- Deploy your first application
- Cluster deletion

---

## 🧠 Plain-English terms before you start

| Term | Meaning | Why it matters |
|---|---|---|
| **eksctl** | A command-line tool for creating and managing EKS clusters. | It automates many AWS resources and Kubernetes setup steps. |
| **Cluster** | The managed Kubernetes environment plus its configured networking and compute. | Cluster creation affects cost and permissions, not just a single server. |
| **Node group** | A set of EC2 worker nodes managed as a unit. | It defines the capacity that can host Pods. |
| **kubeconfig** | A local configuration file describing how kubectl reaches a cluster and obtains credentials. | Incorrect or stale configuration can cause confusing connection failures. |
| **Context** | The selected cluster and identity settings kubectl will use. | Always confirm the context before making changes. |
| **Region** | An AWS geographic area containing resources. | The cluster and related resources must be queried in the correct Region. |

**Analogy:** eksctl is a setup assistant; the Region is the site, the cluster is the facility, and kubeconfig is the connection card for the facility.

**Try / observe:** `aws sts get-caller-identity`, `eksctl get cluster --region <region>`, `kubectl config get-contexts`, `kubectl config current-context`.

**Troubleshooting clue:** If a cluster is not found, verify the AWS account identity, Region, cluster name, and active context before recreating anything. Cluster creation can incur charges.

**Explain it back:** What does eksctl automate? Why should you inspect the current context before running apply/delete commands?

---

## 📖 Core Concepts

### eksctl Overview

**eksctl** is a simple CLI tool for creating and managing clusters on EKS. It automates the process of creating VPCs, subnets, IAM roles, and EKS clusters.

### Cluster Creation Flow

```
┌─────────────────────────────────────────────────────────┐
│              eksctl CLUSTER CREATION FLOW                 │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. Run eksctl create cluster                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  eksctl create cluster \                │   │   │
│  │  │    --name my-cluster \                  │   │   │
│  │  │    --region us-west-2 \                 │   │   │
│  │  │    --nodes 2 \                          │   │   │
│  │  │    --node-type t3.medium                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  2. Create VPC (if needed)                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Create VPC                           │   │   │
│  │  │  • Create public subnets                │   │   │
│  │  │  • Create private subnets               │   │   │
│  │  │  • Create Internet Gateway              │   │   │
│  │  │  • Create NAT Gateway                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  3. Create IAM Roles                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • EKS Cluster Role                     │   │   │
│  │  │  • Node IAM Role                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  4. Create EKS Control Plane                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • API Server                           │   │   │
│  │  │  • etcd                                 │   │   │
│  │  │  • Scheduler                            │   │   │
│  │  │  • Controller Manager                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  5. Create Node Group                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Launch EC2 instances                 │   │   │
│  │  │  • Install kubelet                      │   │   │
│  │  │  • Join cluster                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  6. Configure kubectl                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  eksctl utils write-kubeconfig \        │   │   │
│  │  │    --cluster my-cluster                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  7. Cluster Ready!                              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl get nodes                       │   │   │
│  │  │  kubectl cluster-info                    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step-by-Step Cluster Creation

### Step 1: Create Cluster

```bash
# Basic cluster creation
eksctl create cluster \
  --name my-eks-cluster \
  --region us-west-2 \
  --nodes 2 \
  --node-type t3.medium \
  --managed

# With specific VPC
eksctl create cluster \
  --name my-eks-cluster \
  --region us-west-2 \
  --vpc-public-subnets subnet-abc123,subnet-def456 \
  --vpc-private-subnets subnet-ghi789,subnet-jkl012 \
  --nodes 2 \
  --node-type t3.medium
```

### Step 2: Verify Cluster

```bash
# Get cluster info
eksctl get cluster --name my-eks-cluster --region us-west-2

# Get node info
kubectl get nodes
kubectl get nodes -o wide

# Cluster info
kubectl cluster-info
```

### Step 3: Deploy Test Application

```bash
# Create deployment
kubectl create deployment nginx --image=nginx --replicas=2

# Expose deployment
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Check status
kubectl get deployments
kubectl get pods
kubectl get svc nginx
```

### Step 4: Access Application

```bash
# Get external IP
kubectl get svc nginx --watch

# Once EXTERNAL-IP is available, access the demo endpoint:
curl http://<external-ip>

# HTTP is used here only to keep the lab simple; use HTTPS for real applications.
# A LoadBalancer Service can create billable AWS resources. Delete the lab resources when finished.
```

---

## 📊 Node Group Configuration

### Managed Node Group Options

```bash
# Create node group with specific configuration
eksctl create nodegroup \
  --cluster my-eks-cluster \
  --name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --node-volume-size 50 \
  --ssh-access \
  --ssh-public-key ~/.ssh/id_rsa.pub \
  --managed
```

### Node Group Options

```
┌─────────────────────────────────────────────────────────┐
│              NODE GROUP OPTIONS                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Instance Types                                 │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • t3.medium (General purpose)          │   │   │
│  │  │  • m5.large (Compute optimized)         │   │   │
│  │  │  • r5.large (Memory optimized)          │   │   │
│  │  │  • c5.xlarge (CPU intensive)            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Scaling Options                                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • --nodes (fixed count)                 │   │   │
│  │  │  • --nodes-min / --nodes-max (auto)      │   │   │
│  │  │  • Cluster Autoscaler                    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Storage Options                                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • --node-volume-size (GB)              │   │   │
│  │  │  • EBS volume type (gp2, gp3)           │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Network Options                                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • --vpc-public-subnets                 │   │   │
│  │  │  • --vpc-private-subnets                │   │   │
│  │  │  • --ssh-access                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Practice Exercises

### Exercise 1: Create Your First EKS Cluster

```bash
# Create cluster
eksctl create cluster \
  --name my-first-cluster \
  --region us-west-2 \
  --nodes 2 \
  --node-type t3.medium

# Wait for cluster to be created (takes 10-15 minutes)

# Verify
kubectl get nodes
kubectl cluster-info
```

### Exercise 2: Deploy and Test Application

```bash
# Create nginx deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Check pods
kubectl get pods -o wide

# Expose as LoadBalancer
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Get external IP
kubectl get svc nginx --watch

# Test application
curl http://<external-ip>
```

### Exercise 3: Scale Application

```bash
# Scale deployment
kubectl scale deployment nginx --replicas=5

# Watch pods
kubectl get pods -w

# Check node resources
kubectl top nodes
```

---

## 🗑️ Cluster Deletion

```bash
# Delete cluster
eksctl delete cluster --name my-cluster --region us-west-2

# Verify deletion
eksctl get cluster --name my-cluster --region us-west-2
```

---

## 🐛 Troubleshooting Guide

### Cluster Creation Fails

```bash
# Check AWS CLI configuration
aws sts get-caller-identity

# Check permissions
# Ensure your IAM user has:
# - AmazonEKSClusterPolicy
# - AmazonEKSServicePolicy
# - AmazonEC2FullAccess (for VPC)

# Check eksctl logs
eksctl create cluster --name my-cluster --region us-west-2 --verbose 4
```

### Nodes Not Joining Cluster

```bash
# Check node group status
eksctl get nodegroup --cluster my-cluster --region us-west-2

# Check node logs (SSH into node)
ssh ec2-user@<node-ip>
sudo journalctl -u kubelet -f
```

---

## 📝 Interview Questions

1. **What is eksctl and why use it?**
   - **Answer:** eksctl is a CLI tool for creating and managing EKS clusters. It automates VPC creation, IAM roles, and cluster setup, making it much easier than manual configuration.

2. **What are the prerequisites for creating an EKS cluster?**
   - **Answer:** AWS CLI configured, eksctl installed, kubectl installed, and IAM permissions for EKS, EC2, VPC, and IAM.

3. **How do you choose node types for EKS?**
   - **Answer:** Consider workload requirements (CPU, memory), cost, and scaling needs. t3.medium for general, m5.large for compute, r5.large for memory-intensive.

4. **What is the difference between managed and self-managed node groups?**
   - **Answer:** AWS manages the managed node group service and several lifecycle operations, but version/AMI updates still require a planned rollout or configured update process. Self-managed nodes require more direct operational work.

5. **How do you scale an EKS cluster?**
   - **Answer:** Use Cluster Autoscaler for automatic scaling based on pod requirements, or manually update node group size with eksctl.

---

## 📚 Resources

- [eksctl Documentation](https://eksctl.io/)
- [EKS Getting Started](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- [EKS Workshop](https://www.eksworkshop.com/)

---

## 🎯 Today's Assignment

1. **Create:** An EKS cluster with eksctl
2. **Deploy:** A test application (nginx)
3. **Access:** The application via LoadBalancer
4. **Document:** The cluster creation process

---

## 📊 Progress Tracker

```
Day 8: Introduction to Amazon EKS ✅
Day 9: EKS Cluster Setup with eksctl ⬜
```

---

## 🔗 Next Day Preview

**Day 10:** EKS Networking and VPC
- VPC configuration for EKS
- Subnet types (Public/Private)
- NAT Gateway vs Internet Gateway
- Security Groups for EKS

---

**Remember:** Practice creating and deleting clusters to understand the process! 🚀
