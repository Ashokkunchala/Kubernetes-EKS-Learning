# 📅 Day 23: Volume Snapshots, Restore & Backups

## 🎯 Learning Objectives
By the end of today, you will understand:
- Creating volume snapshots
- Restoring from snapshots
- Backup strategies with Velero
- Complete backup/disaster recovery flow
- Hands-on practical backup lab

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Snapshot** | A point-in-time storage copy used as a recovery source. |
| **Restore** | Creating a usable volume/data set from a backup or snapshot. |
| **CSI snapshot controller** | Kubernetes components that coordinate volume snapshot API requests with a storage driver. |
| **RPO (Recovery Point Objective)** | The maximum acceptable amount of data loss measured in time. |
| **RTO (Recovery Time Objective)** | The target maximum time to restore service after an incident. |
| **Data consistency** | Whether the restored data is mutually coherent, especially across active writes or multiple volumes. |

**Analogy:** A snapshot is a saved game; RPO says how far back you can tolerate going, and RTO says how long you can tolerate being unable to play.

**Try / observe:** Verify the snapshot class/driver, create a test snapshot, restore to a new claim, and validate the application data before trusting the procedure.

**Troubleshooting clue:** A snapshot object can exist while the restore is not usable. Check snapshot readiness, driver support, events, capacity, zone constraints, and application consistency.

**Explain it back:** Why is “backup succeeded” not enough evidence that disaster recovery works?

---

> **Lab credential safety:** These examples use a placeholder, not a usable password. Before running a database lab, replace `REPLACE_WITH_STRONG_LAB_PASSWORD` in the manifest with a strong, lab-only value and set `MYSQL_ROOT_PASSWORD` to the same value in your Bash shell (for example, `export MYSQL_ROOT_PASSWORD='your-lab-only-value'`). Do not reuse real credentials or commit them to Git. For production, use a Kubernetes Secret with restricted access or an external secret manager.

## 📖 Core Concepts

### What are Volume Snapshots?

**Volume Snapshots** are point-in-time copies of persistent volumes. They enable data backup, restore, and migration.

```
┌─────────────────────────────────────────────────────────┐
│              VOLUME SNAPSHOT ARCHITECTURE                 │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Kubernetes API                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  VolumeSnapshot Class                    │   │   │
│  │  │  (Like StorageClass for snapshots)       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  VolumeSnapshot                          │   │   │
│  │  │  (Request for snapshot)                  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  VolumeSnapshotContent                   │   │   │
│  │  │  (Actual snapshot)                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EBS CSI Driver                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Creates snapshot                     │   │   │
│  │  │  • Handles restore                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Amazon EBS                                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Volume Snapshot: snap-1234567890       │   │   │
│  │  │  • Point-in-time copy                   │   │   │
│  │  │  • S3 backed (durable)                  │   │   │
│  │  │  • Restorable to new volume             │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: Snapshot and Restore

### Step 1: Verify Snapshot Support

```bash
# Check if snapshot is enabled for EBS CSI Driver
kubectl get crd volumesnapshotclasses.snapshot.storage.k8s.io

# Check snapshot controller
kubectl get pods -n kube-system | grep snapshot
```

### Step 2: Create a Database with Data

```yaml
# mysql-pvc.yaml
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
      storage: 10Gi
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
          value: "REPLACE_WITH_STRONG_LAB_PASSWORD"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: mysql-data
```

```bash
kubectl apply -f mysql-pvc.yaml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s

