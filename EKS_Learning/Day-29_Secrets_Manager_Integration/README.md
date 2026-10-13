# 📅 Day 29: AWS Secrets Manager Integration

## 🎯 Learning Objectives
By the end of today, you will understand:
- Storing secrets in AWS Secrets Manager
- Syncing secrets to Kubernetes
- Using the Secrets Store CSI Driver
- Automatic secret rotation
- Best practices for secret management

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **AWS Secrets Manager** | A managed service for storing, controlling access to, and rotating secrets such as database passwords. |
| **Secret rotation** | Replacing a credential periodically and ensuring dependent applications can use the new value. |
| **External Secrets integration** | A controller or integration that copies/references external secret-store values as Kubernetes Secret objects. |
| **CSI secret mount** | A driver-based way to mount secret-store values into a Pod as files, depending on the chosen provider/driver. |
| **Encryption at rest** | Protecting stored data using encryption while it is saved on disk/service storage. |
| **IAM permission** | An AWS policy grant allowing a principal to perform specific API actions on specific resources. |

**Analogy:** Secrets Manager is a managed vault; IAM is the access list, and rotation is changing the lock and coordinating the new key with applications.

**Try / observe:** Verify the application's workload identity and the smallest required read permission. Use a test secret; do not place real secret values in YAML committed to Git.

**Troubleshooting clue:** If the secret cannot be fetched, check identity, IAM resource ARN/region, network access, secret version/stage, and integration logs. If fetched but app still fails, check key names and refresh/restart behavior.

**Explain it back:** What does rotation change? Why should an app's IAM role be limited to a specific secret?

---

## 📖 Core Concepts

### Secret Management Options

```
┌─────────────────────────────────────────────────────────┐
│        SECRET MANAGEMENT OPTIONS COMPARISON              │
│                                                         │
│  ┌─────────────────┐  ┌─────────────────┐              │
│  │  Kubernetes      │  │  AWS Secrets     │              │
│  │  Secrets         │  │  Manager         │              │
│  │  ─────────       │  │  ─────────       │              │
│  │  + Simple        │  │  + Centralized   │              │
│  │  + Built-in      │  │  + Auto rotation │              │
│  │  - Stored in     │  │  + Auditing      │              │
│  │    etcd          │  │  + Versioning    │              │
│  │  - No rotation   │  │  - External      │              │
│  │  - Base64 only   │  │    dependency    │              │
│  └─────────────────┘  └─────────────────┘              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │         BEST: Use Both Together                   │   │
│  │  K8s Secrets + AWS Secrets Manager               │   │
│  │  via Secrets Store CSI Driver                     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Secrets Store CSI Driver Architecture

```
┌─────────────────────────────────────────────────────────┐
│        SECRETS STORE CSI DRIVER ARCHITECTURE              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EKS Cluster                                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Pod                                      │   │   │
│  │  │  ┌─────────────────────────────────┐     │   │   │
│  │  │  │ Secret Volume Mount             │     │   │   │
│  │  │  │ /mnt/secrets/db-password        │     │   │   │
│  │  │  └─────────────────────────────────┘     │   │   │
│  │  └──────────────┬──────────────────────────┘   │   │
│  │                 │ CSI Mount                     │   │
│  │  ┌──────────────▼──────────────────────────┐   │   │
│  │  │  Secrets Store CSI Driver                │   │   │
│  │  │  (pod running on node)                   │   │   │
│  │  └──────────────┬──────────────────────────┘   │   │
│  │                 │ Fetch Secrets                │   │   │
│  │  ┌──────────────▼──────────────────────────┐   │   │
│  │  │  AWS Provider                            │   │   │
│  │  │  (for Secrets & Config)                  │   │   │
│  │  └──────────────┬──────────────────────────┘   │   │
│  └─────────────────┼────────────────────────────────┘   │
│                    │ IRSA (credentials)                 │
│                    ▼                                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AWS Secrets Manager                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  db_password (rotated automatically)     │   │   │
│  │  │  api_key                                │   │   │
│  │  │  tls_certificate                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: Setup Secrets Manager with CSI

### Step 1: Install Secrets Store CSI Driver

```bash
# Add Helm repo
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

# Install CSI driver
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set enableSecretRotation=true \
  --set rotationPollInterval=2m
```

### Step 2: Install AWS Provider

```bash
# Add AWS provider repo
helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws

# Install AWS provider
helm install secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws \
  --namespace kube-system
```

### Step 3: Create Secrets in AWS

```bash
# Create a database password secret
aws secretsmanager create-secret \
  --name db-password \
  --secret-string '{"username":"admin","password":"P@ssw0rd123!"}'

# Create an API key secret
aws secretsmanager create-secret \
  --name api-key \
  --secret-string '{"key":"abc123xyz789"}' \
  --rotation-lambda-arn arn:aws:lambda:us-west-2:123456789012:function:rotate-api-key \
  --rotation-rules '{"AutomaticallyAfterDays":30}'
```

