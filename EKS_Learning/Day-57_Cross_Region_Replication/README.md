# 📅 Day 57: Cross-Region Replication

## 🎯 Learning Objectives
By the end of today, you will understand:
- The RPO/RTO framework for disaster recovery
- Replicating container images to ECR across regions
- Replicating data (RDS, EFS, S3) across regions
- Cross-region EKS infrastructure replication
- Continuous replication vs periodic backup

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **AWS Region** | A geographic area containing multiple Availability Zones. |
| **Replication** | Copying data or configuration from one location to another, often continuously or periodically. |
| **Replication lag** | The delay between a change at the source and its availability at the destination. |
| **RPO** | Recovery Point Objective: how much recent data loss, measured in time, is acceptable. |
| **RTO** | Recovery Time Objective: how long restoring service is allowed to take. |
| **Failover** | Switching service to a prepared alternate location/system. |

**Analogy:** Replication keeps a second copy of a notebook; lag is how far behind it is, RPO is the acceptable missing pages, and RTO is how long it takes to resume writing.

**Try / observe:** Verify that data and images are actually replicated, credentials and networking work in the destination Region, and restore/failover steps are tested.

**Troubleshooting clue:** A replicated resource does not guarantee an application is recoverable. Check dependencies, secret availability, capacity, DNS, and the data replication state.

**Explain it back:** How do RPO and RTO differ? Why must cross-Region recovery be tested end to end?

---

## 📖 Core Concepts

### RPO / RTO

```
┌───────────────────────────────────────────────────────────────┐
│                 RECOVERY FRAMEWORK                             │
│                                                               │
│  RPO (Recovery Point Objective) = max acceptable data loss    │
│  RTO (Recovery Time Objective) = max acceptable downtime      │
│                                                               │
│  ┌──────────┐   strategy   ┌──────────┐                       │
│  │   RPO    │──▶  sync vs  │   RTO    │                       │
│  │   RTO    │    async rep │           │                      │
│  └──────────┘              └──────────┘                       │
│                                                               │
│  · RPO ~ 0      → synchronous replication (Aurora Global,     │
│                   DynamoDB Global Tables)                     │
│  · RPO minutes   → async replication (S3 CRR, RDS async)     │
│  · RPO hours     → periodic backups (Velero, EBS snapshots)   │
└───────────────────────────────────────────────────────────────┘
```

### What to Replicate

| Asset | Region 1 | Region 2 | Mechanism |
|-------|----------|----------|-----------|
| Container images | ECR us-east-1 | ECR us-west-2 | ECR replication rule |
| Manifests/Gitops | Git repo | (region-independent) | Git |
| DB data | RDS primary | RDS read replica | RDS cross-region |
| Files/objects | S3 bucket | S3 replica | S3 CRR |
| Volumes | EBS/EKS | Gelane/CSI | Backup → restore |

---

## 🛠️ Step 1: Replicate Container Images to ECR

```bash
# Create ECR replication rule (pushes copies to all regions)
aws ecr put-replication-configuration \
  --region us-east-1 \
  --replication-configuration '{
    "rules":[{
      "destinations":[{"region":"us-west-2","registryId":"ACCOUNT_ID"}],
      "repositoryFilters":[{"filterType":"PREFIX_MATCH","filter":"prod/"}]
    }]
  }'
```

Now any image pushed to `prod/*` in us-east-1 is auto-copied to us-west-2.

---

## 🛠️ Step 2: Replicate Data

### RDS Cross-Region Read Replica (async, low RPO)

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier app-replica-west \
  --source-db-instance-identifier app-primary-east \
  --region us-west-2 \
  --availability-zone us-west-2a
```

> For RPO≈0 and fast failover to a writable secondary, use **Aurora Global Database**.

### S3 Cross-Region Replication (CRR)

```bash
# Bucket A replication config
aws s3api put-bucket-replication \
  --bucket myapp-us-east \
  --replication-configuration '{
    "Role":"arn:aws:iam::ACCOUNT:role/s3crr-role",
    "Rules":[{
      "Status":"Enabled",
      "Priority":1,
      "DeleteMarkerReplication":{"Status":"Enabled"},
      "Filter":{"Prefix":""},
      "Destination":{"Bucket":"arn:aws:s3:::myapp-us-west","StorageClass":"STANDARD"}
    }]
  }'
