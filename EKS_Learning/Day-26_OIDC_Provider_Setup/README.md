# 📅 Day 26: OIDC Provider Setup

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is OIDC and why EKS uses it
- Setting up OIDC provider
- How OIDC enables IRSA
- Security implications
- Hands-on configuration

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **OIDC** | OpenID Connect, a standard for conveying identity using tokens on top of OAuth 2.0. |
| **Identity provider** | A system that issues or vouches for identity information. |
| **JWT** | JSON Web Token: a compact token containing claims that can be checked by a recipient. |
| **Issuer** | The identity provider that issued a token; its identity must match the configured trust. |
| **Audience (`aud`)** | The intended recipient of a token; it helps prevent using a token in the wrong place. |
| **Trust policy** | The IAM role rule that states which principal may assume the role and under what conditions. |

**Analogy:** OIDC is a trusted ID office, the JWT is the signed ID card, the issuer identifies the office, and audience says which desk should accept it.

**EKS connection:** IRSA uses the cluster's OIDC identity provider so a Pod's Kubernetes ServiceAccount can be mapped to an AWS IAM role under explicit trust conditions.

**Try / observe:** Inspect the cluster's OIDC issuer URL and compare the IAM trust policy's issuer, audience, and ServiceAccount subject carefully. Never paste live tokens into chat or logs.

**Troubleshooting clue:** A role-assumption failure often comes from a mismatched issuer URL, audience, namespace/ServiceAccount subject, or missing trust relationship.

**Explain it back:** Why is OIDC setup not the same thing as granting S3 permissions? What do issuer and audience claims protect?

---

## 📖 Core Concepts

### What is OIDC?

**OpenID Connect (OIDC)** is an authentication protocol built on OAuth 2.0. EKS uses it to allow pods to assume IAM roles (IRSA).

```
┌─────────────────────────────────────────────────────────┐
│              OIDC ARCHITECTURE                            │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EKS Cluster                                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Pod → ServiceAccount → JWT Token      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │               │                                 │   │
│  │               ▼                                 │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  EKS OIDC Provider URL                   │   │   │
│  │  │  oidc.eks.us-west-2.amazonaws.com/id/ABC │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AWS IAM                                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  IAM Role Trust Policy                   │   │   │
│  │  │  "Federated": OIDC Provider              │   │   │
│  │  │  "Condition": subject =                  │   │   │
│  │  │    system:serviceaccount:<ns>:<sa>      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### OIDC Workflow

```
┌─────────────────────────────────────────────────────────┐
│              OIDC WORKFLOW                                │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. Pod Started                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • ServiceAccount injected               │   │   │
│  │  │  • JWT token mounted                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  2. Call AWS Service                             │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • App calls S3 API                      │   │   │
│  │  │  • Uses JWT token for auth               │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  3. Token Exchange                              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • AWS verifies token with OIDC         │   │   │
│  │  │  • Checks if pod is authorized          │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  4. Temporary Credentials                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • STS returns temporary creds          │   │   │
│  │  │  • App uses creds for S3 access        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: OIDC Setup

### Step 1: Check Current OIDC Status

```bash
# Check if OIDC is already configured
eksctl get cluster --name my-cluster --region us-west-2

# Check OIDC issuer URL
aws eks describe-cluster \
  --name my-cluster \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

### Step 2: Associate OIDC Provider

```bash
# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region us-west-2 \
  --approve
```

### Step 3: Verify OIDC Provider

```bash
# Get OIDC provider in IAM
aws iam list-open-id-connect-providers

# Get the complete URL
OIDC_PROVIDER=$(aws eks describe-cluster \
  --name my-cluster \
  --query 'cluster.identity.oidc.issuer' \
  --output text | sed -e "s/^https:\/\///")
echo "OIDC Provider: $OIDC_PROVIDER"
```

### Step 4: Create IAM Role for a Service Account

```bash
# Create trust policy JSON
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/$OIDC_PROVIDER"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$OIDC_PROVIDER:aud": "sts.amazonaws.com",
          "$OIDC_PROVIDER:sub": "system:serviceaccount:default:my-app-sa"
        }
      }
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
  --role-name my-app-role \
  --assume-role-policy-document file://trust-policy.json

# Attach permissions
aws iam attach-role-policy \
  --role-name my-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Step 5: Create ServiceAccount

```yaml
# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```

```bash
kubectl apply -f service-account.yaml

# Verify ServiceAccount
kubectl get sa my-app-sa -o yaml
```

### Step 6: Deploy Pod Using ServiceAccount

```yaml
# pod-with-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: amazon/aws-cli
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-with-sa.yaml

# Test AWS access
kubectl exec -it my-app-pod -- aws sts get-caller-identity
# Should show the IAM role ARN
```

---

## 📝 Practical Lab: Multi-Service IRSA

### Set up different service accounts for different applications

```bash
# Create OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# Create service accounts with different permissions
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=api-sa \
  --namespace=api \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=worker-sa \
  --namespace=worker \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess \
  --approve

eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=monitoring-sa \
  --namespace=monitoring \
  --attach-policy-arn=arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess \
  --approve
```

---

## 🔍 Understanding Trust Policies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

**Key Components:**
- **Federated:** The OIDC provider ARN
- **Action:** `sts:AssumeRoleWithWebIdentity`
- **Condition - aud:** `sts.amazonaws.com` (validates audience)
- **Condition - sub:** Can restrict to specific service accounts

---

## 🐛 Troubleshooting OIDC

```bash
# Common issues and fixes

# 1. OIDC provider not found
# Re-associate the provider
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# 2. Pod can't assume role
# Verify ServiceAccount annotation
kubectl get sa <sa-name> -o yaml

# 3. Access denied error
# Check IAM role trust policy
aws iam get-role --role-name <role-name>

# 4. Check STS tokens
kubectl exec -it <pod> -- cat .aws/config
kubectl exec -it <pod> -- env | grep AWS
```

---

## 📝 Interview Questions

1. **What is OIDC and why does EKS need it?**
   - **Answer:** OIDC (OpenID Connect) allows Kubernetes service accounts to be trusted by AWS IAM. It enables IRSA (IAM Roles for Service Accounts).

2. **How does a pod assume an IAM role?**
   - **Answer:** The pod's service account gets a JWT token. AWS validates this token with the OIDC provider and gives temporary credentials via STS.

3. **What security benefits does OIDC provide?**
   - **Answer:** Fine-grained permissions, no long-lived credentials, automatic token rotation, reduced node-level access.

4. **How do you restrict a pod to only assume a specific role?**
   - **Answer:** Use conditions in the IAM trust policy (sub condition) to specify the namespace and service account.

5. **Why is IRSA better than node-level IAM permissions?**
   - **Answer:** IRSA gives per-pod permissions (least privilege). Node-level grants all pods on that node the same access.

---

## 📚 Resources

- [EKS OIDC](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [IAM OIDC Providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)

---

## 🎯 Today's Assignment

1. **Associate:** OIDC provider on your cluster
2. **Create:** An IAM role for a service account
3. **Test:** Pod can access AWS resources
4. **Verify:** IAM role with correct trust policy

---

## 📊 Progress Tracker

```
Day 25: Storage Interview Prep ✅
Day 26: OIDC Provider Setup ⬜
```

---

## 🔗 Next Day Preview

**Day 27:** IRSA for Multiple Services
- Multiple IRSA patterns
- Real-world service integration
- Advanced permission management

---

**Remember:** OIDC is foundational to secure pod-to-AWS permissions! 🚀
