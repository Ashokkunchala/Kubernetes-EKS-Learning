# 📅 Day 24: Advanced Storage Practice

## 🎯 Learning Objectives
By the end of today, you will:
- Build a complete multi-storage application
- Use EBS, EFS, and S3 together
- Implement storage best practices
- Configure storage policies
- Solve real-world storage challenges

---

## 🖥️ Project: Complete Storage Solution for a Content Management System

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│        CONTENT MANAGEMENT SYSTEM (CMS)                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Web Tier (NGINX)                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Serves static content                │   │   │
│  │  │  • 3 replicas for HA                    │   │   │
│  │  │  • Uses EFS for shared content          │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Application Tier                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Application logic                    │   │   │
│  │  │  • Uses S3 for media storage             │   │   │
│  │  │  • Uses EBS for app data                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Database Tier                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • MySQL database                       │   │   │
│  │  │  • EBS for data (fast, durable)          │   │   │
│  │  │  • Snapshots for backups                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Storage Summary:                                       │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EBS:  Database data (fast, durable)            │   │
│  │  EFS:  Web content (shared across pods)          │   │
│  │  S3:   Media files (durable, cheap)             │   │
│  │  Snapshots: Database backups                    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Step-by-Step Implementation

### Step 1: Setup EFS for Web Content

```yaml
# shared-web-content.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: web-content-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-12345678  # Your EFS ID
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-content-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

### Step 2: Setup EBS for Database

```yaml
# database-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
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
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 20Gi
```

### Step 3: Setup S3 for Media

```yaml
# s3-media.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: s3-sc
provisioner: s3.csi.aws.com
parameters:
  bucket: my-cms-media-bucket
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: media-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: s3-sc
  resources:
    requests:
      storage: 100Gi
```

### Step 4: Deploy Web Tier (uses EFS)

```yaml
# web-tier.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: web-content
        persistentVolumeClaim:
          claimName: web-content-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

### Step 5: Deploy Database (uses EBS)

```yaml
# database-deployment.yaml
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
        - name: data
          mountPath: /var/lib/mysql
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: mysql-data
```

### Step 6: Deploy Application (uses S3)

```yaml
# application-tier.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cms-app
  template:
    metadata:
      labels:
        app: cms-app
    spec:
      serviceAccountName: s3-access-sa
      containers:
      - name: app
        image: node:18-alpine
        command: ["node", "-e", "console.log('CMS App running'); setInterval(()=>{}, 1000)"]
        volumeMounts:
        - name: media
          mountPath: /media
      volumes:
      - name: media
        persistentVolumeClaim:
          claimName: media-pvc
```

```bash
# Apply all storage
kubectl apply -f shared-web-content.yaml
kubectl apply -f database-storage.yaml
kubectl apply -f s3-media.yaml

# Apply all tiers
kubectl apply -f web-tier.yaml
kubectl apply -f database-deployment.yaml
kubectl apply -f application-tier.yaml

# Verify everything
kubectl get pods -o wide
kubectl get pvc
kubectl get pv
```

---

## 📝 Practice Lab: Storage Operations

### Lab 1: Test Shared Content Across Pods

```bash
# Write from pod 1
kubectl exec -it $(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}') -- sh -c "echo 'Welcome to CMS' > /usr/share/nginx/html/index.html"

# Read from pod 2 (same content - EFS works!)
kubectl exec -it $(kubectl get pods -l app=web -o jsonpath='{.items[1].metadata.name}') -- cat /usr/share/nginx/html/index.html

# Read from pod 3
kubectl exec -it $(kubectl get pods -l app=web -o jsonpath='{.items[2].metadata.name}') -- cat /usr/share/nginx/html/index.html
```

### Lab 2: Database with Persistent Data

```bash
# Connect to MySQL
MYSQL_POD=$(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}')

# Create data
kubectl exec -it $MYSQL_POD -- mysql -u root -ppassword123 -e "CREATE DATABASE cms; CREATE TABLE cms.posts (id INT, title VARCHAR(100), body TEXT); INSERT INTO cms.posts VALUES (1, 'Hello World', 'First post');"

# Delete pod (data should persist)
kubectl delete pod $MYSQL_POD
kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s

# Verify data still exists (EBS persistence)
NEW_MYSQL=$(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $NEW_MYSQL -- mysql -u root -ppassword123 -e "SELECT * FROM cms.posts;"
```

