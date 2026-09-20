# 📅 Day 21: EFS CSI Driver

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is Amazon EFS
- EFS vs EBS comparison
- Installing EFS CSI Driver
- Creating EFS storage
- Hands-on with shared storage

---

## 📖 Core Concepts

### What is Amazon EFS?

**Amazon EFS (Elastic File System)** is a serverless, fully elastic file system that scales automatically. It supports ReadWriteMany access, making it ideal for shared storage.

```
┌─────────────────────────────────────────────────────────┐
│              EFS CSI DRIVER ARCHITECTURE                  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Worker Nodes                                    │   │
│  │  ┌─────────────────────┐  ┌─────────────────────┐│   │
│  │  │  Node 1              │  │  Node 2              ││   │
│  │  │  ┌───────────────┐  │  │  ┌───────────────┐  ││   │
│  │  │  │  Pod A        │  │  │  │  Pod B        │  ││   │
│  │  │  │  ┌─────────┐  │  │  │  │  ┌─────────┐  │  ││   │
│  │  │  │  │ /data   │  │  │  │  │  │ /data   │  │  ││   │
│  │  │  │  └────┬────┘  │  │  │  │  └────┬────┘  │  ││   │
│  │  │  └───────┼───────┘  │  │  └───────┼───────┘  ││   │
│  │  │          │           │  │          │           ││   │
│  │  │  ┌───────┴───────┐  │  │  ┌───────┴───────┐  ││   │
│  │  │  │  efs-csi-node │  │  │  │  efs-csi-node │  ││   │
│  │  │  └───────┬───────┘  │  │  └───────┬───────┘  ││   │
│  │  └──────────┼──────────┘  └──────────┼──────────┘│   │
│  └─────────────┼────────────────────────┼────────────┘   │
│                │                        │                │
│                └──────────┬─────────────┘                │
│                           ▼                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Amazon EFS                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Multi-AZ (highly available)          │   │   │
│  │  │  • Automatically scales                │   │   │
│  │  │  • Supports ReadWriteMany              │   │   │
│  │  │  • Serverless (no capacity planning)   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Mount Targets                         │   │   │
│  │  │  • One per AZ                           │   │   │
│  │  │  • Accessible from cluster              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### EFS vs EBS Comparison

```
┌─────────────────────────────────────────────────────────┐
│                  EFS vs EBS                               │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EBS (Block Storage)                             │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  ✅ Fast (SSD throughput)               │   │   │
│  │  │  ✅ Great for databases                 │   │   │
│  │  │  ❌ Single AZ                           │   │   │
│  │  │  ❌ Single node (ReadWriteOnce)         │   │   │
│  │  │  ❌ Manual capacity planning            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EFS (File Storage)                               │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  ✅ Multi-AZ                             │   │   │
│  │  │  ✅ Multi-node (ReadWriteMany)            │   │   │
│  │  │  ✅ Auto-scaling                         │   │   │
│  │  │  ✅ Serverless                          │   │   │
│  │  │  ❌ Slower than EBS                      │   │   │
│  │  │  ❌ Cost per GB stored + transfer       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Use Cases:                                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EBS: Databases, stateful apps                  │   │
│  │  EFS: Shared files, content management           │   │
│  │  Combine: Database on EBS, shared files on EFS   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: Install and Use EFS

### Step 1: Create EFS File System

```bash
# Get VPC ID
VPC_ID=$(aws eks describe-cluster --name my-cluster --query 'cluster.resourcesVpcConfig.vpcId' --output text)
echo $VPC_ID

# Create security group for EFS
SG_ID=$(aws ec2 create-security-group \
  --group-name efs-sg \
  --description "EFS security group" \
  --vpc-id $VPC_ID \
  --output text)
echo $SG_ID

# Allow NFS from VPC
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 2049 \
  --cidr 0.0.0.0/0

# Create EFS
FS_ID=$(aws efs create-file-system \
  --encrypted \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --output text)
echo $FS_ID

# Create mount targets (one per AZ)
SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:kubernetes.io/cluster/my-cluster,Values=shared" \
  --query 'Subnets[*].SubnetId' --output text)

for subnet in $SUBNETS; do
  aws efs create-mount-target \
    --file-system-id $FS_ID \
    --subnet-id $subnet \
    --security-group $SG_ID
done

echo "EFS File System: $FS_ID"
```