# Insert data
MYSQL_POD=$(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $MYSQL_POD -- mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "CREATE DATABASE testdb; CREATE TABLE testdb.users (id INT, name VARCHAR(50)); INSERT INTO testdb.users VALUES (1, 'Alice'), (2, 'Bob');"

# Verify data exists
kubectl exec -it $MYSQL_POD -- mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "SELECT * FROM testdb.users;"
```

### Step 3: Create Volume Snapshot

```yaml
# mysql-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tagSpecification: 'SnapshotTag=test'
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: mysql-data
```

```bash
kubectl apply -f mysql-snapshot.yaml

# Wait for snapshot to complete
kubectl get volumesnapshot mysql-snapshot
kubectl wait --for=jsonpath='{.status.readyToUse}'=true volumesnapshot mysql-snapshot --timeout=180s

# Verify in AWS
aws ec2 describe-snapshots --snapshot-ids $(kubectl get volumesnapshotcontent -o jsonpath='{.items[0].status.snapshotHandle}')
```

### Step 4: Simulate Data Loss

```bash
# Delete database
kubectl delete -f mysql-pvc.yaml

# Verify data is gone
kubectl get pvc mysql-data  # Should show Terminating or gone
```

### Step 5: Restore from Snapshot

```yaml
# mysql-restore.yaml
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
      storage: 10Gi
  dataSource:
    name: mysql-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-restored
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-restored
  template:
    metadata:
      labels:
        app: mysql-restored
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "REPLACE_WITH_STRONG_LAB_PASSWORD"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: mysql-data-restored
```

```bash
kubectl apply -f mysql-restore.yaml

# Wait for restore
kubectl wait --for=condition=ready pod -l app=mysql-restored --timeout=120s

# Verify data was restored
MYSQL_RESTORED=$(kubectl get pods -l app=mysql-restored -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $MYSQL_RESTORED -- mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "SELECT * FROM testdb.users;"
# Should show: 1 Alice, 2 Bob
```

---

## 🛠️ Complete Backup Solution: Velero

### What is Velero?

**Velero** is a backup/restore tool for Kubernetes that backs up cluster resources AND persistent volumes.

### Install Velero

```bash
# Download Velero
# Windows/Chocolatey
choco install velero

# Create S3 bucket for backups
aws s3 mb s3://my-eks-backup-bucket

# Create IAM Policy and role
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=velero \
  --namespace=velero \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --approve

# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket my-eks-backup-bucket \
  --backup-location-config region=us-west-2 \
  --snapshot-location-config region=us-west-2 \
  --service-account-name velero
```

### Backup and Restore with Velero

```bash
# Create an on-demand backup
velero backup create my-full-backup --include-namespaces default,ecommerce

# Check backup status
velero backup describe my-full-backup

# Create scheduled backup (daily at 2am)
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces default,ecommerce

# List all backups
velero get backups

# Restore from backup
velero restore create --from-backup my-full-backup

# Check restore status
velero restore describe <restore-name>
```

### Backup Schedule YAML

```yaml
# velero-schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
    - default
    - ecommerce
    ttl: 720h  # 30 days retention
    storageLocation: default
    volumeSnapshotLocations:
    - default
```

---

## 🖥️ Real-World Backup Strategy

```
┌─────────────────────────────────────────────────────────┐
│              BACKUP STRATEGY DESIGN                       │
│                                                         │
│  Tier 1: Real-time                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • RDS automated backups                        │   │
│  │  • S3 versioning                                │   │
│  │  • Replication across AZs                       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Tier 2: Daily (Velero)                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • Cluster resources (deployments, configs)     │   │
│  │  • Persistent volumes (EBS snapshots)           │   │
│  │  • Namespace configuration                     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Tier 3: Weekly                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • Full cluster backup to S3                    │   │
│  │  • Cross-region copy                            │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  RTO (Recovery Time Objective): 1 hour                 │
│  RPO (Recovery Point Objective): 24 hours               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🐛 Troubleshooting

```bash
# Snapshot issues
kubectl describe volumesnapshot <name>
kubectl describe volumesnapshotcontent <content-name>

# Velero issues
kubectl get pods -n velero
velero get backups
velero logs <backup-name>

# Check backup location
velero backup-location get

# Restore issues
velero restore logs <restore-name>
```

---

## 📚 Resources

- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Velero Documentation](https://velero.io/docs/main/)
- [EBS Snapshots](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html)

---

## 🎯 Today's Assignment

1. **Create:** A volume with important data
2. **Snapshot:** Take a point-in-time copy
3. **Delete:** The original volume
4. **Restore:** From the snapshot
5. **Verify:** The data is back

---

## 📊 Progress Tracker

```
Day 22: FSx for Lustre ✅
Day 23: Volume Snapshots & Backups ⬜
```

---

## 🔗 Next Day Preview

**Day 24:** Advanced Storage Practice
- Complete storage lab
- Multi-storage application
- Real-world scenario

---

**Remember:** Backups are your insurance policy - always have a recovery plan! 🚀