### Lab 3: S3 Media Access

```bash
# Check S3 file system
kubectl exec -it $(kubectl get pods -l app=cms-app -o jsonpath='{.items[0].metadata.name}') -- ls /media
```

### Lab 4: Backup and Restore

```bash
# Create snapshot of database
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-backup
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: mysql-data
EOF

# Simulate disaster (delete everything)
kubectl delete deployment mysql web app
kubectl delete pvc mysql-data web-content-pvc media-pvc

# Restore from snapshot
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-restored
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 20Gi
  dataSource:
    name: mysql-backup
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
EOF
```

---

## 📝 Storage Best Practices Checklist

### Performance
- [ ] Use EBS gp3 for general workloads
- [ ] Use io2 for high-performance databases
- [ ] Use EFS for shared content
- [ ] Use S3 for object/blob storage
- [ ] Monitor storage performance

### Durability
- [ ] Enable EBS encryption
- [ ] Take regular snapshots
- [ ] Enable S3 versioning
- [ ] Cross-region backups
- [ ] Test restore procedures

### Cost
- [ ] Use EBS gp3 (cheaper than gp2)
- [ ] Delete unused volumes
- [ ] Use S3 lifecycle policies
- [ ] Monitor storage costs
- [ ] Right-size storage

### Security
- [ ] Enable encryption at rest
- [ ] Use IRSA for storage access
- [ ] Restrict S3 bucket access
- [ ] Use network policies
- [ ] Audit storage access

---

## 🐛 Common Storage Issues & Solutions

| Issue | Symptom | Solution |
|-------|---------|----------|
| PVC Pending | Stuck in Pending | Check StorageClass exists |
| Pod ContainerCreating | Can't mount volume | Check CSI Driver |
| ReadOnlyMany | Can't write | Check access mode |
| Slow performance | High latency | Upgrade volume type |
| Data loss | Data missing | Enable snapshots |
| Out of space | Volume full | Expand volume |

---

## 📝 Interview Questions

1. **Design storage for a photo-sharing app.**
   - **Answer:** Use S3 for user photos (durable, cheap), EFS for shared content delivery, EBS for app databases (MySQL), all with snapshots.

2. **How do you handle storage for a multi-region app?**
   - **Answer:** Use EFS for shared storage, replicate S3 cross-region, use RDS multi-region for databases.

3. **What are the tradeoffs between block, file, and object storage?**
   - **Answer:** Block (EBS) - fast, single-node. File (EFS) - shared, multi-node. Object (S3) - durable, cheap, HTTP access.

4. **How do you optimize storage costs?**
   - **Answer:** Use appropriate volume types (gp3 over gp2), lifecycling S3 data, deleting unused volumes, and using snapshots judiciously.

5. **What's your backup and DR strategy?**
   - **Answer:** Velero for cluster resources, EBS snapshots for volumes, RDS backups for databases, S3 versioning for object storage, with regular restore testing.

---

## 📚 Resources

- [EKS Storage](https://docs.aws.amazon.com/eks/latest/userguide/storage.html)
- [Kubernetes Storage](https://kubernetes.io/docs/concepts/storage/)
- [Storage Best Practices](https://docs.aws.amazon.com/whitepapers/latest/storage-optimization/best-practices.html)

---

## 🎯 Today's Assignment

1. **Build:** The complete CMS with multiple storage tiers
2. **Test:** Data persistence (restart pods)
3. **Test:** Data sharing (multiple pods)
4. **Backup:** Create snapshots
5. **Restore:** Practice disaster recovery

---

## 📊 Progress Tracker

```
Day 23: Volume Snapshots & Backups ✅
Day 24: Advanced Storage Practice ⬜
```

---

## 🔗 Next Day Preview

**Day 25:** Storage Interview Preparation
- Review all storage concepts
- Practice scenarios
- Assessment

---

**Remember:** Use the RIGHT storage for the RIGHT workload - know your tradeoffs! 🚀
