# 📅 Day 5: Persistent Storage

## 🎯 Learning Objectives
By the end of today, you will understand:
- PersistentVolumes (PV) and PersistentVolumeClaims (PVC)
- StorageClasses and dynamic provisioning
- Volume types in Kubernetes
- Hands-on practice with persistent storage

---

## 📖 Core Concepts

### The Problem: Ephemeral Storage

```
┌─────────────────────────────────────────────────────────┐
│                EPHEMERAL STORAGE PROBLEM                 │
│                                                         │
│  Without Persistent Storage:                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Pod → Container → Data                         │   │
│  │                                                   │   │
│  │  ❌ Data lost when pod is terminated             │   │
│  │  ❌ Data lost when pod is rescheduled           │   │
│  │  ❌ Data lost when node fails                    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Example:                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. Deploy database pod                         │   │
│  │  2. Store data in pod                           │   │
│  │  3. Pod crashes and is rescheduled              │   │
│  │  4. All data is lost!                           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Solution: Persistent Storage (PV, PVC, StorageClass)  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Persistent Storage Components

```
┌─────────────────────────────────────────────────────────┐
│              PERSISTENT STORAGE COMPONENTS               │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  PersistentVolume (PV)                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Cluster-level storage resource        │   │   │
│  │  │  • Created by admin or dynamically       │   │   │
│  │  │  • Has lifecycle independent of pods     │   │   │
│  │  │  • Actual storage (EBS, EFS, NFS)        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  PersistentVolumeClaim (PVC)                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Request for storage by user           │   │   │
│  │  │  • Specifies: size, access modes,        │   │   │
│  │  │    storage class                         │   │   │
│  │  │  • Bound to a PV                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  StorageClass                                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Defines "classes" of storage          │   │   │
│  │  │  • Dynamic provisioning                  │   │   │
│  │  │  • Different types: SSD, HDD, etc.       │   │   │
│  │  │  • Cloud provider specific               │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### PV-PVC Relationship

```
┌─────────────────────────────────────────────────────────┐
│                PV-PVC RELATIONSHIP                       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Pod                                              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  volumeMounts:                          │   │   │
│  │  │    mountPath: /data                     │   │   │
│  │  │    name: my-storage                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  PersistentVolumeClaim (PVC)                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Request: 10Gi storage                  │   │   │
│  │  │  Access Mode: ReadWriteOnce             │   │   │
│  │  │  StorageClass: gp2                      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  PersistentVolume (PV)                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Capacity: 10Gi                         │   │   │
│  │  │  Access Mode: ReadWriteOnce             │   │   │
│  │  │  StorageClass: gp2                      │   │   │
│  │  │  Status: Bound                          │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Actual Storage                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  AWS EBS Volume (gp2)                   │   │   │
│  │  │  Volume ID: vol-0123456789abcdef0        │   │   │
│  │  │  Zone: us-west-2a                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📊 Access Modes

| Access Mode | Abbreviation | Description | Use Case |
|-------------|--------------|-------------|----------|
| ReadWriteOnce | RWO | Read/write by single node | Databases |
| ReadOnlyMany | ROX | Read-only by many nodes | Static content |
| ReadWriteMany | RWX | Read/write by many nodes | Shared files |
| ReadWriteOncePod | RWOP | Read/write by single pod | Exclusive access |

---

## 📊 Volume Types

```
┌─────────────────────────────────────────────────────────┐
│                   KUBERNETES VOLUME TYPES                │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EmptyDir                                        │   │
│  │  • Empty directory for scratch space            │   │
│  │  • Deleted when pod is removed                  │   │
│  │  • Use: Cache, temp files                       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  HostPath                                        │   │
│  │  • Mounts file from host node                   │   │
│  │  • Node-specific                                 │   │
│  │  • Use: Development, system access              │   │
│  │  ⚠️ Security risk                               │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  PersistentVolumeClaim                          │   │
│  │  • Requests storage from PV                     │   │
│  │  • Cloud-agnostic                                │   │
│  │  • Use: Production storage                      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ConfigMap / Secret                             │   │
│  │  • Mounts config data as files                  │   │
│  │  • Read-only                                     │   │
│  │  • Use: Configuration files                     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AWS EBS                                         │   │
│  │  • AWS Elastic Block Store                       │   │
│  │  • Single-AZ, single-node                       │   │
│  │  • Use: Databases, stateful apps                │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AWS EFS                                         │   │
│  │  • AWS Elastic File System                       │   │
│  │  • Multi-AZ, multi-node                         │   │
│  │  • Use: Shared storage, content                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ YAML Examples

### PersistentVolume (Manual)

```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /mnt/data
```

### PersistentVolumeClaim

```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
```

### Pod Using PVC

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### StorageClass (Dynamic Provisioning)

```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
```

