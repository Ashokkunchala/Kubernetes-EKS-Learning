# 📅 Day 12: EKS Storage with EBS

## 🎯 Learning Objectives
By the end of today, you will understand:
- Amazon EBS CSI Driver
- StorageClass configuration
- Dynamic provisioning with EBS
- Volume types for EKS
- Hands-on practice

---

## 🧠 Plain-English terms before you start

| Term | Meaning | Why it matters |
|---|---|---|
| **Amazon EBS** | Block storage volumes used like disks, commonly attached to EC2 instances. | It is a common fit for a workload needing durable, single-node-attached block storage. |
| **CSI driver** | A plugin that lets Kubernetes communicate with a storage system through the Container Storage Interface. | Kubernetes needs a driver to provision, attach, and mount AWS storage. |
| **PV / PVC** | A PV represents storage; a PVC is a workload's request for storage. | Pods consume storage through claims rather than manually selecting AWS volume IDs. |
| **StorageClass** | A named storage profile describing how volumes should be provisioned. | It can specify parameters such as volume type and provisioning behavior. |
| **Availability Zone (AZ)** | An isolated location within an AWS Region. | EBS volumes are tied to an AZ, so a Pod may not be schedulable in a different AZ from its volume. |
| **Dynamic provisioning** | Creating a storage volume automatically when a claim requests it. | It avoids manually creating and matching every volume. |

**Analogy:** The PVC is a disk request, the CSI driver is the translator to AWS, and EBS is the disk AWS provides.

**Try / observe:** `kubectl get pvc,pv,storageclass`, `kubectl describe pvc <claim-name>`, `kubectl get pods -o wide`.

**Troubleshooting clue:** Check PVC Events and the EBS CSI controller/node Pods when claims or mounts fail. Also inspect AZ placement and IAM permissions for the driver.

**Explain it back:** Why does EBS need a CSI driver in Kubernetes? Why can AZ placement matter for a Pod using EBS?

---

## 📖 Core Concepts

### EBS CSI Driver Architecture

```
┌─────────────────────────────────────────────────────────┐
│              EBS CSI DRIVER ARCHITECTURE                  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Pod                                              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  volumeMounts:                          │   │   │
│  │  │    mountPath: /data                     │   │   │
│  │  │    name: ebs-volume                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  PersistentVolumeClaim (PVC)                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  storageClassName: ebs-sc               │   │   │
│  │  │  resources: storage: 10Gi               │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  StorageClass (ebs-sc)                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  provisioner: ebs.csi.aws.com           │   │   │
│  │  │  volumeBindingMode: WaitForFirstConsumer│   │   │
│  │  │  parameters:                            │   │   │
│  │  │    type: gp3                            │   │   │
│  │  │    encrypted: "true"                    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EBS CSI Driver                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Runs as DaemonSet on nodes           │   │   │
│  │  │  • Manages EBS volumes                  │   │   │
│  │  │  • Handles create/delete/attach/detach  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Amazon EBS Volume                               │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • gp3 (General Purpose SSD)            │   │   │
│  │  │  • io2 (High Performance SSD)           │   │   │
│  │  │  • st1 (Throughput Optimized HDD)       │   │   │
│  │  │  • sc1 (Cold HDD)                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### EBS Volume Types

```
┌─────────────────────────────────────────────────────────┐
│                EBS VOLUME TYPES                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  General Purpose SSD (gp3)                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Best for most workloads              │   │   │
│  │  │  • Up to 16,000 IOPS                   │   │   │
│  │  │  • Up to 1,000 MB/s throughput          │   │   │
│  │  │  • Cost-effective                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  High Performance SSD (io2)                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Mission-critical applications        │   │   │
│  │  │  • Up to 64,000 IOPS                   │   │   │
│  │  │  • Up to 1,000 MB/s throughput          │   │   │
│  │  │  • Sub-millisecond latency              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Throughput Optimized HDD (st1)                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Frequently accessed data             │   │   │
│  │  │  • Up to 500 IOPS                      │   │   │
│  │  │  • Up to 500 MB/s throughput            │   │   │
│  │  │  • Low cost                             │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Cold HDD (sc1)                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Infrequently accessed data           │   │   │
│  │  │  • Up to 250 IOPS                      │   │   │
│  │  │  • Up to 250 MB/s throughput            │   │   │
│  │  │  • Lowest cost                          │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step-by-Step Setup

### Step 1: Install EBS CSI Driver

```bash
# Install EBS CSI Driver addon
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster my-eks-cluster \
  --region us-west-2 \
  --force
```

### Step 2: Create IAM Role for EBS CSI

```bash
# Create IAM role
eksctl create iamserviceaccount \
  --cluster=my-eks-cluster \
  --name=ebs-csi-controller-sa \
  --namespace=kube-system \
  --attach-policy-arn=arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```

### Step 3: Create StorageClass

```yaml
# ebs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
```

### Step 4: Create PVC

```yaml
# ebs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 20Gi
```

### Step 5: Deploy Pod with EBS Volume

```yaml
# pod-with-ebs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: ebs-storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: ebs-storage
    persistentVolumeClaim:
      claimName: ebs-pvc
```

### Step 6: Test

```bash
# Apply manifests
kubectl apply -f ebs-storageclass.yaml
kubectl apply -f ebs-pvc.yaml
kubectl apply -f pod-with-ebs.yaml

# Check PVC status
kubectl get pvc ebs-pvc

# Check pod
kubectl get pod ebs-pod

# Write data
kubectl exec -it ebs-pod -- sh -c "echo 'Hello EBS' > /usr/share/nginx/html/index.html"

# Verify data
kubectl exec -it ebs-pod -- cat /usr/share/nginx/html/index.html
```

---

## 📝 Practice Exercises

### Exercise 1: Create EBS Storage for MySQL

```yaml
# mysql-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-storage
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: mysql-storage
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password123"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-data
```

---

## 📝 Interview Questions

1. **What is the EBS CSI Driver?**
   - **Answer:** The EBS CSI Driver is a Container Storage Interface driver that manages Amazon EBS volumes in Kubernetes. It handles volume creation, attachment, and mounting.

2. **When to use EBS vs EFS?**
   - **Answer:** EBS for single-AZ, single-node workloads (databases). EFS for multi-AZ, multi-node workloads (shared files).

3. **What is volumeBindingMode?**
   - **Answer:** WaitForFirstConsumer delays volume binding until a pod is scheduled. Immediate binding happens at PVC creation.

4. **How to enable encryption for EBS volumes?**
   - **Answer:** Set `encrypted: "true"` in StorageClass parameters. Use KMS keys for custom encryption.

5. **What is the reclaim policy?**
   - **Answer:** Retain keeps the volume after PVC deletion. Delete removes the volume. Use Retain for important data.

---

## 📚 Resources

- [EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

---

## 🎯 Today's Assignment

1. **Install:** EBS CSI Driver on your cluster
2. **Create:** A StorageClass with encryption
3. **Deploy:** A pod with EBS volume
4. **Test:** Data persistence

---

## 📊 Progress Tracker

```
Day 11: EKS IAM Roles and Policies ✅
Day 12: EKS Storage with EBS ⬜
```

---

## 🔗 Next Day Preview

**Day 13:** EKS Load Balancing with ALB
- AWS Load Balancer Controller
- ALB Ingress
- NLB Service
- Hands-on with load balancing

---

**Remember:** EBS is ideal for stateful workloads on EKS! 🚀