```

### EFS / EBS for stateful apps

- **EFS:** enable **replication** to a destination region (managed).
- **EBS:** replicate via snapshot copy to the DR region (periodic).

```bash
# Copy latest EBS snapshot to DR region
SNAP=$(aws ec2 describe-snapshots --filters "Name=tag:Name,Values=app-data" \
  --query 'Snapshots[*].SnapshotId' --output text | head -1)
aws ec2 copy-snapshot --region us-west-2 \
  --source-region us-east-1 --source-snapshot-id $SNAP \
  --description "DR copy"
```

---

## 🛠️ Step 3: Replicate the Cluster Infrastructure

Use GitOps (ArgoCD) + IaC to (re)create the cluster in the DR region with the same config:

```bash
# Cluster-as-code: cluster.yaml per region
eksctl create cluster -f cluster-dr-west.yaml   # in us-west-2

# Deploy same apps via ArgoCD ApplicationSet pointing to DR cluster
# (Day 49 covered multi-cluster GitOps)
```

Infrastructure state lives in Git/Terraform — replication is a re-run, not a one-off.

---

## 🛠️ Step 4: Continuous vs Periodic Strategy Matrix

| Data | Sync/Async | RPO | Tool |
|------|-----------|-----|------|
| Container images | Periodic + event | minutes | ECR replication |
| Control-plane objects | Periodic | hours | Velero |
| DB (RDS/Aurora) | Async / Global | seconds | RR / Global DB |
| Object store (S3) | Async | minutes | S3 CRR |
| Managed EBS state | Periodic | hours | Snapshot copy |
| Git/Config | Region-independent | ~0 | Git |

---

## 📝 Interview Questions

1. **What is RPO and RTO, and how do they relate to EKS DR?**
   - **Answer:** RPO = max data loss; RTO = max downtime. They drive the replication/backup strategy: sync for ~0 RPO, async for minutes, periodic backups for hours.

2. **How do you replicate ECR images across regions?**
   - **Answer:** ECR replication rules copy pushed images to destination region registries automatically for chosen repo prefixes.

3. **How do you replicate an RDS database for DR?**
   - **Answer:** Cross-region read replica (async) or Aurora Global Database (sync, RPO≈0) for writable failover.

4. **What's the difference between replication and backup?**
   - **Answer:** Replication continuously copies state for fast failover (low RPO); backup is a restorable point-in-time copy (higher RPO) that also protects against logical corruption.

5. **How do you keep clusters consistent across regions?**
   - **Answer:** GitOps (ArgoCD) + IaC in Git; a DR cluster is recreated from Git/Terraform and synced by ArgoCD — config is already region-independent.

---

## 📚 Resources for Review

- [Day 56: Cluster Backup Strategies](Day-56_Cluster_Backup_Strategies/README.md)
- [Day 58: Failover Procedures](Day-58_Failover_Procedures/README.md)
- [Day 49: Multi-Cluster Practice](Day-49_Multi_Cluster_Practice/README.md)

---

## 🎯 Today's Assignment

1. **Define** RPO/RTO for a sample workload
2. **Set up** ECR cross-region replication
3. **Configure** S3 CRR for an object bucket
4. **Create** an RDS read replica in a second region
5. **Copy** an EBS snapshot to the DR region

---

## 📊 Progress Tracker

```
Week 19-20 Disaster Recovery:
Day 56: Cluster Backup Strategies ⬜
Day 57: Cross-Region Replication ⬜
Day 58: Failover Procedures ⬜
Day 59: Disaster Recovery Practice ⬜
Day 60: Final Review & Certification ⬜
```

---

## 🔗 Next Week Preview

**Day 58:** Failover Procedures

---

**Remember:** Pick your replication strategy from RPO/RTO — everything else follows! 🚀
