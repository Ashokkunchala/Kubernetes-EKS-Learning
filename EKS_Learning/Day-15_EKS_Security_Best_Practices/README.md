# 📅 Day 15: EKS Security Best Practices

## 🎯 Learning Objectives
By the end of today, you will understand:
- Pod Security Standards
- Network Policies
- Secret encryption
- Image scanning with ECR
- Security best practices checklist

---

## 🧠 Plain-English terms before you start

| Term | Meaning | Why it matters |
|---|---|---|
| **Least privilege** | Grant only the permissions required for a specific task. | It reduces the impact of mistakes or compromised credentials. |
| **RBAC** | Kubernetes rules that bind identities to allowed API actions. | It controls who can read, change, or delete cluster objects. |
| **Pod Security Standards (PSS)** | Kubernetes-defined security levels for Pod configurations: Privileged, Baseline, and Restricted. | They help prevent risky container settings from being admitted. |
| **Image scanning** | Checking a container image for known vulnerabilities and policy violations. | It catches some risks before the image runs; it does not guarantee an image is safe. |
| **Audit log** | A record of API activity, including who made a request and what happened. | It supports investigations and accountability. |
| **Secret management** | Securely storing, delivering, limiting access to, and rotating credentials. | Credentials should not be baked into images or committed to Git. |

**Analogy:** Security is layered like a building: identity checks, room permissions, locked cabinets, and camera logs all serve different purposes.

**Try / observe:** `kubectl auth can-i --list -n <namespace>`, `kubectl get pod <pod-name> -o yaml`, and inspect enabled EKS control-plane audit logging.

**Troubleshooting clue:** A denied request is not always an AWS IAM problem. Identify whether the denial comes from AWS IAM/EKS access, Kubernetes RBAC, admission policy, or a network control.

**Explain it back:** Why are IAM and Kubernetes RBAC separate controls? What can image scanning not prove?

---

## 📖 Core Concepts

### EKS Security Layers

```
┌─────────────────────────────────────────────────────────┐
│              EKS SECURITY LAYERS                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 1: Cluster Security                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • IAM roles and policies               │   │   │
│  │  │  • Cluster endpoint access              │   │   │
│  │  │  • Encryption at rest (KMS)             │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 2: Node Security                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Private subnets                      │   │   │
│  │  │  • Security groups                      │   │   │
│  │  │  • Node IAM role (least privilege)      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 3: Pod Security                          │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Pod Security Standards               │   │   │
│  │  │  • Security contexts                    │   │   │
│  │  │  • Read-only root filesystem            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 4: Container Security                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Image scanning (ECR)                 │   │   │
│  │  │  • Image signing                        │   │   │
│  │  │  • Minimal base images                  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 5: Network Security                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Network Policies                     │   │   │
│  │  │  • Service mesh (Istio/App Mesh)        │   │   │
│  │  │  • WAF integration                      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Pod Security Standards

```
┌─────────────────────────────────────────────────────────┐
│              POD SECURITY STANDARDS                       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Privileged                                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Unrestricted access                  │   │   │
│  │  │  • Can run as root                      │   │   │
│  │  │  • Can access host namespace            │   │   │
│  │  │  • Use: System pods only                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Baseline                                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Prevents known privilege escalations │   │   │
│  │  │  • No host access                      │   │   │
│  │  │  • Limited capabilities                │   │   │
│  │  │  • Use: Most workloads                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Restricted                                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Heavily restricted                   │   │   │
│  │  │  • Non-root required                   │   │   │
│  │  │  • Read-only filesystem                 │   │   │
│  │  │  • Use: Security-sensitive workloads    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Pod Security Context

```yaml
# secure-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## 🔒 Encryption at Rest

```bash
# Enable encryption at rest for secrets
aws eks update-cluster-config \
  --name my-eks-cluster \
  --region us-west-2 \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:us-west-2:ACCOUNT:key/KEY_ID"}}]'

# Create KMS key
aws kms create-key --description "EKS Secrets Encryption"

# Enable envelope encryption
aws eks update-cluster-config \
  --name my-eks-cluster \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:us-west-2:ACCOUNT:key/KEY_ID"},"effect":"Encrypt"}]'
```

---

## 🛠️ Step-by-Step Security Setup

### Step 1: Restrict Cluster Endpoint

```bash
# Restrict cluster endpoint access
aws eks update-cluster-config \
  --name my-eks-cluster \
  --region us-west-2 \
  --resources-vpc-config 'endpointPublicAccess=false,endpointPrivateAccess=true'
```

### Step 2: Create Network Policy

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

### Step 3: Scan Images

```bash
# Enable ECR scanning
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true

# Get scan findings
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=latest
```

---

## 📝 Practice Exercises

### Exercise 1: Apply Pod Security Standards

```yaml
# restricted-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  labels:
    security: restricted
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### Exercise 2: Implement Network Policies

```yaml
# allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from: []
```

---

## 📝 Interview Questions

1. **What are Pod Security Standards?**
   - **Answer:** Three levels (Privileged, Baseline, Restricted) that define security policies for pods. Restricted is the most secure.

2. **How to encrypt secrets in EKS?**
   - **Answer:** Use KMS to create a key, then enable envelope encryption with `aws eks update-cluster-config`.

3. **What is IRSA and why is it important?**
   - **Answer:** IRSA provides pod-level IAM permissions without node-level access. It's more secure than using node IAM roles.

4. **How to implement network policies in EKS?**
   - **Answer:** Install a CNI plugin that supports network policies (Calico), then create NetworkPolicy resources.

5. **What is the difference between Security Context and Pod Security Standards?**
   - **Answer:** Security Context is per-pod settings. Pod Security Standards are cluster-wide policies enforced by admission controllers.

---

## 📚 Resources

- [EKS Security](https://docs.aws.amazon.com/eks/latest/userguide/security.html)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

---

## 🎯 Today's Assignment

1. **Enable:** Encryption at rest for secrets
2. **Create:** Network policies for your applications
3. **Scan:** Container images for vulnerabilities
4. **Document:** Security best practices for your cluster

---

## 📊 Progress Tracker

```
Week 2 Complete:
Day 8: Introduction to Amazon EKS ✅
Day 9: EKS Cluster Setup with eksctl ✅
Day 10: EKS Networking and VPC ✅
Day 11: EKS IAM Roles and Policies ✅
Day 12: EKS Storage with EBS ✅
Day 13: EKS Load Balancing with ALB ✅
Day 14: EKS Monitoring and Logging ✅
Day 15: EKS Security Best Practices ⬜
```

---

## 🔗 Next Week Preview

**Week 3-4:** Advanced Networking
- Day 16: VPC CNI Plugin
- Day 17: Network Policies with Calico
- Day 18: Service Mesh with Istio
- Day 19: Advanced Networking Practice
- Day 20: Networking Interview Preparation

---

**Remember:** Security is not optional in production! Implement these practices from day one. 🚀
