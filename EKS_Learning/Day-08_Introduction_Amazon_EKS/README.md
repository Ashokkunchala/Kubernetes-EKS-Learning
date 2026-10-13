# 📅 Day 8: Introduction to Amazon EKS

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is Amazon EKS
- EKS vs Self-managed Kubernetes
- EKS Architecture and Components
- EKS Pricing Model
- Benefits of using EKS

---

## 🧠 Plain-English terms before you start

| Term | Everyday meaning | Why it matters |
|---|---|---|
| **Amazon EKS** | AWS's managed Kubernetes service. | AWS operates the Kubernetes control plane; you still manage workload design and much of the worker environment. |
| **Managed control plane** | AWS runs and maintains key Kubernetes API/control-plane components for you. | You do not log in to EKS control-plane machines like ordinary EC2 servers. |
| **Data plane** | The capacity where application Pods actually run, such as worker nodes. | Its CPU, memory, networking, and scaling affect workload availability. |
| **Node group** | A set of worker nodes managed together. | It provides capacity and a consistent node configuration. |
| **kubeconfig / context** | kubeconfig stores cluster connection and credential instructions; a context selects a cluster, user, and optional Namespace. | A wrong context can send commands to the wrong cluster. |
| **IAM** | AWS Identity and Access Management: identities, roles, and policies for AWS API access. | EKS users and workloads need a deliberate identity path; AWS IAM and Kubernetes RBAC are related but different systems. |

**Analogy:** EKS is like a managed building's control room; AWS maintains that room, while you still organise the workstations and applications inside.

**Try / observe:** `aws sts get-caller-identity`, `aws eks list-clusters --region <region>`, `kubectl config get-contexts`, `kubectl get nodes`.

**Troubleshooting clue:** If `kubectl` reports an authentication or access error, first confirm the AWS identity and current kubeconfig context. Then distinguish identity/authentication from Kubernetes authorization.

**Explain it back:** What does AWS manage in EKS, and what remains your responsibility? What is the difference between a kubeconfig context and an IAM role?

---

## 📖 Core Concepts

### What is Amazon EKS?

**Amazon EKS (Elastic Kubernetes Service)** is a managed Kubernetes service that makes it easy to run Kubernetes on AWS and on-premises.

**Key Features:**
- **Managed Control Plane:** AWS manages master nodes
- **High Availability:** Multi-AZ deployment
- **Security:** AWS IAM integration
- **Scalability:** Auto Scaling support
- **Compliance:** SOC, PCI, HIPAA compliant

### EKS Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      AWS Cloud                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Amazon EKS Control Plane            │   │
│  │  (AWS Managed - Multi-AZ, Encrypted, HA)        │   │
│  │                                                   │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐           │   │
│  │  │ API  │ │Sched │ │ Ctrl │ │ etcd │           │   │
│  │  │Server│ │uler  │ │ Mgr  │ │      │           │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘           │   │
│  │                                                   │   │
│  │  • Runs in AWS managed account                   │   │
│  │  • Multi-AZ deployment                          │   │
│  │  • Encrypted at rest (KMS)                      │   │
│  │  • 99.95% SLA                                   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              VPC (Your AWS Account)              │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Private Subnets                         │   │   │
│  │  │  ┌─────────────┐  ┌─────────────┐      │   │   │
│  │  │  │  EKS Worker │  │  EKS Worker │      │   │   │
│  │  │  │  Node 1     │  │  Node 2     │      │   │   │
│  │  │  │  (m5.large) │  │  (m5.large) │      │   │   │
│  │  │  └─────────────┘  └─────────────┘      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Public Subnets                          │   │   │
│  │  │  ┌─────────────┐  ┌─────────────┐      │   │   │
│  │  │  │  NAT        │  │  Internet   │      │   │   │
│  │  │  │  Gateway    │  │  Gateway    │      │   │   │
│  │  │  └─────────────┘  └─────────────┘      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Supporting AWS Services                         │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  Amazon ECR │  │  Amazon VPC │              │   │
│  │  │  (Registry) │  │  (Network)  │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  AWS IAM    │  │  Amazon     │              │   │
│  │  │  (Security) │  │  CloudWatch │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### EKS vs Self-Managed Kubernetes

