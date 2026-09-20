# 📅 Day 30: Advanced IAM Practice - Comprehensive Security Lab

## 🎯 Learning Objectives
By the end of today, you will:
- Build a full security architecture
- Combine RBAC, IRSA, and Secrets Manager
- Implement least privilege at scale
- Pass a comprehensive security assessment

---

## 🖥️ Project: Secure Microservices Platform

### Complete Architecture

```
┌─────────────────────────────────────────────────────────┐
│         SECURE MICROSERVICES PLATFORM                     │
│                                                         │
│                        EKS Cluster                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │             ┌─────────────────────┐             │   │
│  │             │  monitoring namespace │             │   │
│  │             │  ┌───────────────┐  │             │   │
│  │             │  │ prometheus-sa │  │  → CloudWatch│   │
│  │             │  └───────────────┘  │             │   │
│  │             └─────────────────────┘             │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────────┐ │   │
│  │  │            production namespace               │ │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐     │ │   │
│  │  │  │ api-sa  │  │worker-sa│  │report-sa│     │ │   │
│  │  │  │ IRSA:   │  │ IRSA:   │  │ IRSA:   │     │ │   │
│  │  │  │ S3,SQS  │  │DynamoDB │  │S3,Athena│     │ │   │
│  │  │  └─────────┘  └─────────┘  └─────────┘     │ │   │
│  │  └─────────────────────────────────────────────┘ │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────────┐ │   │
│  │  │            staging namespace                  │ │   │
│  │  │  ┌─────────┐  ┌─────────┐                   │ │   │
│  │  │  │ stg-sa  │  │ test-sa │                   │ │   │
│  │  │  │ limited │  │ read    │                   │ │   │
│  │  │  └─────────┘  └─────────┘                   │ │   │
│  │  └─────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  AWS Component:                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  OIDC Provider → IRSA → IAM Roles               │   │
│  │  Secrets Manager → CSI Driver → Pod Secrets     │   │
│  │  VPC Endpoints → Restrict service access        │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Step-by-Step Implementation

### Step 1: Create IAM Policies for Each Service

```bash
# API service policy (S3 + SQS)
cat > api-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["s3:PutObject", "s3:GetObject"],
            "Resource": "arn:aws:s3:::orders/*"
        },
        {
            "Effect": "Allow",
            "Action": ["sqs:SendMessage"],
            "Resource": "arn:aws:sqs:us-west-2:123456789012:order-queue"
        }
    ]
}
EOF

# Worker service policy (DynamoDB + SQS + S3)
cat > worker-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"],
            "Resource": "arn:aws:dynamodb:us-west-2:123456789012:table/Orders"
        },
        {
            "Effect": "Allow",
            "Action": ["sqs:ReceiveMessage", "sqs:DeleteMessage"],
            "Resource": "arn:aws:sqs:us-west-2:123456789012:order-queue"
        },
        {
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:PutObject"],
            "Resource": "arn:aws:s3:::processed/*"
        }
    ]
}
EOF

# Create policies
aws iam create-policy --policy-name api-service-policy --policy-document file://api-policy.json
aws iam create-policy --policy-name worker-service-policy --policy-document file://worker-policy.json
```

### Step 2: Create Service Accounts

```bash
# API service account
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=api-sa \
  --namespace=production \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/api-service-policy \
  --approve

# Worker service account
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=worker-sa \
  --namespace=production \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/worker-service-policy \
  --approve

# Monitoring service account
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=prometheus-sa \
  --namespace=monitoring \
  --attach-policy-arn=arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess \
  --approve

# Staging (limited access)
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=stg-sa \
  --namespace=staging \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

### Step 3: Set Up Secrets Manager

```bash
# Create secrets
aws secretsmanager create-secret \
  --name production/db-password \
  --secret-string '{"username":"prod_admin","password":"Pr0d#P@ssw0rd123"}'

aws secretsmanager create-secret \
  --name production/api-key \
  --secret-string '{"key":"prod-abc123xyz789"}' \
  --rotation-lambda-arn arn:aws:lambda:us-west-2:123456789012:function:rotate \
  --rotation-rules '{"AutomaticallyAfterDays":30}'

# Create secret access policy
cat > secret-access-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["secretsmanager:GetSecretValue"],
            "Resource": "arn:aws:secretsmanager:us-west-2:123456789012:secret:production/*-??????"
        }
    ]
}
EOF

aws iam create-policy --policy-name production-secret-access --policy-document file://secret-access-policy.json

# Attach to api-sa
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=api-sa \
  --namespace=production \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/production-secret-access \
  --approve
```