### PVC with StorageClass

```yaml
# pvc-with-storageclass.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 20Gi
```

---

## 🔄 Dynamic Provisioning Flow

```
┌─────────────────────────────────────────────────────────┐
│              DYNAMIC PROVISIONING FLOW                   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. User creates PVC                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kind: PersistentVolumeClaim            │   │   │
│  │  │  storageClassName: fast                 │   │   │
│  │  │  resources: storage: 10Gi              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  2. Kubernetes checks StorageClass               │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  StorageClass: fast                     │   │   │
│  │  │  Provisioner: kubernetes.io/aws-ebs     │   │   │
│  │  │  Parameters: type=gp2                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  3. Provisioner creates PV                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  AWS EBS Volume created                 │   │   │
│  │  │  Volume ID: vol-0123456789abcdef0       │   │   │
│  │  │  Size: 10Gi                             │   │   │
│  │  │  Type: gp2                              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  4. PVC bound to PV                             │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  PVC Status: Bound                       │   │   │
│  │  │  PV Status: Bound                       │   │   │
│  │  │  Ready for use!                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Essential kubectl Commands

```bash
# PersistentVolumes
kubectl get persistentvolumes
kubectl get pv  # Short form
kubectl describe pv <pv-name>

# PersistentVolumeClaims
kubectl get persistentvolumeclaims
kubectl get pvc  # Short form
kubectl describe pvc <pvc-name>

# StorageClasses
kubectl get storageclasses
kubectl get sc  # Short form
kubectl describe sc <sc-name>

# Delete PVC
kubectl delete pvc <pvc-name>

# Delete PV
kubectl delete pv <pv-name>
```

---

## 📝 Practice Exercises

### Exercise 1: Manual PV and PVC

```bash
# Create PV
kubectl apply -f persistent-volume.yaml

# Create PVC
kubectl apply -f persistent-volume-claim.yaml

# Verify
kubectl get pv
kubectl get pvc

# Create pod using PVC
kubectl apply -f pod-with-pvc.yaml

# Test storage
kubectl exec -it my-pod -- sh -c "echo 'Hello World' > /usr/share/nginx/html/index.html"
kubectl exec -it my-pod -- curl localhost
```

### Exercise 2: Dynamic Provisioning

```bash
# Create StorageClass
kubectl apply -f storage-class.yaml

# Create PVC (PV will be created automatically)
kubectl apply -f pvc-with-storageclass.yaml

# Verify
kubectl get sc
kubectl get pvc
kubectl get pv

# Create pod using PVC
kubectl apply -f pod-with-pvc.yaml
```

---

## 🐛 Troubleshooting Guide

### PVC Stuck in Pending

```bash
# Check PVC status
kubectl describe pvc <pvc-name>

# Common causes:
# 1. No matching PV
# 2. StorageClass not available
# 3. Insufficient resources
# 4. Cloud provider issues

# Check events
kubectl get events --field-selector involvedObject.name=<pvc-name>
```

### Pod Stuck in ContainerCreating

```bash
# Check pod events
kubectl describe pod <pod-name>

# Check PVC status
kubectl get pvc

# Check node disk space
kubectl describe nodes
```

---

## 📝 Interview Questions

1. **What is the difference between PV and PVC?**
   - **Answer:** PV is the actual storage resource. PVC is a request for storage. PV is cluster-level, PVC is namespace-level.

2. **What is StorageClass and when to use it?**
   - **Answer:** StorageClass defines types of storage and enables dynamic provisioning. Use it to automatically create PVs when PVCs are created.

3. **What access modes are available for PV?**
   - **Answer:** ReadWriteOnce (single node), ReadOnlyMany (multiple nodes), ReadWriteMany (multiple nodes), ReadWriteOncePod (single pod).

4. **How to perform volume snapshots?**
   - **Answer:** Use VolumeSnapshot API to create point-in-time copies of PersistentVolumeClaims.

5. **What is the difference between EBS and EFS?**
   - **Answer:** EBS is single-AZ, single-node. EFS is multi-AZ, multi-node, supports ReadWriteMany access mode.

---

## 📚 Resources

- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)

---

## 🎯 Today's Assignment

1. **Create:** A StorageClass for dynamic provisioning
2. **Create:** A PVC requesting 5Gi storage
3. **Deploy:** A pod mounting the PVC
4. **Test:** Data persistence across pod restarts

---

## 📊 Progress Tracker

```
Day 1-4: ✅ Complete
Day 5: Persistent Storage ⬜
Day 6: ConfigMaps and Secrets Practice ⬜
...
```

---

## 🔗 Next Day Preview

**Day 6:** ConfigMaps and Secrets Practice
- Hands-on exercises
- Real-world scenarios
- Best practices

---

**Remember:** Persistent storage is essential for stateful applications! 🚀
