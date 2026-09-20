# 📅 Day 27: IRSA for Multiple Services

## 🎯 Learning Objectives
By the end of today, you will understand:
- IRSA patterns for multiple AWS services
- Real-world service integration (S3, DynamoDB, SQS, Secrets Manager)
- Creating IAM policies
- Best practices for pod permissions

---

## 📖 Core Concepts

### IRSA for Multiple Services

```
┌─────────────────────────────────────────────────────────┐
│         IRSA FOR MULTIPLE AWS SERVICES                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                 EKS Cluster                       │   │
│  │                                                   │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │   │
│  │  │  API App    │ │  Worker App │ │  Report App │ │   │
│  │  │  ┌───────┐  │ │  ┌───────┐  │ │  ┌───────┐  │ │   │
│  │  │  │api-sa │  │ │  │work-sa│  │ │  │rep-sa │  │ │   │
│  │  │  └───────┘  │ │  └───────┘  │ │  └───────┘  │ │   │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ │   │
│  └─────────┼───────────────┼───────────────┼────────┘   │
│            │               │               │            │
│            ▼               ▼               ▼            │
│  ┌─────────────────────────────────────────────────┐   │
│  │                 IAM ROLES                         │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │   │
│  │  │ api-role    │ │ work-role   │ │ rep-role    │ │   │
│  │  │ Pipeline:S3 │ │ Pipeline:DDB│ │ Pipeline:   │ │   │
│  │  │ SQS,SM      │ │ S3, SQS     │ │ S3, Athena  │ │   │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ │   │
│  └─────────┼───────────────┼───────────────┼────────┘   │
│            │               │               │            │
│            ▼               ▼               ▼            │
│  ┌─────────────────────────────────────────────────┐   │
│  │                 AWS SERVICES                      │   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐      │   │
│  │  │ S3  │ │Dynamo│ │ SQS │ │SM   │ │Athena│      │   │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: Set Up IRSA for Multiple Services

### Step 1: Create Custom IAM Policy for S3

```bash
# Create policy JSON
cat > s3-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-app-bucket/*",
                "arn:aws:s3:::my-app-bucket"
            ]
        }
    ]
}
EOF

# Create policy
aws iam create-policy \
  --policy-name my-s3-policy \
  --policy-document file://s3-policy.json
```

### Step 2: Create IAM Policy for DynamoDB

```bash
# Create policy JSON
cat > dynamodb-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:Query",
                "dynamodb:Scan"
            ],
            "Resource": [
                "arn:aws:dynamodb:us-west-2:123456789012:table/Orders"
            ]
        }
    ]
}
EOF

# Create policy
aws iam create-policy \
  --policy-name my-dynamodb-policy \
  --policy-document file://dynamodb-policy.json
```

### Step 3: Create IAM Policy for SQS

```bash
# Create policy JSON
cat > sqs-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sqs:SendMessage",
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage"
            ],
            "Resource": "arn:aws:sqs:us-west-2:123456789012:order-queue"
        }
    ]
}
EOF

# Create policy
aws iam create-policy \
  --policy-name my-sqs-policy \
  --policy-document file://sqs-policy.json
```

### Step 4: Create Service Accounts with Multiple Policies

```bash
# Create service account for API service
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=api-sa \
  --namespace=api \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/my-s3-policy \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/my-sqs-policy \
  --approve

# Create service account for Worker
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=worker-sa \
  --namespace=worker \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/my-dynamodb-policy \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/my-s3-policy \
  --approve
```

### Step 5: Verify Service Accounts

```bash
# Check service account annotations
kubectl get sa api-sa -n api -o yaml
kubectl get sa worker-sa -n worker -o yaml
```

---

## 🖥️ Practical Lab: Build a Multi-Service Application

### Application Architecture

```
┌─────────────────────────────────────────────────────────┐
│         MULTI-SERVICE E-COMMERCE APPLICATION              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  API Service (api-sa)                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Saves orders to S3                   │   │   │
│  │  │  • Sends messages to SQS                │   │   │
│  │  │  • Reads product images from S3         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Worker Service (worker-sa)                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Processes messages from SQS          │   │   │
│  │  │  • Stores order in DynamoDB             │   │   │
│  │  │  • Updates status in S3                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Step 1: Deploy API Service

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      serviceAccountName: api-sa
      containers:
      - name: api
        image: my-api:latest
        env:
        - name: S3_BUCKET
          value: "my-app-bucket"
        - name: SQS_QUEUE
          value: "order-queue"
        ports:
        - containerPort: 8080
```

### Step 2: Deploy Worker Service

```yaml
# worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-service
  namespace: worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      serviceAccountName: worker-sa
      containers:
      - name: worker
        image: my-worker:latest
        env:
        - name: DYNAMODB_TABLE
          value: "Orders"
        - name: SQS_QUEUE
          value: "order-queue"
```

---

## 📝 Practice Labs: Test AWS Access

### Lab 1: Verify S3 Access

```bash
# Get API pod
API_POD=$(kubectl get pods -n api -o jsonpath='{.items[0].metadata.name}')

# Test S3 access
kubectl exec -it api/$API_POD -- aws s3 ls s3://my-app-bucket/
```

### Lab 2: Verify DynamoDB Access

```bash
# Get worker pod
WORKER_POD=$(kubectl get pods -n worker -o jsonpath='{.items[0].metadata.name}')

# Test DynamoDB
kubectl exec -it worker/$WORKER_POD -- aws dynamodb list-tables
```

### Lab 3: Verify SQS Access

```bash
# Test SQS
kubectl exec -it api/$API_POD -- aws sqs receive-message --queue-url https://sqs.us-west-2.amazonaws.com/123456789012/order-queue
```

### Lab 4: Test Least Privilege

```bash
# API pod should NOT access DynamoDB
kubectl exec -it api/$API_POD -- aws dynamodb list-tables
# Should FAIL (api-sa doesn't have DynamoDB permissions)

# Worker pod should NOT access SQS (only DynamoDB + S3)
kubectl exec -it worker/$WORKER_POD -- aws sqs receive-message --queue-url ... 
# Should FAIL (worker-sa doesn't have full SQS permissions)
```

---

## 🐛 Troubleshooting IRSA

```bash
# 1. Check role assumption
kubectl exec -it <pod> -- aws sts get-caller-identity

# 2. Check service account annotations
kubectl get sa <name> -n <ns> -o yaml

# 3. Verify IAM role trust policy
aws iam get-role --role-name <role>

# 4. Check pod env variables
kubectl exec -it <pod> -- env | grep AWS

# 5. Check IAM policy permissions
aws iam list-attached-role-policies --role-name <role>
```

---

## 📝 Interview Questions

1. **How do you provide different AWS permissions to different services?**
   - **Answer:** Create separate IAM roles with specific policies, and separate ServiceAccounts. Attach to pods via annotations.

2. **What happens if a pod needs multiple AWS services?**
   - **Answer:** Attach multiple policies to the same IAM role, or use IRSA with a custom policy covering all needed services.

3. **How do you implement least privilege?**
   - **Answer:** Give only the specific permissions each service needs. For example, API only needs S3 + SQS, Worker needs DynamoDB + S3.

4. **How to test IRSA permissions?**
   - **Answer:** Exec into the pod and try AWS commands. Verify some work (allowed) and some fail (denied).

5. **What are the security best practices for IRSA?**
   - **Answer:** Use separate roles per service, minimal permissions, scoped resource ARNs, and regular permission audits.

---

## 📚 Resources

- [IRSA Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [AWS IAM Policy Examples](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)

---

## 🎯 Today's Assignment

1. **Create:** Custom IAM policies (S3, DynamoDB, SQS)
2. **Set up:** IRSA for multiple services
3. **Deploy:** API and Worker with different permissions
4. **Test:** Least privilege is enforced

---

## 📊 Progress Tracker

```
Day 26: OIDC Provider Setup ✅
Day 27: IRSA for Multiple Services ⬜
```

---

## 🔗 Next Day Preview

**Day 28:** Pod-Level IAM Permissions
- Advanced IRSA patterns
- Dynamic permissions
- Fine-grained access control

---

**Remember:** Use IRSA to give each service exactly the permissions it needs - no more, no less! 🚀