```
┌─────────────────────────────────────────────────────────┐
│              EKS vs SELF-MANAGED KUBERNETES              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Amazon EKS                                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  ✅ Managed control plane                │   │   │
│  │  │  ✅ Automatic updates                    │   │   │
│  │  │  ✅ High availability built-in           │   │   │
│  │  │  ✅ AWS IAM integration                  │   │   │
│  │  │  ✅ Compliance (SOC, PCI, HIPAA)         │   │   │
│  │  │  ✅ AWS support                           │   │   │
│  │  │  ❌ Cost ($73/month control plane)       │   │   │
│  │  │  ❌ Less control over control plane      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Self-Managed Kubernetes                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  ✅ Full control over cluster            │   │   │
│  │  │  ✅ No control plane cost                │   │   │
│  │  │  ✅ Custom configurations                │   │   │
│  │  │  ❌ Manual management required           │   │   │
│  │  │  ❌ Manual updates                       │   │   │
│  │  │  ❌ High availability is complex         │   │   │
│  │  │  ❌ No AWS support                       │   │   │
│  │  │  ❌ More operational overhead            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### EKS Pricing

```
┌─────────────────────────────────────────────────────────┐
│                  EKS PRICING MODEL                       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Control Plane                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  $0.10/hour per cluster                  │   │   │
│  │  │  = $73/month per cluster                 │   │   │
│  │  │                                          │   │   │
│  │  │  Includes:                               │   │   │
│  │  │  • API Server                            │   │   │
│  │  │  • Scheduler                             │   │   │
│  │  │  • Controller Manager                    │   │   │
│  │  │  • etcd (managed)                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Worker Nodes                                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  EC2 instance costs                      │   │   │
│  │  │  (Pay for what you use)                  │   │   │
│  │  │                                          │   │   │
│  │  │  Example:                                │   │   │
│  │  │  • m5.large: $0.096/hour = $70/month    │   │   │
│  │  │  • 3 nodes = $210/month                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Total Monthly Cost (Example)                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Control Plane:  $73                    │   │   │
│  │  │  Worker Nodes:  $210 (3x m5.large)     │   │   │
│  │  │  Data Transfer: $50 (estimate)          │   │   │
│  │  │  ─────────────────────                  │   │   │
│  │  │  Total: ~$333/month                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🏗️ EKS Components

### Control Plane Components

```
┌─────────────────────────────────────────────────────────┐
│              EKS CONTROL PLANE COMPONENTS                │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AWS Account                                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • VPC (Your account)                    │   │   │
│  │  │  • Subnets (Your account)                │   │   │
│  │  │  • Security Groups (Your account)        │   │   │
│  │  │  • Worker Nodes (Your account)           │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AWS Managed Account (EKS)                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • API Server (ENI in your VPC)         │   │   │
│  │  │  • etcd (managed, encrypted)            │   │   │
│  │  │  • Scheduler (managed)                  │   │   │
│  │  │  • Controller Manager (managed)         │   │   │
│  │  │  • Automatic backups                    │   │   │
│  │  │  • Automatic updates                    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Worker Node Types

```
┌─────────────────────────────────────────────────────────┐
│              EKS WORKER NODE TYPES                       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. Managed Node Groups                          │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • AWS manages the nodes                │   │   │
│  │  │  • Automatic updates                    │   │   │
│  │  │  • Auto Scaling                         │   │   │
│  │  │  • Spot or On-Demand instances          │   │   │
│  │  │  • Recommended for most use cases       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  2. Self-Managed Node Groups                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • You manage the nodes                 │   │   │
│  │  │  • More control over configuration      │   │   │
│  │  │  • Manual updates                       │   │   │
│  │  │  • Use for custom AMIs                  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  3. Fargate (Serverless)                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • No nodes to manage                   │   │   │
│  │  │  • Pay per pod                          │   │   │
│  │  │  • Isolated environment                 │   │   │
│  │  │  • Limited instance types               │   │   │
│  │  │  • Good for batch workloads             │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🌍 Real-Time Use Cases

### Companies Using EKS

| Company | Use Case | Scale |
|---------|----------|-------|
| **Netflix** | Streaming infrastructure | Millions of pods |
| **Samsung** | IoT workloads | Global scale |
| **Expedia** | Travel platform | High availability |
| **GoDaddy** | Domain management | Enterprise scale |
| **Verizon** | 5G infrastructure | Low latency |

### Real-World Scenario

