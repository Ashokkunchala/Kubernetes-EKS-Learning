# 📅 Day 22: FSx for Lustre and S3 CSI Driver

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is FSx for Lustre
- Use cases for high-performance storage
- Installing FSx CSI Driver
- S3 CSI Driver for object storage
- Practical ML/HPC workloads

---

## 📖 Core Concepts

### What is FSx for Lustre?

**FSx for Lustre** is a high-performance file system optimized for machine learning and high-performance computing (HPC). It provides sub-millisecond latency and up to hundreds of GB/s throughput.

```
┌─────────────────────────────────────────────────────────┐
│          FSx FOR LUSTRE ARCHITECTURE                      │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ML/HPC Workload                                 │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐│   │
│  │  │ Pod (GPU)   │ │ Pod (GPU)   │ │ Pod (CPU)   ││   │
│  │  │  ┌───────┐  │ │  ┌───────┐  │ │  ┌───────┐  ││   │
│  │  │  │/data  │  │ │  │/data  │  │ │  │/data  │  ││   │
│  │  │  └───┬───┘  │ │  └───┬───┘  │ │  └───┬───┘  ││   │
│  │  └──────┼──────┘ └──────┼──────┘ └──────┼──────┘│   │
│  └─────────┼───────────────┼───────────────┼───────┘   │
│            │               │               │            │
│  ┌─────────┴───────────────┴───────────────┴───────┐   │
│  │              FSx for Lustre                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Sub-millisecond latency              │   │   │
│  │  │  • Up to hundreds GB/s throughput        │   │   │
│  │  │  • Massive parallel access              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Linked to S3                            │   │   │
│  │  │  • Reads data from S3 bucket             │   │   │
│  │  │  • Writes results back to S3             │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └────────────────────────────────────────────┴───┘   │
│            │                                          │
│  ┌─────────┴───────────────────────────────────────┐  │
│  │              Amazon S3                           │  │
│  │  s3://my-ml-bucket/training-data/               │  │
│  └─────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Use Cases

```
┌─────────────────────────────────────────────────────────┐
│                 FSx USE CASES                             │
│                                                         │
│  🧠 Machine Learning:                                   │
│  • Training data                      data/...         │
│  • Model checkpoints                                   │
│  • Parallel GPUs need fast data                        │
│                                                         │
│  🔬 High-Performance Computing:                         │
│  • Scientific simulations                              │
│  • Weather forecasting                                 │
│  • Genomic analysis                                    │
│                                                         │
│  🎬 Media Processing:                                   │
│  • Video rendering                                     │
│  • 3D modeling                                         │
│  • Animation                                          │
│                                                         │
│  📊 Big Data Analytics:                                 │
│  • Real-time processing                                │
│  • Data transformation                                 │
│  • Streaming analytics                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🖥️ What is S3 CSI Driver?

**S3 CSI Driver** allows Kubernetes pods to mount S3 buckets as filesystems. It's useful for accessing object storage from applications.

```
┌─────────────────────────────────────────────────────────┐
│              S3 CSI DRIVER ARCHITECTURE                  │
│                                                         │
│  Pod → Mount → S3 Bucket (as filesystem)               │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Common Patterns:                                │   │
│  │  • Static mount (read-only)                     │   │
│  │  • Dynamic mounting                            │   │
│  │  • Cache data                                   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: Install and Use FSx for Lustre

### Step 1: Install FSx CSI Driver

```bash
# Install using eksctl
eksctl create addon \
  --name aws-fsx-csi-driver \
  --cluster my-cluster \
  --force
```

### Step 2: Create FSx for Lustre

```bash
# Create FSx for Lustre (linked to S3)
FSX_ID=$(aws fsx create-file-system \
  --file-system-type LUSTRE \
  --storage-capacity 1200 \
  --subnet-ids <private-subnet-id> \
  --lustre-configuration '{
    "DeploymentType": "PERSISTENT_1",
    "PerUnitStorageThroughput": 200,
    "DataRepositoryConfiguration": {
      "AutoImportPolicy": "NEW_CHANGED",
      "DataRepositoryPath": "s3://my-bucket/training-data"
    }
  }' \
  --query 'FileSystem.FileSystemId' \
  --output text)

echo "FSx ID: $FSX_ID"
```

### Step 3: Create StorageClass and PVC

```yaml
# fsx-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fsx-sc
provisioner: fsx.csi.aws.com
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fsx-pv
spec:
  capacity:
    storage: 1200Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fsx-sc
  csi:
    driver: fsx.csi.aws.com
    volumeHandle: fs-12345678
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fsx-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: fsx-sc
  resources:
    requests:
      storage: 1200Gi
```

```bash
kubectl apply -f fsx-storage.yaml
```

### Step 4: Deploy ML Training Pod

```yaml
# ml-training.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml
  template:
    metadata:
      labels:
        app: ml
    spec:
      containers:
      - name: trainer
        image: tensorflow/tensorflow:latest-gpu
        command: ["python", "-c", "import time; time.sleep(3600)"]
        resources:
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: training-data
          mountPath: /data
        - name: checkpoints
          mountPath: /checkpoints
      volumes:
      - name: training-data
        persistentVolumeClaim:
          claimName: fsx-pvc
      - name: checkpoints
        emptyDir: {}
