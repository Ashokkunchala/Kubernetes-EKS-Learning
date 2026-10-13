# 📅 Day 25: Advanced Storage - Interview Preparation & Assessment

## 🎯 Learning Objectives
By the end of today, you will:
- Review all storage concepts (Days 5, 12, 21-24)
- Practice storage interview questions
- Complete storage scenarios
- Pass a storage assessment

---
## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **PV / PVC** | A PersistentVolume represents storage; a PersistentVolumeClaim asks for storage. |
| **StatefulSet** | A controller for workloads that need stable Pod identities and often persistent storage. |
| **CSI** | The Container Storage Interface, which lets Kubernetes storage drivers integrate with providers. |
| **Snapshot** | A point-in-time storage copy that can be used as a restore source. |
| **EBS vs EFS** | EBS is block storage typically tied to one AZ; EFS is shared network file storage for concurrent clients. |
| **Binding mode** | StorageClass behavior controlling when a claim is matched/provisioned, including waiting until a Pod's scheduling constraints are known. |

**Analogy:** Pick storage like you pick a vehicle: a fast single-user disk, a shared filing system, and a backup snapshot solve different problems.

**Try / observe:** Explain a storage design aloud: access pattern → durability → zone/multi-zone needs → performance → backup/restore → cost.

**Troubleshooting clue:** Never assume a PVC being Bound proves the application can mount, write, or recover the data. Validate all three.

**Explain it back:** When would you choose EBS, EFS, or a snapshot? What does a StatefulSet add beyond a Deployment?

---


## 📋 Storage Concepts Review

### Quick Self-Test

```
1. What are the 3 main storage types in AWS?
   Answer: Block (EBS), File (EFS), Object (S3)

2. How many access modes does Kubernetes support?
   Answer: RWO, ROX, RWX, RWOP

3. What's the CSI driver for EBS?
   Answer: aws-ebs-csi-driver (ebs.csi.aws.com)

4. What tool backs up cluster resources?
   Answer: Velero

5. What's the difference between PV and PVC?
   Answer: PV = actual storage, PVC = request
```

### Storage Decision Tree