```
┌─────────────────────────────────────────────────────────┐
│              NETFLIX - EKS USE CASE                      │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Challenge:                                      │   │
│  │  • Millions of concurrent users                  │   │
│  │  • Global content delivery                       │   │
│  │  • Zero downtime deployments                     │   │
│  │  • Cost optimization                             │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Solution: Amazon EKS                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Multi-region EKS clusters            │   │   │
│  │  │  • Auto-scaling (Spot + On-Demand)      │   │   │
│  │  │  • Service mesh (Istio)                  │   │   │
│  │  │  • CI/CD with Spinnaker                  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Results:                                        │   │
│  │  • 99.99% availability                          │   │
│  │  • 60% cost reduction                           │   │
│  │  • 1000+ deployments/day                        │   │
│  │  • Global scale                                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ EKS Prerequisites

### AWS Account Setup

```bash
# Install AWS CLI
# Windows
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Verify installation
aws --version

# Configure AWS CLI
aws configure
# Enter:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region (us-west-2)
# - Default output format (json)
```

### Install eksctl

```bash
# Windows (using Chocolatey)
choco install eksctl

# macOS
brew install weaveworks/tap/eksctl

# Verify installation
eksctl version
```

### Install kubectl

```bash
# Windows
curl.exe -LO "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe"

# Or use Chocolatey
choco install kubernetes-cli

# Verify installation
kubectl version --client
```

---

## 📝 Practice Exercises

### Exercise 1: Verify AWS Configuration

```bash
# Check AWS identity
aws sts get-caller-identity

# Check available regions
aws ec2 describe-regions --query 'Regions[*].RegionName' --output table
```

### Exercise 2: Install and Verify Tools

```bash
# Verify installations
aws --version
eksctl version
kubectl version --client
```

---

## 📝 Interview Questions

### Beginner Level

1. **What is Amazon EKS?**
   - **Answer:** Amazon EKS is a managed Kubernetes service that makes it easy to run Kubernetes on AWS. It manages the control plane and integrates with AWS services.

2. **How does EKS differ from self-managed Kubernetes?**
   - **Answer:** EKS manages the control plane (API server, etcd, scheduler), provides automatic updates, high availability, and AWS integration. Self-managed requires you to manage everything.

3. **What is the cost structure of EKS?**
   - **Answer:** EKS charges $0.10/hour for the control plane ($73/month). Worker nodes are billed as EC2 instances. Data transfer charges apply.

4. **What are the benefits of using EKS?**
   - **Answer:** Managed control plane, high availability, AWS IAM integration, compliance certifications, automatic updates, and AWS support.

5. **What types of worker nodes does EKS support?**
   - **Answer:** Managed Node Groups, Self-Managed Node Groups, and Fargate (serverless).

### Intermediate Level

6. **How does EKS handle high availability?**
   - **Answer:** EKS runs the control plane across multiple Availability Zones. It automatically handles failover and provides a 99.95% SLA.

7. **What is the difference between Managed and Self-Managed Node Groups?**
   - **Answer:** Managed Node Groups are managed by AWS (automatic updates, scaling). Self-Managed gives you more control but requires manual management.

8. **How does EKS integrate with AWS IAM?**
   - **Answer:** EKS uses IAM for cluster authentication, node permissions, and IRSA (IAM Roles for Service Accounts) for pod-level permissions.

9. **What is Fargate in EKS?**
   - **Answer:** Fargate is a serverless compute engine for containers. It runs pods without managing EC2 instances. Good for batch workloads.

10. **How do you update an EKS cluster?**
    - **Answer:** Use `eksctl upgrade cluster` for the control plane. Update node groups separately. Test in non-production first.

---

## 📚 Resources

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [EKS Workshop](https://www.eksworkshop.com/)
- [eksctl Documentation](https://eksctl.io/)

---

## 🎯 Today's Assignment

1. **Install:** AWS CLI, eksctl, and kubectl
2. **Configure:** AWS CLI with your credentials
3. **Research:** EKS pricing for your use case
4. **Plan:** Your first EKS cluster architecture

---

## 📊 Progress Tracker

```
Week 1: ✅ Complete
Week 2 Day 8: Introduction to Amazon EKS ⬜
```

---

## 🔗 Next Day Preview

**Day 9:** EKS Cluster Setup with eksctl
- Create your first EKS cluster
- Configure node groups
- Test cluster connectivity
- Deploy your first application

---

**Remember:** EKS simplifies Kubernetes management on AWS! 🚀
