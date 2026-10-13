# 📅 Day 11: EKS IAM Roles and Policies

## 🎯 Learning Objectives
By the end of today, you will understand:
- IAM Roles for Service Accounts (IRSA)
- Node IAM Role
- EKS Cluster IAM Role
- OIDC Provider for IRSA
- Best practices for IAM in EKS

---

## 🧠 Plain-English terms before you start

| Term | Meaning | Why it matters |
|---|---|---|
| **Authentication (AuthN)** | Proving which identity is making a request. | A system must know who or what is calling before it evaluates permissions. |
| **Authorization (AuthZ)** | Deciding what an authenticated identity is allowed to do. | Knowing who you are does not mean you can perform every action. |
| **IAM role** | An AWS identity with permissions that a trusted principal can assume. | It allows temporary credentials without embedding long-lived keys in applications. |
| **IAM policy** | A JSON document describing allowed or denied AWS actions on resources under conditions. | It defines what an identity can do in AWS APIs. |
| **Trust policy** | The role's policy defining which principals may assume the role. | A permissions policy does not by itself decide who can assume a role. |
| **Kubernetes RBAC** | Kubernetes permissions expressed through Roles/ClusterRoles and bindings. | AWS IAM permissions and Kubernetes API permissions are not interchangeable. |

**Analogy:** Authentication is showing your ID; authorization is checking which rooms your badge can open. An IAM role's trust policy decides who may borrow the badge.

**EKS connection:** EKS access has multiple layers: AWS identity, EKS cluster access configuration, and Kubernetes RBAC where applicable. Diagnose the exact denied API before changing broad permissions.

**Try / observe:** `aws sts get-caller-identity`, `kubectl auth can-i get pods -n default`, `kubectl auth can-i get secrets -n default`.

**Troubleshooting clue:** `Unauthorized` often points to identity/authentication; `Forbidden` usually means the request reached authorization but lacks permission. Check AWS IAM and Kubernetes RBAC separately.

**Explain it back:** What is the difference between an IAM trust policy and a permissions policy? How do AuthN and AuthZ differ?

---

## 📖 Core Concepts

### IAM Roles Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   IAM ROLES FOR EKS                      │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EKS Cluster IAM Role                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Permissions:                            │   │   │
│  │  │  • eks:CreateCluster                    │   │   │
│  │  │  • eks:DescribeCluster                  │   │   │
│  │  │  • ec2:Describe*                        │   │   │
│  │  │  • ec2:CreateSecurityGroup              │   │   │
│  │  │  • ec2:CreateNetworkInterface           │   │   │
│  │  │                                          │   │   │
│  │  │  Trust Policy:                           │   │   │
│  │  │  • eks.amazonaws.com                    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Node IAM Role                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Permissions:                            │   │   │
│  │  │  • ecr:GetAuthorizationToken            │   │   │
│  │  │  • ecr:BatchCheckLayerAvailability      │   │   │
│  │  │  • ecr:GetDownloadUrlForLayer           │   │   │
│  │  │  • ecr:BatchGetImage                   │   │   │
│  │  │  • logs:CreateLogGroup                  │   │   │
│  │  │  • logs:CreateLogStream                 │   │   │
│  │  │  • logs:PutLogEvents                   │   │   │
│  │  │                                          │   │   │
│  │  │  Trust Policy:                           │   │   │
│  │  │  • ec2.amazonaws.com                    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  IAM Role for Service Accounts (IRSA)           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Pod-level IAM permissions            │   │   │
│  │  │  • Uses OIDC provider                   │   │   │
│  │  │  • AWS service access from pods         │   │   │
│  │  │  • No node-level permissions needed     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### IRSA (IAM Roles for Service Accounts)

