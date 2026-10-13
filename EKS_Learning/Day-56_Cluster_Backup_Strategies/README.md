# 📅 Day 56: Cluster Backup Strategies (Velero)

## 🎯 Learning Objectives
By the end of today, you will understand:
- What to back up in an EKS cluster
- Velero architecture and installation
- Backing up Kubernetes resources + persistent volumes
- Scheduled and on-demand backups to S3
- Restore and backup verification strategies

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Backup** | A recoverable copy of selected data or configuration. |
| **Restore** | Using a backup to recreate the data/resources needed for service. |
| **etcd** | The Kubernetes key-value store that holds cluster API state; in EKS, AWS manages the control-plane etcd layer. |
| **Kubernetes objects** | API resources such as Deployments, Services, ConfigMaps, and policy objects. |
| **Persistent data** | Application data stored beyond the lifetime of a Pod/container. |
| **Recovery test** | A practical exercise proving that backups can be restored to a usable state. |

**Analogy:** A backup is a spare key; a restore test proves that the key actually opens the door.

**Try / observe:** Define what is being backed up (manifests, application data, external databases, secrets/configuration), who owns it, retention, and how restoration is verified.

**Troubleshooting clue:** A backup of Kubernetes YAML alone does not back up data in EBS/EFS, RDS, or external systems. Map every stateful dependency.

**Explain it back:** What should be backed up outside the cluster? Why is a restore test mandatory?

---

## 📖 Core Concepts

### What to Back Up

```
┌───────────────────────────────────────────────────────────────┐
│                WHAT EXISTS IN A CLUSTER                        │
│                                                               │
│  ┌────────────────────────────────────────────────────┐      │
│  │  CONFIG / STATE (etcd-resident objects)            │      │
│  │  Deployments, Services, ConfigMaps, Secrets,       │      │
│  │  CRDs, PDBs, RBAC, Namespaces, PVCs, Ingresses...  │      │
│  └────────────────────────────────────────────────────┘      │
│  ┌────────────────────────────────────────────────────┐      │
│  │  DATA (persistent volumes)                        │      │
│  │  EBS volumes / EFS / S3-referenced data            │      │
│  └────────────────────────────────────────────────────┘      │
│                                                               │
│  Velero captures BOTH: resource objects (API) +              │
│  volume snapshots (via CSI / cloud snapshot).                │
└───────────────────────────────────────────────────────────────┘
```

### Key Concepts

- **Backup:** Captures selected resources + optional PV snapshots at a point in time.
- **Restore:** Recreates resources from a backup into (possibly another) cluster.
- **Schedule:** Recurring backups, e.g., daily.
- **Storage location:** Where backups live (S3 bucket).
- **Ansibles:** Only captures API objects — the app data in DBs is handled separately.

---

## 🛠️ Step 1: Install Velero

```bash
# Install the CLI
brew install velero   # or download binary

# Create the S3 bucket
aws s3 mb s3://my-eks-backups --region us-east-1

# Create the IAM policy (Velero uses IRSA)
```

`velero-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot", "ec2:CreateTags", "ec2:DeleteSnapshot",
        "ec2:DescribeSnapshots", "ec2:DescribeVolumes",
        "s3:PutObject", "s3:GetObject", "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam create-policy --policy-name velero-policy \
  --policy-document file://velero-policy.json

# Install with IRSA
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket my-eks-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-restic \
  --use-volume-snapshots=true \
  --irsa-role velero-role \
  --set-file serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT:role/velero-role \
  --namespace velero
```

```bash
# Verify
velero get backup-locations
velero get snapshot-locations
kubectl -n velero get pods
```

---

## 🛠️ Step 2: Create a Backup

```bash
# On-demand backup of everything in production
velero backup create prod-full --include-namespaces production

# Backup specific resources
velero backup create web-backup \
  --include-namespaces production \
  --include-resources deployments,configmaps,secrets,services,pvc

# Backup with label selector
velero backup create web-labelled \
  --selector app=web

# Track progress
velero backup describe prod-full
velero backup logs prod-full
```

---

## 🛠️ Step 3: Scheduled Backups

```bash
velero schedule create daily-backup \
  --schedule "0 2 * * *" \
  --include-namespaces production \
  --ttl 720h   # keep 30 days

velero schedule get
```

Schedule with retention & snapshot:

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: prod-daily
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    ttl: 720h
    includedNamespaces: [production]
    includeClusterResources: true
    snapshotVolumes: true
```

---

## 🛠️ Step 4: Restore

```bash
# List backups
velero backup get

# Restore a backup
velero restore create --from-backup prod-full

# Restore into another cluster/namespace with remapping
velero restore create --from-backup prod-full \
  --namespace-mappings production:prod-restored

velero restore describe <restore-name>
velero restore logs <restore-name>
```

Verify:

```bash
kubectl -n production get deploy,svc,pvc
velero restore get
```

---

## 🛠️ Step 5: Backup Verification & DR Drill

**Test restores regularly** into a scratch namespace/cluster — a backup you never restore is a backup that may not work:

```bash
# Restore into a scratch namespace
velero restore create --from-backup prod-full \
  --namespace-mappings production:dr-verify

# Smoke test the restored app
kubectl -n dr-verify port-forward svc/web 8080:80
curl localhost:8080/health
```

Check backup health alerting (Prometheus velero metrics):

```promql
velero_backup_attempt_total
velero_backup_errors_total
```

---

## 📝 Interview Questions

1. **What does Velero back up in a cluster?**
   - **Answer:** API objects (Deployments, Secrets, ConfigMaps, RBAC, CRDs, PVCs, namespaces) plus optional persistent-volume snapshots (EBS/CSI), stored in S3.

2. **How do you do a point-in-time restorable backup on EKS?**
   - **Answer:** Velero on a schedule, capturing namespaces + volumes to S3 with TTL retention; plus app-level DB backup (RDS snapshots) which Velero doesn't cover.

3. **Why test restores?**
   - **Answer:** To validate the backup is complete and the restore path works in a real outage, ensuring your RTO/RPO targets are actually achievable.

4. **What's NOT backed up by Velero?**
   - **Answer:** Data managed outside the cluster API (RDS, ElastiCache, S3 content), and the etcd/control-plane (for managed EKS, AWS manages the control plane).

5. **Backup vs replication vs snapshot?**
   - **Answer:** Backup = restorable copy (point-in-time); snapshot = volume-level copy; replication = continuous copies for failover (low RPO). Use all appropriately per data class.

---

## 📚 Resources for Review

- [Day 57: Cross-Region Replication](Day-57_Cross_Region_Replication/README.md)
- [Day 58: Failover Procedures](Day-58_Failover_Procedures/README.md)
- [Day 59: Disaster Recovery Practice](Day-59_Disaster_Recovery_Practice/README.md)

### External
- [Velero](https://velero.io/)
- [AWS Backup + EKS](https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-eks.html)

---

## 🎯 Today's Assignment

1. **Install** Velero with S3 + IRSA
2. **Create** an on-demand backup of a namespace
3. **Set up** a daily schedule with TTL
4. **Perform** a restore into a scratch namespace and validate
5. **Add** Velero backup-failure alerts

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

**Day 57:** Cross-Region Replication

---

**Remember:** A backup you can't restore is not a backup. Validate restores routinely! 🚀