```
┌─────────────────────────────────────────────────────────┐
│              STORAGE DECISION TREE                        │
│                                                         │
│  Question 1: Does your app need persistent data?        │
│  ├── NO → Use emptyDir (ephemeral)                     │
│  └── YES → Question 2                                  │
│                                                         │
│  Question 2: Do multiple nodes/pods need shared access? │
│  ├── NO → EBS (Block Storage)                          │
│  └── YES → Question 3                                  │
│                                                         │
│  Question 3: Do you need high performance?              │
│  ├── YES (ML/HPC) → FSx for Lustre                    │
│  └── NO → Question 4                                   │
│                                                         │
│  Question 4: Is it file or object data?                  │
│  ├── File → EFS (Shared Filesystem)                    │
│  └── Object → S3 (Object Storage)                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Interview Scenario Questions

### Scenario 1: Database on Kubernetes

**Question:** How would you design a highly available PostgreSQL on EKS?

**Answer:**
```yaml
# Use StatefulSet (pod identity + stable storage)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 3
  serviceName: postgres
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-data
```

**Best Practices:**
- Use `StatefulSet` for stable pod identity
- Use `Headless Service` for direct pod access
- Use EBS gp3/io2 for performance
- Use EFS for shared (if needed)
- Take regular snapshots
- Use Velero for backup

---

### Scenario 2: Content Delivery at Scale

**Question:** Design storage for a video streaming platform.

**Answer:**
```
├── User videos           → S3 (large files, durable)
├── Video cache           → EFS (shared across pods)
├── CDN origin            → S3 + CloudFront
├── User data             → PostgreSQL (EBS)
├── Session data          → Redis (memory)
├── Thumbnails            → S3 + CDN
├── Analytics             → S3 + Athena
└── Backups               → Velero + S3
```

---

### Scenario 3: Cost Optimization

**Question:** A company is spending too much on storage. How do you reduce costs?

**Answer:**
```
1. Right-size EBS volumes (not too large)
2. Use gp3 instead of gp2 (cheaper)
3. Lifecycle S3 data (IA, Glacier)
4. Delete unused/unattached volumes
5. Use EFS Infrequent Access
6. Clean up old snapshots
7. Use Spot instances where possible
8. Monitor and alert on storage costs
```

---

## 📝 Storage Interview Questions (All Levels)

### Beginner (0-2 years)

1. **What is PersistentVolume?**
   - **Answer:** A piece of storage in the cluster that is provisioned by an administrator. It's cluster-level storage with lifecycle independent of pods.

2. **What is PersistentVolumeClaim?**
   - **Answer:** A request for storage by a user. It specifies size, access modes, and storage class. It's bound to a PV.

3. **What is a StorageClass?**
   - **Answer:** Defines a "class" of storage with provisioning parameters (like EBS gp3). Enables dynamic PV creation.

4. **What are Kubernetes access modes?**
   - **Answer:** ReadWriteOnce (single node), ReadOnlyMany, ReadWriteMany, ReadWriteOncePod.

5. **What is the difference between EBS and EFS?**
   - **Answer:** EBS is block storage for a single node. EFS is file storage shared across multiple nodes.

### Intermediate (2-5 years)

6. **Explain the CSI (Container Storage Interface).**
   - **Answer:** CSI is a standard interface that allows Kubernetes to use different storage systems. The EBS CSI Driver, EFS CSI Driver, etc. implement this interface.

7. **How do you perform a volume snapshot?**
   - **Answer:** Create a VolumeSnapshot resource with a VolumeSnapshotClass. The CSI driver creates the snapshot in AWS.

8. **What is Velero and what does it do?**
   - **Answer:** Velero backs up cluster resources (Kubernetes objects) and persistent volumes. Can restore to the same or different cluster.

9. **How do you dynamically provision storage?**
   - **Answer:** Create a PVC with a StorageClass. The provisioner (CSI driver) automatically creates the PV and storage.

10. **How do you expand a PVC?**
    - **Answer:** Jika StorageClass has `allowVolumeExpansion: true`, update the PVC's storage request. The volume is expanded automatically.

### Senior (5+ years)

11. **Design a storage solution for a microservices platform.**
    - **Answer:** Combination of EBS (databases), EFS (shared services), S3 (object storage), with snapshots and Velero for DR.

12. **How do you handle stateful applications on EKS?**
    - **Answer:** Use StatefulSets, Headless Services, EBS with proper access modes, and implement proper backup strategies.

13. **What are the tradeoffs between self-managed and managed storage?**
    - **Answer:** Managed (RDS, ElastiCache) has less control but easier ops. Self-managed (EKS with EBS) has more control but more ops responsibility.

14. **How do you automate storage management?**
    - **Answer:** Use IaC (Terraform), StorageClasses for dynamic provisioning, Velero for scheduled backups, and monitoring/alerting.

15. **How do you test disaster recovery for storage?**
    - **Answer:** Regularly restore from snapshots in a test environment, verify data integrity, and measure RTO/RPO.

---

## 📝 Final Assessment

### Assessment 1: Multiple Choice

1. Which access mode allows multiple pods to read AND write?
   - A) ReadWriteOnce
   - B) ReadWriteMany ✅
   - C) ReadOnlyMany
   - D) ReadWriteOncePod

2. Which storage is best for a MySQL database?
   - A) EFS
   - B) S3
   - C) EBS ✅
   - D) Local disk

3. Which tool backs up entire Kubernetes clusters?
   - A) kubectl
   - B) Helmsman
   - C) Velero ✅
   - D) K9s

4. What does `allowVolumeExpansion` do?
   - A) Allows PVC to grow ✅
   - B) Allows more replicas
   - C) Enables snapshots
   - D) Enables encryption

5. What is the provisioner for EBS?
   - A) ebs.csi.aws.com ✅
   - B) efs.csi.aws.com
   - C) s3.csi.aws.com
   - D) fsx.csi.aws.com

### Assessment 2: Scenario-Based

**You're building a service that:**
1. Needs a database (PostgreSQL)
2. Serves user-generated content (images/videos)
3. Needs shared file system across 5 pods
4. Should survive node failures

**Design:**

```
Storage Needs:
1. Database: EBS (gp3) - ReadWriteOnce
   - Single pod, fast, durable
   - Regular snapshots

2. User content: S3 - Object storage
   - Durable, scalable, cheap
   - Accessed via SDK/IRSA

3. Shared files: EFS (EFS) - ReadWriteMany
   - Accessed by all 5 pods
   - Automatically scales

4. Backup: Velero + EBS snapshots
   - Scheduled daily
   - Stored in S3
```

---

## 📚 Resources for Review

- [Day 5: Persistent Storage](Day-05_Persistent_Storage/README.md)
- [Day 12: EKS Storage with EBS](Day-12_EKS_Storage_EBS/README.md)
- [Day 21: EFS CSI Driver](Day-21_EFS_CSI_Driver/README.md)
- [Day 22: FSx for Lustre](Day-22_FSx_Lustre/README.md)
- [Day 23: Volume Snapshots](Day-23_Volume_Snapshots_Restore/README.md)
- [Day 24: Advanced Storage Practice](Day-24_Advanced_Storage_Practice/README.md)

---

## 🎯 Today's Assignment

1. **Review:** All storage concepts
2. **Complete:** The assessment
3. **Practice:** Answering questions aloud
4. **Build:** A simple stateful application with snapshots

---

## 📊 Progress Tracker

```
Week 5-6 Storage Complete:
Day 21: EFS CSI Driver ✅
Day 22: FSx for Lustre & S3 ✅
Day 23: Volume Snapshots & Backups ✅
Day 24: Advanced Storage Practice ✅
Day 25: Storage Interview Preparation ⬜
```

---

## 🔗 Next Week Preview

**Week 7-8:** Advanced IAM and Security
- Day 26: OIDC Provider Setup
- Day 27: IRSA for Multiple Services
- Day 28: Pod-Level IAM Permissions
- Day 29: Secrets Manager Integration
- Day 30: Advanced IAM Practice

---

**Remember:** Master storage to build reliable, durable, and cost-effective applications! 🚀