```
┌─────────────────────────────────────────────────────────┐
│              IRSA ARCHITECTURE                            │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Pod                                              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  ServiceAccount: my-app-sa              │   │   │
│  │  │  Annotation:                            │   │   │
│  │  │  eks.amazonaws.com/role-arn:            │   │   │
│  │  │  arn:aws:iam::123456789012:role/my-role│   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  OIDC Provider                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Trusts EKS cluster                   │   │   │
│  │  │  • Validates service account tokens     │   │   │
│  │  │  • Maps to IAM role                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  IAM Role                                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Trust: OIDC provider                │   │   │
│  │  │  • Permissions: S3, DynamoDB, etc.     │   │   │
│  │  │  • Pod can assume this role            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AWS Services                                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • S3 (Bucket access)                   │   │   │
│  │  │  • DynamoDB (Table access)              │   │   │
│  │  │  • SQS (Queue access)                   │   │   │
│  │  │  • Secrets Manager (Secret access)      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step-by-Step IRSA Setup

### Step 1: Associate OIDC Provider

```bash
# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-eks-cluster \
  --region us-west-2 \
  --approve
```

### Step 2: Create IAM Role

```bash
# Create IAM role with policy
eksctl create iamserviceaccount \
  --cluster=my-eks-cluster \
  --name=s3-access-sa \
  --namespace=default \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

### Step 3: Deploy Pod with IRSA

```yaml
# pod-with-irsa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/AmazonS3ReadOnlyAccess
---
apiVersion: v1
kind: Pod
metadata:
  name: s3-access-pod
spec:
  serviceAccountName: s3-access-sa
  containers:
  - name: app
    image: amazon/aws-cli
    command: ["aws", "s3", "ls"]
```

### Step 4: Verify IRSA

```bash
# Deploy pod
kubectl apply -f pod-with-irsa.yaml

# Check pod
kubectl get pod s3-access-pod

# Check logs
kubectl logs s3-access-pod

# Test AWS access
kubectl exec -it s3-access-pod -- aws sts get-caller-identity
```

---

## 📝 Practice Exercises

### Exercise 1: Create IRSA for S3 Access

```bash
# Associate OIDC
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region us-west-2 \
  --approve

# Create service account with S3 access
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=s3-reader \
  --namespace=default \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Verify
kubectl get sa s3-reader -o yaml
```

### Exercise 2: Deploy Application with S3 Access

```yaml
# s3-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3-app
  template:
    metadata:
      labels:
        app: s3-app
    spec:
      serviceAccountName: s3-reader
      containers:
      - name: app
        image: amazon/aws-cli
        command: ["sleep", "3600"]
```

```bash
# Deploy
kubectl apply -f s3-app.yaml

# Test S3 access
kubectl exec -it <pod-name> -- aws s3 ls
```

---

## 📝 Interview Questions

1. **What is IRSA and why is it important?**
   - **Answer:** IRSA (IAM Roles for Service Accounts) allows pods to assume IAM roles. It provides fine-grained AWS permissions without giving node-level access.

2. **How does IRSA work with OIDC?**
   - **Answer:** EKS creates an OIDC provider that trusts the cluster. Service accounts get tokens that OIDC validates. IAM roles trust the OIDC provider, allowing pods to assume roles.

3. **What are the different IAM roles in EKS?**
   - **Answer:** 
     - Cluster IAM Role: For EKS control plane
     - Node IAM Role: For worker nodes
     - IRSA: For pod-level permissions

4. **How to grant AWS permissions to pods?**
   - **Answer:** Use IRSA to create a service account with an IAM role annotation. Deploy pods with that service account.

5. **What are the security benefits of IRSA?**
   - **Answer:** Fine-grained permissions, no node-level access needed, automatic credential rotation, audit trail via CloudTrail.

---

## 📚 Resources

- [IRSA Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [EKS IAM](https://docs.aws.amazon.com/eks/latest/userguide/security_iam.html)

---

## 🎯 Today's Assignment

1. **Set up:** OIDC provider for your cluster
2. **Create:** An IRSA for S3 access
3. **Deploy:** A pod with S3 access
4. **Test:** AWS service access from the pod

---

## 📊 Progress Tracker

```
Day 10: EKS Networking and VPC ✅
Day 11: EKS IAM Roles and Policies ⬜
```

---

## 🔗 Next Day Preview

**Day 12:** EKS Storage with EBS
- Amazon EBS CSI Driver
- StorageClass configuration
- Dynamic provisioning
- Hands-on with EBS volumes

---

**Remember:** IRSA is the best practice for pod-level AWS permissions! 🚀