### Step 2: Install EFS CSI Driver

```bash
# Install EFS CSI Driver
eksctl create addon \
  --name aws-efs-csi-driver \
  --cluster my-cluster \
  --force
```

### Step 3: Create StorageClass and PVC

```yaml
# efs-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-12345678  # Your EFS File System ID
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f efs-storage.yaml

# Check PVC status
kubectl get pvc efs-pvc
```

### Step 4: Deploy Multi-Pod Application

```yaml
# multi-pod-efs.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-storage-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shared
  template:
    metadata:
      labels:
        app: shared
    spec:
      containers:
      - name: app
        image: nginx:alpine
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: efs-pvc
```

```bash
kubectl apply -f multi-pod-efs.yaml

# Get all pods
kubectl get pods -l app=shared -o wide
```

### Step 5: Test Shared Storage

```bash
# Write from first pod
kubectl exec -it $(kubectl get pods -l app=shared -o jsonpath='{.items[0].metadata.name}') -- sh -c "echo 'Shared Data' > /usr/share/nginx/html/index.html"

# Read from second pod
kubectl exec -it $(kubectl get pods -l app=shared -o jsonpath='{.items[1].metadata.name}') -- cat /usr/share/nginx/html/index.html

# Should show "Shared Data" - this proves shared storage works!
```

---

## 🖥️ Real-World Use Case: WordPress with EFS

```yaml
# wordpress-efs.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          value: "mysql-service"
        - name: WORDPRESS_DB_PASSWORD
          value: "password123"
        volumeMounts:
        - name: wordpress-data
          mountPath: /var/www/html
      volumes:
      - name: wordpress-data
        persistentVolumeClaim:
          claimName: wordpress-pvc
```

---

## 🐛 Troubleshooting EFS

```bash
# Check EFS CSI pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-efs-csi-driver

# Check PVC status
kubectl describe pvc efs-pvc

# Check mount target
kubectl describe pv efs-pv

# Test file system in AWS
aws efs describe-mount-targets --file-system-id $FS_ID

# Check Security Group rules
aws ec2 describe-security-groups --group-ids $SG_ID
```

---

## 📝 Interview Questions

1. **When to use EFS vs EBS?**
   - **Answer:** EFS for shared storage across nodes (ReadWriteMany). EBS for single-node storage (ReadWriteOnce) like databases.

2. **What is the EFS CSI Driver?**
   - **Answer:** A CSI driver that allows Kubernetes to manage EFS file systems, enabling dynamic PV creation.

3. **How does EFS handle high availability?**
   - **Answer:** EFS is multi-AZ by default. Mount targets are created in each AZ, providing redundancy.

4. **What are the EFS performance modes?**
   - **Answer:** General Purpose (low latency) and Max I/O (high throughput). Also Auto and Bursting throughput modes.

5. **Can multiple pods use the same EFS volume?**
   - **Answer:** Yes, EFS supports ReadWriteMany access mode, allowing multiple pods across nodes to share the same file system.

---

## 📚 Resources

- [EFS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)
- [Amazon EFS](https://aws.amazon.com/efs/)
- [EFS on EKS](https://docs.aws.amazon.com/eks/latest/userguide/efs-on-eks.html)

---

## 🎯 Today's Assignment

1. **Create:** An EFS file system
2. **Install:** EFS CSI Driver
3. **Deploy:** Multiple pods sharing storage
4. **Test:** Verify shared data access

---

## 📊 Progress Tracker

```
Day 20: Networking Interview Preparation ✅
Day 21: EFS CSI Driver ⬜
```

---

## 🔗 Next Day Preview

**Day 22:** FSx for Lustre
- High-performance storage
- Machine learning workloads
- HPC (High-Performance Computing)

---

**Remember:** EFS is perfect for shared, scalable storage across multiple pods! 🚀