### Step 4: Create IAM Policy & Role for Secret Access

```bash
# Create IAM policy
cat > secrets-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": [
                "arn:aws:secretsmanager:us-west-2:123456789012:secret:db-password-*",
                "arn:aws:secretsmanager:us-west-2:123456789012:secret:api-key-*"
            ]
        }
    ]
}
EOF

# Create policy
aws iam create-policy \
  --policy-name secrets-access-policy \
  --policy-document file://secrets-policy.json

# Create service account with secret access
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=secrets-sa \
  --namespace=default \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/secrets-access-policy \
  --approve
```

### Step 5: Create SecretProviderClass

```yaml
# secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "db-password"
        objectType: "secretsmanager"
      - objectName: "api-key"
        objectType: "secretsmanager"
```

### Step 6: Use Secrets in Pod

```yaml
# app-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  serviceAccountName: secrets-sa
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "app-secrets"
```

```bash
kubectl apply -f secret-provider-class.yaml
kubectl apply -f app-pod.yaml

# Verify secrets are mounted
kubectl exec -it app-with-secrets -- ls /mnt/secrets/
kubectl exec -it app-with-secrets -- cat /mnt/secrets/db-password
kubectl exec -it app-with-secrets -- cat /mnt/secrets/api-key
```

---

## 📝 Advanced: Sync Secrets to Kubernetes Secrets

```yaml
# secret-provider-class-sync.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets-sync
spec:
  provider: aws
  secretObjects:
  - secretName: app-db-secret
    type: Opaque
    data:
    - objectName: "db-password"
      key: "password"
    - objectName: "db-password"
      key: "username"
  parameters:
    objects: |
      - objectName: "db-password"
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: username
```

```yaml
# app-pod-sync.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secret-sync
spec:
  serviceAccountName: secrets-sa
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-db-secret
          key: password
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: app-db-secret
          key: username
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "app-secrets-sync"
```

---

## 📝 Practice: Auto Rotation with Secrets Manager

### Enable Rotation with Lambda

```bash
# Create rotation Lambda
aws lambda create-function \
  --function-name rotate-db-password \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/lambda-rotation-role \
  --handler rotation.lambda_handler \
  --zip-file fileb://rotation.zip

# Enable rotation
aws secretsmanager rotate-secret \
  --secret-id db-password \
  --rotation-lambda-arn arn:aws:lambda:us-west-2:123456789012:function:rotate-db-password \
  --rotation-rules '{"AutomaticallyAfterDays":7}'
```

### Update CSI Driver for Rotation

```bash
# Update driver to poll for changes
helm upgrade csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set enableSecretRotation=true \
  --set rotationPollInterval=2m
```

---

## 🐛 Troubleshooting Secrets Store CSI

```bash
# 1. Check driver pods
kubectl get pods -n kube-system | grep secrets

# 2. Check provider pods
kubectl get pods -n kube-system | grep aws

# 3. View driver logs
kubectl logs $(kubectl get pods -n kube-system -l app=secrets-store-csi-driver -o jsonpath='{.items[0].metadata.name}') -n kube-system

# 4. Check pod events
kubectl describe pod app-with-secrets

# 5. Verify permissions
kubectl exec -it secrets-sa-test -- aws secretsmanager get-secret-value --secret-id db-password
```

---

## 📝 Interview Questions

1. **Why use AWS Secrets Manager over Kubernetes Secrets?**
   - **Answer:** Centralized management, automatic rotation, auditing, versioning, and no storage in etcd.

2. **How does the Secrets Store CSI Driver work?**
   - **Answer:** When a pod mounts a volume with the CSI driver, it fetches secrets from AWS via the AWS provider, using IRSA for credentials.

3. **What are the security benefits?**
   - **Answer:** Secrets not stored in etcd, fine-grained IAM permissions, auto rotation, and central auditing.

4. **How do you rotate secrets automatically?**
   - **Answer:** Set up a Lambda rotation function in AWS Secrets Manager, and enable rotation polling in the CSI driver.

5. **What's the cost of using AWS Secrets Manager?**
   - **Answer:** $0.40 per secret per month, plus $0.05 per 10,000 API calls. Costs are predictable.

---

## 📚 Resources

- [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [AWS Provider for CSI](https://aws.github.io/secrets-store-csi-driver-provider-aws/)

---

## 🎯 Today's Assignment

1. **Install:** Secrets Store CSI Driver and AWS provider
2. **Create:** Secrets in AWS Secrets Manager
3. **Configure:** SecretProviderClass
4. **Test:** Mount secrets in a pod
5. **Set up:** Auto rotation

---

## 📊 Progress Tracker

```
Day 28: Pod-Level IAM Permissions ✅
Day 29: Secrets Manager Integration ⬜
```

---

## 🔗 Next Day Preview

**Day 30:** Advanced IAM Practice
- Comprehensive security lab
- Real-world scenarios
- Security assessment

---

**Remember:** Never store secrets in plain text - use Secrets Manager with rotation! 🚀