### Step 4: Apply Kubernetes RBAC

```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

```yaml
# rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: production-deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: production-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: production-deployer-binding
  namespace: production
subjects:
- kind: Group
  name: deployers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: production-deployer
  apiGroup: rbac.authorization.k8s.io
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: production-reader-binding
  namespace: production
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: production-reader
  apiGroup: rbac.authorization.k8s.io
```

### Step 5: Configure Network Security

```yaml
# network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-only-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: worker
  - to:
    - ipBlock:
        cidr: 10.100.0.0/16   # VPC CIDR for AWS endpoints
```
---

## 📝 Security Verification Checklist

### AWS Permissions
- [ ] Each service has minimal IAM permissions
- [ ] Trust policies restrict to specific service accounts
- [ ] No overly broad `*` permissions
- [ ] Secrets rotation enabled
- [ ] CloudTrail enabled for auditing

### Kubernetes RBAC
- [ ] Namespace-scoped roles
- [ ] No cluster-admin for regular users
- [ ] Production vs staging separation
- [ ] Regular permission audits

### Network Security
- [ ] Network policies applied
- [ ] VPC endpoints for AWS services
- [ ] No broad ingress
- [ ] EKS control plane private

### Secrets Management
- [ ] No secrets in etcd (using CSI driver)
- [ ] Auto rotation configured
- [ ] Least privilege secret access
- [ ] Encryption at rest

---

## 📝 Practice Exercises

### Exercise 1: Verify Least Privilege

```bash
# Test API pod - should have S3 + SQS but NOT DynamoDB
API_POD=$(kubectl get pods -n production -l app=api -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $API_POD -- aws s3 ls s3://orders/      # ✓ Works
kubectl exec -it $API_POD -- aws sqs get-queue-url --queue-name order-queue  # ✓ Works
kubectl exec -it $API_POD -- aws dynamodb list-tables   # ✗ Fails (good!)
```

### Exercise 2: Test RBAC

```bash
# Test read-only access
kubectl auth can-i delete pod -n production --as=system:serviceaccount:staging:stg-sa
# Should be NO

# Test deploy
kubectl auth can-i create deployment -n production --as=system:serviceaccount:production:api-sa
# Should be YES
```

---

## 📝 Final Security Assessment

### Assessment 1: Identify Problems

**Scenario:** You see this trust policy for an IRSA role. Identify 3 problems:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123:oidc-provider/oidc.eks..."
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc...:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

**Problems:**
1. **No sub condition** - ANY pod in the cluster can assume this role
2. **Broad IAM policies attached** - Check what permissions the role has
3. **No session duration limit** - Tokens may last too long

### Assessment 2: Design

**Complete the security matrix for this application:**

| Service | AWS Services | IAM Actions | K8s Permissions |
|---------|-------------|-------------|-----------------|
| API | S3, SQS | GetObject, PutObject, SendMessage | Deploy, scale |
| Worker | DynamoDB, SQS, S3 | GetItem, ReceiveMessage, GetObject | Read pods |
| Monitor | CloudWatch | GetMetricsData, DescribeLogs | Read all |
| Staging | S3 (read only) | GetObject | Read, limited deploy |

---

## 📚 Resources for Review

- [Day 26: OIDC Provider Setup](Day-26_OIDC_Provider_Setup/README.md)
- [Day 27: IRSA for Multiple Services](Day-27_IRSA_Multiple_Services/README.md)
- [Day 28: Pod-Level IAM](Day-28_Pod_Level_IAM_Permissions/README.md)
- [Day 29: Secrets Manager](Day-29_Secrets_Manager_Integration/README.md)

---

## 🎯 Today's Assignment

1. **Build:** The secure microservices platform
2. **Verify:** Least privilege for each service
3. **Test:** RBAC restrictions
4. **Review:** Security best practices
5. **Assess:** Complete the security assessment

---

## 📊 Progress Tracker

```
Week 7-8 IAM & Security Complete:
Day 26: OIDC Provider ✅
Day 27: IRSA Multiple Services ✅
Day 28: Pod-Level IAM ✅
Day 29: Secrets Manager ✅
Day 30: Advanced IAM Practice ⬜
```

---

## 🔗 Next Week Preview

**Week 9-10:** GitOps with ArgoCD
- Day 31: GitOps Concepts & ArgoCD Introduction
- Day 32: ArgoCD Installation & Configuration
- Day 33: ArgoCD Application Patterns
- Day 34: ArgoCD Sync & Rollback Strategies
- Day 35: ArgoCD Advanced & Best Practices

---

**Remember:** Security is not a feature - it's the foundation of your platform! 🚀
