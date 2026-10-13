# 📅 Day 28: Pod-Level IAM Permissions (Advanced IRSA)

## 🎯 Learning Objectives
By the end of today, you will understand:
- Dynamic pod-level permissions
- Scoping IAM roles per pod
- Kubernetes RBAC with IAM
- Security best practices
- Real-world permission patterns

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Workload identity** | The identity an application uses when it calls another system. |
| **EKS Pod Identity** | An EKS mechanism for associating Kubernetes ServiceAccounts with AWS IAM roles for workloads. |
| **IRSA** | An alternative workload-identity approach using the cluster OIDC provider and IAM role trust. |
| **Credential provider chain** | The ordered ways an AWS SDK/CLI tries to discover credentials. |
| **Temporary credentials** | Short-lived credentials that expire and are refreshed through a trusted mechanism. |
| **Least privilege** | Limit the role to only the actions/resources the application needs. |

**Analogy:** Do not give every employee the master key. Issue each application its own time-limited badge with only the doors it needs.

**Try / observe:** Confirm the Pod's ServiceAccount, the selected identity mechanism, and the AWS caller identity from inside a safe test workload. Never print secret keys or session tokens.

**Troubleshooting clue:** Check that the required EKS Pod Identity Agent/add-on and association or IRSA trust configuration exist, then verify role permissions and SDK support.

**Explain it back:** Why is a Pod identity safer than embedding AWS access keys in a container image?

---

## 📖 Core Concepts

### Pod-Level vs Node-Level Permissions

```
┌─────────────────────────────────────────────────────────┐
│        POD-LEVEL vs NODE-LEVEL PERMISSIONS                │
│                                                         │
│  ❌ Node-Level (LESS SECURE):                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ALL pods on node share same permissions         │   │
│  │  ┌─────────────┐                                │   │
│  │  │    Node      │ ← Node IAM Role (wide access) │   │
│  │  │  ┌───────┐  │                                │   │
│  │  │  │ Pod A │  │← S3, DynamoDB, SQS, etc.      │   │
│  │  │  ├───────┤  │                                │   │
│  │  │  │ Pod B │  │← Same access as Pod A          │   │
│  │  │  └───────┘  │   (blast radius is huge)      │   │
│  │  └─────────────┘                                │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ✅ Pod-Level (SECURE - IRSA):                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │    Node                                          │   │
│  │  ┌───────┐      ┌───────┐      ┌───────┐      │   │
│  │  │ Pod A │      │ Pod B │      │ Pod C │      │   │
│  │  │  ↓    │      │  ↓    │      │  ↓    │      │   │
│  │  │ S3    │      │DynamoDB│     │ SQS   │      │   │
│  │  │ only  │      │ only  │      │ only  │      │   │
│  │  └───────┘      └───────┘      └───────┘      │   │
│  │   (minimal)      (minimal)      (minimal)      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### IRSA Token Mechanics

```
┌─────────────────────────────────────────────────────────┐
│              IRSA TOKEN MECHANISM                         │
│                                                         │
│  Pod                                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ServiceAccount                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  annotation: eks.amazonaws.com/role-arn │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  Volume Mount:                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  /var/run/secrets/eks.amazonaws.com/    │   │   │
│  │  │    serviceaccount/token                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  Path: /var/run/secrets/eks.amazonaws.com/       │   │
│  │    serviceaccount/eks-token                     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: Advanced IRSA

### Lab 1: Restrict IAM by Service Account

```json
// trust-policy-restricted.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/EXAMPLE:aud": "sts.amazonaws.com",
          "oidc.eks.us-west-2.amazonaws.com/id/EXAMPLE:sub": [
            "system:serviceaccount:api:api-sa",
            "system:serviceaccount:worker:worker-sa"
          ]
        }
      }
    }
  ]
}
```

```bash
# Create role with restricted access
aws iam create-role \
  --role-name restricted-role \
  --assume-role-policy-document file://trust-policy-restricted.json
```

### Lab 2: Time-Based Permissions

```json
// time-based-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::reports/*",
      "Condition": {
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T00:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-12-31T23:59:59Z"
        }
      }
    }
  ]
}
```

### Lab 3: IP & VPC-Based Permissions

```json
// network-based-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:*",
      "Resource": "arn:aws:dynamodb:us-west-2:123456789012:table/Orders",
      "Condition": {
        "StringEquals": {
          "aws:SourceVpce": "vpce-12345678"
        }
      }
    }
  ]
}
```

---

## 🖥️ Practical Lab: Advanced Security Setup

### Step 1: Create Separate Service Accounts

```yaml
# service-accounts.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/admin-role
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: read-only-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/readonly-role
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: temp-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/temp-role
```

### Step 2: Apply Kubernetes RBAC by Service Account

```yaml
# rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "deployments"]
  verbs: ["get", "list", "watch", "create", "delete", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: read-only-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: admin-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

---

## 📝 Practice Exercise: Test Permission Differences

```bash
# Create namespaces if not exist
kubectl create namespace production

# Create service accounts
kubectl apply -f service-accounts.yaml

# Apply RBAC
kubectl apply -f rbac.yaml

# Test readonly SA (should fail on delete)
kubectl auth can-i delete pod -n production --as=system:serviceaccount:production:read-only-sa
# Result: no

# Test admin SA (should succeed)
kubectl auth can-i delete pod -n production --as=system:serviceaccount:production:admin-sa
# Result: yes

# Test resource access via SA
# Create test pod with readonly SA
kubectl run test-pod --image=busybox --restart=Never --dry-run=client -o yaml > test-pod.yaml

# Edit to add serviceAccountName: read-only-sa
kubectl apply -f test-pod.yaml
```

---

## 🐛 Common IRSA Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| Access Denied | Wrong IAM role | Check role ARN annotation |
| Assume Role Failed | Wrong trust policy | Check OIDC ARN in policy |
| ConditionalCheckFailed | Missing OIDC provider | Re-associate provider |
| NoSuchEntity | Role doesn't exist | Create the role first |
| Token Invalid | Expired token | Check pod token rotation |

---

## 📝 Interview Questions

1. **How do you implement least privilege with IRSA?**
   - **Answer:** Create separate IAM roles per service with only the required permissions. Restrict trust policies by service account.

2. **What's the difference between IRSA and RBAC?**
   - **Answer:** IRSA controls AWS API access (S3, DynamoDB). RBAC controls Kubernetes API access (pods, deployments).

3. **How do you audit pod permissions?**
   - **Answer:** Use `kubectl auth can-i`, check IAM CloudTrail logs, and review IAM roles/policies regularly.

4. **Can a pod have multiple IAM roles?**
   - **Answer:** A pod can only use one IAM role (via one service account). For multiple services, attach multiple policies to one role.

5. **How do you rotate credentials for pods?**
   - **Answer:** IRSA automatically rotates tokens. For temporary credentials, AWS SDK handles refresh automatically.

---

## 📚 Resources

- [IRSA Advanced](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [IAM Conditions](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition.html)

---

## 🎯 Today's Assignment

1. **Create:** Multiple service accounts with different permissions
2. **Apply:** RBAC rules per service account
3. **Test:** Permission differences
4. **Implement:** Least privilege access

---

## 📊 Progress Tracker

```
Day 27: IRSA for Multiple Services ✅
Day 28: Pod-Level IAM Permissions ⬜
```

---

## 🔗 Next Day Preview

**Day 29:** Secrets Manager Integration
- Store secrets in AWS Secrets Manager
- Sync to Kubernetes
- Rotate secrets automatically

---

**Remember:** Combine AWS permissions and Kubernetes RBAC for defense in depth! 🚀