```

---

## 🛠️ Hands-On: Install and Use S3 CSI Driver

### Step 1: Install S3 CSI Driver

```bash
# Install via Helm
helm repo add aws-s3-csi-driver https://awslabs.github.io/mountpoint-s3-csi-driver
helm repo update

helm install aws-s3-csi-driver aws-s3-csi-driver/aws-s3-csi-driver \
  --namespace kube-system
```

### Step 2: Create StorageClass for S3

```yaml
# s3-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: s3-sc
provisioner: s3.csi.aws.com
mountOptions:
- allow-delete
- region=us-west-2
parameters:
  bucket: my-application-bucket
```

### Step 3: Create PVC and Pod

```yaml
# s3-app.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: s3-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: s3-sc
  resources:
    requests:
      storage: 100Gi  # Logical size only for S3
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: s3-app
  template:
    metadata:
      labels:
        app: s3-app
    spec:
      serviceAccountName: s3-access-sa  # IRSA role
      containers:
      - name: app
        image: nginx:alpine
        volumeMounts:
        - name: s3-data
          mountPath: /data
      volumes:
      - name: s3-data
        persistentVolumeClaim:
          claimName: s3-pvc
```

---

## 📝 Practice Exercises

### Exercise 1: Compare Storage Options

```
┌─────────────────────────────────────────────────────────┐
│              STORAGE COMPARISON TABLE                     │
│                                                         │
│  ┌──────────┬────────────┬────────────┬────────────────┐│
│  │ Feature  │    EBS     │    EFS     │  FSx for        ││
│  │          │            │            │  Lustre         ││
│  ├──────────┼────────────┼────────────┼────────────────┤│
│  │ Access   │ Block      │ File       │ File           ││
│  │ Modes    │ RWO        │ RWX        │ RWX            ││
│  │ AZ       │ Single     │ Multi      │ Single/Multi   ││
│  │ Latency  │ Sub-ms     │ ~ms        │ Sub-ms         ││
│  │ Through  │ up to 1000 │ ~100s MB/s │ 100s GB/s      ││
│  │ Use      │ Databases  │ Shared     │ ML/HPC         ││
│  │ Case     │            │ Files      │                ││
│  └──────────┴────────────┴────────────┴────────────────┘│
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Exercise 2: Real-World Scenario

**Scenario:** You're building a video processing pipeline
- 100 videos need transcoding
- Multiple workers process videos in parallel
- Need shared temporary storage
- Final results go to S3

**Solution:**
1. EBS for temporary processing storage (fast, single worker)
2. EFS for shared results (multi-worker accessible)
3. S3 for final archive (durable, cheap)

---

## 🐛 Troubleshooting

```bash
# FSx CSI Driver issues
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-fsx-csi-driver
kubectl logs -n kube-system <fsx-csi-pod>

# S3 CSI Driver issues
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-s3-csi-driver
kubectl logs -n kube-system <s3-csi-pod>

# Check PVC status
kubectl describe pvc <pvc-name>

# Check IRSA permissions for S3
kubectl get sa <service-account> -o yaml
```

---

## 📝 Interview Questions

1. **What is FSx for Lustre and when to use it?**
   - **Answer:** High-performance file system for ML/HPC workloads. Use when you need very high throughput and low latency for parallel processing.

2. **How does FSx differ from EFS?**
   - **Answer:** FSx for Lustre is much faster (hundreds GB/s) with sub-millisecond latency but is less flexible. EFS is more general-purpose.

3. **What storage options does EKS support?**
   - **Answer:** EBS (block), EFS (shared files), FSx for Lustre (high performance), S3 (object storage), plus local storage.

4. **When would you use S3 CSI Driver?**
   - **Answer:** When applications need to access S3 objects as a filesystem, such as for serving static content or processing data in buckets.

5. **How do you choose the right storage?**
   - **Answer:** Consider: access mode (RWO/RWX), performance needs, cost, latency requirements, and data durability needs.

---

## 📚 Resources

- [FSx for Lustre CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/fsx-csi.html)
- [S3 CSI Driver](https://github.com/awslabs/mountpoint-s3-csi-driver)

---

## 🎯 Today's Assignment

1. **Install:** FSx and S3 CSI Drivers
2. **Create:** Storage classes for each
3. **Deploy:** Apps using FSx and S3
4. **Test:** Performance and functionality

---

## 📊 Progress Tracker

```
Day 21: EFS CSI Driver ✅
Day 22: FSx for Lustre & S3 CSI ⬜
```

---

## 🔗 Next Day Preview

**Day 23:** Volume Snapshots and Restore
- Create volume snapshots
- Restore from snapshots
- Backup strategies
- Practical disaster recovery

---

**Remember:** Choose the right storage for your workload - each has specific use cases! 🚀
